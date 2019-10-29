# Eureka其它特性

上次记录了eureka最基本的特性，包括服务注册中心以及服务提供者，而且还是单机版本。这次记录一下eureka其它特性。

## Eureka多实例

Eureka不仅可以像上回那样部署为单机版本，也可以启动多个实例，然后相互注册来保证服务的可用性。修改为多实例版本也很简单，而且Eureka默认就是多实例模式。下面通过修改spring-cloud-eureka-server模块来演示。

1. 备份spring-cloud-eureka-server模块的application.yml为application-单机.yml。然后再修改application.yml文件如下

```yaml
server:
  port: 8761

spring:
  profiles: peer1
  application:
    name: eureka-server
    
eureka:
  instance:
    hostname: peer1
  client:
    service-url:
      defaultZone: http://peer2:8762/eureka
---
server:
  port: 8762

spring:
  profiles: peer2
  application:
    name: eureka-server
    
eureka:
  instance:
    hostname: peer2
  client:
    service-url:
      defaultZone: http://peer1:8761/eureka
```

配置了两个profile，并且peer1和peer2相互注册对方为defaultZone

2. 修改host文件，增加配置如下

```
127.0.0.1       peer1 peer2
```

3. 分别启动profile为peer1的EurekaServerApplication和profile为peer2的EurekaServerApplication。然后访问http://peer1:8761和http://peer2:8762
4. 第一点中的配置文件也可以配置为如下格式

```yaml
eureka:
  client:
    service-url:
      defaultZone: http://peer1:8761/eureka,http://peer2:8762/eureka
spring:
  application:
    name: eureka-server
---
server:
  port: 8761

spring:
  profiles: peer1
eureka:
  instance:
    hostname: peer1
---
server:
  port: 8762

spring:
  profiles: peer2

eureka:
  instance:
    hostname: peer2
```
和第一点配置的区别是将defaultZone提出来，这样如果配置三个Eureka服务时，只修改一个地方的defaultZone就可以了

## 使用ip注册到Eureka上而不是hostname

默认情况下，是使用hostname注册到Eureka服务器的。如果hostname无法被java获取时，ip地址就会发送给Eureka服务器。也可以配置优先使用ip地址。示例如下

```yaml
eureka:
  instance:
    prefer-ip-address: true
```

## 保护Eureka（后续需要通过代码演示）

可以通过增加`spring-boot-starter-security`依赖来保护Eureka服务器

## Eureka可配置的内容

前面看到的只是Eureka配置的一部分内容，其它可以配置的内容可以参考 `EurekaInstanceConfigBean` 和 `EurekaClientConfigBean`，其中`EurekaInstanceConfigBean` 对应的是` eureka.instance.* `配置（instance表示将自己注册为一个实例），而`EurekaClientConfigBean`对应的是`eureka.client.*`配置（client表示通过查询注册中心来获取其它服务信息）

## 状态页和健康检查

状态页对应的地址为`/info`，健康检查对应的地址为`/health`。如果使用上下文路径，则需要修改这两个默认的地址

```yaml
#指定上下文路径
server:
  servlet:
    path: eureka
    
eureka:
  instance:
    status-page-url-path: ${server.servlet.path}/info
    health-check-url-path: ${server.servlet.path}/health
```

## 注册安全应用（后续需要查看eureka源码来确认具体的含义）

如果应用通过https访问，需要设置如下标识，说明需要用到安全端口（默认为443）和禁用非安全端口（默认为80）来接收流量（receive traffic）

```yaml
eureka:
  instance:
    secure-port-enabled: true
    non-secure-port-enabled: false
```

## 健康检查

默认情况下，Eureka使用客户端心跳来判断是否客户端还活着。除非特别指定，


 


