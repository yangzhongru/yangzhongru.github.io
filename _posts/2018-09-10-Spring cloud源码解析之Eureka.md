---
title: Eureka源码解析
description: 本文基于Spring cloud的Edgware.SR3版本，主要介绍了Eureka客户端和服务端的运行机制，如服务注册、发现、续约等内容。
categories: SpringCloud, Eureka
tags: SpringCloud, Eureka
---

## Region和Zone
Region和Zone是Eureka非常重要的概念，他们起源于AWS，可以理解为集群和特定的机房，比如华东地区集群的A机房和华北地区集群的B机房，在服务注册、获取、心跳的代码中，也可以看到针对Region和Zone做处理的地方，针对开发者而言，更多的注意力应该在于Zone的配置上，Ribbon的默认策略是优先访问相同Zone上的服务，所以可以结合物理架构对Zone进行配置。

## 客户端
### 启动过程
引入了`org.springframework.cloud:spring-cloud-starter-eureka`依赖后，我们只需要在启动类上打上`EnableDiscoveryClient`注解，Spring就会将Controller注册到Eureka注册中心，所以接下来，我们跟踪`EnableDiscoveryClient`注解，来看下启动过程中，Spring都做了什么事情，服务时如何注册到Eureka的。

#### 加载DiscoveryClient
我们依次进入`EnableDiscoveryClient`->`EnableDiscoveryClientImportSelector`实现，可以看到下列代码：
```java
        if (autoRegister) {
			List<String> importsList = new ArrayList<>(Arrays.asList(imports));
			importsList.add("org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationConfiguration");
			imports = importsList.toArray(new String[0]);
		} else {
			……
		}
```  

`autoRegister`是`EnableDiscoveryClient`注解默认为true的属性，所以此处会告知Spring加载`AutoServiceRegistrationConfiguration`类，当我们查看唯一一个引用它的类`AutoServiceRegistrationAutoConfiguration`时会发现，上面已经有了一个一个Import注解:  

```java
@Configuration
@Import(AutoServiceRegistrationConfiguration.class)
@ConditionalOnProperty(value = "spring.cloud.service-registry.auto-registration.enabled", matchIfMissing = true)
public class AutoServiceRegistrationAutoConfiguration {

	@Autowired(required = false)
	private AutoServiceRegistration autoServiceRegistration;

	@Autowired
	private AutoServiceRegistrationProperties properties;

	@PostConstruct
	protected void init() {
		if (autoServiceRegistration == null && this.properties.isFailFast()) {
			throw new IllegalStateException("Auto Service Registration has been requested, but there is no AutoServiceRegistration bean");
		}
	}
}
```

那`EnableDiscoveryClient`注解还有什么用呢？查阅Spring官方文档可以看到下面这句话：
    
    The use of @EnableDiscoveryClient is no longer required. It is enough to just have a DiscoveryClient implementation on the classpath to cause the Spring Boot application to register with the service discovery server.

手动笑Cry，也就是说这个注解已经不是必要的了，我们只需要引入必要依赖和配置，那么Spring自己会自动的完成注册过程。

好在查找`AutoServiceRegistrationAutoConfiguration`的引用，我们可以发现`EurekaClientAutoConfiguration`类，其中就有注册EurekaClient的代码：

```java
        @Bean(destroyMethod = "shutdown")
		@ConditionalOnMissingBean(value = EurekaClient.class, search = SearchStrategy.CURRENT)
		@org.springframework.cloud.context.config.annotation.RefreshScope
		@Lazy
		public EurekaClient eurekaClient(ApplicationInfoManager manager, EurekaClientConfig config, EurekaInstanceConfig instance) {
			manager.getInfo(); // force initialization
			return new CloudEurekaClient(manager, config, this.optionalArgs,
					this.context);
		}
```

接下来的事情就比较简单，我们在此处打一个断点，启动一个Provider或者Consumer应用，就可以看到整个EurekaCilent的加载过程了。  

