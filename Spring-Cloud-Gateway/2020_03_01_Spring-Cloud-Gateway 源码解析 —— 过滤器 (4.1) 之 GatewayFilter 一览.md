title: Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.1) 之 GatewayFilter 一览
date: 2020-03-01
tags:
categories: Spring-Cloud-Gateway
permalink: Spring-Cloud-Gateway/filter-intro

-------

摘要: 原创出处 http://www.iocoder.cn/Spring-Cloud-Gateway/filter-intro/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Spring-Cloud-Gateway 2.0.X M4**  

- [1. 概述](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-intro/)
- [2. GatewyFilter](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-intro/)
  - [2.1 GatewayFilterFactory 内部类](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-intro/)
  - [2.2 OrderedGatewayFilter](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-intro/)
  - [2.3 GatewayFilterAdapter](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-intro/)
- [3. GlobalFilter](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-intro/)
- [4. GatewayFilterChain](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-intro/)
- [666. 彩蛋](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-intro/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2018_05_18.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

# 1. 概述

本文主要对 **过滤器 GatewayFilter 做整体的认识**。

过滤器整体类图如下 ：

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_03_01/01.png)

是不是有点疑惑 GlobalFilter 与 GatewayFilter 的关系 ？且见本文分晓。

-------

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



# 2. GatewyFilter

`org.springframework.cloud.gateway.filter.GatewayFilter` ，网关过滤器**接口**，代码如下 ：

```Java
public interface GatewayFilter {

	/**
	 * Process the Web request and (optionally) delegate to the next
	 * {@code WebFilter} through the given {@link GatewayFilterChain}.
	 * @param exchange the current server exchange
	 * @param chain provides a way to delegate to the next filter
	 * @return {@code Mono<Void>} to indicate when request processing is complete
	 */
	Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);

}
```

* 从接口方法可以看到，和 [`javax.servlet.Filter`](https://tomcat.apache.org/tomcat-5.5-doc/servletapi/javax/servlet/Filter.html) 类似。

GatewayFilter 有三种类型的子类实现，我们下面每节介绍一种。

## 2.1 GatewayFilterFactory 内部类

在每个 GatewayFilterFactory 实现类的 `#apply(Tuple)` 方法里，都声明了一个实现 GatewayFilter 的**内部类**，以 AddRequestHeaderGatewayFilterFactory 的代码举例子 ：

```Java
  1: public class AddRequestHeaderGatewayFilterFactory implements GatewayFilterFactory {
  2: 
  3: 	@Override
  4: 	public List<String> argNames() {
  5: 		return Arrays.asList(NAME_KEY, VALUE_KEY);
  6: 	}
  7: 
  8: 	@Override
  9: 	public GatewayFilter apply(Tuple args) {
 10: 		String name = args.getString(NAME_KEY);
 11: 		String value = args.getString(VALUE_KEY);
 12: 
 13: 		return (exchange, chain) -> { // GatewayFilter  
 14: 			ServerHttpRequest request = exchange.getRequest().mutate()
 15: 					.header(name, value)
 16: 					.build();
 17: 
 18: 			return chain.filter(exchange.mutate().request(request).build());
 19: 		};
 20: 	}
 21: }
```

* 第 13 至 19 行 ：定义了一个 GatewayFilter **内部实现类**。

在 [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.2) 之 GatewayFilterFactory 过滤器工厂》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-factory/?self) ，我们会详细解析每个 GatewayFilterFactory 定义的GatewayFilter **内部实现类**。

## 2.2 OrderedGatewayFilter

`org.springframework.cloud.gateway.filter.OrderedGatewayFilter` ，**有序的**网关过滤器**实现类**。在 FilterChain 里，过滤器数组首先会按照 `order` 升序排序，按照**顺序**过滤请求。代码如下 ：

```Java
public class OrderedGatewayFilter implements GatewayFilter, Ordered {

	private final GatewayFilter delegate;
	private final int order;

	public OrderedGatewayFilter(GatewayFilter delegate, int order) {
		this.delegate = delegate;
		this.order = order;
	}

	@Override
	public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
		return this.delegate.filter(exchange, chain);
	}

	@Override
	public int getOrder() {
		return this.order;
	}
}
```

* `delegate` 属性，委托的 GatewayFilter 。
* `order` 属性，顺序。
* `#filter(ServerWebExchange, GatewayFilterChain)` 方法，使用 `delegate` 过滤请求。

## 2.3 GatewayFilterAdapter

`org.springframework.cloud.gateway.handler.FilteringWebHandler.GatewayFilterAdapter` ，网关过滤器适配器。在 GatewayFilterChain 使用 GatewayFilter 过滤请求，所以通过 GatewayFilterAdapter 将 GlobalFilter 适配成 GatewayFilter 。GatewayFilterAdapter 代码如下 ：

```Java
private static class GatewayFilterAdapter implements GatewayFilter {

    private final GlobalFilter delegate;
    
    public GatewayFilterAdapter(GlobalFilter delegate) {
        this.delegate = delegate;
    }
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return this.delegate.filter(exchange, chain);
    }
}
```

* `delegate` 属性，委托的 GlobalFilter 。
* `#filter(ServerWebExchange, GatewayFilterChain)` 方法，使用 `delegate` 过滤请求。

-------

在 FilteringWebHandler 初始化时，将 GlobalFilter 委托成 GatewayFilterAdapter ，代码如下 ：

