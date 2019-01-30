title: Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.9) 之 HystrixGatewayFilterFactory 熔断
date: 2020-04-05
tags:
categories: Spring-Cloud-Gateway
permalink: Spring-Cloud-Gateway/filter-hystrix

-------

摘要: 原创出处 http://www.iocoder.cn/Spring-Cloud-Gateway/filter-hystrix/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Spring-Cloud-Gateway 2.0.X M4**  

- [1. 概述](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-hystrix/)
- [2. 环境搭建](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-hystrix/)
- [3. HystrixGatewayFilterFactory](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-hystrix/)
- [4. 注意事项](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-hystrix/)
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

本文主要分享 **HystrixGatewayFilterFactory 的代码实现**。

在 [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.2) 之 GatewayFilterFactory 过滤器工厂》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-factory/?self) 一文中，我们看到 Spring Cloud Gateway 提供了多种 GatewayFilterFactory 的实现，而 HystrixGatewayFilterFactory 也是其中的一种。

通过 HystrixGatewayFilterFactory ，可以创建 HystrixGatewayFilter ( 实际是内部匿名类，为了表述方便，下面继续这么称呼 ) 。

HystrixGatewayFilter 使用 [Hystrix](https://github.com/Netflix/Hystrix) ，实现基于 **Route** 级别的熔断功能。

这里，笔者一本正经的推荐下自己分享的 [《Hystrix 源码解析系列》](http://www.iocoder.cn/categories/Hystrix/?self) ，简直业界良心。

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
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
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
        - Hystrix=myCommandName
```

* `- Hystrix=myCommandName` ，配置 HystrixGatewayFilterFactory ，并以 `myCommandName` 为 **Hystrix Command 名字**。

第三步，配置完成，启动 `spring-cloud-gateway-sample` 项目。

# 3. HystrixGatewayFilterFactory

`org.springframework.cloud.gateway.filter.factory.HystrixGatewayFilterFactory` ，熔断网关过滤器**工厂**。代码如下 ：

```Java
  1: public class HystrixGatewayFilterFactory implements GatewayFilterFactory {
  2: 
  3: 	@Override
  4: 	public List<String> argNames() {
  5: 		return Arrays.asList(NAME_KEY);
  6: 	}
  7: 
  8: 	@Override
  9: 	public GatewayFilter apply(Tuple args) {
 10: 		//TODO: if no name is supplied, generate one from command id (useful for default filter)
 11: 		final String commandName = args.getString(NAME_KEY);
 12: 		final HystrixCommandGroupKey groupKey = HystrixCommandGroupKey.Factory.asKey(getClass().getSimpleName());
 13: 		final HystrixCommandKey commandKey = HystrixCommandKey.Factory.asKey(commandName);
 14: 
 15: 		final HystrixObservableCommand.Setter setter = HystrixObservableCommand.Setter
 16: 				.withGroupKey(groupKey)
 17: 				.andCommandKey(commandKey);
 18: 
 19: 		return (exchange, chain) -> {
 20: 			RouteHystrixCommand command = new RouteHystrixCommand(setter, exchange, chain);
 21: 
 22: 			return Mono.create(s -> {
 23: 			    // 使用 Hystrix Command Observable 订阅
 24: 				Subscription sub = command.toObservable().subscribe(s::success, s::error, s::success);
 25: 				// Mono 取消时，取消 Hystrix Command Observable 的订阅，结束 Hystrix Command 的执行
 26: 				s.onCancel(sub::unsubscribe);
 27: 			}).onErrorResume((Function<Throwable, Mono<Void>>) throwable -> {
 28: 				if (throwable instanceof HystrixRuntimeException) {
 29: 					HystrixRuntimeException e = (HystrixRuntimeException) throwable;
 30: 					if (e.getFailureType() == TIMEOUT) { //TODO: optionally set status
 31: 						setResponseStatus(exchange, HttpStatus.GATEWAY_TIMEOUT);
 32: 						return exchange.getResponse().setComplete();
 33: 					}
 34: 				}
 35: 				return Mono.empty();
 36: 			}).then();
 37: 		};
 38: 	}
 39: }
```

* `#argNames()` 方法，定义了 Tuple 参数的 Key 为 `name` 。
* `#apply()` 方法，创建 HystrixGatewayFilter 对象。
* 第 11 行 ：从 Tuple 参数获得 Hystrix Command 名字，例如上面举例的 RouteDefinition 时，`commandName = myCommandName` 。
* 第 12 行 ：创建 Hystrix Command 分组 Key 为 `HystrixGatewayFilterFactory` 。
* 第 13 行 ：创建 Hystrix Command Key 为 `commandName` 。
* 第 15 至 17 行 ：创建 HystrixObservableCommand.Setter 对象。
* --------- 第 19 至 37 行 ：**创建 HystrixGatewayFilter 对象并返回**。
* 第 20 行 ：创建 RouteHystrixCommand 对象。代码如下 ：

    ```Java
    private class RouteHystrixCommand extends HystrixObservableCommand<Void> {
    	private final ServerWebExchange exchange;
    	private final GatewayFilterChain chain;
    
    	RouteHystrixCommand(Setter setter, ServerWebExchange exchange, GatewayFilterChain chain) {
    		super(setter);
    		this.exchange = exchange;
    		this.chain = chain;
    	}
    
    	@Override
    	protected Observable<Void> construct() {
    		return RxReactiveStreams.toObservable(this.chain.filter(this.exchange));
    	}
    }
    ```

* 第 22 至 26 行 ：调用 `Mono#create(Consumer<MonoSink<T>>)` 方法，创建 Mono 对象。点击 [传送门](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html#create-java.util.function.Consumer-) 查看该方法详细说明。因为 Hystrix 基于 RxJava ，而 GatewayFilter 基于 Reactor ( Mono 是其内部的一个类 )，通过这个方法，实现订阅的适配。**未来，会实现 [HystrixMonoCommand](https://github.com/Netflix/Hystrix/issues/1089#issuecomment-180512000) 替换 HystrixObservableCommand ，从而统一订阅，去除适配代码**。
    * 第 24 行 ：1）调用 `RouteHystrixCommand#toObservable()` 方法，内部会调用 `RouteHystrixCommand#construct()` 方法，获得执行 `this.chain.filter(this.exchange)` 的 Observable 。2）订阅 Observable ：成功或完成时，调用 `Mono#success(Object)` 方法，目前创建的 Mono 上没有相关的订阅；**异常时**，调用 `Mono#error(Object)` 方法，目前创建的 Mono 上调用 `Mongo#onErrorResume(Function<Throwable, Mono<Void>>))` 方法，进行订阅。
    * 第 26 行 ：Mono 取消时，取消 Hystrix Command Observable 的订阅，结束 Hystrix Command 的执行。
* 第 27 至 34 行 ：当 Hystrix Command **执行超时**时，设置响应 504 状态码，并回写客户端**响应**( `exchange.getResponse().setComplete()` ) 。
* 第 35 行 ：**当 Hystrix Command 发生其他异常时，例如断路器打开，返回 `Mono.empty()` ，最终返回客户端 200 状态码，内容为空**。
* 第 36 行 ：调用 `Mono#then()` 方法，**参数为空**，返回空 Mono ，不再向后发射数据。

# 4. 注意事项

1. 目前 Hystrix Command 执行超时时，返回客户端 504 状态码，如果使用 JSON 格式作为数据返回，则需要修改下该 HystrixGatewayFilter 的代码实现。
2. 当 Hystrix 熔断时，最终返回客户端 200 状态码，内容为空，此处建议该 HystrixGatewayFilter 的代码实现。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

嘿嘿嘿，写完熔断，准备限流过滤器走起。鸡冻！

胖友，分享一波朋友圈可好！

[《Hystrix —— Configuration》](https://github.com/Netflix/Hystrix/wiki/Configuration#contents)
