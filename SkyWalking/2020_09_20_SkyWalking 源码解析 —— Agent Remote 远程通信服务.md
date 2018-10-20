title: SkyWalking 源码分析 —— Agent Remote 远程通信服务
date: 2020-09-20
tags:
categories: SkyWalking
permalink: SkyWalking/agent-remote-manager

-------

摘要: 原创出处 http://www.iocoder.cn/SkyWalking/agent-remote-manager/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 SkyWalking 3.2.6 正式版**

- [1. 概述](http://www.iocoder.cn/SkyWalking/agent-remote-manager/)
- [2. GRPCChannelManager](http://www.iocoder.cn/SkyWalking/agent-remote-manager/)
- [3. GRPCChannelListener](http://www.iocoder.cn/SkyWalking/agent-remote-manager/)
- [666. 彩蛋](http://www.iocoder.cn/SkyWalking/agent-remote-manager/)

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

本文主要分享 **SkyWalking Agent Remote 远程通信服务**。该服务用于 Agent 和Collector 集群的通信。

![](http://www.iocoder.cn/images/SkyWalking/2020_09_20/01.png)

在 [《SkyWalking 源码分析 —— Collector Naming Server 命名服务》](http://www.iocoder.cn/SkyWalking/collector-naming-server/?self) 一文中，我们已经看到，Agent 使用**定时轮询**，从 Collector Naming Server 中，获得 Collector 集群的 Collector Agent gRPC Server 的**所有地址**。

![](http://www.iocoder.cn/images/SkyWalking/2020_09_20/02.jpeg)

* **红框**部分，即为 Agent 和Collector 集群的通信部分。
* 另外，Collector 也提供 [Collector Agent Jetty Server](https://github.com/YunaiV/skywalking/tree/ba73b05b99a05bb67fd485188a6c6e0a4ad5fe57/apm-collector/apm-collector-agent-jetty) ，目前暂不使用。相比来说，[Collector Agent gRPC Server](https://github.com/YunaiV/skywalking/tree/ba73b05b99a05bb67fd485188a6c6e0a4ad5fe57/apm-collector/apm-collector-agent-grpc) 性能更优。

# 2. GRPCChannelManager

`org.skywalking.apm.agent.core.remote.GRPCChannelManager` ，实现 [BootService](https://github.com/YunaiV/skywalking/blob/ac6c98c1732d6aa62b9d244369478654411ac203/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/boot/BootService.java) 、Runnable **接口**，gRPC Channel 管理器。GRPCChannelManager 负责管理与 Collector Agent gRPC Server 集群的**连接的管理**，**提供给其他服务使用**。

* [`managedChannel`](https://github.com/YunaiV/skywalking/blob/ba73b05b99a05bb67fd485188a6c6e0a4ad5fe57/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/GRPCChannelManager.java#L54) 属性，连接 gRPC Server 的 Channel 。**同一时间，GRPCChannelManager 只连接一个 Collector Agent gRPC Server 节点，并且在 Channel 不因为各种网络问题断开的情况下，持续保持**。
* [`connectCheckFuture`](https://github.com/YunaiV/skywalking/blob/ba73b05b99a05bb67fd485188a6c6e0a4ad5fe57/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/GRPCChannelManager.java#L58) 属性，定时重连 gRPC Server 的**定时任务**。
* [`reconnect`](https://github.com/YunaiV/skywalking/blob/ba73b05b99a05bb67fd485188a6c6e0a4ad5fe57/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/GRPCChannelManager.java#L63) 属性，是否重连。当 Channel 未连接需要连接，或者 Channel 断开需要重连时，标记 `reconnect = true` 。后台线程会根据该标识进行连接( 重连 )。
* [`listeners`](https://github.com/YunaiV/skywalking/blob/ba73b05b99a05bb67fd485188a6c6e0a4ad5fe57/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/GRPCChannelManager.java#L68) 属性，监听器( `org.skywalking.apm.agent.core.remote.GRPCChannelListener` ) 数组。使用 Channel 的其他服务，注册监听器到 GRPCChannelManager 上，从而根据连接状态( `org.skywalking.apm.agent.core.remote.GRPCChannelStatus` )，实现自定义逻辑。

[`#boot()`](https://github.com/YunaiV/skywalking/blob/ba73b05b99a05bb67fd485188a6c6e0a4ad5fe57/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/GRPCChannelManager.java#L76) **实现**方法，调用 `ScheduledExecutorService#scheduleAtFixedRate(...)` 方法，创建定时任务。该定时任务**无初始化延迟**，每 `Config.GRPC_CHANNEL_CHECK_INTERVAL` ( 默认：30 s ) 执行一次 `#run()` 方法。

[`#run()`](https://github.com/YunaiV/skywalking/blob/ba73b05b99a05bb67fd485188a6c6e0a4ad5fe57/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/GRPCChannelManager.java#L97) **实现**方法，执行 Channel 的连接( 重连 )逻辑。代码如下：

* 第 99 行：当 `reconnect = true` 时，才执行连接( 重连 )。
* 第 100 行：当本地已经获取到 Collector Agent gRPC Server 集群地址时，参见 [《SkyWalking 源码分析 —— Collector Naming Server 命名服务》](http://www.iocoder.cn/SkyWalking/collector-naming-server/?self) 。
* 第 101 至 106 行：**随机选择**准备链接的 Collector Agent gRPC Server 。
* 第 107 至 113 行：创建 Channel 并进行连接。此处主要是 gRPC 的 API 使用，不熟悉的胖友，请 Google 下进行了解了解。
* 第 115 至 117 行：连接**成功**，标记 `reconnect = false` ，这样，下次执行 `#run()` 方法不会重连。而后，调用 [`#notify(GRPCChannelStatus.CONNECTED)`](https://github.com/YunaiV/skywalking/blob/ba73b05b99a05bb67fd485188a6c6e0a4ad5fe57/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/GRPCChannelManager.java#L160) 方法，通知监听器连接成功。
* 第 118 至 121 行：连接**成功**，不标记 `reconnect` ，这样，下次执行 `#run()` 方法会继续重连。而后，调用 [`#notify(GRPCChannelStatus.DISCONNECT)`](https://github.com/YunaiV/skywalking/blob/ba73b05b99a05bb67fd485188a6c6e0a4ad5fe57/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/GRPCChannelManager.java#L160) 方法，通知监听器连接处于断开状态。
* 第 124 至 126 行：连接**异常**，不标记 `reconnect` ，这样，下次执行 `#run()` 方法会继续重连。而后，调用 [`#notify(GRPCChannelStatus.DISCONNECT)`](https://github.com/YunaiV/skywalking/blob/ba73b05b99a05bb67fd485188a6c6e0a4ad5fe57/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/GRPCChannelManager.java#L160) 方法，通知监听器连接处于断开状态。

实际使用中，Channel 可能因为各种原因断开，那么 GRPCChannelManager 是怎么检测的呢？在使用 Channel 的**其他服务**，当使用 Channel 时发生异常，调用 [`#reportError(Throwable)`](https://github.com/YunaiV/skywalking/blob/ba73b05b99a05bb67fd485188a6c6e0a4ad5fe57/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/GRPCChannelManager.java#L149) 方法，判断是否为网络异常( [`#isNetworkError(Throwable)`](https://github.com/YunaiV/skywalking/blob/ba73b05b99a05bb67fd485188a6c6e0a4ad5fe57/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/GRPCChannelManager.java#L176) ) 。若是，标记 `reconnect = true` ，等待后台进行重连。

# 3. GRPCChannelListener

`org.skywalking.apm.agent.core.remote.GRPCChannelListener` ，gRPC Channel 的监听器**接口**，定义了 [`#statusChanged(GRPCChannelStatus)`](https://github.com/YunaiV/skywalking/blob/ba73b05b99a05bb67fd485188a6c6e0a4ad5fe57/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/GRPCChannelListener.java#L33) ，通知 gRPC Channel 状态变更。

GRPCChannelListener 实现类如下图，后续文章会详细解析。

![](http://www.iocoder.cn/images/SkyWalking/2020_09_20/03.png)

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

"水"文一篇，哈哈哈。

胖友，分享一波朋友圈可好。


