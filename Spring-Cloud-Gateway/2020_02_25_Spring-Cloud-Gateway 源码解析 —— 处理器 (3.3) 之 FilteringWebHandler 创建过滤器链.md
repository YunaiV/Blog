title: Spring-Cloud-Gateway 源码解析 —— 处理器 (3.3) 之 FilteringWebHandler 创建过滤器链 
date: 2020-02-25
tags:
categories: Spring-Cloud-Gateway
permalink: Spring-Cloud-Gateway/handler-filtering-web-handler

-------

摘要: 原创出处 http://www.iocoder.cn/Spring-Cloud-Gateway/handler-filtering-web-handler/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Spring-Cloud-Gateway 2.0.X M4**  

- [1. 概述](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-filtering-web-handler/)
- [2. FilteringWebHandler](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-filtering-web-handler/)
- [666. 彩蛋](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-filtering-web-handler/)

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

本文主要分享 **FilteringWebHandler**。

在 [《Spring-Cloud-Gateway 源码解析 —— 处理器 (3.2) 之 RoutePredicateHandlerMapping 路由匹配 》「2.1 SimpleHandlerAdapter」](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-route-predicate-handler-mapping/?self) 里，我们看到 `SimpleHandlerAdapter#handle(ServerWebExchange, Object)` 调用 `FilteringWebHandler#handle(ServerWebExchange)` 方法，处理请求。

FilteringWebHandler 通过创建请求对应的 Route 对应的 GatewayFilterChain 进行处理。

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_02_20/01.jpeg)

-------

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



# 2. FilteringWebHandler

`org.springframework.cloud.gateway.handler.FilteringWebHandler` ，`#handle(ServerWebExchange)` 代码如下 ：

```Java
  1: public class FilteringWebHandler implements WebHandler {
  2: 
  3: 	/**
  4: 	 * 全局过滤器
  5: 	*/
  6: 	private final List<GatewayFilter> globalFilters;
  7: 
  8: 	@Override
  9: 	public Mono<Void> handle(ServerWebExchange exchange) {
 10: 	    // 获得 Route
 11: 		Route route = exchange.getRequiredAttribute(GATEWAY_ROUTE_ATTR);
 12: 		// 获得 GatewayFilter
 13: 		List<GatewayFilter> gatewayFilters = route.getFilters();
 14: 		List<GatewayFilter> combined = new ArrayList<>(this.globalFilters);
 15: 		combined.addAll(gatewayFilters);
 16: 
 17: 		// 排序
 18: 		//TODO: needed or cached?
 19: 		AnnotationAwareOrderComparator.sort(combined);
 20: 		logger.debug("Sorted gatewayFilterFactories: "+ combined);
 21: 
 22: 		// 创建 DefaultGatewayFilterChain
 23: 		return new DefaultGatewayFilterChain(combined).filter(exchange);
 24: 	}
 25: }
```

* 本方法涉及到的过滤器 GlobalFilter / GatewayFilter / GatewayFilterAdapter / OrderedGatewayFilter 类，我们在 [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.1) 之 GatewayFilter 一览》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-intro/?self) 详细解析。
* 本方法涉及到的过滤器链 GatewayFilterChain / DefaultGatewayFilterChain 类，我们在 [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.1) 之 GatewayFilter 一览》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-intro/?self) 详细解析。
* 第 11 行 ：从 `GATEWAY_ROUTE_ATTR` 获得 请求对应的 Route 。
* 第 13 至 15 行 ：获得 GatewayFilter 数组，包含 `route.filters` 和 `globalFilters` 。
* 第 19 行 ：排序获得的 GatewayFilter 数组。
* 第 23 行 ：使用获得的 GatewayFilter 数组创建 DefaultGatewayFilterChain ，**过滤处理请求**。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

哈哈哈，我水更我快乐。主要还是考虑文章尽量解耦，所以这篇内容偏水( 很水 )。

胖友，分享一波朋友圈可好！


