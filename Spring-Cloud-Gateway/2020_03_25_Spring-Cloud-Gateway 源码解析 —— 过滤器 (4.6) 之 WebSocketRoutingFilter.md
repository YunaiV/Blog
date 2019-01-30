title: Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.6) 之 WebSocketRoutingFilter
date: 2020-03-25
tags:
categories: Spring-Cloud-Gateway
permalink: Spring-Cloud-Gateway/filter-websocket-routing

-------

摘要: 原创出处 http://www.iocoder.cn/Spring-Cloud-Gateway/filter-websocket-routing/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Spring-Cloud-Gateway 2.0.X M4**  

- [1. 概述](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-websocket-routing/)
- [2. 环境搭建](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-websocket-routing/)
- [3. WebsocketRoutingFilter](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-websocket-routing/)
  - [3.1 ProxyWebSocketHandler](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-websocket-routing/)
- [666. 彩蛋](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-websocket-routing/)

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

本文主要分享 **WebsocketRoutingFilter 的代码实现**。

WebsocketRoutingFilter ，Websocket **路由**网关过滤器。其根据 `ws://` / `wss://` 前缀( Scheme )过滤处理，**代理后端 Websocket 服务**，提供给客户端连接。如下图 ：

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_03_25/01.png)

* 目前**一个** RouteDefinition 只能指定**一个**后端 WebSocket 服务。官方正在计划在 LoadBalancerClientFilter 上实现 Websocket 的负载均衡功能。也就说，未来**一个** RouteDefinition 能够指定**多个**后端 WebSocket 服务。

Websocket 的 RouteDefinition 配置如下 ：

```YAML
cloud:
    gateway:
      routes:
      - id: websocket_test
        uri: ws://localhost:9000
        order: 8000
        predicates:
        - Path=/echo
```

* `uri` 使用 `ws://` 或者 `wss://` 为前缀。

-------

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



# 2. 环境搭建

