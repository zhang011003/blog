# hs-web中的数据库初始化

hs-web中的数据库初始化跟平常见过的数据库初始化都不太一样。平常数据库初始化使用`flyway`或`liquibase`。但hs-web中的数据库初始化是通过自定义的js脚本来实现的。

## SystemInitializeAutoConfiguration类

`SystemInitializeAutoConfiguration`类为主要的数据库初始化类。该类做的事情有：

1. 在依赖注入完成后，获取js引擎，增加`logger`、`sqlExecutor`和`spring`三个全局变量。
   
   ```java
       @PostConstruct
       public void init() {
           engines = Stream.of("js", "groovy")
                   .map(DynamicScriptEngineFactory::getEngine)
                   .filter(Objects::nonNull)
                   .collect(Collectors.toList());
   
           addGlobalVariable("logger", LoggerFactory.getLogger("org.hswebframework.script"));
           addGlobalVariable("sqlExecutor", sqlExecutor);
           addGlobalVariable("spring", applicationContext);
       }
   
       @SuppressWarnings("all")
       protected void addGlobalVariable(String var, Object val) {
           engines.forEach(engine -> {
                       try {
                           engine.addGlobalVariable(Collections.singletonMap(var, val));
                       } catch (NullPointerException ignore) {
                       }
                   }
           );
       }
   ```

2. 获取当前数据库类型
   
   由于继承了`org.springframework.boot.CommandLineRunner`，所以会自动执行`run`方法。在`run`方法中首先判断如果`autoInit`为false，则直接退出执行，否则获取当前的数据库类型。查看代码，发现当前的数据库类型并没有设置的地方，但却直接判断如果为空，则会抛出异常。
   
   ```java
       /**
        * 动态数据源服务
        */
       static volatile DynamicDataSourceService dynamicDataSourceService;    
       /**
        * @return 当前使用的数据源
        */
       public static DynamicDataSource currentDataSource() {
           String id = dataSourceSwitcher.currentDataSourceId();
           if (id == null) {
               return defaultDataSource();
           }
           checkDynamicDataSourceReady();
           return dynamicDataSourceService.getDataSource(id);
       }
      /**
   
        * @return 默认数据源
        */
       public static DynamicDataSource defaultDataSource() {
           checkDynamicDataSourceReady();
           return dynamicDataSourceService.getDefaultDataSource();
       }
       public static void checkDynamicDataSourceReady() {
           if (dynamicDataSourceService == null) {
               throw new UnsupportedOperationException("dataSourceService not ready");
           }
       }
   ```
   
   查找代码，发现`org.hswebframework.web.datasource.DynamicDataSourceAutoConfiguration`中有两处设置的地方
   
   ```java
   @Bean
       public BeanPostProcessor switcherInitProcessor() {
           return new BeanPostProcessor() {
               @Override
               public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
                   return bean;
               }
   
               @Override
               public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
                   if (bean instanceof DynamicDataSourceService) {
                       DataSourceHolder.dynamicDataSourceService = ((DynamicDataSourceService) bean);
                   }
                   if (bean instanceof DataSourceSwitcher) {
                       DataSourceHolder.dataSourceSwitcher = ((DataSourceSwitcher) bean);
                   }
                   if (bean instanceof TableSwitcher) {
                       DataSourceHolder.tableSwitcher = ((TableSwitcher) bean);
                   }
                   if (bean instanceof DatabaseSwitcher) {
                       DataSourceHolder.databaseSwitcher = ((DatabaseSwitcher) bean);
                   }
                   return bean;
               }
           };
       }
   
       @Configuration
       public static class AutoRegisterDataSource {
           @Autowired
           public void setDataSourceService(DynamicDataSourceService dataSourceService) {
               DataSourceHolder.dynamicDataSourceService = dataSourceService;
           }
       }
   ```
   
   查找`DynamicDataSourceService`的实现类有四个，其中`InDBDynamicDataSourceService`和`InDBJtaDynamicDataSourceService`是定义在`org.hswebframework.web.datasource.starter.InDBDynamicDataSourceAutoConfiguration`中，而`InDBDynamicDataSourceAutoConfiguration`类是在spring.factories中配置，而它们又是在hsweb-system-datasource-starter模块中。
   
   而`InSpringContextDynamicDataSourceService`是在`org.hswebframework.web.datasource.DynamicDataSourceAutoConfiguration`类中定义，是在hsweb-datasource-api模块中
   
   剩下的`JtaDynamicDataSourceService`是在`org.hswebframework.web.datasource.jta.AtomikosDataSourceAutoConfiguration`中定义，是在hsweb-datasource-jta模块中
   
   可以看到，如果引入了hsweb-datasource-jta模块，则使用`JtaDynamicDataSourceService`，如果引入了hsweb-system-datasource-starter模块，则使用`InDBDynamicDataSourceService`，如果都没有引入，则使用`InSpringContextDynamicDataSourceService`，而`InDBJtaDynamicDataSourceService`是在`JtaDynamicDataSourceService`存在的情况下才会引入，目前还不清楚为何在`JtaDynamicDataSourceService`存在的情况下要引入两个不同的`DynamicDataSourceService`。
   
   从上面的分析可以看出，系统中默认使用的`dynamicDataSourceService`为`InDBDynamicDataSourceService`