```Java
  1: public class FilteringWebHandler implements WebHandler {
  2: 
  3:     /**
  4:      * 全局过滤器
  5:      */
  6: 	private final List<GatewayFilter> globalFilters;
  7: 
  8: 	public FilteringWebHandler(List<GlobalFilter> globalFilters) {
  9: 		this.globalFilters = loadFilters(globalFilters);
 10: 	}
 11: 
 12: 	private static List<GatewayFilter> loadFilters(List<GlobalFilter> filters) {
 13: 		return filters.stream()
 14: 				.map(filter -> {
 15: 					GatewayFilterAdapter gatewayFilter = new GatewayFilterAdapter(filter);
 16: 					if (filter instanceof Ordered) {
 17: 						int order = ((Ordered) filter).getOrder();
 18: 						return new OrderedGatewayFilter(gatewayFilter, order);
 19: 					}
 20: 					return gatewayFilter;
 21: 				}).collect(Collectors.toList());
 22: 	}
 23: }
```

* 第 16 至 19 行 ：当 GlobalFilter 子类实现了 `org.springframework.core.Ordered` 接口，在委托一层 OrderedGatewayFilter 。这样 `AnnotationAwareOrderComparator#sort(List)` 方法好排序。
* 第 20 行 ：当 GlobalFilter 子类**没有**实现了 `org.springframework.core.Ordered` 接口，在 `AnnotationAwareOrderComparator#sort(List)` 排序时，顺序值为 `Integer.MAX_VALUE` 。
* 目前 GlobalFilter 都实现了 `org.springframework.core.Ordered` 接口。

# 3. GlobalFilter

`org.springframework.cloud.gateway.filter.GlobalFilter` ，全局过滤器**接口**，代码如下 ：

```Java
public interface GlobalFilter {

	/**
	 * Process the Web request and (optionally) delegate to the next
	 * {@code WebFilter} through the given {@link GatewayFilterChain}.
	 * @param exchange the current server exchange
	 * @param chain provides a way to delegate to the next filter
	 * @return {@code Mono<Void>} to indicate when request processing is complete
	 */
	Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);

}
```

* GlobalFilter 和 GatewayFilter 的 `#filter(ServerWebExchange, GatewayFilterChain)` **方法签名一致**。官方说，未来的版本将作出一些调整。

    > FROM [《Spring Cloud Gateway》](https://github.com/spring-cloud/spring-cloud-gateway/blob/9ffb0f18678460fda9b25c572c12f9054a62ca52/docs/src/main/asciidoc/spring-cloud-gateway.adoc#path-route-predicate-factory#user-content-global-filters)    
    > This interface and usage are subject to change in future milestones

* GlobalFilter 会作用到**所有的** Route 上。

GlobalFilter 实现类如下类图 ：

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_03_01/02.png)

* RoutingFilter
    * NettyRoutingFilter
    * WebClientHttpRoutingFilter
    * WebsocketRoutingFilter
    * ForwardRoutingFilter

* 成对的 Filter
    * NettyRoutingFilter / NettyRoutingFilter
    * WebClientHttpRoutingFilter / WebClientWriteResponseFilter

梳理 GlobalFilter 的顺序如下 ：


| GlobalFilter | 顺序 | 文章 |
| --- | --- | --- |
| NettyWriteResponseFilter | -1 | [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.7) 之 NettyRoutingFilter》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-netty-routing?self) |
| WebClientWriteResponseFilter | -1 | [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.8) 之 WebClientHttpRoutingFilter》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-web-client-http-routing?self) |
| RouteToRequestUrlFilter | 10000 | [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.3) 之 RouteToRequestUrlFilter》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-route-to-request/?self) |
| LoadBalancerClientFilter | 10100 | [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.4) 之 LoadBalancerClientFilter 负载均衡》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-load-balancer-client/?self) |
| ForwardRoutingFilter | `Integer.MAX_VALUE` | [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.5) 之 ForwardRoutingFilter》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-forward-routing/?self) |
| NettyRoutingFilter | `Integer.MAX_VALUE` | [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.7) 之 NettyRoutingFilter》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-netty-routing?self) |
| WebClientHttpRoutingFilter | `Integer.MAX_VALUE` | [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.8) 之 WebClientHttpRoutingFilter》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-web-client-http-routing?self) |
| WebsocketRoutingFilter | `Integer.MAX_VALUE` | [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.6) 之 WebSocketRoutingFilter》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-websocket-routing/?self) |

TODO 【3030】 

# 4. GatewayFilterChain

`org.springframework.cloud.gateway.filter.GatewayFilterChain` ，网关过滤器链**接口**。代码如下 ：

```Java
public interface GatewayFilterChain {

	/**
	 * Delegate to the next {@code WebFilter} in the chain.
	 * @param exchange the current server exchange
	 * @return {@code Mono<Void>} to indicate when request handling is complete
	 */
	Mono<Void> filter(ServerWebExchange exchange);

}
```

* 从接口方法可以看到，和 [`javax.servlet.FilterChain`](https://tomcat.apache.org/tomcat-8.0-doc/servletapi/javax/servlet/FilterChain.html) 类似。

-------

`org.springframework.cloud.gateway.handler.FilteringWebHandler.GatewayFilterAdapter` ，网关过滤器**链默认实现类**。代码如下 ：

```Java
private static class DefaultGatewayFilterChain implements GatewayFilterChain {

	private int index;
	private final List<GatewayFilter> filters;

	public DefaultGatewayFilterChain(List<GatewayFilter> filters) {
		this.filters = filters;
	}

	@Override
	public Mono<Void> filter(ServerWebExchange exchange) {
		if (this.index < filters.size()) {
			GatewayFilter filter = filters.get(this.index++);
			return filter.filter(exchange, this);
		} else {
			return Mono.empty(); // complete
		}
	}
}
```

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

啦啦啦，终于到过滤器了。开森！

胖友，分享一波朋友圈可好！


