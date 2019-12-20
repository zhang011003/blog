# Turbine

之前在Hystrix监控台中看到的数据只有一个单独实例的Hystrix数据，整个系统的健康状态则无法直接查看。Turbine就是用来将所有的`/hystrix.stream`聚合到`/turbine.stream`供用户在Hystrix监控台中查看。运行Turbine只需要在main方法中增加`@EnableTurbine`注解即可。默认配置的属性就是在[the Turbine 1 wiki](https://github.com/Netflix/Turbine/wiki/Configuration-(1.x)中说明的属性。唯一的不同就是`turbine.instanceUrlSuffix`不需要端口附在前面，因为这个会自动处理，除非配置了`turbine.instanceInsertPort=false`。

> 默认情况下，Turbine会通过查找在Eureka上注册的实例的`hostName` 和 `port`来查找`/hystrix.stream`，然后将`/hystrix.stream`附在它后面。如果实例的元数据包含`management.port`，则会用来替换`port`值。默认情况下，`management.port`元数据和`management.port`配置文件属性相同。可以通过如下配置覆盖

```yaml
eureka:
  instance:
    metadata-map:
      management.port: ${management.port:8081}
```

`turbine.appConfig`配置键是turbine用于查找实例的Eureka的serviceId列表。Turbine Stream在Hystrix监控台中使用如下类似的URL：

`https://my.turbine.server:8080/turbine.stream?cluster=CLUSTERNAME`

如果名称是`default`，则cluster参数可以忽略。`cluster`参数必须与`turbine.aggregator.clusterConfig`中的条目匹配。从Eureka返回的值是大写。因此，如果有应用叫做`customers`的注册到Eureka，则下面的例子可用：

```yaml
turbine:
  aggregator:
    clusterConfig: CUSTOMERS
  appConfig: customers
```

如果希望定制哪个cluster名称被Turbine用到（因为不想在`turbine.aggregator.clusterConfig`配置中存储cluster名称），可以提供`TurbineClustersProvider`类型的bean。

`clusterName`可以通过`turbine.clusterNameExpression`配置的SpEL表达式来定制，其中root作为`InstanceInfo`的实例。默认值为`appName`，意味着Eureka的`serviceId`成为cluster key（也就是说，customers的`InstanceInfo`有一个`CUSTOMERS`的`appName`）。不同的例子是`turbine.clusterNameExpression=aSGName`，它从AWS ASG名称中获取cluster名称。下面展示了另一个例子：

```yaml
turbine:
  aggregator:
    clusterConfig: SYSTEM,USER
  appConfig: customers,stores,ui,admin
  clusterNameExpression: metadata['cluster']
```

在这个例子中，服务端的cluster名称从元数据map（metadata map）中获取，期望包含`SYSTEM` 和 `USER`。

如果希望所有应用使用“default” cluster，需要配置字符串文字表达式（包括单引号，如果在YAML中时需要用双引号转义）

```yaml
turbine:
  appConfig: customers,stores
  clusterNameExpression: "'default'"
```

SpringCloud提供了`spring-cloud-starter-netflix-turbine`，包含所有需要的依赖来运行Turbine。为了添加Turbine，创建SpringBoot应用，并增加`@EnableTurbine`注解

> 默认情况下，SpringCloud允许Turbine使用主机和端口来支持单个cluster，单个主机的多个进程。如果希望构建到Turbine中的本地Netflix行为不允许单个cluster，单个主机的多个进程（实例ID的关键是主机名），设置`turbine.combineHostPort=false`

## 集群端点

某些情况下，对其它应用来说知道在Turbine中配置了哪个cluster是有用的。可以使用`/clusters`端点来达到目的，它返回的是配置的cluster的JSON数组

**GET /clusters**

```json
[
  {
    "name": "RACES",
    "link": "http://localhost:8383/turbine.stream?cluster=RACES"
  },
  {
    "name": "WEB",
    "link": "http://localhost:8383/turbine.stream?cluster=WEB"
  }
]
```

该端点可以通过配置`turbine.endpoints.clusters.enabled`为`false`来设置为不可用。

## Turbine Stream

在某些环境中（如PaaS设置中），从所有分布式的Hystrix命令中获取度量值的典型Turbine模型不起作用。这种情况下，需要通过Hystrix命令将测量值推到Turbine中。SpringCloud中可以通过发送消息做到。只需要增加依赖`spring-cloud-netflix-hystrix-stream`以及你选择的`spring-cloud-starter-stream-*`，参考[Spring Cloud Stream documentation](https://docs.spring.io/spring-cloud-stream/docs/current/reference/htmlsingle/)关于broker以及如何配置客户端凭据的详细信息。对于本地broker应该是开箱即用的。

在服务端，创建SpringBoot应用并增加`@EnableTurbineStream`注解。Turbine Stream需要用到Spring Webflux，因此需要引入`spring-boot-starter-webflux`包。当增加`spring-cloud-starter-netflix-turbine-stream`依赖时，`spring-boot-starter-webflux`会默认包含进来。

此时可以将Hystrix控制台指向Turbine Stream服务器而不是单独的Hystrix流。如果Turbine Stream在myhost的8989端口上运行，那么就在Hystrix控制台流输入框中输入`http://myhost:8989 `。回路最前面是各自的`serviceId`，然后是点（`.`），然后是回路名称。

SpringCloud提供了`spring-cloud-starter-netflix-turbine-stream`依赖包，它有所有使用Turbine Stream服务器所需要用到的依赖。可以增加选择的Stream binder，例如`spring-cloud-starter-stream-rabbit`。

Turbine Stream服务器也支持`cluster`参数。不像Turbine服务器，Turbine Stream使用Eureka的serviceId作为cluster名称，而它们不可配置。

如果Turbine Stream服务器在`my.turbine.server`的8989端口上运行，并且有两个Eureka serviceId，`customers`和`products`，下面的URL地址在Turbine Stream服务器可用。`default`和空的cluster名称会提供所有的Turbine Stream服务器接收到的所有度量值。

```textile
https://my.turbine.sever:8989/turbine.stream?cluster=customers
https://my.turbine.sever:8989/turbine.stream?cluster=products
https://my.turbine.sever:8989/turbine.stream?cluster=default
https://my.turbine.sever:8989/turbine.stream
```

可以使用Eureka的serviceId作为cluster名称用于Turbine监控台（或任何兼容的监控台）中。不需要配置Turbine Stream服务器的任何属性，比如`turbine.appConfig`, `turbine.clusterNameExpression`以及`turbine.aggregator.clusterConfig`

> Turbine Stream服务器通过SpringCloud Stream收集所有配置的输入渠道的度量值。意味着不会主动收集每个实例的Hystrix度量值。它只会提供每个实例收集到的输入渠道的的度量值。