3. 初始化数据库元数据
   
   获取到当前数据库类型后，会根据具体类型初始化`RDBDatabaseMetaData`，以及设置对应的`TableMetaParser`，之后会调用`init`方法进行初始化。查看`init`方法，发现`init`方法主要是将不同类型的`SqlRender`都保存起来，供后续自动生成对应的sql时用到。接着构造`SimpleDatabase`，并发布`SystemInitializeEvent`事件。如果`event`的`ignore`设置为`true`，则直接返回，不再进行后续操作。

4. 安装`SystemInitialize`
   
   构造`SystemInitialize`，并设置脚本上下文变量`db`和`dbType`以及需要排除的表，其中排除的表为`hsweb.app.initTableExcludes`配置的表，然后调用`install`方法进行安装。具体安装方法如下：
   
   ```java
       public void install() throws Exception {
           init();
           initInstallInfo();
           initReadyToInstallDependencies();
           doInstall();
           syncSystemVersion();
       } 
   ```

5. `init`方法
   
   `init`方法主要是设置脚本上下文参数`sqlExecutor`、`database`和`logger`。其中`sqlExecutor`为`DefaultJdbcExecutor`
   
   ```java
       public void init() {
           if (initialized) {
               return;
           }
           if (!CollectionUtils.isEmpty(excludeTables)) {
               this.database = new SkipCreateOrAlterRDBDatabase(database, excludeTables, sqlExecutor);
           }
           scriptContext.put("sqlExecutor", sqlExecutor);
           scriptContext.put("database", database);
           scriptContext.put("logger", logger);
           initialized = true;
       }
   ```

6. `initInstallInfo`方法
   
   构造`s_system`表
   
   ```java
       protected void initInstallInfo() throws SQLException {
           boolean tableInstall = sqlExecutor.tableExists("s_system");
           database.createOrAlter("s_system")
                   .addColumn().name("name").varchar(128).comment("系统名称").commit()
                   .addColumn().name("major_version").alias(majorVersion).number(32).javaType(Integer.class).comment("主版本号").commit()
                   .addColumn().name("minor_version").alias(minorVersion).number(32).javaType(Integer.class).comment("次版本号").commit()
                   .addColumn().name("revision_version").alias(revisionVersion).number(32).javaType(Integer.class).comment("修订版").commit()
                   .addColumn().name("snapshot").number(1).javaType(Boolean.class)
                   .custom(column -> column.setValueConverter(new NumberValueConverter(Boolean.class)))
                   .comment("是否快照版").commit()
                   .addColumn().name("comment").varchar(2000).comment("系统说明").commit()
                   .addColumn().name("website").varchar(2000).comment("系统网址").commit()
                   .addColumn().name("framework_version").notNull().alias(frameworkVersion).clob()
                   .custom(column -> column.setValueConverter(new JSONValueConverter(SystemVersion.FrameworkVersion.class, new ClobValueConverter()))).notNull().comment("框架版本").commit()
                   .addColumn().name("dependencies").notNull().alias(dependencies).clob()
                   .custom(column -> column.setValueConverter(new JSONValueConverter(SystemVersion.Dependency.class, new ClobValueConverter()))).notNull().comment("依赖详情").commit()
                   .comment("系统信息")
                   .custom(table -> table.setObjectWrapper(new BeanWrapper<SystemVersion>(SystemVersion::new, table)))
                   .commit();
   
           if (!tableInstall) {
               installed = null;
               return;
           }
           RDBTable<SystemVersion> rdbTable = database.getTable("s_system");
           installed = rdbTable.createQuery().where("name", targetVersion.getName()).single();
       }
   ```

