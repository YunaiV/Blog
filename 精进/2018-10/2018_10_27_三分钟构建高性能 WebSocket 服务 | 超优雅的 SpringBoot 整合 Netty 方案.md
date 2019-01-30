title: 三分钟构建高性能 WebSocket 服务 | 超优雅的 SpringBoot 整合 Netty 方案
date: 2018-10-27
tags:
categories: 精进
permalink: Fight/Build-a-high-performance-WebSocket-service-in-three-minutes
author: Yeauty
from_url: https://my.oschina.net/u/3580577/blog/2088114
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247485454&idx=2&sn=4b03c68b6650f264c902c79faa4c2f36&chksm=fa4977bfcd3efea9ee96e2ec0f10a7584393516ad0fac251552d3e615d598e7fc0616723369a&token=982309024&lang=zh_CN#rd

-------

摘要: 原创出处 https://my.oschina.net/u/3580577/blog/2088114 「Yeauty」欢迎转载，保留摘要，谢谢！

- [前言](http://www.iocoder.cn/Fight/Build-a-high-performance-WebSocket-service-in-three-minutes/)
- [netty-websocket-spring-boot-starter](http://www.iocoder.cn/Fight/Build-a-high-performance-WebSocket-service-in-three-minutes/)
- [快速入门](http://www.iocoder.cn/Fight/Build-a-high-performance-WebSocket-service-in-three-minutes/)
- [测试](http://www.iocoder.cn/Fight/Build-a-high-performance-WebSocket-service-in-three-minutes/)
- [总结](http://www.iocoder.cn/Fight/Build-a-high-performance-WebSocket-service-in-three-minutes/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# 前言

> 每当使用SpringBoot进行Weboscket开发时，最容易想到的就是`spring-boot-starter-websocket`（或`spring-websocket`）。它可以让我们使用注解，很简单的进行Websocket开发，让我们更多的关注业务逻辑。它底层使用的是Tomcat，且不说把整个Tomcat放进一个WebSocket服务中是否会太重，但在大数据量高并发的场景下，它的表现并不是非常理想。

> Netty一款高性能的NIO网络编程框架，在推送量激增时，表现依然出色。(关于性能与表现的讨论，网上很多，这里不过多说明。)很多流行开源项目都在使用Netty，如:Dubbo、Storm、Spark、Elasticsearch、Apache Cassandra等，这得益于Netty的并发高、传输快、封装好等特点。

> 但是，要在SpringBoot项目中整合Netty来开发WebSocket不是一件舒服的事，这会让你过多的关注非业务逻辑的实现。那么，是否有一款框架，能使得在SpringBoot项目中使用Netty开发WebSocket变得简单，甚至优雅，并且可以从使用`spring-boot-starter-websocket`开发的项目无缝的迁移过来呢？

# netty-websocket-spring-boot-starter

这是个开源的框架。通过它，我们可以像`spring-boot-starter-websocket`一样使用注解进行开发，只需关注需要的事件(如OnMessage)。并且底层是使用Netty,当需要调参的时候只需要修改配置参数即可，无需过多的关心handler的设置。

# 快速入门

- 创建SpringBoot项目(v2.0.0以上)并添加依赖:

```XML
	<dependency>
		<groupId>org.yeauty</groupId>
		<artifactId>netty-websocket-spring-boot-starter</artifactId>
		<version>0.6.3</version>
	</dependency>
```

- new一个`ServerEndpointExporter`对象，交给Spring容器，表示要开启WebSocket功能：

```JAVA
@Configuration
public class WebSocketConfig {
    @Bean
    public ServerEndpointExporter serverEndpointExporter() {
        return new ServerEndpointExporter();
    }
}
```

- 在端点类上加上`@ServerEndpoint`、`@Component`注解，并在相应的方法上加上`@OnOpen`、`@OnClose`、`@OnError`、`@OnMessage`注解（不想关注某个事件可不添加对应的注解）：

```java
@ServerEndpoint
@Component
public class MyWebSocket {

    @OnOpen
    public void onOpen(Session session, HttpHeaders headers) throws IOException {
        System.out.println("new connection");
    }

    @OnClose
    public void onClose(Session session) throws IOException {
       System.out.println("one connection closed");
    }

    @OnError
    public void onError(Session session, Throwable throwable) {
        throwable.printStackTrace();
    }

    @OnMessage
    public void OnMessage(Session session, String message) {
        System.out.println(message);
        session.sendText("Hello Netty!");
    }
}
```

- 一个高性能的WebSocket服务端就完成了,直接run起来就可以了。

# 测试

- 服务端是写完了，接下来需要测试一下，看看效果
- 首先，新建一个html文件，把页面撸出来

```html
<!DOCTYPE html>
<html lang="en">
<body>
<div id="msg"></div>
<input type="text" id="text">
<input type="submit" value="send" onclick="send()">
</body>
<script>
    var msg = document.getElementById("msg");
    var wsServer = 'ws://127.0.0.1:80';
    var websocket = new WebSocket(wsServer);
    //监听连接打开
    websocket.onopen = function (evt) {
        msg.innerHTML = "The connection is open";
    };

    //监听服务器数据推送
    websocket.onmessage = function (evt) {
        msg.innerHTML += "<br>" + evt.data;
    };

    //监听连接关闭
    websocket.onclose = function (evt) {
        alert("连接关闭");
    };

    function send() {
        var text = document.getElementById("text").value
        websocket.send(text);
    }
</script>
</html>
```

- 页面撸完，直接用Chrome打开上面html文件即可连上你的WebSocket服务。

# 总结

> 这个框架是基于Netty的，所以直接使用Netty优化时的理念即可。如：堆外内存的0拷贝、接收及发送缓冲区的调整、高低写水位的调整等。

> 生产环境的项目在充分调优后，Netty甚至能比Tomcat高效20倍。(当然，这是特定的场景下)

> 框架详细文档:<https://github.com/YeautyYE/netty-websocket-spring-boot-starter>