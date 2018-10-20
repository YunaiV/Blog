title: Spring Cloud Ribbon 学习二：Ribbon 轮询策略源码分析
date: 2018-01-06
tag: 
categories: Ribbon
permalink: Ribbon/eryuechunfeng/strategy
author: 二月_春风
from_url: https://www.jianshu.com/p/96b0e0d6bf1b
wechat_url: 

-------

摘要: 原创出处 https://www.jianshu.com/p/96b0e0d6bf1b 「二月_春风」欢迎转载，保留摘要，谢谢！

- [源码分析](http://www.iocoder.cn/Ribbon/eryuechunfeng/strategy/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

> 看spring cloud源码分析好绕，还是坚持看完了。

先总结一下Ribbon的运行流程，可以跳过总结看下面，然后重新看总结。

> - 项目启动的时候会自动的为我们加载`LoadBalancerAutoConfiguration`自动配置类，该自动配置类初始化条件是要求classpath必须要有`RestTemplate`这个类，必须要有`LoadBalancerClient`实现类。
> - `LoadBalancerAutoConfiguration`为我们干了二件事，第一件是创建了`LoadBalancerInterceptor`拦截器bean，用于实现对客户端发起请求时进行拦截，以实现客户端负载均衡。创建了一个
>   `RestTemplateCustomizer`的bean，用于给`RestTemplate`增加`LoadBalancerInterceptor`拦截器。
> - 每次请求的时候都会执行`org.springframework.cloud.client.loadbalancer.LoadBalancerInterceptor`的`intercept`方法，而`LoadBalancerInterceptor`具有`LoadBalancerClient`（客户端负载客户端）实例的一个引用，
>   在拦截器中通过方法获取服务名的请求url（比如`http://user-service/user`），及服务名（比如user-service），然后调用负载均衡客户端的execute方法。
> - 执行负载客户端`RibbonLoadBalancerClient`（LoadBalancerClient的实现）的`execute`方法，得到`ILoadBalancer`（负载均衡器）的实现`ZoneAwareLoadBalancer`，并且通过调用其`chooseServer`方法获得服务列表中的一个实例，比如说user-service列表注册到eureka中一个实例。然后向其中的一个具体实例发起请求，得到结果。

## 源码分析

之前我们实现负载均衡是在消费端的`RestTemplate`加上注解`@LoadBalanced`，便可以实现负载均衡了

```Java
@Bean
@LoadBalanced
public RestTemplate restTemplate(){
    return new RestTemplate();
}
```

查看注解内容：

```Java
/**
 * Annotation to mark a RestTemplate bean to be configured to use a LoadBalancerClient
 * @author Spencer Gibb
 */
@Target({ ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Qualifier
public @interface LoadBalanced {
}
```

这个注解给`RestTemplate`做标记，标记为`LoadBalancerClient`。

查看`LoadBalancerClient`源码：

```Java
/**
 * Represents a client side load balancer
 * @author Spencer Gibb
 */
public interface LoadBalancerClient extends ServiceInstanceChooser {

    /**
     * 通过LoadBalancer的ServiceInstance对指定的服务执行请求操作
     */
    <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;

    <T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException;

    /**
     * 为系统构建一个合适的host:port形式的url。在分布式系统中，我们使用逻辑上的服务名称作为host来构建URI
     * （替代服务实例的host:port形式）进行请求，比如说myservice/path/to/service。
     */
    URI reconstructURI(ServiceInstance instance, URI original);
}
```

继承自接口`ServiceInstanceChooser`：

```Java
/**
 * Implemented by classes which use a load balancer to choose a server to
 * send a request to.
 *
 * @author Ryan Baxter
 */
public interface ServiceInstanceChooser {

    /**
     * Choose a ServiceInstance from the LoadBalancer for the specified service
     * @param serviceId the service id to look up the LoadBalancer
     * @return a ServiceInstance that matches the serviceId
     */
    ServiceInstance choose(String serviceId);
}
```

- ServiceInstance choose(String serviceId)：根据传入的服务名serviceId，从负载均衡器中挑选一个对应服务的实例。
- T execute(String serviceId, LoadBalancerRequest<T> request)：使用从负载均衡器中挑选出来的服务实例来执行请求内容。
- T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request)：使用从负载均衡器中挑选出来的服务实例来执行请求内容。
- URI reconstructURI(ServiceInstance instance, URI original)：为系统构建一个合适的host:port形式的url。在分布式系统中，我们使用逻辑上的服务名称作为host来构建URI（替代服务实例的host:port形式）进行请求，比如说myservice/path/to/service。

