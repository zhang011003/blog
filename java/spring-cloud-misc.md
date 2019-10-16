# Spring Boot和Spring Cloud FAQ

这里记录一些在使用Spring Boot和Spring Cloud过程中遇到的问题，以便后续再次遇到可以直接查找

## Spring Boot放开Endpoint的发现页（/actuator）

需要在pom中增加如下依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-hateoas</artifactId>
</dependency>
```



## Spring Cloud增加Hystrix监控

之前一直认为使用feign就会默人集成hystrix，然后hystrix相关的信息就都会被包含进来，比如hystrix的监控端点(/hystrix.stream)，但仔细观察启动的页面，发现并没有打印对应信息。查找了半天文档，才发现需要增加`@EnableCircuitBreaker`注解，并且pom中需要增加如下依赖。

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

## Spring Cloud中hystrix两种隔离策略

hystrix有两种隔离策略：THREAD和SEMAPHORE（参见`ExecutionIsolationStrategy`)，默认为THREAD（具体配置在Spring Cloud文档中没有说明，需要参见`HystrixCommandProperties`）。

两者解释如下：

- THREAD: 在一个独立的线程池中执行`HystrixCommand.run()`方法，并且使用线程池中线程的数量限制并发请求。
- SEMAPHORE: 在调用线程上执行`HystrixCommand.run()`方法，并且使用信号量计数器（semaphore permit count）限制并发请求。

在Spring Cloud的文档中说明的是，在Feign中，如果需要在`RequestInterceptor`中需要使用`ThreadLocal` 绑定变量，则必须设置线程隔离策略为SEMAPHORE，或者禁用Hystrix。

具体参考[Spring Cloud中hystrix两种隔离策略](spring-hystrix-isolate.md)
