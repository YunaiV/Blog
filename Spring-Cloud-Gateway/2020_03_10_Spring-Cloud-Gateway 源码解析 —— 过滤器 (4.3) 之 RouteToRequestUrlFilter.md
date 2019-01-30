title: Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.3) 之 RouteToRequestUrlFilter
date: 2020-03-10
tags:
categories: Spring-Cloud-Gateway
permalink: Spring-Cloud-Gateway/filter-route-to-request

-------

摘要: 原创出处 http://www.iocoder.cn/Spring-Cloud-Gateway/filter-route-to-request/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Spring-Cloud-Gateway 2.0.X M4**  

- [1. 概述](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-route-to-request/)
- [2. RouteToRequestUrlFilter](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-route-to-request/)
- [666. 彩蛋](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-route-to-request/)

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

本文主要分享 **RouteToRequestUrlFilter 的代码实现**。

RouteToRequestUrlFilter 根据**匹配**的 Route ，计算请求的地址。**注意，这里的地址指的是 URL ，而不是 URI** 。

😈 RouteToRequestUrlFilter 的代码十分少，所以这会是一篇简单的文章。

-------

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



# 2. RouteToRequestUrlFilter

`org.springframework.cloud.gateway.filter.RouteToRequestUrlFilter` ，代码如下 ：

```Java
  1: public class RouteToRequestUrlFilter implements GlobalFilter, Ordered {
  2: 
  3: 	private static final Log log = LogFactory.getLog(RouteToRequestUrlFilter.class);
  4: 	public static final int ROUTE_TO_URL_FILTER_ORDER = 10000;
  5: 
  6: 	@Override
  7: 	public int getOrder() {
  8: 		return ROUTE_TO_URL_FILTER_ORDER;
  9: 	}
 10: 
 11: 	@Override
 12: 	public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
 13: 	    // 获得 Route
 14: 		Route route = exchange.getAttribute(GATEWAY_ROUTE_ATTR);
 15: 		if (route == null) {
 16: 			return chain.filter(exchange);
 17: 		}
 18: 		log.trace("RouteToRequestUrlFilter start");
 19: 		// 拼接 requestUrl
 20: 		URI requestUrl = UriComponentsBuilder.fromHttpRequest(exchange.getRequest())
 21: 				.uri(route.getUri())
 22: 				.build(true) // encoded=true
 23: 				.toUri();
 24: 		// 设置 requestUrl 到 GATEWAY_REQUEST_URL_ATTR {@link RewritePathGatewayFilterFactory}
 25: 		exchange.getAttributes().put(GATEWAY_REQUEST_URL_ATTR, requestUrl);
 26: 		// 提交过滤器链继续过滤
 27: 		return chain.filter(exchange);
 28: 	}
 29: 
 30: }
```

* 实现 **GlobalFilter** 接口。
* `#getOrder()` 方法，返回顺序为 10000 。在 [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.1) 之 GatewayFilter 一览》「3. GlobalFilter」](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-intro/?self) ，我们列举了所有 GlobalFilter 的顺序。
* 第 13 至 18 行 ：获得请求匹配的 Route ，在 [《Spring-Cloud-Gateway 源码解析 —— 处理器 (3.2) 之 RoutePredicateHandlerMapping 路由匹配》](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-route-predicate-handler-mapping/?self) 有详细解析。
* 第 20 至 23 行 ：拼接 `requestUrl` 。这里有一点要注意下，如果 `Route.uri` 属性配置带有 Path ，则会覆盖请求的 Path 。我们来举个例子 ：

| `Route.uri` | `Request.uri` | `requestUrl` |
| --- | --- | --- |
| http://bin.org:80 | http://127.0.0.1:8080/test/segment | http://httpbin.org:80/123 |
| http://bin.org:80/123 | http://127.0.0.1:8080/test/segment | http://httpbin.org:80/test/segment |

* 为什么会导致覆盖的情况呢 ？答案在 [`UriComponentsBuilder#uri(URI)`](https://github.com/spring-projects/spring-framework/blob/master/spring-web/src/main/java/org/springframework/web/util/UriComponentsBuilder.java#L413) 方法，代码如下 ：

    ```Java
      1: public UriComponentsBuilder uri(URI uri) {
      2: 	Assert.notNull(uri, "URI must not be null");
      3: 	this.scheme = uri.getScheme();
      4: 	if (uri.isOpaque()) {
      5: 		this.ssp = uri.getRawSchemeSpecificPart();
      6: 		resetHierarchicalComponents();
      7: 	}
      8: 	else {
      9: 		if (uri.getRawUserInfo() != null) {
     10: 			this.userInfo = uri.getRawUserInfo();
     11: 		}
     12: 		if (uri.getHost() != null) {
     13: 			this.host = uri.getHost();
     14: 		}
     15: 		if (uri.getPort() != -1) {
     16: 			this.port = String.valueOf(uri.getPort());
     17: 		}
     18: 		if (StringUtils.hasLength(uri.getRawPath())) {
     19: 			this.pathBuilder = new CompositePathComponentBuilder();
     20: 			this.pathBuilder.addPath(uri.getRawPath());
     21: 		}
     22: 		if (StringUtils.hasLength(uri.getRawQuery())) {
     23: 			this.queryParams.clear();
     24: 			query(uri.getRawQuery());
     25: 		}
     26: 		resetSchemeSpecificPart();
     27: 	}
     28: 	if (uri.getRawFragment() != null) {
     29: 		this.fragment = uri.getRawFragment();
     30: 	}
     31: 	return this;
     32: }
    ``` 
    * 第 18 至 21 行 ：当 `uri` 参数有 Path 时，**新建**一个 CompositePathComponentBuilder ，因此原有的 `this.pathBuilder` 被**覆盖**了。

* 第 25 行 ：设置 `requestUrl` 到 `GATEWAY_REQUEST_URL_ATTR` 。后面 Routing 相关的 GatewayFilter 会通过该属性，发起请求。
* 第 27 行 ：提交过滤器链继续过滤。**注意**，这里不需要创建**新**的 ServerWebExchange 。 

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

😈 硬生生把这个文章丰富了下。人生如戏，全靠套路。

胖友，分享一波朋友圈可好！


