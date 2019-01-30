title: Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.10) 之 RequestRateLimiterGatewayFilterFactory 请求限流
date: 2020-04-10
tags:
categories: Spring-Cloud-Gateway
permalink: Spring-Cloud-Gateway/filter-request-rate-limiter

-------

摘要: 原创出处 http://www.iocoder.cn/Spring-Cloud-Gateway/filter-request-rate-limiter/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Spring-Cloud-Gateway 2.0.X M4**  

- [1. 概述](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-hystrix/)
- [2. 环境搭建](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-hystrix/)
- [3. RequestRateLimiterGatewayFilterFactory](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-hystrix/)
- [4. KeyResolver](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-hystrix/)
  - [4.1 PrincipalNameKeyResolver](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-hystrix/)
  - [4.2 自定义 KeyResolver](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-hystrix/)
- [5. RateLimiter](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-hystrix/)
  - [5.1 GatewayRedisAutoConfiguration](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-hystrix/)
  - [5.2 RedisRateLimiter](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-hystrix/)
  - [5.3 Redis Lua 脚本](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-hystrix/)
- [666. 彩蛋](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-hystrix/)

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

本文主要分享 **RequestRateLimiterGatewayFilterFactory 的代码实现**。

在 [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.2) 之 GatewayFilterFactory 过滤器工厂》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-factory/?self) 一文中，我们看到 Spring Cloud Gateway 提供了多种 GatewayFilterFactory 的实现，而 RequestRateLimiterGatewayFilterFactory 也是其中的一种。

通过 RequestRateLimiterGatewayFilterFactory ，可以创建 RequestRateLimiterGatewayFilter ( 实际是内部匿名类，为了表述方便，下面继续这么称呼 ) 。

