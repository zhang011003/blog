# hs-web中比较重要的几个类

## AuthorizationArgumentResolver

org.hswebframework.web.starter.resolver.AuthorizationArgumentResolver类在hsweb-spring-boot-starter模块下。

主要作用：自动注入参数Authentication。任意方法可以增加Authentication参数，获取当前登录用户的权限信息，包括用户的基本信息,角色,权限集合等常用信息，获取不到则抛出UnAuthorizedException异常

## JsonParamResolver

org.hswebframework.web.starter.resolver.JsonParamResolver类在hsweb-spring-boot-starter模块下。

主要作用：自动注入注解为JsonParam的参数。从request中获取@JsonParam注解值对应的值，使用FastJsonGenericHttpMessageConverter类将获取到的值解析为@JsonParam注解的类实例

## WebUserTokenInterceptor

org.hswebframework.web.authorization.basic.web.WebUserTokenInterceptor类在hsweb-authorization-basic模块下。

主要作用：使用userTokenParser解析token。判断token是否过期。如果旧token过期，则先注销旧token，生成新token，更新请求次数统计与最后请求时间，并设置到UserTokenHolder中。

## TwoFactorHandlerInterceptorAdapter

org.hswebframework.web.authorization.basic.twofactor.TwoFactorHandlerInterceptorAdapter类在hsweb-authorization-basic模块下。

主要作用：获取方法注解@TwoFactor。后续再补充

## CorsAutoConfiguration

org.hswebframework.web.starter.CorsAutoConfiguration类在hsweb-spring-boot-starter模块下。

主要作用：设置跨域访问

## RestControllerExceptionTranslator

org.hswebframework.web.starter.RestControllerExceptionTranslator类在hsweb-spring-boot-starter模块下。

主要作用：统一定义系统中各个异常抛出时返回的状态

## DefaultAuthorizingHandler

org.hswebframework.web.authorization.basic.handler.DefaultAuthorizingHandler类在hsweb-authorization-basic模块下。

主要作用：对权限进行验证，包括角色权限和数据权限。具体的权限验证逻辑都在该类中定义。

## AopAuthorizingController

org.hswebframework.web.authorization.basic.aop.AopAuthorizingController类在hsweb-authorization-basic模块下。

主要作用：每次方法调用时对角色权限进行验证，对数据权限进行过滤。定义角色权限和数据权限的先后验证顺序。







