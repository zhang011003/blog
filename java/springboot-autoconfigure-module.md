# SpringBoot的`spring-autoconfigure-metadata.properties`配置文件作用

之前一直不清楚SpringBoot的jar包中的`spring-autoconfigure-metadata.properties`的作用，今天查了一下官方文档，发现是有介绍的。记录一下。

-------

`autoconfigure`模块包含了库启动必须的所有信息。也可以包含配置key的定义（如`@ConfigurationProperties`）以及任何用于定制组件如何初始化的回调接口。

> 最好标识对库的依赖为可选，这样你就可以更加容易引入`autoconfigure`模块。如果你这样做了（指标识对库的依赖为可选），如果库没有提供，那么SpringBoot会使用默认的方式。

SpringBoot使用注解处理器从元数据文件（metadata file）中收集自动配置条件（`META-INF/spring-autoconfigure-metadata.properties`）。如果文件存在，则用它来过滤不匹配的自动配置，这样可以提高启动时间。建议添加如下包括自动配置的依赖。

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-autoconfigure-processor</artifactId>
	<optional>true</optional>
</dependency>
```
-------

SpringBoot中关于`spring-autoconfigure-metadata.properties`文件作用的描述就上面一点点，看了和没看一个样。看来只能通过查看源代码的方式看它的作用了

其实关于这个文件，在`@SpringBootApplication`注解中就有看到，只不过当时没有注意而已。这里接着[@SpringBootApplication注解](spring-boot-application.md)中`AutoConfigurationImportSelector`类那里来写。

`AutoConfigurationImportSelector`类中`selectImports`方法如下：

```java
public String[] selectImports(AnnotationMetadata annotationMetadata) {
	if (!isEnabled(annotationMetadata)) {
		return NO_IMPORTS;
	}
	AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
			.loadMetadata(this.beanClassLoader);
	AnnotationAttributes attributes = getAttributes(annotationMetadata);
	List<String> configurations = getCandidateConfigurations(annotationMetadata,
			attributes);
	configurations = removeDuplicates(configurations);
	Set<String> exclusions = getExclusions(annotationMetadata, attributes);
	checkExcludedClasses(configurations, exclusions);
	configurations.removeAll(exclusions);
	configurations = filter(configurations, autoConfigurationMetadata);
	fireAutoConfigurationImportEvents(configurations, exclusions);
	return StringUtils.toStringArray(configurations);
}
```

其中会调用`AutoConfigurationMetadataLoader`类的`loadMetadata`方法

`loadMetadata`方法如下：
```java
protected static final String PATH = “META-INF/“
			+ “spring-autoconfigure-metadata.properties”;
public static AutoConfigurationMetadata loadMetadata(ClassLoader classLoader) {
		return loadMetadata(classLoader, PATH);
	}

	static AutoConfigurationMetadata loadMetadata(ClassLoader classLoader, String path) {
		try {
			Enumeration<URL> urls = (classLoader != null) ? classLoader.getResources(path)
					: ClassLoader.getSystemResources(path);
			Properties properties = new Properties();
			while (urls.hasMoreElements()) {
				properties.putAll(PropertiesLoaderUtils
						.loadProperties(new UrlResource(urls.nextElement())));
			}
			return loadMetadata(properties);
		}
		catch (IOException ex) {
			throw new IllegalArgumentException(
					“Unable to load @ConditionalOnClass location [“ + path + “]”, ex);
		}
	}
static AutoConfigurationMetadata loadMetadata(Properties properties) {
		return new PropertiesAutoConfigurationMetadata(properties);
	}	
