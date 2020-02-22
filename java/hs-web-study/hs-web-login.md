# hs-web登录和退出

## hs-web登录启动

在main方法所在类增加@`EnableAopAuthorize`注解启动登录功能，不过貌似如果去掉对应的注解，还是会调用登录的接口。所以登录是必须的。

## 登录流程

登录接口在`hsweb-authorization-basic`项目的`AuthorizationController`类中定义。

在`doLogin`方法中，先发布`AuthorizationDecodeEvent`事件，再发布`AuthorizationBeforeEvent`事件，再调用到`SimpleAuthenticationManager`的`authenticate`方法，如果没抛出异常，则发布`AuthorizationSuccessEvent`事件，如果抛出异常，发布`AuthorizationFailedEvent`事件。

在`SimpleAuthenticationManager`的`authenticate`方法中，先到数据表`s_user`中查询对应的用户名是否存在。如果存在，则在`getByUserId`方法中通过查询出的id构造`Authentication`对象并返回。

后续的操作都是获取用户的权限数据，需要了解权限表设计。[数据表设计](hs-web-db.md)

`getByUserId`方法先判断是否存在key为`userId`的缓存，如果不存在，则调用`SimpleAuthorizationSettingService`类的`initUserAuthorization`方法来获取，之后存入缓存。

如下是`initUserAuthorization`方法代码。

```java
public Authentication initUserAuthorization(String userId) {
    if (null == userId) {
        return null;
    }
    UserEntity userEntity = userService.selectByPk(userId);
    if (userEntity == null) {
        return null;
    }
    SimpleAuthentication authentication = new SimpleAuthentication();
    // 用户信息
    authentication.setUser(SimpleUser.builder()
            .id(userId)
            .username(userEntity.getUsername())
            .name(userEntity.getName())
            .type("default")
            .build());
    //角色
    authentication.setRoles(userService.getUserRole(userId)
            .stream()
            .map(role -> new SimpleRole(role.getId(), role.getName()))
            .collect(Collectors.toList()));

    List<String> settingIdList = getUserSetting(userId)
            .stream()
            .map(AuthorizationSettingEntity::getId)
            .collect(Collectors.toList());

    if (settingIdList.isEmpty()) {
        authentication.setPermissions(new ArrayList<>());
        return authentication;
    }

    // where status=1 and setting_id in (?,?,?)
    List<AuthorizationSettingDetailEntity> detailList = DefaultDSLQueryService
            .createQuery(authorizationSettingDetailDao)
            .where(status, STATE_OK)
            .and().in(settingId, settingIdList)
            .listNoPaging();

    authentication.setPermissions(initPermission(detailList));

    return authentication;
}
```

分为如下几步：

1. 通过`userId`查询表`s_user`获取`UserEntity`对象，实例化`SimpleAuthentication`类并设置`user`为`SimpleUser`，
2. 通过`userId`查询表`s_user_role`获取`RoleEntity`对象，并设置`role`为`SimpleRole`，
3. 通过`userId`调用`getUserSetting`方法获取`AuthorizationSettingEntity`，
4. 取出`AuthorizationSettingEntity`的`id`，并查询`s_autz_detail`表获取`setting_id`匹配的`AuthorizationSettingDetailEntity`，
5. 调用`initPermission`将`AuthorizationSettingDetailEntity`转化为`Permission`。

前两步较简单。

第3步中，`getUserSetting`方法中，循环调用`authorizationSettingTypeSuppliers`中的`get`方法，然后通过`SettingInfo`的`type`分组，如果有多个分组，则使用并行调用，再查询表`s_autz_setting`获取`type`和`settingFor`匹配的结果。

```java
private List<AuthorizationSettingEntity> getUserSetting(String userId) {
    Map<String, List<SettingInfo>> settingInfo =
            authorizationSettingTypeSuppliers.stream()
                    .map(supplier -> supplier.get(userId))
                    .flatMap(Set::stream)
                    .collect(Collectors.groupingBy(SettingInfo::getType));
    Stream<Map.Entry<String, List<SettingInfo>>> settingInfoStream = settingInfo.entrySet().stream();
    //大于1 使用并行处理
    if (settingInfo.size() > 1) {
        settingInfoStream = settingInfoStream.parallel();
    }
    return settingInfoStream
            .map(entry ->
                    createQuery()
                            // where type = ? and setting_for in (?,?,?....)
                            .where(type, entry.getKey())
                            .and()
                            .in(settingFor, entry.getValue().stream().map(SettingInfo::getSettingFor).collect(Collectors.toList()))
                            .listNoPaging())
            .flatMap(List::stream)
            .collect(Collectors.toList());
}
```