顺着`LoadBalancerClient`接口的所属包`org.springframework.cloud.client.loadbalancer`,我们对内容进行整理，可以得到下面的关系：

![img](http://upload-images.jianshu.io/upload_images/5225109-a51b34369eeb10de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

`LoadBalancerAutoConfiguration`为客户端Ribbon负载均衡的自动化配置类，

```Java
@Configuration
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
@EnableConfigurationProperties(LoadBalancerRetryProperties.class)
public class LoadBalancerAutoConfiguration {

    @LoadBalanced
    @Autowired(required = false)
    private List<RestTemplate> restTemplates = Collections.emptyList();

    @Bean
    public SmartInitializingSingleton loadBalancedRestTemplateInitializer(
            final List<RestTemplateCustomizer> customizers) {
        return new SmartInitializingSingleton() {
            @Override
            public void afterSingletonsInstantiated() {
                for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
                    for (RestTemplateCustomizer customizer : customizers) {
                    //通过调用RestTemplateCustomizer的实例来给需要的客户端负载均衡的RestTemplate增加LoadBalancerInterceptor拦截器。
                        customizer.customize(restTemplate);
                    }
                }
            }
        };
    }

    @Autowired(required = false)
    private List<LoadBalancerRequestTransformer> transformers = Collections.emptyList();

    @Bean
    @ConditionalOnMissingBean
    public LoadBalancerRequestFactory loadBalancerRequestFactory(
            LoadBalancerClient loadBalancerClient) {
        return new LoadBalancerRequestFactory(loadBalancerClient, transformers);
    }

    @Configuration
    @ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
    static class LoadBalancerInterceptorConfig {
       //创建了一个LoadBalancerInterceptor的bean，用于实现对客户端发起请求时进行拦截，以实现客户端负载均衡。
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
            //创建了一个RestTemplateCustomizer的bean，用于给RestTemplate增加LoadBalancerInterceptor拦截器。
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

`@ConditionalOnClass(RestTemplate.class)`：当前项目的classpath路径下有RestTemplate这个类。

`@ConditionalOnBean(LoadBalancerClient.class)`：spring容器中必须有LoadBalancerClient的实现bean

该自动化配置主要完成了三件事

- 创建了一个`LoadBalancerInterceptor`的bean，用于实现对客户端发起请求时进行拦截，以实现客户端负载均衡。
- 创建了一个`RestTemplateCustomizer`的bean，用于给RestTemplate增加`LoadBalancerInterceptor`拦截器。
- 维护了一个被`@LoadBalanced`注解修饰的`RestTemplate`对象列表，并在这里进行维护，通过调用`RestTemplateCustomizer`的实例来给需要的客户端负载均衡的`RestTemplate`增加`LoadBalancerInterceptor`拦截器。

看看`LoadBalancerInterceptor`拦截器是如何让一个普通的`RestTemplate`变成负载均衡的：

![img](http://upload-images.jianshu.io/upload_images/5225109-e2e4ac82dae559f9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

LoadBalancerInterceptor拦截器

`LoadBalancerClient`是一个抽象的接口，`originalUri.getHost()`获取到的是服务名，`execute`函数去根据服务名来选择实例并发起实际的请求。

`org.springframework.cloud.netflix.ribbon.RibbonLoadBalancerClient`是实现`LoadBalancerClient`接口，看其实现：
execute方法，使用从负载均衡器中挑选出来的服务实例来执行请求内容。

![img](http://upload-images.jianshu.io/upload_images/5225109-eec7bb2df1b8c298.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

execute方法

getServer方法：

![img](http://upload-images.jianshu.io/upload_images/5225109-a2ca8b07c1c59289.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

getServer方法

去调用ILoadBalancer实例的chooseServer方法

认识一下`com.netflix.loadbalancer.ILoadBalancer`接口：
ILoadBalancer负载均衡器

> Interface that defines the operations for a software loadbalancer. A typical
> loadbalancer minimally need a set of servers to loadbalance for, a method to
> mark a particular server to be out of rotation and a call that will choose a
> server from the existing list of server.
> 定义软件负载平衡器操作的接口。 一个典型的负载均衡器最低限度地需要一组服务器来负载平衡，一种方法标记一个特定的服务器，以避免旋转和葱已有的服务列表中选择一个实例进行调用。

```Java
public interface ILoadBalancer {

    //向负载均衡器中维护的实例列表增加服务实例
    public void addServers(List<Server> newServers);

    //从负载均衡器中挑选出一个具体的服务实例
    public Server chooseServer(Object key);

    //用来通知和标记负载均衡器中的某个具体实例已经停止服务，不然负载均衡器在下一次获取服务实例清单前都会认为服务实例均是正常服务的。
    public void markServerDown(Server server);

    /**
     * @deprecated 2016-01-20 This method is deprecated in favor of the
     * cleaner {@link #getReachableServers} (equivalent to availableOnly=true)
     * and {@link #getAllServers} API (equivalent to availableOnly=false).
     *
     * Get the current list of servers.
     *
     * @param availableOnly if true, only live and available servers should be returned
     */
    @Deprecated
    public List<Server> getServerList(boolean availableOnly);

    //获取当前正常服务的实例列表
    public List<Server> getReachableServers();

   //获取所有已知的服务实例列表，包括正常服务和停止服务实例。
    public List<Server> getAllServers();
}
```

`com.netflix.loadbalancer.Server`对象定义是一个传统的服务端节点，在该类中存储了服务节点的一些元数据信息，包括host,post以及一些部署信息等。

`com.netflix.loadbalancer.ILoadBalancer`接口的一些实现，

![img](http://upload-images.jianshu.io/upload_images/5225109-f1d580bec6d7e485.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

ILoadBalancer的一些实现类

springcloud整合Ribbon的时候选择采用的是`com.netflix.loadbalancer.ZoneAwareLoadBalancer`负载均衡器。调用它的`chooseServer`方法。

![img](http://upload-images.jianshu.io/upload_images/5225109-1b6d3d9c423280b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

ZoneAwareLoadBalancer的chooseServer方法

回到`org.springframework.cloud.netflix.ribbon.RibbonLoadBalancerClient`的`execute`方法，

```Java
public <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException {
        //从上面跟过来我们知道这边的serviceId其实就是服务名
        ILoadBalancer loadBalancer = this.getLoadBalancer(serviceId);
        //通过ZoneAwareLoadBalancer的chooseServer函数获取了负载均衡策略分配的服务实例对象Server之后，将其包装成RibbonServer（增加了服务名serverid，是否需要使用Https等其他信息）
        Server server = this.getServer(loadBalancer);
        if(server == null) {
            throw new IllegalStateException("No instances available for " + serviceId);
        }
       RibbonServer ribbonServer = new RibbonServer(serviceId, server, isSecure(server,
    serviceId), serverIntrospector(serviceId).getMetadata(server));

   return execute(serviceId, ribbonServer, request);
}
```

`ILoadBalancer`的实现`com.netflix.loadbalancer.ZoneAwareLoadBalancer`，将其包装成`RibbonServer`，调用`ZoneAwareLoadBalancer`的`chooseServer`函数。

回到`org.springframework.cloud.netflix.ribbon.RibbonLoadBalancerClient`的`execute`方法，调用`LoadBalancerRequest`的`apply`方法，向一个实际的具体服务实例发起请求，从而实现一开始以服务名为host的URI请求到host:port形式的实际访问地址的转换

![img](http://upload-images.jianshu.io/upload_images/5225109-cc55b4db1a9db63a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

图片.png

`apply`方法参数`ServiceInstance`实例，`ServiceInstance`类

![img](http://upload-images.jianshu.io/upload_images/5225109-8a3ee6a497e27b59.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

ServiceInstance接口

上面说到的`RibbonServer`对象就是`ServiceInstance`接口的实现

![img](http://upload-images.jianshu.io/upload_images/5225109-d44bca07d48dc700.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

RibbonServer实现

我们已经可以大概理清了Spring Cloud Ribbon中实现客户端负载均衡的基本脉络，了解它是如何通过`LoadBalancerInterceptor`拦截器对`RestTemplate`的请求进行拦截，并利用Spring Cloud的负载均衡器`LoadBalancerClient`将以逻辑服务名为host的URI转换成具体的服务实例的过程。同时通过分析`LoadBalancerClient`的Ribbon实现`RibbonLoadBalancerClient`，可以知道在使用Ribbon实现负载均衡器的实现，实际使用的还是Ribbon中定义的ILoadBalancer接口的实现，自动化配置会采用`ZoneAwareLoadBalancer`的实例来实现客户端负载均衡。

# 666. 彩蛋

如果你对 Ribbon   感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)