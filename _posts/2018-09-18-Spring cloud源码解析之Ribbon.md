---
title: Ribbon源码解析（未完成）
description: 本文基于Spring cloud的Edgware.SR3版本，主要介绍了Ribbon的主要功能，生产配置及实现机制。
categories: SpringCloud Ribbon
tags: SpringCloud Ribbon
---
## Ribbon 如何实现负载均衡
在Spring cloud中，如果要通过Ribbon来实现负载均衡，我们需要通过如下代码来声明一个`RestTemplate`的实例：
```java
    @Bean
    @LoadBalanced
    RestTemplate restTemplate(){
        return new RestTemplate();
    }
```

并通过它来调用远程的服务，如：
```java
    @Autowired
    private RestTemplate restTemplate;

    @GetMapping("hello")
    public String helloByRibbon(){
        return restTemplate.getForEntity("http://PROVIDER/hello", String.class).getBody();
    }
```  

以上代码需要注意两个地方，其一是在声明`RestTemplate`实例的时候，我们添加了`@LoadBalanced`注解，其二是在调用的时候，我们在`getForEntity`方法的参数中传输的URL，host是以服务名来替代的，这说明，Ribbon可以通过服务名来自动识别要调用那些服务，并实现负载均衡。  
下面我们依次来看下这两个地方，并了解Ribbon是如何实现负载均衡调用的。

### 启用负载均衡：@LoadBalanced
查看`@LoadBalanced`源码：
```java
@Target({ ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Qualifier
public @interface LoadBalanced {
}
```  

可知，这是一个用于属性、方法、参数上的注解，查看这个类的引用可以找到`LoadBalancerAutoConfiguration`类，其第一个属性就装配了`RestTemplate`对象的集合：
```java
    @LoadBalanced
	@Autowired(required = false)
	private List<RestTemplate> restTemplates = Collections.emptyList();
```

Spring在注入对象的时候，如果目标对象标注了`@Qualifier`或其他被`@Qualifier`修饰的注解，那么在自动注入的时候，就会去查找被相同注解标注的Bean，所以可知，当我们自己声明了用`@LoadBalanced`标注的`RestTemplate`对象后，应用启动时，这个Bean就会被注入到`LoadBalancerAutoConfiguration`类的`restTemplates`属性中。

`LoadBalancerAutoConfiguration`类中还声明了一些类，其中需要关注的是这几个：
```java
    @Bean
	public SmartInitializingSingleton loadBalancedRestTemplateInitializer(
			final List<RestTemplateCustomizer> customizers) {
		return new SmartInitializingSingleton() {
			@Override
			public void afterSingletonsInstantiated() {
				for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
					for (RestTemplateCustomizer customizer : customizers) {
						customizer.customize(restTemplate);
					}
				}
			}
		};
	}

    @Configuration
	@ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
	static class LoadBalancerInterceptorConfig {
		@Bean
		public LoadBalancerInterceptor ribbonInterceptor(
				LoadBalancerClient loadBalancerClient,
				LoadBalancerRequestFactory requestFactory) {
			return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
		}

		@Bean
		@ConditionalOnMissingBean
		public RestTemplateCustomizer restTemplateCustomizer(
				final LoadBalancerInterceptor loadBalancerInterceptor) {
			return new RestTemplateCustomizer() {
				@Override
				public void customize(RestTemplate restTemplate) {
					List<ClientHttpRequestInterceptor> list = new ArrayList<>(
							restTemplate.getInterceptors());
					list.add(loadBalancerInterceptor);
					restTemplate.setInterceptors(list);
				}
			};
		}
	}
```

`LoadBalancerInterceptor`这个类顾名思义可知是负载均衡的拦截器，而`RestTemplateCustomizer`通过实现代码也可以看出来，主要是为了向`RestTemplate`类添加该拦截器的，调用过程就在`SmartInitializingSingleton`类的`afterSingletonsInstantiated`方法中。

至此，我们思路比较明确了，在我们声明了标注过`@LoadBalanced`注解的`RestTemplate`对象后，Spring将其加载到了`LoadBalancerAutoConfiguration`类的`restTemplates`属性中，并加入了`LoadBalancerInterceptor`拦截器，下面我们跟踪一下`RestTemplate`调用链路，来看看这个拦截器如何实现调用端的负载均衡。