其中`AuthorizationSettingTypeSupplier`实例为`SimpleUserService`，所以调用的是`SimpleUserService`的`get`方法，`SimpleUserService`的`get`方法代码如下：

```java
public Set<SettingInfo> get(String userId) {
    UserEntity userEntity = selectByPk(userId);
    if (null == userEntity) {
        return new HashSet<>();
    }
    List<UserRoleEntity> roleEntities = userRoleDao.selectByUserId(userId);
    //使用角色配置
    Set<SettingInfo> settingInfo = roleEntities.stream()
            .map(entity -> new SettingInfo(SETTING_TYPE_ROLE, entity.getRoleId()))
            .collect(Collectors.toSet());
    //使用用户的配置
    settingInfo.add(new SettingInfo(SETTING_TYPE_USER, userId));
    return settingInfo;
}
```

逻辑是先通过`userId`查询记录，再通过`userId`查询表`s_user_role`获取角色实体`UserRoleEntity`，并转化为相应的`SettingInfo`实例后返回。

第5步的逻辑在方法`initPermission`中。

```java
private List<Permission> initPermission(List<AuthorizationSettingDetailEntity> detailList) {
    //权限id集合
    List<String> permissionIds = detailList.stream()
            .map(AuthorizationSettingDetailEntity::getPermissionId)
            .distinct()
            .collect(Collectors.toList());
    //权限信息缓存
    Map<String, PermissionEntity> permissionEntityCache =
            permissionService.selectByPk(permissionIds)
                    .stream()
                    .collect(Collectors.toMap(PermissionEntity::getId, Function.identity()));

    //防止越权
    detailList = detailList.stream().filter(detail -> {
        PermissionEntity entity = permissionEntityCache.get(detail.getPermissionId());
        if (entity == null || !STATUS_ENABLED.equals(entity.getStatus())) {
            return false;
        }
        List<String> allActions = entity.getActions().stream().map(ActionEntity::getAction).collect(Collectors.toList());

        if (isNotEmpty(entity.getActions()) && isNotEmpty(detail.getActions())) {

            detail.setActions(detail.getActions().stream().filter(allActions::contains).collect(Collectors.toSet()));
        }
        if (isEmpty(entity.getSupportDataAccessTypes())) {
            detail.setDataAccesses(new java.util.ArrayList<>());
        } else if (isNotEmpty(detail.getDataAccesses()) && !entity.getSupportDataAccessTypes().contains("*")) {
            //重构为权限支持的数据权限控制方式,防止越权设置权限
            detail.setDataAccesses(detail
                    .getDataAccesses()
                    .stream()
                    .filter(access ->
                            //以设置支持的权限开头就认为拥有该权限
                            //比如支持的权限为CUSTOM_SCOPE_ORG_SCOPE
                            //设置的权限为CUSTOM_SCOPE 则通过检验
                            entity.getSupportDataAccessTypes().stream()
                                    .anyMatch(type -> type.startsWith(access.getType())))
                    .collect(Collectors.toList()));
        }
        return true;
    }).collect(Collectors.toList());

    //全部权限设置
    Map<String, List<AuthorizationSettingDetailEntity>> settings = detailList
            .stream()
            .collect(Collectors.groupingBy(AuthorizationSettingDetailEntity::getPermissionId));

    List<Permission> permissions = new ArrayList<>();
    //获取关联的权限信息
    Map<String, List<ParentPermission>> parentsPermissions = permissionEntityCache.values().stream()
            .map(PermissionEntity::getParents)
            .filter(Objects::nonNull)
            .flatMap(Collection::stream)
            .collect(Collectors.groupingBy(ParentPermission::getPermission));

    settings.forEach((permissionId, details) -> {
        SimplePermission permission = new SimplePermission();
        permission.setId(permissionId);
        Set<String> actions = new HashSet<>();
        Set<DataAccessConfig> dataAccessConfigs = new HashSet<>();
        //排序,根据优先级进行排序
        Collections.sort(details);
        for (AuthorizationSettingDetailEntity detail : details) {
            //如果指定不合并相同的配置,则清空之前的配置
            if (Boolean.FALSE.equals(detail.getMerge())) {
                actions.clear();
                dataAccessConfigs.clear();
            }
            // actions
            if (null != detail.getActions()) {
                actions.addAll(detail.getActions());
            }
            // 数据权限控制配置
            if (null != detail.getDataAccesses()) {
                dataAccessConfigs.addAll(detail.getDataAccesses()
                        .stream()
                        .map(dataAccessFactory::create)
                        .collect(Collectors.toSet()));
            }
        }
        //是否有其他权限关联了此权限
        List<ParentPermission> parents = parentsPermissions.get(permissionId);
        if (parents != null) {
            actions.addAll(parents.stream()
                    .map(ParentPermission::getActions)
                    .filter(Objects::nonNull)
                    .flatMap(Collection::stream)
                    .collect(Collectors.toSet()));
            parentsPermissions.remove(permissionId);
        }
        permission.setActions(actions);
        permission.setDataAccesses(dataAccessConfigs);
        permissions.add(permission);
    });

    //关联权限
    parentsPermissions.forEach((per, all) -> {
        SimplePermission permission = new SimplePermission();
        permission.setId(per);
        permission.setActions(all.stream()
                .map(ParentPermission::getActions)
                .filter(Objects::nonNull)
                .flatMap(Collection::stream)
                .collect(Collectors.toSet()));
        permissions.add(permission);
    });
    return permissions;
}
```

