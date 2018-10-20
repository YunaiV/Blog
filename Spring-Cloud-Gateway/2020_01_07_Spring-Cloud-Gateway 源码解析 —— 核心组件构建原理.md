title: Spring-Cloud-Gateway 源码解析 —— 核心组件构建原理
date: 2020-01-07
tag:
categories: Spring-Cloud-Gateway
permalink: Spring-Cloud-Gateway/ouwenxue/intro
author:  欧文雪
from_url: https://mp.weixin.qq.com/s/jCQR1WkOsiOzozMAeIPk3g
wechat_url:

-------

摘要: 原创出处 https://mp.weixin.qq.com/s/jCQR1WkOsiOzozMAeIPk3g 「欧文雪」欢迎转载，保留摘要，谢谢！

- [引言](http://www.iocoder.cn/Spring-Cloud-Gateway/ouwenxue/intro/)
- [1. 准备知识](http://www.iocoder.cn/Spring-Cloud-Gateway/ouwenxue/intro/)
  - [1.1. Gateway 相关技术栈](http://www.iocoder.cn/Spring-Cloud-Gateway/ouwenxue/intro/)
  - [1.2. Gateway 工作机制](http://www.iocoder.cn/Spring-Cloud-Gateway/ouwenxue/intro/)
- [2. 基本组件介绍](http://www.iocoder.cn/Spring-Cloud-Gateway/ouwenxue/intro/)
  - [2.1. Route](http://www.iocoder.cn/Spring-Cloud-Gateway/ouwenxue/intro/)
  - [2.2. AsyncPredicate](http://www.iocoder.cn/Spring-Cloud-Gateway/ouwenxue/intro/)
  - [2.3. GatewayFilter](http://www.iocoder.cn/Spring-Cloud-Gateway/ouwenxue/intro/)
- [3. 构建 Route](http://www.iocoder.cn/Spring-Cloud-Gateway/ouwenxue/intro/)
  - [3.1. 外部化配置](http://www.iocoder.cn/Spring-Cloud-Gateway/ouwenxue/intro/)
  - [3.2. 编程方式](http://www.iocoder.cn/Spring-Cloud-Gateway/ouwenxue/intro/)
- [4. Route 构建的原理](http://www.iocoder.cn/Spring-Cloud-Gateway/ouwenxue/intro/)
  - [4.1.  GatewayProperties](http://www.iocoder.cn/Spring-Cloud-Gateway/ouwenxue/intro/)
  - [4.2. RouteDefinition](http://www.iocoder.cn/Spring-Cloud-Gateway/ouwenxue/intro/)
  - [4.3. FilterDefinition](http://www.iocoder.cn/Spring-Cloud-Gateway/ouwenxue/intro/)
  - [4.4. PredicateDefinition](http://www.iocoder.cn/Spring-Cloud-Gateway/ouwenxue/intro/)
  - [4.5.  RoutePredicateFactory](http://www.iocoder.cn/Spring-Cloud-Gateway/ouwenxue/intro/)
  - [4.6.  GatewayFilterFactory](http://www.iocoder.cn/Spring-Cloud-Gateway/ouwenxue/intro/)
  - [这文章这么长（chou），您竟然能坚持看到这里，厉害了。](http://www.iocoder.cn/Spring-Cloud-Gateway/ouwenxue/intro/)
  - [4.7. Predicate 示例由浅入深](http://www.iocoder.cn/Spring-Cloud-Gateway/ouwenxue/intro/)
  - [4.8. RouteLocator](http://www.iocoder.cn/Spring-Cloud-Gateway/ouwenxue/intro/)
  - [4.9. RouteDefinitionRouteLocator](http://www.iocoder.cn/Spring-Cloud-Gateway/ouwenxue/intro/)
- [5. 小结](http://www.iocoder.cn/Spring-Cloud-Gateway/ouwenxue/intro/)
- [6. 附录](http://www.iocoder.cn/Spring-Cloud-Gateway/ouwenxue/intro/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# 引言

在当下学习和使用 spring cloud 技术栈的热潮中，网关已经成了不可或缺的内容。开发者在选择用来解决特定领域内问题的框架时，多了解几款相关同类产品可加大选择余地。除了 Netflix 的 zuul 之外，spring cloud gateway 可作为开发者的另一个选择。

> Zuul 分 1.x 和 2.x 版本。Zuul 2.x 版本和 spring cloud gateway 都使用 Non-Blocking I/O 模型。

本文主要从源码上介绍 spring cloud gateway 的核心组件以及构建这些组件的原理。只有正确地构建相关组件，网关才能正常运行，在了解相关底层知识点后，在框架的使用上会更加得心应手。

# 1. 准备知识

## 1.1. Gateway 相关技术栈

开发者了解以下技术，学习 gateway 会更加容易。

① **project reactor**，遵循 Reactive Streams Specification，使用非阻塞编程模型。

② **webflux**，基于 spring 5.x 和 `reactor-netty` 构建，不依赖于 Servlet 容器，但是可以在支持 Servlet 3.1 Non-Blocking IO API 的容器上运行。

## 1.2. Gateway 工作机制

Spring cloud gateway 的工作机制大体如下：

① Gateway 接收客户端请求。

② 客户端请求与路由信息进行匹配，匹配成功的才能够被发往相应的下游服务。

③ 请求经过 Filter 过滤器链，执行 pre 处理逻辑，如修改请求头信息等。

④ 请求被转发至下游服务并返回响应。

⑤ 响应经过 Filter 过滤器链，执行 post 处理逻辑。

⑥ 向客户端响应应答。

Gateway 工作机制可参考下图（图片来自官方文档）：

![Spring Cloud Gateway Diagram](http://static.iocoder.cn/87f0b41dec5709fe3768bf7601650941)

这里提到了路由、匹配、过滤器等概念，下文将作详细介绍 ，这也是本文需要阐述的重点。

# 2. 基本组件介绍

## 2.1. Route

Route 是 gateway 中最基本的组件之一，表示一个具体的路由信息载体。

`Route`源代码：

```java
public class Route implements Ordered {

    private final String id; // ①

    private final URI uri; // ②

    private final int order; // ③

    private final AsyncPredicate<ServerWebExchange> predicate; // ④

    private final List<GatewayFilter> gatewayFilters; // ⑤
```

Route 主要定义了如下几个部分：

①  **id**，标识符，区别于其他 Route。

②  **destination uri**，路由指向的目的地 uri，即客户端请求最终被转发的目的地。

③  **order**，用于多个 Route 之间的排序，数值越小排序越靠前，匹配优先级越高。

④  **predicate**，谓语，表示匹配该 Route 的前置条件，即满足相应的条件才会被路由到目的地 uri。

⑤  **gateway filters**，过滤器用于处理切面逻辑，如路由转发前修改请求头等。

## 2.2. AsyncPredicate

Predicate 即 Route 中所定义的部分，用于条件匹配，请参考 Java 8 提供的  Predicate 和 Function。

`AsyncPredicate`源代码：

```java
public interface AsyncPredicate<T> extends Function<T, Publisher<Boolean>> {

    default AsyncPredicate<T> and(AsyncPredicate<? super T> other) { // ①
        Objects.requireNonNull(other, "other must not be null");
        return t -> Flux.zip(apply(t), other.apply(t))
                .map(tuple -> tuple.getT1() && tuple.getT2());
    }

    default AsyncPredicate<T> negate() { // ②
        return t -> Mono.from(apply(t)).map(b -> !b);
    }

    default AsyncPredicate<T> or(AsyncPredicate<? super T> other) { // ③
        Objects.requireNonNull(other, "other must not be null");
        return t -> Flux.zip(apply(t), other.apply(t))
                .map(tuple -> tuple.getT1() || tuple.getT2());
    }
}
```

AsyncPredicate 定义了 3 种逻辑操作方法：

①  **and** ，与操作，即两个 Predicate 组成一个，需要同时满足。

②  **negate**，取反操作，即对 Predicate 匹配结果取反。

③  **or**，或操作，即两个 Predicate 组成一个，只需满足其一。

## 2.3. GatewayFilter

很多框架都有 Filter 的设计，用于实现可扩展的切面逻辑。

`GatewayFilter` 源代码：

```java
public interface GatewayFilter extends ShortcutConfigurable {

    String NAME_KEY = "name";
    String VALUE_KEY = "value";

    /**
     * Process the Web request and (optionally) delegate to the next
     * {@code WebFilter} through the given {@link GatewayFilterChain}.
     * @param exchange the current server exchange
     * @param chain provides a way to delegate to the next filter
     * @return {@code Mono<Void>} to indicate when request processing is complete
     */
    Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);
}
```

Filter 最终是通过 filter chain 来形成链式调用的，每个 filter 处理完 pre filter 逻辑后委派给  filter chain，filter chain 再委派给下一下 filter。

`GatewayFilterChain` 源代码：

```java
public interface GatewayFilterChain {

    /**
     * Delegate to the next {@code WebFilter} in the chain.
     * @param exchange the current server exchange
     * @return {@code Mono<Void>} to indicate when request handling is complete
     */
    Mono<Void> filter(ServerWebExchange exchange);

}
```

# 3. 构建 Route

如何构建 Route 组件？

Spring 提供了两种方式：外部化配置和编程的方式。

> **注** ：详细的使用说明请参考 spring cloud gateway 官方文档，这并不是本文的阐述重点。

## 3.1. 外部化配置

Spring boot 提供了强大的外部化配置功能，基本上每个 starter 模块都有提供相应的支持，开发者在应用程序中使用相应模块时可根据需要来进行调整。

### 3.1.1. Hello World

Spring cloud gateway 提供了一些开箱即用的 Predicate 和 Filter，它们通过工厂模式来生产。

下面这个例子由官方文档中的两个示例组合而成。

```yaml
spring:
  cloud:
    gateway: # ①
      routes: # ②
      - id: cookie_route # ③
        uri: http://example.org # ④
        predicates: # ⑤
        - Cookie=chocolate, ch.p # ⑥
        filters: # ⑦
        - AddRequestHeader=X-Request-Foo, Bar # ⑧
```

详细说明：

① "spring.cloud.gateway" 为固定前缀。

② 定义路由信息列表，即可定义多个路由。

③ 声明了一个 id 为 "cookie_route" 的路由。

④ 定义了路由的目的地 uri，即请求转发的目的地。

⑤ 声明 predicates，即请求满足相应的条件才能匹配成功。

⑥ 定义了一个 Predicate，当名称为 `chocolate` 的 Cookie  的值匹配`ch.p`时 Predicate 才能够匹配，它由 CookieRoutePredicateFactory 来生产。

⑦ 声明 filters，即路由转发前后处理的过滤器。

⑧ 定义了一个 Filter，所有的请求转发至下游服务时会添加请求头 `X-Request-Foo:Bar` ，由AddRequestHeaderGatewayFilterFactory 来生产。

文字表达不够直观，附本示例对应的流程图：

![scg-demo](http://static.iocoder.cn/997c8c889791e9e1d9a30d68d4b35f92)

## 3.2. 编程方式

开发者还可以通过编程的方式来定义 Route，编程的方式会更加灵活。

### 3.2.1.  Hello world

通过 fluent API RouteLocatorBuilder 来构建 RouteLocator。

示例（根据官方文档改造）：

```java
// static imports from GatewayFilters and RoutePredicates
@Bean
public RouteLocator customRouteLocator(RouteLocatorBuilder builder) { // ①
    return builder.routes() // ②
            .route(r -> r.host("**.abc.org").and().path("/image/png") // ③
                .filters(f ->
                        f.addResponseHeader("X-TestHeader", "foobar")) // ④
                .uri("http://httpbin.org:80") // ⑤
            )
            .build();
}
```

① RouteLocatorBuilder bean 在 spring-cloud-starter-gateway 模块自动装配类中已经声明，可直接使用。RouteLocator 封装了对 Route 获取的定义，可简单理解成工厂模式。

② RouteLocatorBuilder 可以构建多个路由信息。

③ 指定了 Predicates，这里包含两个：

1. 请求头`Host`需要匹配`**.abc.org`，通过 HostRoutePredicateFactory 产生。
2. 请求路径需要匹配`/image/png`，通过 PathRoutePredicateFactory 产生。

④ 指定了一个 Filter，下游服务响应后添加响应头`X-TestHeader:foobar`，通过AddResponseHeaderGatewayFilterFactory 产生。

⑤ 指定路由转发的目的地 uri。

如果说外部化配置完全是个黑盒，那么通过编程的方式使开发者向白盒靠近了一步。因为开发者需要使用 gateway 的 api，需要开发者对其有工作机制有所了解。

# 4. Route 构建的原理

外部化配置是如何工作的？

Spring boot 遵循规约大于配置的原则，starter 模块都有对应的以模块名称作前缀，以 "AutoConfiguration" 后缀的自动装配类。同样的还有以模块名前缀，以`Properties`后缀的配置类作为支持。

Gateway 模块自动装配类为 GatewayAutoConfiguration，对应的配置类为 GatewayProperties。

> **注**：想了解外部化配置（application.yaml 等）如绑定到对应的配置类对象的请参阅 spring 相关源码，亦不是本文的阐述重点。

## 4.1.  GatewayProperties

GatewayProperties 是 Spring cloud gateway 模块提供的外部化配置类。

源码：

```java
@ConfigurationProperties("spring.cloud.gateway") // ①
@Validated
public class GatewayProperties {

    /**
     * List of Routes
     */
    @NotNull
    @Valid
    private List<RouteDefinition> routes = new ArrayList<>(); // ②

    /**
     * List of filter definitions that are applied to every route.
     */
    private List<FilterDefinition> defaultFilters = new ArrayList<>(); // ③
}
```

①  表明以 "spring.cloud.gateway" 前缀的 properties 会绑定 GatewayProperties。

②  用来对 Route 进行定义。

③  用于定义默认的 Filter 列表，默认的 Filter 会应用到每一个 Route 上，gateway 处理时会将其与 Route 中指定的 Filter 进行合并后并逐个执行。

## 4.2. RouteDefinition

顾名思义，该组件用来对 Route 信息进行定义，最终会被 RouteLocator 解析成 Route。

源码：

```java
public class RouteDefinition {
    @NotEmpty
    private String id = UUID.randomUUID().toString(); // ①

    @NotEmpty
    @Valid
    private List<PredicateDefinition> predicates = new ArrayList<>();  // ②

    @Valid
    private List<FilterDefinition> filters = new ArrayList<>();  // ③

    @NotNull
    private URI uri;  // ④

    private int order = 0; // ⑤
}
```

① 定义 Route 的 id，默认使用 UUID。

② 定义 Predicate。

③ 定义 Filter。

④ 定义目的地 URI。

⑤ 定义 Route 的序号。

可见，RouteDefinition 中所定义的属性与 Route 本身是一一对应的。

## 4.3. FilterDefinition

同样遵循组件名前缀 + `Definition` 后缀的命名规范，用于定义 Filter。

源码：

```java
public class FilterDefinition {
    @NotNull
    private String name; // ①
    private Map<String, String> args = new LinkedHashMap<>(); // ②
```

① 定义了 Filter 的名称，符合特定的命名规范，为对应的工厂名前缀。

② 一个键值对参数用于构造 Filter 对象。

### 4.3.1. 外部配置到 FilterDefinition 对象绑定

以 AddRequestHeader GatewayFilter Factory 为例:

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: add_request_header_route
        uri: http://example.org
        filters:
        - AddRequestHeader=X-Request-Foo, Bar # ①
```

① 这一行配置被 spring 解析后会绑定到一个 FilterDefinition 对象。

**AddRequestHeader** ，对应 FilterDefinition 中的 `name` 属性。`AddRequestHeader`为AddRequestHeaderGatewayFilterFactory 的类名前缀。

**X-Request-Foo, Bar** ，会被解析成 FilterDefinition 中的 Map 类型属性 `args`。此处会被解析成两组键值对，以英文逗号将`=`后面的字符串分隔成数组，`key`是固定字符串 `_genkey_`  + 数组元素下标，`value`为数组元素自身。

相关源码：

```java
// FilterDefinition 构造函数
public FilterDefinition(String text) {
    int eqIdx = text.indexOf("=");
    if (eqIdx <= 0) {
        setName(text);
        return;
    }
    setName(text.substring(0, eqIdx));
    String[] args = tokenizeToStringArray(text.substring(eqIdx+1), ",");
    for (int i=0; i < args.length; i++) {
        this.args.put(NameUtils.generateName(i), args[i]); // ①
    }
}

// ① 使用到的工具类 NameUtils 源码
public class NameUtils {
    public static final String GENERATED_NAME_PREFIX = "_genkey_";
    public static String generateName(int i) {
        return GENERATED_NAME_PREFIX + i;
    }
}
```

## 4.4. PredicateDefinition

同样遵循组件名前缀 + `Definition` 后缀的命名规范，用于定义 Predicate。

源码：

```java
public class PredicateDefinition {
    @NotNull
    private String name; // ①
    private Map<String, String> args = new LinkedHashMap<>(); // ②
```

① 定义了 Predicate 的名称，它们要符固定的命名规范，为对应的工厂名称。

② 一个 Map 类型的参数，构造 Predicate 使用到的键值对参数。

外部化配置绑定到 PredicateDefinition 源码逻辑与 FilterDefinition 类似，不再赘述。

## 4.5.  RoutePredicateFactory

RoutePredicateFactory 是所有 predicate factory 的顶级接口，职责就是生产 Predicate。

创建一个用于配置用途的对象（config），以其作为参数应用到 `apply`方法上来生产一个 Predicate 对象，再将 Predicate 对象包装成 AsyncPredicate。

源码：

```java
import static org.springframework.cloud.gateway.support.ServerWebExchangeUtils.toAsyncPredicate;

@FunctionalInterface // ①
public interface RoutePredicateFactory<C> extends ShortcutConfigurable,
    Configurable<C> { // ④

    Predicate<ServerWebExchange> apply(C config); // ②

    default AsyncPredicate<ServerWebExchange> applyAsync(C config) { // ③
        return toAsyncPredicate(apply(config));
    }
}
// RoutePredicateFactory 扩展了 Configurable
public interface Configurable<C> {
    Class<C> getConfigClass();  // ⑤
    C newConfig(); // ⑥
}
```

**注：**为保持排版简洁，方便阅读，只贴出了源码中最重要的部分，略去了部分内容。

① 声明它是一个函数接口。

② 核心方法，即函数接口的唯一抽象方法，用于生产 Predicate，接收一个范型参数  config。

③ 对参数 config 应用工厂方法，并将返回结果 Predicate 包装成 AsyncPredicate。包装成 AsyncPredicate 是为了使用非阻塞模型。

④ 扩展了 Configurable 接口，从命名上可以推断 Predicate 工厂是支持配置的。

⑤ 获取配置类的类型，支持范型，具体的 config 类型由子类指定。

⑥ 创建一个 config 实例，由具体的实现类来完成。

## 4.6.  GatewayFilterFactory

GatewayFilterFactory 职责就是生产 GatewayFilter。

源码：

```java
@FunctionalInterface
public interface GatewayFilterFactory<C> extends ShortcutConfigurable,
    Configurable<C> { // ①
    String NAME_KEY = "name";
    String VALUE_KEY = "value";

    GatewayFilter apply(C config); // ②
}
```

① 同样继承了 ShortcutConfigurable 和 Configurable 接口，支持配置。

② 核心方法，用于生产 GatewayFilter，接收一个范型参数 config 。

------

## 这文章这么长（chou），您竟然能坚持看到这里，厉害了。

前面花了大量篇幅介绍 gateway 基础组件的定义，内容十分乏味，接下来以一个例子来逐步剖析外部化配置究竟是如何被转换成 Route 对象本身的。

------

## 4.7. Predicate 示例由浅入深

结合 Predicate 示例（来自官方文档）来说明，以 After Route Predicate Factory 为例：

它匹配当前日期时间之后产生的请求，仅需要提供一个时间参数。

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: after_route
        uri: http://example.org
        predicates:
        - After=2017-01-20T17:42:47.789-07:00[America/Denver] # ①
```

仅匹配发生在 2017-01-20 17:42 北美山区时区 (Denver) 之后的请求。

① 结合上文内容，可得知这一行配置会被绑定至 PredicateDefinition 对象，将其可视化：

```json
// 可将下面内容想象成 PredicateDefinition 对象 toString() 方法返回结果
PredicateDefinition {
    name='After',
    args={_genkey_0=2017-01-20T17:42:47.789-07:00[America/Denver]}
}
```

AfterRoutePredicateFactory 源码：

```java
public class AfterRoutePredicateFactory extends AbstractRoutePredicateFactory<AfterRoutePredicateFactory.Config> { // ①
    public static final String DATETIME_KEY = "datetime";

    public AfterRoutePredicateFactory() {
        super(Config.class);
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return Collections.singletonList(DATETIME_KEY);
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {  // ②
        ZonedDateTime datetime = getZonedDateTime(config.getDatetime());
        return exchange -> {
            final ZonedDateTime now = ZonedDateTime.now();
            return now.isAfter(datetime);
        };
    }

    public static class Config { // ③
        private String datetime;
        public String getDatetime() {
            return datetime;
        }
        public void setDatetime(String datetime) {
            this.datetime = datetime;
        }
    }
}
```

①  声明了范型，即使用到的配置类为 AfterRoutePredicateFactory 中定义的内部类 Config。

②  生产 Predicate 对象，逻辑是判断当前时间（执行时）是否在 Config 中指定的 `datetime`之后。

③  该配置类只包含一个`datetime`时间字符串属性。

**疑问**：PredicateDefinition 对象又是如何转换成 AfterRoutePredicateFactory.Config 对象的？

至此，还是没有说明是如何转换的，这要涉及 RouteLocator 组件。

## 4.8. RouteLocator

从名称上来推断是 Route 的定位器或者说探测器，是用来获取 Route 信息的。

源码：

```java
public interface RouteLocator {
    Flux<Route> getRoutes(); // ①
}
```

①  源码传达的含义十分简单，获取 Route。

外部化配置定义 Route 使用的是 RouteDefinition 组件。同样的也有配套的 RouteDefinitionLocator 组件。

源码：

```java
public interface RouteDefinitionLocator {
    Flux<RouteDefinition> getRouteDefinitions(); // ①
}
```

① 源码也很简单，其职责是获取 RouteDefinition。

## 4.9. RouteDefinitionRouteLocator

RouteLocator 最主要的实现类，用于将 RouteDefinition  转换成 Route。

### 4.9.1. 构造函数

构造函数源码：

```java
public class RouteDefinitionRouteLocator implements RouteLocator, BeanFactoryAware, ApplicationEventPublisherAware {
    private final RouteDefinitionLocator routeDefinitionLocator;
    private final Map<String, RoutePredicateFactory> predicates = new LinkedHashMap<>();
    private final Map<String, GatewayFilterFactory> gatewayFilterFactories = new HashMap<>(); private final GatewayProperties gatewayProperties;
    private final SpelExpressionParser parser = new SpelExpressionParser();
    private BeanFactory beanFactory;
    private ApplicationEventPublisher publisher;

    public RouteDefinitionRouteLocator(RouteDefinitionLocator routeDefinitionLocator,// ①
        List<RoutePredicateFactory> predicates, // ②
        List<GatewayFilterFactory> gatewayFilterFactories, // ③
        GatewayProperties gatewayProperties) { // ④
        this.routeDefinitionLocator = routeDefinitionLocator;
        initFactories(predicates);
        gatewayFilterFactories.forEach(factory -> this.gatewayFilterFactories.put(factory.name(), factory));
        this.gatewayProperties = gatewayProperties;
}
```

构造函数依赖 4 个对象，分别是：

① **RouteDefinition Locator**，一个 RouteDefinitionLocator 对象。

② **predicates factories**，Predicate 工厂列表，会被映射成 `key` 为 name, `value` 为 factory 的 Map。可以猜想出 gateway 是如何根据 PredicateDefinition 中定义的 `name` 来匹配到相对应的 factory 了。

③ **filter factories**，Gateway Filter 工厂列表，同样会被映射成 `key` 为 name, `value` 为 factory 的 Map。

④ **gateway properties**，外部化配置类。

**疑问**：该类依赖 GatewayProperties 对象，后者已经携带了 List 结构的 RouteDefinition，那为什么还要依赖 RouteDefinitionLocator 来提供 RouteDefinition？

1. 这里并不会直接使用到 GatewayProperties 类中的 RouteDefinition，仅是用到其定义的 default filters，这会应用到每一个 Route 上。
2. 最终传入的 RouteDefinitionLocator 实现上是 CompositeRouteDefinitionLocator 的实例，它组合了 GatewayProperties 中所定义的 routes。

自动装配类 GatewayAutoConfiguration 中的定义：

```java
@Bean
@ConditionalOnMissingBean
public PropertiesRouteDefinitionLocator propertiesRouteDefinitionLocator(
    GatewayProperties properties) {
    return new PropertiesRouteDefinitionLocator(properties); // ①
}

@Bean
@Primary // ③
public RouteDefinitionLocator routeDefinitionLocator(List<RouteDefinitionLocator> routeDefinitionLocators) { // ②
    return new CompositeRouteDefinitionLocator(
        Flux.fromIterable(routeDefinitionLocators));
}
```

①  RouteDefinitionLocator 的实现类，RouteDefinition 信息来自 GatewayProperties。

②  声明 bean`routeDefinitionLocator`，使用 CompositeRouteDefinitionLocator 实现，它组合了多个 RouteDefinitionLocator 实例。这给用户（开发者）提供了可扩展的余地，用户可以根据需要扩展自己的 RouteDefinitionLocator，比如 RouteDefinition 可源自数据库。

### 4.9.2. 核心方法

getRoutes 源码：

```java
// 实现 RouteLocator 的 getRoutes() 方法
@Override
public Flux<Route> getRoutes() {
    return this.routeDefinitionLocator.getRouteDefinitions()
        .map(this::convertToRoute) // ①
        .map(route -> {
            if (logger.isDebugEnabled()) {
                logger.debug("RouteDefinition matched: " + route.getId());
            }
            return route;
        });
}
// ① 所调用的方法
private Route convertToRoute(RouteDefinition routeDefinition) {
    AsyncPredicate<ServerWebExchange> predicate = combinePredicates(routeDefinition);// ②
    List<GatewayFilter> gatewayFilters = getFilters(routeDefinition); // ③

    return Route.async(routeDefinition) // ④
        .asyncPredicate(predicate)
        .replaceFilters(gatewayFilters)
        .build();
}
```

① 调用 convertToRoute 方法将 RouteDefinition 转换成 Route。

② 将 PredicateDefinition 转换成 AsyncPredicate。

③ 将 FilterDefinition 转换成 GatewayFilter。

④ 根据 ②  和 ③  两步骤定义的变量生成 Route 对象。

### 4.9.3. PredicateDefinition 转换成 AsyncPredicate

```java
private AsyncPredicate<ServerWebExchange> combinePredicates(
    RouteDefinition  routeDefinition) {
    List<PredicateDefinition> predicates = routeDefinition.getPredicates();
    AsyncPredicate<ServerWebExchange> predicate =
        lookup(routeDefinition, predicates.get(0)); // ①
    for (PredicateDefinition andPredicate : predicates.subList(1, predicates.size())) {
        AsyncPredicate<ServerWebExchange> found = lookup(routeDefinition,
                                                         andPredicate); // ②
        predicate = predicate.and(found); // ③
    }

    return predicate;
}
```

①  调用 `lookup` 方法，将列表中第一个 PredicateDefinition 转换成 AsyncPredicate。

②  循环调用，将列表中每一个 PredicateDefinition 都转换成 AsyncPredicate。

③  应用`and`操作，将所有的 AsyncPredicate 组合成一个 AsyncPredicate 对象。

具体的转换逻辑：

```java
private AsyncPredicate<ServerWebExchange> lookup(
    RouteDefinition route, PredicateDefinition predicate) {
    RoutePredicateFactory<Object> factory = this.predicates.get(predicate.getName());// ①
    if (factory == null) {
        throw new IllegalArgumentException("Unable to find RoutePredicateFactory with name             " + predicate.getName());
    }
    Map<String, String> args = predicate.getArgs();// ②
    if (logger.isDebugEnabled()) {
        logger.debug("RouteDefinition " + route.getId() + " applying "
                     + args + " to " + predicate.getName());
    }

    Map<String, Object> properties = factory.shortcutType().normalize(
        args, factory, this.parser, this.beanFactory);// ③
    Object config = factory.newConfig();// ④
    ConfigurationUtils.bind(config, properties,
                            factory.shortcutFieldPrefix(), predicate.getName(),                                     validator); // ⑤
    if (this.publisher != null) {
        this.publisher.publishEvent(
            new PredicateArgsEvent(this, route.getId(), properties));
    }
    return factory.applyAsync(config); // ⑥
}
```

① 根据 predicate 名称获取对应的 predicate factory。

② 获取 PredicateDefinition 中的 Map 类型参数，`key` 是固定字符串`_genkey_` + 数字拼接而成。

③ 对第 ② 步获得的参数作进一步转换，`key`为 config 类（工厂类中通过范型指定）的属性名称。

④ 调用 factory 的 newConfig 方法创建一个 config 类对象。

⑤ 将第 ③ 步中产生的参数绑定到 config 对象上。

⑥ 将 cofing 作参数代入，调用 factory 的 applyAsync 方法创建 AsyncPredicate 对象。

### 4.9.4. FilterDefinition 转换成 GatewayFilter

```java
private List<GatewayFilter> getFilters(RouteDefinition routeDefinition) {
    List<GatewayFilter> filters = new ArrayList<>();
    if (!this.gatewayProperties.getDefaultFilters().isEmpty()) { // ①
        filters.addAll(loadGatewayFilters("defaultFilters",
                                          this.gatewayProperties.getDefaultFilters()));
    }
    if (!routeDefinition.getFilters().isEmpty()) { // ②
        filters.addAll(loadGatewayFilters(
            routeDefinition.getId(), routeDefinition.getFilters()));
    }
    AnnotationAwareOrderComparator.sort(filters); // ③
    return filters;
}
```

① 处理 GatewayProperties 中定义的默认的 FilterDefinition，转换成 GatewayFilter。

② 将 RouteDefinition 中定义的 FilterDefinition 转换成 GatewayFilter。

③ 对 GatewayFilter 进行排序，排序的详细逻辑请查阅 spring 中的 `Ordered` 接口。

具体的转换逻辑与 predicate 的转换逻辑非常相似，源码就不贴了，简单概括：

根据名称获取对应的 filter factory，生成 config 对象，绑定属性，调用工厂方法产生 GatewayFilter 对象。

至此，外部化配置是如转换成 Route 对象本身的已介绍完毕。

# 5. 小结

本文主要介绍了 spring cloud gateway 核心组件的定义以及如何构建这些组件和相应的构建原理。

主要涉及以下概念：

- Route

  路由信息，包含 destination uri、predicate 和 filter。

- AsyncPredicate

  匹配相应的 Predicate 才能被路由。

- GatewayFilter

  请求转发至下游服务前后的业务逻辑链。

- GatewayProperties

  外部化配置类，配置路由信息。

- RoutePredicateFactory

  Predicate 工厂，用于生产 Predicate。

- GatewayFilterFactory

  GatewayFilter 工厂，用于生产 GatewayFilter。

- RouteDefinitionRouteLocator

  RouteLocator 接口核心实现类，用于将 RouteDefinition 转换成 Route。

# 6. 附录

贴一张根据 GatewayAutoConfiguration 自动装配类整理的类图。

**注**：图中部分概念在文中并未提及，涉及到 gateway 工作流程的原理和 webflux 的内容，需要另开新篇再述。

![scg-class](http://static.iocoder.cn/8c891d90b4428241cb1350121db1d925)