7.    `initReadyToInstallDependencies`方法
   
   获取js引擎，获取所有`hsweb-starter.js`脚本。循环每个脚本，设置`dependency`变量为`SimpleDependencyInstaller`，编译并执行。只要有执行失败的脚本，则直接抛出异常（`getIfSuccess`方法实现）。如果都执行成功，则将`SimpleDependencyInstaller`保存到`readyToInstall`变量中。
   
   ```java
   private String installScriptPath = "classpath*:hsweb-starter.js";
   
   private void initReadyToInstallDependencies() {
           DynamicScriptEngine engine = DynamicScriptEngineFactory.getEngine("js");
           try {
               Resource[] resources = new PathMatchingResourcePatternResolver().getResources(installScriptPath);
               List<SimpleDependencyInstaller> installers = new ArrayList<>();
               for (Resource resource : resources) {
                   String script = StreamUtils.copyToString(resource.getInputStream(), Charset.forName("utf-8"));
                   SimpleDependencyInstaller installer = new SimpleDependencyInstaller();
                   engine.compile("__tmp", script);
                   Map<String, Object> context = getScriptContext();
                   context.put("dependency", installer);
                   engine.execute("__tmp", context).getIfSuccess();
                   installers.add(installer);
               }
               readyToInstall = installers;
           } catch (Exception e) {
               throw new RuntimeException(e);
           } finally {
               engine.remove("__tmp");
           }
   
       }
   ```
   
   因为`JavaScriptEngine`类的编译方法会在脚本前后增加`(function(){`和`\n})();`，所以执行时会自动执行对应js脚本的内容。

```java
    public boolean compile(String id, String code) throws Exception {
        if (logger.isDebugEnabled()) {
            logger.debug("compile {} {} : {}", getScriptName(), id, code);
        }
        if (compilable == null)
            init();
        CompiledScript compiledScript = compilable.compile(StringUtils.concat("(function(){", code, "\n})();"));
        CommonScriptContext scriptContext = new CommonScriptContext(id, DigestUtils.md5Hex(code), compiledScript);
        scriptBase.put(id, scriptContext);
        return true;
    }    
```

8. `hsweb-starter.js`脚本
   
   拿demo里的脚本为例。
   
   ```js
   //组件信息
   var info = {
       groupId: "${project.groupId}",
       artifactId: "${project.artifactId}",
       version: "${project.version}",
       website: "https://github.com/hs-web/",
       author: "admin@hsweb.me",
       comment: "演示系统"
   };
   var menus = [{
       "describe": " ",
       "icon": "fa fa-cogs",
       "id": "e9dc96d6b677cbae865670e6813f5e8b",
       "name": "系统设置",
       "parentId": "-1",
       "path": "sOrB",
       "permissionId": "",
       "sortIndex": 1,
       "status": 1,
       "url": ""
   }, {
       "describe": " ",
       "icon": "fa fa-sitemap",
       "id": "org-01",
       "name": "组织架构",
       "parentId": "-1",
       "path": "a2o0",
       "permissionId": "",
       "sortIndex": 2,
       "status": 1,
       "url": ""
   }];
   
   var user = [
       {
           "id": "4291d7da9005377ec9aec4a71ea837f",
           "name": "超级管理员",
           "username": "admin",
           "password": "ba7a97be0609c22fa1d300691dfcd790",
           "salt": "HX8Hr5Yd",
           "status": 1,
           "creatorId": "admin",
           "createTime": new Date().getTime()
       }
   ];
   
   var autz_setting = [
       {
           "id": "98d74130b3cb06afc0ae8e5b57a6c052",
           "type": "user",
           "settingFor": "4291d7da9005377ec9aec4a71ea837f",
           "describe": null,
           "status": 1
       }
   ];
   var autz_menu = [];
   menus.forEach(function (menu) {
       autz_menu.push({
           id: org.hswebframework.web.id.IDGenerator.MD5.generate(),
           parentId: "-1",
           menuId: menu.id,
           status: 1,
           settingId: "98d74130b3cb06afc0ae8e5b57a6c052",
           path: "-"
       });
   });
   //版本更新信息
   var versions = [
       // {
       //     version: "3.0.0",
       //     upgrade: function (context) {
       //         java.lang.System.out.println("更新到3.0.2了");
       //     }
       // }
   ];
   var JDBCType = java.sql.JDBCType;
   
   function install(context) {
       var database = context.database;
   
   }
   
   function initialize(context) {
       var database = context.database;
       database.getTable("s_menu").createInsert().values(menus).exec();
       database.getTable("s_autz_setting").createInsert().values(autz_setting).exec();
       database.getTable("s_autz_menu").createInsert().values(autz_menu).exec();
       database.getTable("s_user").createInsert().values(user).exec();
   }
   
   //设置依赖
   dependency.setup(info)
       .onInstall(install)
       .onUpgrade(function (context) { //更新时执行
           var upgrader = context.upgrader;
           upgrader.filter(versions)
               .upgrade(function (newVer) {
                   newVer.upgrade(context);
               });
       })
       .onUninstall(function (context) { //卸载时执行
   
       }).onInitialize(initialize);
   ```
   
   `hsweb-starter.js`脚本中的`dependency`其实就是`SimpleDependencyInstaller`类的实例，`dependency.setup(info)`就是调用`SimpleDependencyInstaller`类的`setup`方法。
   
   根据js脚本的内容，再结合`SimpleDependencyInstaller`类，可以发现，js脚本的工作其实就是将`SimpleDependencyInstaller`中对应的`installer`、`upgrader`、`unInstaller`和`initializer`进行对应的设置。
   
   其中`installer`用于编写数据表创建脚本，`upgrader`用于编写更新脚本，`unInstaller`用于编写卸载脚本，`initializer`用于编写初始化数据，也就是插入初始数据的脚本

