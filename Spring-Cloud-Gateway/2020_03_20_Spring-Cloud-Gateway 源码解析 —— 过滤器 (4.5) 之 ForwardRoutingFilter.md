title: Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.5) 之 ForwardRoutingFilter
date: 2020-03-20
tags:
categories: Spring-Cloud-Gateway
permalink: Spring-Cloud-Gateway/filter-forward-routing

-------

摘要: 原创出处 http://www.iocoder.cn/Spring-Cloud-Gateway/filter-forward-routing/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Spring-Cloud-Gateway 2.0.X M4**  

- [1. 概述](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-forward-routing/)
- [2. RouteToRequestUrlFilter](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-forward-routing/)
- [666. 彩蛋](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-forward-routing/)

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

本文主要分享 **ForwardRoutingFilter 的代码实现**。

RouteToRequestUrlFilter ，转发**路由**网关过滤器。其根据 `forward://` 前缀( Scheme )过滤处理，**将请求转发到当前网关实例本地接口**。

举个例子，配置 RouteDefinition 路由定义如下 ：

```YAML
spring:
  application:
      name: juejin-gateway
  cloud:
    gateway:
      routes:
      # =====================================
      - id: forward_sample
        uri: forward:///globalfilters
        order: 10000
        predicates:
        - Path=/globalfilters
        filters:
        - PrefixPath=/application/gateway
```

* 我们假定网关端口为 `8080` 。
* 当请求 `http://127.0.0.1:8080/globalfilters` 接口，Spring Cloud Gateway 处理过程如下 ：
    * 匹配到路由 Route (`id = forward_sample`) 。
    * **配置**的 PrefixPathGatewayFilterFactory 将请求**改写**成 `http://127.0.0.1:8080/application/gateway/globalfilters` 。
    * ForwardRoutingFilter 判断有 `forward://` 前缀( Scheme )，过滤处理，将请求**转发**给 DispatcherHandler 。
    * DispatcherHandler 匹配并转发到**当前网关实例本地接口** [`application/gateway/globalfilters`](https://github.com/spring-cloud/spring-cloud-gateway/blob/83496b78944269050373bb92bb2181e1b7c070e8/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/actuate/GatewayWebfluxEndpoint.java#L96) 。
* 为什么需要配置 PrefixPathGatewayFilterFactory ？需要通过 PrefixPathGatewayFilterFactory 将请求重写路径，以匹配本地 API ，否则 DispatcherHandler 转发会失败。


另外，RouteToRequestUrlFilter 是 Spring Cloud Gateway 实现的一种**路由**网关过滤器，目前还提供 WebsocketRoutingFilter / NettyRoutingFilter / WebClientHttpRoutingFilter 。

-------

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



# 2. RouteToRequestUrlFilter

`org.springframework.cloud.gateway.filter.ForwardRoutingFilter` ，代码如下 ：

```Java
  1: public class ForwardRoutingFilter implements GlobalFilter, Ordered {
  2: 
  3: 	private static final Log log = LogFactory.getLog(ForwardRoutingFilter.class);
  4: 
  5: 	private final DispatcherHandler dispatcherHandler;
  6: 
  7: 	public ForwardRoutingFilter(DispatcherHandler dispatcherHandler) {
  8: 		this.dispatcherHandler = dispatcherHandler;
  9: 	}
 10: 
 11: 	@Override
 12: 	public int getOrder() {
 13: 		return Ordered.LOWEST_PRECEDENCE;
 14: 	}
 15: 
 16: 	@Override
 17: 	public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
 18: 	    // 获得 requestUrl
 19: 		URI requestUrl = exchange.getRequiredAttribute(GATEWAY_REQUEST_URL_ATTR);
 20: 
 21: 		// 判断是否能够处理
 22: 		String scheme = requestUrl.getScheme();
 23: 		if (isAlreadyRouted(exchange) || !scheme.equals("forward")) {
 24: 			return chain.filter(exchange);
 25: 		}
 26: 
 27: 		// 设置已经路由
 28: 		setAlreadyRouted(exchange);
 29: 
 30: 		//TODO: translate url?
 31: 
 32: 		if (log.isTraceEnabled()) {
 33: 			log.trace("Forwarding to URI: "+requestUrl);
 34: 		}
 35: 
 36: 		// DispatcherHandler 匹配并转发到当前网关实例本地接口
 37: 		return this.dispatcherHandler.handle(exchange);
 38: 	}
 39: }
```

* 实现 **GlobalFilter** 接口。
* `#getOrder()` 方法，返回顺序为 `Integer.MAX_VALUE` 。在 [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.1) 之 GatewayFilter 一览》「3. GlobalFilter」](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-intro/?self) ，我们列举了所有 GlobalFilter 的顺序。
* 第 19 行 ：获得 `requestUrl` 。
* 第 22 至 25 行 ：判断 ForwardRoutingFilter 是否能够处理该请求，需要满足两个条件 ：
    * `forward://` 前缀( Scheme ) 。
    * 调用 `ServerWebExchangeUtils#isAlreadyRouted(ServerWebExchange)` 方法，判断该请求暂未被其他 Routing 网关处理。代码如下 ：

        ```Java
        public static boolean isAlreadyRouted(ServerWebExchange exchange) {
            return exchange.getAttributeOrDefault(GATEWAY_ALREADY_ROUTED_ATTR, false);
        }
        ```
        * x
* 第 28 行 ：设置该请求已经被处理。代码如下 ：

    ```Java
    public static void setAlreadyRouted(ServerWebExchange exchange) {
        exchange.getAttributes().put(GATEWAY_ALREADY_ROUTED_ATTR, true);
    }
    ```

* 第 37 行 ：将请求**转发**给 DispatcherHandler 。DispatcherHandler 匹配并转发到**当前网关实例本地接口**。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

一开始想错了 ForwardRoutingFilter 了的用途，调试许久，后面看了官方提供的示例 `org.springframework.cloud.gateway.test.ForwardTests` ，豁然开朗。

胖友，分享一波朋友圈可好！


