# Feign源码学习

因为在使用Feign时，都是会增加`@EnableFeignClients`和`@FeignClient`注解，所以从`@EnableFeignClients`注解看起。

## 注解`@EnableFeignClients`、`@FeignClient`及相关类

从`@EnableFeignClients`注解中可以看到导入`FeignClientsRegistrar`类，`FeignClientsRegistrar`类主要做了两件事，1、注册在`@EnableFeignClients`中指定的默认Configuration；2、注册配置了`@FeignClient`注解的类。

```java
	public void registerBeanDefinitions(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
		registerDefaultConfiguration(metadata, registry);
		registerFeignClients(metadata, registry);
	}
```

注册`@FeignClient`注解时，如果`@EnableFeignClients`中配置了`clients`时，则只注册配置的类，否则会扫描`@EnableFeignClients`中`value`，`basePackages`和`basePackageClasses`指定的包或类，如果三个都没有配置，则扫描注解了`@EnableFeignClients`类所在包且注解了`@FeignClient`的类。

注册客户端配置的代码如下，因为每个`@FeignClient`注解的类，都可以有自己对应的configuration，所以每个Feign客户端都会注册一个configuration。***注意注册的类是`FeignClientSpecification`***：

```java
	private void registerClientConfiguration(BeanDefinitionRegistry registry, Object name,
			Object configuration) {
		BeanDefinitionBuilder builder = BeanDefinitionBuilder
				.genericBeanDefinition(FeignClientSpecification.class);
		builder.addConstructorArgValue(name);
		builder.addConstructorArgValue(configuration);
		registry.registerBeanDefinition(
				name + "." + FeignClientSpecification.class.getSimpleName(),
				builder.getBeanDefinition());
	}
```
注册Feign客户端配置的代码如下，***注意注册的类是`FeignClientFactoryBean`***：

```java
	private void registerFeignClient(BeanDefinitionRegistry registry,
			AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
		String className = annotationMetadata.getClassName();
		BeanDefinitionBuilder definition = BeanDefinitionBuilder
				.genericBeanDefinition(FeignClientFactoryBean.class);
		validate(attributes);
		definition.addPropertyValue("url", getUrl(attributes));
		definition.addPropertyValue("path", getPath(attributes));
		String name = getName(attributes);
		definition.addPropertyValue("name", name);
		definition.addPropertyValue("type", className);
		definition.addPropertyValue("decode404", attributes.get("decode404"));
		definition.addPropertyValue("fallback", attributes.get("fallback"));
		definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
		definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);

		String alias = name + "FeignClient";
		AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();

		boolean primary = (Boolean)attributes.get("primary"); // has a default, won't be null

		beanDefinition.setPrimary(primary);

		String qualifier = getQualifier(attributes);
		if (StringUtils.hasText(qualifier)) {
			alias = qualifier;
		}

		BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
				new String[] { alias });
		BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
	}
```

## 自动配置文件`spring.factories`及相关类

在`@EnableFeignClients`所在的jar包下，会看到`spring.factories`配置文件。主要看`FeignAutoConfiguration`类。

在`FeignAutoConfiguration`中，会自动织入解析`@EnableFeignClients`和`@FeignClient`时注册的`FeignClientSpecification`类。`feignContext`方法主要用来构造`FeignContext`并传入之前注入的`FeignClientSpecification`类。

```java
public FeignContext feignContext() {
		FeignContext context = new FeignContext();
		context.setConfigurations(this.configurations);
		return context;
	}
```

在`FeignContext`的构造方法中，会将Feign默认的配置`FeignClientsConfiguration`类传入。
```java
public FeignContext() {
		super(FeignClientsConfiguration.class, "feign", "feign.client.name");
	}
```

其它注册的就是后续需要用到的各种不同的bean

## 通过工厂bean来获取相应的bean实例

之前提到过已经将`FeignClientFactoryBean`注册到Spring里，后续需要获取Feign客户端实例时，就会调用`FeignClientFactoryBean`的`getObject`方法。

