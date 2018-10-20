title: Spring Cloud Ribbon 源码解析
date: 2018-01-05
tag: 
categories: Ribbon
permalink: Ribbon/shouzhugang/spring-cloud-ribbon
author: 瘦竹竿
from_url: https://www.jianshu.com/p/aa40be7da368
wechat_url: 

-------

摘要: 原创出处 https://www.jianshu.com/p/aa40be7da368 「瘦竹竿」欢迎转载，保留摘要，谢谢！

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

#### 简介

这篇文章是关于Spring Cloud Ribbon源码的解析的文章，在开始前大家必须搞清楚一件事，那就是Spring Cloud Ribbon和Netflix Ribbon，这个很关键，因我刚开始就弄混了，以为Spring Cloud Ribbon就是Netflix的Ribbon，这对查资料会有很大的误区。

##### Spring Cloud  Ribbon 和 Netflix Ribbon

1. Spring Cloud Ribbon是在Netflix Ribbon的基础上做了进一步的封装，使它更加适合与微服。
2. 在用法上Spring Cloud Ribbon的路由的服务清单是根据"注册中心"微服列表来的会实时更新，Netflix Ribbon需要手动设置。
3. Spring Cloud Ribbon的均衡器使用的是Netflix Ribbon的ZoneAwareLoadBalancer，如下图所示。

