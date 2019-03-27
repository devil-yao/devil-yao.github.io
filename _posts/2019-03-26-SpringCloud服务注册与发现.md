### SpringCloud 注册发现流程

> Spring Cloud微服务注册和发现的流程。Spring Cloud是Spring提出的关于微服务的解决方案，它本身集成了多种配置中心，eureka，dubbo，consul等等。

### 一. Spring Cloud 中的DiscoveryClient

······

```java
public interface DiscoveryClient extends Ordered {

   /**
    * Default order of the discovery client.
    */
   int DEFAULT_ORDER = 0;

   /**
    * A human-readable description of the implementation, used in HealthIndicator.
    * @return The description.
    */
   String description();

   /**
    * Gets all ServiceInstances associated with a particular serviceId.
    * @param serviceId The serviceId to query.
    * @return A List of ServiceInstance.
    */
   List<ServiceInstance> getInstances(String serviceId);

   /**
    * @return All known service IDs.
    */
   List<String> getServices();

   /**
    * Default implementation for getting order of discovery clients.
    * @return order
    */
   @Override
   default int getOrder() {
      return DEFAULT_ORDER;
   }

}
```
DiscoveryClient 里面定义了两个方法getInstances和getServices。getInstances可以通过微服务的Vip来获取微服务节点信息的列表，通过这个方法我们可以实现微服务的负载均衡，获取服务节点的列表然后通过一系列负载均衡的策略：随机，轮询，权重等方式来选取服务节点。getServices方法可以得到所有已注册的服务列表。

### 二. 开启微服务的注解EnableDiscoveryClient

在启动类上添加EnableDiscoveryClient注解，可以将项目作为服务注册到配置中心上去。先来看下EnableDiscoveryClient的代码。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(EnableDiscoveryClientImportSelector.class)
public @interface EnableDiscoveryClient {

   /**
    * If true, the ServiceRegistry will automatically register the local server.
    * @return - {@code true} if you want to automatically register.
    */
   boolean autoRegister() default true;

}
```



Import 可以将一个普通的Java 类引入到Spring中，注册成Spring中的Bean。EnableDiscoveryClientImportSelector 这个类实现了SpringFactoryImportSelector 的工厂类，在这个方法中，Spring Cloud会去加载spring-cloud-eureka-client 目录下的spring.factoies文件，实现服务的自动注册。

```java
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.netflix.eureka.config.EurekaClientConfigServerAutoConfiguration,\
org.springframework.cloud.netflix.eureka.config.EurekaDiscoveryClientConfigServiceAutoConfiguration,\
org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration,\
org.springframework.cloud.netflix.ribbon.eureka.RibbonEurekaAutoConfiguration,\
org.springframework.cloud.netflix.eureka.EurekaDiscoveryClientConfiguration

org.springframework.cloud.bootstrap.BootstrapConfiguration=\
org.springframework.cloud.netflix.eureka.config.EurekaDiscoveryClientConfigServiceBootstrapConfiguration
```

我们可以看到在EurekaClientAutoConfiguration这个自动配置类里面，初始化了eureka的服务信息 ，eureka相关的EurekaClient等等和eureka相关的信息。初始化eurekaClient的时候也同时初始化了获取心跳发送以及服务获取的两个线程。我们可以在CloudEurekaClient里面找到初始化EurekaClient的流程。

refreshRegistry 刷新服务注册信息的方法，默认30秒执行一次.

```yml
eureka:
	server:
 	 peer-eureka-status-refresh-time-interval-ms: 30000
```

```java
void refreshRegistry() {
    try {
        boolean isFetchingRemoteRegionRegistries = isFetchingRemoteRegionRegistries();

        boolean remoteRegionsModified = false;
        // This makes sure that a dynamic change to remote regions to fetch is honored.
        String latestRemoteRegions = clientConfig.fetchRegistryForRemoteRegions();
        if (null != latestRemoteRegions) {
            String currentRemoteRegions = remoteRegionsToFetch.get();
            if (!latestRemoteRegions.equals(currentRemoteRegions)) {
                // Both remoteRegionsToFetch and AzToRegionMapper.regionsToFetch need to be in sync
                synchronized (instanceRegionChecker.getAzToRegionMapper()) {
                    if (remoteRegionsToFetch.compareAndSet(currentRemoteRegions, latestRemoteRegions)) {
                        String[] remoteRegions = latestRemoteRegions.split(",");
                        remoteRegionsRef.set(remoteRegions);
                        instanceRegionChecker.getAzToRegionMapper().setRegionsToFetch(remoteRegions);
                        remoteRegionsModified = true;
                    } else {
                        logger.info("Remote regions to fetch modified concurrently," +
                                " ignoring change from {} to {}", currentRemoteRegions, latestRemoteRegions);
                    }
                }
            } else {
                // Just refresh mapping to reflect any DNS/Property change
                instanceRegionChecker.getAzToRegionMapper().refreshMapping();
            }
        }

        boolean success = fetchRegistry(remoteRegionsModified);
        if (success) {
            registrySize = localRegionApps.get().size();
            lastSuccessfulRegistryFetchTimestamp = System.currentTimeMillis();
        }

        if (logger.isDebugEnabled()) {
            StringBuilder allAppsHashCodes = new StringBuilder();
            allAppsHashCodes.append("Local region apps hashcode: ");
            allAppsHashCodes.append(localRegionApps.get().getAppsHashCode());
            allAppsHashCodes.append(", is fetching remote regions? ");
            allAppsHashCodes.append(isFetchingRemoteRegionRegistries);
            for (Map.Entry<String, Applications> entry : remoteRegionVsApps.entrySet()) {
                allAppsHashCodes.append(", Remote region: ");
                allAppsHashCodes.append(entry.getKey());
                allAppsHashCodes.append(" , apps hashcode: ");
                allAppsHashCodes.append(entry.getValue().getAppsHashCode());
            }
            logger.debug("Completed cache refresh task for discovery. All Apps hash code is {} ",
                    allAppsHashCodes);
        }
    } catch (Throwable e) {
        logger.error("Cannot fetch registry from server", e);
    }
}
```

Renew 方法定义维持心跳的方法，默认也是30秒执行一次

```java
eureka:
  instance:
    lease-renewal-interval-in-seconds: 30
```

```java
boolean renew() {
    EurekaHttpResponse<InstanceInfo> httpResponse;
    try {
        httpResponse = eurekaTransport.registrationClient.sendHeartBeat(instanceInfo.getAppName(), instanceInfo.getId(), instanceInfo, null);
        logger.debug(PREFIX + "{} - Heartbeat status: {}", appPathIdentifier, httpResponse.getStatusCode());
        if (httpResponse.getStatusCode() == Status.NOT_FOUND.getStatusCode()) {
            REREGISTER_COUNTER.increment();
            logger.info(PREFIX + "{} - Re-registering apps/{}", appPathIdentifier, instanceInfo.getAppName());
            long timestamp = instanceInfo.setIsDirtyWithTime();
            boolean success = register();
            if (success) {
                instanceInfo.unsetIsDirty(timestamp);
            }
            return success;
        }
        return httpResponse.getStatusCode() == Status.OK.getStatusCode();
    } catch (Throwable e) {
        logger.error(PREFIX + "{} - was unable to send heartbeat!", appPathIdentifier, e);
        return false;
    }
}
```