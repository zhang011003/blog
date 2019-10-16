## Spring Cloud中hystrix两种隔离策略

hystrix有两种隔离策略：THREAD和SEMAPHORE（参见`ExecutionIsolationStrategy`)，默认为THREAD（具体配置在Spring Cloud文档中没有说明，需要参见`HystrixCommandProperties`）。

两者解释如下：

- THREAD: 在一个独立的线程池中执行`HystrixCommand.run()`方法，并且使用线程池中线程的数量限制并发请求。
- SEMAPHORE: 在调用线程上执行`HystrixCommand.run()`方法，并且使用信号量计数器（semaphore permit count）限制并发请求。

在Spring Cloud的文档中说明的是，在Feign中，如果需要在`RequestInterceptor`中使用`ThreadLocal` 绑定变量，则必须设置线程隔离策略为SEMAPHORE，或者禁用Hystrix。

明白了两种隔离策略就明白为什么官网上要这样说明了，其实就是因为使用THREAD的隔离策略会导致在一个新的线程池上执行，而不同线程之间是无法传递`ThreadLocal` 绑定的变量的。

在网上查了一下，发现是可以解决这个问题的，参见 [Spring Cloud中Hystrix 线程隔离导致ThreadLocal数据丢失（上）](https://blog.csdn.net/gududedabai/article/details/83059226 )和[Spring Cloud中Hystrix 线程隔离导致ThreadLocal数据丢失（下）](https://blog.csdn.net/gududedabai/article/details/83059381)。

以下是对这两篇文章的大概总结。

`ThreadLocal` 只能在线程中保存变量，但如果需要在线程之间传递线程中`ThreadLocal` 保存的变量，可以使用`InheritableThreadLocal`来保存变量。它们两个的基本原理都是在`Thread`中有相应的变量`threadLocals`和`inheritableThreadLocals`，设置了相应的数据后，就会被保存到相应的变量中，而在新线程创建时，原线程中`inheritableThreadLocals`保存的变量就会被复制到新线程中。

使用`inheritableThreadLocal`有一个问题就是它是在新建线程的时候才会被赋值，对于线程池的情况就无能为力了。但是还有另外的解决方案，那就是[transmittable-thread-local](https://github.com/alibaba/transmittable-thread-local)，它解决的问题就是跨线程传递数据。

现在的问题就是改造hystrix，让它使用transmittable-thread-local来创建线程。

这篇文章中给出了两种解决方案，但第一种方法写得比较模糊，也许是太简单了就一笔带过吧。

其实查了一下hystrix的官网，发现hystrix有plugins的概念，其中plugin的类型就有并发策略，具体可以参考官方文档 [hystrix plugins](https://github.com/Netflix/Hystrix/wiki/Plugins)。

官方文档上面说，可以实现`HystrixConcurrencyStrategy` 类来替换默认的并发策略。 

查看`HystrixConcurrencyStrategy`类，发现有两个实现类，自己默认的实现类`HystrixConcurrencyStrategyDefault`以及Spring Cloud的实现类`SecurityContextConcurrencyStrategy`，而`SecurityContextConcurrencyStrategy`只是包装了`HystrixConcurrencyStrategy`类，并没有做其它的事情。再看`SecurityContextConcurrencyStrategy`所在的包，发现同一个包下还有一个类`HystrixSecurityAutoConfiguration`,在这个类里，配置了使用`SecurityContextConcurrencyStrategy`的场景。