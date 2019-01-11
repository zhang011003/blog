#@SpringBootApplication注解

spring boot项目一般都会写如下代码来启动项目。

```java
@SpringBootApplication  
public class Application {  
    public static void main(String[] args) {  
        SpringApplication.run(Application.class, args);  
    }  
}
```
今天就主要看一下@SpringBootApplication注解的作用。

@SpringBootApplication注解代码

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
}
```
其中的@SpringBootConfiguration就是@Configuration

@EnableAutoConfiguration注解的代码
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
}
```
会导入一个AutoConfigurationImportSelector的类。
AutoConfigurationImportSelector类中selectImports方法其实就是在查找需要导入的@Configuration注解的类。selectImports方法会调用getCandidateConfigurations方法，在这里又会调用SpringFactoriesLoader类的静态方法loadFactoryNames，来获取classloader的jar包中所有META-INF/spring.factories中配置了key为EnableAutoConfiguration的所有类。
打开spring-boot-autoconfigure-x.x.x.jar包，可以看到META-INF/spring.factories中配置了很多，这些在启动的时候都会自动实例化
```
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
...
```
随便打开一个CacheAutoConfiguration类，可以看到这个类有很多的注解

```java
@Configuration
@ConditionalOnClass(CacheManager.class)
@ConditionalOnBean(CacheAspectSupport.class)
@ConditionalOnMissingBean(value = CacheManager.class, name = "cacheResolver")
@EnableConfigurationProperties(CacheProperties.class)
@AutoConfigureAfter({ CouchbaseAutoConfiguration.class, HazelcastAutoConfiguration.class,
		HibernateJpaAutoConfiguration.class, RedisAutoConfiguration.class })
@Import(CacheConfigurationImportSelector.class)
public class CacheAutoConfiguration {
    ...
}
```
SpringBoot中提供了一系列的@ConditionalOnXXX的注解，称为条件注解。对这些条件注解的支持也是SpringBoot的核心思想，正是通过这些条件注解，SpringBoot做到了需要什么功能，就增加什么依赖包这种动态可插拔的机制。


