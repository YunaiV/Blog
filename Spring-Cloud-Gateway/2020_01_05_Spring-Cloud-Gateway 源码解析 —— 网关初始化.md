title: Spring-Cloud-Gateway 源码解析 —— 网关初始化
date: 2020-01-05
tags:
categories: Spring-Cloud-Gateway
permalink: Spring-Cloud-Gateway/init

---

摘要: 原创出处 http://www.iocoder.cn/Spring-Cloud-Gateway/init/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Spring-Cloud-Gateway 2.0.X M4**

- [1. 概述](http://www.iocoder.cn/Spring-Cloud-Gateway/init/)
- [2. GatewayClassPathWarningAutoConfiguration](http://www.iocoder.cn/Spring-Cloud-Gateway/init/)
- [3. GatewayLoadBalancerClientAutoConfiguration](http://www.iocoder.cn/Spring-Cloud-Gateway/init/)
- [4. GatewayRedisAutoConfiguration](http://www.iocoder.cn/Spring-Cloud-Gateway/init/)
- [5. GatewayAutoConfiguration](http://www.iocoder.cn/Spring-Cloud-Gateway/init/)
  - [5.1 网关的开启与关闭](http://www.iocoder.cn/Spring-Cloud-Gateway/init/)
  - [5.2 初始化 NettyConfiguration](http://www.iocoder.cn/Spring-Cloud-Gateway/init/)
  - [5.3 初始化 GlobalFilter](http://www.iocoder.cn/Spring-Cloud-Gateway/init/)
  - [5.4 初始化 FilteringWebHandler](http://www.iocoder.cn/Spring-Cloud-Gateway/init/)
  - [5.5 初始化 GatewayProperties](http://www.iocoder.cn/Spring-Cloud-Gateway/init/)
  - [5.6 初始化 PrefixPathGatewayFilterFactory](http://www.iocoder.cn/Spring-Cloud-Gateway/init/)
  - [5.7 初始化 RoutePredicateFactory](http://www.iocoder.cn/Spring-Cloud-Gateway/init/)
  - [5.8 初始化 RouteDefinitionLocator](http://www.iocoder.cn/Spring-Cloud-Gateway/init/)
  - [5.9 初始化 RouteLocator](http://www.iocoder.cn/Spring-Cloud-Gateway/init/)
  - [5.10 初始化 RoutePredicateHandlerMapping](http://www.iocoder.cn/Spring-Cloud-Gateway/init/)
  - [5.11 初始化 GatewayWebfluxEndpoint](http://www.iocoder.cn/Spring-Cloud-Gateway/init/)
- [666. 彩蛋](http://www.iocoder.cn/Spring-Cloud-Gateway/init/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2018_05_18.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

---

# 1. 概述

本文主要分享 **Spring Cloud Gateway 启动初始化的过程**。

在初始化的过程中，涉及到的组件会较多，本文不会细说，留到后面每篇文章针对每个组件详细述说。

**那么这有什么意义呢**？先对 Spring Cloud Gateway 内部的组件有**整体**的印象。

-------

在官方提供的实例项目 `spring-cloud-gateway-sample` ，我们看到 GatewaySampleApplication 上有 `@EnableAutoConfiguration` 注解。因为该项目导入了 `spring-cloud-gateway-core` 依赖库，它会扫描 Spring Cloud Gateway 的配置。

在 [`org.springframework.cloud.gateway.config`](https://github.com/YunaiV/spring-cloud-gateway/tree/f552f51fc42db9ed88f783dc5f1291a22b34dcbc/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/config) 包下，我们可以看到**四个**配置类 ：

* GatewayAutoConfiguration
* GatewayClassPathWarningAutoConfiguration
* GatewayLoadBalancerClientAutoConfiguration
* GatewayRedisAutoConfiguration

它们的初始化顺序如下图 ：

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_01_05/01.png)

-------

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



# 2. GatewayClassPathWarningAutoConfiguration

Spring Cloud Gateway 2.x 基于 Spring WebFlux 实现。

`org.springframework.cloud.gateway.config.GatewayClassPathWarningAutoConfiguration` ，用于检查项目是否**正确**导入 `spring-boot-starter-webflux` 依赖，而不是错误**导入** `spring-boot-starter-web` 依赖。

点击链接 [链接](https://github.com/YunaiV/spring-cloud-gateway/blob/382a4cd98fbb8ac53a83a5559bacb0f885838074/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/config/GatewayClassPathWarningAutoConfiguration.java) 查看 GatewayClassPathWarningAutoConfiguration 的代码实现。

# 3. GatewayLoadBalancerClientAutoConfiguration

`org.springframework.cloud.gateway.config.GatewayLoadBalancerClientAutoConfiguration` ，初始化 LoadBalancerClientFilter ，点击 [链接](https://github.com/YunaiV/spring-cloud-gateway/blob/382a4cd98fbb8ac53a83a5559bacb0f885838074/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/config/GatewayLoadBalancerClientAutoConfiguration.java) 查看代码。

在 [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.4) 之 LoadBalancerClientFilter 负载均衡》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-load-balancer-client/?self) 详细解析 LoadBalancerClientFilter 的代码实现。

# 4. GatewayRedisAutoConfiguration

`org.springframework.cloud.gateway.config.GatewayRedisAutoConfiguration` ，初始化 RedisRateLimiter 。

RequestRateLimiterGatewayFilterFactory 基于 RedisRateLimiter 实现网关的**限流**功能，在 [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.10) 之 RequestRateLimiterGatewayFilterFactory 请求限流》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-request-rate-limiter/?self) 详细解析相关的代码实现。

# 5. GatewayAutoConfiguration

`org.springframework.cloud.gateway.config.GatewayAutoConfiguration` ，Spring Cloud Gateway **核心**配置类，初始化如下 ：

* Netty**Configuration**
* Global**Filter**
* FilteringWeb**Handler**
* Gateway**Properties**
* PrefixPathGateway**FilterFactory**
* Route**PredicateFactory**
* Route**DefinitionLocator**
* Route**Locator**
* Route**PredicateHandlerMapping**
* GatewayWebflux**Endpoint**

组件关系交互如下图 ：

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_01_05/04.png)

## 5.1 网关的开启与关闭

从 GatewayAutoConfiguration 上的注解 `@ConditionalOnProperty(name = "spring.cloud.gateway.enabled", matchIfMissing = true)` ，我们可以看出 ：

* 通过 `spring.cloud.gateway.enabled` 配置网关的开启与关闭。
* `matchIfMissing = true` => 网关**默认开启**。

## 5.2 初始化 NettyConfiguration

`org.springframework.cloud.gateway.config.NettyConfiguration` ，Netty 配置类。代码如下 ：

```Java
  1: @Configuration
  2: @ConditionalOnClass(HttpClient.class)
  3: protected static class NettyConfiguration {
  4: 	@Bean // 1.2
  5: 	@ConditionalOnMissingBean
  6: 	public HttpClient httpClient(@Qualifier("nettyClientOptions") Consumer<? super HttpClientOptions.Builder> options) {
  7: 		return HttpClient.create(options);
  8: 	}
  9: 
 10: 	@Bean // 1.1
 11: 	public Consumer<? super HttpClientOptions.Builder> nettyClientOptions() {
 12: 		return opts -> {
 13: 			opts.poolResources(PoolResources.elastic("proxy"));
 14: 			// opts.disablePool(); //TODO: why do I need this again?
 15: 		};
 16: 	}
 17: 
 18: 	@Bean // 1.3
 19: 	public NettyRoutingFilter routingFilter(HttpClient httpClient) {
 20: 		return new NettyRoutingFilter(httpClient);
 21: 	}
 22: 
 23: 	@Bean // 1.4
 24: 	public NettyWriteResponseFilter nettyWriteResponseFilter() {
 25: 		return new NettyWriteResponseFilter();
 26: 	}
 27: 
 28: 	@Bean // 1.5 {@link org.springframework.cloud.gateway.filter.WebsocketRoutingFilter}
 29: 	public ReactorNettyWebSocketClient reactorNettyWebSocketClient(@Qualifier("nettyClientOptions") Consumer<? super HttpClientOptions.Builder> options) {
 30: 		return new ReactorNettyWebSocketClient(options);
 31: 	}
 32: }
```

* 每个 `@Bean` 注解后的数字为 Bean 的**初始化顺序**。
* 第 10 至 16 行 ：创建一个类型为 `java.util.Objects.Consumer` 的 Bean 对象。该 Consumer 会将传入类型为 `reactor.ipc.netty.options.HttpClientOptions.Builder` 的参数 `opts` ，设置 `opts` 的 `poolResources` 属性。
    * 调用 `PoolResources.elastic("proxy")` 方法，创建 `name` 属性为 `"proxy"` 的 `reactor.ipc.netty.resources.PoolResources` 。其中 `"proxy"` 用于实际使用时，打印日志的**标记**。
* 第 4 至 8 行 ：创建一个类型为 `reactor.ipc.netty.http.client.HttpClient` 的 Bean 对象。该 HttpClient 使用 Netty 实现的 Client 。
* 第 18 至 21 行 ：使用 HttpClient Bean ，创建一个类型为 `org.springframework.cloud.gateway.filter.NettyRoutingFilter` 的 Bean 对象。在 [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.7) 之 NettyRoutingFilter》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-netty-routing/?self) 详细解析 NettyRoutingFilter 的代码实现。
* 第 23 至 26 行 ：创建一个类型为 `org.springframework.cloud.gateway.filter.NettyWriteResponseFilter` 的 Bean 对象。在 [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.7) 之 NettyRoutingFilter》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-netty-routing/?self) 详细解析 NettyWriteResponseFilter 的代码实现。
* 第 28 至 31 行 ：创建一个类型为 `org.springframework.web.reactive.socket.client.ReactorNettyWebSocketClient` 的 Bean 对象，用于下文 WebsocketRoutingFilter 的 Bean 对象创建。

## 5.3 初始化 GlobalFilter

代码如下 ：

```Java
  1: @Bean // 2.1
  2: public RouteToRequestUrlFilter routeToRequestUrlFilter() {
  3: 	return new RouteToRequestUrlFilter();
  4: }
  5: 
  6: @Bean // 2.2
  7: @ConditionalOnBean(DispatcherHandler.class)
  8: public ForwardRoutingFilter forwardRoutingFilter(DispatcherHandler dispatcherHandler) {
  9: 	return new ForwardRoutingFilter(dispatcherHandler);
 10: }
 11: 
 12: @Bean // 2.3
 13: public WebSocketService webSocketService() {
 14: 	return new HandshakeWebSocketService();
 15: }
 16: 
 17: @Bean // 2.4
 18: public WebsocketRoutingFilter websocketRoutingFilter(WebSocketClient webSocketClient, WebSocketService webSocketService) {
 19: 	return new WebsocketRoutingFilter(webSocketClient, webSocketService);
 20: }
```

* 每个 `@Bean` 注解后的数字为 Bean 的**初始化顺序**。
* 第 1 至 4 行 ：创建一个类型为 `org.springframework.cloud.gateway.filter.RouteToRequestUrlFilter`  的 Bean 对象。在 [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.3) 之 RouteToRequestUrlFilter》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-route-to-request/?self) 详细解析 RouteToRequestUrlFilter 的代码实现。
* 第 6 至 10 行 ：创建一个类型为 `org.springframework.cloud.gateway.filter.ForwardRoutingFilter` 的 Bean 对象。在 [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.5) 之 ForwardRoutingFilter》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-forward-routing/?self) 详细解析 ForwardRoutingFilter 的代码实现。
* 第 12 至 15 行 ：创建一个类型为 `org.springframework.web.reactive.socket.server.WebSocketService` 的 Bean 对象。
* 第 17 至 20 行 ：创建一个类型为 `org.springframework.cloud.gateway.filter.WebsocketRoutingFilter` 的 Bean 对象。在 [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.6) 之 WebsocketRoutingFilter》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-websocket-routing/?self) 详细解析 WebsocketRoutingFilter 的代码实现。

## 5.4 初始化 FilteringWebHandler

当所有 `org.springframework.cloud.gateway.filter.GlobalFilter` 初始化完成时( 包括上面的 NettyRoutingFilter / NettyWriteResponseFilter )，创建一个类型为 `org.springframework.cloud.gateway.handler.FilteringWebHandler` 的 Bean 对象，代码如下 ：

```Java
@Bean // 2.6
public FilteringWebHandler filteringWebHandler(List<GlobalFilter> globalFilters) {
    return new FilteringWebHandler(globalFilters);
}
```

* 在 [《Spring-Cloud-Gateway 源码解析 —— 处理器 (3.3) 之 FilteringWebHandler 创建过滤器链 》](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-filtering-web-handler/?self) 详细解析 FilteringWebHandler 的代码实现。


## 5.5 初始化 GatewayProperties

创建一个类型为 `org.springframework.cloud.gateway.config.GatewayProperties` 的 Bean 对象，用于加载**配置文件**配置的 RouteDefinition / FilterDefinition 。代码如下 ：

```Java
@Bean // 2.7
public GatewayProperties gatewayProperties() {
    return new GatewayProperties();
}
```

* 在 [《Spring-Cloud-Gateway 源码解析 —— 路由（1.2）之 PropertiesRouteDefinitionLocator 配置文件》「3. GatewayProperties」](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-properties?self) 详细解析 GatewayProperties 的代码实现。

## 5.6 初始化 PrefixPathGatewayFilterFactory

在 [链接](https://github.com/YunaiV/spring-cloud-gateway/blob/382a4cd98fbb8ac53a83a5559bacb0f885838074/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/config/GatewayAutoConfiguration.java#L288) 的【第 288 至 372 行】，创建 `org.springframework.cloud.gateway.filter.factory` 包下的 `org.springframework.cloud.gateway.filter.factory.GatewayFilterFactory` **接口**的实现们。

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_01_05/02.png)

后续我们会对**每个** GatewayFilterFactory 的实现代码做详细解析。

## 5.7 初始化 RoutePredicateFactory

在 [链接](https://github.com/YunaiV/spring-cloud-gateway/blob/382a4cd98fbb8ac53a83a5559bacb0f885838074/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/config/GatewayAutoConfiguration.java#L236) 的【第 236 至 284 行】，创建 `org.springframework.cloud.gateway.handler.predicate` 包下的 `org.springframework.cloud.gateway.handler.predicate.RoutePredicateFactory` **接口**的实现们。

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_01_05/03.png)

后续我们会对**每个** RoutePredicateFactory 的实现代码做详细解析。

## 5.8 初始化 RouteDefinitionLocator

代码如下 ：

```Java
  1: @Bean // 4.1
  2: @ConditionalOnMissingBean
  3: public PropertiesRouteDefinitionLocator propertiesRouteDefinitionLocator(GatewayProperties properties) {
  4: 	return new PropertiesRouteDefinitionLocator(properties);
  5: }
  6: 
  7: @Bean // 4.2
  8: @ConditionalOnMissingBean(RouteDefinitionRepository.class)
  9: public InMemoryRouteDefinitionRepository inMemoryRouteDefinitionRepository() {
 10: 	return new InMemoryRouteDefinitionRepository();
 11: }
 12: 
 13: @Bean // 4.3
 14: @Primary // 优先被注入
 15: public RouteDefinitionLocator routeDefinitionLocator(List<RouteDefinitionLocator> routeDefinitionLocators) {
 16: 	return new CompositeRouteDefinitionLocator(Flux.fromIterable(routeDefinitionLocators));
 17: }
```

* 每个 `@Bean` 注解后的数字为 Bean 的**初始化顺序**。
* 第 1 至 5 行 ：使用 GatewayProperties Bean ，创建一个类型为 `org.springframework.cloud.gateway.config.PropertiesRouteDefinitionLocator`  的 Bean 对象。在 [《Spring-Cloud-Gateway 源码解析 —— 路由（1.2）之 PropertiesRouteDefinitionLocator 配置文件》「2. PropertiesRouteDefinitionLocator」](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-properties?self) 详细解析 PropertiesRouteDefinitionLocator 的代码实现。
* 第 7 至 11 行 ：创建一个类型为 `org.springframework.cloud.gateway.route.InMemoryRouteDefinitionRepository` 的 Bean 对象。在 [《Spring-Cloud-Gateway 源码解析 —— 路由（1.3）之 RouteDefinitionRepository 存储器》](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-repository?self) 详细解析 InMemoryRouteDefinitionRepository 的代码实现。
* 第 13 至 17 行 ：使用上面创建的 RouteDefinitionLocator 的 Bean **对象们**，创建一个类型为 `org.springframework.cloud.gateway.route.CompositeRouteDefinitionLocator` 的 Bean 对象。
    * 第 14 行的 `@Primary` 注解，用于下文注入类型为 RouteDefinitionLocator 的 Bean 对象时，使用该对象。点击 [《spring中少用的注解@primary解析》](http://blog.csdn.net/u013400939/article/details/52953804) 查看 `@Primary` 的详细介绍。

-------

`org.springframework.cloud.gateway.discovery.DiscoveryClientRouteDefinitionLocator` ，基于 DiscoveryClient 注册发现的 RouteDefinitionLocator **实现类**，需要**手动引入配置**，点击 [链接](https://github.com/YunaiV/spring-cloud-gateway/blob/382a4cd98fbb8ac53a83a5559bacb0f885838074/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/discovery/DiscoveryClientRouteDefinitionLocator.java) 查看。在 [《Spring-Cloud-Gateway 源码解析 —— 路由（1.4）之 DiscoveryClientRouteDefinitionLocator 注册中心》](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-discover-client/?self) 详细解析 DiscoveryClientRouteDefinitionLocator 的代码实现。

## 5.9 初始化 RouteLocator

代码如下 ：

```Java
  1: @Bean // 4.4
  2: public RouteLocator routeDefinitionRouteLocator(GatewayProperties properties,
  3: 											   List<GatewayFilterFactory> GatewayFilters,
  4: 											   List<RoutePredicateFactory> predicates,
  5: 											   RouteDefinitionLocator routeDefinitionLocator) {
  6: 	return new RouteDefinitionRouteLocator(routeDefinitionLocator, predicates, GatewayFilters, properties);
  7: }
  8: 
  9: @Bean // 4.5
 10: @Primary
 11: public RouteLocator routeLocator(List<RouteLocator> routeLocators) {
 12: 	return new CachingRouteLocator(new CompositeRouteLocator(Flux.fromIterable(routeLocators)));
 13: }
```

* 每个 `@Bean` 注解后的数字为 Bean 的**初始化顺序**。
* 第 1 至 7 行 ：创建一个类型为 `org.springframework.cloud.gateway.route.RouteDefinitionRouteLocator` 的 Bean 对象。在 [《Spring-Cloud-Gateway 源码解析 —— 路由（2.2）之 RouteDefinitionRouteLocator 路由配置》](http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-route-definition?self) 详细解析 RouteDefinitionRouteLocator 的代码实现。
    * 此处的 `routeDefinitionLocator` 参数，使用了 `@Primary` 注解的 CompositeRouteDefinitionLocator 的 Bean 对象。
* 第 9 至 13 行 ：创建一个类型为 `org.springframework.cloud.gateway.route.CachingRouteLocator` 的 Bean 对象。该 Bean 对象内嵌 `org.springframework.cloud.gateway.route.CompositeRouteLocator` 对象。在 [《Spring-Cloud-Gateway 源码解析 —— 路由（2.1）之 RouteLocator 一览》](http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-intro/?self) 详细解析 CachingRouteLocator / CompositeRouteLocator 的代码实现。

-------

另外，有如下两种方式实现**自定义** RouteLocator ：

* 使用 `Routes#locator()#build()` 方法，创建 RouteLocator ，**例子**代码如下 ：

    ```Java
    @Bean
    public RouteLocator customRouteLocator() {
        //@formatter:off
        return Routes.locator()
    			.route("test")
    				.predicate(host("**.abc.org").and(path("/image/png")))
    				.addResponseHeader("X-TestHeader", "foobar")
    				.uri("http://httpbin.org:80")
    			.route("test2")
    				.predicate(path("/image/webp"))
    				.add(addResponseHeader("X-AnotherHeader", "baz"))
    				.uri("http://httpbin.org:80")
    			.build();
        ////@formatter:on
    }
    ```
    * 在 [《Spring-Cloud-Gateway 源码解析 —— 路由（2.3）之 Java 自定义 RouteLocator》](http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-route-custom-java/?self) 详细解析 `org.springframework.cloud.gateway.route.Routes` 的代码实现。

* 使用 `RouteLocatorDsl#gateway()` 方法，创建 RouteLocator ，**该方式使用 Kotlin 实现**，**例子**代码如下 ：

    ```Java
    @Configuration
    class AdditionalRoutes {
    
    	@Bean
    	fun additionalRouteLocator(): RouteLocator = gateway {
    		route(id = "test-kotlin") {
    			uri("http://httpbin.org:80")
    			predicate(host("kotlin.abc.org") and path("/image/png"))
    			add(addResponseHeader("X-TestHeader", "foobar"))
    		}
    	}
    
    }
    ```
    * 在 [《Spring-Cloud-Gateway 源码解析 —— 路由（2.4）之 Kotlin 自定义 RouteLocator 》](http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-route-custom-kotlin/?self) 详细解析 `org.springframework.cloud.gateway.route.RouteLocatorDsl` 的代码实现。

## 5.10 初始化 RoutePredicateHandlerMapping

创建一个类型为 `org.springframework.cloud.gateway.handler.RoutePredicateHandlerMapping` 的 Bean 对象，用于查找匹配到 Route ，并进行处理。代码如下 ：

```Java
@Bean
public RoutePredicateHandlerMapping routePredicateHandlerMapping(FilteringWebHandler webHandler,
																   RouteLocator routeLocator) {
    return new RoutePredicateHandlerMapping(webHandler, routeLocator);
}
```

* 在 [《Spring-Cloud-Gateway 源码解析 —— 处理器 (3.2) 之 RoutePredicateHandlerMapping 路由匹配 》](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-route-predicate-handler-mapping/?self) 详细解析 RoutePredicateHandlerMapping 的代码实现。

## 5.11 初始化 GatewayWebfluxEndpoint

创建一个类型为 `org.springframework.cloud.gateway.actuate.GatewayWebfluxEndpoint` 的 Bean 对象，提供**管理**网关的 HTTP API ，代码如下 ：

```Java
@ManagementContextConfiguration
@ConditionalOnProperty(value = "management.gateway.enabled", matchIfMissing = true)
@ConditionalOnClass(Health.class)
protected static class GatewayActuatorConfiguration {

    @Bean
    public GatewayWebfluxEndpoint gatewayWebfluxEndpoint(RouteDefinitionLocator routeDefinitionLocator, List<GlobalFilter> globalFilters,
										
        List<GatewayFilterFactory> GatewayFilters, RouteDefinitionWriter routeDefinitionWriter,
														 RouteLocator routeLocator) {
		return new GatewayWebfluxEndpoint(routeDefinitionLocator, globalFilters, GatewayFilters, routeDefinitionWriter, routeLocator);
    }
}
```

* 在 [《Spring-Cloud-Gateway 源码解析 —— 网关管理 HTTP API》](http://www.iocoder.cn/Spring-Cloud-Gateway/manager-http-api/?self) 详细解析 GatewayWebfluxEndpoint 的代码实现。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

是不是内心里面有种感觉，“劳资看了一堆构造函数”？

嘿嘿嘿，后面咱一篇一篇走起！

胖友，分享一波朋友圈可好！