```

`loadMetadata`会加载所有jar包中的`META-INF/spring-autoconfigure-metadata.properties`文件，然后返回`PropertiesAutoConfigurationMetadata`类的实例。

回到`selectImports`方法中，通过调用`getCandidateConfigurations`方法获取所有`META-INF/spring.factories`文件中key为`org.springframework.boot.autoconfigure.EnableAutoConfiguration`的值，将配置了`exclude`和`excludeName`的类排除掉。然后调用`filter`方法

```java
	private List<String> filter(List<String> configurations,
			AutoConfigurationMetadata autoConfigurationMetadata) {
		long startTime = System.nanoTime();
		String[] candidates = StringUtils.toStringArray(configurations);
		boolean[] skip = new boolean[candidates.length];
		boolean skipped = false;
		for (AutoConfigurationImportFilter filter : getAutoConfigurationImportFilters()) {
			invokeAwareMethods(filter);
			boolean[] match = filter.match(candidates, autoConfigurationMetadata);
			...
		}
		...
	}
```

`getAutoConfigurationImportFilters`方法返回的是`META-INF/spring.factories`文件中key为`org.springframework.boot.autoconfigure.AutoConfigurationImportFilter`的值。

`AutoConfigurationImportFilter`的实现类只有一个，`org.springframework.boot.autoconfigure.condition.OnClassCondition`,对应的`match`方法及相关方法如下：

```java
public boolean[] match(String[] autoConfigurationClasses,
			AutoConfigurationMetadata autoConfigurationMetadata) {
		ConditionEvaluationReport report = getConditionEvaluationReport();
		ConditionOutcome[] outcomes = getOutcomes(autoConfigurationClasses,
				autoConfigurationMetadata);
		...
	}
	
	private ConditionOutcome[] getOutcomes(String[] autoConfigurationClasses,
			AutoConfigurationMetadata autoConfigurationMetadata) {
		// Split the work and perform half in a background thread. Using a single
		// additional thread seems to offer the best performance. More threads make
		// things worse
		int split = autoConfigurationClasses.length / 2;
		OutcomesResolver firstHalfResolver = createOutcomesResolver(
				autoConfigurationClasses, 0, split, autoConfigurationMetadata);
		OutcomesResolver secondHalfResolver = new StandardOutcomesResolver(
				autoConfigurationClasses, split, autoConfigurationClasses.length,
				autoConfigurationMetadata, this.beanClassLoader);
		ConditionOutcome[] secondHalf = secondHalfResolver.resolveOutcomes();
		ConditionOutcome[] firstHalf = firstHalfResolver.resolveOutcomes();
		ConditionOutcome[] outcomes = new ConditionOutcome[autoConfigurationClasses.length];
		System.arraycopy(firstHalf, 0, outcomes, 0, firstHalf.length);
		System.arraycopy(secondHalf, 0, outcomes, split, secondHalf.length);
		return outcomes;
	}
```

`org.springframework.boot.autoconfigure.condition.OnClassCondition.StandardOutcomesResolver`的`resolveOutcomes`方法
```java
public ConditionOutcome[] resolveOutcomes() {
			return getOutcomes(this.autoConfigurationClasses, this.start, this.end,this.autoConfigurationMetadata);
	}

	private ConditionOutcome[] getOutcomes(String[] autoConfigurationClasses,
				int start, int end, AutoConfigurationMetadata autoConfigurationMetadata) {
		ConditionOutcome[] outcomes = new ConditionOutcome[end - start];
		for (int i = start; i < end; i++) {
			String autoConfigurationClass = autoConfigurationClasses[i];
			Set<String> candidates = autoConfigurationMetadata
					.getSet(autoConfigurationClass, "ConditionalOnClass");
			if (candidates != null) {
				outcomes[i - start] = getOutcome(candidates);
			}
		}
		return outcomes;
	}
```

整体读下来，发现其实获取的是`META-INF/spring-autoconfigure-metadata.properties`文件中配置了类名.ConditionalOnClass对应的值。

打开spring-boot-actuator-autoconfigure-2.0.9.RELEASE.jar中的`/META-INF/spring-autoconfigure-metadata.properties`文件，随便找一个类的配置，比如`RedisHealthIndicatorAutoConfiguration`，发现这里的配置和class文件中注解其实是一一对应的。 所以感觉`spring-autoconfigure-metadata.properties`应该是工具自动生成出来的。但现在还不知道这个猜测对不对。后续知道了再补充吧