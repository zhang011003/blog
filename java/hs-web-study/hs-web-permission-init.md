# hs-web中权限数据初始化

之前看的hs-web中demo里的`hsweb-starter.js`脚本有初始化系统菜单表（s_menu）、权限设置表（s_autz_setting）、权限设置菜单表（s_autz_menu）、用户表（s_user）中的数据，但是对权限表（s_permission）却没有初始化的脚本，但启动后在权限表中却有数据，具体是在哪里初始化的呢？

其实之前在分析`AopAuthorizingController`类时就已经有了，只不过没有注意。

`AopAuthorizingController`类的`matches`方法

```java
    public boolean matches(Method method, Class<?> aClass) {
        boolean support = AopUtils.findAnnotation(aClass, Controller.class) != null
                || AopUtils.findAnnotation(aClass, RestController.class) != null
                || AopUtils.findAnnotation(aClass, method, Authorize.class) != null;

        if (support && autoParse) {
            defaultParser.parse(aClass, method);
        }
        return support;
    }
```

其中的`autoParse`变量是在声明的时候注入的

`AopAuthorizeAutoConfiguration`中对`AopAuthorizingController`的声明部分

```java
    @Bean
    @ConfigurationProperties(prefix = "hsweb.authorize")
    public AopAuthorizingController aopAuthorizingController(AuthorizingHandler authorizingHandler,
                                                             AopMethodAuthorizeDefinitionParser aopMethodAuthorizeDefinitionParser) {

        return  new AopAuthorizingController(authorizingHandler, aopMethodAuthorizeDefinitionParser);
    }
```

`ConfigurationProperties`注解的注释如下：

```javadoc
Annotation for externalized configuration. Add this to a class definition or a @Bean method in a @Configuration class if you want to bind and validate some external Properties (e.g. from a .properties file).
Note that contrary to @Value, SpEL expressions are not evaluated since property values are externalized.
```

大概意思是如果想要绑定和验证一些外部属性文件，则将该注解加到注解了` @Configuration`的类定义或`@Bean`方法上。

再说`matches`的方法中调用的`parse`方法

```java
    public AuthorizeDefinition parse(Class target, Method method, MethodInterceptorContext context) {

        if (excludeMethodName.contains(method.getName())) {
            return null;
        }
        CacheKey key = buildCacheKey(target, method);

        AuthorizeDefinition definition = cache.get(key);
        if (definition instanceof EmptyAuthorizeDefinition) {
            return null;
        }
        if (null != definition) {
            return definition;
        }
        //使用自定义
        if (!CollectionUtils.isEmpty(parserCustomizers)) {
            definition = parserCustomizers.stream()
                    .map(customizer -> customizer.parse(target, method, context))
                    .filter(Objects::nonNull)
                    .findAny().orElse(null);
            if (definition instanceof EmptyAuthorizeDefinition) {
                return null;
            }
            if (definition != null) {
                return definition;
            }
        }
        Authorize classAuth = AopUtils.findAnnotation(target, Authorize.class);
        Authorize methodAuth = AopUtils.findMethodAnnotation(target, method, Authorize.class);

        RequiresDataAccess classDataAccess = AopUtils.findAnnotation(target, RequiresDataAccess.class);

        RequiresDataAccess methodDataAccess = AopUtils.findMethodAnnotation(target, method, RequiresDataAccess.class);

        RequiresExpression expression = AopUtils.findAnnotation(target, RequiresExpression.class);

        if (classAuth == null && methodAuth == null && classDataAccess == null && methodDataAccess == null && expression == null) {
            cache.put(key, EmptyAuthorizeDefinition.instance);
            return null;
        }

        if ((methodAuth != null && methodAuth.ignore()) || (classAuth != null && classAuth.ignore())) {
            cache.put(key, EmptyAuthorizeDefinition.instance);
            return null;
        }
        synchronized (cache) {
            DefaultBasicAuthorizeDefinition authorizeDefinition = new DefaultBasicAuthorizeDefinition();
            authorizeDefinition.setTargetClass(target);
            authorizeDefinition.setTargetMethod(method);
            if (methodAuth == null || methodAuth.merge()) {
                authorizeDefinition.put(classAuth);
            }

            authorizeDefinition.put(methodAuth);

            authorizeDefinition.put(expression);

            authorizeDefinition.put(classDataAccess);

            authorizeDefinition.put(methodDataAccess);

            if (authorizeDefinition.getPermissionDescription().length == 0) {
                if (classAuth != null) {
                    authorizeDefinition.put(classAuth.dataAccess());
                    String[] desc = classAuth.description();
                    if (desc.length > 0) {
                        authorizeDefinition.setPermissionDescription(desc);
                    }
                }
            }

            if (authorizeDefinition.getActionDescription().length == 0) {
                if (methodAuth != null) {
                    if (methodAuth.description().length != 0) {
                        authorizeDefinition.setActionDescription(methodAuth.description());
                    }
                }
            }

            log.info("parsed authorizeDefinition {}.{} => {}.{} permission:{} actions:{}",
                    target.getSimpleName(),
                    method.getName(),
                    authorizeDefinition.getPermissionDescription(),
                    authorizeDefinition.getActionDescription(),
                    authorizeDefinition.getPermissions(),
                    authorizeDefinition.getActions());
            cache.put(key, authorizeDefinition);
            return authorizeDefinition;
        }
    }
```

