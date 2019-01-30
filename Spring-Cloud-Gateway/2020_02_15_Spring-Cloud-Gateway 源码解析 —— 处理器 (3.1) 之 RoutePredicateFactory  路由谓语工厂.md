title: Spring-Cloud-Gateway 源码解析 —— 处理器 (3.1) 之 RoutePredicateFactory  路由谓语工厂
date: 2020-02-15
tags:
categories: Spring-Cloud-Gateway
permalink: Spring-Cloud-Gateway/handler-route-predicate-factory

-------

摘要: 原创出处 http://www.iocoder.cn/Spring-Cloud-Gateway/handler-route-predicate-factory/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Spring-Cloud-Gateway 2.0.X M4**  

- [1. 概述](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-route-predicate-factory/)
- [2. RoutePredicateFactory](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-route-predicate-factory/)
- [3. AfterRoutePredicateFactory](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-route-predicate-factory/)
- [4. BeforeRoutePredicateFactory](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-route-predicate-factory/)
- [5. BetweenRoutePredicateFactory](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-route-predicate-factory/)
- [6. CookieRoutePredicateFactory](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-route-predicate-factory/)
- [7. HeaderRoutePredicateFactory](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-route-predicate-factory/)
- [8. HostRoutePredicateFactory](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-route-predicate-factory/)
- [9. MethodRoutePredicateFactory](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-route-predicate-factory/)
- [10. PathRoutePredicateFactory](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-route-predicate-factory/)
- [11. QueryRoutePredicateFactory](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-route-predicate-factory/)
- [12. RemoteAddrRoutePredicateFactory](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-route-predicate-factory/)
- [666. 彩蛋](http://www.iocoder.cn/Spring-Cloud-Gateway/handler-route-predicate-factory/)

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

本文主要分享 **RoutePredicateFactory 路由谓语工厂**。

RoutePredicateFactory 涉及到的类在 `org.springframework.cloud.gateway.handler.predicate` 包下，如下图 ：

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_02_15/01.png)

