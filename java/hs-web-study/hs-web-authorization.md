# hs-web权限模块

## 权限管理

权限管理涉及到的模块有hsweb-authorization和hsweb-system-authorization。

在demo中，增加了@EnableAopAuthorize的注解来启动AOP权限控制。

@EnableAopAuthorize注解就定义在hsweb-authorization-basic模块中。该注解导入了两个类：AopAuthorizeAutoConfiguration和AuthorizingHandlerAutoConfiguration。

AuthorizingHandlerAutoConfiguration类中定义了很多的bean，其中就有负责登录的AuthorizationController的bean定义，还定义了两个WebMvcConfigurer的bean，twoFactorHandlerConfigurer只有在配置文件中配置了属性hsweb.authorize.two-factor且enable=true的时候才有效，而另外的webUserTokenInterceptorConfigurer优先级最高，在这个configurer中，增加了一个用户令牌拦截器WebUserTokenInterceptor，它继承自org.springframework.web.servlet.handler.HandlerInterceptorAdapter类，在每次方法调用之前，先将request请求解析成ParsedToken，如果返回为空，则返回true，否则，可能是用户已经登录，先踢出旧的token，再将新的token保存到UserTokenHolder中，其实就是ThreadLocal变量中。

AopAuthorizeAutoConfiguration类定义了两个比较重要的bean，DefaultAopMethodAuthorizeDefinitionParser类用来解析Authorize相关注解，而AopAuthorizingController类继承自org.springframework.aop.support.StaticMethodMatcherPointcutAdvisor，并且实现了org.springframework.boot.CommandLineRunner接口，这个类其实定义了一个切面，如果类定义了@Controller，@RestController注解，且方法定义了@Authorize注解，则会使用DefaultAopMethodAuthorizeDefinitionParser解析这些注解的方法，并返回是否支持。

```java
/**
*Perform static checking whether the given method matches. If this returns false or if the isRuntime() method returns false, no runtime check (i.e. no. matches(java.lang.reflect.Method, Class, Object []) call) will be made.
**/
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
解析完成后，会在run中发布AuthorizeDefinitionInitializedEvent事件。通过搜索，发现在hsweb-system-authorization-starter的AutoSyncPermission类监听了AuthorizeDefinitionInitializedEvent事件，在onApplicationEvent方法中，会获取到所有解析到AuthorizeDefinition实例，然后会创建对应的实体，根据注解@ApiModelProperty设置名称，最后更新到数据库中。

AopAuthorizingController的构造方法传入了一个MethodInterceptor实例，其实就是在mathes方法返回true后动态调用该方法时执行的切面。这个要注意，**matches是静态判断方法是否匹配，如果匹配，则动态调用该方法时MethodInterceptor实例对应的invoke方法就会执行，也就是该类构造方法中传入的lambda表达式。** 在这个lambda表达式中，会判断如果Authentication.current()有值，则获取Authentication实例，否则抛出未授权异常。获取到Authentication实例后，就根据方法定义时的注解进行相应的权限判断。

这里其实我感觉使用Spring security做起来更方便一些，不太清楚作者为何舍弃现有的成熟框架而自己重新发明轮子。

继续看AopAuthorizingController类构造方法的lambda表达式。为了便于说明，先把这部分代码贴出来。

```java
boolean isControl = false;
if (null != definition) {
    Authentication authentication = Authentication.current().orElseThrow(UnAuthorizedException::new);
    //空配置也进行权限控制
//                if (!definition.isEmpty()) {

    AuthorizingContext context = new AuthorizingContext();
    context.setAuthentication(authentication);
    context.setDefinition(definition);
    context.setParamContext(paramContext);
    isControl = true;

    Phased dataAccessPhased = null;
    if (definition.getDataAccessDefinition() != null) {
        dataAccessPhased = definition.getDataAccessDefinition().getPhased();
    }
    if (definition.getPhased() == Phased.before) {
        //RDAC before
        authorizingHandler.handRBAC(context);

        //方法调用前验证数据权限
        if (dataAccessPhased == Phased.before) {
            authorizingHandler.handleDataAccess(context);
        }

        result = methodInvocation.proceed();

        //方法调用后验证数据权限
        if (dataAccessPhased == Phased.after) {
            context.setParamContext(holder.createParamContext(result));
            authorizingHandler.handleDataAccess(context);
        }
    } else {
        //方法调用前验证数据权限
        if (dataAccessPhased == Phased.before) {
            authorizingHandler.handleDataAccess(context);
        }

        result = methodInvocation.proceed();
        context.setParamContext(holder.createParamContext(result));

        authorizingHandler.handRBAC(context);

        //方法调用后验证数据权限
        if (dataAccessPhased == Phased.after) {
            authorizingHandler.handleDataAccess(context);
        }
    }
//                }
}
if (!isControl) {
    result = methodInvocation.proceed();
}
```

1. 有数据访问权限控制的注解
2. 设置权限判断上下文类AuthorizingContext
3. 由于有两种类型的权限：角色权限和数据权限，所以分几种情况调用
    *  调用方法前验证角色权限，调用方法前验证数据权限，方法调用
    *  调用方法前验证角色权限，方法调用，调用方法后验证数据权限
    *  调用方法前验证数据权限，方法调用，调用方法后验证角色权限
    *  方法调用，调用方法后验证角色权限，调用方法前验证数据权限
    
    **需要注意的是：角色权限优先级要高于数据权限**
    
具体角色权限和数据权限的验证逻辑参看[hs-web角色权限和数据权限验证逻辑](hs-web-rbac-data.md)