逻辑如下：

1. 通过传入的`AuthorizationSettingDetailEntity`实体获取`permissionId`，根据`permissionId`从数据表`s_permission`中查询所有`PermissionEntity`。
2. 对每一个`AuthorizationSettingDetailEntity`实体的`permissionId`，查看是否有对应的`PermissionEntity`，如果找不到或者状态不是`STATUS_ENABLED`，则过滤掉
3. 如果`AuthorizationSettingDetailEntity`的`action`不为空并且`PermissionEntity`的`action`不为空，则将`AuthorizationSettingDetailEntity`的`action`设置为`PermissionEntity`的`action`中存在的`action`，因为`PermissionEntity`的`action`是包含了所有的`action`，这样就保证了`AuthorizationSettingDetailEntity`的`action`取值的正确性。
4. 判断是否全局`permssion`的`supportDataAccessTypes`是否为空，如果为空，则设置`DataAccessEntity`为空`list`，如果不为空，。。。
5. 获取`PermissionEntity`的父权限`ParentPermission`列表
6. 将`AuthorizationSettingDetailEntity`转化为`SimplePermission`对象，并移除权限id相同的父权限列表中对应的父权限
7. 将`ParentPermission`转化为`SimplePermission`对象
8. 返回所有的`SimplePermission`对象

现在我们再看`AuthorizationController`的`doLogin`方法在调用`authenticate`方法验证通过后发布的`AuthorizationSuccessEvent`事件，查找`AuthorizationSuccessEvent`事件后发现系统中有监听该事件的类：`UserOnSignIn`，在`hsweb-authorization-basic`模块中。

```java
@Override
public void onApplicationEvent(AuthorizationSuccessEvent event) {
    UserToken token = UserTokenHolder.currentToken();
    String tokenType = (String) event.getParameter("token_type").orElse(defaultTokenType);

    if (token != null) {
        //先退出已登陆的用户
        userTokenManager.signOutByToken(token.getToken());
    }
    //创建token
    GeneratedToken newToken = userTokenGenerators.stream()
            .filter(generator -> generator.getSupportTokenType().equals(tokenType))
            .findFirst()
            .orElseThrow(() -> new UnsupportedOperationException(tokenType))
            .generate(event.getAuthentication());
    //登入
    userTokenManager.signIn(newToken.getToken(), newToken.getType(), event.getAuthentication().getUser().getId(), newToken.getTimeout());

    //响应结果
    event.getResult().putAll(newToken.getResponse());

}
```

逻辑如下：

1. 获取参数`token_type`字段，默认为`sessionId`
2. 如果用户登录，则先退出
3. 通过`userTokenGenerators`的`generate`方法创建`token`
4. 调用`userTokenManager`的`signIn`登录

第三步中的`userTokenGenerators`中缓存着的`UserTokenGenerator`接口的实现的实例。默认有两个实现。先看`SessionIdUserTokenGenerator`。`SessionIdUserTokenGenerator`中`generate`方法其实就是使用`sessionId`作为`token`的，只是封装为`GeneratedToken`对象罢了。

第四步中`userTokenManager`接口实现为`DefaultUserTokenManager`类，看`signIn`方法

