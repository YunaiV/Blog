title: Spring-Cloud-Gateway 源码解析 —— 处理器 (3.2) 之 RoutePredicateHandlerMapping 路由匹配
date: 2020-02-20
tags:
categories: Spring-Cloud-Gateway
permalink: Spring-Cloud-Gateway/handler-route-predicate-handler-mapping

-------

摘要: 原创出处 http://www.iocoder.cn/Spring-Cloud-Gateway/handler-route-predicate-handler-mapping/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Spring-Cloud-Gateway 2.0.X M4**  

- [1. 概述](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-route-predicate-handler-mapping/)
- [2. DispatcherHandler](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-route-predicate-handler-mapping/)
  - [2.1 SimpleHandlerAdapter](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-route-predicate-handler-mapping/)
- [3. RoutePredicateHandlerMapping](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-route-predicate-handler-mapping/)
- [666. 彩蛋](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-route-predicate-handler-mapping/)

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

本文主要分享 **RoutePredicateHandlerMapping 路由匹配**。

我们先一起来看看，一个请求是怎么被 Spring Cloud Gateway 处理的，如下图 ：

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_02_20/01.jpeg)

1. `org.springframework.web.reactive.DispatcherHandler` ：接收到请求，匹配 HandlerMapping ，此处会匹配到 RoutePredicateHandlerMapping 。
2. `org.springframework.cloud.gateway.handler.RoutePredicateHandlerMapping` ：接收到请求，匹配 Route 。
3. `org.springframework.cloud.gateway.handler.FilteringWebHandler` ：获得 Route 的 GatewayFilter 数组，创建 GatewayFilterChain 处理请求。

第一、二步，在本文分享。第三步，在 [《Spring-Cloud-Gateway 源码解析 —— 处理器 (3.3) 之 FilteringWebHandler 创建过滤器链 》](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-filtering-web-handler/?self) 分享。

-------

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



# 2. DispatcherHandler