在解析源码之前，我们先以 [wscat](https://github.com/websockets/wscat)  搭建一个 WebSocket 服务。

第一步，安装 wscat 。

```
npm install -g wscat
```

第二步，启动 wscat 。

```
wscat --listen 9000
```

第三步，连接 wscat 。

```
 wscat --listen 9000
```

第四步，配置 RouteDefinition ，并启动 Spring Cloud Gateway 。

```
cloud:
    gateway:
      routes:
      - id: websocket_test
        uri: ws://localhost:9000
        order: 8000
        predicates:
        - Path=/echo
```

第五步，通过 Gateway 连接 wscat 。

```
wscat --connect ws://localhost:8080/echo
```

大功告成。

注意，wscat 同一时间仅允许一个客户端连接。

# 3. WebsocketRoutingFilter

`org.springframework.cloud.gateway.filter.WebsocketRoutingFilter` ，Websocket **路由**网关过滤器。

**构造方法**，代码如下 ：

```Java
public class WebsocketRoutingFilter implements GlobalFilter, Ordered {
	public static final String SEC_WEBSOCKET_PROTOCOL = "Sec-WebSocket-Protocol";

	private final WebSocketClient webSocketClient;
	private final WebSocketService webSocketService;

	public WebsocketRoutingFilter(WebSocketClient webSocketClient) {
		this(webSocketClient, new HandshakeWebSocketService());
	}

	public WebsocketRoutingFilter(WebSocketClient webSocketClient,
			WebSocketService webSocketService) {
		this.webSocketClient = webSocketClient;
		this.webSocketService = webSocketService;
	}
	
}
```

* `webSocketClient` 属性，在 [《Spring-Cloud-Gateway 源码解析 —— 网关初始化》「5.2 初始化 NettyConfiguration」](http://www.iocoder.cn/Spring-Cloud-Gateway/init/?self) 一文中，我们可以看到使用的是 `org.springframework.web.reactive.socket.client.ReactorNettyWebSocketClient` 实现类。通过该属性，**连接后端【被代理】的 WebSocket 服务**。
* `webSocketService` 属性，在 [《Spring-Cloud-Gateway 源码解析 —— 网关初始化》「5.3 初始化 GlobalFilter」](http://www.iocoder.cn/Spring-Cloud-Gateway/init/?self) 一文中，我们可以看到使用的是 `org.springframework.web.reactive.socket.server.support.HandshakeWebSocketService` 实现类。通过该属性，处理客户端发起的连接请求( Handshake Request ) 。

-------

`#getOrder()` 方法，代码如下 ：

```Java
@Override
public int getOrder() {
    return Ordered.LOWEST_PRECEDENCE;
}
```
* 返回顺序为 `Integer.MAX_VALUE` 。在 [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.1) 之 GatewayFilter 一览》「3. GlobalFilter」](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-intro/?self) ，我们列举了所有 GlobalFilter 的顺序。

-------

`#filter(ServerWebExchange, GatewayFilterChain)` 方法，代码如下 ：

```Java
  1: @Override
  2: public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
  3: 	// 获得 requestUrl
  4: 	URI requestUrl = exchange.getRequiredAttribute(GATEWAY_REQUEST_URL_ATTR);
  5: 
  6: 	// 判断是否能够处理
  7: 	String scheme = requestUrl.getScheme();
  8: 	if (isAlreadyRouted(exchange) || (!scheme.equals("ws") && !scheme.equals("wss"))) {
  9: 		return chain.filter(exchange);
 10: 	}
 11: 
 12: 	// 设置已经路由
 13: 	setAlreadyRouted(exchange);
 14: 
 15: 	// 处理连接请求
 16: 	return this.webSocketService.handleRequest(exchange,
 17: 			new ProxyWebSocketHandler(requestUrl, this.webSocketClient, exchange.getRequest().getHeaders()));
 18: }
```

* 第 4 行 ：获得 `requestUrl` 。
* 第 7 至 10 行 ：判断 ForwardRoutingFilter 是否能够处理该请求，需要满足两个条件 ：
    * `ws://` 或者 `wss://` 前缀( Scheme ) 。
    * 调用 `ServerWebExchangeUtils#isAlreadyRouted(ServerWebExchange)` 方法，判断该请求暂未被其他 Routing 网关处理。代码如下 ：

        ```Java
        public static boolean isAlreadyRouted(ServerWebExchange exchange) {
            return exchange.getAttributeOrDefault(GATEWAY_ALREADY_ROUTED_ATTR, false);
        }
        ```
        * x
* 第 13 行 ：设置该请求已经被处理。代码如下 ：

    ```Java
    public static void setAlreadyRouted(ServerWebExchange exchange) {
        exchange.getAttributes().put(GATEWAY_ALREADY_ROUTED_ATTR, true);
    }
    ```

* 第 15 至 16 行 ：调用 `WebSocketService#hanldeRequest(ServerWebExchange, WebSocketHandler)` 方法，处理客户端发起的连接请求( Handshake Request ) 。这个方法的实现不在本文范围内，但是良心如笔者，大概讲下涉及到的类 ：
    * 主要逻辑在 [`org.springframework.web.reactive.socket.server.upgrade.ReactorNettyRequestUpgradeStrategy`](https://github.com/spring-projects/spring-framework/blob/master/spring-webflux/src/main/java/org/springframework/web/reactive/socket/server/upgrade/ReactorNettyRequestUpgradeStrategy.java#L50) 类里。
    * 【第一步】 ReactorNettyRequestUpgradeStrategy 调用 [`reactor.ipc.netty.http.server.HttpServerWSOperations`](https://github.com/reactor/reactor-netty/blob/master/src/main/java/reactor/ipc/netty/http/server/HttpServerWSOperations.java) ，处理客户端发起的连接请求。处理成功，告知客户端连接成功。
    * 【第二步】ReactorNettyRequestUpgradeStrategy 调用 [`org.springframework.web.reactive.socket.server.upgrade.ReactorNettyRequestUpgradeStrategy`](https://github.com/spring-projects/spring-framework/blob/8f69b5ff23d6835eee89a26c0e1e3e63a64a21a0/spring-webflux/src/main/java/org/springframework/web/reactive/socket/WebSocketHandler.java) **接口**的 `#handle(WebSocketSession)` 方法，处理客户端 WebSocket Session 。ProxyWebSocketHandler 是 WebSocketHandler 的**实现类**，在 [「3.1 ProxyWebSocketHandler」](#) 来详细解析 `#handle(WebSocketSession)` 实现了什么逻辑。

## 3.1 ProxyWebSocketHandler

`org.springframework.cloud.gateway.filter.WebsocketRoutingFilter.ProxyWebSocketHandler` ，**代理**后端 WebSocket 服务处理器。

**构造方法**，代码如下 ：

```Java
  1: private static class ProxyWebSocketHandler implements WebSocketHandler {
  2: 
  3: 	private final WebSocketClient client;
  4: 	private final URI url;
  5: 	private final HttpHeaders headers;
  6: 	private final List<String> subProtocols;
  7: 
  8: 	public ProxyWebSocketHandler(URI url, WebSocketClient client, HttpHeaders headers) {
  9: 		this.client = client;
 10: 		this.url = url;
 11: 		this.headers = new HttpHeaders();//headers;
 12: 		//TODO: better strategy to filter these headers?
 13: 		headers.entrySet().forEach(header -> {
 14: 			if (!header.getKey().toLowerCase().startsWith("sec-websocket")
 15: 					&& !header.getKey().equalsIgnoreCase("upgrade")
 16: 					&& !header.getKey().equalsIgnoreCase("connection")) {
 17: 				this.headers.addAll(header.getKey(), header.getValue());
 18: 			}
 19: 		});
 20: 		List<String> protocols = headers.get(SEC_WEBSOCKET_PROTOCOL);
 21: 		if (protocols != null) {
 22: 			this.subProtocols = protocols;
 23: 		} else {
 24: 			this.subProtocols = Collections.emptyList();
 25: 		}
 26: 	}
 27: }
```

* `client` 属性，在 [《Spring-Cloud-Gateway 源码解析 —— 网关初始化》「5.2 初始化 NettyConfiguration」](http://www.iocoder.cn/Spring-Cloud-Gateway/init/?self) 一文中，我们可以看到使用的是 `org.springframework.web.reactive.socket.client.ReactorNettyWebSocketClient` 实现类。通过该属性，**连接后端【被代理】的 WebSocket 服务**。
* `url` 属性，后端【被代理】的 WebSocket 服务的地址。
* `header` 属性，请求头，在 [《 【计网】HTTP与WebSocket的区别》](http://blog.csdn.net/baiye_xing/article/details/73938360) 有详细解析，包括为什么【第 14 至 18 行】的代码这样处理。
* `subProtocols` 属性，最终通信使用的协议。

-------

`#handle(WebSocketSession)` 方法，代码如下 ：

```Java
  1: @Override
  2: public Mono<Void> handle(WebSocketSession session) {
  3: 	// pass headers along so custom headers can be sent through
  4: 	return client.execute(url, this.headers, new WebSocketHandler() {
  5: 		@Override
  6: 		public Mono<Void> handle(WebSocketSession proxySession) {
  7: 			// Use retain() for Reactor Netty
  8: 			// 转发消息 客户端 =》后端服务
  9: 			Mono<Void> proxySessionSend = proxySession
 10: 					.send(session.receive().doOnNext(WebSocketMessage::retain));
 11: 			// 转发消息 后端服务=》客户端
 12: 			// .log("proxySessionSend", Level.FINE);
 13: 			Mono<Void> serverSessionSend = session
 14: 					.send(proxySession.receive().doOnNext(WebSocketMessage::retain));
 15: 					// .log("sessionSend", Level.FINE);
 16: 
 17: 			// 
 18: 			return Mono.when(proxySessionSend, serverSessionSend).then();
 19: 		}
 20: 
 21: 		/**
 22: 		 * Copy subProtocols so they are available downstream.
 23: 		 * @return
 24: 		 */
 25: 		@Override
 26: 		public List<String> getSubProtocols() {
 27: 			return ProxyWebSocketHandler.this.subProtocols;
 28: 		}
 29: 	});
 30: }
```
* 第 6 行 ：调用 `WebSocketClient#execute(URI, HttpHeaders, WebSocketHandler)` 方法，**连接后端【被代理】的 WebSocket 服务**。连接成功后，回调 WebSocketHandler 实现的内部类的 `#handle(WebSocketSession)` 方法。
* WebSocketHandler 实现的内部类 
    * 第 9 至 10 行 ：转发消息，客户端 `=>` 后端服务。
    * 第 13 至 14 行 ：转发消息，后端服务 `=>` 客户端。
    * 第 18 行 ：调用 `Mono#when()` 方法，合并 `proxySessionSend` / `serverSessionSend` 两个 Mono 。调用 `Mono#then()` 方法，**参数为空**，合并的 Mono 不发射数据出来。RxJava 和 Reactor 类似，可以参考 [《ReactiveX文档中文翻译 —— And/Then/When》](https://mcxiaoke.gitbooks.io/rxdocs/content/operators/And.html) 学习下 `when / and / then` 操作符。
    * 下图可以帮助理解下这个类的用途 ：![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_03_25/01.png)

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

😈 限于对 Reactor 和 Netty 了解不够深入，写的不够透彻。回头深入理解下它们。

胖友，分享一波朋友圈可好！


