# hs-web登录和退出

## hs-web登录启动

在main方法所在类增加@EnableAopAuthorize注解启动登录功能，不过貌似如果去掉对应的注解，还是会调用登录的接口。所以登录是必须的。

## 登录流程

登录接口在hsweb-authorization-basic项目的AuthorizationController类中定义。

在doLogin方法中，先发布AuthorizationDecodeEvent事件，再发布AuthorizationBeforeEvent事件，再调用到SimpleAuthenticationManager的authenticate方法，如果没抛出异常，则发布AuthorizationSuccessEvent事件，如果抛出异常，发布AuthorizationFailedEvent事件。

在SimpleAuthenticationManager的authenticate方法中，先到数据表s_user中查询对应的用户名是否存在。如果存在，则在getByUserId方法中通过查询出的id构造Authentication对象并返回。

后续的操作都是获取用户的权限数据，需要了解权限表设计。[数据表设计](hs-web-db.md)

getByUserId方法先判断是否存在key为userId的缓存，如果不存在，则调用SimpleAuthorizationSettingService类的initUserAuthorization方法来获取，之后存入缓存。

如下是initUserAuthorization方法代码。

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
1. 通过userId查询表s_user获取UserEntity对象，实例化SimpleAuthentication类并设置user为SimpleUser，
2. 通过userId查询表s_user_role获取RoleEntity对象，并设置role为SimpleRole，
3. 通过userId调用getUserSetting方法获取AuthorizationSettingEntity，
4. 取出AuthorizationSettingEntity的id，并查询s_autz_detail表获取setting_id匹配的AuthorizationSettingDetailEntity，
5. 调用initPermission将AuthorizationSettingDetailEntity转化为Permission。

前两步较简单。

第3步中，getUserSetting方法中，循环调用authorizationSettingTypeSuppliers中的get方法，然后通过SettingInfo的type分组，如果有多个分组，则使用并行调用，再查询表s_autz_setting获取type和settingFor匹配的结果。

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

其中AuthorizationSettingTypeSupplier实例为SimpleUserService，所以调用的是SimpleUserService的get方法，SimpleUserService的get方法代码如下：

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

逻辑是先通过userId查询记录，再通过userId查询表s_user_role获取角色实体UserRoleEntity，并转化为相应的SettingInfo实例后返回。

第5步的逻辑在方法initPermission中。

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
1. 通过传入的AuthorizationSettingDetailEntity实体获取permissionId，根据permissionId从数据表s_permission中查询所有PermissionEntity。
2. 对每一个AuthorizationSettingDetailEntity实体的permissionId，查看是否有对应的PermissionEntity，如果找不到或者状态不是STATUS_ENABLED，则过滤掉
3. 如果AuthorizationSettingDetailEntity的action不为空并且PermissionEntity的action不为空，则将AuthorizationSettingDetailEntity的action设置为PermissionEntity的action中存在的action，因为PermissionEntity的action是包含了所有的action，这样就保证了AuthorizationSettingDetailEntity的action取值的正确性。
4. 判断是否全局permssion的supportDataAccessTypes是否为空，如果为空，则设置DataAccessEntity为空list，如果不为空，。。。
5. 获取PermissionEntity的父权限ParentPermission列表
6. 将AuthorizationSettingDetailEntity转化为SimplePermission对象，并移除权限id相同的父权限列表中对应的父权限
6. 将ParentPermission转化为SimplePermission对象
7. 返回所有的SimplePermission对象

现在我们再看AuthorizationController的doLogin方法在调用authenticate方法验证通过后发布的AuthorizationSuccessEvent事件，查找AuthorizationSuccessEvent事件后发现系统中有监听该事件的类：UserOnSignIn，在hsweb-authorization-basic模块中。

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
1. 获取参数token_type字段，默认为sessionId
2. 如果用户登录，则先退出
3. 通过userTokenGenerators的generate方法创建token
4. 调用userTokenManager的signIn登录

第三步中的userTokenGenerators中缓存着的UserTokenGenerator接口的实现的实例。默认有两个实现。先看SessionIdUserTokenGenerator。SessionIdUserTokenGenerator中generate方法其实就是使用sessionId作为token的，只是封装为GeneratedToken对象罢了。

第四步中userTokenManager接口实现为DefaultUserTokenManager类，看signIn方法

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

1. 构造SimpleUserToken类实例
2. 根据type获取AllopatricLoginMode类
3. 如果为deny，缓存中获取userId对应的UserToken，判断type是否相同，判断是否超时，判断状态是否为正常。如果都满足，则抛出AccessDenyException
4. 如果为offlineOther，缓存中获取userId对应的UserToken，如果type相同，设置状态为offline
5. 否则，设置状态为normal，缓存token，发布UserTokenCreatedEvent事件并返回

登录接口调用完成

## 退出流程

退出流程相对简单一些。也是在hsweb-authorization-basic项目的AuthorizationController类中定义。

直接发布AuthorizationExitEvent事件。同一个包中的类UserOnSignOut监听了该事件。从UserTokenHolder.currentToken()方法中获取到token字符串，并调用userTokenManager.signOutByToken方法

DefaultUserTokenManager类的signOutByToken方法只是将缓存中的token移除，并发布UserTokenRemovedEvent事件

有一个疑问：UserTokenHolder.currentToken()是如何设置进去的？

其实这个在[hs-web权限模块](hs-web-authorization.md)里已经提到了，就是用户令牌拦截器WebUserTokenInterceptor。每次接口调用都会被该拦截器拦截到

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

1. 根据请求解析出token
2. 如果token不为空，且支持基本鉴权验证，且为HandlerMethod方法实例，则解析方法的注解，解析出则添加response的相应header（这步没看懂）
3. 循环解析出的token，如果token已登录，则获取到token，调用getByToken方法时已经检查了token是否过期。
4. 如果token为空或者token过期，并且token为AuthorizedToken，则先调用退出的方法来踢出旧token，再用新token登录
5. 如果token不为空，则调用DefaultUserTokenManager的touch方法，在touch方法中，会从缓存中取出SimpleUserToken，并调用SimpleUserToken的touch方法，其实就是请求次数加1，并且更新最后请求时间。然后再调用syncToken方法。目前该方法为空，注释为如果使用redission，可重写此方法
6. 调用UserTokenHolder.setCurrent设置token


