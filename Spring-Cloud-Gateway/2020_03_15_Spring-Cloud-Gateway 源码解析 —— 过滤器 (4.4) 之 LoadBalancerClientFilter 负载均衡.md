title: Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.4) 之 LoadBalancerClientFilter 负载均衡
date: 2020-03-15
tags:
categories: Spring-Cloud-Gateway
permalink: Spring-Cloud-Gateway/filter-load-balancer-client

-------

摘要: 原创出处 http://www.iocoder.cn/Spring-Cloud-Gateway/filter-load-balancer-client/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Spring-Cloud-Gateway 2.0.X M4**  

- [1. 概述](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-load-balancer-client/)
- [2. 环境搭建](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-load-balancer-client/)
- [3. LoadBalancerClientFilter](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-load-balancer-client/)
- [4. 高能](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-load-balancer-client/)
- [666. 彩蛋](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-load-balancer-client/)

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

本文主要分享 **LoadBalancerClientFilter 的代码实现**。

LoadBalancerClientFilter 根据 `lb://` 前缀过滤处理，使用 `serviceId` 选择**一个**服务实例，从而实现**负载均衡**。

-------

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



# 2. 环境搭建

在 [《Spring-Cloud-Gateway 源码解析 —— 路由（1.4）之 DiscoveryClientRouteDefinitionLocator 注册中心》「2. 环境搭建」](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-discover-client/?self) 有详细教程。

# 3. LoadBalancerClientFilter

`org.springframework.cloud.gateway.filter.LoadBalancerClientFilter` ，代码如下 ：

```Java
  1: public class LoadBalancerClientFilter implements GlobalFilter, Ordered {
  2: 
  3: 	private static final Log log = LogFactory.getLog(LoadBalancerClientFilter.class);
  4: 	public static final int LOAD_BALANCER_CLIENT_FILTER_ORDER = 10100;
  5: 
  6: 	private final LoadBalancerClient loadBalancer;
  7: 
  8: 	public LoadBalancerClientFilter(LoadBalancerClient loadBalancer) {
  9: 		this.loadBalancer = loadBalancer;
 10: 	}
 11: 
 12: 	@Override
 13: 	public int getOrder() {
 14: 		return LOAD_BALANCER_CLIENT_FILTER_ORDER;
 15: 	}
 16: 
 17: 	@Override
 18: 	public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
 19: 		// 获得 URL
 20: 		URI url = exchange.getAttribute(GATEWAY_REQUEST_URL_ATTR);
 21: 		if (url == null || !url.getScheme().equals("lb")) {
 22: 			return chain.filter(exchange);
 23: 		}
 24: 		// 添加 原始请求URI 到 GATEWAY_ORIGINAL_REQUEST_URL_ATTR
 25: 		//preserve the original url
 26: 		addOriginalRequestUrl(exchange, url);
 27: 
 28: 		log.trace("LoadBalancerClientFilter url before: " + url);
 29: 
 30: 		// 获取 服务实例
 31: 		final ServiceInstance instance = loadBalancer.choose(url.getHost());
 32: 		if (instance == null) {
 33: 			throw new NotFoundException("Unable to find instance for " + url.getHost());
 34: 		}
 35: 
 36: 		/*URI uri = exchange.getRequest().getURI();
 37: 		URI requestUrl = loadBalancer.reconstructURI(instance, uri);*/
 38: 		//
 39: 		URI requestUrl = UriComponentsBuilder.fromUri(url)
 40: 				.scheme(instance.isSecure()? "https" : "http") //TODO: support websockets
 41: 				.host(instance.getHost())
 42: 				.port(instance.getPort())
 43: 				.build(true)
 44: 				.toUri();
 45: 		log.trace("LoadBalancerClientFilter url chosen: " + requestUrl);
 46: 
 47: 		// 添加 请求URI 到 GATEWAY_REQUEST_URL_ATTR
 48: 		exchange.getAttributes().put(GATEWAY_REQUEST_URL_ATTR, requestUrl);
 49: 
 50: 		// 提交过滤器链继续过滤
 51: 		return chain.filter(exchange);
 52: 	}
 53: 
 54: }
```

* 第 19 至 23 行 ：获得 URL 。**只处理 `lb://` 为前缀( Scheme )的地址**。
* 第 第 26 行 ：调用 `ServerWebExchangeUtils#addOriginalRequestUrl(...)` 添加原始请求 URI 到 `GATEWAY_ORIGINAL_REQUEST_URL_ATTR` 。代码如下 ：

    ```Java
    public static void addOriginalRequestUrl(ServerWebExchange exchange, URI url) {
    		exchange.getAttributes().computeIfAbsent(GATEWAY_ORIGINAL_REQUEST_URL_ATTR, s -> new LinkedHashSet<>()); // 数组，考虑多次重写
        LinkedHashSet<URI> uris = exchange.getRequiredAttribute(GATEWAY_ORIGINAL_REQUEST_URL_ATTR);
        uris.add(url);
    }
    ```
    * 为什么使用 LinkedHashSet ？因为可以使用 RewritePathGatewayFilterFactory / PrefixPathGatewayFilterFactory 多次重写。

* 第 30 至 34 行 ：调用 `LoadBalancerClient#choose(String)` 方法，获得**一个**服务实例( ServiceInstance ) ，从而实现**负载均衡**。
    * 熟悉 Spring Cloud 的同学都知道，一般情况下 LoadBalancerClient 实现类为 `org.springframework.cloud.netflix.ribbon.RibbonLoadBalancerClient` 。
    * 举个 `instance` 的**值**例子 ：![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_03_15/01.png)
* 第 39 至 45 行 ：创建 `requestUrl` 。举个例子 ：![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_03_15/02.png)
* 第 48 行 ：设置 `requestUrl` 到 `GATEWAY_REQUEST_URL_ATTR` 。后面 Routing 相关的 GatewayFilter 会通过该属性，发起请求。
* 第 51 行 ：提交过滤器链继续过滤。**注意**，这里不需要创建**新**的 ServerWebExchange 

# 4. 高能

我们回过头看 [《Spring-Cloud-Gateway 源码解析 —— 路由（1.4）之 DiscoveryClientRouteDefinitionLocator 注册中心》「4. 高能」](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-discover-client/?self)

相同服务( `serviceId` 相同) ，服务实例的注册或下线，Ribbon 已经处理，所以不用担心。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

没有彩蛋，继续往下写！当然，《天才麻将少女》的福利还是有的！

胖友，分享一波朋友圈可好！


