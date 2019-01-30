title: Spring-Cloud-Gateway 源码解析 —— 路由（2.3）之 Java 自定义 RouteLocator 
date: 2020-02-05
tags:
categories: Spring-Cloud-Gateway
permalink: Spring-Cloud-Gateway/route-locator-route-custom-java

---

摘要: 原创出处 http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-route-custom-java/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Spring-Cloud-Gateway 2.0.X M4**  

- [1. 概述](http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-route-custom-java/)
- [2. Routes](http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-route-custom-java/)
- [3. RoutePredicates](http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-route-custom-java/)
- [4. GatewayFilters](http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-route-custom-java/)
- [666. 彩蛋](http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-route-custom-java/)

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

本文主要分享**如何使用 Java 实现自定义 RouteLocator**。

*ps ：为什么这里强调 Java 呢？可以使用 Kotlin 实现自定义 RouteLocator ，在下一篇文章我们会详细分享*。

首先我们来看一段**示例**程序，代码如下 ：

```Java
import static org.springframework.cloud.gateway.filter.factory.GatewayFilters.addResponseHeader;
import static org.springframework.cloud.gateway.handler.predicate.RoutePredicates.host;
import static org.springframework.cloud.gateway.handler.predicate.RoutePredicates.path;
import static org.springframework.tuple.TupleBuilder.tuple;

  1: @Bean
  2: public RouteLocator customRouteLocator(ThrottleGatewayFilterFactory throttle) {
  3: 	//@formatter:off
  4: 	return Routes.locator()
  5:            // Route
  6: 			.route("test")
  7: 				.predicate(host("**.abc.org").and(path("/image/png")))
  8: 				.addResponseHeader("X-TestHeader", "foobar")
  9: 				.uri("http://httpbin.org:80")
 10:            // Route
 11: 			.route("test2")
 12: 				.predicate(path("/image/webp"))
 13: 				.add(addResponseHeader("X-AnotherHeader", "baz"))
 14: 				.uri("http://httpbin.org:80")
 15:            // Route
 16: 			.route("test3")
 17: 				.order(-1)
 18: 				.predicate(host("**.throttle.org").and(path("/get")))
 19: 				.add(throttle.apply(tuple().of("capacity", 1,
 20: 						"refillTokens", 1,
 21: 						"refillPeriod", 10,
 22: 						"refillUnit", "SECONDS")))
 23: 				.uri("http://httpbin.org:80")
 24: 			.build();
 25: 	////@formatter:on
 26: }
```

* 使用 Routes 创建了**三个** Route 。
* 使用 RoutePredicates 创建**每个** Route 的 Predicate 。
* 使用 GatewayFilters 创建**每个** Route 的 GatewayFilter 。

-------

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



# 2. Routes

`rg.springframework.cloud.gateway.route.Routes` ，Java 自定义 RouteLocator **Builder** 。

Routes 内置多个 Builder 类，用于创建 Route 相关的各个元素 ：

| Routes 内置 Builder 类 | 组件 |
| --- | --- |
| LocatorBuilder | RouteLocator |
| RouteSpec | Route |
| PredicateSpec | Predicate |
| GatewayFilterSpec | GatewayFilter |

使用时，首先调用 `Routes#locator()` 方法，创建一个 LocatorBuilder 。代码如下 ：

```Java
public static LocatorBuilder locator() {
    return new LocatorBuilder();
}
```

LocatorBuilder ，代码如下 ：

```Java
  1: public static class LocatorBuilder {
  2: 
  3: 	private List<Route> routes = new ArrayList<>();
  4: 
  5: 	public PredicateSpec route(String id) {
  6: 		return new RouteSpec(this).id(id);
  7: 	}
  8: 
  9: 	private void add(Route route) {
 10: 		this.routes.add(route);
 11: 	}
 12: 
 13: 	LocatorBuilder uri(Route.Builder builder, String uri) {
 14: 		Route route = builder.uri(uri).build();
 15: 		routes.add(route);
 16: 		return this;
 17: 	}
 18: 
 19: 	LocatorBuilder uri(Route.Builder builder, URI uri) {
 20: 		Route route = builder.uri(uri).build();
 21: 		routes.add(route);
 22: 		return this;
 23: 	}
 24: 
 25: 	public RouteLocator build() {
 26: 		return () -> Flux.fromIterable(this.routes);
 27: 	}
 28: 
 29: }
```

* `routes` 属性，LocatorBuilder 已创建好的 Route 数组。
* `#add()` 方法，添加已创建好的 Route 。
* `#uri()` 方法，使用 [Route.Builder](https://github.com/spring-cloud/spring-cloud-gateway/blob/9ffb0f18678460fda9b25c572c12f9054a62ca52/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/route/Route.java#L67) 方法，创建 Route 并添加。 
* `#route()` 方法，**不同于上面两个方法**，首先创建 RouteSpec 对象，后调用 `RouteSpec#id(...)` 方法，创建 PredicateSpec 对象。**为什么是这样的呢**？Routes 里**创建** Route 是**有序**的**链式**过程，如下如 ：

    ![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_02_05/01.png)
    
    * 绿线 ：创建**一个** [Route.Builder](https://github.com/spring-cloud/spring-cloud-gateway/blob/9ffb0f18678460fda9b25c572c12f9054a62ca52/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/route/Route.java#L67) 。
    * 红线 ：调用 `LocatorBuilder#uri(uri)` 方法，创建 Route 并添加。后面，可以继续【**绿线**】，创建下一个 [Route.Builder](https://github.com/spring-cloud/spring-cloud-gateway/blob/9ffb0f18678460fda9b25c572c12f9054a62ca52/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/route/Route.java#L67) 。
* `#build()` 方法，创建**自定义** RouteLocator 类。

-------

RouteSpec / PredicateSpec / GatewayFilterSpec 实现上就是常见的 Builder 类，点击链接直接查看代码 ：

* [RouteSpec](https://github.com/spring-cloud/spring-cloud-gateway/blob/9ffb0f18678460fda9b25c572c12f9054a62ca52/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/route/Routes.java#L72)
* [PredicateSpec](https://github.com/spring-cloud/spring-cloud-gateway/blob/9ffb0f18678460fda9b25c572c12f9054a62ca52/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/route/Routes.java#L91)
* [GatewayFilterSpec](https://github.com/spring-cloud/spring-cloud-gateway/blob/9ffb0f18678460fda9b25c572c12f9054a62ca52/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/route/Routes.java#L132)

# 3. RoutePredicates

`org.springframework.cloud.gateway.handler.predicate.RoutePredicates` ，RoutePredicates **工厂**，其调用 RoutePredicateFactory 接口的**实现类**，创建各种 Predicate 。代码比较易懂，点击 [链接](https://github.com/spring-cloud/spring-cloud-gateway/blob/9ffb0f18678460fda9b25c572c12f9054a62ca52/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/handler/predicate/RoutePredicates.java#L38) 查看实现。

# 4. GatewayFilters

`org.springframework.cloud.gateway.filter.factory.GatewayFilters` ，GatewayFilter **工厂**，其调用 GatewayFilterFactory 接口的**实现类**，创建各种 GatewayFilter 。代码比较易懂，点击 [链接](https://github.com/spring-cloud/spring-cloud-gateway/blob/9ffb0f18678460fda9b25c572c12f9054a62ca52/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/filter/factory/GatewayFilters.java) 查看实现。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

原先还在纠结 Routes 怎么解释合适，画了个图，满意。

胖友，分享一波朋友圈可好！

