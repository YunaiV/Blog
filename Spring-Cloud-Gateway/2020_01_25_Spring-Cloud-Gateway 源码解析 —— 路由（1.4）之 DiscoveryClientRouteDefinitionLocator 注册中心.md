title: Spring-Cloud-Gateway 源码解析 —— 路由（1.4）之 DiscoveryClientRouteDefinitionLocator 注册中心
date: 2020-01-25
tags:
categories: Spring-Cloud-Gateway
permalink: Spring-Cloud-Gateway/route-definition-locator-discover-client

-------

摘要: 原创出处 http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-discover-client/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Spring-Cloud-Gateway 2.0.X M4**  

- [1. 概述](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-discover-client/)
- [2. 环境搭建](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-discover-client/)
- [3. DiscoveryClientRouteDefinitionLocator](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-discover-client/)
- [4. 高能](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-discover-client/)
- [666. 彩蛋](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-discover-client/)

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

本文主要对 **DiscoveryClientRouteDefinitionLocator 的源码实现**。

DiscoveryClientRouteDefinitionLocator 通过调用 `org.springframework.cloud.client.discovery.DiscoveryClient` 获取注册在注册中心的服务列表，生成对应的 RouteDefinition 数组。

-------

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



# 2. 环境搭建

在解析源码之前，我们先以 Eureka 为注册中心，讲解下如何配置 DiscoveryClientRouteDefinitionLocator 。

第一步，以 `spring-cloud-gateway-sample` 项目为基础，在 `pom.xml` 文件添加依赖库。

```XML
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-commons</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
    <version>2.0.0.M1</version>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```
* 注意，需要排除 `spring-boot-starter-web` 的依赖，避免和 Spring Cloud Gateway 依赖的 `spring-boot-starter-webflux` 冲突。

第二步，在 `application.yml` 添加 Eureka 相关配置 。

