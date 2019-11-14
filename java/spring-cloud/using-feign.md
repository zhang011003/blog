# Feign使用 

## Feign概览

Feign是一个声明式web服务客户端。使用Feign可以让web服务客户端编写更加简单。它包括可插入的注解支持，包括Feign注解和JAX-RS注解。Feign也支持可插入的编码器和解码器。SpringCloud添加了SpringMVC注解的支持，当使用SpringWeb时使用相同的`HttpMessageConverters`。当使用Feign时，SpringCloud集成了Ribbon和Eureka，用来提供负载均衡。

## 创建WEB服务提供端

1. 新建maven工程`spring-cloud-service-provider`（参考[Eureka服务端和客户端的创建](eureka.md)），修改`spring.application.name`为`service-provider`。

2. 创建一个简单的controller

```java
@RestController
public class ServiceProviderController {
    @GetMapping
    public String sayHello() {
        return "hello";
    }
}
```

3. 启动刚刚创建的工程，查看Eureka监控页面，发现刚刚启动的服务已经注册到Eureka服务器上

## 创建WEB服务消费端

 1. 新建maven工程`spring-cloud-service-consumer`，修改`spring.application.name`为`service-consumer`。
 2. 程序启动类上增加`@EnableFeignClients`注解开启Feign的支持。

```java
@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients
public class ServiceConsumerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceConsumerApplication.class, args);
    }
}
```

 3. 增加`ServiceProviderFeignClient`接口，并增加`@FeignClient`注解，其中`value`为我们刚刚启动的服务名。

```java
@FeignClient("service-provider")
public interface ServiceProviderFeignClient {
    @RequestMapping(value = "sayHello", method = RequestMethod.GET)
    String sayHello();
}
```

 4. 创建一个简单的controller提供对service-provider服务的调用。

```java
@RestController
public class ServiceConsumerController {
    @Autowired
    private ServiceProviderFeignClient feignClient;

    @GetMapping("sayHello")
    public String sayHello() {
        return feignClient.sayHello();
    }

}
```

 5. 启动工程`spring-cloud-service-consumer`，查看Eureka监控页面，发现增加了刚刚启动的service-consumer服务。
 6. 浏览器中输入http://localhost:8771/sayHello，调用服务成功。


## 示例代码优化

上面大概介绍了一下Fegin的基本使用，但是有一个问题。上面service-provider服务提供的方法是在service-consumer中编写的。但是在实际使用中，多个不同服务消费方调用同一个服务提供方接口的场景却很多，如果调用代码都是在服务消费方编写，则会有很多重复性的工作。所以我们需要优化上面的代码，将feign声明部分放在服务提供方

1. 



