title: Spring-Cloud-Gateway 源码解析 —— 路由（1.1）之 RouteDefinitionLocator 一览
date: 2020-01-10
tags:
categories: Spring-Cloud-Gateway
permalink: Spring-Cloud-Gateway/route-definition-locator-intro

---

摘要: 原创出处 http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-intro/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Spring-Cloud-Gateway 2.0.X M4**  

- [1. 概述](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-intro/)
- [2. RouteDefinition](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-intro/)
- [3. PredicateDefinition](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-intro/)
- [4. FilterDefinition](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-intro/)
- [5. RouteDefinitionLocator](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-intro/)
- [6. CompositeRouteDefinitionLocator](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-intro/)
- [666. 彩蛋](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-intro/)

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

本文主要对 **路由定义定位器 RouteDefinitionLocator 做整体的认识**。

在 [《Spring-Cloud-Gateway 源码解析 —— 网关初始化》](http://www.iocoder.cn/Spring-Cloud-Gateway/init/?self) 中，我们看到路由相关的组件 RouteDefinitionLocator / RouteLocator 的初始化。涉及到的类比较多，我们用下图重新梳理下 ：

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_01_10/01.png)

* **RouteDefinitionLocator** 负责读取路由配置( `org.springframework.cloud.gateway.route.RouteDefinition` ) 。从上图中我们可以看到，RouteDefinitionLocator **接口**有四种实现 ：
    * **PropertiesRouteDefinitionLocator** ，从**配置文件**( 例如，YML / Properties 等 ) 读取。在 [《Spring-Cloud-Gateway 源码解析 —— 路由（1.2）之 PropertiesRouteDefinitionLocator 配置文件》「2. PropertiesRouteDefinitionLocator」](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-properties?self) 详细解析。
    * **RouteDefinitionRepository** ，从**存储器**( 例如，内存 / Redis / MySQL 等 )读取。在 [《Spring-Cloud-Gateway 源码解析 —— 路由（1.3）之 RouteDefinitionRepository 存储器》](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-repository/?self) 详细解析。
    * **DiscoveryClientRouteDefinitionLocator** ，从**注册中心**( 例如，Eureka / Consul / Zookeeper / Etcd 等 )读取。在 [《Spring-Cloud-Gateway 源码解析 —— 路由（1.4）之 DiscoveryClientRouteDefinitionLocator 注册中心》](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-discover-client/?self) 详细解析。
    * **CompositeRouteDefinitionLocator** ，组合**多种** RouteDefinitionLocator 的实现，为 RouteDefinitionRouteLocator 提供**统一**入口。在 [本文](#) 详细解析。
    * 另外，**CachingRouteDefinitionLocator** 也是 RouteDefinitionLocator 的实现类，已经被 CachingRouteLocator 取代。
* **RouteLocator** 可以直接**自定义**路由( `org.springframework.cloud.gateway.route.Route` ) ，也可以通过 RouteDefinitionRouteLocator 获取 RouteDefinition ，并转换成 Route 。
* **重要**，对于上层调用者 RoutePredicateHandlerMapping ，使用的是 RouteLocator 和 Route 。而 RouteDefinitionLocator 和 RouteDefinition 用于通过**配置定义路由**。**那么自定义 RouteLocator 呢**？通过**代码定义路由**。

-------

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



# 2. RouteDefinition

`org.springframework.cloud.gateway.route.RouteDefinition` ，路由定义。代码如下 ：

```Java
@Validated
public class RouteDefinition {

	@NotEmpty
	private String id = UUID.randomUUID().toString();
    /**
     * 谓语定义数组
     */
	@NotEmpty
	@Valid
	private List<PredicateDefinition> predicates = new ArrayList<>();
    /**
     * 过滤器定义数组
     */
	@Valid
	private List<FilterDefinition> filters = new ArrayList<>();
    /**
     * 路由向的 URI
     */
	@NotNull
	private URI uri;
    /**
     * 顺序
     */
	private int order = 0;
}	
```

* `id` 属性，ID 编号，**唯一**。
* `predicates` 属性，谓语定义数组。**请求**通过 `predicates` 判断是否**匹配**。在 Route 里，PredicateDefinition 转换成 Predicate 。
* `filters` 属性，过滤器定义数组。在 Route 里，FilterDefinition 转换成 GatewayFilter 。
* `uri` 属性，路由向的 URI 。
* `order` 属性，顺序。当请求匹配到多个路由时，使用顺序**小**的。
* ![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_01_10/03.png)

-------

RouteDefinition 提供 `text` 字符串创建对象，代码如下 ：

```Java
/**
 * 根据 text 创建 RouteDefinition
 *
 * @param text 格式 ${id}=${uri},${predicates[0]},${predicates[1]}...${predicates[n]}
 *             例如 route001=http://127.0.0.1,Host=**.addrequestparameter.org,Path=/get
 */
public RouteDefinition(String text) {
    int eqIdx = text.indexOf("=");
    if (eqIdx <= 0) {
        throw new ValidationException("Unable to parse RouteDefinition text '" + text + "'" +
				", must be of the form name=value");
	}
    // id
    setId(text.substring(0, eqIdx));
    // predicates
	String[] args = tokenizeToStringArray(text.substring(eqIdx+1), ",");
    // uri
	setUri(URI.create(args[0]));

	for (int i=1; i < args.length; i++) {
		this.predicates.add(new PredicateDefinition(args[i]));
	}
}
```

* `text` 参数，格式为 `${id}=${uri},${predicates[0]},${predicates[1]}...${predicates[n]}` 。举个例子, `"route001=http://127.0.0.1,Host=**.addrequestparameter.org,Path=/get"` 。创建的 RouteDefinition 如下图 ：![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_01_10/04.png)
* `filters` 属性，需要通过调用 `RouteDefinition#setFilters(filters)` 方法进行设置。
* `order` 属性，需要通过调用 `RouteDefinition#setOrder(order)` 方法进行设置。
* `predicates` 属性，支持解析，但是如果此处**单个** PredicateDefinition 的 `args[i]` 存在**逗号**( `,` ) ，会被错误的分隔，例如说，`"Query=foo,bz"` 。

# 3. PredicateDefinition

`org.springframework.cloud.gateway.handler.predicate.PredicateDefinition` ，谓语定义。**请求**通过 `predicates` 判断是否**匹配**。代码如下 ：

```Java
@Validated
public class PredicateDefinition {

    /**
     * 谓语定义名字
     */
    @NotNull
	private String name;
    /**
     * 参数数组
     */
	private Map<String, String> args = new LinkedHashMap<>();
}
```

* `name` 属性，谓语定义名字。通过 `name` 对应到 `org.springframework.cloud.gateway.handler.predicate.RoutePredicateFactory` 的**实现类**。例如说，`name=Query` 对应到 QueryRoutePredicateFactory 。
* `args` 属性，参数数组。例如，`name=Host` / `args={"_genkey_0" : "iocoder.cn"}` ，匹配请求的 `hostname` 为 `iocoder.cn` 。

-------

PredicateDefinition 提供 `text` 字符串创建对象，代码如下 ：

```Java
/**
 * 根据 text 创建 PredicateDefinition
 *
 * @param text 格式 ${name}=${args[0]},${args[1]}...${args[n]}
 *             例如 Host=iocoder.cn
 */
public PredicateDefinition(String text) {
    int eqIdx = text.indexOf("=");
    if (eqIdx <= 0) {
		throw new ValidationException("Unable to parse PredicateDefinition text '" + text + "'" +
				", must be of the form name=value");
	}
	// name
	setName(text.substring(0, eqIdx));
	// args
	String[] args = tokenizeToStringArray(text.substring(eqIdx+1), ",");
	for (int i=0; i < args.length; i++) {
		this.args.put(NameUtils.generateName(i), args[i]);
	}
}
```

* `text` 参数，格式为 `${name}=${args[0]},${args[1]}...${args[n]}` 。举个例子, `"Host=iocoder.cn"` 。创建的 PredicateDefinition 如下图 ：![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_01_10/05.png)

# 4. FilterDefinition

FilterDefinition 和 PredicateDefinition 的代码实现上**基本一致**。

`org.springframework.cloud.gateway.filter.FilterDefinition` ，过滤器定义。代码如下 ：

```Java
@Validated
public class FilterDefinition {

    /**
     * 过滤器定义名字
     */
	@NotNull
	private String name;
    /**
     * 参数数组
     */
	private Map<String, String> args = new LinkedHashMap<>();
}
```

* `name` 属性，过滤器定义名字。通过 `name` 对应到 `org.springframework.cloud.gateway.filter.factory.GatewayFilterFactory` 的**实现类**。例如说，`name=AddRequestParameter` 对应到 AddRequestParameterGatewayFilterFactory 。
* `args` 属性，参数数组。例如，`name=AddRequestParameter` / `args={"_genkey_0": "foo", "_genkey_1": "bar"}` ，添加请求参数 `foo` 为 `bar` 。

-------

FilterDefinition 提供 `text` 字符串创建对象，代码如下 ：

```Java
/**
 * 根据 text 创建 FilterDefinition
 *
 * @param text 格式 ${name}=${args[0]},${args[1]}...${args[n]}
 *             例如 AddRequestParameter=foo, bar
 */
public FilterDefinition(String text) {
	int eqIdx = text.indexOf("=");
	if (eqIdx <= 0) {
		setName(text);
		return;
	}
	// name
	setName(text.substring(0, eqIdx));
	// args
	String[] args = tokenizeToStringArray(text.substring(eqIdx+1), ",");
	for (int i=0; i < args.length; i++) {
		this.args.put(NameUtils.generateName(i), args[i]);
	}
}
```

* `text` 参数，格式为 `${name}=${args[0]},${args[1]}...${args[n]}` 。举个例子, `"AddRequestParameter=foo, bar"` 。创建的 FilterDefinition 如下图 ：![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_01_10/06.png)

# 5. RouteDefinitionLocator

`org.springframework.cloud.gateway.route.RouteDefinitionLocator` ，路由定义定位器**接口**，定义获得路由定义数组的方法。代码如下 ：

```Java
public interface RouteDefinitionLocator {

	Flux<RouteDefinition> getRouteDefinitions();
}
```

* 对 Reactor Flux 暂时不熟悉的同学，可以阅读完本文 Google 进行学习。随着 Spring 5 对响应式编程的推广，厉害如你一定要去掌握。

在上文中，我们也看到了 RouteDefinitionLocator 的多个实现类，类图如下 ：

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_01_10/07.png)

* 本文只解析 CompositeRouteDefinitionLocator 的源码实现。其他的实现类会在后面文章详细解析。

# 6. CompositeRouteDefinitionLocator

`org.springframework.cloud.gateway.route.CompositeRouteDefinitionLocator` ，组合**多种** RouteDefinitionLocator 的实现，为 RouteDefinitionRouteLocator 提供**统一**入口。代码如下 ：

```Java
public class CompositeRouteDefinitionLocator implements RouteDefinitionLocator {

    /**
     * RouteDefinitionLocator 数组
     */
	private final Flux<RouteDefinitionLocator> delegates;

	public CompositeRouteDefinitionLocator(Flux<RouteDefinitionLocator> delegates) {
		this.delegates = delegates;
	}

	@Override
	public Flux<RouteDefinition> getRouteDefinitions() {
		return this.delegates.flatMap(RouteDefinitionLocator::getRouteDefinitions);
	}

}
```

* `#getRouteDefinitions()` 方法，提供**统一**方法，将组合的 `delegates` 的路由定义**全部**返回。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

RouteDefinition => Route  
PredicateDefinition => Predication  
FilterDefinition => GatewayFilter  

等等的转换，我们在后续路由相关的文章详细解析。

胖友，分享一波朋友圈可好！


