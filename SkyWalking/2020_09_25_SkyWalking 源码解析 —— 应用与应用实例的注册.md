title: SkyWalking 源码分析 —— 应用于应用实例的注册
date: 2020-09-25
tags:
categories: SkyWalking
permalink: SkyWalking/register

-------

摘要: 原创出处 http://www.iocoder.cn/SkyWalking/register/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 SkyWalking 3.2.6 正式版**

- [1. 概述](http://www.iocoder.cn/SkyWalking/register/)
- [2. Collector 注册相关 API](http://www.iocoder.cn/SkyWalking/register/)
  - [2.1 应用的注册 API](http://www.iocoder.cn/SkyWalking/register/)
  - [2.2 应用实例的正常注册 API](http://www.iocoder.cn/SkyWalking/register/)
  - [2.3 应用实例的恢复注册 API](http://www.iocoder.cn/SkyWalking/register/)
  - [2.4 应用实例的心跳 API](http://www.iocoder.cn/SkyWalking/register/)
- [3. Agent 调用注册 API](http://www.iocoder.cn/SkyWalking/register/)
- [666. 彩蛋](http://www.iocoder.cn/SkyWalking/register/)

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

本文主要分享 **应用与应用实例的注册**。先来简单了解下注册的整体流程：

* 应用启动，Agent 向 Collector 注册**应用**。
* 注册应用成功后，Agent 向 Collector 注册应用**实例**。

下面，我们分成两个小节，分别从 API 的**实现**与**调用**，分享代码的具体实现。

> 友情提示：推荐阅读 [《探针与Collector间通讯协议》](https://github.com/apache/incubator-skywalking/blob/master/docs/cn/How-to-communicate-with-the-collector-CN.md) 。

# 2. Collector 注册相关 API

Collector 注册相关 API 相关有四个接口：

* 2.1 应用的注册 API
* 2.2 应用实例的正常注册 API
* 2.3 应用实例的恢复注册 API
* 2.4 应用实例的心跳 API

API 处理的流程大体如下：

![](http://www.iocoder.cn/images/SkyWalking/2020_09_25/01.png)

* 绿框部分，【2.3】【2.4】两个 API ，直接 Service 调用 DAO 方法，无需经过 Graph / Stream 相关方法。

## 2.1 应用的注册 API

我们先来看看 API 的定义，[`ApplicationRegisterService.proto`](https://github.com/YunaiV/skywalking/blob/ba73b05b99a05bb67fd485188a6c6e0a4ad5fe57/apm-network/src/main/proto/ApplicationRegisterService.proto#L9) ，如下图所示：

![](http://www.iocoder.cn/images/SkyWalking/2020_09_25/02.png)

* 其中，KeyWithIntegerValue 数据类型，在 [`KeyWithIntegerValue.proto`](https://github.com/YunaiV/skywalking/blob/ba73b05b99a05bb67fd485188a6c6e0a4ad5fe57/apm-network/src/main/proto/KeyWithIntegerValue.proto) 中定义。

### 2.1.1 ApplicationRegisterServiceHandler#register(...)

[`ApplicationRegisterServiceHandler#register(Application, StreamObserver<ApplicationMapping>)`](https://github.com/YunaiV/skywalking/blob/b4b2ff52a7dccd000264677a7a6bbb2285a8cd53/apm-collector/apm-collector-agent-grpc/collector-agent-grpc-provider/src/main/java/org/skywalking/apm/collector/agent/grpc/handler/ApplicationRegisterServiceHandler.java#L49), 代码如下：

* 第 51 行：获得请求的应用编码( `applicationCode` )**数组**。
* 第 54 至 64 行：循环应用编码数组，**获取**或**创建**应用。
    * 第 57 行：调用 `IApplicationIDService#getOrCreate(applicationCode)` 方法，**获取**或**创建**应用，最终获得应用编号( `applicationId` )。
    * 第 60 至 63 行：获得到应用编号( `applicationId != 0` )，则添加到响应。**为什么会存在获得不到的情况呢**？在下文中，我们会看到，实际**异步**保存应用，所以会存在获取失败的情况。当获取失败，调用方( 例如 Agent )可以重新发起该请求进行注册应用，从而在异步保存应用，获取到应用编号。
    * 第 67 至 68 行：响应。

### 2.1.2 IApplicationIDService#getOrCreate(...)

`org.skywalking.apm.collector.agent.stream.service.register.IApplicationIDService` ，继承 [Service](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Service.java) 接口，应用编号服务接口。

* 定义了 [`#getOrCreate(applicationCode)`](https://github.com/YunaiV/skywalking/blob/a9873b9bf07882746bd30f29b3c64f4b44887bf2/apm-collector/apm-collector-agent-stream/collector-agent-stream-define/src/main/java/org/skywalking/apm/collector/agent/stream/service/register/IApplicationIDService.java#L36) **接口**方法，根据应用编码获取或创建应用，并获得应用编号。

`org.skywalking.apm.collector.agent.stream.worker.register.ApplicationIDService` ，实现 IApplicationIDService 接口，应用编号服务实现类。

* 实现了 [`#getOrCreate(applicationCode)`](https://github.com/YunaiV/skywalking/blob/a9873b9bf07882746bd30f29b3c64f4b44887bf2/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/register/ApplicationIDService.java#L64) 方法，代码如下：

    * 第 66 行：调用 `ApplicationCacheService#get(applicationCode)` 方法，从**缓存**中获取应用编号。在 [《SkyWalking 源码分析 —— Collector Cache 缓存组件》](http://www.iocoder.cn/SkyWalking/collector-cache-module/?self) 有详细解析。
    * 第 69 至 76 行：当获取不到应用编号时，获取 Application 对应的 `Graph<Application>` 对象，调用 `Graph#start(application)` 方法，进行流式处理。在这过程中，会保存应用到存储器。
    * 第 77 行：返回应用编号。

### 2.1.3 Graph#start(application)

在 [`#createApplicationRegisterGraph()`](https://github.com/YunaiV/skywalking/blob/a9873b9bf07882746bd30f29b3c64f4b44887bf2/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/graph/RegisterStreamGraph.java#L57) 方法中，我们可以看到 Application 对应的 `Graph<Application>` 对象的创建。

* [`org.skywalking.apm.collector.agent.stream.worker.register.ApplicationRegisterRemoteWorker`](https://github.com/YunaiV/skywalking/blob/a9873b9bf07882746bd30f29b3c64f4b44887bf2/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/register/ApplicationRegisterRemoteWorker.java) ，继承 AbstractRemoteWorker 抽象类，应用注册远程 Worker 。
    * [Factory](https://github.com/YunaiV/skywalking/blob/a9873b9bf07882746bd30f29b3c64f4b44887bf2/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/register/ApplicationRegisterRemoteWorker.java#L55) 内部类，实现 AbstractRemoteWorkerProvider 抽象类，在 [《SkyWalking 源码分析 —— Collector Streaming Computing 流式处理（一）》「3.2.3 AbstractRemoteWorker」](http://www.iocoder.cn/SkyWalking/collector-streaming-first/?self) 有详细解析。
    * **AbstractRemoteWorker** ，在 [《SkyWalking 源码分析 —— Collector Streaming Computing 流式处理（一）》「3.2.3 AbstractRemoteWorker」](http://www.iocoder.cn/SkyWalking/collector-streaming-first/?self) 有详细解析。
    * [`#id()`](https://github.com/YunaiV/skywalking/blob/a9873b9bf07882746bd30f29b3c64f4b44887bf2/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/register/ApplicationRegisterRemoteWorker.java#L42) 实现方法，返回 10006 。
    * [`#selector`](https://github.com/YunaiV/skywalking/blob/a9873b9bf07882746bd30f29b3c64f4b44887bf2/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/register/ApplicationRegisterRemoteWorker.java#L51) 实现方法，返回 `Selector.ForeverFirst` 。在 [《SkyWalking 源码分析 —— Collector Remote 远程通信服务》](http://www.iocoder.cn/SkyWalking/collector-remote-module/?self) 有详细解析。
    * [`#onWork(Application)`](https://github.com/YunaiV/skywalking/blob/a9873b9bf07882746bd30f29b3c64f4b44887bf2/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/register/ApplicationRegisterRemoteWorker.java#L46) 实现方法，调用 `Next#execute(message)` 方法，提交数据给下面的节点，继续流式处理。
    * 总结：ApplicationRegisterRemoteWorker ，使用 Collector 集群的第一个节点( [按照 `ip` 排序](https://github.com/YunaiV/skywalking/blob/b4b2ff52a7dccd000264677a7a6bbb2285a8cd53/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/service/GRPCRemoteClient.java#L194) ) 进行后续的流式处理，即，保存应用。
* [`org.skywalking.apm.collector.agent.stream.worker.register.ApplicationRegisterSerialWorker`](https://github.com/YunaiV/skywalking/blob/a9873b9bf07882746bd30f29b3c64f4b44887bf2/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/register/ApplicationRegisterSerialWorker.java) ，继承 AbstractLocalAsyncWorker 抽象类，异步保存应用 Worker 。
    * [Factory](https://github.com/YunaiV/skywalking/blob/a9873b9bf07882746bd30f29b3c64f4b44887bf2/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/register/ApplicationRegisterSerialWorker.java#L84) 内部类，实现 AbstractLocalAsyncWorkerProvider 抽象类，在 [《SkyWalking 源码分析 —— Collector Streaming Computing 流式处理（一）》「3.2.2 AbstractLocalAsyncWorker」](http://www.iocoder.cn/SkyWalking/collector-streaming-first/?self) 有详细解析。
    * **AbstractLocalAsyncWorker** ，在 [《SkyWalking 源码分析 —— Collector Streaming Computing 流式处理（一）》「3.2.2 AbstractLocalAsyncWorker」](http://www.iocoder.cn/SkyWalking/collector-streaming-first/?self) 有详细解析。
    * [`#id()`](https://github.com/YunaiV/skywalking/blob/a9873b9bf07882746bd30f29b3c64f4b44887bf2/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/register/ApplicationRegisterSerialWorker.java#L52) 实现方法，返回 101 。
    * [`#onWork(Application)`](https://github.com/YunaiV/skywalking/blob/69ec2bb309f4fe3cdbda56587a6193be933c4388/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/register/ApplicationRegisterSerialWorker.java#L56) 实现方法，保存应用( Application )。代码如下( 以 ES 作为 DAO 实现为例子 )：
        * 第 58 行：调用 `ApplicationCacheService#get(applicationCode)` 方法，从**缓存**中获取应用编号。
        * 第 60 行：当获取不到应用编号时，使用 `applicationCode` 创建应用并保存。
        * 第 62 行：调用 [`ApplicationH2RegisterDAO#getMinApplicationId()`](https://github.com/YunaiV/skywalking/blob/69ec2bb309f4fe3cdbda56587a6193be933c4388/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/ApplicationEsRegisterDAO.java#L49) 方法，获得 Application 记录的应用编号的**最小值**。
        * --------- 分隔 ---------
        * 第 63 行：当 `min == 0` 时，说明没有 Application 记录。
        * 第 64 至 68 行：创建**第一条**、**特殊**的 Application 记录。该记录 `applicationId = 1` ，`applicationCode = User` ，用于表示用户发起请求。在 SkyWaling UI 中，我们可以看到该条 Application 记录如下图：![](http://www.iocoder.cn/images/SkyWalking/2020_09_25/03.png)
        * 第 70 至 74 行：创建当前请求的对应的 Application 记录，并且 `applicationId = -1` 。
        * --------- 分隔 ---------
        * 第 76 行：调用 [`ApplicationH2RegisterDAO#getMaxApplicationId()`](https://github.com/YunaiV/skywalking/blob/69ec2bb309f4fe3cdbda56587a6193be933c4388/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/ApplicationEsRegisterDAO.java#L45) 方法，获得 Application 记录的应用编号的**最大值**。
        * 第 77 行：调用 `IdAutoIncrement#increment(min, max)` 方法，获得应用编号。该方法较为有趣，在下文详细解析。
        * 第 79 至 82 行：创建当前请求的对应的 Application 记录。
        * --------- 分隔 ---------
        * 第 85 行：调用 [`ApplicationEsRegisterDAO#save(Application)`](https://github.com/YunaiV/skywalking/blob/69ec2bb309f4fe3cdbda56587a6193be933c4388/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/ApplicationEsRegisterDAO.java#L53) 方法，保存应用。

-------

[`IdAutoIncrement#increment(min, max)`](https://github.com/YunaiV/skywalking/blob/69ec2bb309f4fe3cdbda56587a6193be933c4388/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/IdAutoIncrement.java#L36) 方法，**双向**均匀自增。可能看起来比较奇怪，以上文 Application 的调用举例子：


| min | max | result | applicationCode |
| --- | --- | --- | --- |
| 0 | / | 1 | User |
| 0 | / | -1 | 应用 A |
| -1 | 1 | 2 | 应用 B |
| -1 | 2 | -2 | 应用 C |
| -2 | 2 | 3 | 应用 D |

* 【User】和【应用 A】是直接获得 `result` ，不调用 `#increment(min, max)`  方法。
* 总的来说，我们可以看到，以 `min + max = 0` 为中心点( 实际以 `0` 为中心点)， **双向**均匀自增。

TODO 【4007】

### 2.1.4 Application

[`org.skywalking.apm.collector.storage.table.register.Application`](https://github.com/YunaiV/skywalking/blob/69ec2bb309f4fe3cdbda56587a6193be933c4388/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/register/Application.java) ，应用。例如记录在 ES 如下图：

![](http://www.iocoder.cn/images/SkyWalking/2020_09_25/05.png)

## 2.2 应用实例的正常注册 API

我们先来看看 API 的定义，[`InstanceDiscoveryService`](https://github.com/YunaiV/skywalking/blob/ba73b05b99a05bb67fd485188a6c6e0a4ad5fe57/apm-network/src/main/proto/InstanceDiscoveryService.proto#L11) ，如下图所示：

![](http://www.iocoder.cn/images/SkyWalking/2020_09_25/04.png)

整体代码和 [「2.1 应用的注册 API」](#) 非常相似，所以本小节，更多的是提供代码的链接地址。

### 2.2.1 InstanceDiscoveryServiceHandler#register(...)

[`InstanceDiscoveryServiceHandler#register(ApplicationInstance, StreamObserver<ApplicationInstanceMapping>)`](https://github.com/YunaiV/skywalking/blob/69ec2bb309f4fe3cdbda56587a6193be933c4388/apm-collector/apm-collector-agent-grpc/collector-agent-grpc-provider/src/main/java/org/skywalking/apm/collector/agent/grpc/handler/InstanceDiscoveryServiceHandler.java#L54)，注册应用实例。

### 2.2.2 IInstanceIDService#getOrCreate(...)

`org.skywalking.apm.collector.agent.stream.service.register.IInstanceIDService` ，继承 [Service](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Service.java) 接口，应用实例编号服务接口。

* 定义了 [`#getOrCreate(applicationCode)`](https://github.com/YunaiV/skywalking/blob/a4db2c4dd5e2adc861e7fb9e9b7b7ffdc57dfb88/apm-collector/apm-collector-agent-stream/collector-agent-stream-define/src/main/java/org/skywalking/apm/collector/agent/stream/service/register/IInstanceIDService.java#L39) **接口**方法，根据应用编号 + AgentUUID，获取或创建应用实例，并获得应用编号。

`org.skywalking.apm.collector.agent.stream.worker.register.InstanceIDService` ，实现 IInstanceIDService 接口，应用编号服务实现类。

* 实现了 [`#getOrCreate(applicationCode)`](https://github.com/YunaiV/skywalking/blob/a4db2c4dd5e2adc861e7fb9e9b7b7ffdc57dfb88/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/register/InstanceIDService.java#L74) 方法。

### 2.2.3 Graph#start(instance)

在 [`#createInstanceRegisterGraph()`](https://github.com/YunaiV/skywalking/blob/a9873b9bf07882746bd30f29b3c64f4b44887bf2/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/graph/RegisterStreamGraph.java#L68) 方法中，我们可以看到 Instance 对应的 `Graph<Instance>` 对象的创建。

* [`org.skywalking.apm.collector.agent.stream.worker.register.InstanceRegisterRemoteWorker`](https://github.com/YunaiV/skywalking/blob/a9873b9bf07882746bd30f29b3c64f4b44887bf2/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/register/InstanceRegisterRemoteWorker.java) ，继承 AbstractRemoteWorker 抽象类，应用实例注册远程 Worker 。
* [`org.skywalking.apm.collector.agent.stream.worker.register.ApplicationRegisterSerialWorker`](https://github.com/YunaiV/skywalking/blob/a9873b9bf07882746bd30f29b3c64f4b44887bf2/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/register/ApplicationRegisterSerialWorker.java) ，继承 AbstractLocalAsyncWorker 抽象类，异步保存应用 Worker 。
    * 不同 Application ，Instance 的应用实例编号，从 `"1"` **正向**递增。
    * [InstanceEsRegisterDAO#save(Instance)](https://github.com/YunaiV/skywalking/blob/a4db2c4dd5e2adc861e7fb9e9b7b7ffdc57dfb88/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/InstanceEsRegisterDAO.java#L53)

### 2.2.4 Instance

[`org.skywalking.apm.collector.storage.table.register.Instance`](https://github.com/YunaiV/skywalking/blob/69ec2bb309f4fe3cdbda56587a6193be933c4388/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/register/Instance.java) ，应用实例。例如记录在 ES 如下图：

![](http://www.iocoder.cn/images/SkyWalking/2020_09_25/06.png)

## 2.3 应用实例的恢复注册 API

我们先来看看 API 的定义，[`InstanceDiscoveryService.proto`](https://github.com/YunaiV/skywalking/blob/ba73b05b99a05bb67fd485188a6c6e0a4ad5fe57/apm-network/src/main/proto/InstanceDiscoveryService.proto#L17) ，如下图所示：

![](http://www.iocoder.cn/images/SkyWalking/2020_09_25/07.png)

* 其中，Downstream 数据类型，在 [`Downstream.proto`](https://github.com/YunaiV/skywalking/blob/ba73b05b99a05bb67fd485188a6c6e0a4ad5fe57/apm-network/src/main/proto/Downstream.proto) 中定义。

一般情况下，Agent 在注册应用时候成功后，如果因为各种原因原因和 Collector 断开了 gRPC Channel 连接( 例如，网络 )，恢复连接后，需要调用该 API ，进行**恢复**注册。

### 2.3.1 InstanceDiscoveryServiceHandler#recover(...)

[`InstanceDiscoveryServiceHandler#recover(ApplicationInstanceRecover, StreamObserver<Downstream>)`](https://github.com/YunaiV/skywalking/blob/9a01c67f47efc07b9754c77198324cb2d5bb212d/apm-collector/apm-collector-agent-grpc/collector-agent-grpc-provider/src/main/java/org/skywalking/apm/collector/agent/grpc/handler/InstanceDiscoveryServiceHandler.java#L69), 代码如下：

* 第 71 行：调用 [`TimeBucketUtils#getSecondTimeBucket(time)`](https://github.com/YunaiV/skywalking/blob/9a01c67f47efc07b9754c77198324cb2d5bb212d/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/util/TimeBucketUtils.java#L51) 方法，将 `registerTime` 转成 timeBucket 。
* 第 73 行：调用 `IInstanceIDService#recover(instanceId, applicationId, registerTime, osInfo)` 方法，恢复注册应用实例。
* 第 75 至 76 行：响应。

### 2.3.2 IInstanceIDService#recover(...)

[`InstanceIDService#recover(instanceId, applicationId, registerTime, osInfo)`](https://github.com/YunaiV/skywalking/blob/bb8cb1b7dcb428c161f225f0b5d57441105f84c0/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/register/InstanceIDService.java#L95) **实现**方法，恢复注册。代码如下：

* 第 96 至 103 行：创建 Instance 对象，用于下面更新操作。
    * 第 99 行： TODO 【4008】
* 第 106 行：调用 [InstanceEsRegisterDAO#save(Instance)](https://github.com/YunaiV/skywalking/blob/a4db2c4dd5e2adc861e7fb9e9b7b7ffdc57dfb88/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/InstanceEsRegisterDAO.java#L53) 方法，更新应用实例。

## 2.4 应用实例的心跳 API

我们先来看看 API 的定义，[`InstanceDiscoveryService.proto`](https://github.com/YunaiV/skywalking/blob/ba73b05b99a05bb67fd485188a6c6e0a4ad5fe57/apm-network/src/main/proto/InstanceDiscoveryService.proto#L14) ，如下图所示：

![](http://www.iocoder.cn/images/SkyWalking/2020_09_25/08.png)

* 其中，Downstream 数据类型，在 [`Downstream.proto`](https://github.com/YunaiV/skywalking/blob/ba73b05b99a05bb67fd485188a6c6e0a4ad5fe57/apm-network/src/main/proto/Downstream.proto) 中定义。

一般情况下，Agent 在注册应用时候成功后，定时向 Collector 发送**心跳**，记录应用存活。

### 2.4.1 InstanceDiscoveryServiceHandler#heartbeat(...)

[InstanceDiscoveryServiceHandler#heartbeat(ApplicationInstanceHeartbeat, StreamObserver<org.skywalking.apm.network.proto.Downstream>)](#) ，目前该方法暂未实现。实现后，会首先调用一个 Service 方法，而后调用 [`InstanceEsRegisterDAO#updateHeartbeatTime(instanceId, heartbeatTime)`](https://github.com/YunaiV/skywalking/blob/bb8cb1b7dcb428c161f225f0b5d57441105f84c0/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/InstanceEsRegisterDAO.java#L68) 方法，记录应用实例的心跳时间。

# 3. Agent 调用注册 API

[`org.skywalking.apm.agent.core.remote.AppAndServiceRegisterClient`](https://github.com/YunaiV/skywalking/blob/bb8cb1b7dcb428c161f225f0b5d57441105f84c0/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/AppAndServiceRegisterClient.java) ，实现 BootService 、GRPCChannelListener 、Runnable 、TracingContextListener 接口，注册应用与实例的客户端。该客户端会调用上述所有 API 。

* `PROCESS_UUID` **静态**属性，Agent UUID ，使用 **UUID** 算法生成，去除多余 `"-"` 。
* ---------- 分割线 ----------
* `status` 属性，gRPC 连接状态。
* `applicationRegisterServiceBlockingStub` / `instanceDiscoveryServiceBlockingStub` / `serviceNameDiscoveryServiceBlockingStub` 属性，对应 gRPC 提供 API 的**阻塞** Stub 。
* `needRegisterRecover` 属性，是否需要发起恢复的注册。
* 如上五个属性，在 [`#statusChanged(GRPCChannelStatus)`](https://github.com/YunaiV/skywalking/blob/bb8cb1b7dcb428c161f225f0b5d57441105f84c0/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/AppAndServiceRegisterClient.java#L79) **实现**方法，根据 gRPC 连接状态的变更，创建或销毁 Stub 。
* 在 [`#boot()`](https://github.com/YunaiV/skywalking/blob/bb8cb1b7dcb428c161f225f0b5d57441105f84c0/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/AppAndServiceRegisterClient.java#L103) **实现**方法，将自己作为监听器( 因为实现了 GRPCChannelListener 接口 )添加到 GRPCChannelManager 中，从而监听 gRPC Channel 的状态。在 [《SkyWalking 源码分析 —— Agent Remote 远程通信服务》](http://www.iocoder.cn/SkyWalking/agent-remote-manager/?self) 有详细解析。
* ---------- 分割线 ----------
* `applicationRegisterFuture` 属性，注册应用与实例的定时任务。
* [`#boot()`](https://github.com/YunaiV/skywalking/blob/bb8cb1b7dcb428c161f225f0b5d57441105f84c0/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/AppAndServiceRegisterClient.java#L103) **实现**方法，创建 `applicationRegisterFuture` 。该定时任务**无初始化延迟**，每 `Config.Collector.APP_AND_SERVICE_REGISTER_CHECK_INTERVAL` ( 默认：3 s ) 执行一次 `#run()` 方法。
* ---------- 分割线 ----------
* `lastSegmentTime` 属性，最后记录 Segment 的时间。
* [`#afterFinished()`](https://github.com/YunaiV/skywalking/blob/bb8cb1b7dcb428c161f225f0b5d57441105f84c0/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/AppAndServiceRegisterClient.java#L186) **实现**方法，记录 Segment 最后的时间。
* [`#afterBoot()`](https://github.com/YunaiV/skywalking/blob/bb8cb1b7dcb428c161f225f0b5d57441105f84c0/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/AppAndServiceRegisterClient.java#L103) **实现**方法，将自己作为监听器( 因为实现了 TracingContextListener 接口 )添加到 GRPCChannelManager 中，从而监听 Segment 的记录。在 [《SkyWalking 源码分析 —— Agent 收集 Trace 数据》](http://www.iocoder.cn/SkyWalking/agent-collect-trace/?self) 有详细解析。

-------

[`#run()`](https://github.com/YunaiV/skywalking/blob/bb8cb1b7dcb428c161f225f0b5d57441105f84c0/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/AppAndServiceRegisterClient.java#L120) **实现**方法，执行应用的注册，应用实例的正常注册、恢复注册、心跳的逻辑。

* 第 123 行：循环，当 gRPC 处于**连接中**，并且需要重试( `shouldTry` )。可能对 `shouldTry` 会比较疑惑？该变量用于，应用的注册成功后，重新标记 `shouldTry = true` ，继续执行应用实例的注册。
* 第 126 至 135 行：当本地**应用编号**为空时，说明应用暂未注册，调用 [「2.1 应用的注册 API」](#) 。
* 第 138 至 148 行：当本地**应用实例编号**为空时，说明应用实例暂未注册，调用 [「2.2 应用实例的正常注册 API」](#) 。
* 第 150 至 158 行：当需要发起恢复注册时，即 gRPC Channel 断开后重连成功，调用 [「2.3 应用实例的恢复注册 API」](#) 。
* 第 159 至 167 行：当**现在时间**超过 `lastSegmentTime` **一分钟**，调用 [「2.4 应用实例的心跳 API」](#) 。
* 第 170 至 173 行：同步应用字典、操作字典。在 [《SkyWalking 源码分析 —— Agent DictionaryManager 字典管理》](http://www.iocoder.cn/SkyWalking/agent-dictionary/?) 详细解析。
* 第 178 至 180 行：当发生异常时，调用 `GRPCChannelManager#reportError(t)` 方法，处理异常，例如请求超时。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

😈 距离 Segment 已经不远了。

胖友，分享个朋友圈可好？


