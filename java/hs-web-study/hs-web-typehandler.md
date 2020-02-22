# hs-web中类型处理器（typeHandler）的注册

之前看`s_permission`表中的数据时，发现`actions`、`spt_da_types`和`optional_fields`等字段中存储的都是json格式的数据，就会有一个疑问：hs-web中对象是如何转换为json格式数据存入数据表中的？这个其实是用类型注册器（typeHandler）来实现的

## 类型处理器（typeHandler）

mybatis中的typeHandler主要用途就是将java的类型与jdbc类型做对应，具体可以参考[这里](https://mybatis.org/mybatis-3/configuration.html#typeHandlers)。mybatis中自带了多个默认的typeHandler，也可以自定义typeHandler。

## hs-web中的typeHandler注册

hs-web中的关于mybatis的扩展都在hsweb-commons-dao-mybatis模块中。

`MyBatisAutoConfiguration`类的`sqlSessionFactory`方法自定义了`sqlSessionFactory`

```java
    @Bean(name = "sqlSessionFactory")

    @Primary
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
        MybatisProperties mybatisProperties = this.mybatisProperties();
        if (null != entityFactory) {
            factory.setObjectFactory(new MybatisEntityFactory(entityFactory));
        }
        factory.setVfs(SpringBootVFS.class);
        if (mybatisProperties().isDynamicDatasource()) {
            factory.setSqlSessionFactoryBuilder(new DynamicDataSourceSqlSessionFactoryBuilder());
            factory.setTransactionFactory(new SpringManagedTransactionFactory() {
                @Override
                public Transaction newTransaction(DataSource dataSource, TransactionIsolationLevel level, boolean autoCommit) {
                    return new DynamicSpringManagedTransaction();
                }
            });
        }
        factory.setDataSource(dataSource);
        if (StringUtils.hasText(mybatisProperties.getConfigLocation())) {
            factory.setConfigLocation(this.resourceLoader.getResource(mybatisProperties
                    .getConfigLocation()));
        }
        if (mybatisProperties.getConfiguration() != null) {
            factory.setConfiguration(mybatisProperties.getConfiguration());
        }
        if (this.interceptors != null && this.interceptors.length > 0) {
            factory.setPlugins(this.interceptors);
        }
        if (this.databaseIdProvider != null) {
            factory.setDatabaseIdProvider(this.databaseIdProvider);
        }
        factory.setTypeAliasesPackage(mybatisProperties.getTypeAliasesPackage());
        String typeHandlers = "org.hswebframework.web.dao.mybatis.handler";
        if (mybatisProperties.getTypeHandlersPackage() != null) {
            typeHandlers = typeHandlers + ";" + mybatisProperties.getTypeHandlersPackage();
        }
        factory.setTypeHandlersPackage(typeHandlers);
        factory.setMapperLocations(mybatisProperties.resolveMapperLocations());

        SqlSessionFactory sqlSessionFactory = factory.getObject();
        MybatisUtils.sqlSession = sqlSessionFactory;

        EnumDictHandlerRegister.typeHandlerRegistry = sqlSessionFactory.getConfiguration().getTypeHandlerRegistry();
        EnumDictHandlerRegister.register("org.hswebframework.web;" + mybatisProperties.getTypeHandlersPackage());

        try {
            Class.forName("javax.persistence.Table");
            EasyOrmSqlBuilder.getInstance().useJpa = mybatisProperties.isUseJpa();
        } catch (@SuppressWarnings("all") Exception ignore) {
        }
        EasyOrmSqlBuilder.getInstance().entityFactory = entityFactory;

        return sqlSessionFactory;
    }
```

在这里，会自动设置`typeHandlersPackage`属性为`org.hswebframework.web.dao.mybatis.handler`以及配置文件中配置的`typeHandlersPackage`，而将json转换为字符串的typeHandler就在这个包里。`JsonArrayHandler`、`JsonMapHandler`、`JsonSetHandler`为将java三种集合对象转换为字符串的typeHandler，`NumberBooleanTypeHandler`为将数字转换为bool的typeHandler。

## 其它枚举类型的注册

在注册完成默认的类型后，会调用`EnumDictHandlerRegister`类的`register`方法来注册默认的枚举类型。

```java
   public static void register(String[] packages) {

        if (typeHandlerRegistry == null) {
            log.error("请在spring容器初始化后再调用此方法!");
            return;
        }
        for (String basePackage : packages) {
            String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
                    ClassUtils.convertClassNameToResourcePath(basePackage) + "/**/*.class";
            try {
                Resource[] resources = resourcePatternResolver.getResources(packageSearchPath);
                for (Resource resource : resources) {
                    try {
                        MetadataReader reader = metadataReaderFactory.getMetadataReader(resource);
                        Class enumType = Class.forName(reader.getClassMetadata().getClassName());
                        if (enumType.isEnum() && EnumDict.class.isAssignableFrom(enumType)) {
                            log.debug("register enum dict:{}", enumType);
                            DefaultDictDefineRepository.registerDefine(DefaultDictDefineRepository.parseEnumDict(enumType));
                            //注册枚举类型
                            typeHandlerRegistry.register(enumType, new EnumDictHandler(enumType));

                            //注册枚举数组类型
                            typeHandlerRegistry.register(Array.newInstance(enumType, 0).getClass(), new EnumDictArrayHandler(enumType));
                        }
                    } catch (Exception | Error ignore) {

                    }
                }
            } catch (IOException e) {
                log.warn("register enum dict error", e);
            }
        }
    }
```

该方法会查找传入的package下所有的枚举类并且实现了`EnumDict`的类，然后将枚举类和`EnumDictHandler`注册到`typeHandlerRegistry`中，然后将枚举类的数组和`EnumDictArrayHandler`也注册到`typeHandlerRegistry`中。

当完成了以上的typeHandler的注册后，在查询或保存过程中，mybatis就会将java对象和json字符串自动转换了。