![ZoneAwareLoadBalancer.png](http://upload-images.jianshu.io/upload_images/3977236-1307746bba7fbb2e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



Spring Cloud  Ribbon文档的地址：[http://cloud.spring.io/spring-cloud-static/spring-cloud-netflix/2.0.0.M5/single/spring-cloud-netflix.html#spring-cloud-ribbon](https://link.jianshu.com?t=http%3A%2F%2Fcloud.spring.io%2Fspring-cloud-static%2Fspring-cloud-netflix%2F2.0.0.M5%2Fsingle%2Fspring-cloud-netflix.html%23spring-cloud-ribbon)
Netflix Ribbon 文档地址：[https://github.com/Netflix/ribbon/wiki](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2FNetflix%2Fribbon%2Fwiki)

所以如果我们想理解Spring Cloud Ribbon首先应该理解NetFlix的工作原理

##### Netflix Ribbon如何实现均衡器功能

先看一段Netflix Ribbon实现简单路由的demo,代码如下：

```Java
public static void main(String[] args) throws Exception {
  ConfigurationManager.loadPropertiesFromResources("sample-client.properties");  // 1
  System.out.println(ConfigurationManager.getConfigInstance().getProperty("sample-client.ribbon.listOfServers"));
  RestClient client = (RestClient) ClientFactory.getNamedClient("sample-client");  // 2
  HttpClientRequest request = HttpClientRequest.newBuilder().setUri(new URI("/")).build(); // 3
  for (int i = 0; i < 20; i++)  {
    HttpClientResponse response = client.executeWithLoadBalancer(request); // 4
    System.out.println("Status code for " + response.getRequestedURI() + "  :" + response.getStatus());
  }
  ZoneAwareLoadBalancer lb = (ZoneAwareLoadBalancer) client.getLoadBalancer();
  System.out.println(lb.getLoadBalancerStats());
  ConfigurationManager.getConfigInstance().setProperty(
        "sample-client.ribbon.listOfServers", "www.linkedin.com:80,www.google.com:80"); // 5
  System.out.println("changing servers ...");
  Thread.sleep(3000); // 6
  for (int i = 0; i < 20; i++)  {
    HttpClientResponse response = client.executeWithLoadBalancer(request);
    System.out.println("Status code for " + response.getRequestedURI() + "  : " + response.getStatus());
    response.releaseResources();
  }
  System.out.println(lb.getLoadBalancerStats()); // 7
}
```

配置文件如下：

```yaml
 sample-client.ribbon.listOfServers=www.microsoft.com:80,www.yahoo.com:80,www.google.com:80
```

上面代码的步骤如下：

1. 相关数据配置在config文件中，通过[Archaius ConfigurationManager](https://link.jianshu.com?t=http%3A%2F%2Fnetflix.github.com%2Farchaius%2Fjavadoc%2Fcom%2Fnetflix%2Fconfig%2FConfigurationManager.html) 加载配置数据。
2. 通过ClientFactory创建RestClient 和 ZoneAwareLoadBalancer（负载均衡器）。
3. 使用构建器构建http请求。请注意，我们只提供URI的路径部分（“/”）。一旦服务器被ZoneAwareLoadBalancer（负载均衡器）选中，完整的请求链接将由RestClient计算。
4. 发送请求是通过RestClient 的executeWithLoadBalancer()方法触发的。
5. 可以通过修改配置文件来动态的修改可用的服务的列表。

通过上面的步骤我们可以知道，网络请求的动作有RestClient实现，负载均衡的服务清单的维护和负载均衡的算法是在ZoneAwareLoadBalancer中实现的。

#### Spring Cloud Ribbon如何实现均衡器功能

1. 如何发起一个实现了负载均衡器的请求

```Java
@Autowired
RestTemplate restTemplate;

@HystrixCommand(fallbackMethod = "helloFallback")
public String hiService(String name) {
    return restTemplate.getForObject("http://service-hi/hi?name="+name,String.class);
}
```

RestTemplate 是Spring自己封装的http请求的客户端，也就是说它只能发送一个正常的Http请求,这跟我们要求的负载均衡是有出入的，还有就是这个请求的链接上的域名是我们微服的一个服务名，而不是一个真正的域名，那它是怎么实现负载均衡功能的呢？
我们来看看RestTemplate的父类InterceptingHttpAccessor。

```Java
public abstract class InterceptingHttpAccessor extends HttpAccessor {

private List<ClientHttpRequestInterceptor> interceptors = new ArrayList<ClientHttpRequestInterceptor>();

/**
 * Sets the request interceptors that this accessor should use.
 */
public void setInterceptors(List<ClientHttpRequestInterceptor> interceptors) {
    this.interceptors = interceptors;
}

/**
 * Return the request interceptor that this accessor uses.
 */
public List<ClientHttpRequestInterceptor> getInterceptors() {
    return interceptors;
}

@Override
public ClientHttpRequestFactory getRequestFactory() {
    ClientHttpRequestFactory delegate = super.getRequestFactory();
    if (!CollectionUtils.isEmpty(getInterceptors())) {
        return new InterceptingClientHttpRequestFactory(delegate, getInterceptors());
    }
    else {
        return delegate;
    }
}

}
```

从源码我们可以知道InterceptingHttpAccessor中有一个拦截器列表List<ClientHttpRequestInterceptor>，如果这个列表为空，则走正常请求流程，如果不为空则走拦截器，所以只要给RestTemplate添加拦截器，而这个拦截器中的逻辑就是Ribbon的负载均衡的逻辑。通过下面的方式可以为RestTemplate配置添加拦截器。

```Java
@LoadBalanced
RestTemplate restTemplate() {
    return new RestTemplate();
}
```

具体的拦截器的生成在LoadBalancerAutoConfiguration这个配置类中，所有的RestTemplate的请求都会转到Ribbon的负载均衡器上(当然这个时候如果你用RestTemplate发起一个正常的Http请求时走不通，因为它找不到对应的服务。)
这样就实现了Ribbon的请求的触发。

2.拦截器都做了什么？
上面提到过，发起http后请求后，请求会到达到达拦截器中，在拦截其中实现负载均衡，先看看代码：

```Java
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {

private LoadBalancerClient loadBalancer;
private LoadBalancerRequestFactory requestFactory;

public LoadBalancerInterceptor(LoadBalancerClient loadBalancer, LoadBalancerRequestFactory requestFactory) {
    this.loadBalancer = loadBalancer;
    this.requestFactory = requestFactory;
}

public LoadBalancerInterceptor(LoadBalancerClient loadBalancer) {
    // for backwards compatibility
    this(loadBalancer, new LoadBalancerRequestFactory(loadBalancer));
}

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

我们可以看到在intercept()方法中实现拦截的具体逻辑，首先会根据传进来的请求链接，获取微服的名字serviceName,然后调用LoadBalancerClient的execute(String serviceId, LoadBalancerRequest<T> request)方法，这个方法直接返回了请求结果，所以正真的路由逻辑在LoadBalancerClient的实现类中，而这个实现类就是RibbonLoadBalancerClient，看看execute()的源码：

```Java
@Override
public <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException {
    ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
    Server server = getServer(loadBalancer);
    if (server == null) {
        throw new IllegalStateException("No instances available for " + serviceId);
    }
    RibbonServer ribbonServer = new RibbonServer(serviceId, server, isSecure(server,
            serviceId), serverIntrospector(serviceId).getMetadata(server));

    return execute(serviceId, ribbonServer, request);
}
```

首先是获得均衡器ILoadBalancer这个类上面讲到过这是Netflix Ribbon中的均衡器，这是一个抽象类，具体的实现类是ZoneAwareLoadBalancer上面也讲到过，每一个微服名对应一个均衡器，均衡器中维护者微服名下所有的服务清单。getLoadBalancer()方法通过serviceId获得对应的均衡器，getServer()方法通过对应的均衡器在对应的路由的算法下计算得到需要路由到Server，Server中有该服务的具体域名等相关信息。得到了具体的Server后执行正常的Http请
求，整个请求的负载均衡逻辑就完成了。

我画了个Ribbon请求的一个流程图，纵向是调用顺序，横向是继承或实现的关系，如下图：

![ribbon流程图.png](http://upload-images.jianshu.io/upload_images/3977236-8023263fe77b72a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



#### 总结

这篇文章讲到是Spring Cloud Ribbon的源码解析，在微服中Ribbon和 Hystrix通常是一起使用的，其实直接使用Ribbon和Hystrix实现服务间的调用并不是很方便，通常在Spring Cloud中我们使用Feign完成服务间的调用，而Feign是对Ribbon和Hystrix做了进一步的封装方便大家使用，对Ribbon的学习能帮你更好的完成Spring Cloud中服务间的调用。

# 666. 彩蛋

如果你对 Ribbon   感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)