```YAML
spring:
  application:
      name: juejin-gateway

eureka:
  instance:
    leaseRenewalIntervalInSeconds: 10
    leaseExpirationDurationInSeconds: 30
  client:
    serviceUrl:
      defaultZone: http://eureka.didispace.com/eureka/  
```
* 如果你**"懒"**的启动 Eureka ，推荐使用 [《程序猿DD - 公益 - EUREKA SERVER》](http://eureka.didispace.com/) 。感谢 **1024** 。

第三步，在 `org.springframework.cloud.gateway.sample.GatewaySampleApplication` **类**里，添加 RouteDefinitionLocator Bean 配置。

```Java
@EnableDiscoveryClient // {@link DiscoveryClientRouteDefinitionLocator}
public class GatewaySampleApplication {
    
    // ... 省略其他代码

    @Bean
    public RouteDefinitionLocator discoveryClientRouteDefinitionLocator(DiscoveryClient discoveryClient) {
        return new DiscoveryClientRouteDefinitionLocator(discoveryClient);
    }
}
```

第四步，启动一个注册在 Eureka 的应用服务。机智如你，这里我就不啰嗦落。

第五步，在 DiscoveryClientRouteDefinitionLocator 的 `#getRouteDefinitions()` 方法打**断点**，调试启动 `spring-cloud-gateway-sample` 。小功告成。撒花~

# 3. DiscoveryClientRouteDefinitionLocator

`org.springframework.cloud.gateway.discovery.DiscoveryClientRouteDefinitionLocator` ，通过调用 DiscoveryClient 获取注册在注册中心的服务列表，生成对应的 RouteDefinition 数组。

代码如下 ：

```Java
  1: public class DiscoveryClientRouteDefinitionLocator implements RouteDefinitionLocator {
  2: 
  3: 	private final DiscoveryClient discoveryClient;
  4: 	private final String routeIdPrefix;
  5: 
  6: 	public DiscoveryClientRouteDefinitionLocator(DiscoveryClient discoveryClient) {
  7: 		this.discoveryClient = discoveryClient;
  8: 		this.routeIdPrefix = this.discoveryClient.getClass().getSimpleName() + "_";
  9: 	}
 10: 
 11: 	@Override
 12: 	public Flux<RouteDefinition> getRouteDefinitions() {
 13: 		return Flux.fromIterable(discoveryClient.getServices())
 14: 				.map(serviceId -> {
 15: 					RouteDefinition routeDefinition = new RouteDefinition();
 16: 					// 设置 ID
 17: 					routeDefinition.setId(this.routeIdPrefix + serviceId);
 18: 					// 设置 URI
 19: 					routeDefinition.setUri(URI.create("lb://" + serviceId));
 20: 
 21: 					// add a predicate that matches the url at /serviceId
 22: 					/*PredicateDefinition barePredicate = new PredicateDefinition();
 23: 					barePredicate.setName(normalizePredicateName(PathRoutePredicateFactory.class));
 24: 					barePredicate.addArg(PATTERN_KEY, "/" + serviceId);
 25: 					routeDefinition.getPredicates().add(barePredicate);*/
 26: 
 27: 					// 添加 Path 匹配断言
 28: 					// add a predicate that matches the url at /serviceId/**
 29: 					PredicateDefinition subPredicate = new PredicateDefinition();
 30: 					subPredicate.setName(normalizePredicateName(PathRoutePredicateFactory.class));
 31: 					subPredicate.addArg(PATTERN_KEY, "/" + serviceId + "/**");
 32: 					routeDefinition.getPredicates().add(subPredicate);
 33: 
 34: 					//TODO: support for other default predicates
 35: 
 36: 					// 添加 Path 重写过滤器
 37: 					// add a filter that removes /serviceId by default
 38: 					FilterDefinition filter = new FilterDefinition();
 39: 					filter.setName(normalizeFilterName(RewritePathGatewayFilterFactory.class));
 40: 					String regex = "/" + serviceId + "/(?<remaining>.*)";
 41: 					String replacement = "/${remaining}";
 42: 					filter.addArg(REGEXP_KEY, regex);
 43: 					filter.addArg(REPLACEMENT_KEY, replacement);
 44: 					routeDefinition.getFilters().add(filter);
 45: 
 46: 					//TODO: support for default filters
 47: 
 48: 					return routeDefinition;
 49: 				});
 50: 	}
 51: }
```

* `discoveryClient` 属性，注册发现客户端，用于向注册中心发起请求。
* `routeIdPrefix` 属性，路由配置编号前缀，以 DiscoveryClient 类名 + `_` 。
* 第 13 行 ：调用 `discoveryClient` 获取注册在注册中心的服务列表。
* 第 14 行 ：遍历服务列表，生成对应的 RouteDefinition 数组。
* 第 16 行 ：设置 `RouteDefinition.id` 。
* 第 18 行 ：设置 `RouteDefinition.uri` ，格式为 `lb://${serviceId}` 。在 LoadBalancerClientFilter 会根据 `lb://` 前缀过滤处理，负载均衡，选择最终调用的服务地址，在 [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.4) 之 LoadBalancerClientFilter 负载均衡》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-load-balancer-client/?self) 详细解析。
* 第 27 至 32 行 ：使用 PathRoutePredicateFactory 创建 Path 匹配断言。
    * 例如服务的 `serviceId = spring.application.name = juejin-sample` ，通过**网关** `http://${gateway}/${serviceId}/some_api` 访问**服务** `http://some_api` 。
    * PathRoutePredicateFactory 在 [《Spring-Cloud-Gateway 源码解析 —— 处理器 (3.1) 之 RoutePredicateFactory  路由谓语工厂》「10. PathRoutePredicateFactory」](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-route-predicate-factory/?self) 有详细解析。
* 第 36 至 44 行 ：使用 RewritePathGatewayFilterFactory 创建重写网关过滤器，用于移除请求路径里的 `/${serviceId}` 。如果不移除，最终请求不到服务。RewritePathGatewayFilterFactory 在 [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.2) 之 GatewayFilterFactory 过滤器工厂》「4.1 RewritePathGatewayFilterFactory」](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-factory/?self) 有详细解析。
* 第 48 行 ：返回路由定义 RouteDefinition 。举个 RouteDefinition 例子 ：![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_01_25/01.png)

# 4. 高能

本小节建议阅读完 [《Spring-Cloud-Gateway 源码解析 —— 路由（2.1）之 RouteLocator 一览》](http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-intro/?self) 再理解。

RoutePredicateHandlerMapping 使用 CachingRouteLocator 来获取 Route 信息。在 Spring Cloud Gateway 启动后，如果有新加入的服务，则需要刷新 CachingRouteLocator **缓存**。

这里有一点需要**注意**下 ：新加入的服务，指的是新的 `serviceId` ，而不是原有服务新增的实例。

个人建议，写一个定时任务，间隔调用 DiscoveryClient 获取服务列表，若发现**变化**，刷新 CachingRouteLocator **缓存**。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

😈 满足。主要因为 [「4. 高能」](#) 这小节，原来还是非常担心服务列表的缓存与刷新问题，如果不解决，网关基本属于不可用的状态。
