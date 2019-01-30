title: Spring-Cloud-Gateway 源码解析 —— 路由（2.2）之 RouteDefinitionRouteLocator 路由配置
date: 2020-02-01
tags:
categories: Spring-Cloud-Gateway
permalink: Spring-Cloud-Gateway/route-locator-route-definition

---

摘要: 原创出处 http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-route-definition/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Spring-Cloud-Gateway 2.0.X M4**  

- [1. 概述](http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-route-definition/)
- [2. RouteDefinitionRouteLocator](http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-route-definition/)
  - [2.1 构造方法](http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-route-definition/)
  - [2.2 获得 Route](http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-route-definition/)
  - [2.3 转换 Route](http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-route-definition/)
  - [2.4 获得 Tuple](http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-route-definition/)
- [666. 彩蛋](http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-route-definition/)

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

本文主要分享 **RouteDefinitionRouteLocator 的源码实现**。

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_02_01/01.jpeg)

* **蓝色**部分 ：RouteDefinitionRouteLocator 。

-------

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



# 2. RouteDefinitionRouteLocator

`org.springframework.cloud.gateway.route.RouteDefinitionRouteLocator` ，基于 **RouteDefinitionLocator** 的 RouteLocator **实现类**。

RouteDefinitionRouteLocator 从 RouteDefinitionLocator 获取 RouteDefinition ，转换成 Route 。如下图 ：

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_02_01/02.png)

## 2.1 构造方法

RouteDefinitionRouteLocator 构造方法，代码如下 ：

```Java
  1: public class RouteDefinitionRouteLocator implements RouteLocator, BeanFactoryAware {
  2: 	protected final Log logger = LogFactory.getLog(getClass());
  3: 
  4: 	private final RouteDefinitionLocator routeDefinitionLocator;
  5: 	/**
  6: 	 * RoutePredicateFactory 映射
  7: 	 * key ：{@link RoutePredicateFactory#name()}
  8: 	 */
  9: 	private final Map<String, RoutePredicateFactory> predicates = new LinkedHashMap<>();
 10: 	/**
 11: 	 * GatewayFilterFactory 映射
 12: 	 * key ：{@link GatewayFilterFactory#name()}
 13: 	 */
 14: 	private final Map<String, GatewayFilterFactory> gatewayFilterFactories = new HashMap<>();
 15: 	private final GatewayProperties gatewayProperties;
 16: 	private final SpelExpressionParser parser = new SpelExpressionParser();
 17: 	private BeanFactory beanFactory;
 18: 
 19: 	public RouteDefinitionRouteLocator(RouteDefinitionLocator routeDefinitionLocator,
 20: 									   List<RoutePredicateFactory> predicates,
 21: 									   List<GatewayFilterFactory> gatewayFilterFactories,
 22: 									   GatewayProperties gatewayProperties) {
 23: 		// 设置 RouteDefinitionLocator
 24: 		this.routeDefinitionLocator = routeDefinitionLocator;
 25: 		// 初始化 RoutePredicateFactory
 26: 		initFactories(predicates);
 27: 		// 初始化 RoutePredicateFactory
 28: 		gatewayFilterFactories.forEach(factory -> this.gatewayFilterFactories.put(factory.name(), factory));
 29: 		// 设置 GatewayProperties
 30: 		this.gatewayProperties = gatewayProperties;
 31: 	}
 32: 
 33: 	@Override
 34: 	public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
 35: 		this.beanFactory = beanFactory;
 36: 	}
 37:}
```

