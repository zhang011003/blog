# hs-web登录

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