1. 如果缓存中存在，则直接返回，避免重复解析

2. 判断是否有自定义的`AopMethodAuthorizeDefinitionCustomizerParser`类，如果有，则使用自定义的类解析并返回`AuthorizeDefinition`

3. 获取类的`Authorize`注解、方法的`Authorize`注解、类的`RequiresDataAccess`注解、方法的`RequiresDataAccess`注解、类的`RequiresExpression`注解。如果都没有，则默认在缓存中保存`EmptyAuthorizeDefinition`。如果方法有注解，但`ignore`为true或者类有注解，但`ignore`为true，也在缓存中保存`EmptyAuthorizeDefinition`

4. 同步`cache`，设置`DefaultBasicAuthorizeDefinition`实例`authorizeDefinition`的属性。多次调用`DefaultBasicAuthorizeDefinition`的`put`方法，只是在类鉴权注解时只有在方法鉴权注解为空或者`merge`为true（需要合并类的注解信息）时才会调用。

5. 如果`DefaultBasicAuthorizeDefinition`的`permissionDescription`为空，如果类权限注解不为空，则调用`DefaultBasicAuthorizeDefinition`的`put`方法，传入类注解的`dataAccess`值。如果类注解的`description`为空，则设置`DefaultBasicAuthorizeDefinition`的`permissionDescription`

6. 如果`DefaultBasicAuthorizeDefinition`的`actionDescription`为空，如果方法权限注解不为空，则设置`DefaultBasicAuthorizeDefinition`的`actionDescription`

7. 加入缓存后返回`authorizeDefinition`

`DefaultBasicAuthorizeDefinition`类的`put`方法代码如下，其实就是解析不同的注解，将对应的属性保存下来

```java
    public void put(Authorize authorize) {
        if (null == authorize || authorize.ignore()) {
            return;
        }
        permissions.addAll(Arrays.asList(authorize.permission()));
        actions.addAll(Arrays.asList(authorize.action()));
        roles.addAll(Arrays.asList(authorize.role()));
        user.addAll(Arrays.asList(authorize.user()));
        if (authorize.logical() != Logical.DEFAULT) {
            logical = authorize.logical();
        }
        message = authorize.message();
        phased = authorize.phased();
        put(authorize.dataAccess());
    }

    public void put(RequiresExpression expression) {
        if (null == expression) {
            return;
        }
        script = new DefaultScript(expression.language(), expression.value());
    }

    public void put(RequiresDataAccess dataAccess) {
        if (null == dataAccess || dataAccess.ignore()) {
            return;
        }
        if (!"".equals(dataAccess.permission())) {
            permissions.add(dataAccess.permission());
        }
        actions.addAll(Arrays.asList(dataAccess.action()));
        DefaultDataAccessDefinition definition = new DefaultDataAccessDefinition();
        definition.setEntityType(dataAccess.entityType());
        definition.setPhased(dataAccess.phased());
        if (!"".equals(dataAccess.controllerBeanName())) {
            definition.setController(dataAccess.controllerBeanName());
        } else if (DataAccessController.class != dataAccess.controllerClass()) {
            definition.setController(dataAccess.getClass().getName());
        }
        dataAccessDefinition = definition;
        dataAccessControl = true;
    }
```

