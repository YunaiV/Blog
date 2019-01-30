title: SkyWalking 源码分析 —— Collector Server Component 服务器组件
date: 2020-07-28
tags:
categories: SkyWalking
permalink: SkyWalking/collector-server-component

-------

摘要: 原创出处 http://www.iocoder.cn/SkyWalking/collector-server-component/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 SkyWalking 3.2.6 正式版**

- [1. 概述](http://www.iocoder.cn/SkyWalking/collector-server-component/)
- [2. 接口](http://www.iocoder.cn/SkyWalking/collector-server-component/)
  - [2.1 Server](http://www.iocoder.cn/SkyWalking/collector-server-component/)
  - [2.2 ServerHandler](http://www.iocoder.cn/SkyWalking/collector-server-component/)
- [3. gRPC 实现](http://www.iocoder.cn/SkyWalking/collector-server-component/)
  - [3.1 GRPCServer](http://www.iocoder.cn/SkyWalking/collector-server-component/)
  - [3.2 GRPCHandler](http://www.iocoder.cn/SkyWalking/collector-server-component/)
- [4. Jetty 实现](http://www.iocoder.cn/SkyWalking/collector-server-component/)
  - [3.1 JettyServer](http://www.iocoder.cn/SkyWalking/collector-server-component/)
  - [3.2 JettyHandler](http://www.iocoder.cn/SkyWalking/collector-server-component/)
- [666. 彩蛋](http://www.iocoder.cn/SkyWalking/collector-server-component/)

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

本文主要分享 **SkyWalking Collector Server Component 服务器组件**。Collector 通过服务器，提供 API 接口给调用方，例如 Agent 、WebUI 。

Server Component 在 SkyWalking 架构图处于如下位置( **红框** ) ：

> FROM https://github.com/apache/incubating-skywalking  
> ![](http://www.iocoder.cn/images/SkyWalking/2020_07_25/01.jpeg)

下面我们来看看整体的项目结构，如下图所示 ：

![](http://www.iocoder.cn/images/SkyWalking/2020_07_28/02.png)

OK，我们从**接口到实现**的顺序进行分享。

# 2. 接口

## 2.1 Server

`org.skywalking.apm.collector.server.Server` ，服务器**接口**。其实现子类，如下类图 ：![](http://www.iocoder.cn/images/SkyWalking/2020_07_28/03.png)

[`#hostPort()`](https://github.com/YunaiV/skywalking/blob/ceee65ca1e03c34a756922034f85c5d95b8f2178/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/Server.java#L31) **接口**方法，获得服务器地址。  
[`#serverClassify()`](https://github.com/YunaiV/skywalking/blob/ceee65ca1e03c34a756922034f85c5d95b8f2178/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/Server.java#L36) **接口**方法，获得服务器分类。

[`#initialize()`](https://github.com/YunaiV/skywalking/blob/ceee65ca1e03c34a756922034f85c5d95b8f2178/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/Server.java#L43) **接口**方法，初始化服务器。
[`#start()`](https://github.com/YunaiV/skywalking/blob/ceee65ca1e03c34a756922034f85c5d95b8f2178/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/Server.java#L50) **接口**方法，启动服务器。

[`#addHandler()`](https://github.com/YunaiV/skywalking/blob/ceee65ca1e03c34a756922034f85c5d95b8f2178/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/Server.java#L57) **接口**方法，添加请求处理器( ServerHandler )

## 2.2 ServerHandler

[`org.skywalking.apm.collector.server.ServerHandler`](https://github.com/YunaiV/skywalking/blob/ceee65ca1e03c34a756922034f85c5d95b8f2178/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/ServerHandler.java) ，服务器处理器**接口**。其实现子类，如下类图 ：![](http://www.iocoder.cn/images/SkyWalking/2020_07_28/04.png)

ServerHandler 无任何接口方法。

一个 ServerHandler 对应一个请求的处理。

# 3. gRPC 实现

## 3.1 GRPCServer

`org.skywalking.apm.collector.server.grpc.GRPCServer` ，基于 gRPC 的服务器实现。

[`#hostPort()`](https://github.com/YunaiV/skywalking/blob/ceee65ca1e03c34a756922034f85c5d95b8f2178/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/grpc/GRPCServer.java#L47) **实现**方法，获得服务器地址。  
[`#serverClassify()`](https://github.com/YunaiV/skywalking/blob/ceee65ca1e03c34a756922034f85c5d95b8f2178/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/grpc/GRPCServer.java#L51) **实现**方法，获得服务器分类为 `"Google-RPC"`。

[`#initialize()`](https://github.com/YunaiV/skywalking/blob/ceee65ca1e03c34a756922034f85c5d95b8f2178/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/grpc/GRPCServer.java#L55) **实现**方法，调用 `io.grpc.netty.NettyServerBuilder#forAddress(address)` 方法，NettyServerBuilder 。此处，服务器并未创建与启动。  
[`#start()`](https://github.com/YunaiV/skywalking/blob/ceee65ca1e03c34a756922034f85c5d95b8f2178/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/grpc/GRPCServer.java#L61) **实现**方法，创建 `io.grpc.Server` 对象，并启动服务器。

[`#addHandler(handler)`](https://github.com/YunaiV/skywalking/blob/ceee65ca1e03c34a756922034f85c5d95b8f2178/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/grpc/GRPCServer.java#L70) **实现**方法，调用 `NettyServerBuilder#addService(...)` 方法，添加 gRPC 请求处理器( GRPCHandler )。

目前，GRPCServer 使用在 [`collector-agent-grpc-provider`](https://github.com/YunaiV/skywalking/blob/ceee65ca1e03c34a756922034f85c5d95b8f2178/apm-collector/apm-collector-agent-grpc/collector-agent-grpc-provider/) / [`collector-remote-grpc-provider`](https://github.com/YunaiV/skywalking/tree/ceee65ca1e03c34a756922034f85c5d95b8f2178/apm-collector/apm-collector-remote/collector-remote-grpc-provider) 项目。

## 3.2 GRPCHandler

[`org.skywalking.apm.collector.server.grpc.GRPCHandler`](https://github.com/YunaiV/skywalking/blob/ceee65ca1e03c34a756922034f85c5d95b8f2178/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/ServerHandler.java) ，gRPC 请求处理器**接口**。其实现子类，如下类图 ：![](http://www.iocoder.cn/images/SkyWalking/2020_07_28/05.png)

GRPCHandler 无任何接口方法。

# 4. Jetty 实现

## 3.1 JettyServer

`org.skywalking.apm.collector.server.jetty.JettyServer` ，基于 Jetty 的服务器实现。

[`#hostPort()`](https://github.com/YunaiV/skywalking/blob/3c964d8b5678cf6f715dc252e6fe48ba87d0f9e9/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/jetty/JettyServer.java#L52) **实现**方法，获得服务器地址。  
[`#serverClassify()`](https://github.com/YunaiV/skywalking/blob/3c964d8b5678cf6f715dc252e6fe48ba87d0f9e9/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/jetty/JettyServer.java#L56) **实现**方法，获得服务器分类为 `"Jetty"`。

[`#initialize()`](https://github.com/YunaiV/skywalking/blob/3c964d8b5678cf6f715dc252e6fe48ba87d0f9e9/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/jetty/JettyServer.java#L60) **实现**方法，创建 `org.eclipse.jetty.server.Server` 和 `org.eclipse.jetty.servle.ServletContextHandler` 对象。此处，服务器并未启动。  
[`#start()`](https://github.com/YunaiV/skywalking/blob/3c964d8b5678cf6f715dc252e6fe48ba87d0f9e9/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/jetty/JettyServer.java#L80) **实现**方法，启动服务器。

[`#addHandler(handler)`](https://github.com/YunaiV/skywalking/blob/3c964d8b5678cf6f715dc252e6fe48ba87d0f9e9/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/jetty/JettyServer.java#L72) **实现**方法，使用 ServerHandler 创建 `org.eclipse.jetty.servlet.ServletHolder` 对象，并调用 `ServletContextHandler#addServlet(servlet, pathSpec)` 方法进行添加。

目前，JettyServer 使用在 [`collector-agent-jetty-provider`](https://github.com/YunaiV/skywalking/tree/3c964d8b5678cf6f715dc252e6fe48ba87d0f9e9/apm-collector/apm-collector-agent-jetty/collector-agent-jetty-provider) / [`collector-ui-jetty-provider`](https://github.com/YunaiV/skywalking/tree/3c964d8b5678cf6f715dc252e6fe48ba87d0f9e9/apm-collector/apm-collector-ui/collector-ui-jetty-provider) 项目。

## 3.2 JettyHandler

`org.skywalking.apm.collector.server.jetty.JettyHandler` ，继承 `javax.servlet.http.HttpServlet` **抽象类**，Jetty 请求处理。

[`#pathSpec()`](https://github.com/YunaiV/skywalking/blob/3c964d8b5678cf6f715dc252e6fe48ba87d0f9e9/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/jetty/JettyHandler.java#L43) **抽象**方法，请求路径定义。

[`#doGet(HttpServletRequest)`](https://github.com/YunaiV/skywalking/blob/3c964d8b5678cf6f715dc252e6fe48ba87d0f9e9/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/jetty/JettyHandler.java#L61) **抽象**方法，处理 Get 请求，并返回 `com.google.gson.JsonElement` 对象。

* 该抽象方法会被 [`#doGet(HttpServletRequest, HttpServletResponse)`](https://github.com/YunaiV/skywalking/blob/3c964d8b5678cf6f715dc252e6fe48ba87d0f9e9/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/jetty/JettyHandler.java#L46) 方法调用。
    * 成功时，调用 [`#reply(HttpServletResponse, JsonElement)`](https://github.com/YunaiV/skywalking/blob/3c964d8b5678cf6f715dc252e6fe48ba87d0f9e9/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/jetty/JettyHandler.java#L174) 方法，返回 JSON 。
    * 错误时，调用 [`#replyError(HttpServletResponse, errorMessage, status)`](https://github.com/YunaiV/skywalking/blob/3c964d8b5678cf6f715dc252e6fe48ba87d0f9e9/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/jetty/JettyHandler.java#L195) 方法，返回 JSON 。

[`#doPost(HttpServletRequest)`](https://github.com/YunaiV/skywalking/blob/3c964d8b5678cf6f715dc252e6fe48ba87d0f9e9/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/jetty/JettyHandler.java#L79) **抽象**方法，处理 Post 请求，并返回 `com.google.gson.JsonElement` 对象。

* 该抽象方法会被 [`#doPost(HttpServletRequest, HttpServletResponse)`](https://github.com/YunaiV/skywalking/blob/3c964d8b5678cf6f715dc252e6fe48ba87d0f9e9/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/jetty/JettyHandler.java#L64) 方法调用。

**HttpServlet 所有方法被重写，并标记 `final` 修饰符，不允许子类重写**。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

又双叒叕成功更新了一篇水文。😜

胖友，分享个朋友圈可好？