```java
public UserToken signIn(String token, String type, String userId, long maxInactiveInterval) {
    SimpleUserToken detail = new SimpleUserToken(userId, token);
    detail.setType(type);
    detail.setMaxInactiveInterval(maxInactiveInterval);
    AllopatricLoginMode mode = allopatricLoginModes.getOrDefault(type, allopatricLoginMode);
    if (mode == AllopatricLoginMode.deny) {
        boolean hasAnotherToken = getByUserId(userId)
                .stream()
                .filter(userToken -> type.equals(userToken.getType()))
                .map(SimpleUserToken.class::cast)
                .peek(this::checkTimeout)
                .anyMatch(UserToken::isNormal);
        if (hasAnotherToken) {
            throw new AccessDenyException("该用户已在其他地方登陆");
        }
    } else if (mode == AllopatricLoginMode.offlineOther) {
        //将在其他地方登录的用户设置为离线
        List<UserToken> oldToken = getByUserId(userId);
        for (UserToken userToken : oldToken) {
            //相同的tokenType才让其下线
            if (type.equals(userToken.getType())) {
                changeTokenState(userToken.getToken(), TokenState.offline);
            }
        }
    }
    detail.setState(TokenState.normal);
    tokenStorage.put(token, detail);

    getUserToken(userId).add(token);

    publishEvent(new UserTokenCreatedEvent(detail));
    return detail;
}
```

1. 构造`SimpleUserToken`类实例
2. 根据`type`获取`AllopatricLoginMode`类
3. 如果为`deny`，缓存中获取`userId`对应的`UserToken`，判断`type`是否相同，判断是否超时，判断状态是否为正常。如果都满足，则抛出`AccessDenyException`
4. 如果为`offlineOther`，缓存中获取`userId`对应的`UserToken`，如果`type`相同，设置状态为`offline`
5. 否则，设置状态为`normal`，缓存`token`，发布`UserTokenCreatedEvent`事件并返回

登录接口调用完成

## 退出流程

退出流程相对简单一些。也是在`hsweb-authorization-basic`项目的`AuthorizationController`类中定义。

直接发布`AuthorizationExitEvent`事件。同一个包中的类`UserOnSignOut`监听了该事件。从`UserTokenHolder.currentToken()`方法中获取到`token`字符串，并调用`userTokenManager.signOutByToken`方法

`DefaultUserTokenManager`类的`signOutByToken`方法只是将缓存中的`token`移除，并发布`UserTokenRemovedEvent`事件

有一个疑问：`UserTokenHolder.currentToken()`是如何设置进去的？

其实这个在[hs-web权限模块](hs-web-authorization.md)里已经提到了，就是用户令牌拦截器`WebUserTokenInterceptor`。每次接口调用都会被该拦截器拦截到

```java
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
    List<ParsedToken> tokens = userTokenParser.stream()
            .map(parser -> parser.parseToken(request))
            .filter(Objects::nonNull)
            .collect(Collectors.toList());

    if (tokens.isEmpty()) {
        if (enableBasicAuthorization && handler instanceof HandlerMethod) {
            HandlerMethod method = ((HandlerMethod) handler);
            AuthorizeDefinition definition = parser.parse(method.getBeanType(), method.getMethod());
            if (null != definition) {
                response.addHeader("WWW-Authenticate", " Basic realm=\"\"");
            }
        }
        return true;
    }
    for (ParsedToken parsedToken : tokens) {
        UserToken userToken = null;
        String token = parsedToken.getToken();
        if (userTokenManager.tokenIsLoggedIn(token)) {
            userToken = userTokenManager.getByToken(token);
        }
        if ((userToken == null || userToken.isExpired()) && parsedToken instanceof AuthorizedToken) {
            //先踢出旧token
            userTokenManager.signOutByToken(token);

            userToken = userTokenManager
                    .signIn(parsedToken.getToken(), parsedToken.getType(), ((AuthorizedToken) parsedToken).getUserId(), ((AuthorizedToken) parsedToken).getMaxInactiveInterval());
        }
        if (null != userToken) {
            userTokenManager.touch(token);
            UserTokenHolder.setCurrent(userToken);
        }
    }
    return true;
}
```

1. 根据请求解析出`token`
2. 如果`token`不为空，且支持基本鉴权验证，且为`HandlerMethod`方法实例，则解析方法的注解，解析出则添加`response`的相应`header`（这步没看懂）
3. 循环解析出的`token`，如果`token`已登录，则获取到`token`，调用`getByToken`方法时已经检查了`token`是否过期。
4. 如果`token`为空或者`token`过期，并且`token`为`AuthorizedToken`，则先调用退出的方法来踢出旧`token`，再用新`token`登录
5. 如果`token`不为空，则调用`DefaultUserTokenManager`的`touch`方法，在`touch`方法中，会从缓存中取出`SimpleUserToken`，并调用`SimpleUserToken`的`touch`方法，其实就是请求次数加1，并且更新最后请求时间。然后再调用`syncToken`方法。目前该方法为空，注释为如果使用`redission`，可重写此方法
6. 调用`UserTokenHolder.setCurrent`设置`token`