RequestRateLimiterGatewayFilter 使用 **Redis + Lua** 实现分布式限流。而限流的粒度，例如 URL / 用户 / IP 等，通过 `org.springframework.cloud.gateway.filter.ratelimit.KeyResolver` **实现类**决定，在 [「4. KeyResolver」](#) 详细解析。

这里，笔者一本正经的推荐下自己分享的 [《Eureka 源码解析 —— 基于令牌桶算法的 RateLimiter》](http://www.iocoder.cn/Eureka/rate-limiter/?self) ，简直业界良心。

-------

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



# 2. 环境搭建

第一步，以 `spring-cloud-gateway-sample` 项目为基础，在 `pom.xml` 文件添加依赖库。

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

第二步，在 `application.yml` 配置**一个** RouteDefinition 。

```YAML
spring:
  cloud:
    gateway:
      routes:
      # =====================================
      - id: default_path_to_httpbin
        uri: http://127.0.0.1:8081
        order: 10000
        predicates:
        - Path=/**
        filters:
        - RequestRateLimiter=10, 20, #{@principalNameKeyResolver}
```

* `- RequestRateLimiter=10, 20, #{@principalNameKeyResolver}` ，配置 RequestRateLimiterGatewayFilterFactory 。 
    * 默认情况下，基于**令牌桶算法**实现限流。 
    * 第一个参数，`burstCapacity` ，令牌桶上限 。
    * 第二个参数，`replenishRate` ，令牌桶填充平均速率，单位：秒。
    * 第三个参数，`keyResolver` ，限流**键**解析器 Bean 对象名字，根据 `#{@beanName}` ，使用 SpEL 表达式，从 Spring 容器中获取 Bean 对象，详细参见 [`RouteDefinitionRouteLocator#getTuple(ArgumentHints, Map<String, String>, SpelExpressionParser, BeanFactory)`](https://github.com/spring-cloud/spring-cloud-gateway/blob/83496b78944269050373bb92bb2181e1b7c070e8/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/route/RouteDefinitionRouteLocator.java#L182) 处的代码。另外，这里有一个 BUG ：在 YAML 里，`#` 代表注释，所以第三个参数无法正确被读取，需要等待官方修复。如果比较着急使用，可以考虑将此处的 `#` 修改成 `\#` ，并修改部分相关代码以解决该 BUG 。

第三步，配置完成，启动 `spring-cloud-gateway-sample` 项目。

> **友情提示**，RequestRateLimiterGatewayFilter 使用了 RedisTemplate ，生产环境请配置。

# 3. RequestRateLimiterGatewayFilterFactory

`org.springframework.cloud.gateway.filter.factory.RequestRateLimiterGatewayFilterFactory` ，请求限流网关过滤器**工厂**类。代码如下 ：

```Java
  1: public class RequestRateLimiterGatewayFilterFactory implements GatewayFilterFactory {
  2: 
  3: 	public static final String KEY_RESOLVER_KEY = "keyResolver";
  4: 
  5: 	private final RateLimiter rateLimiter;
  6: 	private final KeyResolver defaultKeyResolver;
  7: 
  8: 	public RequestRateLimiterGatewayFilterFactory(RateLimiter rateLimiter,
  9: 			KeyResolver defaultKeyResolver) {
 10: 		this.rateLimiter = rateLimiter;
 11: 		this.defaultKeyResolver = defaultKeyResolver;
 12: 	}
 13: 
 14: 	@Override
 15: 	public List<String> argNames() {
 16:         return Arrays.asList(
 17:                 RedisRateLimiter.REPLENISH_RATE_KEY,
 18:                 RedisRateLimiter.BURST_CAPACITY_KEY,
 19:                 KEY_RESOLVER_KEY
 20:         );
 21: 	}
 22: 
 23: 	@Override
 24: 	public boolean validateArgs() {
 25:  		return false;
 26: 	}
 27: 
 28: 	@SuppressWarnings("unchecked")
 29: 	@Override
 30: 	public GatewayFilter apply(Tuple args) {
 31:         validateMin(2, args);
 32: 
 33:         // 获得 KeyResolver
 34: 		KeyResolver keyResolver;
 35: 		if (args.hasFieldName(KEY_RESOLVER_KEY)) {
 36: 			keyResolver = args.getValue(KEY_RESOLVER_KEY, KeyResolver.class);
 37: 		} else {
 38: 			keyResolver = defaultKeyResolver;
 39: 		}
 40: 
 41: 		return (exchange, chain) -> keyResolver.resolve(exchange).flatMap(key ->
 42:             // TODO: if key is empty?
 43:             rateLimiter.isAllowed(key, args).flatMap(response -> {
 44:                 // TODO: set some headers for rate, tokens left
 45: 
 46:                 // 允许访问
 47:                 if (response.isAllowed()) {
 48:                     return chain.filter(exchange);
 49:                 }
 50: 
 51:                 // 被限流，不允许访问
 52:                 exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
 53:                 return exchange.getResponse().setComplete();
 54:             }));
 55: 	}
 56: 
 57: }
```

* `rateLimiter` 属性，限流器。默认情况下，使用 RedisRateLimiter 。
* `defaultKeyResolver` 属性，默认限流**键**解析器。默认情况下，使用 PrincipalNameKeyResolver 。
* `#argNames()` 方法，定义了 Tuple 参数的 Key 为 `replenishRate` / `burstCapacity` / `keyResolver` 。
* `#validateArgs()` 方法，定义在 [`RouteDefinitionRouteLocator#getTuple(ArgumentHints, Map<String, String>, SpelExpressionParser, BeanFactory)`](https://github.com/spring-cloud/spring-cloud-gateway/blob/83496b78944269050373bb92bb2181e1b7c070e8/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/route/RouteDefinitionRouteLocator.java#L182) 无需校验 Tuple 结果。因为 `keyResolver` 非必填项，在 `#apply()` 方法，创建 RequestRateLimiterGatewayFilter 时**校验**。
* `#apply()` 方法，创建 RequestRateLimiterGatewayFilter 对象。
* 第 31 行 ：校验 Tuple 参数至少有两个元素，即 `replenishRate` 和 `burstCapacity` 。而 `keyResolver` 是**选填**，为空时，使用默认值 `defaultKeyResolver` 。
* 第 34 至 39 行 ：获得 `keyResolver` 。通过它，获得请求的限流**键**，例如URL / 用户 / IP 等。
* --------- 第 41 至 54 行 ：**创建 RequestRateLimiterGatewayFilter 对象并返回**。
* 第 41 行 ：调用 `KeyResolver#resolve(ServerWebExchange)` 方法，获得请求的限流**键**。
    * **注意下**，这里未处理限流**键**为空的情况( `TODO: if key is empty?` )。所以，当限流**键**为空时，过滤器链不会继续向下执行，也就是说，不会请求后端 Http / Websocket 服务，并且最终返回客户端 **200** 状态码，内容为**空**。
* 第 43 至 54 行 ：调用 `RateLimiter#isAllowed(ServerWebExchange, Tuple)` 方法，判断是否被限流。
    * 第 47 至 49 行 ：**未**被限流，允许访问，提交过滤器链继续过滤。
    * 第 52 至 53 行 ：被限流， **不**允许访问，设置响应 429 状态码，并回写客户端**响应**( `exchange.getResponse().setComplete()` ) 。

# 4. KeyResolver

`org.springframework.cloud.gateway.filter.ratelimit.KeyResolver` ，请求**键**解析器**接口**。代码如下 ：

```Java
public interface KeyResolver {
	Mono<String> resolve(ServerWebExchange exchange);
}
```

* `KeyResolver#resolve(ServerWebExchange)` 方法，获得请求的限流**键**。

通过实现 KeyResolver 接口，实现获得不同的请求的限流**键**，例如URL / 用户 / IP 等。

目前版本，Spring Cloud Gateway 提供的 KeyResolver 实现类只有 PrincipalNameKeyResolver 。据官方说法，在未来的里程碑版本中，将会有一些 KeyResolver 具体实现类。

## 4.1 PrincipalNameKeyResolver

`org.springframework.cloud.gateway.filter.ratelimit.PrincipalNameKeyResolver` ，使用请求认证的 `java.security.Principal` 作为限流**键**。代码如下 ：

```Java
public class PrincipalNameKeyResolver implements KeyResolver {

	public static final String BEAN_NAME = "principalNameKeyResolver";

	@Override
	public Mono<String> resolve(ServerWebExchange exchange) {
		return exchange.getPrincipal().map(Principal::getName).switchIfEmpty(Mono.empty());
	}
}
```

## 4.2 自定义 KeyResolver

通过实现 KeyResolver 接口，实现自定义 KeyResolver 。下面我们实现一个使用请求 IP 作为限流**键**的 KeyResolver 。

第一步，创建 RemoteAddrKeyResolver 类，代码如下 ：

```Java
public class RemoteAddrKeyResolver implements KeyResolver {

	public static final String BEAN_NAME = "remoteAddrKeyResolver";

	@Override
	public Mono<String> resolve(ServerWebExchange exchange) {
        return Mono.just(exchange.getRequest().getRemoteAddress().getAddress().getHostAddress());
	}
	
}
```

第二步，配置 RemoteAddrKeyResolver Bean 对象，代码如下 ：

```Java
@Bean(name = RemoteAddrKeyResolver.BEAN_NAME)
@ConditionalOnBean(RateLimiter.class)
public RemoteAddrKeyResolver remoteAddrKeyResolver() {
    return new RemoteAddrKeyResolver();
}
```

第三步，配置 RouteDefinition 路由配置，配置如下 ：

```YAML
spring:
  cloud:
    gateway:
      routes:
      # =====================================
      - id: default_path_to_httpbin
        uri: http://127.0.0.1:8081
        order: 10000
        predicates:
        - Path=/**
        filters:
        - RequestRateLimiter=10, 20, #{@remoteAddrKeyResolver}
```

第四步，**大功告成**，启动 Spring Cloud Gateway 即可。

另外，推荐 [《周立 —— Spring Cloud限流详解（附源码）》](https://mp.weixin.qq.com/s?__biz=MzI4ODQ3NjE2OA==&mid=2247483811&idx=1&sn=16fe7e25a90635e93c60048ebe8b40a2&chksm=ec3c9cc4db4b15d2937fb8b2ef571c941b2bb9fefcf5ed8232e3699b4868392022a62b963699&mpshare=1&scene=1&srcid=1201O46Ma9D5ln5TuxHUgziY#rd)，里面有一些限流维度的分析。

# 5. RateLimiter

`org.springframework.cloud.gateway.filter.ratelimit.RateLimiter` ，限流器**接口**。代码如下 ：

```Java
public interface RateLimiter {

	Mono<Response> isAllowed(String id, Tuple args);

}
```

* `#isAllowed(String id, Tuple args)` 方法，判断是否被限流。
* Response 类，代码如下 ：

    ```Java
    class Response {
        /**
         * 是否允许访问( 未被限流 )
         */
    	private final boolean allowed;
        /**
         * 令牌桶剩余数量
         */
    	private final long tokensRemaining;
    
    	public Response(boolean allowed, long tokensRemaining) {
    		this.allowed = allowed;
    		this.tokensRemaining = tokensRemaining;
    	}
    }
    ```

## 5.1 GatewayRedisAutoConfiguration

`org.springframework.cloud.gateway.config.GatewayRedisAutoConfiguration` ，Redis 相关配置类，代码如下 ：

```Java
  1: @Configuration
  2: @AutoConfigureAfter(RedisReactiveAutoConfiguration.class)
  3: @AutoConfigureBefore(GatewayAutoConfiguration.class)
  4: @ConditionalOnBean(ReactiveRedisTemplate.class)
  5: @ConditionalOnClass({RedisTemplate.class, DispatcherHandler.class})
  6: class GatewayRedisAutoConfiguration {
  7: 
  8: 	@Bean
  9: 	@SuppressWarnings("unchecked")
 10: 	public RedisScript redisRequestRateLimiterScript() {
 11: 		DefaultRedisScript redisScript = new DefaultRedisScript<>();
 12: 		redisScript.setScriptSource(new ResourceScriptSource(new ClassPathResource("META-INF/scripts/request_rate_limiter.lua")));
 13: 		redisScript.setResultType(List.class);
 14: 		return redisScript;
 15: 	}
 16: 
 17: 	@Bean
 18: 	//TODO: replace with ReactiveStringRedisTemplate in future
 19: 	public ReactiveRedisTemplate<String, String> stringReactiveRedisTemplate(
 20: 			ReactiveRedisConnectionFactory reactiveRedisConnectionFactory,
 21: 			ResourceLoader resourceLoader) {
 22: 		RedisSerializer<String> serializer = new StringRedisSerializer();
 23: 		RedisSerializationContext<String , String> serializationContext = RedisSerializationContext
 24: 				.<String, String>newSerializationContext()
 25: 				.key(serializer)
 26: 				.value(serializer)
 27: 				.hashKey(serializer)
 28: 				.hashValue(serializer)
 29: 				.build();
 30: 		return new ReactiveRedisTemplate<>(reactiveRedisConnectionFactory,
 31: 				serializationContext);
 32: 	}
 33: 
 34: 	@Bean
 35: 	public RedisRateLimiter redisRateLimiter(ReactiveRedisTemplate<String, String> redisTemplate,
 36: 											 @Qualifier("redisRequestRateLimiterScript") RedisScript<List<Long>> redisScript) {
 37: 		return new RedisRateLimiter(redisTemplate, redisScript);
 38: 	}
 39: 
 40: }
```

* 第 8 至 15 行 ：创建 `org.springframework.data.redis.core.script.RedisScript` Bean 对象，加载 `META-INF/scripts/request_rate_limiter.lua` 路径下的 Redis Lua 脚本。该脚本使用 Redis 基于**令牌桶算法**实现限流。在本文 [「Redis Lua 脚本」](#) 详细解析。 
* 第 17 至 32 行 ：创建 `org.springframework.data.redis.core.ReactiveRedisTemplate` Bean 对象。
* 第 34 至 38 行 ：使用 RedisScript 和 ReactiveRedisTemplate Bean 对象，创建 RedisRateLimiter Bean 对象。

## 5.2 RedisRateLimiter

`org.springframework.cloud.gateway.filter.ratelimit.RedisRateLimiter` ，基于 Redis 的分布式限流器**实现类**。

**构造方法**，代码如下 ：

```Java
public class RedisRateLimiter implements RateLimiter {
	public static final String REPLENISH_RATE_KEY = "replenishRate";
	public static final String BURST_CAPACITY_KEY = "burstCapacity";

	private final ReactiveRedisTemplate<String, String> redisTemplate;
	private final RedisScript<List<Long>> script;

	public RedisRateLimiter(ReactiveRedisTemplate<String, String> redisTemplate,
			RedisScript<List<Long>> script) {
		this.redisTemplate = redisTemplate;
		this.script = script;
	}
}
```

* `redisTemplate` 属性，RedisTemplate 。
* `script` 属性，Lua 脚本。

-------

`#isAllowed(id, Tuple)` ，代码如下 ：

```Java
  1: public Mono<Response> isAllowed(String id, Tuple args) {
  2: 	// How many requests per second do you want a user to be allowed to do?
  3: 	int replenishRate = args.getInt(REPLENISH_RATE_KEY);
  4: 
  5: 	// How much bursting do you want to allow?
  6: 	int burstCapacity;
  7: 	if (args.hasFieldName(BURST_CAPACITY_KEY)) {
  8: 		burstCapacity = args.getInt(BURST_CAPACITY_KEY);
  9: 	} else {
 10: 		burstCapacity = 0;
 11: 	}
 12: 
 13: 	try {
 14: 		// Make a unique key per user.
 15: 		String prefix = "request_rate_limiter." + id;
 16: 
 17: 		// You need two Redis keys for Token Bucket.
 18: 		List<String> keys = Arrays.asList(prefix + ".tokens", prefix + ".timestamp");
 19: 
 20: 		// The arguments to the LUA script. time() returns unixtime in seconds.
 21: 		List<String> scriptArgs = Arrays.asList(replenishRate + "", burstCapacity + "",
 22: 		 		Instant.now().getEpochSecond() + "", "1");
 23: 		// allowed, tokens_left = redis.eval(SCRIPT, keys, args)
 24: 		Flux<List<Long>> flux = this.redisTemplate.execute(this.script, keys, scriptArgs);
 25: 		// .log("redisratelimiter", Level.FINER);
 26: 		return flux
 27: 				// Throwable => Flux.just(Arrays.asList(1L, -1L)) 。
 28: 				.onErrorResume(throwable -> Flux.just(Arrays.asList(1L, -1L)))
 29: 				// Flux<List<Long>> => Mono<List<Long>>
 30: 				.reduce(new ArrayList<Long>(), (longs, l) -> {
 31: 					longs.addAll(l);
 32: 					return longs;
 33: 				})
 34: 				// Mono<List<Long>> => Mono<Response>
 35: 				.map(results -> {
 36: 					boolean allowed = results.get(0) == 1L;
 37: 					Long tokensLeft = results.get(1);
 38: 
 39: 					Response response = new Response(allowed, tokensLeft);
 40: 
 41: 					if (log.isDebugEnabled()) {
 42: 						log.debug("response: " + response);
 43: 					}
 44: 					return response;
 45: 				});
 46: 	}
 47: 	catch (Exception e) {
 48: 		/*
 49: 		 * We don't want a hard dependency on Redis to allow traffic. Make sure to set
 50: 		 * an alert so you know if this is happening too much. Stripe's observed
 51: 		 * failure rate is 0.01%.
 52: 		 */
 53: 		log.error("Error determining if user allowed from redis", e);
 54: 	}
 55: 	return Mono.just(new Response(true, -1));
 56: }
```

* `id` 方法参数，令牌桶编号。一个令牌桶编号对应令牌桶。
    * 在本文场景中为请求限流**键**。
* 第 3 行 ：获得 `burstCapacity` 令牌桶上限。
* 第 5 至 11 行 ：获得 `replenishRate` ，令牌桶填充平均速率，单位：秒。
* 第 15 行 ：获得令牌桶前缀，`request_rate_limiter.${id}` 。
* 第 18 行 ：获得令牌桶键数组 ：
    * `request_rate_limiter.${id}.tokens` ：令牌桶**剩余**令牌数。
    * `request_rate_limiter.${id}.timestamp` ：令牌桶**最后**填充令牌时间，单位：秒。
* 第 21 至 22 行 ：获得 Lua 脚本参数 ：
    * 第一个参数 ：`replenishRate` 。
    * 第二个参数 ：`burstCapacity` 。
    * 第三个参数 ：得到从 `1970-01-01 00:00:00` 开始的秒数。**为什么在 Java 代码里获取，而不使用 Lua 在 Reids 里获取**？

        > FROM [《亿级流量网站架构核心技术》](https://union-click.jd.com/jdc?d=pT3LH8)  
        > 因为 Redis 的限制（ Lua中有写操作不能使用带随机性质的读操作，如TIME ）不能在 Redis Lua中 使用 TIME 获取时间戳，因此只好从应用获取然后传入，在某些极端情况下（机器时钟不准的情况下），限流会存在一些小问题。
        * 涛哥这本书非常不错，推荐购买。
    
    * 第四个参数 ：消耗令牌数量，默认 1 。

* 第 24 行 ：调用 `ReactiveRedisTemplate#execute(RedisScript<T>, List<K>, List<?>)` 方法，执行 Redis Lua 脚本，获取令牌。返回结果为 `[是否获取令牌成功, 剩余令牌数]` ，其中，`1` 代表获取令牌**成功**，`0` 代表令牌获取**失败**。
* 第 25 行 ：当 Redis Lua 脚本过程中发生**异常**，忽略异常，返回 `Flux.just(Arrays.asList(1L, -1L))` ，即认为**获取令牌成功**。为什么？在 Redis 发生故障时，我们不希望限流器对 Reids 是**强依赖**，并且 Redis 发生故障的概率本身就很低。

    > We don't want a hard dependency on Redis to allow traffic.   
    > Make sure to set an alert so you know if this is happening too much. Stripe's observed failure rate is 0.01%.

* 第 30 至 33 行 ：调用 `Flux#reduce(A, BiFunction<A, ? super T, A>)` 方法，将 `Flux<List<Long>>` 转换成 `Mono<List<Long>>` 。因为 `ReactiveRedisTemplate#execute(RedisScript<T>, List<K>, List<?>)` 方法的执行结果为 Flux ( 多次 )，实际在当前场景里，自行 Redis Lua 脚本只会返回**一次**数组，所以转换成 Mono (一次)。
* 第 35 至 45 行 ：调用 `Mono#map(Function<? super T, ? extends R>)` 方法，将 `Mono<List<Long>>` => `Mono<Response>` 。
* 第 47 至 55 行 ：当【第 15 至 24 行】代码部分执行发生异常时，例如 Redis 挂了，返回 `Flux.just(Arrays.asList(1L, -1L))` ，即认为**获取令牌成功**。

## 5.3 Redis Lua 脚本

`META-INF/scripts/request_rate_limiter.lua` ，Redis Lua 脚本，实现基于**令牌桶算法**实现限流。代码如下 ：

```Lua
  1: local tokens_key = KEYS[1]
  2: local timestamp_key = KEYS[2]
  3: 
  4: local rate = tonumber(ARGV[1])
  5: local capacity = tonumber(ARGV[2])
  6: local now = tonumber(ARGV[3])
  7: local requested = tonumber(ARGV[4])
  8: 
  9: local fill_time = capacity/rate
 10: local ttl = math.floor(fill_time*2)
 11: 
 12: local last_tokens = tonumber(redis.call("get", tokens_key))
 13: if last_tokens == nil then
 14:   last_tokens = capacity
 15: end
 16: 
 17: local last_refreshed = tonumber(redis.call("get", timestamp_key))
 18: if last_refreshed == nil then
 19:   last_refreshed = 0
 20: end
 21: 
 22: local delta = math.max(0, now-last_refreshed)
 23: local filled_tokens = math.min(capacity, last_tokens+(delta*rate))
 24: local allowed = filled_tokens >= requested
 25: local new_tokens = filled_tokens
 26: local allowed_num = 0
 27: if allowed then
 28:   new_tokens = filled_tokens - requested
 29:   allowed_num = 1
 30: end
 31: 
 32: redis.call("setex", tokens_key, ttl, new_tokens)
 33: redis.call("setex", timestamp_key, ttl, now)
 34: 
 35: return { allowed_num, new_tokens }
```
* 第 1 至 2 行 ：KEYS 方法参数 ：
    * 第一个参数 ：`request_rate_limiter.${id}.tokens` ，令牌桶**剩余**令牌数。
    * 第二个参数 ：`request_rate_limiter.${id}.timestamp` ，令牌桶**最后**填充令牌时间，单位：秒。
* 第 4 至 7 行 ：ARGV 方法参数 ：
    * 第一个参数 ：`replenishRate` 。
    * 第二个参数 ：`burstCapacity` 。
    * 第三个参数 ：得到从 `1970-01-01 00:00:00` 开始的秒数。
    * 第四个参数 ：消耗令牌数量，默认 1 。

* 第 9 行 ：计算令牌桶填充**满**令牌需要多久时间，单位：秒。
* 第 10 行 ：计算 `request_rate_limiter.${id}.tokens` / `request_rate_limiter.${id}.timestamp` 的 **ttl** 。`* 2` 保证时间充足。
* 第 12 至 20 行 ：调用 `get` 命令，获得令牌桶**剩余**令牌数( `last_tokens` ) ，令牌桶**最后**填充令牌时间(`last_refreshed`) 。
* 第 22 至 23 行 ：填充令牌，计算**新**的令牌桶**剩余**令牌数( `filled_tokens` )。填充不超过令牌桶令牌**上限**。
* 第 24 至 30 行 ：获取令牌是否成功。
    * 若**成功**，令牌桶**剩余**令牌数(`new_tokens`) **减**消耗令牌数( `requested` )，并设置获取成功( `allowed_num = 1` ) 。
    * 若**失败**，设置获取失败( `allowed_num = 0` ) 。

* 第 32 至 33 行 ：设置令牌桶**剩余**令牌数( `new_tokens` ) ，令牌桶**最后**填充令牌时间(`now`) 。
* 第 35 行 ：返回数组结果，`[是否获取令牌成功, 剩余令牌数]` 。

**Redis Lua 脚本不会有并发问题么**？

> FROM [《亿级流量网站架构核心技术》](https://union-click.jd.com/jdc?d=pT3LH8)   
> 因 Redis 是单线程模型，因此是线程安全的。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

哇哈哈，过滤器全部完成。恩，当然后面需要在考虑一下，例如认证过滤器等等。

胖友，分享一波朋友圈可好！

