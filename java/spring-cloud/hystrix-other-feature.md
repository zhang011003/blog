# Hystrix其它特性

在底层服务上的调用失败可以引起到用户层的层级失败。当调用一个服务在`metrics.rollingStats.timeInMilliseconds`（默认10秒）内超出了`circuitBreaker.requestVolumeThreshold`（默认20个请求），并且失败率超过了`circuitBreaker.errorThresholdPercentage`（默认>50%）时，断路器就会打开且不会再进行调用。当出现错误且断路器打开时，开发人员可以提供一个默认的方法调用。
![HystrixFallback](../../screenshot/spring-cloud/HystrixFallback.png)

断路器的主要目的就是阻止层级调用失败并且允许过载或失败的服务有时间恢复。默认的方法调用可以是另一个Hystrix保护的调用，静态数据或者有意义的空值。默认的方法调用可以是链式的，这样第一个方法调用可以做一些其它的业务调用，其它的则可以调用静态数据。

## 传递安全上下文或使用Spring Scope

如果你想让线程本地上下文传递到`@HystrixCommand`中，默认的声明是不起作用的，因为默认是在线程池中执行的（以防超时）。可以通过配置或注解修改隔离策略，将Hystrix切换为使用相同的线程。参考下面的例子

```java
@HystrixCommand(fallbackMethod = "stubMyService",
    commandProperties = {
      @HystrixProperty(name="execution.isolation.strategy", value="SEMAPHORE")
    }
)
```

如果使用`@SessionScope`或`@RequestScope`时，也会增加同样的声明。如果报错说找不到范围上下文，也需要使用相同的线程。

也可以设置`hystrix.shareSecurityContext`为`true`。其实是自动配置了Hystrix并发策略插件，将在主线程中的`SecurityContext`转移到使用Hystrix command的线程中。Hystrix不允许注册多个并发策略，但可以通过声明自己的Spring bean`HystrixConcurrencyStrategy`来做到。Spring Cloud通过查找上下文来发现你的实现，然后将它包装成自己的插件。

## 健康指标

连接的断路器状态通过`/health`来暴露

```json
{
    "hystrix": {
        "openCircuitBreakers": [
            "StoreIntegration::getStoresByLocationLink"
        ],
        "status": "CIRCUIT_OPEN"
    },
    "status": "UP"
}
```

## Hystrix监控流(Hystrix Metrics Stream)

想要使用Hystrix监控流，需要包含依赖`spring-boot-starter-actuator`以及设置`management.endpoints.web.exposure.include: hystrix.stream`。这样就可以暴露监控端点`/actuator/hystrix.stream`

```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
```

## Hystrix监控台(Hystrix Dashboard)

Hystrix最大的优势就是收集每一个HystrixCommand的指标信息。Hystrix监控台显示了每一个断路器的健康信息。

![](../../screenshot/spring-cloud/Hystrix-dashboard.png)

要想引入Hystrix监控台，增加如下依赖(不同版本的Spring Cloud引用依赖的groupId和artifactId不一样)

```xml
 <dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
 </dependency>
```

想要运行Hystrix监控台，需要在Spring Boot主类中增加注解`@EnableHystrixDashboard`，然后访问`/hystrix`，指定到一个独立实例的`/hystrix.stream`

> 当使用HTTPS连接到`/hystrix.stream`端点时，在服务端使用的证书必须要被JVM所信任。如果证书不被信任，需要将其倒入到JVM中。

## Hystrix超时以及Ribbon客户端

当使用包裹了Ribbon客户端的Hystrix命令时，你需要确保Hystrix超时配置大于Ribbon客户端的超时，包括各种可能的重试。比如如果Ribbon连接超时是1秒，Ribbon客户端可能会重试三次请求，那么Hystrix超时就需要大于三秒。