完成对`Authorize`注解的解析后，在`AopAuthorizingController`的`run`方法中发布`AuthorizeDefinitionInitializedEvent`事件，将解析得到的`AuthorizeDefinition`作为参数传入。

`AutoSyncPermission`类监听了该事件，如下是代码

```java
    public void onApplicationEvent(AuthorizeDefinitionInitializedEvent event) {
        List<AuthorizeDefinition> definitions = event.getAllDefinition();

        Map<String, List<AuthorizeDefinition>> grouping = new HashMap<>();

        //以permissionId分组
        for (AuthorizeDefinition definition : definitions) {
            for (String permissionId : definition.getPermissions()) {
                grouping.computeIfAbsent(permissionId, id -> new ArrayList<>())
                        .add(definition);
            }
        }

        //创建权限实体
        Map<String, PermissionEntity> waitToSyncPermissions = new HashMap<>();
        for (Map.Entry<String, List<AuthorizeDefinition>> permissionDefinition : grouping.entrySet()) {
            String permissionId = permissionDefinition.getKey();
            //一个权限的全部定义(一个permission多个action)
            List<AuthorizeDefinition> allDefinition = permissionDefinition.getValue();
            if (allDefinition.isEmpty()) {
                return;
            }
            AuthorizeDefinition tmp = allDefinition.get(0);
            //action描述
            List<String> actionDescription = allDefinition.stream()
                    .map(AuthorizeDefinition::getActionDescription)
                    .flatMap(Stream::of)
                    .collect(Collectors.toList());
            //action
            List<String> actions = allDefinition
                    .stream()
                    .map(AuthorizeDefinition::getActions)
                    .flatMap(Collection::stream)
                    .collect(Collectors.toList());

            //创建action实体
            Set<ActionEntity> actionEntities = new HashSet<>(actions.size());
            if (!actions.isEmpty()) {
                for (int i = 0; i < actions.size(); i++) {
                    String action = actions.get(i);
                    String desc = actionDescription.size() > i ? actionDescription.get(i) : actionDescMapping.getOrDefault(actions.get(i), action);
                    ActionEntity actionEntity = new ActionEntity();
                    actionEntity.setAction(action);
                    actionEntity.setDescribe(desc);
                    actionEntities.add(actionEntity);
                }
            }
            //创建permission
            PermissionEntity entity = entityFactory.newInstance(PermissionEntity.class);
            if (tmp instanceof AopAuthorizeDefinition) {
                AopAuthorizeDefinition aopAuthorizeDefinition = ((AopAuthorizeDefinition) tmp);
                Class type = aopAuthorizeDefinition.getTargetClass();
                Class genType = entityFactory.getInstanceType(ClassUtils.getGenericType(type));
                List<OptionalField> optionalFields = new ArrayList<>();
                entity.setOptionalFields(optionalFields);
                if (genType != Object.class) {
                    List<Field> fields = new ArrayList<>();

                    ReflectionUtils.doWithFields(genType, fields::add, field -> (field.getModifiers() & Modifier.STATIC) == 0);

                    for (Field field : fields) {
                        if ("id".equals(field.getName())) {
                            continue;
                        }
                        ApiModelProperty property = field.getAnnotation(ApiModelProperty.class);
                        OptionalField optionalField = new OptionalField();
                        optionalField.setName(field.getName());
                        if (null != property) {
                            if (property.hidden()) {
                                continue;
                            }
                            optionalField.setDescribe(property.value());
                        }
                        optionalFields.add(optionalField);
                    }
                }
            }
            entity.setId(permissionId);
            entity.setName(tmp.getPermissionDescription().length > 0 ? tmp.getPermissionDescription()[0] : permissionId);
            entity.setActions(new ArrayList<>(actionEntities));
            entity.setType("default");
            entity.setStatus(DataStatus.STATUS_ENABLED);
            waitToSyncPermissions.putIfAbsent(entity.getId(), entity);
        }

        //查询出全部旧的权限数据并载入缓存
        Map<String, PermissionEntity> oldCache = permissionService
                .select()
                .stream()
                .collect(Collectors.toMap(PermissionEntity::getId, Function.identity()));

        waitToSyncPermissions.forEach((permissionId, permission) -> {
            log.info("try sync permission[{}].{}", permissionId, permission.getActions());
            PermissionEntity oldPermission = oldCache.get(permissionId);
            if (oldPermission == null) {
                permissionService.insert(permission);
            } else {
                Set<ActionEntity> oldAction = new HashSet<>();
                if (oldPermission.getActions() != null) {
                    oldAction.addAll(oldPermission.getActions());
                }
                Map<String, ActionEntity> actionCache = oldAction
                        .stream()
                        .collect(Collectors.toMap(ActionEntity::getAction, Function.identity()));
                boolean permissionChanged = false;
                for (ActionEntity actionEntity : permission.getActions()) {
                    //添加新的action到旧的action
                    if (actionCache.get(actionEntity.getAction()) == null) {
                        oldAction.add(actionEntity);
                        permissionChanged = true;
                    }
                }
                if (permissionChanged) {
                    oldPermission.setActions(new ArrayList<>(oldAction));
                    permissionService.updateByPk(oldPermission.getId(), oldPermission);
                }
                actionCache.clear();
            }


        });

        oldCache.clear();
        waitToSyncPermissions.clear();
        definitions.clear();
        grouping.clear();
    }
```

