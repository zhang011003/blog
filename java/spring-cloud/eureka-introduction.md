# Eureka概览

## Eureka介绍

Eureka是基于REST的服务，起初用在AWS云，用于在负载均衡和中间层服务故障切换的情况下进行查找服务。我们把这个服务叫做Eureka服务端。除了Eureka服务端外，Eureka也包括了一个基于java的客户端组件，叫做Eureka客户端。用于和服务器的便捷交互。客户端也包括一个内建的轮询负载均衡器。

## Eureka架构

从官方github上找到的架构图
![Eureka架构](../../screenshot/spring-cloud/eureka_architecture.png)

每一个region有一个Eureka集群。不同region中的Eureka集群无法交互。一个Region中有多个zone，每个zone至少有一个Eureka服务器，用于处理zone失败。

上图为一个region。其中有三个zone，每个zone中有一个Eureka服务器。

Eureka客户端向服务器注册，每30秒向服务器发送心跳来续约。如果一段时间内客户端无法续约，服务器会在90秒内将其从注册信息中删除。注册信息和续约会备份到集群中其它Eureka节点。其它区域的客户端都可以查询注册信息（每30秒一次），用来定位服务（可能在不同的zone中）以及远程调用

## 应用客户端和应用服务器如何交互

可以使用任意的交互技术。Eureka只帮助你找到想要交互的服务信息，但不会强制你在服务之间使用的协议和交互方法。例如：你可以利用Eureka获取目标服务地址，然后使用如thrift、http(s)或其它任何RPC机制作为协议进行交互。

## 非java服务和客户端

对于非java服务，也可以实现Eureka客户端，也可以使用side car，它其实是一个java应用程序，包括一个嵌入的Eureka客户端去处理注册和心跳。非java客户端可以使用Rest接口查询其它服务信息。

## 可靠性

Eureka客户端构建的目的就是用于处理服务端失败的。因为客户端有注册信息的缓存，所以它们可以很好地工作，即使所有服务端都不可用的时候。

如果其它服务端不可用，Eureka服务端也可以很好地运行。即使客户端和服务端网络发生故障，服务端也有内建的弹性机制来防治大规模的中断发生。

## 监控

Eureka使用Servo跟踪大量服务端和客户端信息，用于性能、监控和警告。数据可以在JMX注册信息中查看，也可以导入到其它地方。

## 参考：
> [Eureka-at-a-glance](https://github.com/Netflix/eureka/wiki/Eureka-at-a-glance)


