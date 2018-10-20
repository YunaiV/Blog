title: SkyWalking 源码分析 —— Collector Remote 远程通信服务
date: 2020-09-10
tags:
categories: SkyWalking
permalink: SkyWalking/collector-remote-module

-------

摘要: 原创出处 http://www.iocoder.cn/SkyWalking/collector-remote-module/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 SkyWalking 3.2.6 正式版**

- [1. 概述](http://www.iocoder.cn/SkyWalking/collector-remote-module/)
- [2. collector-remote-define](http://www.iocoder.cn/SkyWalking/collector-remote-module/)
  - [2.1 RemoteModule](http://www.iocoder.cn/SkyWalking/collector-remote-module/)
  - [2.2 RemoteSenderService](http://www.iocoder.cn/SkyWalking/collector-remote-module/)
  - [2.3 RemoteClientService](http://www.iocoder.cn/SkyWalking/collector-remote-module/)
  - [2.4 RemoteClient](http://www.iocoder.cn/SkyWalking/collector-remote-module/)
  - [2.5 CommonRemoteDataRegisterService](http://www.iocoder.cn/SkyWalking/collector-remote-module/)
  - [2.6 RemoteSerializeService](http://www.iocoder.cn/SkyWalking/collector-remote-module/)
  - [2.7 RemoteSerializeService](http://www.iocoder.cn/SkyWalking/collector-remote-module/)
- [3. collector-remote-grpc-provider](http://www.iocoder.cn/SkyWalking/collector-remote-module/)
  - [3.1 RemoteModuleGRPCProvider](http://www.iocoder.cn/SkyWalking/collector-remote-module/)
  - [3.2 GRPCRemoteSenderService](http://www.iocoder.cn/SkyWalking/collector-remote-module/)
  - [3.3 GRPCRemoteClientService](http://www.iocoder.cn/SkyWalking/collector-remote-module/)
  - [3.4 GRPCRemoteClient](http://www.iocoder.cn/SkyWalking/collector-remote-module/)
  - [3.5 RemoteCommonServiceHandler](http://www.iocoder.cn/SkyWalking/collector-remote-module/)
  - [3.6 GRPCRemoteSerializeService](http://www.iocoder.cn/SkyWalking/collector-remote-module/)
  - [3.7 GRPCRemoteDeserializeService](http://www.iocoder.cn/SkyWalking/collector-remote-module/)
- [4. collector-remote-grpc-provider](http://www.iocoder.cn/SkyWalking/collector-remote-module/)
- [666. 彩蛋](http://www.iocoder.cn/SkyWalking/collector-remote-module/)

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

本文主要分享 **SkyWalking Collector Remote 远程通信服务**。该服务用于 Collector 集群内部通信。

![](http://www.iocoder.cn/images/SkyWalking/2020_09_10/04.png)

目前集群内部通信的目的，跨节点的流式处理。Remote Module **应用**在 SkyWalking 架构图如下位置( **红框** ) ：

> FROM https://github.com/apache/incubating-skywalking  
> ![](http://www.iocoder.cn/images/SkyWalking/2020_09_10/01.jpeg)

下面我们来看看整体的项目结构，如下图所示 ：

![](http://www.iocoder.cn/images/SkyWalking/2020_09_10/02.png)

* `collector-remote-define` ：定义远程通信接口。
* `collector-remote-kafka-provider` ：基于 Kafka 的远程通信实现。*目前暂未完成*。
* `collector-remote-grpc-provider` ：基于 [Google gRPC](https://grpc.io/) 的远程通信实现。**生产环境目前使用**

下面，我们从**接口到实现**的顺序进行分享。

# 2. collector-remote-define

`collector-remote-define` ：定义远程通信接口。项目结构如下 ：

![](http://www.iocoder.cn/images/SkyWalking/2020_09_10/03.png)

整体流程如下图：

![](http://www.iocoder.cn/images/SkyWalking/2020_09_10/05.png)

我们按照整个流程的处理顺序，逐个解析涉及到的类与接口。

## 2.1 RemoteModule

`org.skywalking.apm.collector.remote.RemoteModule` ，实现 [Module](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Module.java) 抽象类，远程通信 Module 。

[`#name()`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/RemoteModule.java#L34) **实现**方法，返回模块名为 `"remote"` 。

[`#services()`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/RemoteModule.java#L38) **实现**方法，返回 Service 类名：RemoteSenderService 、RemoteDataRegisterService 。

## 2.2 RemoteSenderService

`org.skywalking.apm.collector.remote.service.RemoteSenderService` ，继承 [Service](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Service.java) 接口，远程发送服务**接口**，定义了 [`#send(graphId, nodeId, data, selector)`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteSenderService.java#L40) **接口**方法，调用 RemoteClient ，发送数据。

* `graphId` 方法参数，Graph 编号。通过 `graphId` ，可以查找到对应的 Graph 对象。
    * Graph 在 [《SkyWalking 源码分析 —— Collector Streaming Computing 流式处理（一）》「2. apm-collector-core/graph」](http://www.iocoder.cn/SkyWalking/collector-streaming-first/?self) 有详细解析。
* `nodeId` 方法参数，Worker 编号。通过 `workerId` ，可以查找在 Graph 对象中的 Worker 对象，从而 Graph 中的流式处理。
    * Worker 在 [《SkyWalking 源码分析 —— Collector Streaming Computing 流式处理（一）》「3. apm-collector-stream」](http://www.iocoder.cn/SkyWalking/collector-streaming-first/?self) 有详细解析。
* `data` 方法参数，Data 数据对象。例如，流式处理的具体数据对象。
    * Data 在 [《SkyWalking 源码分析 —— Collector Storage 存储组件》「2. apm-collector-core」](http://www.iocoder.cn/SkyWalking/collector-storage-module/) 有详细解析。
* `selector` 方法参数，[`org.skywalking.apm.collector.remote.service.Selector`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/Selector.java) 选择器对象。根据 Selector 对象，使用对应的**负载均衡**策略，选择集群内的 Collector 节点，发送数据。
* [RemoteSenderService.Mode](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteSenderService.java#L45) 返回值，发送模式分成 `Remote` 和 `Local` 两种方式。前者，发送数据到远程的 Collector 节点；后者，发送数据到本地，即本地处理，参见 [`RemoteWorkerRef#in(message)`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/base/RemoteWorkerRef.java#L58) 方法。

## 2.3 RemoteClientService

`org.skywalking.apm.collector.remote.service.RemoteClientService` ，继承 [Service](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Service.java) 接口，远程客户端服务**接口**，定义了 [`#create(host, port, channelSize, bufferSize)`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteClientService.java#L39) **接口**方法，创建 RemoteClient 对象。

## 2.4 RemoteClient

`org.skywalking.apm.collector.remote.service.RemoteClient` ，继承 `java.lang.Comparable` 接口，远程客户端**接口**。定义了如下接口方法：

* [`#push(graphId, nodeId, data, selector)`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteClient.java#L42) **接口**方法，发送数据。
* [`#getAddress()`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteClient.java#L33) **接口**方法，返回客户端连接的远程 Collector 地址。
* [`#equals(address)`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteClient.java#L44) **接口**方法，判断 RemoteClient 是否连接了指定的地址。

## 2.5 CommonRemoteDataRegisterService

在说 CommonRemoteDataRegisterService 之前，首先来说下 CommonRemoteDataRegisterService 的意图。

在上文中，我们可以看到发送给 Collector 是 Data 对象，而 [Data](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/Data.java) 是数据的**抽象类**，在具体反序列化 Data 对象之前，程序是无法得知它是 Data 的哪个实现对象。这个时候，我们可以给 Data 对象的每个实现类，生成一个对应的**数据协议编号**。

* 在发送数据之前，序列化 Data 对象时，增加该 Data 对应的协议编号，一起发送。
* 在接收数据之后，反序列化数据时，根据协议编号，创建 Data 对应的实现类对象。

[`org.skywalking.apm.collector.remote.service.CommonRemoteDataRegisterService`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/CommonRemoteDataRegisterService.java) ，通用远程数据注册服务。

* [`id`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/CommonRemoteDataRegisterService.java#L40) 属性，数据协议自增编号。
* [`dataClassMapping`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/CommonRemoteDataRegisterService.java#L44) 属性，数据类型( Class<? extends Data> )与**数据协议编号**的映射。
* [`dataInstanceCreatorMapping`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/CommonRemoteDataRegisterService.java#L48) 属性，**数据协议编号**与数据对象创建器( RemoteDataInstanceCreator )的映射。

### 2.5.1 RemoteDataRegisterService

`org.skywalking.apm.collector.remote.service.RemoteDataRegisterService` ，继承 [Service](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Service.java) 接口，远程客户端服务**接口**，定义了 [`#register(Class<? extends Data>, RemoteDataInstanceCreator)`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteDataRegisterService.java#L37) **接口**方法，注册数据类型对应的远程数据创建器( [`RemoteDataRegisterService.RemoteDataInstanceCreator`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteDataRegisterService.java#L39) )对象。

CommonRemoteDataRegisterService 实现了 RemoteDataRegisterService 接口，[`#register(Class<? extends Data>, RemoteDataInstanceCreator)`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/CommonRemoteDataRegisterService.java#L56) **实现**方法。

另外，[AgentStreamRemoteDataRegister](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/AgentStreamRemoteDataRegister.java#L42) 会调用 [`RemoteDataRegisterService#register(Class<? extends Data>, RemoteDataInstanceCreator)`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteDataRegisterService.java#L37) 方法，注册每个数据类型的 RemoteDataInstanceCreator 对象。注意，例如 `Application::new` 是 RemoteDataInstanceCreator 的**匿名实现类**。

### 2.5.2 RemoteDataIDGetter

`org.skywalking.apm.collector.remote.service.RemoteDataIDGetter` ，继承 [Service](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Service.java) 接口，远程数据协议编号获取器**接口**，定义了 [`#getRemoteDataId(Class<? extends Data>)`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteDataIDGetter.java#L37) **接口**方法，根据数据类型获取数据协议编号。

CommonRemoteDataRegisterService 实现了 RemoteDataIDGetter 接口，[`#getRemoteDataId(Class<? extends Data>)`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/CommonRemoteDataRegisterService.java#L67) **实现**方法。

### 2.5.3 RemoteDataInstanceCreatorGetter

`org.skywalking.apm.collector.remote.service.RemoteDataInstanceCreatorGetter` ，继承 [Service](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Service.java) 接口，远程数据创建器的获取器**接口**，定义了 [`#getInstanceCreator(remoteDataId`](https://github.com/YunaiV/skywalking/blob/f70019926200139f9fe235ade9aaf7724ab72c8b/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteDataInstanceCreatorGetter.java#L35) **接口**方法，根据数据协议编号获得远程数据创建器( RemoteDataInstanceCreator )。

CommonRemoteDataRegisterService 实现了 RemoteDataInstanceCreatorGetter 接口，[`#getInstanceCreator(remoteDataId)`](https://github.com/YunaiV/skywalking/blob/f70019926200139f9fe235ade9aaf7724ab72c8b/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/CommonRemoteDataRegisterService.java#L75) **实现**方法。

## 2.6 RemoteSerializeService

`org.skywalking.apm.collector.remote.service.RemoteSerializeService` ，远程通信序列化服务**接口**，定义了 [`#serialize(Data)`](https://github.com/YunaiV/skywalking/blob/f70019926200139f9fe235ade9aaf7724ab72c8b/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteSerializeService.java#L36) **接口**方法，序列化数据，生成 Builder 对象。

## 2.7 RemoteSerializeService

`org.skywalking.apm.collector.remote.service.RemoteDeserializeService` ，远程通信序反列化服务**接口**，定义了 [`#deserialize(RemoteData, Data)`](https://github.com/YunaiV/skywalking/blob/f70019926200139f9fe235ade9aaf7724ab72c8b/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteDeserializeService.java#L36) **接口**方法，反序列化传输数据。

# 3. collector-remote-grpc-provider

`collector-remote-grpc-provider` ，基于 [Google gRPC](https://grpc.io/) 的远程通信实现。

项目结构如下 ：![](http://www.iocoder.cn/images/SkyWalking/2020_09_10/06.png)

**默认配置**，在 [`application-default.yml`](https://github.com/YunaiV/skywalking/blob/8b7205313e60e84d50579261992042c8b581492f/apm-collector/apm-collector-core/src/main/resources/application-default.yml#L14) **已经**配置如下：

``` JSON
remote:
  gRPC:
    host: localhost
    port: 11800
```

## 3.1 RemoteModuleGRPCProvider

[`org.skywalking.apm.collector.remote.grpc.RemoteModuleGRPCProvider`](https://github.com/YunaiV/skywalking/blob/0a289e159f472983a0b6f6df6bd62c675e4f0846/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/RemoteModuleGRPCProvider.java) ，实现 [ModuleProvider](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/ModuleProvider.java) **抽象类**，基于 gRPC 的组件服务提供者实现类。

[`#name()`](https://github.com/YunaiV/skywalking/blob/0a289e159f472983a0b6f6df6bd62c675e4f0846/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/RemoteModuleGRPCProvider.java#L38) **实现**方法，返回组件服务提供者名为 `"gRPC"` 。

[`module()`](https://github.com/YunaiV/skywalking/blob/0a289e159f472983a0b6f6df6bd62c675e4f0846/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/RemoteModuleGRPCProvider.java#L42) **实现**方法，返回组件类为 RemoteModule 。

[`#requiredModules()`](https://github.com/YunaiV/skywalking/blob/0a289e159f472983a0b6f6df6bd62c675e4f0846/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/RemoteModuleGRPCProvider.java#L81) **实现**方法，返回依赖组件为 `cluster` 、`gRPC_manager` 。

-------

[`#prepare(Properties)`](https://github.com/YunaiV/skywalking/blob/0a289e159f472983a0b6f6df6bd62c675e4f0846/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/RemoteModuleGRPCProvider.java#L46) **实现**方法，执行准备阶段逻辑。

* 第 53 至 56 行 ：创建 CommonRemoteDataRegisterService 、GRPCRemoteSenderService 对象，并调用 `#registerServiceImplementation()` **父类**方法，注册到 `services` 。

[`#start()`](https://github.com/YunaiV/skywalking/blob/0a289e159f472983a0b6f6df6bd62c675e4f0846/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/RemoteModuleGRPCProvider.java#L59) **实现**方法，执行启动阶段逻辑。

* Server 相关
    * 第 65 行：创建 gRPC Server 对象。
    * 第 67 行：注册 RemoteCommonServiceHandler 对象到 gRPC Server 上，用于接收 gRPC 请求后的处理。
    * [《SkyWalking 源码分析 —— Collector Server Component 服务器组件》「3. gRPC 实现」](http://www.iocoder.cn/SkyWalking/collector-server-component/?self)
    * [《SkyWalking 源码分析 —— Collector gRPC Server Manager》](http://www.iocoder.cn/SkyWalking/collector-grpc-server-module/?self)
* 注册发现相关
    * 第 70 至 71 行：创建 [`org.skywalking.apm.collector.remote.grpc.RemoteModuleGRPCRegistration`](https://github.com/YunaiV/skywalking/blob/0a289e159f472983a0b6f6df6bd62c675e4f0846/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/RemoteModuleGRPCRegistration.java) 对象，将自己注册到集群管理。这样，自己可以被 Collector 集群节点发现，从而被调用。
    * 第 73 至 74 行：注册 GRPCRemoteSenderService 对象到集群管理。这样，自己可以监听到 Collector 集群节点的加入或离开，从而调用。
    * [《SkyWalking 源码分析 —— Collector Cluster 集群管理》](http://www.iocoder.cn/SkyWalking/collector-cluster-module/?self)

[`#notifyAfterCompleted()`](https://github.com/YunaiV/skywalking/blob/0a289e159f472983a0b6f6df6bd62c675e4f0846/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/RemoteModuleGRPCProvider.java#L77) **实现**方法，方法为空。

## 3.2 GRPCRemoteSenderService

`org.skywalking.apm.collector.remote.grpc.service.GRPCRemoteSenderService` ，继承 ClusterModuleListener 抽象类，实现 RemoteSenderService 接口，基于 gPRC 的远程发送服务实现类。

### 3.2.1 注册发现

通过继承 ClusterModuleListener 抽象类，实现了监听 Collector 集群节点的加入或离开。

* [`remoteClients`](https://github.com/YunaiV/skywalking/blob/0a289e159f472983a0b6f6df6bd62c675e4f0846/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/service/GRPCRemoteSenderService.java#L51) 属性，连接 Collector 集群节点的客户端数组。每个 Collector 集群节点，对应一个客户端。
* [`#path()`](https://github.com/YunaiV/skywalking/blob/0a289e159f472983a0b6f6df6bd62c675e4f0846/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/service/GRPCRemoteSenderService.java#L96) **实现**方法，返回监听的目录 `"/" + RemoteModule.NAME + "/" + RemoteModuleGRPCProvider.NAME` 。Collector 集群中，每个节点的 Remote Server 都会注册到该目录下。
* [`#serverJoinNotify(serverAddress)`](https://github.com/YunaiV/skywalking/blob/0a289e159f472983a0b6f6df6bd62c675e4f0846/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/service/GRPCRemoteSenderService.java#L100) **实现**方法，当新的节点加入，**创建**新的客户端连接。
* [`#serverQuitNotify(serverAddress)`](https://github.com/YunaiV/skywalking/blob/0a289e159f472983a0b6f6df6bd62c675e4f0846/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/service/GRPCRemoteSenderService.java#L114) **实现**方法，当老的节点离开，**移除**对应的客户端连接。

## 3.2.2 负载均衡

RemoteModuleGRPCProvider 基于不同的选择器 ( [Selector](https://github.com/YunaiV/skywalking/blob/0a289e159f472983a0b6f6df6bd62c675e4f0846/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/Selector.java#L26) ) ，提供不同的客户端选择( [`org.skywalking.apm.collector.remote.grpc.service.selector.RemoteClientSelector`](https://github.com/YunaiV/skywalking/blob/0a289e159f472983a0b6f6df6bd62c675e4f0846/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/service/selector/RemoteClientSelector.java) )实现 ：![](http://www.iocoder.cn/images/SkyWalking/2020_09_10/07.png)

* [`hashCodeSelector`](https://github.com/YunaiV/skywalking/blob/0a289e159f472983a0b6f6df6bd62c675e4f0846/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/service/GRPCRemoteSenderService.java#L53) 属性，[HashCodeSelector](https://github.com/YunaiV/skywalking/blob/0a289e159f472983a0b6f6df6bd62c675e4f0846/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/service/selector/HashCodeSelector.java) ，基于数据的哈希码。
* [`foreverFirstSelector`](https://github.com/YunaiV/skywalking/blob/0a289e159f472983a0b6f6df6bd62c675e4f0846/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/service/GRPCRemoteSenderService.java#L54) 属性，[ForeverFirstSelector](https://github.com/YunaiV/skywalking/blob/0a289e159f472983a0b6f6df6bd62c675e4f0846/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/service/selector/ForeverFirstSelector.java) ，基于客户端数组的顺序，选择第一个。
* [`rollingSelector`](https://github.com/YunaiV/skywalking/blob/0a289e159f472983a0b6f6df6bd62c675e4f0846/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/service/GRPCRemoteSenderService.java#L55) 属性，[RollingSelector](https://github.com/YunaiV/skywalking/blob/0a289e159f472983a0b6f6df6bd62c675e4f0846/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/service/selector/RollingSelector.java) ，基于客户端数组的顺序，顺序向下选择。
* [`#send(graphId, nodeId, data, selector)`](https://github.com/YunaiV/skywalking/blob/0a289e159f472983a0b6f6df6bd62c675e4f0846/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/service/GRPCRemoteSenderService.java#L59) 方法，代码如下：
    * 第 63 、66 、69 行：根据选择器，调用 `RemoteClientSelector#select(clients, data)` 方法，选择客户端。
    * 第 64 、67 、70 行：调用 [`#sendToRemoteWhenNotSelf(remoteClient, graphId, nodeId, data)`](https://github.com/YunaiV/skywalking/blob/0a289e159f472983a0b6f6df6bd62c675e4f0846/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/service/GRPCRemoteSenderService.java#L75) 方法，发送请求数据。
        * 第 76 至 77 行：当选择的客户端连接的是本地时，不发送数据，交给本地处理，参见 [`RemoteWorkerRef#in(message)`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/base/RemoteWorkerRef.java#L58) 方法。
        * 第 78 至 81 行：当选择的客户端连接的是远程时，调用 `RemoteClient#push(graphId, nodeId, data)` 方法，发送数据。

## 3.3 GRPCRemoteClientService

`org.skywalking.apm.collector.remote.grpc.service.GRPCRemoteClientService` ，实现 RemoteClientService **接口**，基于 gRPC 的远程客户端服务实现类。

[`#create(host, port, channelSize, bufferSize)`](https://github.com/YunaiV/skywalking/blob/0a289e159f472983a0b6f6df6bd62c675e4f0846/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/service/GRPCRemoteClientService.java#L44) **实现**方法，创建 GRPCRemoteClient 对象。

## 3.4 GRPCRemoteClient

> 友情提示：本小节会涉及较多 gRPC 相关的知识，建议不熟悉的胖友自己 Google ，补充下姿势。

`org.skywalking.apm.collector.remote.grpc.service.GRPCRemoteClient` ，实现 RemoteClient **接口**，基于 gRPC 的远程客户端实现类。

* [`client`](https://github.com/YunaiV/skywalking/blob/4cb80651dee25e985f974d691467a0a53d7dfbe9/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/service/GRPCRemoteClient.java#L48) 属性，GRPCClient 对象。相比来说，GRPCRemoteClient 偏业务的封装，内部调用 GRPCClient 对象。
* [`carrier`](https://github.com/YunaiV/skywalking/blob/4cb80651dee25e985f974d691467a0a53d7dfbe9/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/service/GRPCRemoteClient.java#L52) 属性，DataCarrier 对象，本地消息队列。GRPCRemoteClient 在被调用发送数据时，先提交到本地队列，异步消费进行发送到远程 Collector 节点。DataCarrier 在 [《SkyWalking 源码分析 —— DataCarrier 异步处理库》](http://www.iocoder.cn/SkyWalking/data-carrier/?self) 详细解析。
    * 第 63 行：调用 `DataCarrier#consume(IConsumer, num)` 方法，设置消费者为 RemoteMessageConsumer 对象。

-------

[`#push(graphId, nodeId, data)`](https://github.com/YunaiV/skywalking/blob/4cb80651dee25e985f974d691467a0a53d7dfbe9/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/service/GRPCRemoteClient.java#L70) **实现**方法，**异步**发送消息到远程 Collector 。

* 第 73 行：调用 `RemoteDataIDGetter#getRemoteDataId(Class<? extends Data>)` 方法，获得**数据协议编号**。
* 第 76 至 80 行：创建传输数据( RemoteMessage.Builder ) 对象。RemoteMessage 通过 [Protobuf](https://github.com/google/protobuf) 创建定义，如下图所示：![](http://www.iocoder.cn/images/SkyWalking/2020_09_10/08.png)
* 第 83 行：调用 `DataCarrier#produce(data)` 方法，发送数据到本地队列。

[RemoteMessageConsumer](https://github.com/YunaiV/skywalking/blob/4cb80651dee25e985f974d691467a0a53d7dfbe9/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/service/GRPCRemoteClient.java#L93) ，**批量**消费本地队列的数据，逐条发送数据到远程 Collector 节点。

* [`#consume(List<RemoteMessage>)`](https://github.com/YunaiV/skywalking/blob/4cb80651dee25e985f974d691467a0a53d7dfbe9/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/service/GRPCRemoteClient.java#L98) **实现**方法，代码如下：
    * 第 100 行：创建 [StreamObserver](https://github.com/YunaiV/skywalking/blob/4cb80651dee25e985f974d691467a0a53d7dfbe9/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/service/GRPCRemoteClient.java#L118) 对象。StreamObserver 主要是 gPRC 相关的 API 的调用。
    * 第 101 至 103 行：调用 `io.grpc.stub.StreamObserver#onNext(RemoteMessage)` 方法，逐条发送数据。
    * 第 106 行：调用 `io.grpc.stub.StreamObserver#onCompleted()` 方法，全部请求数据发送**完成**。

## 3.5 RemoteCommonServiceHandler

`org.skywalking.apm.collector.remote.grpc.handler.RemoteCommonServiceHandler` ，实现 `org.skywalking.apm.collector.server.grpc.GRPCHandler` 接口，继承 RemoteCommonServiceGrpc.RemoteCommonServiceImplBase **抽象类**，远程通信通用逻辑处理器。

其中，RemoteCommonServiceGrpc.RemoteCommonServiceImplBase 在 `RemoteCommonService.proto` 文件的定义如下图：![](http://www.iocoder.cn/images/SkyWalking/2020_09_10/09.png)

[`#call(StreamObserver<Empty>)`](https://github.com/YunaiV/skywalking/blob/ece7d2e156d4434edcc6ef08a5ed79e2a7b39fa1/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/handler/RemoteCommonServiceHandler.java#L55) **实现**方法，代码如下：

* [`#onNext(RemoteMessage)`](https://github.com/YunaiV/skywalking/blob/ece7d2e156d4434edcc6ef08a5ed79e2a7b39fa1/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/handler/RemoteCommonServiceHandler.java#L57) 方法，处理每一条消息，代码如下：
    * 第 65 行：调用 `RemoteDataInstanceCreatorGetter#getInstanceCreator(remoteDataId)` 方法，获得**数据协议编号**对应的 RemoteDataInstanceCreator 对象。然后，调用 `RemoteDataInstanceCreator#createInstance(id)` 方法，创建**数据协议编号**对应的 Data 实现类对应的对象。
    * 第 70 行：调用 [`GraphManager#findGraph(graphId)`](https://github.com/YunaiV/skywalking/blob/ece7d2e156d4434edcc6ef08a5ed79e2a7b39fa1/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/graph/GraphManager.java#L60) 方法，获得 `graphId` 对应的 Graph 对象。然后，调动 [`GraphNodeFinder#findNext(nodeId)`](https://github.com/YunaiV/skywalking/blob/ece7d2e156d4434edcc6ef08a5ed79e2a7b39fa1/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/graph/GraphNodeFinder.java#L52) 方法，获得 Next 对象。
    * 第 71 行：调用 `Next#execute(Data)` 方法，继续流式处理。

## 3.6 GRPCRemoteSerializeService

[`org.skywalking.apm.collector.remote.grpc.service.GRPCRemoteSerializeService`](https://github.com/YunaiV/skywalking/blob/ece7d2e156d4434edcc6ef08a5ed79e2a7b39fa1/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/service/GRPCRemoteSerializeService.java) ，实现 RemoteSerializeService 接口，基于 gRPC 的远程通信序列化服务实现类。

## 3.7 GRPCRemoteDeserializeService

[`org.skywalking.apm.collector.remote.grpc.service.GRPCRemoteDeserializeService`](https://github.com/YunaiV/skywalking/blob/ece7d2e156d4434edcc6ef08a5ed79e2a7b39fa1/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/service/GRPCRemoteDeserializeService.java) ，实现 GRPCRemoteDeserializeService 接口，基于 gRPC 的远程通信反序列化服务实现类。

# 4. collector-remote-grpc-provider

`collector-remote-kafka-provider` ：基于 Kafka 的远程通信实现。

*目前暂未完成*。

TODO 【4005】collector-remote-grpc-provider

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

写的有丢丢烦躁，不清晰或者错误的地方，胖友望见谅。

欢迎微信我一起交流。

胖友，分享一波朋友圈可好。


