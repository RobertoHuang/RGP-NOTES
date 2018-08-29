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

```java
public interface ServiceInstanceChooser {
	// 根据serviceId选择要执行请求的实例
    ServiceInstance choose(String serviceId);
}

public interface LoadBalancerClient extends ServiceInstanceChooser {
    // 使用从负载均衡器中挑选出的服务实例来执行请求内容
	<T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;
	
    // 使用指定实例在执行请求内容
	<T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException;

    // 构建请求URI
	URI reconstructURI(ServiceInstance instance, URI original);
}
```

接下来跟进`RibbonLoadBalancerClient`的`execute`方法，源代码如下

```java
public <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException {
	ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
	Server server = getServer(loadBalancer);
	if (server == null) {
		throw new IllegalStateException("No instances available for " + serviceId);
	}
	RibbonServer ribbonServer = new RibbonServer(serviceId, server, isSecure(server, serviceId), serverIntrospector(serviceId).getMetadata(server));
	return execute(serviceId, ribbonServer, request);
}
```

该方法主要完成如下功能:

- 1.根据`ServerId`获取到对应的`ILoadBalancer`，`RibbonLoadBalancerClient`的负载即通过它实现的，

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

  如果不进行额外配置的话，`Spring Cloud`默认的`ILoadBalancer`实现为`ZoneAwareLoadBalancer`

  ```java
  @Bean
  @ConditionalOnMissingBean
  public ILoadBalancer ribbonLoadBalancer(IClientConfig config, ServerList<Server> serverList, ServerListFilter<Server> serverListFilter, IRule rule, IPing ping, ServerListUpdater serverListUpdater) {
  	if (this.propertiesFactory.isSet(ILoadBalancer.class, name)) {
  		return this.propertiesFactory.get(ILoadBalancer.class, config, name);
  	}
  	return new ZoneAwareLoadBalancer<>(config, rule, ping, serverList, serverListFilter, serverListUpdater);
  }
  ```

- 2.根据`ILoadBalancer`选出具体实例

  ```java
  protected Server getServer(ILoadBalancer loadBalancer) {
  	if (loadBalancer == null) {
  		return null;
  	}
  	return loadBalancer.chooseServer("default");
  }
  ```

- 3.将选出的实例信息封装成`RibbonServer`对象，`RibbonServer`继承自`ServiceInstance`接口，并且在此基础上提供了一些额外的属性，接下来先看一下`ServiceInstance`的接口定义

  ```java
  public interface ServiceInstance {
  	String getServiceId();
  
  	String getHost();
  
  	int getPort();
  
  	boolean isSecure();
  
  	URI getUri();
  
  	Map<String, String> getMetadata();
  
  	default String getScheme() {
  		return null;
  	}
  }
  ```

- 在获取到`ServiceInstance后`，将调用`execute`方法进行执行

  ```java
  public <T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException {
  	Server server = ((RibbonServer)serviceInstance).getServer();
  	T returnVal = request.apply(serviceInstance);
      return returnVal;
  }
  ```

  去掉一些健壮性分支的代码，`execute`方法主要完成的功能就是调用`request.apply(serviceInstance)`

  ```java
  HttpRequest serviceRequest = new ServiceRequestWrapper(request, instance, loadBalancer);
  if (transformers != null) {
      for (LoadBalancerRequestTransformer transformer : transformers) {
          serviceRequest = transformer.transformRequest(serviceRequest, instance);
      }
  }
  return execution.execute(serviceRequest, body);
  ```

- `ServiceRequestWrapper`对象，该对象继承了`HttpRequestWrapper`并重写了`getURI`函数，重写后的`getURI`会通过调用`LoadBalancerClient`接口的`reconstructURI`函数来重新构建一个URI来进行访问

  ```java
  public URI getURI() {
  	URI uri = this.loadBalancer.reconstructURI(this.instance, getRequest().getURI());
  	return uri;
  }
  ```

