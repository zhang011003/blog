# Hystrix是如何工作的

## 流图

![hystrix-command-flow-chart.png](../../screenshot/spring-cloud/hystrix-command-flow-chart.png)

1. 构造 `HystrixCommand`或`HystrixObservableCommand`对象
   
   构造 `HystrixCommand`或`HystrixObservableCommand`对象来代替你发起的请求。将请求需要的参数传递给构造函数。
   
   如果返回单个响应，则构造`HystrixCommand`对象
   
   ```java
   HystrixCommand command = new HystrixCommand(arg1, arg2);
   ```
   
   如果返回的是持续发出响应的Observable对象，则构造`HystrixObservableCommand`对象
   
   ```java
   HystrixObservableCommand command = new HystrixObservableCommand(arg1, arg2);
   ```

2. 执行命令
   
   执行命令有四种方式（前两种适用于`HystrixCommand`，并不适用于`HystrixObservableCommand`）
   
   - `execute()`：返回单一的响应（或当发生错误时抛出异常）之前一直阻塞
   
   - `queue()`：返回`Future`对象，可以使用它来获取单一响应
   
   - `observe()`：订阅代表响应的`Observable`对象，返回`Observable`对象的副本
   
   - `toObservable()`：返回`Observable`对象，当你订阅它时，会执行Hystrix命令并产生响应
     
     ```java
     K             value   = command.execute();
     Future<K>     fValue  = command.queue();
     Observable<K> ohValue = command.observe();         //hot observable
     Observable<K> ocValue = command.toObservable();    //cold observable
     ```
   
   同步方法`execute()`调用`queue().get().queue()`，而它又调用`toObservable().toBlocking().toFuture()`。最终每一个`HystrixCommand`都是使用[`Observable`](http://reactivex.io/documentation/observable.html)的实现，即使是返回单个响应的命令。

3. 响应是否被缓存？
   
   如果命令的请求缓存可用，并且该请求的响应在缓存中可用，那么缓存的响应会立即以`Observable`的形式返回。参见请求缓存。

4. 是否断路器打开？
   
   当执行Hystrix命令时，它会检查断路器是否打开。
   
   如果断路器打开，Hystrix不会执行命令，但会路由到第8点

5. 是否线程池/队列/信号量已满？
   
   如果与命令相关的线程池和队列（或者当不在线程中运行时为信号量）满了时，Hystrix不会执行命令，而是直接路由到第8点

6. `HystrixObservableCommand.construct()`或者`HystrixCommand.run()`
   
   这里Hystrix调用的请求依赖于你写的方法的目的
   
   - `HystrixCommand.run()`：返回单个响应或抛出异常
   
   - `HystrixObservableCommand.construct() `：返回一个Observable对象来产生响应或发送`onError`通知
   
   如果`run()`或者`construct()`方法调用超时，线程将抛出`TimeoutException`异常（如果命令不在自己的线程中运行，则会有单独的定时器线程抛出异常）。在这种情况下Hystrix会通过第8点路由响应，并且抛弃`run()`或者`construct()`方法的最终返回值。
   
   需要注意的是，并没有方法可以强制线程停止。Hystrix能在JVM层做到的是抛出InterruptedException异常。如果由Hystrix包裹的方法不理会InterruptedExceptions异常，则它会继续执行，即使客户端已经收到TimeoutException异常。这种行为会占满Hystrix线程池，即使负载已经降下（This behavior can saturate the Hystrix thread pool, though the load is 'correctly shed'）。大多数Java Http客户端库并不会解析InterruptedExceptions，因此确保正确配置Http客户端连接和读写超时。
   
   如果命令并没有抛出任何异常，并且返回了响应，Hystrix会在记录日志和报告指标后返回响应。如果`run()`或者`construct()`返回的是产生单一响应的`Observable`对象，则会发送`onCompleted`通知。如果是`construct()`方法，Hystrix则返回相同的`Observable`对象

7. 计算回路是否健康
   
   Hystrix会向断路器报告成功、失败、拒绝和超时，而断路器会维护一个滚动的计数器来记录统计信息。
   
   断路器使用这些统计信息来决定是否需要打开，此时会短路任何后续的请求。当恢复周期过后，断路器在首次健康检查通过后又关闭回路。

8. 默认方法调用（Fallback）
   
   当如下几种情况发生时，Hystrix会使用默认调用。异常是由`construct()`or`run()`抛出，命令由于断路器打开而短路，命令线程池和队列或信号量超过最大值时或当命令超时。
   
   编写默认方法来提供从内存中或者通过静态逻辑生成的通用响应，不要依赖网络。如果在默认方法中通过网络调用，你需要再提供另一个`HystrixCommand`或`HystrixObservableCommand`。
   
   如果使用`HystrixCommand`，需要实现`HystrixCommand.getFallback() `并返回单一的默认方法调用值。
   
   如果使用`HystrixObservableCommand`，需要实现`HystrixObservableCommand.resumeWithFallback()`并返回Observable对象生成单个或多个默认方法值。
   
   当默认方法返回响应时，Hystrix会将响应返回给调用者。如果使用`HystrixCommand.getFallback() `，则会返回Observable对象生成值并从方法中返回，如果使用`HystrixObservableCommand.resumeWithFallback()`，则会从方法中返回相同的Observable对象。
   
   如果没有实现默认方法，或者默认方法中抛出异常，Hystrix仍然会返回Observable对象，但不会产生任何值，而且立即终止并生成`onError`通知。`onError`通知会将引起方法调用的异常返回给调用者。（默认方法中调用失败并不是一个好的办法。不应当在默认方法中执行任何可能失败的逻辑）
   
   失败的或不存在的默认方法产生的结果会根据你调用Hystrix命令的不同而改变
   
   - `execute()`— 抛出异常
   
   - `queue()`— 返回`Future`，但当调用`get()`方法时会抛出异常
   
   - `observe()`— 返回`Observable`对象，当订阅它时，会调用订阅者的`onError`方法并立即终止
   
   - `toObservable()`— 返回`Observable`对象，当订阅它时，会调用订阅者的`onError`方法并终止

9. 返回成功响应
   
   当Hystrix命令成功时，会以`Observable`的方式返回响应给调用者。`Observable`在返回之前会进行转化，这个依赖于第2步中调用命令的方式
   
   ![](../../screenshot/spring-cloud/hystrix-return-flow.png)
   
   - `execute()`— 和调用`.queue()`方式一样返回`Future`。当调用`Future`的`get()`方法时返回`Observable`对象生成的单个值
   - `queue()`— 将`Observable`转换为`BlockingObservable`再转换为`Future`并返回该`Future`
   - `observe()`— 立即订阅`Observable`对象并开始了执行命令的流； 返回`Observable`对象，当调用`subscribe`时，重放相应的生成和通知
   - `toObservable()`— 返回未改变的`Observable`对象；必须手工调用`subscribe`才能真正开始执行命令的流

## 序列图

## 断路器

   下图展示了`HystrixCommand`或`HystrixObservableCommand`与[`HystrixCircuitBreaker`](http://netflix.github.io/Hystrix/javadoc/index.html?com/netflix/hystrix/HystrixCircuitBreaker.html)交互的逻辑流程以及相关决策，包括计数器是如何在断路器中发挥作用的。

   ![](../../screenshot/spring-cloud/circuit-breaker-1280.png)

   回路打开和关闭的详细方法如下：

1. 假定回路中的值到了指定的阈值 (`HystrixCommandProperties.circuitBreakerRequestVolumeThreshold()`)...

2. 假定错误率超过了阈值错误百分比 (`HystrixCommandProperties.circuitBreakerErrorThresholdPercentage()`)...

3. 断路器由关闭（`CLOSED`）转换为开启（`OPEN`）。

4. 当断路器开启时，短路所有请求。

5. 指定时间过后(`HystrixCommandProperties.circuitBreakerSleepWindowInMilliseconds()`)断路器会让下一个单个请求通过 (即半开（`HALF-OPEN`）状态)。如果请求失败，断路器再次回到开启（`OPEN`）状态并一直持续睡眠窗口时长（for the duration of the sleep window）。如果请求成功，断路器状态由开启（`OPEN`）转换为关闭（`CLOSED`）并再次执行第1步逻辑。

## 隔离

Hystrix使用隔舱模式（bulkhead pattern）来分离相互依赖并限制并发访问

![](../../screenshot/spring-cloud/soa-5-isolation-focused-640.png)

### 线程和线程池

客户端（包括库，网络调用等）在单独的线程中执行。这样可以将调用线程（tomcat线程池）隔离开来，以便调用者在遇到长时间执行的独立调用时能够做别的事情。

Hystrix使用分离的、独立的线程池来限制给定的依赖，因此当前执行延迟只会占满所在线程池的可用线程。

> **注意**：即使使用线程隔离，代码也需要设置超时以及对线程阻塞要有响应，这样就不会永远阻塞并影响Hystrix线程池

### 信号量

可以使用信号量（或计数器）限制对任何给定依赖的并发调用次数，用于替换线程池/队列。这让Hystrix可以不用线程池也能摆脱复杂，但信号量不支持超时（This allows Hystrix to shed load without using thread pools but it does not allow for timing out and walking away，这句话实在不知道怎么翻译）。如果你信任客户端且你只是想摆脱负载，可以使用这种方式。

`HystrixCommand` 和`HystrixObservableCommand`在两个地方支持信号量：

- FallBack：当Hystrix调用完成默认方法时，总是用这种方式调用Tomcat线程

- Execution：如果设置属性`execution.isolation.strategy` 为 `SEMAPHORE`，Hystrix会使用信号量而不是线程池限制调用命令的并发父线程的数量。

可以配置动态属性定义多少并发线程同时执行来使用信号量。需要限制它们的大小，类似于限制线程池大小（毫秒级的内存调用在配置信号量为1或者2时表现很好，但默认是10）。

> **注意**：如果依赖使用信号量隔离，但是变得延迟，父线程也会被阻塞，直到网络调用超时。

一旦到达限制，信号量拒绝就会启动，但充满信号量的线程还将会被阻塞。

## 请求合并（Request Collapsing）

你会发现`HystrixCommand`有一个请求合并器（[`HystrixCollapser`](http://netflix.github.io/Hystrix/javadoc/index.html?com/netflix/hystrix/HystrixCollapser.html)是抽象父类），通过它能够将多个请求合并为一个单一的后端依赖调用。

下面展示了两种不同的场景

![](../../screenshot/spring-cloud/hystrix-collapser-1280.png)

### 为什么使用请求合并

主要是为了减少执行同步的`HystrixCommand`线程和网络的连接数量。请求合并会自动执行，并不需要开发人员手工调整请求数。

#### 全局上下文（跨所有Tomcat线程）

请求合并最理想的地方是全局应用层，这样用户从任何Tomcat线程中的请求都能合并。

例如，如果`HystrixCommand`被配置用来批量获取电影评分，则当相同JVM中用户线程发起该请求时，Hystrix会将请求添加到合并的网络调用中。

需要注意的是，合并器会将HystrixRequestContext对象传递到合并的网络调用中，因此下游系统可以使用它来作为优化选项来处理。

#### 用户请求上下文（单个Tomcat线程）

如果配置`HystrixCommand`只是为了批处理单个用户的请求，Hystrix可以将请求合并到单个Tomcat线程中

例如，如果用户想要加载300个视频项目的书签，而不是执行300个网络调用，那么Hystrix可以将它们合并为一个请求。

#### 请求合并的代价

使用请求合并的代价是在实际方法调用前增加了延迟。最大的代价是批量窗口的大小。

如果有一个命令执行花费5ms，批量窗口需要10ms，则最坏情况下的执行时间会变为15ms。通常情况下，请求不可能恰好在窗口打开时提交，中位数情况是窗口时间的一半，这个示例中是5ms。

代价是否值得依赖于命令执行时长。高延迟命令不会受到少量额外平均延迟的影响。另外，给定命令的并发量是关键：如果很少超过1到2个请求会批量处理，那使用请求合并就没有必要了。事实上，在单线程顺序调用中，合并会是最大的性能瓶颈，因为每次遍历都会等待10ms的批处理窗口时间。

然而，如果有特殊的命令被大量并发使用，可以数十个甚至几百个调用同时进行，那么增加的吞吐量通常远远超过花费，因为Hystrix降低了需要的线程数以及依赖的网络连接数。

## 缓存请求

`HystrixCommand` 和`HystrixObservableCommand`实现可以定义缓存key，然后使用它在并发感知的方式下消除请求上下文的重复调用。

下面展示了一个调用HTTP请求的例子，该请求中有两个线程在工作：

![](../../screenshot/spring-cloud/hystrix-request-cache-1280.png)

缓存请求优点：

- 不同代码可以执行Hystrix命令，而不用担心重复工作

这在多个开发人员正在实现不同功能的大型代码库中特别有用。

例如，代码中多个需要获取用户的`Account`对象的地方都可以如下方式调用

```java
Account account = new UserGetAccount(accountId).execute();

//or

Observable<Account> accountObservable = new UserGetAccount(accountId).observe();
```

Hystrix的`RequestCache`仅会执行`run()`方法一次，多个执行`HystrixCommand`的线程会收到相同数据，尽管初始化的是不同的实例

- 在一个请求中的数据返回是一致的

同一个请求的第一个响应会缓存，其它后续调用会直接返回，而不是每次执行时返回不同的值。

- 消除重复线程执行

由于请求缓存在`construct()` 或`run()`方法之前执行，Hystrix可以在线程执行前消除重复调用。

如果Hystrix没有实现请求缓存功能，那么每一个命令都需要在`construct` 或`run`方法中自己实现，并且会在线程排队且执行后放入。

## 参考文档

> [How it Works](https://github.com/Netflix/Hystrix/wiki/How-it-Works)
