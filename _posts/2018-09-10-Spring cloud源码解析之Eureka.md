---
title: Eureka源码解析（未完成）
description: 通过本文可以获悉Spring注册服务到Eureka的整个过程，了解Eureka运行机制，文末附RPC应用注册到Eureka案例。
categories: SpringCloud, Eureka
tags: SpringCloud, Eureka
---

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

### 服务发现

## 注册中心
### 启动过程

### <span id="serviceRegist">服务注册

### 故障处理

## RPC注册到Eureka
### Dubbo

### Sofa

## 总结