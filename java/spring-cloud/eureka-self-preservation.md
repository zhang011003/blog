# Eureka自我保护模式

Eureka的自我保护模式算是比较重要的特性，但SpringCloud官方文档却一字未提，也有可能是SpringCloud没有。

## 介绍

如果Eureka检测到大量的客户端未正常与服务器断连，则Eureka会进入自我保护模式（Self Preservation Mode）。

正常情况下，当客户端有连续3次心跳失败时，对服务端来说，该客户端就会进入未知状态，会被服务端后台清除线程所清除。当服务端检测到大于15%的客户端都处于此种状态时，服务端就会进入自我保护模式。

当进入自我保护模式时，服务端会停止清除客户端实例。当心跳数量超过设置的阈值时，服务端会退出自我保护模式。

默认情况下自我保护模式是开启的，也可以设置为关闭。

## 为何要有自我保护模式

主要是解决网络不稳定的问题。如果由于网络不稳定导致Eureka服务器突然无法收到客户端发送的心跳信息，这时不能简单地认为客户端有问题而将客户端在服务器端删除。有了自我保护模式，可以有效降低此类问题的出现。

## 配置自我保护阈值

```yaml
eureka:
  server:
    enable-self-preservation: true
    renewal-percent-threshold: 0.85
    renewal-threshold-update-interval-ms: 15*60*1000
```

enable-self-preservation表示是否开始自我保护，默认为true

renewal-percent-threshold表示当续约数降到指定的百分比下时，如果enable-self-preservation为trie，就开启自我保护模式，也就是说服务器不再清除未知状态的客户端。默认为0.85，即85%

renewal-threshold-update-interval-ms表示renewal-percent-threshold更新的时间间隔。单位为ms，默认为15分钟

参考文档：
> [Server Self Preservation Mode](https://github.com/Netflix/eureka/wiki/Server-Self-Preservation-Mode)