Spring Cloud Gateway 创建 Route 对象时，使用 RoutePredicateFactory 创建 Predicate 对象。Predicate 对象可以赋值给 [`Route.predicate`](https://github.com/YunaiV/spring-cloud-gateway/blob/6bb8d6f93c289fd3a84c802ada60dd2bb57e1fb7/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/route/Route.java#L51) 属性，用于匹配**请求**对应的 Route 。

-------

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



# 2. RoutePredicateFactory

`org.springframework.cloud.gateway.handler.predicate.RoutePredicateFactory` ， 路由谓语工厂**接口**。代码如下 ：

```Java
@FunctionalInterface
public interface RoutePredicateFactory extends ArgumentHints {

    String PATTERN_KEY = "pattern";

	Predicate<ServerWebExchange> apply(Tuple args);

	default String name() {
		return NameUtils.normalizePredicateName(getClass());
	}

}
```

* `#name()` **默认**方法，调用 `NameUtils#normalizePredicateName(Class)` 方法，获得 RoutePredicateFactory 的名字。该方法截取类名**前半段**，例如 QueryRoutePredicateFactory 的结果为 `Query` 。点击 [链接](https://github.com/YunaiV/spring-cloud-gateway/blob/6bb8d6f93c289fd3a84c802ada60dd2bb57e1fb7/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/support/NameUtils.java#L33) 查看该方法。
* `#apply()` **接口**方法，创建 Predicate 。
* 继承 `org.springframework.cloud.gateway.support.ArgumentHints` **接口** ，在 [《Spring-Cloud-Gateway 源码解析 —— 路由（2.2）之 RouteDefinitionRouteLocator 路由配置》「2.4 获得 Tuple」](http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-route-definition/?self) 有使用到它的代码。

-------

RoutePredicateFactory **实现类**如下图 ：

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_02_15/02.png)

下面我们一个一个 RoutePredicateFactory 实现类理解。代码比较多，实际也比较简单。

另外，`org.springframework.cloud.gateway.handler.predicate.RoutePredicates` ，RoutePredicates **工厂**，其调用 RoutePredicateFactory 接口的**实现类**，创建各种 Predicate 。

# 3. AfterRoutePredicateFactory

* Route 匹配 ：请求**时间**满足在配置时间**之后**。
* 配置 ：

    ```YAML
    spring:
      cloud:
        gateway:
          routes:
          # =====================================
          - id: after_route
            uri: http://example.org
            predicates:
            - After=2017-01-20T17:42:47.789-07:00[America/Denver]
    ```

* RoutePredicates 方法 ：[#after(ZonedDateTime)](https://github.com/YunaiV/spring-cloud-gateway/blob/6bb8d6f93c289fd3a84c802ada60dd2bb57e1fb7/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/handler/predicate/RoutePredicates.java#L40) 。
* 代码 ：

    ```Java
      1: public class AfterRoutePredicateFactory implements RoutePredicateFactory {
      2: 
      3: 	public static final String DATETIME_KEY = "datetime";
      4: 
      5: 	@Override
      6: 	public List<String> argNames() {
      7: 		return Collections.singletonList(DATETIME_KEY);
      8: 	}
      9: 
     10: 	@Override
     11: 	public Predicate<ServerWebExchange> apply(Tuple args) {
     12: 		Object value = args.getValue(DATETIME_KEY);
     13: 		final ZonedDateTime dateTime = BetweenRoutePredicateFactory.getZonedDateTime(value);
     14: 
     15: 		return exchange -> {
     16: 			final ZonedDateTime now = ZonedDateTime.now();
     17: 			return now.isAfter(dateTime);
     18: 		};
     19: 	}
     20: 
     21: }
    ```
    * Tulpe 参数 ：`datetime` 。
    * 第 13 行 ：调用 `BetweenRoutePredicateFactory#getZonedDateTime(value)` 方法，解析配置的时间值，在 [「5. BetweenRoutePredicateFactory」](#) 详细解析。

# 4. BeforeRoutePredicateFactory

* Route 匹配 ：请求**时间**满足在配置时间**之前**。
* RoutePredicates 方法 ：[#before(ZonedDateTime)](https://github.com/YunaiV/spring-cloud-gateway/blob/6bb8d6f93c289fd3a84c802ada60dd2bb57e1fb7/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/handler/predicate/RoutePredicates.java#L44) 。
* 配置 ：

    ```YAML
    spring:
      cloud:
        gateway:
          routes:
          # =====================================
          - id: before_route
            uri: http://example.org
            predicates:
            - Before=2017-01-20T17:42:47.789-07:00[America/Denver]
    ```

* 代码 ：

    ```Java
      1: public class BeforeRoutePredicateFactory implements RoutePredicateFactory {
      2: 
      3: 	public static final String DATETIME_KEY = "datetime";
      4: 
      5: 	@Override
      6: 	public List<String> argNames() {
      7: 		return Collections.singletonList(DATETIME_KEY);
      8: 	}
      9: 
     10: 	@Override
     11: 	public Predicate<ServerWebExchange> apply(Tuple args) {
     12: 		Object value = args.getValue(DATETIME_KEY);
     13: 		final ZonedDateTime dateTime = BetweenRoutePredicateFactory.getZonedDateTime(value);
     14: 
     15: 		return exchange -> {
     16: 			final ZonedDateTime now = ZonedDateTime.now();
     17: 			return now.isBefore(dateTime);
     18: 		};
     19: 	}
     20: 
     21: }
    ```
    * Tulpe 参数 ：`datetime` 。
    * 第 13 行 ：调用 `BetweenRoutePredicateFactory#getZonedDateTime(value)` 方法，解析配置的时间值，在 [「5. BetweenRoutePredicateFactory」](#) 详细解析。

# 5. BetweenRoutePredicateFactory

* Route 匹配 ：请求**时间**满足在配置时间**之间**。
* RoutePredicates 方法 ：[`#between(ZonedDateTime, ZonedDateTime)`](https://github.com/spring-cloud/spring-cloud-gateway/blob/9ffb0f18678460fda9b25c572c12f9054a62ca52/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/handler/predicate/RoutePredicates.java#L48) 。
* 配置 ：

    ```YAML
    spring:
      cloud:
        gateway:
          routes:
          # =====================================
          - id: between_route
            uri: http://example.org
            predicates:
            - Betweeen=2017-01-20T17:42:47.789-07:00[America/Denver], 2017-01-21T17:42:47.789-07:00[America/Denver]
    ```

* 代码 ：

    ```Java
      1: public class BetweenRoutePredicateFactory implements RoutePredicateFactory {
      2: 
      3: 	public static final String DATETIME1_KEY = "datetime1";
      4: 	public static final String DATETIME2_KEY = "datetime2";
      5: 
      6: 	@Override
      7: 	public Predicate<ServerWebExchange> apply(Tuple args) {
      8: 		//TODO: is ZonedDateTime the right thing to use?
      9: 		final ZonedDateTime dateTime1 = getZonedDateTime(args.getValue(DATETIME1_KEY));
     10: 		final ZonedDateTime dateTime2 = getZonedDateTime(args.getValue(DATETIME2_KEY));
     11: 		Assert.isTrue(dateTime1.isBefore(dateTime2), args.getValue(DATETIME1_KEY) +
     12: 				" must be before " + args.getValue(DATETIME2_KEY));
     13: 
     14: 		return exchange -> {
     15: 			final ZonedDateTime now = ZonedDateTime.now();
     16: 			return now.isAfter(dateTime1) && now.isBefore(dateTime2);
     17: 		};
     18: 	}
     19: 
     20: 	public static ZonedDateTime getZonedDateTime(Object value) {
     21: 		ZonedDateTime dateTime;
     22: 		if (value instanceof ZonedDateTime) {
     23: 			dateTime = ZonedDateTime.class.cast(value);
     24: 		} else {
     25: 			dateTime = parseZonedDateTime(value.toString());
     26: 		}
     27: 		return dateTime;
     28: 	}
     29: 
     30: 	public static ZonedDateTime parseZonedDateTime(String dateString) {
     31: 		ZonedDateTime dateTime;
     32: 		try {
     33: 			// 数字
     34: 			long epoch = Long.parseLong(dateString);
     35: 			dateTime = Instant.ofEpochMilli(epoch).atOffset(ZoneOffset.ofTotalSeconds(0))
     36: 					.toZonedDateTime();
     37: 		} catch (NumberFormatException e) {
     38: 			// 字符串
     39: 			// try ZonedDateTime instead
     40: 			dateTime = ZonedDateTime.parse(dateString);
     41: 		}
     42: 
     43: 		return dateTime;
     44: 	}
     45: 
     46: }
    ```
    * Tulpe 参数 ：`datetime1` / `datetime2` 。
    * 第 20 至 44 行 ：解析配置的时间值 。
        * 第 22 至 23 行 ：当值类型为 **ZonedDateTime** 。主要使用 Java / Kotlin 配置 Route 时，例如 [`RoutePredicates#between(ZonedDateTime, ZonedDateTime)`](https://github.com/spring-cloud/spring-cloud-gateway/blob/9ffb0f18678460fda9b25c572c12f9054a62ca52/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/handler/predicate/RoutePredicates.java#L48) 。 
        * 第 33 至 36 行 ：当值类型为 **Long** 。例如配置文件 `1511795602765`。
        * 当 38 至 41 行 ：当值类型为 **String** 。例如配置文件里 `2017-01-20T17:42:47.789-07:00[America/Denver]` 。

# 6. CookieRoutePredicateFactory

* Route 匹配 ：请求**指定 Cookie** 正则匹配**指定值**。
* RoutePredicates 方法 ：[`#cookie(String， String)`](https://github.com/spring-cloud/spring-cloud-gateway/blob/9ffb0f18678460fda9b25c572c12f9054a62ca52/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/handler/predicate/RoutePredicates.java#L48) 。
* 配置 ：

    ```YAML
    spring:
      cloud:
        gateway:
          routes:
          # =====================================
          - id: cookie_route
            uri: http://example.org
            predicates:
            - Cookie=chocolate, ch.p
    ```

* 代码 ：

    ```Java
      1: public class CookieRoutePredicateFactory implements RoutePredicateFactory {
      2: 
      3: 	public static final String NAME_KEY = "name";
      4: 	public static final String REGEXP_KEY = "regexp";
      5: 
      6: 	@Override
      7: 	public List<String> argNames() {
      8: 		return Arrays.asList(NAME_KEY, REGEXP_KEY);
      9: 	}
     10: 
     11: 	@Override
     12: 	public Predicate<ServerWebExchange> apply(Tuple args) {
     13: 		String name = args.getString(NAME_KEY);
     14: 		String regexp = args.getString(REGEXP_KEY);
     15: 
     16: 		return exchange -> {
     17: 			List<HttpCookie> cookies = exchange.getRequest().getCookies().get(name);
     18: 			for (HttpCookie cookie : cookies) {
     19: 				// 正则匹配
     20: 				if (cookie.getValue().matches(regexp)) {
     21: 					return true;
     22: 				}
     23: 			}
     24: 			return false;
     25: 		};
     26: 	}
     27: }
    ```
    * Tulpe 参数 ：`name` / `regexp` 。
    * 第 20 行 ：指定 Cookie 正则匹配**指定值**。

# 7. HeaderRoutePredicateFactory

* Route 匹配 ：请求**指定 Cookie** 正则匹配**指定值**。
* RoutePredicates 方法 ：[`#header(String， String)`](https://github.com/YunaiV/spring-cloud-gateway/blob/6bb8d6f93c289fd3a84c802ada60dd2bb57e1fb7/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/handler/predicate/RoutePredicates.java#L58) 。
* 配置 ：

    ```YAML
    spring:
      cloud:
        gateway:
          routes:
          # =====================================
          - id: header_route
            uri: http://example.org
            predicates:
            - Header=X-Request-Id, \d+
    ```

* 代码 ：

    ```Java
      1: public class HeaderRoutePredicateFactory implements RoutePredicateFactory {
      2: 
      3: 	public static final String HEADER_KEY = "header";
      4: 	public static final String REGEXP_KEY = "regexp";
      5: 
      6: 	@Override
      7: 	public List<String> argNames() {
      8: 		return Arrays.asList(HEADER_KEY, REGEXP_KEY);
      9: 	}
     10: 
     11: 	@Override
     12: 	public Predicate<ServerWebExchange> apply(Tuple args) {
     13: 		String header = args.getString(HEADER_KEY);
     14: 		String regexp = args.getString(REGEXP_KEY);
     15: 
     16: 		return exchange -> {
     17: 			List<String> values = exchange.getRequest().getHeaders().get(header);
     18: 			for (String value : values) {
     19: 				// 正则匹配
     20: 				if (value.matches(regexp)) {
     21: 					return true;
     22: 				}
     23: 			}
     24: 			return false;
     25: 		};
     26: 	}
     27: }
    ```
    * Tulpe 参数 ：`header` / `regexp` 。
    * 第 20 行 ：指定 Header 正则匹配**指定值**。

# 8. HostRoutePredicateFactory

* Route 匹配 ：请求 **Host** 匹配**指定值**。
* RoutePredicates 方法 ：[`#host(String)`](https://github.com/YunaiV/spring-cloud-gateway/blob/6bb8d6f93c289fd3a84c802ada60dd2bb57e1fb7/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/handler/predicate/RoutePredicates.java#L63) 。
* 配置 ：

    ```YAML
    spring:
      cloud:
        gateway:
          routes:
          # =====================================
          - id: host_route
            uri: http://example.org
            predicates:
            - Host=**.somehost.org
    ```

* 代码 ：

    ```Java
      1: public class HostRoutePredicateFactory implements RoutePredicateFactory {
      2: 
      3: 	private PathMatcher pathMatcher = new AntPathMatcher(".");
      4: 
      5: 	public void setPathMatcher(PathMatcher pathMatcher) {
      6: 		this.pathMatcher = pathMatcher;
      7: 	}
      8: 
      9: 	@Override
     10: 	public List<String> argNames() {
     11: 		return Collections.singletonList(PATTERN_KEY);
     12: 	}
     13: 
     14: 	@Override
     15: 	public Predicate<ServerWebExchange> apply(Tuple args) {
     16: 		String pattern = args.getString(PATTERN_KEY);
     17: 
     18: 		return exchange -> {
     19: 			String host = exchange.getRequest().getHeaders().getFirst("Host");
     20: 			// 匹配
     21: 			return this.pathMatcher.match(pattern, host);
     22: 		};
     23: 	}
     24: }
    ```
    * Tulpe 参数 ：`pattern` 。
    * `pathMatcher` 属性，路径匹配器，默认使用 `org.springframework.util.AntPathMatcher` 。通过 `#setPathMatcher(PathMatcher)` 方法，可以重新设置。
    * 第 21 行 ：请求**路径** 匹配**指定值**。

# 9. MethodRoutePredicateFactory

* Route 匹配 ：请求 **Method** 匹配**指定值**。
* RoutePredicates 方法 ：[`#method(String)`](https://github.com/YunaiV/spring-cloud-gateway/blob/6bb8d6f93c289fd3a84c802ada60dd2bb57e1fb7/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/handler/predicate/RoutePredicates.java#L67) 。
* 配置 ：

    ```YAML
    spring:
      cloud:
        gateway:
          routes:
          # =====================================
          - id: method_route
            uri: http://example.org
            predicates:
            - Method=GET
    ```

* 代码 ：

    ```Java
      1: public class MethodRoutePredicateFactory implements RoutePredicateFactory {
      2: 
      3: 	public static final String METHOD_KEY = "method";
      4: 
      5: 	@Override
      6: 	public List<String> argNames() {
      7: 		return Arrays.asList(METHOD_KEY);
      8: 	}
      9: 
     10: 	@Override
     11: 	public Predicate<ServerWebExchange> apply(Tuple args) {
     12: 		String method = args.getString(METHOD_KEY);
     13: 		return exchange -> {
     14: 			HttpMethod requestMethod = exchange.getRequest().getMethod();
     15: 			// 正则匹配
     16: 			return requestMethod.matches(method);
     17: 		};
     18: 	}
     19: }
    ```
    * Tulpe 参数 ：`method` 。
    * 第 16 行 ：请求 **Method** 匹配**指定值**。

# 10. PathRoutePredicateFactory

* Route 匹配 ：请求 **Path** 匹配**指定值**。
* RoutePredicates 方法 ：[`#path(String, String)`](https://github.com/spring-cloud/spring-cloud-gateway/blob/9ffb0f18678460fda9b25c572c12f9054a62ca52/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/handler/predicate/RoutePredicates.java#L71) 。
* 配置 ：

    ```YAML
    spring:
      cloud:
        gateway:
          routes:
          # =====================================
          - id: host_route
            uri: http://example.org
            predicates:
            - Path=/foo/{segment}
    ```

* 代码 ：

    ```Java
      1: public class PathRoutePredicateFactory implements RoutePredicateFactory {
      2: 
      3: 	private PathPatternParser pathPatternParser = new PathPatternParser();
      4: 
      5: 	public void setPathPatternParser(PathPatternParser pathPatternParser) {
      6: 		this.pathPatternParser = pathPatternParser;
      7: 	}
      8: 
      9: 	@Override
     10: 	public List<String> argNames() {
     11: 		return Collections.singletonList(PATTERN_KEY);
     12: 	}
     13: 
     14: 	@Override
     15: 	public Predicate<ServerWebExchange> apply(Tuple args) {
     16: 	    // 解析 Path ，创建对应的 PathPattern
     17: 		String unparsedPattern = args.getString(PATTERN_KEY);
     18: 		PathPattern pattern;
     19: 		synchronized (this.pathPatternParser) {
     20: 			pattern = this.pathPatternParser.parse(unparsedPattern);
     21: 		}
     22: 
     23: 		return exchange -> {
     24: 			PathContainer path = parsePath(exchange.getRequest().getURI().getPath());
     25: 
     26: 			// 匹配
     27: 			boolean match = pattern.matches(path);
     28: 			traceMatch("Pattern", pattern.getPatternString(), path, match);
     29: 			if (match) {
     30: 				// 解析 路径参数，例如 path=/foo/123 <=> /foo/{segment}
     31: 				PathMatchInfo uriTemplateVariables = pattern.matchAndExtract(path);
     32: 				exchange.getAttributes().put(URI_TEMPLATE_VARIABLES_ATTRIBUTE, uriTemplateVariables);
     33: 				return true;
     34: 			}
     35: 			else {
     36: 				return false;
     37: 			}
     38: 		};
     39: 	}
     40: }
    ```
    * Tulpe 参数 ：`pattern` 。
    * `pathPatternParser` 属性，路径模式解析器。
    * 第 17 至 21 行 ：解析**配置**的 Path ，创建对应的 PathPattern 。考虑到解析过程中的**线程安全**，此处使用 `synchronized` 修饰符，详见 [`PathPatternParser#parse(String)`](https://github.com/spring-projects/spring-framework/blob/master/spring-web/src/main/java/org/springframework/web/util/pattern/PathPatternParser.java#L99) 方法的注释。
    * 第 24 至 27 行 ：解析**请求**的 Path ，匹配**配置**的 Path 。
    * 第 30 至 32 行 ：解析**路径参数**，设置到 `ServerWebExchange.attributes` 属性中，提供给后续的 GatewayFilter 使用。举个例子，当配置的 Path 为 `/foo/{segment}` ，请求的 Path 为 `/foo/123` ，在此处打断点，结果如下图 ：![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_02_15/03.png)
    
        > FROM [《Spring Cloud Gateway》](https://github.com/spring-cloud/spring-cloud-gateway/blob/9ffb0f18678460fda9b25c572c12f9054a62ca52/docs/src/main/asciidoc/spring-cloud-gateway.adoc#path-route-predicate-factory)   
        > This predicate extracts the URI template variables (like `segment` defined in the example above) as a map of names and values and places it in the `ServerWebExchange.getAttributes()` with a key defined in `PathRoutePredicate.URL_PREDICATE_VARS_ATTR`. Those values are then available for use by [GatewayFilter Factories](https://github.com/spring-cloud/spring-cloud-gateway/blob/9ffb0f18678460fda9b25c572c12f9054a62ca52/docs/src/main/asciidoc/spring-cloud-gateway.adoc#setpath-gatewayfilter-factory)

# 11. QueryRoutePredicateFactory

* Route 匹配 ：请求 **QueryParam** 匹配**指定值**。
* RoutePredicates 方法 ：[`#query(String, String)`](https://github.com/YunaiV/spring-cloud-gateway/blob/6bb8d6f93c289fd3a84c802ada60dd2bb57e1fb7/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/handler/predicate/RoutePredicates.java#L75) 。
* 配置 ：

    ```YAML
    spring:
      cloud:
        gateway:
          routes:
          # =====================================
          - id: query_route
            uri: http://example.org
            predicates:
            - Query=baz
            - Query=foo, ba.
    ```

* 代码 ：

    ```Java
      1: public class QueryRoutePredicateFactory implements RoutePredicateFactory {
      2: 
      3: 	public static final String PARAM_KEY = "param";
      4: 	public static final String REGEXP_KEY = "regexp";
      5: 
      6: 	@Override
      7: 	public List<String> argNames() {
      8: 		return Arrays.asList(PARAM_KEY, REGEXP_KEY);
      9: 	}
     10: 
     11: 	@Override
     12: 	public boolean validateArgs() {
     13: 		return false;
     14: 	}
     15: 
     16: 	@Override
     17: 	public Predicate<ServerWebExchange> apply(Tuple args) {
     18: 		validateMin(1, args);
     19: 		String param = args.getString(PARAM_KEY);
     20: 
     21: 		return exchange -> {
     22: 		    // 包含 参数
     23: 			if (!args.hasFieldName(REGEXP_KEY)) {
     24: 				// check existence of header
     25: 				return exchange.getRequest().getQueryParams().containsKey(param);
     26: 			}
     27: 
     28: 			// 正则匹配 参数
     29: 			String regexp = args.getString(REGEXP_KEY);
     30: 			List<String> values = exchange.getRequest().getQueryParams().get(param);
     31: 			for (String value : values) {
     32: 				if (value.matches(regexp)) {
     33: 					return true;
     34: 				}
     35: 			}
     36: 			return false;
     37: 		};
     38: 	}
     39: }
    ```
    * Tulpe 参数 ：`param` ( 必填 ) / `regexp` ( 选填 ) 。
    * 第 18 行 ：调用 `#validateMin(...)` 方法，校验参数数量至少为 `1` ，即 `param` 非空 。
    * 第 22 至 26 行 ：当 `regexp` 为空时，校验 `param` 对应的 `QueryParam` 存在。
    * 第 28 至 35 行 ：当 `regexp` 非空时，请求 `param` 对应的 **QueryParam** 正则匹配**指定值**。
        * 当 `QueryParams` 为空时，会报空指针 **BUG** 。

# 12. RemoteAddrRoutePredicateFactory

* Route 匹配 ：请求**来源 IP** 在**指定范围内**。
* RoutePredicates 方法 ：[`#remoteAddr(String...)`](https://github.com/YunaiV/spring-cloud-gateway/blob/6bb8d6f93c289fd3a84c802ada60dd2bb57e1fb7/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/handler/predicate/RoutePredicates.java#L80) 。
* 配置 ：

    ```YAML
    spring:
      cloud:
        gateway:
          routes:
          # =====================================
          - id: remoteaddr_route
            uri: http://example.org
            predicates:
            - RemoteAddr=192.168.1.1/24
    ```
    
* 代码 ：    

    ```Java
      1: public class RemoteAddrRoutePredicateFactory implements RoutePredicateFactory {
      2: 
      3: 	private static final Log log = LogFactory.getLog(RemoteAddrRoutePredicateFactory.class);
      4: 
      5: 	@Override
      6: 	public Predicate<ServerWebExchange> apply(Tuple args) {
      7: 		validate(1, args);
      8: 
      9: 		//
     10: 		List<SubnetUtils> sources = new ArrayList<>();
     11: 		if (args != null) {
     12: 			for (Object arg : args.getValues()) {
     13: 				addSource(sources, (String) arg);
     14: 			}
     15: 		}
     16: 
     17: 		return exchange -> {
     18: 			InetSocketAddress remoteAddress = exchange.getRequest().getRemoteAddress();
     19: 			if (remoteAddress != null) {
     20: 				// 来源 IP
     21: 				String hostAddress = remoteAddress.getAddress().getHostAddress();
     22: 				String host = exchange.getRequest().getURI().getHost();
     23: 				if (!hostAddress.equals(host)) {
     24: 					log.warn("Remote addresses didn't match " + hostAddress + " != " + host);
     25: 				}
     26: 
     27: 				//
     28: 				for (SubnetUtils source : sources) {
     29: 					if (source.getInfo().isInRange(hostAddress)) {
     30: 						return true;
     31: 					}
     32: 				}
     33: 			}
     34: 
     35: 			return false;
     36: 		};
     37: 	}
     38: 
     39: 	private void addSource(List<SubnetUtils> sources, String source) {
     40: 		boolean inclusiveHostCount = false;
     41: 		if (!source.contains("/")) { // no netmask, add default
     42: 			source = source + "/32";
     43: 		}
     44: 		if (source.endsWith("/32")) {
     45: 			//http://stackoverflow.com/questions/2942299/converting-cidr-address-to-subnet-mask-and-network-address#answer-6858429
     46: 			inclusiveHostCount = true;
     47: 		}
     48: 		//TODO: howto support ipv6 as well?
     49: 		SubnetUtils subnetUtils = new SubnetUtils(source);
     50: 		subnetUtils.setInclusiveHostCount(inclusiveHostCount);
     51: 		sources.add(subnetUtils);
     52: 	}
     53: }
    ```
    * Tulpe 参数 ：字符串**数组**。
    * 第 7 行 ：调用 `#validateMin(...)` 方法，校验参数数量至少为 `1` ，字符串数组**非空**。
    * 第 10 至 15 行 ：使用 **SubnetUtils** 工具类，解析配置的值。
    * 第 21 至 25 行 ：获得请求来源 IP 。
    * 第 28 至 32 行 ：请求**来源 IP** 在**指定范围内**。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

😈 代码好多，贴的手都抽了。嘿嘿，RemoteAddrRoutePredicateFactory 写的有点偷懒。

胖友，分享一波朋友圈可好！

