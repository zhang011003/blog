# Spring Oauth

这篇文章其实是对 [OAuth 2 Developers Guide](https://projects.spring.io/spring-security-oauth/docs/oauth2.html)文章的大概翻译

分为两部分：OAuth 2.0提供者以及OAuth 2.0客户端

## OAuth 2.0  Provider

OAuth 2.0提供者机制用于暴露OAuth 2.0保护的资源。配置涉及建立OAuth 2.0客户端的连接，它用于独立访问保护的资源，或者代表了用户。提供者通过管理和验证用于访问保护资源的OAuth 2.0 token来达到这一目的的。当可用的时候，provider也必须提供给用户接口用来确认用户能够被授予访问保护资源的权限（例如：确认页面）

### OAuth 2.0 Provider实现

在OAuth 2.0中，provider角色实际上在鉴权服务和资源服务间被分开，有时它们在同一个应用中。使用Spring Security OAuth，你可以将它们分开到两个应用中，也可以多个资源服务共享同一个鉴权服务。获取token的请求由Spring MVC Controller端点处理，对保护资源的访问由标准的Spring Security请求过滤器处理。如下的端点在Spring Security过滤器链中会用来实现OAuth 2.0鉴权服务：

- [`AuthorizationEndpoint`](https://docs.spring.io/spring-security/oauth/apidocs/org/springframework/security/oauth2/provider/endpoint/AuthorizationEndpoint.html)用于鉴权的服务请求。默认的URL地址：/oauth/authorize
- [`TokenEndpoint`](https://docs.spring.io/spring-security/oauth/apidocs/org/springframework/security/oauth2/provider/endpoint/TokenEndpoint.html)用于访问token的服务请求。默认的URL地址：/oauth/token

如下过滤器用于实现OAuth 2.0资源服务

- [`OAuth2AuthenticationProcessingFilter`](https://docs.spring.io/spring-security/oauth/apidocs/org/springframework/security/oauth2/provider/authentication/OAuth2AuthenticationProcessingFilter.html)用于加载给定鉴权token的请求的鉴权

对于所有OAuth 2.0 provider特性，通过使用Spring OAuth的@Configuration注解，配置非常简单。

### 鉴权服务配置

当配置鉴权服务的时候，必须考虑客户端用于获取访问token的授权类型（如：鉴权码，用户授信，刷新token）。服务端的配置用来提供客户端详情服务和token服务的实现，以及全局配置某些切面是否可用。但是要注意，每个客户端可以单独配置使用某种鉴权机制和访问授权的权限，仅仅是因为provider配置支持“客户授信”鉴权类型，它并不代表特定客户端能够使用那种鉴权类型来授权。

@EnableAuthorizationServer注解用于配置OAuth 2.0鉴权服务机制，与实现了AuthorizationServerConfigurer的@Bean配合使用（已提供便利的空方法实现的适配器）。如下的特性委派给各自的配置，它们由Spring创建且传递到AuthorizationServerConfigurer中。

- ClientDetailsServiceConfigurer：定义了客户端服务的配置。客户端详情可以被初始化，也可以指定到存在的store
- AuthorizationServerSecurityConfigurer：定义了token端的安全常量
- AuthorizationServerEndpointsConfigurer：定义了鉴权和token端点和token服务

Provider配置的一个重要的方面是提供给OAuth客户端的鉴权码的方式（在鉴权码授权情况下）。鉴权码由OAuth客户端通过直连终端用户和鉴权页面方式来活得，用户在终端页面输入授信，然后带着鉴权码从provider鉴权服务器跳回到OAuth客户端。这个的示例在OAuth 2的定义中有例子。

xml配置中有<authorization-server/>元素用于配置OAuth 2.0鉴权服务。

### 配置客户端详情

ClientDetailsServiceConfigurer（从AuthorizationServerConfigurer的回调）用于定义客户端详情服务的内存中或jdbc实现。客户端重要的的属性有：

- clientId：（必须）客户端id
- secret：（对于信任客户端必须）客户端密码
- scope：客户端限定的范围。如果scope未定义或者为空（默认情况），客户端不被限定范围
- authorizedGrantTypes：客户端鉴权使用的授权类型。默认为空
- authorities：客户端授予的权限（常规的Spring Security权限）

客户端详情在运行中的应用中可以通过直接访问存储（例如：如果使用JdbcClientDetailsService则是数据库表）或者通过ClientDetailsManager接口（两者都实现了ClientDetailsService接口）来更新。

> **注意**：jdbc服务的schema并没有和库一起打包（因为在实践中数据库有太多不用类型），但你可以参考如下例子[test code in github](https://github.com/spring-projects/spring-security-oauth/blob/master/spring-security-oauth2/src/test/resources/schema.sql).

### 管理token

[`AuthorizationServerTokenServices`](https://docs.spring.io/spring-security/oauth/apidocs/org/springframework/security/oauth2/provider/token/AuthorizationServerTokenServices.html)接口用来定义管理OAuth 2.0 token的操作。注意以下几点：

- 当访问的token被创建时，鉴权必须保存下来。接受访问token的资源后续可以引用它。
- 访问的token用于加载用来授权创建的鉴权

当创建AuthorizationServerTokenServices实现的时候，你需要考虑[`DefaultTokenServices`](https://docs.spring.io/spring-security/oauth/apidocs/org/springframework/security/oauth2/provider/token/DefaultTokenServices.html)，它有许多可以插入的策略用来改变访问token的格式和存储。默认它通过随机数创建token，处理除了token存储的所有事情，token存储委派給TokenStore。默认的存储是[in-memory implementation](https://docs.spring.io/spring-security/oauth/apidocs/org/springframework/security/oauth2/provider/token/store/InMemoryTokenStore.html)，但有其它可用的实现。下面是各个不同存储的讨论：

- 默认的InMemoryTokenStore对于单个服务器来说非常适用（低阻塞，失败时没有与备份服务器的热交换）。大多数项目可以从这里开始，也可以在开发模型下使用，不需要其它依赖就可以启动
- JdbcTokenStore是[JDBC版本](https://projects.spring.io/spring-security-oauth/docs/JdbcTokenStore)，用来存储到关系数据库中。如果可以在不同服务间共享数据库，既可以通过扩容相同服务器，也可以扩容鉴权服务器和资源服务器，则可以使用JDBC版本。如果使用JdbcTokenStore，需要增加spring-jdbc
- [JSON Web Token (JWT)版本](https://projects.spring.io/spring-security-oauth/docs/`JwtTokenStore`)编码所有授权的数据到token中（因此后端没有存储是最大的好处）。一个缺点是不容易收回访问的token，因此通常被授予短期时效，在刷新token的时候就可以收回token。另一个缺点是如果存储了太多的用户授信信息，则token会非常大。JwtTokenStore并不是真正的存储因为它不持久化任何数据，但它通过DefaultTokenServices在token值和鉴权信息的转换上扮演同样的角色

> 注意：jdbc服务的schema并没有和库一起打包（因为在实践中数据库有太多不用类型），但你可以参考如下例子[test code in github](https://github.com/spring-projects/spring-security-oauth/blob/master/spring-security-oauth2/src/test/resources/schema.sql)。确保使用@EnableTransactionManagement防止客户端应用在token创建时竞争相同行而导致的冲突。

### JWT Token

如果使用JWT token，需要在鉴权服务器中定义JwtTokenStore。资源服务器也需要解码token，因此JwtTokenStore需要有JwtAccessTokenConverter的依赖。相同的实现在鉴权服务器和资源服务器中都需要。token默认被签名，资源服务器必须验证签名，因此它需要使用和鉴权服务器相同的对称（签名）key（共享secret或者对称key），或者它需要public key（验证key）来匹配鉴权服务器的private key（签名key）（public-private或者非对称key）。public key（如果可用）由鉴权服务器在/oauth/token_key端点暴露，默认是使用访问规则“denyAll()”。你可以通过在AuthorizationServerSecurityConfigurer中注入SpEL表达式来放开（例如：如果为public key，则可以使用“permitAll()”）。

要想使用JwtTokenStore，你需要将“spring-security-jwt”放入classpath（可以在Spring OAuth相同的github仓库中找到，不过发布周期不一样）

### 授权类型

授权类型由AuthorizationEndpoint支持，通过AuthorizationServerEndpointsConfigurer配置。默认所有授权类型都支持，除了password（参加下面如何开启）。如下的属性影响授权类型：

- authenticationManager：通过注入AuthenticationManager开启password授权
- userDetailsService：如果注入UserDetailsService，或者全局有配置（如在GlobalAuthenticationManagerConfigurer中配置），则刷新token授权会包括用户详情的检查，用来确保账户仍然可用
- authorizationCodeServices：定义了授权码服务（AuthorizationCodeServices的实例）用于授权码授权
- implicitGrantService：管理隐式授权期间的状态
- tokenGranter：TokenGranter（用于完全控制授权以及忽略上面提到的其它的属性）

XML中，授权类型作为authorization-server的孩子节点

### 配置端点URL

AuthorizationServerEndpointsConfigurer有pathMapping()方法，它有两个参数：

- 端点的默认URL路径（框架已实现）
- 客户自定义路径（以“/”开头）

框架提供的URL路径是/oauth/authorize（鉴权端点），/oauth/token（token端点），/oauth/confirm_access（用户通过该端口授权），/oauth/error（鉴权服务器用于渲染错误），/oauth/check_token（资源服务器用来解码访问token）以及/oauth/token_key（如果使用JWT token，则用于暴露token验证的public key）

注意：鉴权端点/oauth/authorize（或者它的其它映射）需要通过Spring Security来保护，以便它只能被授权的用户访问。例如使用标准的Spring Security WebSecurityConfigurer：

```java
@Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests().antMatchers("/login").permitAll().and()
        // default protection for all resources (including /oauth/authorize)
            .authorizeRequests()
                .anyRequest().hasRole("USER")
        // ... more configuration, e.g. for form login
    }
```

> 注意：如果你的鉴权服务器也是资源服务器，那么也有另一个低优先级的安全过滤器控制api资源。对于这些需要通过访问token来保护的请求，你需要让它们的路径在面对用户的过滤器链中被过滤，因此确保包括一个请求匹配器，将WebSecurityConfigurer配置的非API资源的请求挑出来

token端点默认由Spring OAuth通过@Configuration配置，通过使用HTTP Basic鉴权客户密钥来保护。在xml中不是这样（因此需要显示配置）

在xml中，<authorization-server/>元素有一些属性可以用来改变URL的默认端点。/check_token端点需要显示地使能（通过check-token-enabled属性）

### 定制UI



