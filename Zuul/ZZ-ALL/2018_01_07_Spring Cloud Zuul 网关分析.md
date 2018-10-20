title: Spring Cloud Zuul 网关分析
date: 2018-01-07
tag: 
categories: Zuul
permalink: Zuul/laoyuan/spring-cloud-zuul-gateway-tutorial
author: 老袁
from_url: https://laoyuan.me/posts/spring-cloud-zuul-gateway-tutorial.html
wechat_url: 

-------

摘要: 原创出处 https://laoyuan.me/posts/spring-cloud-zuul-gateway-tutorial.html 「老袁」欢迎转载，保留摘要，谢谢！

  - [前话](http://www.iocoder.cn/Zuul/laoyuan/spring-cloud-zuul-gateway-tutorial/)
  - [源码分析](http://www.iocoder.cn/Zuul/laoyuan/spring-cloud-zuul-gateway-tutorial/)
  - [启动流程](http://www.iocoder.cn/Zuul/laoyuan/spring-cloud-zuul-gateway-tutorial/)
  - [扩展](http://www.iocoder.cn/Zuul/laoyuan/spring-cloud-zuul-gateway-tutorial/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

目前微服务越来越流行，Spring Cloud算得上目前比较流行的微服务治理框架，从Spring移植应用过来也非常方便。服务多了，对外部调用客户端不可能每个服务都给出一个对外服务的主机地址，很多客户端也不具备加入Spring注册中心的条件，如果服务升级，服务地址变动，则需要通知的客户端太多，且容易出错。
最初对网关的理解来源于Nginx的反向代理，通过修改配置文件实现服务聚合与分流。上述的问题是可以通过Nginx的反向代理聚合服务，修改服务路径，新旧服务版本流量控制，不过每次都需要修改配置文件。
不过Spring Cloud已经提供了对网关（基于Zuul）的支持，那没有理由不去使用Spring Cloud Zuul来实现服务的聚合和管理。对于开发人员来说，动态调整网关代理配置，能动态配置最好，如果每次变动都需要联系运维审批修改Nginx配置来实现，确实太费周章。

## 前话

Spring Cloud 网关是基于Netflix开发的Zuul组件实现的，Zuul核心的功能就是一堆的过滤器，通过对请求的过滤，则可以实现对权限认证，路由转发，请求参数变更，流量控制等功能。下面是一张官方的Zuul请求生命周期图：
![Zuul请求生命周期](https://laoyuan.me/statics/posts/2017/07/zuul-request-lifecycle.png)
Zuul的过滤器主要定义了四种不同生命周期的过滤器类型：

| 类型    | 描述                             |
| ------- | -------------------------------- |
| pre     | 可以在请求被路由之前调用         |
| routing | 在路由请求时候被调用             |
| post    | 在routing和error过滤器之后被调用 |
| error   | 处理请求时发生错误时被调用       |

每个过滤器主要需要实现如下抽象方法或接口：

```Java
/**
	 * 是否使用此过滤器
	 */
	boolean shouldFilter();

	/**
	 * 需要执行的过滤操作
	 */
	Object run();

	/**
	 * 过滤器类型
	 */
	String filterType();

	/**
	 * 过滤器执行顺序
	 */
	int filterOrder();
```

## 源码分析

Spring Cloud Gateway则对Zuul进行二次封装，对上述的四种生命周期都进行了对应的实现：

- PRE:

| 名称                   | 顺序 | 用途                                                         |
| ---------------------- | ---- | ------------------------------------------------------------ |
| ServletDetectionFilter | -3   | 判断后续处理的Servlet:DispatcherServlet（Spting）、ZuulServlet（zuul） |
| Servlet30WrapperFilter | -2   | 将原始的HttpServletRequest封装成Servlet30RequestWrapper对象  |
| FormBodyWrapperFilter  | -1   | 将Servlet30RequestWrapper封装成FormBodyRequestWrapper对象    |
| DebugFilter            | 1    | 处理请求中是否带debug参数，来开启后续过滤器是否打印调试信息  |
| PreDecorationFilter    | 5    | 最核心的也是最后一个PRE过滤器，此处查询路由配置，在请求中设置转发信息 |

- ROUTING:

| 名称                    | 顺序 | 用途                           |
| ----------------------- | ---- | ------------------------------ |
| RibbonRoutingFilter     | 10   | 处理Cloud服务类型的路由配置    |
| SimpleHostRoutingFilter | 100  | 处理代理地址连接类型的路由配置 |
| SendForwardFilter       | 500  | 处理转发类型的路由配置         |

- POST:

| 名称               | 顺序 | 用途                   |
| ------------------ | ---- | ---------------------- |
| SendErrorFilter    | 10   | 返回客户端错误响应     |
| SendResponseFilter | 100  | 返回客户端代理结果响应 |

## 启动流程

Spring Cloud提供了网关的自动配置（ZuulProxyConfiguration），自动配置主要实现如下功能：

- 读取yml中的路由配置

- 根据路由配置生成RouteLocator，提供给`PreDecorationFilter`根据URL查找对于的路由配置

- 初始化上述列表中的各阶段过滤器

  ## 使用教程

- 配置路由规则

  ```
  server:
    port: 4100  #配置网关端口

  spring:
    application:
      name: my-gateway     # 指定网关微服务名
  eureka:
    client:
      serviceUrl:
        defaultZone: http://localhost:8010/eureka/     # 指定服务注册中心的地址
  zuul:
    routes: # 配置路由规则，可以配置服务代理规则、本地转发规则、地址代理规则
      service-my-api:
        path: /api/**
        serviceId: my-api
      service-my-api-1:
        path: /api-1/**
        serviceId: my-api-1
  ```

- 启动网关

  ```Java
  @SpringBootApplication
  @EnableZuulProxy //开启网关代理
  public class MyGatewayApplication {
  	public static void main(String[] args) {
  		SpringApplication.run(MyGatewayApplication.class, args);
  	}
  }
  ```

## 扩展

Spring Cloud Zuul提供的默认路由方式，小规模的应用应该足够了，不过对于需要动态路由配置的网关，则不能满足需求，所以需要我们对网关进行扩展，主要有以下扩展思路：

- 扩展`DiscoveryClientRouteLocator`，即管理路由规则的类，默认是读取配置文件和注册中心的ServiceId自动生成路由规则，则这里可以修改为基于Zookeeper或Redis的动态配置路由规则定位器。
- 扩展`PreDecorationFilter`，即路由匹配设置过滤器，默认是使用`DiscoveryClientRouteLocator`进行路由规则匹配，则可以扩展为基于正则匹配路由配置等操作。
- 扩展`SendErrorFilter`，即代理错误响应过滤器，默认的错误响应或许对前端客户端不友好，则可以扩展封装错误信息。
- 新增`PRE`类型的前置过滤器，可以考虑进行客户端调用的权限认证与权限控制，也可以考虑对客户端的调用频率进行限制。

# 666. 彩蛋

如果你对 Zuul  感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)