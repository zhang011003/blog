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

## 手工创建Feign客户端

如果上面创建feign客户端的方法都无法使用，就需要用到Feign Builder API来创建，例子如下:

```java
@Import(FeignClientsConfiguration.class)
class FooController {

	private FooClient fooClient;

	private FooClient adminClient;

    	@Autowired
	public FooController(Decoder decoder, Encoder encoder, Client client, Contract contract) {
		this.fooClient = Feign.builder().client(client)
				.encoder(encoder)
				.decoder(decoder)
				.contract(contract)
				.requestInterceptor(new BasicAuthRequestInterceptor("user", "user"))
				.target(FooClient.class, "http://PROD-SVC");

		this.adminClient = Feign.builder().client(client)
				.encoder(encoder)
				.decoder(decoder)
				.contract(contract)
				.requestInterceptor(new BasicAuthRequestInterceptor("admin", "admin"))
				.target(FooClient.class, "http://PROD-SVC");
    }
}
```
> 其中`FeignClientsConfiguration.class`是SpringCloud提供的默认配置
> `PROD-SVC`是客户端希望请求的服务名称
> `Contract`对象定义了接口中有效的注解和值。自动织入的`Contract`bean提供了对SpringMVC注解的支持，而不是默认的Feign的注解。

## Feign对Hystrix的支持

如果Hystrix在类路径上，并且`feign.hystrix.enabled=true`，Feign会将所有方法用断路器包裹。返回`com.netflix.hystrix.HystrixCommand`也可以。这样可以使用响应模式（调用`.toObservable()`或者`.observe()`)或者异步使用（通过调用`.queue()`）

如果不想使用Hystrix，需要创建`Feign.Builder`,scope为prototype
```java
@Configuration
public class FooConfiguration {
    	@Bean
	@Scope("prototype")
	public Feign.Builder feignBuilder() {
		return Feign.builder();
	}
}
```

## Feign Hystrix回退方法

Hystrix支持回退方法，也就是当断路器打开或有错误时的默认代码。如果希望`@FeignClient`支持回退方法，设置`fallback`属性为实现了回退的类名称。也需要将实现定义为Spring的bean

```java
@FeignClient(name = "hello", fallback = HystrixClientFallback.class)
protected interface HystrixClient {
    @RequestMapping(method = RequestMethod.GET, value = "/hello")
    Hello iFailSometimes();
}

static class HystrixClientFallback implements HystrixClient {
    @Override
    public Hello iFailSometimes() {
        return new Hello("fallback");
    }
}
```

如果需要访问触发调用回退方法的原因，可以使用`@FeignClient`的`fallbackFactory`

```java
@FeignClient(name = "hello", fallbackFactory = HystrixClientFallbackFactory.class)
protected interface HystrixClient {
	@RequestMapping(method = RequestMethod.GET, value = "/hello")
	Hello iFailSometimes();
}

@Component
static class HystrixClientFallbackFactory implements FallbackFactory<HystrixClient> {
	@Override
	public HystrixClient create(Throwable cause) {
		return new HystrixClient() {
			@Override
			public Hello iFailSometimes() {
				return new Hello("fallback; reason was: " + cause.getMessage());
			}
		};
	}
}
```

> Feign中使用回退方法实现以及Hystrix的回退方法工作模式有一个限制。回退方法目前不支持返回`com.netflix.hystrix.HystrixCommand`和`rx.Observable`

## Feign和`@Primary`

当使用Feign的Hystrix回退方法时，在`ApplicationContext`中会有多个相同类型的bean。这会导致`@Autowired`无法工作，因为有不止一个bean，或标识为primary的bean。为了能够工作，SpringCloud用`@Primary`标记了所有Feign实例，因此Spring框架知道哪个bean被注入。某些情况，这个不是想要的。可以设置`@FeignClient`的`primary`为false

```java
@FeignClient(name = "hello", primary = false)
public interface HelloClient {
	// methods here
}
```

## Feign继承

```java
public interface UserService {

    @RequestMapping(method = RequestMethod.GET, value ="/users/{id}")
    User getUser(@PathVariable("id") long id);
}
```

```java
@RestController
public class UserResource implements UserService {

}
```

```java
package project.user;

@FeignClient("users")
public interface UserClient extends UserService {

}
```

> 通常不建议服务端和客户端共享接口。会导致紧耦合。并且在SpringMVC当前表单中也不起作用（方法参数不支持映射）。

## Feign请求/响应压缩

可以如下设置

```yaml
feign.compression.request.enabled=true
feign.compression.response.enabled=true
```
Feign请求压缩设置类似于对web server的设置

```yaml
feign.compression.request.enabled=true
feign.compression.request.mime-types=text/xml,application/xml,application/json
feign.compression.request.min-request-size=2048
```

## Feign日志

默认的logger名称是创建feign客户端的全路径。Feign日志只有设置`DEBUG`级别才有效

```yaml
logging.level.project.user.UserClient: DEBUG
```

每个客户端都可以配置`Logger.Level`，可以配置的值有

- NONE，不记录日志（默认）
- BASIC，只记录请求方法和URL以及响应状态和执行时长
- HEADERS，记录的基本信息以及请求和响应头
- FULL，记录了请求和响应的消息头、消息体以及源数据

如下设置`Logger.Level`为`FULL`

```java
@Configuration
public class FooConfiguration {
    @Bean
    Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
}
```


