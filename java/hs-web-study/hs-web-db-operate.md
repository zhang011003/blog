# hs-web中的数据库操作

hs-web中的数据库操作使用的是mybatis，不过是高度定制过的。

拿demo中的`SimpleUserService`类的`selectByUsername`方法为例来了解下。

首先`SimpleUserService`类继承自`AbstractService`类，实现了`DefaultDSLQueryService`接口。其中`AbstractService`提供了通用的模板方法，而`DefaultDSLQueryService`接口提供了通用的查询方法。

`SimpleUserService`类的`selectByUsername`方法

```java
    @Override
    @Transactional(readOnly = true)
    public UserEntity selectByUsername(String username) {
        if (!StringUtils.hasLength(username)) {
            return null;
        }
        return createQuery().where("username", username).single();
    }
```

`DefaultDSLQueryService`接口的`createQuery`方法。在这里设置了`listExecutor`、`totalExecutor`和`singleExecutor`

```java
    default Query<E, QueryParamEntity> createQuery() {
        Query<E, QueryParamEntity> query = Query.empty(new QueryParamEntity());
        query.setListExecutor(this::select);
        query.setTotalExecutor(this::count);
        query.setSingleExecutor(this::selectSingle);
        query.noPaging();
        return query;
    }
```

`org.hswebframework.ezorm.core.Conditional`中的`where`方法

```java
    default T where(String column, Object value) {
        return and(column, TermType.eq, value);
    }
```

`org.hswebframework.ezorm.core.dsl.Query`中的`and`方法和`single`方法

```java
    @Override
    public Query<T, Q> and(String column, String termType, Object value) {
        this.param.and(column, termType, value);
        return this;
    }

    public T single() {
        return singleExecutor.doExecute(param);
    }
```

其实`single`方法调用的就是`DefaultDSLQueryService`中的`selectSingle`方法

```java
    default E selectSingle(Entity param) {
        if (param instanceof QueryParamEntity) {
            ((QueryParamEntity) param).doPaging(0, 1);
        }
        List<E> list = this.select(param);
        if (list.isEmpty()) {
            return null;
        } else {
            return list.get(0);
        }
    }
    @Override
    @Transactional(readOnly = true)
    default List<E> select(Entity param) {
        if (param == null) {
            param = QueryParamEntity.empty();
        }
        return getDao().query(param);
    }
```

最终会调用dao中的`query`方法

`SimpleUserService`类的`getDao`方法

```java
    @Override
    public UserDao getDao() {
        return userDao;
    }
```

`org.hswebframework.web.dao.authorization.UserDao`接口

```java
public interface UserDao extends CrudDao<UserEntity, String> {
}
```

对应的mapper

```xml
<mapper namespace="org.hswebframework.web.dao.authorization.UserDao">
    <resultMap id="UserResultMap" type="org.hswebframework.web.entity.authorization.UserEntity">
        <id property="id" column="u_id" javaType="string" jdbcType="VARCHAR"/>
        <result property="name" column="name" javaType="string" jdbcType="VARCHAR"/>
        <result property="username" column="username" javaType="string" jdbcType="VARCHAR"/>
        <result property="password" column="password" javaType="String" jdbcType="VARCHAR"/>
        <result property="salt" column="salt" javaType="String" jdbcType="VARCHAR"/>
        <result property="status" column="status" javaType="Byte" jdbcType="NUMERIC"/>
        <result property="createTime" column="create_time" javaType="Long" jdbcType="NUMERIC"/>
        <result property="creatorId" column="creator_id" javaType="String" jdbcType="VARCHAR"/>
    </resultMap>

    <!--用于动态生成sql所需的配置-->
    <sql id="config">
        <bind name="resultMapId" value="'UserResultMap'"/>
        <bind name="tableName" value="'s_user'"/>
    </sql>

    <select id="query" parameterType="org.hswebframework.web.commons.entity.Entity" resultMap="UserResultMap">
        <include refid="config"/>
        <include refid="BasicMapper.buildSelectSql"/>
    </select>
</mapper>
```

`query`方法引入的`BasicMapper.buildSelectSql`

