```java
public interface ServiceInstanceChooser {
	// 根据serviceId选择要执行请求的实例
    ServiceInstance choose(String serviceId);
}
```

```java
public interface LoadBalancerClient extends ServiceInstanceChooser {
    // 使用从负载均衡器中挑选出的服务实例来执行请求内容
	<T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;
	
    // 使用指定实例在执行请求内容
	<T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException;

    // 构建请求URI
	URI reconstructURI(ServiceInstance instance, URI original);
}
```

从上一篇博客可以知道在声明`RestTemplate`时如果添加了`@LoadBalanced`注解的话，会往其拦截器链上添加一个`LoadBalancerInterceptor`拦截器，接下来我们看一下`LoadBalancerInterceptor`是如何起作用的

```java
@Override
public ClientHttpResponse intercept(final HttpRequest request, final byte[] body, final ClientHttpRequestExecution execution) throws IOException {
	final URI originalUri = request.getURI();
	String serviceName = originalUri.getHost();
	Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
	return this.loadBalancer.execute(serviceName, requestFactory.createRequest(request, body, execution));
}
```

其核心是调用了`loadBalancer`的`execute`方法，`loadBalancer`是在`LoadBalancerInterceptor`构造方法中传入的，如果不额外声明默认为`RibbonLoadBalancerClient`，该配置在`RibbonAutoConfiguration`

```java
@Bean
@ConditionalOnMissingBean(LoadBalancerClient.class)
public LoadBalancerClient loadBalancerClient() {
	return new RibbonLoadBalancerClient(springClientFactory());
}	
```

接下来跟进`RibbonLoadBalancerClient`的`execute`方法，源代码如下

```java

```

```java
public interface ILoadBalancer {
    // 向负载均衡器中维护的实例列表增加服务实例
    public void addServers(List<Server> newServers);

    // 从负载均衡器中挑选出一个具体的服务实例
    public Server chooseServer(Object key);
    
    // 用来通知和标识负载均衡器中某个具体实例已经停止服务
    public void markServerDown(Server server);
    
    // 获取当前正常服务的实例列表
    public List<Server> getReachableServers();

    // 获取所有已知的服务实例列表，包括正常服务和停止服务的实例
    public List<Server> getAllServers();
}

```

