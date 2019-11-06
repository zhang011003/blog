# Eureka监控页面

访问http://localhost:8761就会看到Eureka监控页面，如下所示
![eureka-monitor](../../screenshot/spring-cloud/eureka-monitor.png)

分为四部分，系统状态（System Status）、数据中心备份（DS Replicas）、当前注册到Eureka上的实例（Instances currently registered with Eureka）、通用信息（General Info）、实例信息（Instance Info）

## Eureka监控页面代码

在Service->Endpoints下可以看到我们访问的监控页面对应的方法为：`org.springframework.cloud.netflix.eureka.server.EurekaController#status`

![eureka-endpoints](../../screenshot/spring-cloud/eureka-endpoints.png)

`EurekaController`类的`status`方法如下：

```java
	@RequestMapping(method = RequestMethod.GET)
	public String status(HttpServletRequest request, Map<String, Object> model) {
		populateBase(request, model);
		populateApps(model);
		StatusInfo statusInfo;
		try {
			statusInfo = new StatusResource().getStatusInfo();
		}
		catch (Exception e) {
			statusInfo = StatusInfo.Builder.newBuilder().isHealthy(false).build();
		}
		model.put("statusInfo", statusInfo);
		populateInstanceInfo(model, statusInfo);
		filterReplicas(model, statusInfo);
		return "eureka/status";
	}
```

返回的是`eureka/status`,可以发现`/templates/eureka/`目录下的`status.ftl`文件。这个就是Eureka监控页面对应的代码了

## 系统状态（System Status）

主要是系统的一些状态信息。包括：

* Environment：Eureka服务器环境，默认是test。可以使用如下配置去修改

```yaml
eureka:
  environment: prod
```

在Eureka服务器启动的时候会设置页面上对应的值

对应的代码位置

`org.springframework.cloud.netflix.eureka.server.EurekaServerBootstrap#initEurekaEnvironment`

```java
protected void initEurekaEnvironment() throws Exception {
		log.info("Setting the eureka configuration..");

		String dataCenter = ConfigurationManager.getConfigInstance()
				.getString(EUREKA_DATACENTER);
		if (dataCenter == null) {
			log.info(
					"Eureka data center value eureka.datacenter is not set, defaulting to default");
			ConfigurationManager.getConfigInstance()
					.setProperty(ARCHAIUS_DEPLOYMENT_DATACENTER, DEFAULT);
		}
		else {
			ConfigurationManager.getConfigInstance()
					.setProperty(ARCHAIUS_DEPLOYMENT_DATACENTER, dataCenter);
		}
		String environment = ConfigurationManager.getConfigInstance()
				.getString(EUREKA_ENVIRONMENT);
		if (environment == null) {
			ConfigurationManager.getConfigInstance()
					.setProperty(ARCHAIUS_DEPLOYMENT_ENVIRONMENT, TEST);
			log.info(
					"Eureka environment value eureka.environment is not set, defaulting to test");
		}
		else {
			ConfigurationManager.getConfigInstance()
					.setProperty(ARCHAIUS_DEPLOYMENT_ENVIRONMENT, environment);
		}
	}
```
    
*  Data center：数据中心名称，默认为default。可以使用如下配置去修改

```yaml
eureka:
  datacenter: ds1
```

代码同Environment

* Current time：当前服务器时间
* Uptime：服务器启动时间
* Lease expiration enabled：租期超时是否可用，也就是是否进入自我保护模式。如果为false，表明进入自我保护模式，客户端心跳超时不会被清除。如果为true，表明没有进入自我保护模式，客户端心跳超时后会被清除

从`navbar.ftl`中，可以看到，该值是从`registry.leaseExpirationEnabled`中获取的

```html
      <tr>
        <td>Lease expiration enabled</td>
        <td>${registry.leaseExpirationEnabled?c}</td>
      </tr>
```

`EurekaController`中获取registry的代码如下

```java
	private PeerAwareInstanceRegistry getRegistry() {
		return getServerContext().getRegistry();
	}

	private EurekaServerContext getServerContext() {
		return EurekaServerContextHolder.getInstance().getServerContext();
	}
```

可以查到`PeerAwareInstanceRegistryImpl`代码中对于的方法

```java
    public boolean isLeaseExpirationEnabled() {
        if (!isSelfPreservationModeEnabled()) {
            // The self preservation mode is disabled, hence allowing the instances to expire.
            return true;
        }
        return numberOfRenewsPerMinThreshold > 0 && getNumOfRenewsInLastMin() > numberOfRenewsPerMinThreshold;
    }
```

如果自我保护模式不可用，表明Eureka服务端会清除超时的客户端信息，所以直接返回true，如果自我保护模式可用，需要对`getNumOfRenewsInLastMin()`（上一分钟的续约次数）和`numberOfRenewsPerMinThreshold`（每分钟续约阈值）做比较。如果上一分钟的续约次数大于每分钟续约阈值，表明没有进入保护模式，会清除超时的客户端信息。查找代码，可以发现`numberOfRenewsPerMinThreshold`的计算方式如下

```java
@Override
    public void openForTraffic(ApplicationInfoManager applicationInfoManager, int count) {
        // Renewals happen every 30 seconds and for a minute it should be a factor of 2.
        this.expectedNumberOfRenewsPerMin = count * 2;
        this.numberOfRenewsPerMinThreshold =
                (int) (this.expectedNumberOfRenewsPerMin * serverConfig.getRenewalPercentThreshold());
        logger.info("Got {} instances from neighboring DS node", count);
        logger.info("Renew threshold is: {}", numberOfRenewsPerMinThreshold);
        
        ...        
}
```

30秒续约一次（即心跳周期30秒），所以一分钟要乘2。从打印信息来看，count为注册的实例数量。`count*2`算出的就是`expectedNumberOfRenewsPerMin`（每分钟期望续约数）。`numberOfRenewsPerMinThreshold`（每分钟续约的阈值）即为`expectedNumberOfRenewsPerMin`（每分钟期望续约数）`*` 续约百分比阈值。而续约百分比阈值是通过配置文件配置的，默认值为0.85

```yaml
eureka:
  server:
    renewal-percent-threshold: 0.85
```


* 