```xml
    <!--生成查询sql-->
    <sql id="buildSelectSql">
        <include refid="BasicMapper.switcher"/>
        <trim>
            select
            <include refid="BasicMapper.buildSelectField"/>
            from ${_fullTableName}
            <where>
                <include refid="BasicMapper.buildWhere"/>
            </where>
            <include refid="BasicMapper.buildSortField"/>
        </trim>
    </sql>
   <sql id="switcher">

        <bind name="_fullTableName" value="tableName"/>
        <!--当前数据库-->
        <bind name="_databaseName"
              value="@org.hswebframework.web.datasource.DataSourceHolder@databaseSwitcher().currentDatabase()"/>
        <!--表全名前缀-->
        <bind name="_databasePrefix" value="''"/>

        <!--当前表名-->
        <bind name="_currentTableName"
              value="@org.hswebframework.web.datasource.DataSourceHolder@tableSwitcher().getTable(tableName)"/>

        <if test="_currentTableName==null">
            <bind name="_currentTableName" value="tableName"/>
        </if>
        <if test="_databaseName!=null">
            <bind name="_databasePrefix" value="_databaseName+'.'"/>
        </if>
        <if test="_currentTableName!=null">
            <bind name="_fullTableName" value="_databasePrefix+_currentTableName"/>
        </if>
    </sql>
        <!--通用查询条件-->
    <sql id="buildWhere">
        ${@org.hswebframework.web.dao.mybatis.builder.SqlBuilder@current().buildWhere(resultMapId,tableName,#this['_parameter'])}
    </sql>
    <sql id="buildWhereForUpdate">
        ${@org.hswebframework.web.dao.mybatis.builder.SqlBuilder@current().buildWhereForUpdate(resultMapId,tableName,#this['_parameter'])}
    </sql>
    <!--生成查询字段-->
    <sql id="buildSelectField">
        ${@org.hswebframework.web.dao.mybatis.builder.SqlBuilder@current().buildSelectFields(resultMapId,tableName,#this['_parameter'])}
    </sql>
```

`switcher`用于设定一些变量，`buildSelectField`根据变量构造select子句，这里用的是ognl的语法来实现代码的调用

`buildSelectFields`方法

```java
    public String buildSelectFields(String resultMapId, String tableName, Object arg) {
        QueryParam param = null;
        if (arg instanceof QueryParam) {
            param = ((QueryParam) arg);
            if (param.isPaging()) {
                if (Pager.get() == null) {
                    Pager.doPaging(param.getPageIndex(), param.getPageSize());
                }
            } else {
                Pager.reset();
            }
        }
        if (param == null) {
            return "*";
        }

        RDBTableMetaData tableMetaData = createMeta(tableName, resultMapId);
        RDBDatabaseMetaData databaseMetaDate = getActiveDatabase();
        Dialect dialect = databaseMetaDate.getDialect();
        CommonSqlRender render = (CommonSqlRender) databaseMetaDate.getRenderer(SqlRender.TYPE.SELECT);
        List<CommonSqlRender.OperationColumn> columns = render.parseOperationField(tableMetaData, param);
        SqlAppender appender = new SqlAppender();
        columns.forEach(column -> {
            RDBColumnMetaData columnMetaData = column.getRDBColumnMetaData();
            if (columnMetaData == null) {
                return;
            }
            String cname = columnMetaData.getName();
            if (!cname.contains(".")) {
                cname = tableMetaData.getName().concat(".").concat(cname);
            }
            boolean isJpa = columnMetaData.getProperty("fromJpa", false).isTrue();

            appender.add(",", encodeColumn(dialect, cname)
                    , " AS "
                    , dialect.getQuoteStart()
                    , isJpa ? columnMetaData.getAlias() : columnMetaData.getName()
                    , dialect.getQuoteEnd());
        });
        param.getIncludes().remove("*");
        if (appender.isEmpty()) {
            return "*";
        }
        appender.removeFirst();
        return appender.toString();
    }
```

`createMeta`方法

