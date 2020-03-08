# Spring Oauth

这篇文章其实是对 [OAuth 2 Developers Guide](https://projects.spring.io/spring-security-oauth/docs/oauth2.html)文章的大概翻译

分为两部分：OAuth 2.0提供者以及OAuth 2.0客户端。最好的示例代码是[i集成测试](https://github.com/spring-projects/spring-security-oauth/tree/master/tests)和[示例应用](https://github.com/spring-projects/spring-security-oauth/tree/master/samples/oauth2).

## OAuth 2.0  提供者

OAuth 2.0提供者机制用于暴露OAuth 2.0保护的资源。配置包括建立OAuth 2.0客户端连接来访问保护资源，或者代表了用户。提供者通过管理和验证用于访问保护资源的OAuth 2.0 token来达到这一目的。当应用的时候，provider也必须提供给用户接口用来确认用户能够被授予访问保护资源的权限（例如：确认页面）

### OAuth 2.0 提供者实现

在OAuth 2.0中，提供者角色实际上在鉴权服务和资源服务间被分开，有时它们在同一个应用中。使用Spring Security OAuth，你可以将它们分开到两个应用中，也可以多个资源服务共享同一个鉴权服务。获取token的请求由Spring MVC Controller端点处理，对保护资源的访问由标准的Spring Security请求过滤器处理。如下的端点在Spring Security过滤器链中会用来实现OAuth 2.0鉴权服务：

- [`AuthorizationEndpoint`](https://docs.spring.io/spring-security/oauth/apidocs/org/springframework/security/oauth2/provider/endpoint/AuthorizationEndpoint.html)用于鉴权的服务请求。默认的URL地址：/oauth/authorize
- [`TokenEndpoint`](https://docs.spring.io/spring-security/oauth/apidocs/org/springframework/security/oauth2/provider/endpoint/TokenEndpoint.html)用于访问token的服务请求。默认的URL地址：/oauth/token

如下过滤器用于实现OAuth 2.0资源服务

- [`OAuth2AuthenticationProcessingFilter`](https://docs.spring.io/spring-security/oauth/apidocs/org/springframework/security/oauth2/provider/authentication/OAuth2AuthenticationProcessingFilter.html)用于加载给定授权访问token请求的鉴权

对于所有OAuth 2.0 提供者的特性，通过使用Spring OAuth的@Configuration注解，配置非常简单。也提供了基于XML命名空间的OAuth配置。

### 鉴权服务配置

当配置鉴权服务的时候，必须考虑客户端用于获取访问token的授权类型（如：鉴权码，用户凭证，刷新token）。服务端的配置用来提供客户端详情服务和token服务的实现，以及全局配置某些切面是否可用。但是要注意，每个客户端可以单独配置权限，以便能够使用某种授权机制和访问授权，也就是说，仅仅是因为提供者配置支持“客户凭据”授权类型，它并不代表特定客户端能够使用那种鉴权类型来授权。

`@EnableAuthorizationServer`注解用于配置OAuth 2.0鉴权服务机制，与实现了`AuthorizationServerConfigurer`的`@Bean`配合使用（已提供便利的空方法实现的适配器）。如下的特性委派给各自的配置，它们由Spring创建且传递到`AuthorizationServerConfigurer`中。

- `ClientDetailsServiceConfigurer`：定义了客户端详情服务的配置。客户端详情可以被初始化，也可以指定到存在的store
- `AuthorizationServerSecurityConfigurer`：定义了token端点的安全常量
- `AuthorizationServerEndpointsConfigurer`：定义了鉴权和token端点和token服务

提供者配置的一个重要的方面是向OAuth客户端提供授权码的方式（在授权码授权情况下）。授权码由OAuth客户端通过直连终端用户和鉴权页面方式来获得，用户在终端页面输入凭据，然后带着授权码从提供者鉴权服务器跳回到OAuth客户端。这在OAuth 2的规范中有详细的例子。

在XML中，`<authorization-server/>`元素用于配置OAuth 2.0鉴权服务。

### 配置客户端详情

`ClientDetailsServiceConfigurer`（从`AuthorizationServerConfigurer`的回调）用于定义客户端详情服务的内存中或jdbc实现。客户端重要的的属性有：

- `clientId`：（必须）客户端id
- `secret`：（对于信任客户端必须）客户端密码
- `scope`：客户端限定的范围。如果scope未定义或者为空（默认情况），客户端不被限定范围
- `authorizedGrantTypes`：客户端鉴权使用的授权类型。默认为空
- `authorities`：授予给客户端的权限（常规的Spring Security权限）

客户端详情在运行中的应用中可以通过直接访问存储（例如：如果使用`JdbcClientDetailsService`则是数据库表）或者通过`ClientDetailsManager`接口（两者都实现了`ClientDetailsService`接口）来更新。

> **注意**：jdbc服务的schema并没有和库一起打包（因为在实践中数据库有太多不用类型），但你可以参考如下例子[test code in github](https://github.com/spring-projects/spring-security-oauth/blob/master/spring-security-oauth2/src/test/resources/schema.sql).

### 管理token

[`AuthorizationServerTokenServices`](https://docs.spring.io/spring-security/oauth/apidocs/org/springframework/security/oauth2/provider/token/AuthorizationServerTokenServices.html)接口用来定义管理OAuth 2.0 token的操作。注意以下几点：

- 当访问的token被创建时，鉴权必须保存下来。接受访问token的资源后续可以引用它。
- 访问的token用来加载用于授权创建的鉴权

当创建`AuthorizationServerTokenServices`实现的时候，你需要考虑[`DefaultTokenServices`](https://docs.spring.io/spring-security/oauth/apidocs/org/springframework/security/oauth2/provider/token/DefaultTokenServices.html)，它有许多可以插入的策略用来改变访问token的格式和存储。默认它通过随机数创建token，处理除了token存储的所有事情，token存储委派給`TokenStore`。默认的存储是[in-memory implementation](https://docs.spring.io/spring-security/oauth/apidocs/org/springframework/security/oauth2/provider/token/store/InMemoryTokenStore.html)，但有其它可用的实现。下面是各个不同存储的讨论：

- 默认的`InMemoryTokenStore`对于单个服务器来说非常适用（低阻塞，失败时没有与备份服务器的热交换）。大多数项目可以从这里开始，也可以在开发模型下使用，不需要其它依赖就可以启动
- `JdbcTokenStore`是[JDBC版本](https://projects.spring.io/spring-security-oauth/docs/JdbcTokenStore)，用来存储token到关系数据库中。如果可以在不同服务间共享数据库，或者通过扩容相同的服务器（如果只有一台），或者通过扩容鉴权服务器和资源服务器（如果有多个组件），则可以使用JDBC版本。如果使用`JdbcTokenStore`，需要在类路径上增加spring-jdbc
- [JSON Web Token (JWT)版本](https://projects.spring.io/spring-security-oauth/docs/`JwtTokenStore`)编码所有授权数据到token中（因此最大的好处是后端没有存储）。一个缺点是不容易收回访问的token，因此通常被授予短期时效，在刷新token的时候就可以收回token。另一个缺点是如果存储了太多的用户授信信息，则token会非常大。`JwtTokenStore`并不是真正的存储因为它不持久化任何数据，但它在`DefaultTokenServices`中在token值和鉴权信息的转换上扮演同样的角色

> 注意：jdbc服务的schema并没有和库一起打包（因为在实践中数据库有太多不用类型），但你可以参考如下例子[test code in github](https://github.com/spring-projects/spring-security-oauth/blob/master/spring-security-oauth2/src/test/resources/schema.sql)。确保使用`@EnableTransactionManagement`防止客户端应用在token创建时竞争相同行而导致的冲突。

### JWT Token

如果使用JWT token，需要在鉴权服务器中定义`JwtTokenStore`。资源服务器也需要解码token，因此`JwtTokenStore`需要有`JwtAccessTokenConverter`的依赖。相同的实现在鉴权服务器和资源服务器中都需要。token默认被签名，资源服务器必须验证签名，因此它需要使用和鉴权服务器相同的对称（签名）key（共享secret或者对称key），或者它需要public key（验证key）来匹配鉴权服务器的private key（签名key）（public-private或者非对称key）。public key（如果可用）由鉴权服务器在/oauth/token_key端点暴露，默认是使用访问规则“denyAll()”。你可以通过在AuthorizationServerSecurityConfigurer中注入SpEL表达式来放开（例如：如果为public key，则可以使用“permitAll()”）。

要想使用JwtTokenStore，你需要将“spring-security-jwt”放入classpath（可以在Spring OAuth相同的github仓库中找到，不过发布周期不一样）

### 授权类型

授权类型由`AuthorizationEndpoint`支持，通过`AuthorizationServerEndpointsConfigurer`配置。默认所有授权类型都支持，除了password（参见下面如何开启）。如下的属性影响授权类型：

- `authenticationManager`：通过注入`AuthenticationManager`开启password授权
- `userDetailsService`：如果注入`UserDetailsService`，或者全局有配置（如在`GlobalAuthenticationManagerConfigurer`中配置），则刷新token授权会包括用户详情的检查，用来确保账户仍然可用
- `authorizationCodeServices`：定义了授权码服务（`AuthorizationCodeServices`的实例）用于授权码授权
- `implicitGrantService`：管理隐式授权期间的状态
- `tokenGranter`：`TokenGranter`（完全控制授权，忽略上面提到的其它的属性）

XML中，授权类型在`authorization-server`的子节点中配置

### 配置端点URL

`AuthorizationServerEndpointsConfigurer`有`pathMapping()`方法，它有两个参数：

- 端点的默认URL路径（框架已实现）
- 客户自定义路径（以“/”开头）

框架提供的URL路径是`/oauth/authorize`（鉴权端点），`/oauth/token`（token端点），`/oauth/confirm_access`（用户通过该端口授权），`/oauth/error`（鉴权服务器用于渲染错误），`/oauth/check_token`（资源服务器用来解码访问token）以及`/oauth/token_key`（如果使用JWT token，则用于暴露token验证的public key）

注意：鉴权端点`/oauth/authorize`（或者它的其它映射）需要通过Spring Security来保护，以便它只能被授权的用户访问。例如使用标准的Spring Security的 `WebSecurityConfigurer`：

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

> 注意：如果你的鉴权服务器也是资源服务器，那么还有一个低优先级的安全过滤器控制api资源。对于这些需要通过访问token来保护的请求，你需要让它们的路径在面对用户的过滤器链中被过滤，因此确保包括一个请求匹配器，将`WebSecurityConfigurer`配置的非API资源的请求挑出来

token端点默认由Spring OAuth通过`@Configuration`配置，通过使用HTTP Basic鉴权客户密钥来保护。在xml中不是这样（因此需要显示配置）

在xml中，`<authorization-server/>`元素有一些属性可以用来改变URL的默认端点。`/check_token`端点需要显示地设置为可用（通过`check-token-enabled`属性）

### 定制UI

大部分鉴权服务器端点主要是被机器使用，但也有几个资源需要用到UI，它们是对`/oauth/confirm_access`的GET请求以及`/oauth/error`的HTML响应。在框架中，它们通过whitelabel实现的方式被提供，因此鉴权服务器的大部分真实实例需要提供自己的实现以便他们能控制样式和内容。你所需要做的是对那些端点通过`@RequestMappings`提供一个Spring MVC控制器，框架默认实现的优先级在dispatcher中就会降低。在`/oauth/confirm_access`端点中，你可以使用`AuthorizationRequest` 绑定到session上来携带所有需要查找用户同意的信息（默认实现是`WhitelabelApprovalEndpoint` ，可以参考）。你可以从request中获取所有数据，渲染成你想要的，然后所有的用户需要做的是POST请求到`/oauth/authorize`，带上同意或拒绝授权的信息。请求参数被直接传递到`AuthorizationEndpoint` 中的`UserApprovalHandler` ，因此你可以解析数据。默认的`UserApprovalHandler` 依赖于是否你在`AuthorizationServerEndpointsConfigurer` 中提供了`ApprovalStore`（此时是`ApprovalStoreUserApprovalHandler`）还是没有提供（此时是`TokenStoreUserApprovalHandler`）。标准的同意处理器接受如下：

- TokenStoreUserApprovalHandler：简单的是否决定权，通过比较`user_oauth_approval` 是否为true或false判断
- ApprovalStoreUserApprovalHandler：`scope.\*`参数集合，\*与请求的scope相同。参数值可以是true或approved（如果用户同意授权），否则用户被认为拒绝了那个scope。至少一个scope被同意授权就成功

> 注意：不要忘记在渲染给用户的form中包含CSRF保护。Spring Security默认会使用请求参数_csrf（在请求属性中包含该值）。更多信息可以参考Spring Security用户手册，或者参见whitelabel 实现

### 强制SSL

普通HTTP对测试来说很好，但是在生产环境中鉴权服务器应当使用SSL。可以让应用运行在安全的容器中或者在代理后。如果你正确设置了代理和容器的话，应该会运行得很好（和OAuth 2没有什么关系）。你也可以用Spring Security `requiresChannel()` 保护端点。对于`/authorize`端点来说需要你的应用安全是你自己的事情。对于`/token`端点来说，在`AuthorizationServerEndpointsConfigurer` 中有一个标志，你可以通过`sslOnly()`方法来设置。在这两种情况下，安全通道的设置是可选的，但是如果Spring Security在非安全的通道中探测到请求时，它会重定向到它认为的安全通道。

### 定制错误处理

鉴权服务器的错误处理使用标准的Spring MVC特性，即在端点上申明 `@ExceptionHandler`的方法。用户也可以为端点提供`WebResponseExceptionTranslator` ，这是一个修改响应内容而不是它们渲染方式的好方法。异常的渲染在token端点的情况下委派给`HttpMesssageConverters` （可以添加到MVC配置中），在鉴权端点的情况下委派给OAuth错误页面（/oauth/error）。whitelabel 错误端点用于HTML响应，但用户可以提供一个定制的实现（例如增加`@Controller`和`@RequestMapping("/oauth/error")`）

### 映射用户角色到Scope

有时候限制token的scope是有用的，限制方法可以是通过委派给客户的scope，也可以根据用户自己的权限限制。如果在`AuthorizationEndpoint` 中使用`DefaultOAuth2RequestFactory`，你可以设置`checkUserScopes=true`来限制那些允许的scope给匹配到角色的用户。也可以在`TokenEndpoint` 中注入`OAuth2RequestFactory` （例如使用password授权），这只有在有`TokenEndpointAuthenticationFilter` 时才有效，你仅仅需要在HTTP `BasicAuthenticationFilter`后增加它即可。当然，你也可以实现你自己的规则来将scope和角色对应起来，使用你自己的`OAuth2RequestFactory`。`AuthorizationServerEndpointsConfigurer` 允许你注入定制的`OAuth2RequestFactory` ，这样如果使用`@EnableAuthorizationServer`你就可以使用这个属性设置一个factory。

### 资源服务器配置

资源服务器（可以和鉴权服务器相同，也可以是一个单独的应用）服务的是由OAuth2 token保护的资源。Spring OAuth是通过提供Spring Security鉴权过滤器来实现的。可以通过在配置了`@Configuration` 的类上设置`@EnableResourceServer` 来打开，然后使用`ResourceServerConfigurer`来配置（必须配置）。如下的属性可以配置：

- tokenServices：定义token服务的bean（`ResourceServerTokenServices`实例）
- resourceId：资源id（可选，但建议配置。如果存在，则鉴权服务器会验证）
- 其它资源服务器的扩展点（如`tokenExtractor`用于从请求中抽取token）
- 匹配受保护资源的请求（默认是所有请求）
- 受保护资源的访问规则（默认是"authenticated"）
- 其它为受保护资源的定制内容，且被Spring Security中`HttpSecurity` 配置所允许

`@EnableResourceServer` 注解自动在Spring Security过滤器链中增加了`OAuth2AuthenticationProcessingFilter` 过滤器

在XML中，`<resource-server/>` 元素有id属性，这个是servlet 的`Filter` 对应的bean的id，可以手动将它增加到Spring Security过滤器链中。

`ResourceServerTokenServices` 是鉴权服务器的另一半。如果资源服务器和鉴权服务器在同一个应用中，而且使用了`DefaultTokenServices` ，那么不需要过多考虑因为它已经实现了必要的接口。如果资源服务器是独立应用，则需要提供`ResourceServerTokenServices` 来正确解码token。在鉴权服务器上，你可以使用`DefaultTokenServices` ，通常通过`TokenStore` 来设置（后端存储或本地编码）。也可以使用`RemoteTokenServices`（它是Spring OAuth的特性，并不是规范的一部分）让资源服务器通过鉴权服务器上的HTTP响应解码token（`/oauth/check_token`）。如果资源服务器上流量不是很大（每一个请求在鉴权服务器上都会被验证），或者可以缓存结果，那么`RemoteTokenServices` 是非常便利的。要想使用`/oauth/check_token` 端点，需要通过在`AuthorizationServerSecurityConfigurer`中修改访问规则来暴露它（默认是denyAll()）。例如:

```java
@Override
        public void configure(AuthorizationServerSecurityConfigurer oauthServer) throws Exception {
            oauthServer.tokenKeyAccess("isAnonymous() || hasAuthority('ROLE_TRUSTED_CLIENT')").checkTokenAccess(
                    "hasAuthority('ROLE_TRUSTED_CLIENT')");
        }
```

在这个例子中，我们配置了`/oauth/check_token`和`/oauth/token_key` 端点（因此可信资源能够获取用于JWT验证的public key）。通过HTTP Basic鉴权方式使用用户凭据保护这两个端点。

### 配置OAuth感知的表达式处理器

也许想使用Spring Security的 [基于表达式的访问控制](https://docs.spring.io/spring-security/site/docs/3.2.5.RELEASE/reference/htmlsingle/#el-access)。表达式处理器默认在`@EnableResourceServer`启动的时候被注册。表达式包括*#oauth2.clientHasRole*, *#oauth2.clientHasAnyRole*以及*#oath2.denyClient*，它们可以被用于提供基于OAuth客户端角色的方法控制（全部表达式参见`OAuth2SecurityExpressionMethods`）。在XML中，你可以在`<http/>`配置中使用`expression-handler` 元素注册一个OAuth感知的表达式处理器。

## OAuth 2.0客户端

OAuth 2.0客户端机制用于访问OAuth 2.0保护的资源。配置保护建立用户可能会访问的相关保护资源。客户端也需要提供存储鉴权码和用户访问token的机制。

### 受保护资源配置

受保护的资源（或者说远端资源）通过[`OAuth2ProtectedResourceDetails`](https://projects.spring.io/spring-security-oauth2/src/main/java/org/springframework/security/oauth2/client/resource/OAuth2ProtectedResourceDetails.java)类型的bean定义来定义。它有如下属性：

- `id`： 资源id。id只被客户端用于查找资源。它不会在OAuth协议中被使用。也可以用于bean的id
- `clientId`：OAuth的客户端id。它用于OAuth提供者来识别客户端
- `clientSecret`：资源相关的密码。默认的密码为空
- `accessTokenUri`：OAuth提供者用于提供访问token的URI
- `scope`：指定了访问资源的scope，逗号分隔。默认的scope未指定
- `clientAuthenticationScheme`：客户端使用的用于鉴权访问token端点的scheme。建议的值：http_basic和form。默认是http_basic。参见OAuth 2规范的2.1节

不同的授权类型有`OAuth2ProtectedResourceDetails` 的不同实现（如对于client_credentials类型的实现为`ClientCredentialsResource` ）。对于需要用户鉴权的授权类型有其它的属性：

- `userAuthorizationUri`：用户如果需要授权访问资源时，被重定向的URI。注意，它不是必须的，依赖于支持哪种OAuth 2 profile

XML中`<resource/>` 元素用于创建`OAuth2ProtectedResourceDetails`类型的bean。它有上面提到的所有属性

### 客户端配置

对于OAuth 2.0客户端，使用`@EnableOAuth2Client`可以简化配置。它做了两件事情：

- 创建了过滤器bean（id为`oauth2ClientContextFilter`）来存储当前请求和上下文。在请求过程中需要鉴权的情况下，它管理了重定向到OAuth鉴权URI的方向
- 在请求范围内创建了`AccessTokenRequest` 的bean。它用于通过鉴权码（或简化模式）授权的客户端来保存不同用户的状态。

过滤器需要被织入应用（例如对于`DelegatingFilterProxy`来说，通过Servlet初始化器或`web.xml` 配置 ）

`AccessTokenRequest` 能够在`OAuth2RestTemplate` 中像如下这样使用

```java
@Autowired
private OAuth2ClientContext oauth2Context;

@Bean
public OAuth2RestTemplate sparklrRestTemplate() {
    return new OAuth2RestTemplate(sparklr(), oauth2Context);
}
```

OAuth2ClientContext放置在session范围内来独立保存不同用户的状态。如果没有则需要在服务器上管理相同的数据结构，将传入的请求映射给用户，将每一个用户和`OAuth2ClientContext`的实例关联起来。

在XML中，包括`<client/>` 元素和`id` 属性，它是servlet过滤器的bean的id，就像在`@Configuration` 中一样，必须映射到`DelegatingFilterProxy` （有相同的名称）

### 访问保护资源

一旦对所有的资源都进行了配置，你就可以访问那些资源了。推荐的访问方法是通过 [Spring 3中的`RestTemplate`](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/client/RestTemplate.html)。如果要通过用户token（鉴权码方法）使用它，你可以通过`@EnableOAuth2Client` 配置来使用（XML中是`<oauth:rest-template/>`），它创建了请求和session范围的上下文对象以便不同用户的请求不会冲突

作为一个通用准则，web应用不应当使用password授权，因此避免使用`ResourceOwnerPasswordResourceDetails` ，可以用`AuthorizationCodeResourceDetails`来替代。如果你需要通过java客户端来使用密码授权，可以通过配置`OAuth2RestTemplate` 添加凭据到`AccessTokenRequest`（是一个`Map`且生存期短暂）而不是`ResourceOwnerPasswordResourceDetails` （所有访问的token共享）

### 在客户端持久化token

客户端不需要持久化token，但如果客户端app每次重启后不需要获取新的token，那么对用户来说会很友好。 [`ClientTokenServices`](https://projects.spring.io/spring-security-oauth2/src/main/java/org/springframework/security/oauth2/client/token/ClientTokenServices.java)接口定义了需要存储特定用户的OAuth 2.0 token的操作。默认提供了JDBC的实现，但你可以实现自己的存储访问token的服务以及数据库层关联的鉴权实例。如果想要使用这个特性，你需要为`OAuth2RestTemplate`配置`TokenProvider` 。例如：

```java
@Bean
@Scope(value = "session", proxyMode = ScopedProxyMode.INTERFACES)
public OAuth2RestOperations restTemplate() {
    OAuth2RestTemplate template = new OAuth2RestTemplate(resource(), new DefaultOAuth2ClientContext(accessTokenRequest));
    AccessTokenProviderChain provider = new AccessTokenProviderChain(Arrays.asList(new AuthorizationCodeAccessTokenProvider()));
    provider.setClientTokenServices(clientTokenServices());
    return template;
}
```

### 为外部OAuth2提供者定制化客户端

一些外部的OAuth2 提供者（例如Facebook）没有正确实现规范，或者它们实现的是比Spring Security OAuth老的版本。为了在你的客户端中使用这些提供者，你需要适配客户端框架的不同部分。

例如为了使用Facebook，在`tonr2` 应用中有Facebook特性（你需要增加你自己的、有效的客户端id和密码 ，它们在Facebook网站上很容易生成）。

Facebook token响应也包含了token过期时间不兼容的JSON（它们使用`expires` 而不是`expires_in`），所以如果你在你的应用中需要使用过期时间，你需要使用定制化的`OAuth2SerializationService`来解码。