- 在`LoadBalancerInterceptor`拦截器中，`ClientHttpRequestExecution`的实例具体执行`execution.execute(serviceRequest, body)`时，会调用`InterceptingClientHttpRequest`下`InterceptingRequestExecution`类的`execute`函数，具体实现如下：

  ```java
  public ClientHttpResponse execute(HttpRequest request, byte[] body) throws IOException {
      if (this.iterator.hasNext()) {
          ClientHttpRequestInterceptor nextInterceptor = this.iterator.next();
          return nextInterceptor.intercept(request, body, this);
      } else {
          ClientHttpRequest delegate = requestFactory.createRequest(request.getURI(), request.getMethod());
          delegate.getHeaders().putAll(request.getHeaders());
          if (body.length > 0) {
              StreamUtils.copy(body, delegate.getBody());
          }
          return delegate.execute();
      }
  }
  ```

  从`reconstructURI`函数中，我们可以看到，它通过`ServiceInstance`实例对象的`serviceId`，从`SpringClientFactory`类的`clientFactory`对象中获取对应`serviceId`的负载均衡器的上下文`RibbonLoadBalancerContext`对象。然后根据`ServiceInstance`中的信息来构建具体服务实例信息的`Server`对象，并使用`RibbonLoadBalancerContext`对象的`reconstructURIWithServer`函数来构建服务实例的URI



  - 为了帮助理解，简单介绍一下上面提到的`SpringClientFactory`和`RibbonLoadBalancerContext`：

  - `SpringClientFactory`类是一个用来创建客户端负载均衡器的工厂类，该工厂会为每一个不同名的ribbon客户端生成不同的Spring上下文。
  - `RibbonLoadBalancerContext`类是`LoadBalancerContext`的子类，该类用于存储一些被负载均衡器使用的上下文内容和Api操作（`reconstructURIWithServer`就是其中之一）



  从`reconstructURIWithServer`的实现中我们可以看到，它同`reconstructURI`的定义类似。只是`reconstructURI`的第一个保存具体服务实例的参数使用了Spring Cloud定义的`ServiceInstance`，而`reconstructURIWithServer`中使用了Netflix中定义的`Server`，所以在`RibbonLoadBalancerClient`实现`reconstructURI`时候，做了一次转换，使用`ServiceInstance`的host和port信息来构建了一个`Server`对象来给`reconstructURIWithServer`使用。



  从`reconstructURIWithServer`的实现逻辑中，我们可以看到，它从`Server`对象中获取host和port信息，然后根据以服务名为host的`URI`对象original中获取其他请求信息，将两者内容进行拼接整合，形成最终要访问的服务实例的具体地址

- ```java
  public class LoadBalancerContext implements IClientConfigAware {
  
      ...
  
      public URI reconstructURIWithServer(Server server, URI original) {
          String host = server.getHost();
          int port = server .getPort();
          if (host.equals(original.getHost())
                  && port == original.getPort()) {
              return original;
          }
          String scheme = original.getScheme();
          if (scheme == null) {
              scheme = deriveSchemeAndPortFromPartialUri(original).first();
          }
  
          try {
              StringBuilder sb = new StringBuilder();
              sb.append(scheme).append("://");
              if (!Strings.isNullOrEmpty(original.getRawUserInfo())) {
                  sb.append(original.getRawUserInfo()).append("@");
              }
              sb.append(host);
              if (port >= 0) {
                  sb.append(":").append(port);
              }
              sb.append(original.getRawPath());
              if (!Strings.isNullOrEmpty(original.getRawQuery())) {
                  sb.append("?").append(original.getRawQuery());
              }
              if (!Strings.isNullOrEmpty(original.getRawFragment())) {
                  sb.append("#").append(original.getRawFragment());
              }
              URI newURI = new URI(sb.toString());
              return newURI;
          } catch (URISyntaxException e) {
              throw new RuntimeException(e);
          }
      }
  
      ...
  }
  ```

- 另外，从`RibbonLoadBalancerClient`的`execute`的函数逻辑中，我们还能看到在回调拦截器中，执行具体的请求之后，ribbon还通过`RibbonStatsRecorder`对象对服务的请求还进行了跟踪记录，这里不再展开说明，有兴趣的读者可以继续研究。

  分析到这里，我们已经可以大致理清Spring Cloud中使用Ribbon实现客户端负载均衡的基本脉络。了解了它是如何通过`LoadBalancerInterceptor`拦截器对`RestTemplate`的请求进行拦截，并利用Spring Cloud的负载均衡器`LoadBalancerClient`将以逻辑服务名为host的URI转换成具体的服务实例的过程。同时通过分析`LoadBalancerClient`的Ribbon实现`RibbonLoadBalancerClient`，可以知道在使用Ribbon实现负载均衡器的时候，实际使用的还是Ribbon中定义的`ILoadBalancer`接口的实现，自动化配置会采用`ZoneAwareLoadBalancer`的实例来进行客户端负载均衡实现