我们会发现，`CloudEurekaClient`只有在缓存刷新的时候会用到，真正处理注册、续约等功能代码在Netflix包中的`DiscoveryClient`类中，Spring自己定义了一个叫`org.springframework.cloud.client.discovery.DiscoveryClient`的接口(下文无特殊说明，DiscoveryClient均指的Netflix中的类)，用来统一规定服务发现的方法，这个接口的实现类`EurekaDiscoveryClient`持有了Netflix包中的`DiscoveryClient`对象，Spring通过这种方式实现了服务的注册发现功能。  

## 服务注册
下面我们继续看源码，`CloudEurekaClient`类的构造方法调用了`DiscoveryClient`的构造方法，在进行了一系列参数封装、校验后，执行了`initScheduledTasks()`方法，这个方法有两个if判断语句，分别是：  
1. 是否要获取服务注册信息；
2. 是否要注册到Eureka。  

```java
    private void initScheduledTasks() {
        if (clientConfig.shouldFetchRegistry()) {
            // 服务获取定时任务
            int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
            int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
            scheduler.schedule(
                    new TimedSupervisorTask(
                            "cacheRefresh",
                            scheduler,
                            cacheRefreshExecutor,
                            registryFetchIntervalSeconds,
                            TimeUnit.SECONDS,
                            expBackOffBound,
                            new CacheRefreshThread()
                    ),
                    registryFetchIntervalSeconds, TimeUnit.SECONDS);
        }

        if (clientConfig.shouldRegisterWithEureka()) {
            int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
            int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
            logger.info("Starting heartbeat executor: " + "renew interval is: " + renewalIntervalInSecs);

            // 服务续约定时任务
            scheduler.schedule(
                    new TimedSupervisorTask(
                            "heartbeat",
                            scheduler,
                            heartbeatExecutor,
                            renewalIntervalInSecs,
                            TimeUnit.SECONDS,
                            expBackOffBound,
                            new HeartbeatThread()
                    ),
                    renewalIntervalInSecs, TimeUnit.SECONDS);

            // 服务注册线程
            instanceInfoReplicator = new InstanceInfoReplicator(
                    this,
                    instanceInfo,
                    clientConfig.getInstanceInfoReplicationIntervalSeconds(),
                    2); // burstSize

            statusChangeListener = new ApplicationInfoManager.StatusChangeListener() {
                @Override
                public String getId() {
                    return "statusChangeListener";
                }

                @Override
                public void notify(StatusChangeEvent statusChangeEvent) {
                    if (InstanceStatus.DOWN == statusChangeEvent.getStatus() ||
                            InstanceStatus.DOWN == statusChangeEvent.getPreviousStatus()) {
                        // log at warn level if DOWN was involved
                        logger.warn("Saw local status change event {}", statusChangeEvent);
                    } else {
                        logger.info("Saw local status change event {}", statusChangeEvent);
                    }
                    instanceInfoReplicator.onDemandUpdate();
                }
            };

            if (clientConfig.shouldOnDemandUpdateStatusChange()) {
                applicationInfoManager.registerStatusChangeListener(statusChangeListener);
            }
            //启动服务注册
            instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
        } else {
            logger.info("Not registering with Eureka server per configuration");
        }
    }
```  
从代码中可知，`instanceInfoReplicator`实现了服务注册，其run方法如下：
```java
    public void run() {
        try {
            discoveryClient.refreshInstanceInfo();

            Long dirtyTimestamp = instanceInfo.isDirtyWithTime();
            if (dirtyTimestamp != null) {
                discoveryClient.register();
                instanceInfo.unsetIsDirty(dirtyTimestamp);
            }
        } catch (Throwable t) {
            logger.warn("There was a problem with the instance info replicator", t);
        } finally {
            Future next = scheduler.schedule(this, replicationIntervalSeconds, TimeUnit.SECONDS);
            scheduledPeriodicRef.set(next);
        }
    }
```  
走了这么远，终于调用到了`discoveryClient.register()`方法：
```java
    boolean register() throws Throwable {
        logger.info(PREFIX + appPathIdentifier + ": registering service...");
        EurekaHttpResponse<Void> httpResponse;
        try {
            httpResponse = eurekaTransport.registrationClient.register(instanceInfo);
        } catch (Exception e) {
            logger.warn("{} - registration failed {}", PREFIX + appPathIdentifier, e.getMessage(), e);
            throw e;
        }
        if (logger.isInfoEnabled()) {
            logger.info("{} - registration status: {}", PREFIX + appPathIdentifier, httpResponse.getStatusCode());
        }
        return httpResponse.getStatusCode() == 204;
    }
```  
可以看到，Spring实际是通过HTTP请求将`InstanceInfo`信息注册到Eureka的，此对象中包括ip地址、端口、应用名等信息，至于服务端的处理过程，将在[后文](#serviceRegist)叙述。

### 服务续约
进入`DiscoveryClient`的`initScheduledTasks()`方法，下面的这段代码定义了一个定时任务，到Eureke服务端做心跳检测：
```java
            // Heartbeat timer
            scheduler.schedule(
                    new TimedSupervisorTask(
                            "heartbeat",
                            scheduler,
                            heartbeatExecutor,
                            renewalIntervalInSecs,
                            TimeUnit.SECONDS,
                            expBackOffBound,
                            new HeartbeatThread()
                    ),
                    renewalIntervalInSecs, TimeUnit.SECONDS);
```  
其中`renewalIntervalInSecs`参数为发送心跳的时间间隔，默认为30秒。

### 服务发现
同样的，我们可以看到服务发现的定时任务：
```java
                scheduler.schedule(
                    new TimedSupervisorTask(
                            "cacheRefresh",
                            scheduler,
                            cacheRefreshExecutor,
                            registryFetchIntervalSeconds,
                            TimeUnit.SECONDS,
                            expBackOffBound,
                            new CacheRefreshThread()
                    ),
                    registryFetchIntervalSeconds, TimeUnit.SECONDS);
```  

其中`registryFetchIntervalSeconds`参数为拉取服务信息的时间间隔，默认为30秒。

查看`CacheRefreshThread`源码（这个类定义在`DiscoveryClient`中），我们可以看到在这个类的run方法中，调用了注册信息刷新的方法，并且根据`eureka.client.disable-delta`配置来决定是否增量刷新，如果禁用掉这个配置采用全量刷新策略的话，可能会增大网络流量。

`DiscoveryClient`的`boolean fetchRegistry(boolean forceFullRegistryFetch)`方法：
```java
    private boolean fetchRegistry(boolean forceFullRegistryFetch) {
        Stopwatch tracer = FETCH_REGISTRY_TIMER.start();

        try {
            // 如果增量获取被禁用，或第一次加载，则获取全部注册信息
            Applications applications = getApplications();

            if (clientConfig.shouldDisableDelta()
                    || (!Strings.isNullOrEmpty(clientConfig.getRegistryRefreshSingleVipAddress()))
                    || forceFullRegistryFetch
                    || (applications == null)
                    || (applications.getRegisteredApplications().size() == 0)
                    || (applications.getVersion() == -1)) //Client application does not have latest library supporting delta
            {
                //省略日志代码
                getAndStoreFullRegistry();
            } else {
                getAndUpdateDelta(applications);
            }
            applications.setAppsHashCode(applications.getReconcileHashCode());
            logTotalInstances();
        } catch (Throwable e) {
            logger.error(PREFIX + appPathIdentifier + " - was unable to refresh its cache! status = " + e.getMessage(), e);
            return false;
        } finally {
            if (tracer != null) {
                tracer.stop();
            }
        }

        // 刷新缓存，发送心跳
        onCacheRefreshed();

        // 更新实例状态
        updateInstanceRemoteStatus();

        return true;
    }
```

可以看到全量更新的方法`getAndStoreFullRegistry`和增量更新的方法`getAndUpdateDelta`,二者将通过HTTP请求到Eureka注册中心拉取服务信息，并更新本地数据。  
而`onCacheRefreshed()`方法就是前文提到的在`CloudEurekaClient`中重写的缓存刷新方法。

## 注册中心
### 启动过程
查看Eureka服务端的注解`@EnableEurekaServer`,可以看到其加载了`EurekaServerMarkerConfiguration`类，并实例化一个`Marker`对象，查看引用可知，通过Spring的Condition机制触发了类`EurekaServerAutoConfiguration`的加载，而Eureka的加载代码正在其中。  
查阅过客户端的代码可知，Spring通过HTTP请求向Eureka服务端注册服务，所以我们会注意到`EurekaServerAutoConfiguration`中比较显眼的一段代码：
```java
    /**
	 * Construct a Jersey {@link javax.ws.rs.core.Application} with all the resources
	 * required by the Eureka server.
	 */
	@Bean
	public javax.ws.rs.core.Application jerseyApplication(Environment environment,
			ResourceLoader resourceLoader) {
		ClassPathScanningCandidateComponentProvider provider = new ClassPathScanningCandidateComponentProvider(
				false, environment);
		// Filter to include only classes that have a particular annotation.
		provider.addIncludeFilter(new AnnotationTypeFilter(Path.class));
		provider.addIncludeFilter(new AnnotationTypeFilter(Provider.class));
		// Find classes in Eureka packages (or subpackages)
		Set<Class<?>> classes = new HashSet<Class<?>>();
		for (String basePackage : EUREKA_PACKAGES) {
			Set<BeanDefinition> beans = provider.findCandidateComponents(basePackage);
			for (BeanDefinition bd : beans) {
				Class<?> cls = ClassUtils.resolveClassName(bd.getBeanClassName(),
						resourceLoader.getClassLoader());
				classes.add(cls);
			}
		}
		// Construct the Jersey ResourceConfig
		Map<String, Object> propsAndFeatures = new HashMap<String, Object>();
		propsAndFeatures.put(
				// Skip static content used by the webapp
				ServletContainer.PROPERTY_WEB_PAGE_CONTENT_REGEX,
				EurekaConstants.DEFAULT_PREFIX + "/(fonts|images|css|js)/.*");

		DefaultResourceConfig rc = new DefaultResourceConfig(classes);
		rc.setPropertiesAndFeatures(propsAndFeatures);

		return rc;
	}
```
在这段代码中，`EUREKA_PACKAGES`是字符串数组，包括`com.netflix.discovery`和`com.netflix.eureka`两个包路径，通过断点查看加载结果，可以发现Eureka服务端的所有REST请求都在`com.netflix.eureka.resources`包中。

## <span id="serviceRegist">注册服务</span>
接受服务注册的接口就在`com.netflix.eureka.resources`这个包下的`ApplicationResource`中：
```java
    @POST
    @Consumes({"application/json", "application/xml"})
    public Response addInstance(InstanceInfo info,
                                @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication) {
        //  此处省略了参数校验等代码
        registry.register(info, "true".equals(isReplication));
        return Response.status(204).build();  // 204 to be backwards compatible
    }
```
进入register方法的实现查看，可以看到`InstanceRegistry`类先将接收到的信息发布到了其他节点，然后通过`AbstractInstanceRegistry`类的register方法进行注册。  
`AbstractInstanceRegistry`将InstanceInfo数据保存在了`ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry`中，第一层key为应用名，第二层key为instanceId，过程中有一些判断租约是否存在的代码，大家可以自行查看。

服务续约与服务获取功能服务端处理逻辑较少，此处跳过。

## 总结
总体看，一套Eureka集群属于一个Region，不同的机房可以设置不同的Zone，服务通过多机房部署可以实现灾备，当然正常情况下可以优先调用相同Zone下的服务，避免网络消耗。  
Eureka的注册中心、服务提供者、服务消费者都被视为一个服务，通过HTTP请求完成注册、续约、获取等功能，所以Eureka注册中心的多个节点可以相互注册组建集群，实现高可用配置。   