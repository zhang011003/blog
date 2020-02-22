# hs-web中比较重要的几个类

## AuthorizationArgumentResolver

`org.hswebframework.web.starter.resolver.AuthorizationArgumentResolver`类在`hsweb-spring-boot-starter`模块下。

主要作用：自动注入参数`Authentication`。任意方法可以增加`Authentication`参数，获取当前登录用户的权限信息，包括用户的基本信息,角色,权限集合等常用信息，获取不到则抛出`UnAuthorizedException`异常

## JsonParamResolver

`org.hswebframework.web.starter.resolver.JsonParamResolver`类在`hsweb-spring-boot-starter`模块下。

主要作用：自动注入注解为`JsonParam`的参数。从`request`中获取`@JsonParam`注解值对应的值，使用`FastJsonGenericHttpMessageConverter`类将获取到的值解析为`@JsonParam`注解的类实例

## WebUserTokenInterceptor

`org.hswebframework.web.authorization.basic.web.WebUserTokenInterceptor`类在`hsweb-authorization-basic`模块下。

主要作用：使用`userTokenParser`解析`token`。判断`token`是否过期。如果旧`token`过期，则先注销旧`token`，生成新`token`，更新请求次数统计与最后请求时间，并设置到`UserTokenHolder`中。

## TwoFactorHandlerInterceptorAdapter

`org.hswebframework.web.authorization.basic.twofactor.TwoFactorHandlerInterceptorAdapter`类在`hsweb-authorization-basic`模块下。

主要作用：获取方法注解`@TwoFactor`。后续再补充

## CorsAutoConfiguration

`org.hswebframework.web.starter.CorsAutoConfiguration`类在`hsweb-spring-boot-starter`模块下。

主要作用：设置跨域访问

## RestControllerExceptionTranslator

`org.hswebframework.web.starter.RestControllerExceptionTranslator`类在`hsweb-spring-boot-starter`模块下。

主要作用：统一定义系统中各个异常抛出时返回的状态

## DefaultAuthorizingHandler

`org.hswebframework.web.authorization.basic.handler.DefaultAuthorizingHandler`类在`hsweb-authorization-basic`模块下。

主要作用：对权限进行验证，包括角色权限和数据权限。具体的权限验证逻辑都在该类中定义。

## AopAuthorizingController

`org.hswebframework.web.authorization.basic.aop.AopAuthorizingController`类在`hsweb-authorization-basic`模块下。

主要作用：每次方法调用时对角色权限进行验证，对数据权限进行过滤。定义角色权限和数据权限的先后验证顺序。

## SystemInitializeAutoConfiguration

`org.hswebframework.web.starter.SystemInitializeAutoConfiguration`类在`hsweb-spring-boot-starter`模块下。

主要作用：初始化脚本引擎、初始化数据库。详见[hs-web中的数据库初始化](hs-web-classes.md)
