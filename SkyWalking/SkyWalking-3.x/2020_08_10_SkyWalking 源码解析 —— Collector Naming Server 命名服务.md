title: SkyWalking 源码分析 —— Collector Naming Server 命名服务
date: 2020-08-10
tags:
categories: SkyWalking
permalink: SkyWalking/collector-naming-server


-------

摘要: 原创出处 http://www.iocoder.cn/SkyWalking/collector-naming-server/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 SkyWalking 3.2.6 正式版**

- [1. 概述](http://www.iocoder.cn/SkyWalking/collector-naming-server/)
- [2. Collector Naming Server](http://www.iocoder.cn/SkyWalking/collector-naming-server/)
  - [2.1 NamingModule](http://www.iocoder.cn/SkyWalking/collector-naming-server/)
  - [2.2 NamingModuleJettyProvider](http://www.iocoder.cn/SkyWalking/collector-naming-server/)
  - [2.3 NamingHandlerRegisterService](http://www.iocoder.cn/SkyWalking/collector-naming-server/)
  - [2.4 配置文件](http://www.iocoder.cn/SkyWalking/collector-naming-server/)
- [3. CollectorDiscoveryService](http://www.iocoder.cn/SkyWalking/collector-naming-server/)
  - [3.1 CollectorDiscoveryService](http://www.iocoder.cn/SkyWalking/collector-naming-server/)
  - [3.2 配置文件](http://www.iocoder.cn/SkyWalking/collector-naming-server/)
- [666. 彩蛋](http://www.iocoder.cn/SkyWalking/collector-naming-server/)

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

本文主要分享 **Collector Naming Server 命名服务**。主要包含如下部分：

* Collector Naming Server 提供 Http 两个接口，提供 Agent **分别**查询 Collector Agent Jetty Server 、Collector Agent gRPC Server 集群。
* Collector Agent Jetty Server 、Collector Agent gRPC Server 集群内部的注册与发现。

> 友情提示，建议胖友已经读过 [《SkyWalking 源码分析 —— Collector Server Component 服务器组件》](http://www.iocoder.cn/SkyWalking/collector-cluster-module//?self) 、[《SkyWalking 源码分析 —— Collector Server Component 服务器组件》](http://www.iocoder.cn/SkyWalking/collector-server-component/?self)

Collector Agent Server ( 包括 Jetty 和 gRPC )，提供上传调用链路，JVM Metric 等等 API 给 Agent 调用。  
Agent 通过 Collector Naming Server 调用 Collector Agent Server 的 API ，查询 Collector Agent Server **最新**的集群地址。

Naming Server 在 SkyWalking 架构图处于如下位置( **红框** ) ：

> FROM https://github.com/apache/incubating-skywalking  
> ![](http://www.iocoder.cn/images/SkyWalking/2020_08_10/01.jpeg)

下面我们来看看整体的项目结构，如下图所示 ：

![](http://www.iocoder.cn/images/SkyWalking/2020_08_10/02.png)

# 2. Collector Naming Server

Collector Naming Server 通过 `apm-collector-naming` 项目实现，其中：

* `collector-naming-define` 项目：定义了 Naming Server 的接口。
* `collector-naming-jetty-provider` 项目：基于 Jetty Server 的 Naming Server 实现。

## 2.1 NamingModule

`org.skywalking.apm.collector.cluster.ClusterModule` ，实现 [Module](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Module.java) 抽象类，集群管理 Module 。

[`#name()`](https://github.com/YunaiV/skywalking/blob/8eece7df8a9174067793f0714b8b71d09f142312/apm-collector/apm-collector-naming/collector-naming-define/src/main/java/org/skywalking/apm/collector/naming/NamingModule.java#L34) **实现**方法，返回模块名为 `"naming"` 。

[`#services()`](https://github.com/YunaiV/skywalking/blob/8eece7df8a9174067793f0714b8b71d09f142312/apm-collector/apm-collector-naming/collector-naming-define/src/main/java/org/skywalking/apm/collector/naming/NamingModule.java#L38) **实现**方法，返回 Service 类名：NamingHandlerRegisterService 。

## 2.2 NamingModuleJettyProvider

`org.skywalking.apm.collector.naming.jetty.NamingModuleJettyProvider` ，实现 [ModuleProvider](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/ModuleProvider.java) **抽象类**，基于 Jetty 的命名组件服务提供者实现类。

[`#name()`](https://github.com/YunaiV/skywalking/blob/8eece7df8a9174067793f0714b8b71d09f142312/apm-collector/apm-collector-naming/collector-naming-jetty-provider/src/main/java/org/skywalking/apm/collector/naming/jetty/NamingModuleJettyProvider.java#L44) **实现**方法，返回组件服务提供者名为 `"jetty"` 。

[`module()`](https://github.com/YunaiV/skywalking/blob/8eece7df8a9174067793f0714b8b71d09f142312/apm-collector/apm-collector-naming/collector-naming-jetty-provider/src/main/java/org/skywalking/apm/collector/naming/jetty/NamingModuleJettyProvider.java#L48) **实现**方法，返回组件类为 NamingModule 。

[`#requiredModules()`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-zookeeper-provider/src/main/java/org/skywalking/apm/collector/cluster/zookeeper/ClusterModuleZookeeperProvider.java#L94) **实现**方法，返回依赖组件为 ClusterModule 、JettyManagerModule。

-------

[`#prepare(Properties)`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-zookeeper-provider/src/main/java/org/skywalking/apm/collector/cluster/zookeeper/ClusterModuleZookeeperProvider.java#L52) **实现**方法，执行准备阶段逻辑。

* 第 55 行 ：创建 ZookeeperModuleListenerService / NamingJettyHandlerRegisterService 对象，并调用 `#registerServiceImplementation()` **父类**方法，注册到 `services` 。

[`#start()`](https://github.com/YunaiV/skywalking/blob/8eece7df8a9174067793f0714b8b71d09f142312/apm-collector/apm-collector-naming/collector-naming-jetty-provider/src/main/java/org/skywalking/apm/collector/naming/jetty/NamingModuleJettyProvider.java#L58) **实现**方法，执行启动阶段逻辑。

* 第 65 行 ：调用 `JettyManagerService#createIfAbsent(host, port, contextPath)` 方法，创建 Jetty Server ，**此时不会启动 Jetty Server**。在 `JettyManagerProvider#notifyAfterCompleted()` 方法，统一启动所有 Jetty Server，在 [《SkyWalking 源码分析 —— Collector Jetty Server Manager》「3. JettyManagerProvider」](http://www.iocoder.cn/SkyWalking/collector-jetty-server-module/?self) 有详细解析。

[`#notifyAfterCompleted()`](https://github.com/YunaiV/skywalking/blob/8eece7df8a9174067793f0714b8b71d09f142312/apm-collector/apm-collector-naming/collector-naming-jetty-provider/src/main/java/org/skywalking/apm/collector/naming/jetty/NamingModuleJettyProvider.java#L68) **实现**方法，执行启动完成逻辑。目前是个空方法。

## 2.3 NamingHandlerRegisterService

`org.skywalking.apm.collector.naming.service.NamingHandlerRegisterService` ，继承 [Service](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Service.java) 接口，命名处理器注册服务**接口**。

[`#register(ServerHandler)`](https://github.com/YunaiV/skywalking/blob/8eece7df8a9174067793f0714b8b71d09f142312/apm-collector/apm-collector-naming/collector-naming-define/src/main/java/org/skywalking/apm/collector/naming/service/NamingHandlerRegisterService.java#L36) **接口**方法，注册 Server 请求处理器。Collector Agent Server 会调用该方法，将其实现的 用于 Naming 的 ServerHandler 进行注册。如下图所示：![](http://www.iocoder.cn/images/SkyWalking/2020_08_10/03.png)

### 2.3.1 NamingJettyHandlerRegisterService

`org.skywalking.apm.collector.naming.jetty.service.service.NamingJettyHandlerRegisterService` ，基于 Jetty 的命名处理器注册服务**实现类**。

[`#register(moduleName, providerName, registration)`](https://github.com/YunaiV/skywalking/blob/8eece7df8a9174067793f0714b8b71d09f142312/apm-collector/apm-collector-naming/collector-naming-jetty-provider/src/main/java/org/skywalking/apm/collector/naming/jetty/service/NamingJettyHandlerRegisterService.java#L48) **实现**方法，调用 `JettyManagerService#addHandler(path, registration)` 方法，注册 Jetty Server 请求处理器。

### 2.3.2 AgentJettyNamingHandler

`org.skywalking.apm.collector.agent.jetty.handler.naming.AgentJettyNamingHandler` ，实现 [JettyHandler](https://github.com/YunaiV/skywalking/blob/8eece7df8a9174067793f0714b8b71d09f142312/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/jetty/JettyHandler.java) **抽象类**，Collector Agent Jetty Server 实现的命名处理器。

[`#pathSpec()`](https://github.com/YunaiV/skywalking/blob/8eece7df8a9174067793f0714b8b71d09f142312/apm-collector/apm-collector-agent-jetty/collector-agent-jetty-provider/src/main/java/org/skywalking/apm/collector/agent/jetty/handler/naming/AgentJettyNamingHandler.java#L39) **实现**方法，获得请求路径为 `"/agent/jetty"` 。

[`#doGet()`](https://github.com/YunaiV/skywalking/blob/8eece7df8a9174067793f0714b8b71d09f142312/apm-collector/apm-collector-agent-jetty/collector-agent-jetty-provider/src/main/java/org/skywalking/apm/collector/agent/jetty/handler/naming/AgentJettyNamingHandler.java#L43) **实现**方法，调用 `AgentJettyNamingListener#getAddresses()` 方法，获得 Collector Agent Jetty Server 集群地址。

* [`org.skywalking.apm.collector.agent.jetty.handler.naming.AgentJettyNamingListener`](https://github.com/YunaiV/skywalking/blob/8eece7df8a9174067793f0714b8b71d09f142312/apm-collector/apm-collector-agent-jetty/collector-agent-jetty-provider/src/main/java/org/skywalking/apm/collector/agent/jetty/handler/naming/AgentJettyNamingListener.java#L28) 基于 Collector Cluster 组件，**实现了集群地址变化的发现**，在 [《SkyWalking 源码分析 —— Collector Cluster 集群管理》](http://www.iocoder.cn/SkyWalking/collector-cluster-module/?self) 有详细解析。

### 2.3.3 AgentGRPCNamingHandler

`org.skywalking.apm.collector.agent.grpc.handler.naming.AgentGRPCNamingHandler` ，实现 [JettyHandler](https://github.com/YunaiV/skywalking/blob/8eece7df8a9174067793f0714b8b71d09f142312/apm-collector/apm-collector-component/server-component/src/main/java/org/skywalking/apm/collector/server/jetty/JettyHandler.java) **抽象类**，Collector Agent gRPC Server 实现的命名处理器。

[`#pathSpec()`](https://github.com/YunaiV/skywalking/blob/ac6c98c1732d6aa62b9d244369478654411ac203/apm-collector/apm-collector-agent-grpc/collector-agent-grpc-provider/src/main/java/org/skywalking/apm/collector/agent/grpc/handler/naming/AgentGRPCNamingHandler.java#L42) **实现**方法，获得请求路径为 `"/agent/gRPC"` 。

[`#doGet()`](https://github.com/YunaiV/skywalking/blob/ac6c98c1732d6aa62b9d244369478654411ac203/apm-collector/apm-collector-agent-grpc/collector-agent-grpc-provider/src/main/java/org/skywalking/apm/collector/agent/grpc/handler/naming/AgentGRPCNamingHandler.java#L46) **实现**方法，调用 `AgentGRPCNamingListener#getAddresses()` 方法，获得 Collector Agent gRPC Server 集群地址。

* [`org.skywalking.apm.collector.agent.grpc.handler.naming.AgentGRPCNamingListener`](https://github.com/YunaiV/skywalking/blob/ac6c98c1732d6aa62b9d244369478654411ac203/apm-collector/apm-collector-agent-grpc/collector-agent-grpc-provider/src/main/java/org/skywalking/apm/collector/agent/grpc/handler/naming/AgentGRPCNamingListener.java) 基于 Collector Cluster 组件，**实现了集群地址变化的发现**，在 [《SkyWalking 源码分析 —— Collector Cluster 集群管理》](http://www.iocoder.cn/SkyWalking/collector-cluster-module/?self) 有详细解析。

## 2.4 配置文件

配置文件如下 ：

![](http://www.iocoder.cn/images/SkyWalking/2020_08_10/02.png)

* 配置 Naming Server 启动在 10800 端口。
* Naming Server **内嵌**在 Collector Server 。通过启动多个 Collector Server 节点，形成 Naming Server 集群。Agent 配置多个 Naming Server 地址。

# 3. CollectorDiscoveryService

`org.skywalking.apm.agent.core.remote.CollectorDiscoveryService` ， 实现 Agent 的 [BootService](https://github.com/YunaiV/skywalking/blob/ac6c98c1732d6aa62b9d244369478654411ac203/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/boot/BootService.java) **接口**，Collector Agent Server 地址**发现**服务。

[`#boot()`](https://github.com/YunaiV/skywalking/blob/ac6c98c1732d6aa62b9d244369478654411ac203/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/CollectorDiscoveryService.java#L45) **实现**方法，调用 `ScheduledExecutorService#scheduleAtFixedRate(...)` 方法，创建定时任务。该定时任务**无初始化延迟**，每 `Config.Collector.DISCOVERY_CHECK_INTERVAL` ( 默认：60 s ) 执行一次。

* DiscoveryRestServiceClient 实现 `java.lang.Runnable` **接口**，即创建的任务。

## 3.1 CollectorDiscoveryService

`org.skywalking.apm.agent.core.remote.CollectorDiscoveryService` ，实现 `java.lang.Runnable` **接口**，Collector 服务发现客户端，基于 **Rest** 方式通信。

[**构造方法**](https://github.com/YunaiV/skywalking/blob/e30787c3a7ce91ae6314003cd488e71c1a534d7c/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/DiscoveryRestServiceClient.java#L61) ，首先随机选择一个 Collector Naming Server ，用于下面 `#findServerList()` 方法，首次获取 Collector Agent Server 集群地址。

[`#run()`](https://github.com/YunaiV/skywalking/blob/e30787c3a7ce91ae6314003cd488e71c1a534d7c/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/DiscoveryRestServiceClient.java#L76) **实现**方法，调用 `#findServerList()` 方法，获取 Collector Agent Server 集群地址。

[`#findServerList()`](https://github.com/YunaiV/skywalking/blob/e30787c3a7ce91ae6314003cd488e71c1a534d7c/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/DiscoveryRestServiceClient.java#L84) 方法，获取 Collector Agent Server 集群地址。

* 第 85 行 ：创建 `org.apache.http.impl.client.CloseableHttpClient` 对象。目前使用 HttpClient `4.5.3` 版本。
* 第 87 行 ：调用 [`#buildGet()`](https://github.com/YunaiV/skywalking/blob/e30787c3a7ce91ae6314003cd488e71c1a534d7c/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/DiscoveryRestServiceClient.java#L142) 方法，创建 `org.apache.http.client.methods.HttpGet` 对象。目前 Agent 查询的是 Collector Agent gRPC Server 集群地址，因为 gRPC 的性能相比 HTTP 更优秀。
* 第 89 行 ：向 Collector Naming Server 发起请求。
* 第 90 至 93 行 ：当响应状态码非 `200` 时，调用 `#findBackupServer()` 方法，**顺序**选择 Collector Naming Server 列表的下一个。**注意**，此时不会再发起请求，需要等下一次执行。
* 第 95 至 111 行 ：处理响应结果，若 Collector Agent gRPC Server 集群地址发生变化，进行更新到 `RemoteDownstreamConfig.Collector.GRPC_SERVERS` 。
* 第 114 至 117 行 ：请求发生异常，调用 `#findBackupServer()` 方法，**顺序**选择 Collector Naming Server 列表的下一个。
* 第 119 行 ：调用 `CloseableHttpClient#close()` 方法，进行关闭。

## 3.2 配置文件

配置文件如下 ：

![](http://www.iocoder.cn/images/SkyWalking/2020_08_10/05.png)

* 生产环境使用时，**推荐** Agent 配置多个 Naming Server 地址。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

2017.12.15 归途 从北京回上海。

突然有种感觉，获得像包方便面，快捷而不营养。

有点"丧"。

胖友，分享一波朋友圈可好！