### 请求调用入口：RestTemplate
`RestTemplate`提供了很多个请求方法，其中有一些应用场景上的差别，我们可以暂时忽略，这里用之前提到的`getForEntity`方法举例。

依次进入`getForEntity` → `execute` → `doExecute` 方法：
```java
protected <T> T doExecute(URI url, HttpMethod method, RequestCallback requestCallback,
			ResponseExtractor<T> responseExtractor) throws RestClientException {
		……
		ClientHttpResponse response = null;
		try {
			ClientHttpRequest request = createRequest(url, method);
			if (requestCallback != null) {
				requestCallback.doWithRequest(request);
			}
			response = request.execute();
			handleResponse(url, method, response);
			if (responseExtractor != null) {
				return responseExtractor.extractData(response);
			}
			else {
				return null;
			}
		} catch (IOException ex) {
			……
		} finally {
			……
		}
	}
```

`createRequest`方法在`RestTemplate`的父类`HttpAccessor`中：
```java
    protected ClientHttpRequest createRequest(URI url, HttpMethod method) throws IOException {
		ClientHttpRequest request = getRequestFactory().createRequest(url, method);
		if (logger.isDebugEnabled()) {
			logger.debug("Created " + method.name() + " request for \"" + url + "\"");
		}
		return request;
	}
```
主要逻辑就是判断下有没有配置拦截器，如果有的话，就使用`InterceptingClientHttpRequestFactory`类来创建`ClientHttpRequest`对象，否则就使用`HttpAccessor`默认定义的`SimpleClientHttpRequestFactory`类。

这里通过断点，我们也很容易能够看到创建的`ClientHttpRequest`对象是由`InterceptingClientHttpRequest`类实现的，`request.execute()`调用的是其父类中的方法，最终调用的是`InterceptingClientHttpRequest`重写的`executeInternal`方法：
```java
    @Override
	protected final ClientHttpResponse executeInternal(HttpHeaders headers, byte[] bufferedOutput) throws IOException {
		InterceptingRequestExecution requestExecution = new InterceptingRequestExecution();
		return requestExecution.execute(this, bufferedOutput);
	}
```

`InterceptingRequestExecution`是`InterceptingClientHttpRequest`的内部类，创建时会从`InterceptingClientHttpRequest`的`interceptors`属性中获得一个迭代器，需要注意的是，各拦截器中会递归调用`InterceptingRequestExecution`的`excute`方法，`excute`方法代码如下：
```java
        @Override
		public ClientHttpResponse execute(HttpRequest request, final byte[] body) throws IOException {
			if (this.iterator.hasNext()) {
				ClientHttpRequestInterceptor nextInterceptor = this.iterator.next();
				return nextInterceptor.intercept(request, body, this);
			} else {
                ……
			}
		}
```

### 拦截请求：LoadBalancerInterceptor
`LoadBalancerInterceptor`类代码如下：
```java
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {

	private LoadBalancerClient loadBalancer;
	private LoadBalancerRequestFactory requestFactory;

	//省略了构造方法

	@Override
	public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
			final ClientHttpRequestExecution execution) throws IOException {
		final URI originalUri = request.getURI();
		String serviceName = originalUri.getHost();
		Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
		return this.loadBalancer.execute(serviceName, requestFactory.createRequest(request, body, execution));
	}
}
```

通过代码可知，这个拦截器的核心逻辑就是通过`LoadBalancerClient`接口来实现负载均衡，查看这个接口，可以看到它继承、定义了以下几种抽象方法：
```java
//继承的抽象方法
ServiceInstance choose(String serviceId);
//定义的抽象方法
<T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;
<T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException;
URI reconstructURI(ServiceInstance instance, URI original);
```

这些方法是经Spring封装过的，屏蔽了大量负载均衡的逻辑，主要提供了：
1. choose：选择服务器
2. execute：执行请求
3. reconstructURI：替换url

之所以有个替换url的方法，是因为我们传输进来的URL中，host段填写的是服务名，所以需要查询出要调用哪一个服务，再将其host填写上去。

### 负载均衡：RibbonLoadBalancerClient


## Ribbon 的配置优化