* `routeDefinitionLocator` 属性，提供 RouteDefinition 的 RouteDefinitionLocator 。
* `predicates` 属性，RoutePredicateFactory Bean 对象**映射**。
    * `key` 为 `{@link RoutePredicateFactory#name()}` 。
    * 通过它，将 `RouteDefinition.predicates` 转换成 `Route.predicates` 。
    * 第 26 行 ：调用 `#initFactories()` 方法，初始化映射。逻辑比较简单，点击 [链接](https://github.com/YunaiV/spring-cloud-gateway/blob/6bb8d6f93c289fd3a84c802ada60dd2bb57e1fb7/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/route/RouteDefinitionRouteLocator.java#L90) 查看代码。
* `gatewayFilterFactories` 属性，RoutePredicateFactory Bean 对象**映射**。
    * `key` 为 `{@link GatewayFilterFactory#name()}` 。
    * 通过它，将 `RouteDefinition.filters` 转换成 `Route.filters` 。
    * 第 28 行 ：初始化映射。
* `gatewayProperties` 属性，使用 `GatewayProperties.defaultFilters` **默认过滤器定义数组**，添加到每个 Route 。下文会看到相关代码的实现。
* `parser` 属性，Spring EL 表达式解析器。在 [「2.4 获得 Tuple」](#) 会看到它的使用。
* `beanFactory` 属性，Bean 工厂。

## 2.2 获得 Route

`#getRoutes()` 方法，获得 Route 数组。代码如下 ：

```Java
  1: @Override
  2: public Flux<Route> getRoutes() {
  3: 	return this.routeDefinitionLocator.getRouteDefinitions()
  4: 			.map(this::convertToRoute) // RouteDefinition => Route
  5: 			//TODO: error handling
  6: 			.map(route -> { // 打印日志
  7: 				if (logger.isDebugEnabled()) {
  8: 					logger.debug("RouteDefinition matched: " + route.getId());
  9: 				}
 10: 				return route;
 11: 			});
 12: 
 13: }
```

* 第 3 行 ： 调用 `RouteDefinitionLocator#getRouteDefinitions()` 方法，获得 RouteDefinitions 数组。
* 第 4 行 ：调用 `#convertToRoute()` 方法，将每个 RouteDefinition **转换**成  Route 。该方法在 [「2.3 转换 Route」](#) 详细解析。
* 第 7 至 11 行 ：打印**输出**每个 Route 。

## 2.3 转换 Route

`#convertToRoute()` 方法，将每个 RouteDefinition **转换**成  Route 。代码如下 ：

```Java
  1: private Route convertToRoute(RouteDefinition routeDefinition) {
  2:     // 合并 Predicate
  3: 	Predicate<ServerWebExchange> predicate = combinePredicates(routeDefinition);
  4: 	// 获得 GatewayFilter
  5: 	List<GatewayFilter> gatewayFilters = getFilters(routeDefinition);
  6: 	// 构建 Route
  7: 	return Route.builder(routeDefinition)
  8: 			.predicate(predicate)
  9: 			.gatewayFilters(gatewayFilters)
 10: 			.build();
 11: }
```

* 第 3 行 ：调用 `#combinePredicates()` 方法，将 `RouteDefinition.predicates` **数组**合并成**一个** `java.util.function.Predicate` ，这样 RoutePredicateHandlerMapping 为请求**匹配** Route ，只要调用**一次** `Predicate#test(ServerWebExchange)` 方法即可。
* 第 5 行 ：调用 `#getFilters()` 方法，获得 GatewayFilter **数组**。
* 第 7 至 10 行 ：构建 Route 。

-------

`#combinePredicates()` 方法，代码如下 ：

```Java
  1: private Predicate<ServerWebExchange> combinePredicates(RouteDefinition routeDefinition) {
  2:     // 寻找 Predicate
  3: 	List<PredicateDefinition> predicates = routeDefinition.getPredicates();
  4: 	Predicate<ServerWebExchange> predicate = lookup(routeDefinition, predicates.get(0));
  5: 	// 拼接 Predicate
  6: 	for (PredicateDefinition andPredicate : predicates.subList(1, predicates.size())) {
  7: 		Predicate<ServerWebExchange> found = lookup(routeDefinition, andPredicate);
  8: 		predicate = predicate.and(found);
  9: 	}
 10: 	// 返回 Predicate
 11: 	return predicate;
 12: }
 13: 
 14: private Predicate<ServerWebExchange> lookup(RouteDefinition routeDefinition, PredicateDefinition predicate) {
 15:     // 获得 RoutePredicateFactory
 16: 	RoutePredicateFactory found = this.predicates.get(predicate.getName());
 17: 	if (found == null) {
 18: 		throw new IllegalArgumentException("Unable to find RoutePredicateFactory with name " + predicate.getName());
 19: 	}
 20: 	// 获得 Tuple
 21: 	Map<String, String> args = predicate.getArgs();
 22: 	if (logger.isDebugEnabled()) {
 23: 		logger.debug("RouteDefinition " + routeDefinition.getId() + " applying "
 24: 				+ args + " to " + predicate.getName());
 25: 	}
 26: 	Tuple tuple = getTuple(found, args, this.parser, this.beanFactory);
 27: 	// 获得 Predicate
 28: 	return found.apply(tuple);
 29: }
```

* 第 2 至 9 行 ：通过调用 `#lookup()` 方法，查找 PredicateDefinition 对应的 Predicate 。为什么拆成**两部分**？第一部分找到 `java.util.function.Predicate` ，第二部分通过 `Predicate#and(Predicate)` 方法不断拼接。
* 第 11 行 ：返回 Predicate 。
* ---------------------------- 分割线 --------------------------
* 第 14 至 29 行 ：`#lookup()` 方法。
    * 第 16 至 19 行 ：获得 RoutePredicateFactory Bean 对象。
    * 第 21 至 26 行 ：调用 `#getTuple()` 方法，获得 Tuple 。该方法比较复杂，在 [「2.4 获得 Tuple」](#) 详细解析。
    * 第 28 行 ：调用 `RoutePredicateFactory#apply(Tuple)` 方法，创建 Predicate 。在 [《Spring-Cloud-Gateway 源码解析 —— 处理器 (3.1) 之 RoutePredicateFactory  路由谓语工厂 》](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-route-predicate-factory/?self) 详细解析。

-------

`#getFilters()` 方法，代码如下 ：

```Java
  1: private List<GatewayFilter> getFilters(RouteDefinition routeDefinition) {
  2: 	List<GatewayFilter> filters = new ArrayList<>();
  3: 	// 添加 默认过滤器
  4: 	//TODO: support option to apply defaults after route specific filters?
  5: 	if (!this.gatewayProperties.getDefaultFilters().isEmpty()) {
  6: 		filters.addAll(loadGatewayFilters("defaultFilters",
  7: 				this.gatewayProperties.getDefaultFilters()));
  8: 	}
  9: 	// 添加 配置的过滤器
 10: 	if (!routeDefinition.getFilters().isEmpty()) {
 11: 		filters.addAll(loadGatewayFilters(routeDefinition.getId(), routeDefinition.getFilters()));
 12: 	}
 13: 	// 排序
 14: 	AnnotationAwareOrderComparator.sort(filters);
 15: 	return filters;
 16: }
 17: 
 18: private List<GatewayFilter> loadGatewayFilters(String id, List<FilterDefinition> filterDefinitions) {
 19: 	List<GatewayFilter> filters = filterDefinitions.stream()
 20: 			.map(definition -> { // FilterDefinition => GatewayFilter
 21: 			    // 获得 GatewayFilterFactory
 22: 				GatewayFilterFactory filter = this.gatewayFilterFactories.get(definition.getName());
 23: 				if (filter == null) {
 24: 					throw new IllegalArgumentException("Unable to find GatewayFilterFactory with name " + definition.getName());
 25: 				}
 26: 				// 获得 Tuple
 27: 				Map<String, String> args = definition.getArgs();
 28: 				if (logger.isDebugEnabled()) {
 29: 					logger.debug("RouteDefinition " + id + " applying filter " + args + " to " + definition.getName());
 30: 				}
 31: 				Tuple tuple = getTuple(filter, args, this.parser, this.beanFactory);
 32: 				// 获得 GatewayFilter
 33: 				return filter.apply(tuple);
 34: 			})
 35: 			.collect(Collectors.toList()); // 转成 List
 36:    // GatewayFilter => OrderedGatewayFilter
 37: 	ArrayList<GatewayFilter> ordered = new ArrayList<>(filters.size());
 38: 	for (int i = 0; i < filters.size(); i++) {
 39: 		ordered.add(new OrderedGatewayFilter(filters.get(i), i+1));
 40: 	}
 41: 	// 返回 GatewayFilter 数组
 42: 	return ordered;
 43: }
```

* 第 4 至 8 行 ：调用 `#loadGatewayFilters()` 方法，使用 `GatewayProperties.defaultFilters` **默认**的过滤器配置，将 FilterDefinition 转换成 GatewayFilter 。
* 第 10 至 12 行 ：调用 `#loadGatewayFilters()` 方法，使用 `RouteDefinition.filters` **配置**的过滤器配置，将 FilterDefinition 转换成 GatewayFilter 。
* ---------------------------- 分割线 --------------------------
* 第 18 至 43 行 ：`#loadGatewayFilters()` 方法。
    * 第 20 至 34 行 ：将 FilterDefinition 转换成 GatewayFilter 。
        * 第 21 至 25 行 ：获得 GatewayFilterFactory Bean 对象。
        * 第 27 至 31 行 ：调用 `#getTuple()` 方法，获得 Tuple 。该方法比较复杂，在 [「2.4 获得 Tuple」](#) 详细解析。
        * 第 33 行 ：创建 GatewayFilter 。
    * 第 35 行 ：获得 GatewayFilter 数组。
* 第 37 至 40 行 ：将 GatewayFilter 数组**转换成** OrderedGatewayFilter 数组。在 [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.1) 之 GatewayFilter 一览》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-intro/?self) 详细解析。
* 第 42 行 ：返回 GatewayFilter **数组**。

## 2.4 获得 Tuple 

在看 `#getTuple()` 方法的代码实现之前，我们先了解下 Tuple 。

Tuple ，**定义**如下 ：

> FROM [《简单实现 Java 的 Tuple 元组数据类型》](https://unmi.cc/simple-java-tuple-datatype/)  
> 元组类型，即 Tuple 常在脚本语言中出现，例如 Scala 的 ("Unmi", "fantasia@sina.com", "blahbla")  
> 元组可认为是象数组一样的容器，它的目的是让你方便构造和引用，例如 Pair 可认为是一个只能存两个元素的元组，像是个 Map  
> 真正的元组应该是可以任意多个元素的容器，绕来绕去，它还是数组，或列表，所以我们实现上还是要借助于数组或是列表。

截止目前，Java 并未内置 Tuple 的实现。Spring 提供了 `spring-tuple` **类库**，提供了 Tuple 的支持。使用**示例**如下 ：

```Java
// 1 对
Tuple tuple = tuple().of("foo", "bar");
// 2 对
Tuple tuple2 = tuple().of("up", 1, "down", 2);
// 3 对
Tuple tuple3 = tuple().of("up", 1, "down", 2, "charm", 3 );
// 4 对
Tuple tuple4 = tuple().of("up", 1, "down", 2, "charm", 3, "strange", 4);
// 6 对 ( 适用于超过 4 对 )
Tuple tuple6 = tuple().put("up", 1)
                      .put("down", 2)
        		      .put("charm", 3)
        		      .put("strange", 4)
        		      .put("bottom", 5)
        		      .put("top", 6)
        		      .build();
```

* 更多 `spring-tuple` 的文档，请见 [《Spring Docs —— Tuples》](https://docs.spring.io/spring-xd/docs/0.1.x-SNAPSHOT/reference/html/_tuples.html) 。

-------

那么**为什么** `RoutePredicateFactory#apply(Tuple)` / `GatewayFilterFactory#apply(Tuple)` 需要使用 Tuple 呢 ？RoutePredicateFactory / GatewayFilterFactory **子类实现类**需要成对的参数不同，例如 ：

* [`org.springframework.cloud.gateway.filter.factory.SetStatusGatewayFilterFactory`](https://github.com/YunaiV/spring-cloud-gateway/blob/382a4cd98fbb8ac53a83a5559bacb0f885838074/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/filter/factory/SetStatusGatewayFilterFactory.java#L40) ，使用 `status` **一对**参数。
*  [`org.springframework.cloud.gateway.filter.factory.SetResponseHeaderGatewayFilterFactory`](https://github.com/YunaiV/spring-cloud-gateway/blob/382a4cd98fbb8ac53a83a5559bacb0f885838074/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/filter/factory/SetResponseHeaderGatewayFilterFactory.java#L33) ，使用 `name` / `value` **两对**参数。

-------

OK ，我们开始看看 `#getTuple()` 方法，代码如下 ：

```Java
  1: /* for testing */ static Tuple getTuple(ArgumentHints hasArguments, Map<String, String> args, SpelExpressionParser parser, BeanFactory beanFactory) {
  2: 	TupleBuilder builder = TupleBuilder.tuple();
  3: 
  4: 	// 参数为空
  5: 	List<String> argNames = hasArguments.argNames();
  6: 	if (!argNames.isEmpty()) {
  7: 		// ensure size is the same for key replacement later
  8: 		if (hasArguments.validateArgs() && args.size() != argNames.size()) {
  9: 			throw new IllegalArgumentException("Wrong number of arguments. Expected " + argNames
 10: 					+ " " + argNames + ". Found " + args.size() + " " + args + "'");
 11: 		}
 12: 	}
 13: 
 14: 	// 创建 Tuple
 15: 	int entryIdx = 0;
 16: 	for (Map.Entry<String, String> entry : args.entrySet()) {
 17: 	    // 获得参数 KEY
 18: 		String key = entry.getKey();
 19: 		// RoutePredicateFactory has name hints and this has a fake key name
 20: 		// replace with the matching key hint
 21: 		if (key.startsWith(NameUtils.GENERATED_NAME_PREFIX) && !argNames.isEmpty()
 22: 				&& entryIdx < args.size()) {
 23: 			key = argNames.get(entryIdx);
 24: 		}
 25: 		// 获得参数 VALUE
 26: 		Object value;
 27: 		String rawValue = entry.getValue();
 28: 		if (rawValue != null) {
 29: 			rawValue = rawValue.trim();
 30: 		}
 31: 		if (rawValue != null && rawValue.startsWith("#{") && entry.getValue().endsWith("}")) {
 32: 			// assume it's spel
 33: 			StandardEvaluationContext context = new StandardEvaluationContext();
 34: 			context.setBeanResolver(new BeanFactoryResolver(beanFactory));
 35: 			Expression expression = parser.parseExpression(entry.getValue(), new TemplateParserContext());
 36: 			value = expression.getValue(context);
 37: 		} else {
 38: 			value = entry.getValue();
 39: 		}
 40: 		// 添加 KEY / VALUE
 41: 		builder.put(key, value);
 42: 		entryIdx++;
 43: 	}
 44: 	Tuple tuple = builder.build();
 45: 
 46: 	// 校验参数
 47: 	if (hasArguments.validateArgs()) {
 48: 		for (String name : argNames) {
 49: 			if (!tuple.hasFieldName(name)) {
 50: 				throw new IllegalArgumentException("Missing argument '" + name + "'. Given " + tuple);
 51: 			}
 52: 		}
 53: 	}
 54: 	return tuple;
 55: }
```

* `hasArguments` **参数**，点击 [`org.springframework.cloud.gateway.support.ArgumentHints`](https://github.com/YunaiV/spring-cloud-gateway/blob/382a4cd98fbb8ac53a83a5559bacb0f885838074/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/support/ArgumentHints.java) 查看代码实现。RoutePredicateFactory / GatewayFilterFactory 实现 ArgumentHints **接口**。
* 第 5 至 12 行 ：**校验**参数是否正确( 需要参数**非空** )。
* 第 15 至 44 行 ：创建 Tuple 。
    * 第 18 至 24 行 ：获得**一对**参数 KEY 。
    * 第 26 至 39 行 ：获得**一对**参数 VALUE 。
    * 第 41 行 ：添加**一对**参数 KEY / VALUE 。我们在此处打断点，看看此时各变量的值，路由配置如下 ：

        ```YAML
        spring:
          cloud:
            gateway:
              routes:
              - id: websocket_test
                uri: ws://localhost:9000
                order: 9000
                predicates:
                - Path=/echo
                - Query=foo, ba.
        ```
        * PATH 
            * `/echo` ![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_02_01/03.png)
        * Query 
            * `foo` ![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_02_01/04.png)
            * `ba.` ![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_02_01/05.png)
    
    * 第 44 行 ：创建 Tuple 。
* 第 47 至 53 行 ：**校验**参数是否正确( 需要参数**都存在** )。
* 第 54 行 ：返回 Tuple 。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

恩恩，干了一些的文章。这个周末还是木有完成计划写的文章。继续加油！

胖友，分享一波朋友圈可好！

