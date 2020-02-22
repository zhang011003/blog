# hs-web角色权限和数据权限验证逻辑

角色权限和数据权限验证逻辑都在`DefaultAuthorizingHandler`类中，其中`handRBAC`方法是验证角色权限，`handleDataAccess`方法是验证数据权限。

## 角色权限验证逻辑

1. 发布`AuthorizingHandleBeforeEvent`事件，如果不需要进行权限验证，则`isAllow`返回`true`。主要是针对一些特殊用户来取消权限验证，比如admin，demo中启动类就监听了该接口并设置`allow`为`true`。`UserAllowPermissionHandler`类也监听了该事件，这样可以通过配置来直接过滤哪些用户或者角色不需要进行权限的过滤而直接访问。不过目前配置只支持rbac权限过滤，不支持数据权限过滤
2. 进行rbac权限验证
3. 进行表达式权限验证

如下是第二步中rbac权限验证代码

```java
protected void handleRBAC(Authentication authentication, AuthorizeDefinition definition) {
    boolean access = true;
    //多个设置时的判断逻辑
    Logical logical = definition.getLogical() == Logical.DEFAULT ? Logical.OR : definition.getLogical();
    boolean logicalIsOr = logical == Logical.OR;

    Set<String> permissionsDef = definition.getPermissions();
    Set<String> actionsDef = definition.getActions();
    Set<String> rolesDef = definition.getRoles();
    Set<String> usersDef = definition.getUser();


    // 控制权限
    if (!definition.getPermissions().isEmpty()) {
        if (logger.isInfoEnabled()) {
            logger.info("do permission access handle : permissions{}({}),actions{} ,definition:{}.{} ({})",
                    definition.getPermissionDescription(),
                    permissionsDef, actionsDef
                    , definition.getPermissions(),
                    definition.getActions(),
                    definition.getLogical());
        }
        List<Permission> permissions = authentication.getPermissions().stream()
                .filter(permission -> {
                    // 未持有任何一个权限
                    if (!permissionsDef.contains(permission.getId())) {
                        return false;
                    }
                    //未配置action
                    if (actionsDef.isEmpty()) {
                        return true;
                    }
                    //判断action
                    List<String> actions = permission.getActions()
                            .stream()
                            .filter(actionsDef::contains)
                            .collect(Collectors.toList());

                    if (actions.isEmpty()) {
                        return false;
                    }

                    //如果 控制逻辑是or,则只要过滤结果数量不为0.否则过滤结果数量必须和配置的数量相同
                    return logicalIsOr || permission.getActions().containsAll(actions);
                }).collect(Collectors.toList());
        access = logicalIsOr ?
                CollectionUtils.isNotEmpty(permissions) :
                //权限数量和配置的数量相同
                permissions.size() == permissionsDef.size();
    }
    //控制角色
    if (!rolesDef.isEmpty()) {
        if (logger.isInfoEnabled()) {
            logger.info("do role access handle : roles{} , definition:{}", rolesDef, definition.getRoles());
        }
        Function<Predicate<Role>, Boolean> func = logicalIsOr
                ? authentication.getRoles().stream()::anyMatch
                : authentication.getRoles().stream()::allMatch;
        access = func.apply(role -> rolesDef.contains(role.getId()));
    }
    //控制用户
    if (!usersDef.isEmpty()) {
        if (logger.isInfoEnabled()) {
            logger.info("do user access handle : users{} , definition:{} ", usersDef, definition.getUser());
        }
        Function<Predicate<String>, Boolean> func = logicalIsOr
                ? usersDef.stream()::anyMatch
                : usersDef.stream()::allMatch;
        access = func.apply(authentication.getUser().getUsername()::equals);
    }
    if (!access) {
        throw new AccessDenyException(definition.getMessage());
    }
}
```

1. 如果定义的`permission`不为空，通过如下逻辑过滤当前用户的`permission`
   1. 方法上定义的`permission`不包含当前用户的`permission`，说明没有权限，返回`false`
   2. 定义的`action`没有配置，返回`true`
   3. 定义的`action`不包含当前用户的`action`，说明没有操作权限，返回`false`
   4. 如果控制逻辑为`or`，则返回`true`；否则控制逻辑为`and`，需要当前用户的`action`包含所有定义的`action`
