title: Spring-Cloud-Gateway 源码解析 —— 路由（2.4）之 Kotlin 自定义 RouteLocator 
date: 2020-02-10
tags:
categories: Spring-Cloud-Gateway
permalink: Spring-Cloud-Gateway/route-locator-route-custom-kotlin

---

摘要: 原创出处 http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-route-custom-kotlin/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Spring-Cloud-Gateway 2.0.X M4**  

- [1. 概述](http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-route-custom-kotlin/)
- [2. RouteLocatorDsl](http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-route-custom-kotlin/)
- [666. 彩蛋](http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-route-custom-kotlin/)

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

本文主要分享**如何使用 Kotlin 实现自定义 RouteLocator**。

😈 由于笔者暂时不了解 Kotlin ，也比较**懒**，暂时不准备了解 Kotlin ，所以本文很大可能性是 `"一本正经的胡说八道"` 。

-------

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



# 2. RouteLocatorDsl

`org.springframework.cloud.gateway.route.RouteLocatorDsl` ，使用 Kotlin 实现自定义 RouteLocator 。我们先打开 [GatewayDsl.kt](https://github.com/YunaiV/spring-cloud-gateway/blob/6bb8d6f93c289fd3a84c802ada60dd2bb57e1fb7/spring-cloud-gateway-core/src/main/kotlin/org/springframework/cloud/gateway/route/GatewayDsl.kt) ，大体浏览一下。

下面我们来看一段**示例**程序，我们会把 [GatewayDsl.kt](https://github.com/YunaiV/spring-cloud-gateway/blob/6bb8d6f93c289fd3a84c802ada60dd2bb57e1fb7/spring-cloud-gateway-core/src/main/kotlin/org/springframework/cloud/gateway/route/GatewayDsl.kt) 的代码实现嵌入其中。代码如下 ：

```Kotlin
import org.springframework.cloud.gateway.filter.factory.GatewayFilters.addResponseHeader
import org.springframework.cloud.gateway.handler.predicate.RoutePredicates.host
import org.springframework.cloud.gateway.handler.predicate.RoutePredicates.path
import org.springframework.cloud.gateway.route.RouteLocator
import org.springframework.cloud.gateway.route.gateway
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

  1: @Configuration
  2: class AdditionalRoutes {
  3: 
  4: 	@Bean
  5: 	fun additionalRouteLocator(): RouteLocator = gateway {
  6: 		route(id = "test-kotlin") {
  7: 			uri("http://httpbin.org:80") // Route.Builder#uri(uri)
  8: 			predicate(host("kotlin.abc.org") and path("/image/png")) // Route.Builder#predicate(predicate)
  9: 			add(addResponseHeader("X-TestHeader", "foobar")) // Route.Builder#add(webFilter)
 10: 		}
 11: 	}
 12: 
 13: }
```

* 调用 `#gateway(...)` 方法，创建**自定义**的 RouteLocator 。代码如下 ：

    ```Kotlin
    // GatewayDsl.kt
    fun gateway(routeLocator: RouteLocatorDsl.() -> Unit) = RouteLocatorDsl().apply(routeLocator).build()
    ```

* 第 6 至 10 行 ：调用 `RouteLocatorDsl#route(...)` 方法，配置**一个** Route 。代码如下 ：

    ```Kotlin
    // GatewayDsl.kt
    private val routes = mutableListOf<Route>()
    
    /**
     * DSL to add a route to the [RouteLocator]
     *
     * @see [Route.Builder]
     */
    fun route(id: String? = null, order: Int = 0, uri: String? = null, init: Route.Builder.() -> Unit) {
        val builder = Route.builder()
        if (uri != null) {
            builder.uri(uri)
        }
        routes += builder.id(id).order(order).apply(init).build()
    }
    ```

* 第 7 行 ：调用 `Route.Builder#uri(uri)` 方法，设置 `uri` 。
* 第 8 行 ：调用 `Route.Builder#predicate(predicate)` 方法，设置 `predicates` 。
    * 使用 RoutePredicates 创建**每个** Route 的 Predicate 。
    * `and` / `or` 操作符，代码如下 ：

        ```Kotlin
        // GatewayDsl.kt
        
        /**
         * A helper to return a composed [Predicate] that tests against this [Predicate] AND the [other] predicate
         */
        infix fun <T> Predicate<T>.and(other: Predicate<T>) = this.and(other)
        
        /**
         * A helper to return a composed [Predicate] that tests against this [Predicate] OR the [other] predicate
         */
        infix fun <T> Predicate<T>.or(other: Predicate<T>) = this.or(other)
        ```
        * x

* 第 9 行 ：调用 `Route.Builder#add(webFilter)` 方法，添加 `filters` 。
    * 使用 GatewayFilters 创建**每个** Route 的 GatewayFilter 。
 
# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

😈 "**一本正经**" 的写完了，反正我是不管了。哈哈哈哈。

胖友，分享一波朋友圈可好！