```java
public Object getObject() throws Exception {
		return getTarget();
	}

	/**
	 * @param <T> the target type of the Feign client
	 * @return a {@link Feign} client created with the specified data and the context information
	 */
	<T> T getTarget() {
		FeignContext context = applicationContext.getBean(FeignContext.class);
		Feign.Builder builder = feign(context);

		if (!StringUtils.hasText(this.url)) {
			String url;
			if (!this.name.startsWith("http")) {
				url = "http://" + this.name;
			}
			else {
				url = this.name;
			}
			url += cleanPath();
			return (T) loadBalance(builder, context, new HardCodedTarget<>(this.type,
					this.name, url));
		}
		if (StringUtils.hasText(this.url) && !this.url.startsWith("http")) {
			this.url = "http://" + this.url;
		}
		String url = this.url + cleanPath();
		Client client = getOptional(context, Client.class);
		if (client != null) {
			if (client instanceof LoadBalancerFeignClient) {
				// not load balancing because we have a url,
				// but ribbon is on the classpath, so unwrap
				client = ((LoadBalancerFeignClient)client).getDelegate();
			}
			builder.client(client);
		}
		Targeter targeter = get(context, Targeter.class);
		return (T) targeter.target(this, builder, context, new HardCodedTarget<>(
				this.type, this.name, url));
	}
```
而第一步中获取`Feign.Builder`就是调用`FeignContext`的`getInstance`方法获取的

```java
	protected <T> T get(FeignContext context, Class<T> type) {
		T instance = context.getInstance(this.name, type);
		if (instance == null) {
			throw new IllegalStateException("No bean found of type " + type + " for "
					+ this.name);
		}
		return instance;
	}
```

`FeignContext`的`getInstance`方法会先创建`AnnotationConfigApplicationContext`类实例，再通过`AnnotationConfigApplicationContext`类实例的`getBean`方法获取。

在`createContext`方法中，会注册之前传入的`FeignClientSpecification`以及默认的配置类型`defaultConfigType`，也就是`FeignClientsConfiguration`类，还有`PropertyPlaceholderAutoConfiguration`类

```java
protected AnnotationConfigApplicationContext createContext(String name) {
		AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
		if (this.configurations.containsKey(name)) {
			for (Class<?> configuration : this.configurations.get(name)
					.getConfiguration()) {
				context.register(configuration);
			}
		}
		for (Map.Entry<String, C> entry : this.configurations.entrySet()) {
			if (entry.getKey().startsWith("default.")) {
				for (Class<?> configuration : entry.getValue().getConfiguration()) {
					context.register(configuration);
				}
			}
		}
		context.register(PropertyPlaceholderAutoConfiguration.class,
				this.defaultConfigType);
		context.getEnvironment().getPropertySources().addFirst(new MapPropertySource(
				this.propertySourceName,
				Collections.<String, Object> singletonMap(this.propertyName, name)));
		if (this.parent != null) {
			// Uses Environment from parent as well as beans
			context.setParent(this.parent);
		}
		context.setDisplayName(generateDisplayName(name));
		context.refresh();
		return context;
	}
```

再回到`getTarget`方法里，最终会调用到`loadBalance`方法返回

```java
protected <T> T loadBalance(Feign.Builder builder, FeignContext context,
			HardCodedTarget<T> target) {
		Client client = getOptional(context, Client.class);
		if (client != null) {
			builder.client(client);
			Targeter targeter = get(context, Targeter.class);
			return targeter.target(this, builder, context, target);
		}

		throw new IllegalStateException(
				"No Feign Client for loadBalancing defined. Did you forget to include spring-cloud-starter-netflix-ribbon?");
	}
```

其中的`Targeter`类的bean在`FeignAutoConfiguration`中注入，最终会调用到`ReflectiveFeign`的`newInstance`方法。

```java
public <T> T newInstance(Target<T> target) {
    Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
    Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
    List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();

    for (Method method : target.type().getMethods()) {
      if (method.getDeclaringClass() == Object.class) {
        continue;
      } else if(Util.isDefault(method)) {
        DefaultMethodHandler handler = new DefaultMethodHandler(method);
        defaultMethodHandlers.add(handler);
        methodToHandler.put(method, handler);
      } else {
        methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
      }
    }
    InvocationHandler handler = factory.create(target, methodToHandler);
    T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(), new Class<?>[]{target.type()}, handler);

    for(DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
      defaultMethodHandler.bindTo(proxy);
    }
    return proxy;
  }
```

在这里会看到通过反射的方式创建`@FeignClient`注解的接口