2. 如果控制逻辑为`or`，则只要返回的当前用户的`permission`不为空就可以访问，否则需要返回的当前用户的`permission`和定义的`permission`数量一致才可以访问。也就是说，当前用户的`permission`要全部包含定义的`permission`
3. 如果定义的`role`不为空（角色名称匹配），如果控制逻辑为`or`，则任意定义的`role`包含当前用户的`role`即可，否则，定义的`role`都需要包含当前用户的`role`
4. 如果定义的`user`不为空（用户名称匹配），如果控制逻辑为`or`，则任意定义的`user`包含当前用户的`user`即可，否则，定义的`user`都需要包含当前用户的`user`

如果验证通过，则返回，否则抛出`AccessDenyException`异常

如下是第三步中表达式权限验证代码

```java
protected void handleExpression(Authentication authentication, AuthorizeDefinition definition, MethodInterceptorContext paramContext) {
    if (definition.getScript() != null) {
        String scriptId = DigestUtils.md5Hex(definition.getScript().getScript());

        DynamicScriptEngine engine = DynamicScriptEngineFactory.getEngine(definition.getScript().getLanguage());
        if (null == engine) {
            throw new AccessDenyException("{unknown_engine}:" + definition.getScript().getLanguage());
        }
        if (!engine.compiled(scriptId)) {
            try {
                engine.compile(scriptId, definition.getScript().getScript());
            } catch (Exception e) {
                logger.error("express compile error", e);
                throw new AccessDenyException("{expression_error}");
            }
        }
        Map<String, Object> var = new HashMap<>(paramContext.getParams());
        var.put("auth", authentication);
        Object success = engine.execute(scriptId, var).get();
        if (!(success instanceof Boolean) || !((Boolean) success)) {
            throw new AccessDenyException(definition.getMessage());
        }
    }
}
```

获取配置的script，动态脚步引擎编译，将当前用户的authentication放在auth变量中传入，判断执行结果，如果为true，则验证通过，否则抛出AccessDenyException异常

动态脚本验证权限给用户更多的逻辑控制，实现起来也会更加的复杂。在hs-web的代码中目前没有找到具体的示例。

## 数据权限验证逻辑

如下是数据权限验证代码

```java
public void handleDataAccess(AuthorizingContext context) {

    if (dataAccessController == null) {
        logger.warn("dataAccessController is null,skip result access control!");
        return;
    }
    if (context.getDefinition().getDataAccessDefinition() == null) {
        return;
    }
    if (handleEvent(context, HandleType.DATA)) {
        return;
    }

    List<Permission> permission = context.getAuthentication().getPermissions()
            .stream()
            .filter(per -> context.getDefinition().getPermissions().contains(per.getId()))
            .collect(Collectors.toList());

    DataAccessController finalAccessController = dataAccessController;

    //取得当前登录用户持有的控制规则
    Set<DataAccessConfig> accesses = permission
            .stream().map(Permission::getDataAccesses)
            .flatMap(Collection::stream)
            .filter(access -> context.getDefinition().getActions().contains(access.getAction()))
            .collect(Collectors.toSet());
    //无规则,则代表不进行控制
    if (accesses.isEmpty()) {
        return;
    }
    //单个规则验证函数
    Function<Predicate<DataAccessConfig>, Boolean> function = accesses.stream()::allMatch;
    //调用控制器进行验证
    boolean isAccess = function.apply(access -> finalAccessController.doAccess(access, context));
    if (!isAccess) {
        throw new AccessDenyException(context.getDefinition().getMessage());
    }

}
```

1. `dataAccessController`如果为空，则直接返回。默认的`dataAccessController`为`DefaultDataAccessController`
2. 没有定义数据访问权限，则直接返回
3. 发布`AuthorizingHandleBeforeEvent`事件，同rbac权限第一步一致
4. 获取定义的`permission`包含当前用户`permission`的所有`permission`
5. 根据过滤出的`permission`获取对应的`action`，并获取定义的`action`包含当前用户`action`的所有`action`