9. `doInstall`方法
   
   上一步设置了对应的内容，这一步就是执行具体的操作了。
   
   ```java
   protected void doInstall() {
           List<SimpleDependencyInstaller> doInitializeDep = new ArrayList<>();
           List<SystemVersion.Dependency> installedDependencies =
                   readyToInstall.stream().map(installer -> {
                       SystemVersion.Dependency dependency = installer.getDependency();
                       SystemVersion.Dependency installed = getInstalledDependency(dependency.getGroupId(), dependency.getArtifactId());
                       //安装依赖
                       if (installed == null) {
                           doInitializeDep.add(installer);
                           installer.doInstall(getScriptContext());
                       }
                       //更新依赖
                       if (installed == null || installed.compareTo(dependency) < 0) {
                           installer.doUpgrade(getScriptContext(), installed);
                       }
                       return dependency;
                   }).collect(Collectors.toList());
   
           for (SimpleDependencyInstaller installer : doInitializeDep) {
               installer.doInitialize(getScriptContext());
           }
           targetVersion.setDependencies(installedDependencies);
       }
   ```
   
   循环第7步设置的`readyToInstall`变量，判断是否有对应的依赖已经安装，如果没有安装，则安装依赖（避免多次启动的时候重复安装依赖）
   
   如果没有安装或者需要更新依赖，则调用`doUpgrade`方法，`doUpgrade`中会构造`SimpleDependencyUpgrader`类，然后设置给`upgrader`变量。
   
   ```java
       public void doUpgrade(Map<String, Object> context, SystemVersion.Dependency installed) {
           SimpleDependencyUpgrader simpleDependencyUpgrader =
                   new SimpleDependencyUpgrader(installed, dependency, context);
           context.put("upgrader", simpleDependencyUpgrader);
           if (upgrader != null) {
               upgrader.execute(context);
           }
       }
   ```
   
   全部安装完成后，调用`doInitialize`方法对安装的依赖进行初始化。
   
   最后调用`setDependencies`方法记录依赖信息。
   
   ```java
       public void setDependencies(List<Dependency> dependencies) {
           this.dependencies = dependencies;
           initDepCache();
       }
       protected void initDepCache() {
           depCache = new HashMap<>();
           dependencies.forEach(dependency -> depCache.put(getDepKey(dependency.groupId, dependency.artifactId), dependency));
       }
   ```

10. `syncSystemVersion`方法
    
    如果没有安装过，则在`s_system`表中插入一条记录，否则合并已安装的依赖，并更新`s_system`表对应的记录
    
    ```java
        protected void syncSystemVersion() throws SQLException {
            RDBTable<SystemVersion> rdbTable = database.getTable("s_system");
            if (installed == null) {
                rdbTable.createInsert().value(targetVersion).exec();
            } else {
                //合并已安装的依赖
                //修复如果删掉了依赖，再重启会丢失依赖信息的问题
                for (SystemVersion.Dependency dependency : installed.getDependencies()) {
                    SystemVersion.Dependency target = targetVersion.getDependency(dependency.getGroupId(), dependency.getArtifactId());
                    if (target == null) {
                        targetVersion.getDependencies().add(dependency);
                    }
                }
    
                rdbTable.createUpdate().set(targetVersion).where().is("name", targetVersion.getName()).exec();
            }
        }
    ```

至此，数据库表已经全部初始化，并插入了初始化的数据。

现在还有一个疑问，就是在执行js脚本`hsweb-starter.js`时，并没有指定先后顺序，那如何确定先后顺序呢？其实`hsweb-starter.js`脚本并没有依赖关系，也就不需要先后顺序。如果创建的表有先后顺序，则只能把创建表的代码都写在同一个`hsweb-starter.js`脚本里。

最后一个疑问，为什么有现成的技术，但作者却偏重新发明轮子？只能说也许作者在开发这部分代码时可能这两个技术还没有出来吧。


