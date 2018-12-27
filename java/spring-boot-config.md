# Spring Boot条件注解配置

之前只知道spring boot提供了条件注解，比如@ConditionalOnBean、@ConditionalOnClass、@ConditionalOnExpression、@ConditionalOnMissingBean。今天又发现了一些之前没有注意的地方，记录一下

1. 查看条件注解哪些启用

   这个可以通过在配置文件中增加debug=true来查看。之前理解的debug=true只是增加了打印日志。现在发现debug=true还会告诉你哪些条件注解起了作用。

2. @ConditionalOnMissingClass注解的使用

   @ConditionalOnMissingClass意思是如果类加载器中不存在指定类，则被修饰的类会注册到spring容器中，这个注解的参数不能只用类名，还需要写包名。这个是显而易见的，但有时候会忽略

3. 条件注解的先后执行顺序

   如果多个条件注解都生效，那最终容器使用的是哪个条件注解所修饰的类呢？经过测试，发现是跟注解修饰的类名称有关系。所以如果有多个注解都生效的情况发生，那按照字母排序靠后的类最终会被注册到spring容器中

4. 



