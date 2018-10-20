title: Spring Webflux —— 源码阅读之 socket 包
date: 2018-01-09
tag: 
categories: Spring Webflux
permalink: Spring-Webflux/lanneng/socket
author: 一颗懒能
from_url: https://www.jianshu.com/p/105cfb5dd6fc
wechat_url: 

-------

摘要: 原创出处 https://www.jianshu.com/p/105cfb5dd6fc 「一颗懒能」欢迎转载，保留摘要，谢谢！

- [Package org.springframework.web.reactive.socket](http://www.iocoder.cn/Spring-Webflux/lanneng/socket/)
  - [Package org.springframework.web.reactive.socket.adapter](http://www.iocoder.cn/Spring-Webflux/lanneng/socket/)
  - [AbstractListenerWebSocketSession](http://www.iocoder.cn/Spring-Webflux/lanneng/socket/)
  - [package org.springframework.web.reactive.socket.client;](http://www.iocoder.cn/Spring-Webflux/lanneng/socket/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# Package org.springframework.web.reactive.socket

反应性WebSocket交互的抽象和支持类。

### WebSocketHandler

一个WebSocket会话处理程序。

```Java
public interface WebSocketHandler {

/**
 * 返回此处理程序支持的子协议列表。
 * 默认情况下返回一个空列表。
 */
default List<String> getSubProtocols() {
    return Collections.emptyList();
}

/**
 * 处理WebSocket会话。
 * @param session the session to handle
 * @return completion {@code Mono<Void>} to indicate the outcome of the
 * WebSocket session handling.
 */
Mono<Void> handle(WebSocketSession session);

}
```

### WebSocketSession

表示具有反应流输入和输出的WebSocket会话。

在服务器端，可以通过将请求映射到WebSocketHandler来处理WebSocket会话，并确保在Spring配置中注册了WebSocketHandlerAdapter策略。在客户端，可以将WebSocketHandler提供给WebSocketClient。

```Java
public interface WebSocketSession {

/**
 * Return the id for the session.
 * 返回会话的ID。
 */
String getId();

/**
 *从握手请求中返回信息。
 * Return information from the handshake request.
 */
HandshakeInfo getHandshakeInfo();

/**
 * 返回一个DataBuffer Factory来创建消息有效载荷。
 * Return a {@code DataBuffer} Factory to create message payloads.
 * @return the buffer factory for the session
 */
DataBufferFactory bufferFactory();

/**
 * 获取传入消息的流。
 * Get the flux of incoming messages.
 */
Flux<WebSocketMessage> receive();

/**
 * 将给定的消息写入WebSocket连接。
 * Write the given messages to the WebSocket connection.
 * @param messages the messages to write
 */
Mono<Void> send(Publisher<WebSocketMessage> messages);

/**
 * 用CloseStatus.NORMAL关闭WebSocket会话。
 * Close the WebSocket session with {@link CloseStatus#NORMAL}.
 */
default Mono<Void> close() {
    return close(CloseStatus.NORMAL);
}

/**
 * 关闭具有给定状态的WebSocket会话。
 * Close the WebSocket session with the given status.
 * @param status the close status
 */
Mono<Void> close(CloseStatus status);


// WebSocketMessage factory methods

/**Factory方法使用会话的bufferFactory（）创建文本WebSocketMessag * e。
 */
WebSocketMessage textMessage(String payload);

/**
  * Factory方法使用会话的bufferFactory（）创建二进制WebSocketMes* sage。
 */
WebSocketMessage binaryMessage(Function<DataBufferFactory, DataBuffer> payloadFactory);

/**
 * 工厂方法使用会话的bufferFactory（）创建一个ping  *WebSocketMessage。
 */
WebSocketMessage pingMessage(Function<DataBufferFactory, DataBuffer> payloadFactory);

/**
 * Factory方法使用会话的bufferFactory（）创建pong *WebSocketMessage。
 */
WebSocketMessage pongMessage(Function<DataBufferFactory, DataBuffer> payloadFactory);

}
```

### WebSocketMessage

WebSocket消息的表示。

请参阅WebSocketSession中的静态工厂方法，以便为会话创建带有DataBufferFactory的消息。

```Java
public class WebSocketMessage {

private final Type type;

private final DataBuffer payload;


/**
 * WebSocketMessage的构造函数.
 * 请参阅WebSocketSession中的静态工厂方法，或使用 WebSocketSession.bufferFactory（）创建有效内容，然后调用此构造函       数。
 */
public WebSocketMessage(Type type, DataBuffer payload) {
    Assert.notNull(type, "'type' must not be null");
    Assert.notNull(payload, "'payload' must not be null");
    this.type = type;
    this.payload = payload;
}


/**
 * Return the message type (text, binary, etc).
 */
public Type getType() {
    return this.type;
}

/**
 * Return the message payload.
 */
public DataBuffer getPayload() {
    return this.payload;
}

/**
 * Return the message payload as UTF-8 text. This is a useful for text
 * WebSocket messages.
 */
public String getPayloadAsText() {
    byte[] bytes = new byte[this.payload.readableByteCount()];
    this.payload.read(bytes);
    return new String(bytes, StandardCharsets.UTF_8);
}

/**
 * 保留消息有效载荷的数据缓冲区，这在运行时（例如Netty）和池缓冲区中很有用。一个快捷方式：
 */
public WebSocketMessage retain() {
    DataBufferUtils.retain(this.payload);
    return this;
}

/**
释放在运行时（如Netty）使用池缓冲区（如Netty）有用的有效载荷DataBuffer。一个快捷方式：
 */
public void release() {
    DataBufferUtils.release(this.payload);
}


@Override
public boolean equals(Object other) {
    if (this == other) {
        return true;
    }
    if (!(other instanceof WebSocketMessage)) {
        return false;
    }
    WebSocketMessage otherMessage = (WebSocketMessage) other;
    return (this.type.equals(otherMessage.type) &&
            ObjectUtils.nullSafeEquals(this.payload, otherMessage.payload));
}

@Override
public int hashCode() {
    return this.type.hashCode() * 29 + this.payload.hashCode();
}


/**
 * WebSocket 消息类型.
 */
public enum Type { TEXT, BINARY, PING, PONG }

}
```

### CloseStatus

表示WebSocket“关闭”的状态码和原因。 1xxx范围内的状态码由协议预先定义。

```Java
public final class CloseStatus {

    /**
     * "1000 indicates a normal closure, meaning that the purpose for which the connection
     * was established has been fulfilled."
     */
    public static final CloseStatus NORMAL = new CloseStatus(1000);

    /**
     * "1001 indicates that an endpoint is "going away", such as a server going down or a
     * browser having navigated away from a page."
     */
    public static final CloseStatus GOING_AWAY = new CloseStatus(1001);

    /**
     * "1002 indicates that an endpoint is terminating the connection due to a protocol
     * error."
     */
    public static final CloseStatus PROTOCOL_ERROR  = new CloseStatus(1002);

    /**
     * "1003 indicates that an endpoint is terminating the connection because it has
     * received a type of data it cannot accept (e.g., an endpoint that understands only
     * text data MAY send this if it receives a binary message)."
     */
    public static final CloseStatus NOT_ACCEPTABLE = new CloseStatus(1003);

    // 10004: Reserved.
    // The specific meaning might be defined in the future.

    /**
     * "1005 is a reserved value and MUST NOT be set as a status code in a Close control
     * frame by an endpoint. It is designated for use in applications expecting a status
     * code to indicate that no status code was actually present."
     */
    public static final CloseStatus NO_STATUS_CODE = new CloseStatus(1005);

    /**
     * "1006 is a reserved value and MUST NOT be set as a status code in a Close control
     * frame by an endpoint. It is designated for use in applications expecting a status
     * code to indicate that the connection was closed abnormally, e.g., without sending
     * or receiving a Close control frame."
     */
    public static final CloseStatus NO_CLOSE_FRAME = new CloseStatus(1006);

    /**
     * "1007 indicates that an endpoint is terminating the connection because it has
     * received data within a message that was not consistent with the type of the message
     * (e.g., non-UTF-8 [RFC3629] data within a text message)."
     */
    public static final CloseStatus BAD_DATA = new CloseStatus(1007);

    /**
     * "1008 indicates that an endpoint is terminating the connection because it has
     * received a message that violates its policy. This is a generic status code that can
     * be returned when there is no other more suitable status code (e.g., 1003 or 1009)
     * or if there is a need to hide specific details about the policy."
     */
    public static final CloseStatus POLICY_VIOLATION = new CloseStatus(1008);

    /**
     * "1009 indicates that an endpoint is terminating the connection because it has
     * received a message that is too big for it to process."
     */
    public static final CloseStatus TOO_BIG_TO_PROCESS = new CloseStatus(1009);

    /**
     * "1010 indicates that an endpoint (client) is terminating the connection because it
     * has expected the server to negotiate one or more extension, but the server didn't
     * return them in the response message of the WebSocket handshake. The list of
     * extensions that are needed SHOULD appear in the /reason/ part of the Close frame.
     * Note that this status code is not used by the server, because it can fail the
     * WebSocket handshake instead."
     */
    public static final CloseStatus REQUIRED_EXTENSION = new CloseStatus(1010);

    /**
     * "1011 indicates that a server is terminating the connection because it encountered
     * an unexpected condition that prevented it from fulfilling the request."
     */
    public static final CloseStatus SERVER_ERROR = new CloseStatus(1011);

    /**
     * "1012 indicates that the service is restarted. A client may reconnect, and if it
     * chooses to do, should reconnect using a randomized delay of 5 - 30s."
     */
    public static final CloseStatus SERVICE_RESTARTED = new CloseStatus(1012);

    /**
     * "1013 indicates that the service is experiencing overload. A client should only
     * connect to a different IP (when there are multiple for the target) or reconnect to
     * the same IP upon user action."
     */
    public static final CloseStatus SERVICE_OVERLOAD = new CloseStatus(1013);

    /**
     * "1015 is a reserved value and MUST NOT be set as a status code in a Close control
     * frame by an endpoint. It is designated for use in applications expecting a status
     * code to indicate that the connection was closed due to a failure to perform a TLS
     * handshake (e.g., the server certificate can't be verified)."
     */
    public static final CloseStatus TLS_HANDSHAKE_FAILURE = new CloseStatus(1015);


    private final int code;

    @Nullable
    private final String reason;


    /**
     * Create a new {@link CloseStatus} instance.
     * @param code the status code
     */
    public CloseStatus(int code) {
        this(code, null);
    }

    /**
     * Create a new {@link CloseStatus} instance.
     * @param code the status code
     * @param reason the reason
     */
    public CloseStatus(int code, @Nullable String reason) {
        Assert.isTrue((code >= 1000 && code < 5000), "Invalid status code");
        this.code = code;
        this.reason = reason;
    }


    /**
     * Return the status code.
     */
    public int getCode() {
        return this.code;
    }

    /**
     * Return the reason, or {@code null} if none.
     */
    @Nullable
    public String getReason() {
        return this.reason;
    }

    /**
     * Create a new {@link CloseStatus} from this one with the specified reason.
     * @param reason the reason
     * @return a new {@link CloseStatus} instance
     */
    public CloseStatus withReason(String reason) {
        Assert.hasText(reason, "Reason must not be empty");
        return new CloseStatus(this.code, reason);
    }


    public boolean equalsCode(CloseStatus other) {
        return (this.code == other.code);
    }

    @Override
    public boolean equals(Object other) {
        if (this == other) {
            return true;
        }
        if (!(other instanceof CloseStatus)) {
            return false;
        }
        CloseStatus otherStatus = (CloseStatus) other;
        return (this.code == otherStatus.code &&
                ObjectUtils.nullSafeEquals(this.reason, otherStatus.reason));
    }

    @Override
    public int hashCode() {
        return this.code * 29 + ObjectUtils.nullSafeHashCode(this.reason);
    }

    @Override
    public String toString() {
        return "CloseStatus[code=" + this.code + ", reason=" + this.reason + "]";
    }

}
```

| 状态码 | 常量                  | 表示                                                         |
| ------ | --------------------- | ------------------------------------------------------------ |
| 1000   | NORMAL                | 1000表示一个正常的闭包，这意味着建立连接的目的已经完成       |
| 1001   | GOING_AWAY            | 1001表示端点正在“消失”，比如服务器关闭或浏览器从页面上离开。 |
| 1002   | PROTOCOL_ERROR        | 1002表示由于协议错误，端点正在终止连接                       |
| 1003   | NOT_ACCEPTABLE        | 1003表示端点正在终止连接，因为它接收了一种无法接受的数据类型(例如，一个只理解文本数据的端点，如果接收到二进制消息，则可以发送此数据 |
| 1004   | Reserved              | 保留，未来可能在定义                                         |
| 1005   | NO_STATUS_CODE        | 1005是一个保留值，不能在端点的闭合控制帧中设置为状态码。     |
| 1006   | NO_CLOSE_FRAME        | 1006是一个保留值，不能在端点的闭合控制帧中设置为状态码       |
| 1007   | BAD_DATA              | 1007表示端点正在终止连接，因为它在一条消息中接收到与消息类型不一致的数据(例如，文本消息中的非utf - 8[RFC3629]数据)。 |
| 1008   | POLICY_VIOLATION      | 1008表示端点正在终止连接，因为它收到了违反其策略的消息。     |
| 1009   | TOO_BIG_TO_PROCESS    | 1009表示端点正在终止连接，因为它收到了一个太大的消息，无法处理。 |
| 10010  | REQUIRED_EXTENSION    | 1010表示端点(客户端)终止了连接，因为它期望服务器可以协商一个或多个扩展，但是服务器并没有在WebSocket握手的响应消息中返回它们 |
| 10011  | SERVER_ERROR          | 1011表明服务器正在终止连接，因为它遇到了一个意想不到的情况，阻止它完成请求 |
| 10012  | SERVICE_RESTARTED     | 1012表示该服务重新启动                                       |
| 10013  | SERVICE_OVERLOAD      | 1013显示服务正在经历过载。                                   |
| 10015  | TLS_HANDSHAKE_FAILURE | 1015是一个保留的值，不能在端点的闭环控制帧中设置为状态码     |

### HandshakeInfo

与启动WebSocketSession会话的握手请求相关的简单信息容器

```Java
public class HandshakeInfo {

private final URI uri;

private final Mono<Principal> principalMono;

private final HttpHeaders headers;

@Nullable
private final String protocol;


/**
 * Constructor with information about the handshake.
 * @param uri the endpoint URL
 * @param headers request headers for server or response headers or client
 * @param principal the principal for the session
 * @param protocol the negotiated sub-protocol (may be {@code null})
 */
public HandshakeInfo(URI uri, HttpHeaders headers, Mono<Principal> principal, @Nullable String protocol) {
    Assert.notNull(uri, "URI is required");
    Assert.notNull(headers, "HttpHeaders are required");
    Assert.notNull(principal, "Principal is required");
    this.uri = uri;
    this.headers = headers;
    this.principalMono = principal;
    this.protocol = protocol;
}


/**
* 返回WebSocket端点的URL
 * Return the URL for the WebSocket endpoint.
 */
public URI getUri() {
    return this.uri;
}

/**
 * Return the handshake HTTP headers. Those are the request headers for a
 * server session and the response headers for a client session.
 */
public HttpHeaders getHeaders() {
    return this.headers;
}

/**
 *  返回与握手HTTP请求相关的主体。
 * Return the principal associated with the handshake HTTP request.
 */
public Mono<Principal> getPrincipal() {
    return this.principalMono;
}

/**
* 在握手时协商的子协议，如果没有，则为null。
 * The sub-protocol negotiated at handshake time, or {@code null} if none.
 * @see <a href="https://tools.ietf.org/html/rfc6455#section-1.9">
 * https://tools.ietf.org/html/rfc6455#section-1.9</a>
 */
@Nullable
public String getSubProtocol() {
    return this.protocol;
}


@Override
public String toString() {
    return "HandshakeInfo[uri=" + this.uri + ", headers=" + this.headers + "]";
}

}
```

## Package org.springframework.web.reactive.socket.adapter

将Spring的Reactive WebSocket API与WebSocket运行时相适配的类。

### AbstractWebSocketSession<T>

WebSocketSession实现的便捷基类，包含公共字段并暴露给外界来访问。还实现了WebSocketMessage工厂方法。

### AbstractListenerWebSocketSession

在事件侦听器WebSocket API之间架设的WebSocketSession实现的基类

```Java
public abstract class AbstractWebSocketSession<T> implements WebSocketSession {

private final T delegate;

private final String id;

private final HandshakeInfo handshakeInfo;

private final DataBufferFactory bufferFactory;


/**
 * Create a new instance and associate the given attributes with it.
 */
protected AbstractWebSocketSession(T delegate, String id, HandshakeInfo handshakeInfo,
        DataBufferFactory bufferFactory) {

    Assert.notNull(delegate, "Native session is required.");
    Assert.notNull(id, "Session id is required.");
    Assert.notNull(handshakeInfo, "HandshakeInfo is required.");
    Assert.notNull(bufferFactory, "DataBuffer factory is required.");

    this.delegate = delegate;
    this.id = id;
    this.handshakeInfo = handshakeInfo;
    this.bufferFactory = bufferFactory;
}


protected T getDelegate() {
    return this.delegate;
}

@Override
public String getId() {
    return this.id;
}

@Override
public HandshakeInfo getHandshakeInfo() {
    return this.handshakeInfo;
}

// 返回一个DataBuffer Factory来创建消息有效载荷。
@Override
public DataBufferFactory bufferFactory() {
    return this.bufferFactory;
}

//获取传入消息的流。
@Override
public abstract Flux<WebSocketMessage> receive();

//将给定的消息写入WebSocket连接。
@Override
public abstract Mono<Void> send(Publisher<WebSocketMessage> messages);


// WebSocketMessage factory methods

@Override
public WebSocketMessage textMessage(String payload) {
    byte[] bytes = payload.getBytes(StandardCharsets.UTF_8);
    DataBuffer buffer = bufferFactory().wrap(bytes);
    return new WebSocketMessage(WebSocketMessage.Type.TEXT, buffer);
}


// Factory方法使用WebSocketSession.bufferFactory（）为会话创建二进制WebSocketMessage。
@Override
public WebSocketMessage binaryMessage(Function<DataBufferFactory, DataBuffer> payloadFactory) {
    DataBuffer payload = payloadFactory.apply(bufferFactory());
    return new WebSocketMessage(WebSocketMessage.Type.BINARY, payload);
}

//工厂方法使用WebSocketSession.bufferFactory（）为会话创建一个ping WebSocketMessage。

@Override
public WebSocketMessage pingMessage(Function<DataBufferFactory, DataBuffer> payloadFactory) {
    DataBuffer payload = payloadFactory.apply(bufferFactory());
    return new WebSocketMessage(WebSocketMessage.Type.PING, payload);
}

//工厂方法创建一个使用WebSocketSession.bufferFactory pong WebSocketMessage会话()。
@Override
public WebSocketMessage pongMessage(Function<DataBufferFactory, DataBuffer> payloadFactory) {
    DataBuffer payload = payloadFactory.apply(bufferFactory());
    return new WebSocketMessage(WebSocketMessage.Type.PONG, payload);
}


@Override
public String toString() {
    return getClass().getSimpleName() + "[id=" + getId() + ", uri=" + getHandshakeInfo().getUri() + "]";
}
```

}

## AbstractListenerWebSocketSession

在事件侦听器WebSocket API（例如Java WebSocket API JSR-356，Jetty，Undertow）和Reactive Streams之间进行桥接的WebSocketSession实现的基类。

也是订阅者的实现，因此，它可以用作会话处理的完成订阅

```Java
public abstract class AbstractListenerWebSocketSession<T> extends AbstractWebSocketSession<T>
        implements Subscriber<Void> {

    /**
     * The "back-pressure" buffer size to use if the underlying WebSocket API
     * does not have flow control for receiving messages.
     */
    private static final int RECEIVE_BUFFER_SIZE = 8192;


    @Nullable
    private final MonoProcessor<Void> completionMono;

    private final WebSocketReceivePublisher receivePublisher = new WebSocketReceivePublisher();

    @Nullable
    private volatile WebSocketSendProcessor sendProcessor;

    private final AtomicBoolean sendCalled = new AtomicBoolean();


    /**
     * Base constructor.
     * @param delegate the native WebSocket session, channel, or connection
     * @param id the session id
     * @param handshakeInfo the handshake info
     * @param bufferFactory the DataBuffer factor for the current connection
     */
    public AbstractListenerWebSocketSession(T delegate, String id, HandshakeInfo handshakeInfo,
            DataBufferFactory bufferFactory) {

        this(delegate, id, handshakeInfo, bufferFactory, null);
    }

    /**
     * Alternative constructor with completion {@code Mono&lt;Void&gt;} to propagate
     * the session completion (success or error) (for client-side use).
     */
    public AbstractListenerWebSocketSession(T delegate, String id, HandshakeInfo handshakeInfo,
            DataBufferFactory bufferFactory, @Nullable MonoProcessor<Void> completionMono) {

        super(delegate, id, handshakeInfo, bufferFactory);
        this.completionMono = completionMono;
    }


    protected WebSocketSendProcessor getSendProcessor() {
        WebSocketSendProcessor sendProcessor = this.sendProcessor;
        Assert.state(sendProcessor != null, "No WebSocketSendProcessor available");
        return sendProcessor;
    }

    @Override
    public Flux<WebSocketMessage> receive() {
        return canSuspendReceiving() ?
                Flux.from(this.receivePublisher) :
                Flux.from(this.receivePublisher).onBackpressureBuffer(RECEIVE_BUFFER_SIZE);
    }

    @Override
    public Mono<Void> send(Publisher<WebSocketMessage> messages) {
        if (this.sendCalled.compareAndSet(false, true)) {
            WebSocketSendProcessor sendProcessor = new WebSocketSendProcessor();
            this.sendProcessor = sendProcessor;
            return Mono.from(subscriber -> {
                    messages.subscribe(sendProcessor);
                    sendProcessor.subscribe(subscriber);
            });
        }
        else {
            return Mono.error(new IllegalStateException("send() has already been called"));
        }
    }

    /**
     * 底层的WebSocket API是否具有流量控制功能，可以暂停和恢复接收消息。
     */
    protected abstract boolean canSuspendReceiving();

    /**
     * Suspend receiving until received message(s) are processed and more demand
     * is generated by the downstream Subscriber.
     * <p><strong>Note:</strong> if the underlying WebSocket API does not provide
     * flow control for receiving messages, and this method should be a no-op
     * and {@link #canSuspendReceiving()} should return {@code false}.
     */
    protected abstract void suspendReceiving();

    /**
     * Resume receiving new message(s) after demand is generated by the
     * downstream Subscriber.
     * <p><strong>Note:</strong> if the underlying WebSocket API does not provide
     * flow control for receiving messages, and this method should be a no-op
     * and {@link #canSuspendReceiving()} should return {@code false}.
     */
    protected abstract void resumeReceiving();

    /**
     * Send the given WebSocket message.
     */
    protected abstract boolean sendMessage(WebSocketMessage message) throws IOException;


    // WebSocketHandler adapter delegate methods

    /** Handle a message callback from the WebSocketHandler adapter */
    void handleMessage(Type type, WebSocketMessage message) {
        this.receivePublisher.handleMessage(message);
    }

    /** Handle an error callback from the WebSocketHandler adapter */
    void handleError(Throwable ex) {
        this.receivePublisher.onError(ex);
        WebSocketSendProcessor sendProcessor = this.sendProcessor;
        if (sendProcessor != null) {
            sendProcessor.cancel();
            sendProcessor.onError(ex);
        }
    }

    /** Handle a close callback from the WebSocketHandler adapter */
    void handleClose(CloseStatus reason) {
        this.receivePublisher.onAllDataRead();
        WebSocketSendProcessor sendProcessor = this.sendProcessor;
        if (sendProcessor != null) {
            sendProcessor.cancel();
            sendProcessor.onComplete();
        }
    }


    // Subscriber<Void> implementation

    @Override
    public void onSubscribe(Subscription subscription) {
        subscription.request(Long.MAX_VALUE);
    }

    @Override
    public void onNext(Void aVoid) {
        // no op
    }

    @Override
    public void onError(Throwable ex) {
        if (this.completionMono != null) {
            this.completionMono.onError(ex);
        }
        int code = CloseStatus.SERVER_ERROR.getCode();
        close(new CloseStatus(code, ex.getMessage()));
    }

    @Override
    public void onComplete() {
        if (this.completionMono != null) {
            this.completionMono.onComplete();
        }
        close();
    }


    private final class WebSocketReceivePublisher extends AbstractListenerReadPublisher<WebSocketMessage> {

        @Nullable
        private volatile WebSocketMessage webSocketMessage;

        @Override
        protected void checkOnDataAvailable() {
            if (this.webSocketMessage != null) {
                onDataAvailable();
            }
        }

        @Override
        @Nullable
        protected WebSocketMessage read() throws IOException {
            if (this.webSocketMessage != null) {
                WebSocketMessage result = this.webSocketMessage;
                this.webSocketMessage = null;
                resumeReceiving();
                return result;
            }

            return null;
        }

        void handleMessage(WebSocketMessage webSocketMessage) {
            this.webSocketMessage = webSocketMessage;
            suspendReceiving();
            onDataAvailable();
        }
    }


    protected final class WebSocketSendProcessor extends AbstractListenerWriteProcessor<WebSocketMessage> {

        private volatile boolean isReady = true;

        @Override
        protected boolean write(WebSocketMessage message) throws IOException {
            return sendMessage(message);
        }

        @Override
        protected void releaseData() {
            this.currentData = null;
        }

        @Override
        protected boolean isDataEmpty(WebSocketMessage message) {
            return (message.getPayload().readableByteCount() == 0);
        }

        @Override
        protected boolean isWritePossible() {
            return (this.isReady && this.currentData != null);
        }

        /**
         * Sub-classes can invoke this before sending a message (false) and
         * after receiving the async send callback (true) effective translating
         * async completion callback into simple flow control.
         */
        public void setReadyToSend(boolean ready) {
            this.isReady = ready;
        }
    }

}
```

我们可以看到这两个类是由子类来实现的。不同的服务器有不同的实现。
@Override
public abstract Flux<WebSocketMessage> receive();

```Java
@Override
public abstract Mono<Void> send(Publisher<WebSocketMessage> messages);
```

又分为两个分支，一个是NettyWebSocketSessionSupport下的ReactorNettyWebSocketSession

### NettyWebSocketSessionSupport

基于Netty的WebSocketSession适配器的基类，它提供了将Netty WebSocketFrames转换为WebSocketMessages和从WebSocketMessages转换的便利方法。

```Java
public abstract class NettyWebSocketSessionSupport<T> extends AbstractWebSocketSession<T> {

/**
* 默认的最大大小用于聚集入站WebSocket帧。
 * The default max size for aggregating inbound WebSocket frames.
 */
protected static final int DEFAULT_FRAME_MAX_SIZE = 64 * 1024;


private static final Map<Class<?>, WebSocketMessage.Type> MESSAGE_TYPES;

static {
    MESSAGE_TYPES = new HashMap<>(4);
    MESSAGE_TYPES.put(TextWebSocketFrame.class, WebSocketMessage.Type.TEXT);
    MESSAGE_TYPES.put(BinaryWebSocketFrame.class, WebSocketMessage.Type.BINARY);
    MESSAGE_TYPES.put(PingWebSocketFrame.class, WebSocketMessage.Type.PING);
    MESSAGE_TYPES.put(PongWebSocketFrame.class, WebSocketMessage.Type.PONG);
}


protected NettyWebSocketSessionSupport(T delegate, HandshakeInfo info, NettyDataBufferFactory factory) {
    super(delegate, ObjectUtils.getIdentityHexString(delegate), info, factory);
}


//返回一个DataBuffer Factory来创建消息有效载荷。
@Override
public NettyDataBufferFactory bufferFactory() {
    return (NettyDataBufferFactory) super.bufferFactory();
}


protected WebSocketMessage toMessage(WebSocketFrame frame) {
    DataBuffer payload = bufferFactory().wrap(frame.content());
    return new WebSocketMessage(MESSAGE_TYPES.get(frame.getClass()), payload);
}

protected WebSocketFrame toFrame(WebSocketMessage message) {
    ByteBuf byteBuf = NettyDataBufferFactory.toByteBuf(message.getPayload());
    if (WebSocketMessage.Type.TEXT.equals(message.getType())) {
        return new TextWebSocketFrame(byteBuf);
    }
    else if (WebSocketMessage.Type.BINARY.equals(message.getType())) {
        return new BinaryWebSocketFrame(byteBuf);
    }
    else if (WebSocketMessage.Type.PING.equals(message.getType())) {
        return new PingWebSocketFrame(byteBuf);
    }
    else if (WebSocketMessage.Type.PONG.equals(message.getType())) {
        return new PongWebSocketFrame(byteBuf);
    }
    else {
        throw new IllegalArgumentException("Unexpected message type: " + message.getType());
    }
}

}
```

### ReactorNettyWebSocketSession

Spring WebSocketSession实现，可以适应反应堆Netty的WebSocket NettyInbound和NettyOutbound。

```Java
public class ReactorNettyWebSocketSession
    extends NettyWebSocketSessionSupport<ReactorNettyWebSocketSession.WebSocketConnection> {


public ReactorNettyWebSocketSession(WebsocketInbound inbound, WebsocketOutbound outbound,
        HandshakeInfo info, NettyDataBufferFactory bufferFactory) {

    super(new WebSocketConnection(inbound, outbound), info, bufferFactory);
}


//获取传入消息的流。
@Override
public Flux<WebSocketMessage> receive() {
    return getDelegate().getInbound()
            .aggregateFrames(DEFAULT_FRAME_MAX_SIZE)
            .receiveFrames()
            .map(super::toMessage);
}

//将给定的消息写入WebSocket连接。
@Override
public Mono<Void> send(Publisher<WebSocketMessage> messages) {
    Flux<WebSocketFrame> frames = Flux.from(messages).map(this::toFrame);
    return getDelegate().getOutbound()
            .options(NettyPipeline.SendOptions::flushOnEach)
            .sendObject(frames)
            .then();
}

//关闭具有给定状态的WebSocket会话。
@Override
public Mono<Void> close(CloseStatus status) {
    return Mono.error(new UnsupportedOperationException(
            "Currently in Reactor Netty applications are expected to use the " +
                    "Cancellation returned from subscribing to the \"receive\"-side Flux " +
                    "in order to close the WebSocket session."));
}


/**
 * Simple container for {@link NettyInbound} and {@link NettyOutbound}.
 */
public static class WebSocketConnection {

    private final WebsocketInbound inbound;

    private final WebsocketOutbound outbound;


    public WebSocketConnection(WebsocketInbound inbound, WebsocketOutbound outbound) {
        this.inbound = inbound;
        this.outbound = outbound;
    }

    public WebsocketInbound getInbound() {
        return this.inbound;
    }

    public WebsocketOutbound getOutbound() {
        return this.outbound;
    }
}

}
```

### 另外基于AbstractListenerWebSocketSession的集中不同的实现：

#### StandardWebSocketSession

为标准Java（JSR 356）会话跳转WebSocketSession适配器。

#### StandardWebSocketHandlerAdapter

Java WebSocket API（JSR-356）的适配器，它将事件委托给一个被动的WebSocketHandler及其会话。

另外几个也是差不多，都有不同的实现。

![img](http://upload-images.jianshu.io/upload_images/8565418-8263472178a4996e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

websocket.jpg

## package org.springframework.web.reactive.socket.client;

### WebSocketClient

反应式风格处理WebSocket会话的协议。

```Java
public interface WebSocketClient {

// 具有自定义标头的执行
Mono<Void> execute(URI url, WebSocketHandler handler);

//对给定的URL执行握手请求，并使用给定的handler处理生成的WebSocket会话。
Mono<Void> execute(URI url, HttpHeaders headers, WebSocketHandler handler);

}
```

### WebSocketClientSupport

WebSocketClient实现的基类。

```Java
public class WebSocketClientSupport {

    private static final String SEC_WEBSOCKET_PROTOCOL = "Sec-WebSocket-Protocol";


    protected final Log logger = LogFactory.getLog(getClass());


    protected List<String> beforeHandshake(URI url, HttpHeaders requestHeaders, WebSocketHandler handler) {
        if (logger.isDebugEnabled()) {
            logger.debug("Executing handshake to " + url);
        }
        return handler.getSubProtocols();
    }

    protected HandshakeInfo afterHandshake(URI url, HttpHeaders responseHeaders) {
        if (logger.isDebugEnabled()) {
            logger.debug("Handshake response: " + url + ", " + responseHeaders);
        }
        String protocol = responseHeaders.getFirst(SEC_WEBSOCKET_PROTOCOL);
        return new HandshakeInfo(url, responseHeaders, Mono.empty(), protocol);
    }

}
```

### ReactorNettyWebSocketClient

用于Reactor Netty的WebSocketClient实现。

```Java
public class ReactorNettyWebSocketClient extends WebSocketClientSupport implements WebSocketClient {

private final HttpClient httpClient;


/**
 * Default constructor.
 */
public ReactorNettyWebSocketClient() {
    this(options -> {});
}

/**
 * Constructor that accepts an {@link HttpClientOptions.Builder} consumer
 * to supply to {@link HttpClient#create(Consumer)}.
 */
public ReactorNettyWebSocketClient(Consumer<? super HttpClientOptions.Builder> clientOptions) {
    this.httpClient = HttpClient.create(clientOptions);
}


/**
 * Return the configured {@link HttpClient}.
 */
public HttpClient getHttpClient() {
    return this.httpClient;
}

//对给定的URL执行握手请求，并使用给定的处理程序处理生成的WebSocket会话。
@Override
public Mono<Void> execute(URI url, WebSocketHandler handler) {
    return execute(url, new HttpHeaders(), handler);
}

@Override
public Mono<Void> execute(URI url, HttpHeaders headers, WebSocketHandler handler) {
    List<String> protocols = beforeHandshake(url, headers, handler);

    return getHttpClient()
            .ws(url.toString(),
                    nettyHeaders -> setNettyHeaders(headers, nettyHeaders),
                    StringUtils.collectionToCommaDelimitedString(protocols))
            .flatMap(response -> {
                HandshakeInfo info = afterHandshake(url, toHttpHeaders(response));
                ByteBufAllocator allocator = response.channel().alloc();
                NettyDataBufferFactory factory = new NettyDataBufferFactory(allocator);
                return response.receiveWebsocket((in, out) -> {
                    WebSocketSession session = new ReactorNettyWebSocketSession(in, out, info, factory);
                    return handler.handle(session);
                });
            });
}

private void setNettyHeaders(HttpHeaders headers, io.netty.handler.codec.http.HttpHeaders nettyHeaders) {
    headers.forEach(nettyHeaders::set);
}

private HttpHeaders toHttpHeaders(HttpClientResponse response) {
    HttpHeaders headers = new HttpHeaders();
    response.responseHeaders().forEach(entry -> {
        String name = entry.getKey();
        headers.put(name, response.responseHeaders().getAll(name));
    });
    return headers;
}

}
```

![img](http://upload-images.jianshu.io/upload_images/8565418-8b669855afa18388.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

webscoketClient.jpg

### org.springframework.web.reactive.socket.server.support;

WebSocket请求的服务器端支持类。

#### RequestUpgradeStrategy

根据底层网络运行时将HTTP请求升级到WebSocket会话的策略。

public interface RequestUpgradeStrategy {

```Java
/**
 * 升级到WebSocket会话并使用给定的处理程序处理它。
 * @param exchange the current exchange
 * @param webSocketHandler handler for the WebSocket session
 * @param subProtocol the selected sub-protocol got the handler
 * @return completion {@code Mono<Void>} to indicate the outcome of the
 * WebSocket session handling.
 */
Mono<Void> upgrade(ServerWebExchange exchange, WebSocketHandler webSocketHandler, @Nullable String subProtocol);
```

}

#### ReactorNettyRequestUpgradeStrategy

持有RequestUpgradeStrategy的实现。

public class ReactorNettyRequestUpgradeStrategy implements RequestUpgradeStrategy {

```Java
@Override
public Mono<Void> upgrade(ServerWebExchange exchange, WebSocketHandler handler, @Nullable String subProtocol) {
    ReactorServerHttpResponse response = (ReactorServerHttpResponse) exchange.getResponse();
    HandshakeInfo info = getHandshakeInfo(exchange, subProtocol);
    NettyDataBufferFactory bufferFactory = (NettyDataBufferFactory) response.bufferFactory();

    return response.getReactorResponse().sendWebsocket(subProtocol,
            (in, out) -> handler.handle(new ReactorNettyWebSocketSession(in, out, info, bufferFactory)));
}

private HandshakeInfo getHandshakeInfo(ServerWebExchange exchange, @Nullable String protocol) {
    ServerHttpRequest request = exchange.getRequest();
    Mono<Principal> principal = exchange.getPrincipal();
    return new HandshakeInfo(request.getURI(), request.getHeaders(), principal, protocol);
}
```

}

我们可以看到这里的upgrade方法把接收到的http请求协议转换为websocket协议。

还有其他几种。

![img](http://upload-images.jianshu.io/upload_images/8565418-1930e69d193b0f57.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

image.png

#### WebSocketService

一种委托websocket相关的HTTP请求的服务。

对于WebSocket端点，这意味着要处理初始的WebSocket HTTP握手请求。对于SockJS端点，这可能意味着处理所有在SockJS协议中定义的HTTP请求。

public interface WebSocketService {

```Java
/**
 * 处理HTTP请求并使用给定的WebSocketHandler。
 * @param exchange the current exchange
 * @param webSocketHandler handler for WebSocket session
 * @return a completion Mono for the WebSocket session handling
 */
Mono<Void> handleRequest(ServerWebExchange exchange, WebSocketHandler webSocketHandler);
```

}

#### HandshakeWebSocketService

WebSocketService的实现。

public class HandshakeWebSocketService implements WebSocketService, Lifecycle {

```Java
private static final String SEC_WEBSOCKET_KEY = "Sec-WebSocket-Key";

private static final String SEC_WEBSOCKET_PROTOCOL = "Sec-WebSocket-Protocol";


private static final boolean tomcatPresent = ClassUtils.isPresent(
        "org.apache.tomcat.websocket.server.WsHttpUpgradeHandler",
        HandshakeWebSocketService.class.getClassLoader());

private static final boolean jettyPresent = ClassUtils.isPresent(
        "org.eclipse.jetty.websocket.server.WebSocketServerFactory",
        HandshakeWebSocketService.class.getClassLoader());

private static final boolean undertowPresent = ClassUtils.isPresent(
        "io.undertow.websockets.WebSocketProtocolHandshakeHandler",
        HandshakeWebSocketService.class.getClassLoader());

private static final boolean reactorNettyPresent = ClassUtils.isPresent(
        "reactor.ipc.netty.http.server.HttpServerResponse",
        HandshakeWebSocketService.class.getClassLoader());


protected static final Log logger = LogFactory.getLog(HandshakeWebSocketService.class);


private final RequestUpgradeStrategy upgradeStrategy;

private volatile boolean running = false;


/**
    * 默认构造函数自动，基于类路径检测的RequestUpgradeStrategy的发现使用。
 * Default constructor automatic, classpath detection based discovery of the
 * {@link RequestUpgradeStrategy} to use.
 */
public HandshakeWebSocketService() {
    this(initUpgradeStrategy());
}

/**
 * 使用RequestUpgradeStrategy的替代构造函数。
 * @param upgradeStrategy the strategy to use
 */
public HandshakeWebSocketService(RequestUpgradeStrategy upgradeStrategy) {
    Assert.notNull(upgradeStrategy, "RequestUpgradeStrategy is required");
    this.upgradeStrategy = upgradeStrategy;
}

private static RequestUpgradeStrategy initUpgradeStrategy() {
    String className;
    if (tomcatPresent) {
        className = "TomcatRequestUpgradeStrategy";
    }
    else if (jettyPresent) {
        className = "JettyRequestUpgradeStrategy";
    }
    else if (undertowPresent) {
        className = "UndertowRequestUpgradeStrategy";
    }
    else if (reactorNettyPresent) {
        // As late as possible (Reactor Netty commonly used for WebClient)
        className = "ReactorNettyRequestUpgradeStrategy";
    }
    else {
        throw new IllegalStateException("No suitable default RequestUpgradeStrategy found");
    }

    try {
        className = "org.springframework.web.reactive.socket.server.upgrade." + className;
        Class<?> clazz = ClassUtils.forName(className, HandshakeWebSocketService.class.getClassLoader());
        return (RequestUpgradeStrategy) ReflectionUtils.accessibleConstructor(clazz).newInstance();
    }
    catch (Throwable ex) {
        throw new IllegalStateException(
                "Failed to instantiate RequestUpgradeStrategy: " + className, ex);
    }
}


/**
 * Return the {@link RequestUpgradeStrategy} for WebSocket requests.
 */
public RequestUpgradeStrategy getUpgradeStrategy() {
    return this.upgradeStrategy;
}

@Override
public boolean isRunning() {
    return this.running;
}

@Override
public void start() {
    if (!isRunning()) {
        this.running = true;
        doStart();
    }
}

protected void doStart() {
    if (getUpgradeStrategy() instanceof Lifecycle) {
        ((Lifecycle) getUpgradeStrategy()).start();
    }
}

@Override
public void stop() {
    if (isRunning()) {
        this.running = false;
        doStop();
    }
}

protected void doStop() {
    if (getUpgradeStrategy() instanceof Lifecycle) {
        ((Lifecycle) getUpgradeStrategy()).stop();
    }
}


    //处理HTTP请求并使用给定的WebSocketHandler。
@Override
public Mono<Void> handleRequest(ServerWebExchange exchange, WebSocketHandler handler) {
    ServerHttpRequest request = exchange.getRequest();
    HttpMethod method = request.getMethod();
    HttpHeaders headers = request.getHeaders();

    if (logger.isDebugEnabled()) {
        logger.debug("Handling " + request.getURI() + " with headers: " + headers);
    }

    if (HttpMethod.GET != method) {
        return Mono.error(new MethodNotAllowedException(
                request.getMethodValue(), Collections.singleton(HttpMethod.GET)));
    }

    if (!"WebSocket".equalsIgnoreCase(headers.getUpgrade())) {
        return handleBadRequest("Invalid 'Upgrade' header: " + headers);
    }

    List<String> connectionValue = headers.getConnection();
    if (!connectionValue.contains("Upgrade") && !connectionValue.contains("upgrade")) {
        return handleBadRequest("Invalid 'Connection' header: " + headers);
    }

    String key = headers.getFirst(SEC_WEBSOCKET_KEY);
    if (key == null) {
        return handleBadRequest("Missing \"Sec-WebSocket-Key\" header");
    }

    String protocol = selectProtocol(headers, handler);
    return this.upgradeStrategy.upgrade(exchange, handler, protocol);
}

private Mono<Void> handleBadRequest(String reason) {
    if (logger.isDebugEnabled()) {
        logger.debug(reason);
    }
    return Mono.error(new ServerWebInputException(reason));
}

@Nullable
private String selectProtocol(HttpHeaders headers, WebSocketHandler handler) {
    String protocolHeader = headers.getFirst(SEC_WEBSOCKET_PROTOCOL);
    if (protocolHeader != null) {
        List<String> supportedProtocols = handler.getSubProtocols();
        for (String protocol : StringUtils.commaDelimitedListToStringArray(protocolHeader)) {
            if (supportedProtocols.contains(protocol)) {
                return protocol;
            }
        }
    }
    return null;
}
```

}

WebSocketService实现通过委托给RequestUpgradeStrategy处理WebSocket HTTP握手请求，该请求可以从类路径自动检测（无参数构造函数），但也可以显式配置。

# 666. 彩蛋

如果你对 Spring Webflux 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)