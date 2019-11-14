# Feign其它特性

## 覆盖Feign默认属性

每个Feign客户端是组件集合的一部分，这些组件共同完成与服务器的交互。 这些组件的名字就是在 `@FeignClient `声明时给定的名字。 Spring Cloud 通过 ` FeignClientsConfiguration `将这些命名的集合创建为 `ApplicationContext`，它们包括 `feign.Decoder`，`feign.Encoder`和 `feign.Contract`。

 Spring Cloud 支持通过对 `@FeignClient` 进行额外配置让用户对Feign 客户端进行完全控制。

```java
@FeignClient(name = "stores", defaultConfiguration = FooConfiguration.class)
public interface StoreClient {
    //..
}
```

这个例子中客户端中的组件由 `FeignClientsConfiguration`  和`FooConfiguration`共同指定，其中后者优先级高。

>  `FooConfiguration`  不需要增加 `@Configuration` 的注解。如果有注解的话，需要在` @ComponentScan `中将其排除，否则该类中定义的 `feign.Decoder`, `feign.Encoder`, `feign.Contract` 会成为默认的组件。可以通过将其放在单独的包中避免被 `@ComponentScan`或 `@SpringBootApplication` 扫描到，或者通过` @ComponentScan `进行排除

`name`和`url`支持占位符

```java
@FeignClient(name = "${feign.name}", url = "${feign.url}")
public interface StoreClient {
    //..
}
```

 Spring Cloud 默认提供的feign用到的bean

参考代码 `org.springframework.cloud.openfeign.FeignClientsConfiguration`

| BeanType        | beanName      | 类名                                                         |
| --------------- | ------------- | ------------------------------------------------------------ |
| `Decoder`       | feignDecoder  | `ResponseEntityDecoder`(包装了`SpringDecoder`)               |
| `Encoder`       | feignEncoder  | `SpringEncoder`                                              |
| `Logger`        | feignLogger   | `Slf4jLogger`                                                |
| `Contract`      | feignContract | `SpringMvcContract`                                          |
| `Feign.Builder` | feignBuilder  | `HystrixFeign.Builder`                                       |
| `Client`        | feignClient   | 如果Ribbon可用，则为`LoadBalancerFeignClient`,否则为默认的Feign client |

 OkHttpClient 和ApacheHttpClient  可以通过设置 `feign.okhttp.enabled` 或`feign.httpclient.enabled` 为`true` 来设置是否可用，并且它们在类路径上。(参考代码org\springframework\cloud\openfeign\FeignAutoConfiguration.java)

SpringCloud默认并没有提供下面的bean，但在创建feign客户端时仍然去查找这些bean

- `Logger.Level`
- `Retryer`
- `ErrorDecoder`
- `Request.Options`
- `Collection`
- `SetterFactory`

创建这些类型的bean并把它放在` @FeignClient `配置中（如上面的 `FooConfiguration`  ），这样可以覆盖之前描述的bean

```java
@Configuration
public class FooConfiguration {
    @Bean
    public Contract feignContract() {
        return new feign.Contract.Default();
    }

    @Bean
    public BasicAuthRequestInterceptor basicAuthRequestInterceptor() {
        return new BasicAuthRequestInterceptor("user", "password");
    }
}
```

` @FeignClient `也可以通过属性文件来配置

```yaml
feign:
  client:
    config:
      feignName:
        connectTimeout: 5000
        readTimeout: 5000
        loggerLevel: full
        errorDecoder: com.example.SimpleErrorDecoder
        retryer: com.example.SimpleRetryer
        requestInterceptors:
          - com.example.FooRequestInterceptor
          - com.example.BarRequestInterceptor
        decode404: false
        encoder: com.example.SimpleEncoder
        decoder: com.example.SimpleDecoder
        contract: com.example.SimpleContract
```

可以通过指定` @EnableFeignClients `的 `defaultConfiguration`  属性来配置所有feign客户端。

默认配置也可以通过如下方式配置，只需要将feinName修改为`default`即可

```yaml
feign:
  client:
    config:
      default:
        connectTimeout: 5000
        readTimeout: 5000
        loggerLevel: basic
```

如果` @Configuration `方式和属性配置方式都有，默认使用属性配置。也可以通过` feign.client.default-to-properties`为`false`来修改。

> 如果需要 `ThreadLocal`  绑定变量到` RequestInterceptor `,需要设置 Hystrix  线程隔离为`SEMAPHORE`(`hystrix.command.default.execution.isolation.strategy=SEMAPHORE`)或者设置feign中的Hystrix  不可用(`feign.hystrix.enabled=false`)
>
> > 这个具体原因在SpringCloud Feign的对应文档中没有给出具体解释，但之前有做过分析，可参考  [Spring Cloud中hystrix两种隔离策略](..\spring-hystrix-isolate.md)  