1. 将所有之前解析出的`AuthorizeDefinition`以`permissionId`分组

2. 循环分组

3. 获取每个`permissionId`对应的所有action（因为一个`permissionId`对应多个`action`），获取所有action描述，获取所有action。循环action创建`ActionEntity`并保存

4. 使用`entityFactory`的 `newInstance`方法创建`PermissionEntity`的实例。因为`PermissionEntity`是接口，所以默认会创建`SimplePermissionEntity`类的实例。

5. 判断第一个`AuthorizeDefinition`，如果为`AopAuthorizeDefinition`（默认都是），则获取`targetClass`（也就是各个controller）的第一个泛型类（即实体类型，参见[hs-web中的controller结构](hs-web-controller.md)）

6. 如果不是`Object`类，则通过反射获取该类所有的非static字段。循环这些字段，如果非`id`，则获取字段的`ApiModelProperty`注解并构造`OptionalField`，并将`OptionalField`设置到`PermissionEntity`的实例中

7. 设置`PermissionEntity`实例的其它属性

8. 将对象保存到以`permissionId`为key，`PermissionEntity`为value的`HashMap`中，实例名为`waitToSyncPermissions`

9. 将全部之前数据库中的权限数据查出，并以`permissionId`为key进行分组，保存在`Map`的实例`oldCache`中

10. 循环之前设置的`waitToSyncPermissions`

11. 如果`oldCache`中不存在以`permissionId`为key的值，则说明是新增的权限，直接插入数据库

12. 否则，判断旧的action中是否存在新的`permission`中对应的action。如果不存在，则将不存在的action增加到旧的action列表中，并设置`permissionChanged`为true。

13. 如果`permissionChanged`为true，则更新permission对应的记录

14. 完成后清空对应的缓存