`org.springframework.web.reactive.DispatcherHandler` ，请求分发处理器，Spring WebFlux 的访问入口。可能大多数人对这个类都比较陌生，我们来看看他在 Spring MVC 的兄弟 [DispatcherServlet](http://jinnianshilongnian.iteye.com/blog/1602617) 是不是就有点熟悉的感觉。

下面来看看 `DispatcherHandler#handle(ServerWebExchange)` 方法，代码如下 ：

```Java
  1: public class DispatcherHandler implements WebHandler, ApplicationContextAware {
  2: 
  3: 	@Nullable
  4: 	private List<HandlerMapping> handlerMappings;
  5: 
  6: 	@Nullable
  7: 	private List<HandlerAdapter> handlerAdapters;
  8: 
  9: 	@Override
 10: 	public Mono<Void> handle(ServerWebExchange exchange) {
 11: 		if (logger.isDebugEnabled()) {
 12: 			ServerHttpRequest request = exchange.getRequest();
 13: 			logger.debug("Processing " + request.getMethodValue() + " request for [" + request.getURI() + "]");
 14: 		}
 15: 		if (this.handlerMappings == null) {
 16: 			return Mono.error(HANDLER_NOT_FOUND_EXCEPTION);
 17: 		}
 18: 		return Flux.fromIterable(this.handlerMappings)
 19: 				.concatMap(mapping -> mapping.getHandler(exchange))
 20: 				.next()
 21: 				.switchIfEmpty(Mono.error(HANDLER_NOT_FOUND_EXCEPTION))
 22: 				.flatMap(handler -> invokeHandler(exchange, handler))
 23: 				.flatMap(result -> handleResult(exchange, result));
 24: 	}
 25: }
```

* 第 18 至 20 行 ：**顺序**使用 `handlerMappings` 获得对应的 WebHandler 。
    * 使用 `#concatMap(Function)` 操作符的原因是考虑 `handlerMappings` 的顺序性，详见 [《RxJava（四） concatMap操作符用法详解》](http://blog.csdn.net/johnny901114/article/details/51533282) 。
    * 使用官方 `spring-cloud-gateway-sample` 项目，此处打断点，`handlerMappings` 变量值如下图 ：![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_02_20/02.png)
    * 在【**第 19 行**】，调用 `HandlerMapping#getHandler(ServerWebExchange)` 获得 Handler 。在整理，RoutePredicateHandlerMapping 匹配请求对应的 Route ，并返回 FilteringWebHandler 。此时，**FilteringWebHandler 还并未获得 Route 的 GatewayFilter ，创建 GatewayFilterChain 处理请求**。和本文的第一张图有点出入，该图主要描述整个请求经历的流程。
    * 第 21 行 ：如果匹配不到 WebHandler ，返回 `HANDLER_NOT_FOUND_EXCEPTION` 。
    * 第 22 行 ：调用 `#handle()` 方法，执行 Handler 。代码如下 ：

        ```Java
          1: private Mono<HandlerResult> invokeHandler(ServerWebExchange exchange, Object handler) {
          2: 	if (this.handlerAdapters != null) {
          3: 		for (HandlerAdapter handlerAdapter : this.handlerAdapters) {
          4: 			if (handlerAdapter.supports(handler)) {
          5: 				return handlerAdapter.handle(exchange, handler);
          6: 			}
          7: 		}
          8: 	}
          9: 	return Mono.error(new IllegalStateException("No HandlerAdapter: " + handler));
         10: }
        ```
        * 使用官方 `spring-cloud-gateway-sample` 项目，此处打断点，`handlerMappings` 变量值如下图 ：![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_02_20/03.png)
        * 第 2 至 8 行 ：**顺序**匹配 HandlerAdapter ，通过调用 `HandlerAdapter#handle(ServerWebExchange, Object)` 方法，从而执行 Handler 。在此处，我们会匹配到 SimpleHandlerAdapter 。
        * 第 9 行 ：匹配不到 HandlerAdapter ，返回 IllegalStateException 。
    * 第 23 行 ：调用 `#handleResult()` 方法，处理结果。SimpleHandlerAdapter 返回的是 `Mono.empty()` ，所以不会触发该方法。`#handleResult()` 代码如下 ：

        ```Java
          1: private Mono<Void> handleResult(ServerWebExchange exchange, HandlerResult result) {
          2: 	return getResultHandler(result).handleResult(exchange, result)
          3: 			.onErrorResume(ex -> result.applyExceptionHandler(ex).flatMap(exceptionResult ->
          4: 					getResultHandler(exceptionResult).handleResult(exchange, exceptionResult)));
          5: }
        ```
    
## 2.1 SimpleHandlerAdapter

`org.springframework.web.reactive.result.SimpleHandlerAdapter` ，基执行 WebHandler 的处理器适配器。

`#supports(Object)` **方法**，代码如下 ：

```
@Override
public boolean supports(Object handler) {
	return WebHandler.class.isAssignableFrom(handler.getClass());
}
```
* 支持 WebHandler 。

-------

`#handle(ServerWebExchange, Object)` **方法**，代码如下 ：

```Java 
  1: @Override
  2: public Mono<HandlerResult> handle(ServerWebExchange exchange, Object handler) {
  3: 	WebHandler webHandler = (WebHandler) handler;
  4: 	Mono<Void> mono = webHandler.handle(exchange);
  5: 	return mono.then(Mono.empty());
  6: }
```

* 第 3 至 4 行 ：调用 `WebHandler#handle(ServerWebExchange)` 方法，执行处理器。例如，**WebHandler 为 FilteringWebHandler 时，获得 Route 的 GatewayFilter 数组，创建 GatewayFilterChain 处理请求**。
* 第 5 行 ：在 WebHandler **执行完后** ( `#then(Mongo)` )，然后返回 `Mono.empty()` 。

# 3. RoutePredicateHandlerMapping

`org.springframework.cloud.gateway.handler.RoutePredicateHandlerMapping` ，匹配 Route ，并返回处理 Route 的 FilteringWebHandler 。

RoutePredicateHandlerMapping **构造方法**，代码如下 ：

```Java
public class RoutePredicateHandlerMapping extends AbstractHandlerMapping {

	private final FilteringWebHandler webHandler;
	private final RouteLocator routeLocator;

	public RoutePredicateHandlerMapping(FilteringWebHandler webHandler, RouteLocator routeLocator) {
		this.webHandler = webHandler;
		this.routeLocator = routeLocator;

		setOrder(1); // RequestMappingHandlerMapping 之后
	}
}
```
* 调用 `#setOrder(1)` 的原因，Spring Cloud Gateway 的 GatewayWebfluxEndpoint 提供 HTTP API ，不需要经过网关，它通过 RequestMappingHandlerMapping 进行请求匹配处理。RequestMappingHandlerMapping 的 `order = 0` ，需要排在 RoutePredicateHandlerMapping 前面。所有，RoutePredicateHandlerMapping 设置 `order = 1` 。

-------

`#getHandlerInternal()` 方法，在 `DispatcherHandler#handle(ServerWebExchange)` 方法的【**第 19 行**】被调用，匹配 Route ，并返回处理 Route 的 FilteringWebHandler 。代码如下 ：

```Java
  1: @Override
  2: protected Mono<?> getHandlerInternal(ServerWebExchange exchange) {
  3:     // 设置 GATEWAY_HANDLER_MAPPER_ATTR 为 RoutePredicateHandlerMapping
  4: 	exchange.getAttributes().put(GATEWAY_HANDLER_MAPPER_ATTR, getClass().getSimpleName());
  5: 
  6: 	return lookupRoute(exchange) // 匹配 Route
  7: 			// .log("route-predicate-handler-mapping", Level.FINER) //name this
  8: 			.flatMap((Function<Route, Mono<?>>) r -> { // 返回 FilteringWebHandler
  9: 				if (logger.isDebugEnabled()) {
 10: 					logger.debug("Mapping [" + getExchangeDesc(exchange) + "] to " + r);
 11: 				}
 12: 
 13: 				// 设置 GATEWAY_ROUTE_ATTR 为 匹配的 Route
 14: 				exchange.getAttributes().put(GATEWAY_ROUTE_ATTR, r);
 15: 				// 返回
 16: 				return Mono.just(webHandler);
 17: 			}).switchIfEmpty(Mono.empty().then(Mono.fromRunnable(() -> { // 匹配不到 Route
 18: 				if (logger.isTraceEnabled()) {
 19: 					logger.trace("No RouteDefinition found for [" + getExchangeDesc(exchange) + "]");
 20: 				}
 21: 			})));
 22: }
```
* 第 4 行 ：设置 `GATEWAY_HANDLER_MAPPER_ATTR` 为 RoutePredicateHandlerMapping 。
* 第 6 行 ：调用 `#lookupRoute(ServerWebExchange)` 方法，匹配 Route 。
* 第 8 至 16 行 ：返回 Route 的处理器 FilteringWebHandler 。
    * 第 14 行 ：设置 `GATEWAY_ROUTE_ATTR` 为**匹配**的 Route 。
    * 第 16 行 ：返回 FilteringWebHandler。
* 第 17 至 21 行 ：**匹配不到** Route ，返回 `Mono.empty()` ，即不返回处理器。**这样会不会有问题**？不会，在 `DispatcherHandler#handle(ServerWebExchange)` 方法的【**第 21 行**】，我们可以看到，当没有合适的 Handler ，返回 `Mono.error(HANDLER_NOT_FOUND_EXCEPTION)` 。

-------

`#lookupRoute(ServerWebExchange)` 方法，**顺序**匹配 Route 。代码如下 ：

```Java
  1: protected Mono<Route> lookupRoute(ServerWebExchange exchange) {
  2: 	return this.routeLocator.getRoutes()
  3: 			.filter(route -> route.getPredicate().test(exchange))
  4: 			.next()
  5: 			//TODO: error handling
  6: 			.map(route -> {
  7: 				if (logger.isDebugEnabled()) {
  8: 					logger.debug("RouteDefinition matched: " + route.getId());
  9: 				}
 10: 				validateRoute(route, exchange);
 11: 				return route;
 12: 			});
 13: }
```
* 第 2 至 4 行 ：调用 `RouteLocator#getRoutes()` 方法，获得全部 Route ，并调用 `Predicate#test(ServerWebExchange)` 方法，**顺序**匹配**一个** Route。
* 第 5 行 ：未来会增加匹配过程中发生异常的处理。目前，任何一个 `Predicate#test(ServerWebExchange)` 的方法调用发生异常时，都会导致匹配不到 Route 。**一定要注意**。
* 第 6 至 11 行 ：调用 `#validateRoute(Route, ServerWebExchange)` 方法，校验 Route 的有效性。目前该方法是个**空方法**，可以通过继承 RoutePredicateHandlerMapping 进行覆盖重写。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

一不小心简单写了下 WebFlux 的 DispatcherHandler 的源码解析，嘿嘿嘿。

胖友，分享一波朋友圈可好！


