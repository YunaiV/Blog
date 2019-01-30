title: Spring-Cloud-Gateway 源码解析 —— 网关管理 HTTP API
date: 2020-04-15
tags:
categories: Spring-Cloud-Gateway
permalink: Spring-Cloud-Gateway/manager-http-api

-------

摘要: 原创出处 http://www.iocoder.cn/Spring-Cloud-Gateway/manager-http-api/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Spring-Cloud-Gateway 2.0.X M4**  

- [1. 概述](http://www.iocoder.cn/Spring-Cloud-Gateway/manager-http-api/)
  - [2. 过滤器 HTTP API](http://www.iocoder.cn/Spring-Cloud-Gateway/manager-http-api/)
  - [2.1 全局过滤器列表](http://www.iocoder.cn/Spring-Cloud-Gateway/manager-http-api/)
  - [2.2 路由过滤器工厂列表](http://www.iocoder.cn/Spring-Cloud-Gateway/manager-http-api/)
  - [3. 路由 HTTP API](http://www.iocoder.cn/Spring-Cloud-Gateway/manager-http-api/)
  - [2.1 路由列表](http://www.iocoder.cn/Spring-Cloud-Gateway/manager-http-api/)
  - [2.2 单个路由信息](http://www.iocoder.cn/Spring-Cloud-Gateway/manager-http-api/)
  - [2.3 单个路由的过滤器](http://www.iocoder.cn/Spring-Cloud-Gateway/manager-http-api/)
  - [2.4 添加/修改单个路由](http://www.iocoder.cn/Spring-Cloud-Gateway/manager-http-api/)
  - [2.5 删除单个路由](http://www.iocoder.cn/Spring-Cloud-Gateway/manager-http-api/)
  - [2.6 刷新路由缓存](http://www.iocoder.cn/Spring-Cloud-Gateway/manager-http-api/)
- [666. 彩蛋](http://www.iocoder.cn/Spring-Cloud-Gateway/manager-http-api/)

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

本文主要分享**网关管理 HTTP API**。

`org.springframework.cloud.gateway.actuate.GatewayWebfluxEndpoint` ，提供**管理**网关的 HTTP API 。**构造方法**，代码如下：

```Java
@RestController
@RequestMapping("${management.context-path:/application}/gateway")
public class GatewayWebfluxEndpoint implements ApplicationEventPublisherAware {

	private static final Log log = LogFactory.getLog(GatewayWebfluxEndpoint.class);

    /**
     * 路由定义定位器
     */
	private RouteDefinitionLocator routeDefinitionLocator;
    /**
     * 全局过滤器
     */
	private List<GlobalFilter> globalFilters;
    /**
     * 网关过滤器工厂
     */
	private List<GatewayFilterFactory> gatewayFilters;
    /**
     * 存储器 RouteDefinitionLocator 对象
     */
	private RouteDefinitionWriter routeDefinitionWriter;
    /**
     * 路由定位器
     */
	private RouteLocator routeLocator;
    /**
     * 应用事件发布器
     */
	private ApplicationEventPublisher publisher;

	public GatewayWebfluxEndpoint(RouteDefinitionLocator routeDefinitionLocator, List<GlobalFilter> globalFilters,
								  List<GatewayFilterFactory> GatewayFilters, RouteDefinitionWriter routeDefinitionWriter,
								  RouteLocator routeLocator) {
		this.routeDefinitionLocator = routeDefinitionLocator;
		this.globalFilters = globalFilters;
		this.gatewayFilters = GatewayFilters;
		this.routeDefinitionWriter = routeDefinitionWriter;
		this.routeLocator = routeLocator;
	}
}
```

* `@RequestMapping` 注解，HTTP  API 以 `"${management.context-path:/application}/gateway"` 。
* `routeDefinitionLocator` 属性，路由定义定位器。在 [《Spring-Cloud-Gateway 源码解析 —— 路由（2.2）之 RouteDefinitionRouteLocator 路由配置》](http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-route-definition/?self) 有详细解析。
* `globalFilters` 属性，全局过滤器。在 [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.1) 之 GatewayFilter 一览》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-intro/?self) 有详细解析。
* `gatewayFilters` 属性，网关过滤器工厂。在 [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.2) 之 GatewayFilterFactory 过滤器工厂》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-factory/?self)
* `routeLocator` 属性，路由定位器。在 [《Spring-Cloud-Gateway 源码解析 —— 路由（2.1）之 RouteLocator 一览》](http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-intro/?self) 有详细解析。
* `publisher` 属性，应用事件发布器。在 [《Spring5源码解析-Spring框架中的事件和监听器》](https://muyinchen.github.io/2017/09/27/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-Spring%E6%A1%86%E6%9E%B6%E4%B8%AD%E7%9A%84%E4%BA%8B%E4%BB%B6%E5%92%8C%E7%9B%91%E5%90%AC%E5%99%A8/) 有相关解析。

-------

GatewayWebfluxEndpoint 提供两类 HTTP API ：

* 过滤器 HTTP API
* 路由 HTTP API

-------

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



## 2. 过滤器 HTTP API

## 2.1 全局过滤器列表

```Java
@GetMapping("/globalfilters")
public Mono<HashMap<String, Object>> globalfilters() {
	return getNamesToOrders(this.globalFilters);
}

private <T> Mono<HashMap<String, Object>> getNamesToOrders(List<T> list) {
	return Flux.fromIterable(list).reduce(new HashMap<>(), this::putItem);
}

private HashMap<String, Object> putItem(HashMap<String, Object> map, Object o) {
	Integer order = null;
	if (o instanceof Ordered) {
		order = ((Ordered)o).getOrder();
	}
	//filters.put(o.getClass().getName(), order);
	map.put(o.toString(), order);
	return map;
}
```

## 2.2 路由过滤器工厂列表

```Java
@GetMapping("/routefilters")
public Mono<HashMap<String, Object>> routefilers() {
	return getNamesToOrders(this.gatewayFilters);
}
```

## 3. 路由 HTTP API

## 2.1 路由列表

```Java
@GetMapping("/routes")
public Mono<Map<String, List>> routes() {
	Mono<List<RouteDefinition>> routeDefs = this.routeDefinitionLocator.getRouteDefinitions().collectList();
	Mono<List<Route>> routes = this.routeLocator.getRoutes().collectList();
	return Mono.zip(routeDefs, routes).map(tuple -> {
		Map<String, List> allRoutes = new HashMap<>();
		allRoutes.put("routeDefinitions", tuple.getT1());
		allRoutes.put("routes", tuple.getT2());
		return allRoutes;
	});
}
```

## 2.2 单个路由信息

```Java
@GetMapping("/routes/{id}")
public Mono<ResponseEntity<RouteDefinition>> route(@PathVariable String id) {
	//TODO: missing RouteLocator
	return this.routeDefinitionLocator.getRouteDefinitions()
			.filter(route -> route.getId().equals(id))
			.singleOrEmpty()
			.map(route -> ResponseEntity.ok(route))
			.switchIfEmpty(Mono.just(ResponseEntity.notFound().build()));
}
```

* 从 `TODO: missing RouteLocator` ，我们可以看到，目前不支持从 RouteLocator 获取 Route ，只返回 RouteDefinition 。等待未来的版本支持。

## 2.3 单个路由的过滤器

```Java
@GetMapping("/routes/{id}/combinedfilters")
public Mono<HashMap<String, Object>> combinedfilters(@PathVariable String id) {
	//TODO: missing global filters
	return this.routeLocator.getRoutes()
			.filter(route -> route.getId().equals(id))
			.reduce(new HashMap<>(), this::putItem);
}
```

* 从 `TODO: missing global filters` ，我们可以看到，目前返回的过滤器不包括 GlobalFilter ，可以调用 `/globalfilters` 查看。等待未来的版本支持。

## 2.4 添加/修改单个路由

在 [《Spring-Cloud-Gateway 源码解析 —— 路由（1.3）之 RouteDefinitionRepository 存储器》「5. GatewayWebfluxEndpoint」](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-repository/?self) 有详细解析。

## 2.5 删除单个路由

在 [《Spring-Cloud-Gateway 源码解析 —— 路由（1.3）之 RouteDefinitionRepository 存储器》「5. GatewayWebfluxEndpoint」](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-repository/?self) 有详细解析。

## 2.6 刷新路由缓存

在 [《Spring-Cloud-Gateway 源码解析 —— 路由（2.1）之 RouteLocator 一览》「5. CachingRouteLocator」](http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-intro/?self) 有详细解析。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

水更一篇，哈哈哈。

胖友，分享一波朋友圈可好！

