# Hystrix插件

可以通过实现插件来修改Hystrix行为，或者添加额外的操作。

可以通过[`HystrixPlugins`](http://netflix.github.io/Hystrix/javadoc/index.html?com/netflix/hystrix/strategy/HystrixPlugins.html)方式注册插件。Hystrix 会将注册的插件应用到[`HystrixCommand`](http://netflix.github.io/Hystrix/javadoc/index.html?com/netflix/hystrix/HystrixCommand.html),[`HystrixObservableCommand`](http://netflix.github.io/Hystrix/javadoc/index.html?com/netflix/hystrix/HystrixObservableCommand.html), 以及[`HystrixCollapser`](http://netflix.github.io/Hystrix/javadoc/index.html?com/netflix/hystrix/HystrixCollapser.html)的实现中，覆盖其它所有的插件。

## 插件类型

详细参考[Hystrix Javadoc文档](http://netflix.github.io/Hystrix/javadoc/index.html)

### 事件通知

在`HystrixCommand`和`HystrixObservableCommand`执行过程中发生的事件由[`HystrixEventNotifier`](http://netflix.github.io/Hystrix/javadoc/index.html?com/netflix/hystrix/strategy/eventnotifier/HystrixEventNotifier.html)触发来记录告警和统计收集。

### 度量发布

每一个被捕获的度量实例（例如所有给定`HystrixCommandKey`的`HystrixCommands`）都会查找[`HystrixMetricsPublisher`](http://netflix.github.io/Hystrix/javadoc/index.html?com/netflix/hystrix/strategy/metrics/HystrixMetricsPublisher.html)的实现并进行初始化。

这样每一个实例都有机会接收度量数据对象并启动后端进程去用度量对象做一些操作，比如保存到持久层。

默认实现不做任何发布。

如果想要使用[Servo](https://github.com/Netflix/servo)，一个可以通过各种不同的机制接收数据（比如通过轮询器或JMX）的内存系统，请参考文档[https://github.com/Netflix/Hystrix/tree/master/hystrix-contrib/hystrix-servo-metrics-publisher](https://github.com/Netflix/Hystrix/tree/master/hystrix-contrib/hystrix-servo-metrics-publisher)

### 属性策略

如果你实现了[`HystrixPropertiesStrategy`](http://netflix.github.io/Hystrix/javadoc/index.html?com/netflix/hystrix/strategy/properties/HystrixPropertiesStrategy.html)，你可以完全控制系统的属性定义。

默认实现使用了[Archaius](https://github.com/Netflix/archaius)。

### 并发策略

Hystrix使用了`ThreadLocal`,`Callable`,`Runnable`,`ThreadPoolExecutor`以及`BlockingQueue`的实现作为线程隔离和请求范围的功能。

默认情况下，Hystrix有默认的实现。但是很多情况下（包括Netflix）需要定制化，因此该插件允许注入自定义的实现或装饰行为。

可以实现[`HystrixConcurrencyStrategy`](http://netflix.github.io/Hystrix/javadoc/index.html?com/netflix/hystrix/strategy/concurrency/HystrixConcurrencyStrategy.html)以及如下方法：

- `getThreadPool()`和`getBlockingQueue()`方法直接注入其它实现，或仅仅是一个增加了日志记录和度量的装饰版本。

- `wrapCallable()`方法允许修饰每一个Hystrix执行的`Callable`。对于应用功能依赖`ThreadLocal`状态的系统来说是必须的。包装的`Callable`可以根据需要捕获和复制从父线程到子线程的状态。

- `getRequestVariable()`方法需要[`HystrixRequestVariable<T>`](http://netflix.github.io/Hystrix/javadoc/index.html?com/netflix/hystrix/strategy/concurrency/HystrixRequestVariable.html)的实现，功能类似与`ThreadLocal`，但作用域仅限于请求——在请求的所有线程上都可用。通常仅使用[`HystrixRequestContext`](http://netflix.github.io/Hystrix/javadoc/index.html?com/netflix/hystrix/strategy/concurrency/HystrixRequestContext.html)以及它的[`HystrixRequestVariable`](http://netflix.github.io/Hystrix/javadoc/index.html?com/netflix/hystrix/strategy/concurrency/HystrixRequestVariable.html)的默认实现就足够了。

### 命令执行钩子

[`HystrixCommandExecutionHook`](http://netflix.github.io/Hystrix/javadoc/index.html?com/netflix/hystrix/strategy/executionhook/HystrixCommandExecutionHook.html)的实现让`HystrixInvokable`的生命周期能够被访问（`HystrixCommand`或`HystrixObservableCommand`），这样就可以注入行为，日志，覆盖响应，修改线程状态等。可以通过覆盖下面的一个或多个钩子来实现：

| `HystrixCommandExecutionHook`方法 | Hystrix调用方法的时机                                         |
| ------------------------------- | ------------------------------------------------------ |
| `onStart`                       | 在`HystrixInvokable`开始执行前                               |
| `onEmit`                        | 当`HystrixInvokable`产生一个值时                              |
| `onError`                       | 当`HystrixInvokable`失败并抛出异常时                            |
| `onSuccess`                     | 当`HystrixInvokable`成功完成时                               |
| `onThreadStart`                 | 当`HystrixInvokable`是以线程隔离策略执行的`HystrixCommand`时，在线程启动时 |
| `onThreadComplete`              | 当`HystrixInvokable`是以线程隔离策略执行的`HystrixCommand`时，在线程完成时 |
| `onExecutionStart`              | 当在`HystrixInvokable`中的用户定义当方法开始执行时                     |
| `onExecutionEmit`               | 当在`HystrixInvokable`中的用户定义当方法产生一个值时                    |
| `onExecutionError`              | 当在`HystrixInvokable`中的用户定义的方法失败并抛出异常时                  |
| `onExecutionSuccess`            | 当在`HystrixInvokable`中的用户定义的方法池、成功完成时                   |
| `onFallbackStart`               | 当`HystrixInvokable`调用fallback方法时                       |
| `onFallbackEmit`                | 当`HystrixInvokable`中的fallback方法产生值时                    |
| `onFallbackError`               | 当`HystrixInvokable`中的方法调用失败并抛出异常时，或者当调用一个不存在的方法时       |
| `onFallbackSuccess`             | 当`HystrixInvokable`中的fallback方法成功完成时                   |
| `onCacheHit`                    | 当对`HystrixInvokable`的响应在`HystrixRequestCache`中找到时      |

## 如何使用

当第一次调用`HystrixCommand`时，首先访问插件管理的功能。因为不能在运行时包装插件，因此第一次`HystrixCommand`使用的插件变成了`HystrixCommand`在JVM运行过程中使用的插件。

如果在Archaius中注册了插件，则注册的插件会被使用，否则，Hystrix使用默认插件。插件使用示例地址

[https://github.com/eirslett/pull-request-illustration](https://github.com/eirslett/pull-request-illustration)

如果想在第一次调用`HystrixCommand`时注册插件，可以使用如下类似的代码。

```java
HystrixPlugins.getInstance().registerEventNotifier(ACustomHystrixEventNotifierDefaultStrategy.getInstance());
```

## 参考

> [Hystrix Plugins](https://github.com/Netflix/Hystrix/wiki/Plugins)


