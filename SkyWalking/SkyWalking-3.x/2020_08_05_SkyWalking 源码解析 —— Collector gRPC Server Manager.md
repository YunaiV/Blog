title: SkyWalking 源码分析 —— Collector gRPC Server Manager
date: 2020-08-05
tags:
categories: SkyWalking
permalink: SkyWalking/collector-grpc-server-module

-------

摘要: 原创出处 http://www.iocoder.cn/SkyWalking/collector-grpc-server-module/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 SkyWalking 3.2.6 正式版**

- [1. 概述](http://www.iocoder.cn/SkyWalking/collector-grpc-server-module/)
- [2. GRPCManagerModule](http://www.iocoder.cn/SkyWalking/collector-grpc-server-module/)
- [3. GRPCManagerProvider](http://www.iocoder.cn/SkyWalking/collector-grpc-server-module/)
- [4. GRPCManagerService](http://www.iocoder.cn/SkyWalking/collector-grpc-server-module/)
- [666. 彩蛋](http://www.iocoder.cn/SkyWalking/collector-grpc-server-module/)

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

本文主要分享 **Collector gRPC Server Manager**。Collector 通过该管理器，管理启动的多个 gRPC Server，例如 Agent gRPC Server、Remote gRPC Server 。![](http://www.iocoder.cn/images/SkyWalking/2020_08_05/02.png)

> 友情提示：建议胖友已经读过 [《SkyWalking 源码分析 —— Collector Server Component 服务器组件》](http://www.iocoder.cn/SkyWalking/collector-server-component/?self)
>
> 另外，本文和 [《SkyWalking 源码分析 —— Collector Jetty Server Manager》](http://www.iocoder.cn/SkyWalking/collector-jetty-server-module/?self) 相似度 99%

gRPC Server Manager 在 SkyWalking 架构图处于如下位置( **红框** ) ：

> FROM https://github.com/apache/incubating-skywalking  
> ![](http://www.iocoder.cn/images/SkyWalking/2020_08_05/01.jpeg)

下面我们来看看整体的项目结构，如下图所示 ：

![](http://www.iocoder.cn/images/SkyWalking/2020_08_05/03.png)

😈 代码量非常少，考虑到这是个单独的项目，所以单独成文。

# 2. GRPCManagerModule

`org.skywalking.apm.collector.grpc.manager.GRPCManagerModule` ，实现 [Module](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Module.java) 抽象类，gRPC Server 管理器 Module 。

[`#name()`](https://github.com/YunaiV/skywalking/blob/621598af465bfcefee3432c2ef80aff25a33f1bf/apm-collector/apm-collector-grpc-manager/collector-grpc-manager-define/src/main/java/org/skywalking/apm/collector/grpc/manager/GRPCManagerModule.java#L33) **实现**方法，返回模块名为 `"gRPC_manager"` 。

[`#services()`](https://github.com/YunaiV/skywalking/blob/621598af465bfcefee3432c2ef80aff25a33f1bf/apm-collector/apm-collector-grpc-manager/collector-grpc-manager-define/src/main/java/org/skywalking/apm/collector/grpc/manager/GRPCManagerModule.java#L37) **实现**方法，返回 Service 类名：GRPCManagerService 。

# 3. GRPCManagerProvider

`org.skywalking.apm.collector.grpc.manager.GRPCManagerProvider` ，实现 [ModuleProvider](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/ModuleProvider.java) **抽象类**，gRPC Server 管理器组件服务提供者。

[`#name()`](https://github.com/YunaiV/skywalking/blob/f9de7bf75f62c16fd05cc0d1beb8f5b756108ec3/apm-collector/apm-collector-grpc-manager/collector-grpc-manager-provider/src/main/java/org/skywalking/apm/collector/grpc/manager/GRPCManagerProvider.java#L46) **实现**方法，返回组件服务提供者名为 `"gRPC"` 。

[`module()`](https://github.com/YunaiV/skywalking/blob/f9de7bf75f62c16fd05cc0d1beb8f5b756108ec3/apm-collector/apm-collector-grpc-manager/collector-grpc-manager-provider/src/main/java/org/skywalking/apm/collector/grpc/manager/GRPCManagerProvider.java#L50) **实现**方法，返回组件类为 GRPCManagerModule 。

[`#requiredModules()`](https://github.com/YunaiV/skywalking/blob/f9de7bf75f62c16fd05cc0d1beb8f5b756108ec3/apm-collector/apm-collector-grpc-manager/collector-grpc-manager-provider/src/main/java/org/skywalking/apm/collector/grpc/manager/GRPCManagerProvider.java#L72) **实现**方法，返回依赖组件为空。

-------

[`#prepare(Properties)`](https://github.com/YunaiV/skywalking/blob/f9de7bf75f62c16fd05cc0d1beb8f5b756108ec3/apm-collector/apm-collector-grpc-manager/collector-grpc-manager-provider/src/main/java/org/skywalking/apm/collector/grpc/manager/GRPCManagerProvider.java#L54) **实现**方法，执行准备阶段逻辑。

* 第 55 行 ：创建 GRPCManagerServiceImpl 对象，并调用 `#registerServiceImplementation(...)` **父类**方法，注册到 `services` 。

[`#start()`](https://github.com/YunaiV/skywalking/blob/f9de7bf75f62c16fd05cc0d1beb8f5b756108ec3/apm-collector/apm-collector-grpc-manager/collector-grpc-manager-provider/src/main/java/org/skywalking/apm/collector/grpc/manager/GRPCManagerProvider.java#L58) **实现**方法，执行启动阶段逻辑。目前是个**空方法**。

[`#notifyAfterCompleted()`](https://github.com/YunaiV/skywalking/blob/f9de7bf75f62c16fd05cc0d1beb8f5b756108ec3/apm-collector/apm-collector-grpc-manager/collector-grpc-manager-provider/src/main/java/org/skywalking/apm/collector/grpc/manager/GRPCManagerProvider.java#L62) **实现**方法，执行启动完成逻辑。

* 第 63 至 69 行 ：遍历注册的服务器列表，逐个调用 `GRPCServer#start()` 方法，进行启动。

# 4. GRPCManagerService

`org.skywalking.apm.collector.grpc.manager.service.GRPCManagerService` ，继承 [Service](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Service.java) **接口**，gRPC Server 管理器服务**接口**。

[`#createIfAbsent(host, port)`](https://github.com/YunaiV/skywalking/blob/48f76a555c043fee2932230077a8112d4888d10f/apm-collector/apm-collector-grpc-manager/collector-grpc-manager-define/src/main/java/org/skywalking/apm/collector/grpc/manager/service/GRPCManagerService.java#L38) **接口**方法，创建 gRPC Server ，若不存在。

**怎么没有类似 JettyManagerService 的 `#addHandler(...)` 方法**？目前是调用方直接调用 `#createIfAbsent(host, port)` 方法，获得 gRPC Server 后，后调用 `Server#addHandler(ServerHandler)` 方法。例如：

* [`AgentModuleGRPCProvider#start(Properties)`](https://github.com/YunaiV/skywalking/blob/9c586ca730cc89d4d5ad6b4294f2779a23925a8c/apm-collector/apm-collector-agent-grpc/collector-agent-grpc-provider/src/main/java/org/skywalking/apm/collector/agent/grpc/AgentModuleGRPCProvider.java#L72)
* [`RemoteModuleGRPCProvider#start(Properties)`](https://github.com/YunaiV/skywalking/blob/9c586ca730cc89d4d5ad6b4294f2779a23925a8c/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/RemoteModuleGRPCProvider.java#L61)

## 4.1 GRPCManagerServiceImpl

`org.skywalking.apm.collector.grpc.manager.service.GRPCManagerServiceImpl` ，gRPC Server 管理器服务**实现类**。

[**构造方法**](https://github.com/YunaiV/skywalking/blob/9c586ca730cc89d4d5ad6b4294f2779a23925a8c/apm-collector/apm-collector-grpc-manager/collector-grpc-manager-provider/src/main/java/org/skywalking/apm/collector/grpc/manager/service/GRPCManagerServiceImpl.java#L44) ，使用来自 GRPCManagerProvider 的 `servers` 服务器数组。**这是为什么 GRPCManagerProvider 没有对 `servers` 做新增操作，结果里面有数据的原因**。

[`#createIfAbsent(host, port)`](https://github.com/YunaiV/skywalking/blob/9c586ca730cc89d4d5ad6b4294f2779a23925a8c/apm-collector/apm-collector-grpc-manager/collector-grpc-manager-provider/src/main/java/org/skywalking/apm/collector/grpc/manager/service/GRPCManagerServiceImpl.java#L48) **实现**方法，创建 gRPC Server ，若不存在。判断方式为 `host + port` 为唯一。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

呵呵哒的一篇，嘿嘿。

胖友，分享个朋友圈可好？