```java
    protected RDBTableMetaData createMeta(String tableName, String resultMapId) {
//        tableName = getRealTableName(tableName);
        RDBDatabaseMetaData active = getActiveDatabase();
        String cacheKey = tableName.concat("-").concat(resultMapId);
        Map<String, RDBTableMetaData> cache = metaCache.computeIfAbsent(active, k -> new ConcurrentHashMap<>());

        RDBTableMetaData cached = cache.get(cacheKey);
        if (cached != null) {
            return cached;
        }

        RDBTableMetaData rdbTableMetaData = new RDBTableMetaData() {
            @Override
            public String getName() {
                //动态切换表名
                return getRealTableName(tableName);
            }
        };
        ResultMap resultMaps = MybatisUtils.getResultMap(resultMapId);
        rdbTableMetaData.setName(tableName);
        rdbTableMetaData.setDatabaseMetaData(active);

        List<ResultMapping> resultMappings = new ArrayList<>(resultMaps.getResultMappings());
        resultMappings.addAll(resultMaps.getIdResultMappings());

        resultMappings.stream()
                .map(mapping -> this.createColumn(null, null, mapping))
                .flatMap(Collection::stream)
                .forEach(rdbTableMetaData::addColumn);

        if (useJpa) {
            Class type = entityFactory == null ? resultMaps.getType() : entityFactory.getInstanceType(resultMaps.getType());
            RDBTableMetaData parseResult = JpaAnnotationParser.parseMetaDataFromEntity(type);
            if (parseResult != null) {
                for (RDBColumnMetaData columnMetaData : parseResult.getColumns()) {
                    if (rdbTableMetaData.findColumn(columnMetaData.getName()) == null) {
                        columnMetaData = columnMetaData.clone();
                        columnMetaData.setProperty("fromJpa", true);
                        rdbTableMetaData.addColumn(columnMetaData);
                    }
                }
            }
        }
        for (RDBColumnMetaData column : rdbTableMetaData.getColumns()) {
            //时间
            if (column.getJdbcType() == JDBCType.DATE || column.getJdbcType() == JDBCType.TIMESTAMP) {
                ValueConverter dateConvert = new DateTimeConverter("yyyy-MM-dd HH:mm:ss", column.getJavaType()) {
                    @Override
                    public Object getData(Object value) {
                        if (value instanceof Number) {
                            return new Date(((Number) value).longValue());
                        }
                        return super.getData(value);
                    }
                };
                column.setValueConverter(dateConvert);
            } else if (column.getJavaType() == boolean.class || column.getJavaType() == Boolean.class) {
                column.setValueConverter(new BooleanValueConverter(column.getJdbcType()));
                column.setProperty("typeHandler", NumberBooleanTypeHandler.class.getName());
            } else if (TypeUtils.isNumberType(column)) { //数字
                //数字
                column.setValueConverter(new NumberValueConverter(column.getJavaType()));
            }
        }
        cache.put(cacheKey, rdbTableMetaData);
        return rdbTableMetaData;
    }
```

`MybatisUtils`类

```java
public class MybatisUtils {
    volatile static SqlSessionFactory sqlSession;

    public static ResultMap getResultMap(String id) {
        return getSqlSession().getConfiguration().getResultMap(id);
    }

    public static SqlSessionFactory getSqlSession() {
        if (sqlSession == null) {
            throw new UnsupportedOperationException("sqlSession is null");
        }
        return sqlSession;
    }
}
```

这里直接使用`sqlSession`变量，查找了一下，发现

`org.hswebframework.web.dao.mybatis.MyBatisAutoConfiguration`的`sqlSessionFactory`方法设置了`sqlSession`变量

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

这段代码里构造`SqlSessionFactory`实例时会注入插件`interceptors`，查找了一下，有一个默认的拦截器`org.hswebframework.web.dao.mybatis.plgins.pager.PagerInterceptor`

```java
@Intercepts({@Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class,
        RowBounds.class, ResultHandler.class})})
@Component
public class PagerInterceptor implements Interceptor {

    @Override
    public Object intercept(Invocation target) throws Throwable {
        return target.proceed();
    }

    @Override
    public Object plugin(Object target) {
        if (target instanceof StatementHandler) {
            StatementHandler statementHandler = (StatementHandler) target;
            MetaObject metaStatementHandler = SystemMetaObject.forObject(statementHandler);
            String sql = statementHandler.getBoundSql().getSql();
            Pager pager = Pager.getAndReset();

            String lower = sql.trim();

            if (lower.startsWith("select")) {
                if (lower.contains("count(")) {
                    return Plugin.wrap(target, this);
                }
                String newSql = sql;
                if (pager != null) {
                    newSql = EasyOrmSqlBuilder.getInstance()
                            .getActiveDatabase().getDialect()
                            .doPaging(sql, pager.pageIndex(), pager.pageSize());
                }
                Object queryEntity = statementHandler.getParameterHandler().getParameterObject();
                if (queryEntity instanceof QueryParam && ((QueryParam) queryEntity).isForUpdate()) {
                    newSql = newSql + " for update";
                }
                metaStatementHandler.setValue("delegate.boundSql.sql", newSql);
            }

        }
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {
    }
}
```

该拦截器只拦截了`query`方法，然后判断如果sql中包含`count(`，则直接返回，否则根据不同的方言来重新构造sql语句来增加分页的功能。

这部分代码很长，也没有仔细看，不过感觉应该是根据xml中配置的`resultMapId`来自动生成查询的字段，where语句以及sort语句的生成也类似。分页的逻辑则是在构造查询字段时判断传入的查询参数类型如果为`QueryParam`且需要分页则保存当前请求分页的参数，再使用自定义的插件来构造分页的sql语句。

这样查询就完成了。

代码虽然复杂，不过却简化了配置文件的内容。只需要配置的时候配置resultMap以及相关的一些参数就够了。不过现在都使用mybatis-plus组件，这么复杂的设计应该不再需要了吧。
