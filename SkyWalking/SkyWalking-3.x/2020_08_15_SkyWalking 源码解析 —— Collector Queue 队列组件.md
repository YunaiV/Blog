title: SkyWalking 源码分析 —— Collector Queue 队列组件
date: 2020-08-15
tags:
categories: SkyWalking
permalink: SkyWalking/collector-queue-module

-------

摘要: 原创出处 http://www.iocoder.cn/SkyWalking/collector-queue-module/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 SkyWalking 3.2.6 正式版**

- [1. 概述](http://www.iocoder.cn/SkyWalking/collector-queue-module/)
- [2. collector-queue-define](http://www.iocoder.cn/SkyWalking/collector-queue-module/)
  - [2.1 QueueModule](http://www.iocoder.cn/SkyWalking/collector-queue-module/)
  - [2.2 QueueCreatorService](http://www.iocoder.cn/SkyWalking/collector-queue-module/)
  - [2.3 MessageHolder](http://www.iocoder.cn/SkyWalking/collector-queue-module/)
  - [2.4 QueueEventHandler](http://www.iocoder.cn/SkyWalking/collector-queue-module/)
  - [2.5 DaemonThreadFactory](http://www.iocoder.cn/SkyWalking/collector-queue-module/)
- [3. collector-queue-disruptor-provider](http://www.iocoder.cn/SkyWalking/collector-queue-module/)
  - [3.1 QueueModuleDisruptorProvider](http://www.iocoder.cn/SkyWalking/collector-queue-module/)
  - [2.2 DisruptorQueueCreatorService](http://www.iocoder.cn/SkyWalking/collector-queue-module/)
  - [3.3 DisruptorEventHandler](http://www.iocoder.cn/SkyWalking/collector-queue-module/)
- [4. collector-queue-datacarrier-provider](http://www.iocoder.cn/SkyWalking/collector-queue-module/)
- [666. 彩蛋](http://www.iocoder.cn/SkyWalking/collector-queue-module/)

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

本文主要分享 **SkyWalking Collector Queue Module**，队列组件。该组件被 Collector Streaming Module 流式处理使用，提供**异步**执行的特性。

> 友情提示：建议先阅读 [《SkyWalking 源码分析 —— Collector 初始化》](http://www.iocoder.cn/SkyWalking/collector-init/?self) ，以了解 Collector 组件体系。

Cluster Module 在 SkyWalking 架构图处于如下位置( **红框** ) ：

> FROM https://github.com/apache/incubating-skywalking  
> ![](http://www.iocoder.cn/images/SkyWalking/2020_08_15/01.jpeg)

下面我们来看看整体的项目结构，如下图所示 ：

![](http://www.iocoder.cn/images/SkyWalking/2020_08_15/02.png)

* `collector-queue-define` ：定义队列组件接口。
* `collector-queue-datacarrier-provider` ：基于 [apm-datacarrier](https://github.com/YunaiV/skywalking/tree/master/apm-commons/apm-datacarrier) 的队列组件实现。*目前暂未完成*。
* `collector-queue-zookeeper-provider` ：基于 [Disruptor](https://github.com/LMAX-Exchange/disruptor) 的队列组件实现。

下面，我们从**接口到实现**的顺序进行分享。

# 2. collector-queue-define

`collector-queue-define` ：定义队列组件接口。项目结构如下 ：

![](http://www.iocoder.cn/images/SkyWalking/2020_08_15/03.png)

## 2.1 QueueModule

`org.skywalking.apm.collector.queue.QueueModule` ，实现 [Module](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Module.java) 抽象类，队列 Module 。

[`#name()`](https://github.com/YunaiV/skywalking/blob/8b7205313e60e84d50579261992042c8b581492f/apm-collector/apm-collector-queue/collector-queue-define/src/main/java/org/skywalking/apm/collector/queue/QueueModule.java#L33) **实现**方法，返回模块名为 `"queue"` 。

[`#services()`](https://github.com/YunaiV/skywalking/blob/8b7205313e60e84d50579261992042c8b581492f/apm-collector/apm-collector-queue/collector-queue-define/src/main/java/org/skywalking/apm/collector/queue/QueueModule.java#L37) **实现**方法，返回 Service 类名：QueueCreatorService 。

## 2.2 QueueCreatorService

`org.skywalking.apm.collector.queue.service.QueueCreatorService` ，继承 [Service](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Service.java) 接口，队列创建服务**接口**。

[`#create(queueSize, executor)`](https://github.com/YunaiV/skywalking/blob/8b7205313e60e84d50579261992042c8b581492f/apm-collector/apm-collector-queue/collector-queue-define/src/main/java/org/skywalking/apm/collector/queue/service/QueueCreatorService.java#L39) **接口**方法，创建队列处理器。

* 一般情况下，实现该接口方法，调用 [`org.skywalking.apm.collector.queue.base.QueueCreator#create(queueSize, executor)`](https://github.com/YunaiV/skywalking/blob/8b7205313e60e84d50579261992042c8b581492f/apm-collector/apm-collector-queue/collector-queue-define/src/main/java/org/skywalking/apm/collector/queue/base/QueueCreator.java#L35) 方法，创建队列处理器。

## 2.3 MessageHolder

`org.skywalking.apm.collector.queue.base.MessageHolder` ，消息持有者。

* [`message`](https://github.com/YunaiV/skywalking/blob/8b7205313e60e84d50579261992042c8b581492f/apm-collector/apm-collector-queue/collector-queue-define/src/main/java/org/skywalking/apm/collector/queue/base/MessageHolder.java#L33) 属性，持有的消息。
* [`#reset()`](https://github.com/YunaiV/skywalking/blob/8b7205313e60e84d50579261992042c8b581492f/apm-collector/apm-collector-queue/collector-queue-define/src/main/java/org/skywalking/apm/collector/queue/base/MessageHolder.java#L46) 方法，**清空**消息。为什么会有这个方法，下文胖友会看到。

## 2.4 QueueEventHandler

[`org.skywalking.apm.collector.queue.base.QueueEventHandler`](https://github.com/YunaiV/skywalking/blob/8b7205313e60e84d50579261992042c8b581492f/apm-collector/apm-collector-queue/collector-queue-define/src/main/java/org/skywalking/apm/collector/queue/base/QueueEventHandler.java)，队列处理器**接口**。它定义了 [`#tell(message)`](https://github.com/YunaiV/skywalking/blob/8b7205313e60e84d50579261992042c8b581492f/apm-collector/apm-collector-queue/collector-queue-define/src/main/java/org/skywalking/apm/collector/queue/base/QueueEventHandler.java#L33) **接口**方法，输入消息给自己。最终，QueueEventHandler 会"**提交**"消息给 [`org.skywalking.apm.collector.queue.base.QueueExecutor`](https://github.com/YunaiV/skywalking/blob/8b7205313e60e84d50579261992042c8b581492f/apm-collector/apm-collector-queue/collector-queue-define/src/main/java/org/skywalking/apm/collector/queue/base/QueueExecutor.java#L28)，执行处理该消息。

LocalAsyncWorkerRef 实现 QueueEventHandler 接口，在 [《SkyWalking 源码分析 —— Collector Streaming Computing 流式处理（一）》「3.1.2 LocalAsyncWorkerRef」](http://www.iocoder.cn/SkyWalking/collector-streaming-first/?self) 有详细解析。

## 2.5 DaemonThreadFactory

[`org.skywalking.apm.collector.queue.base.DaemonThreadFactory`](https://github.com/YunaiV/skywalking/blob/8b7205313e60e84d50579261992042c8b581492f/apm-collector/apm-collector-queue/collector-queue-define/src/main/java/org/skywalking/apm/collector/queue/base/DaemonThreadFactory.java)，守护进程线程**工厂**，被用于创建消息处理器的线程。

# 3. collector-queue-disruptor-provider

`collector-queue-disruptor-provider` ，基于 [Disruptor](https://github.com/LMAX-Exchange/disruptor) 的队列组件实现。

项目结构如下 ：![](http://www.iocoder.cn/images/SkyWalking/2020_08_15/04.png)

**默认配置**，在 [`application-default.yml`](https://github.com/YunaiV/skywalking/blob/8b7205313e60e84d50579261992042c8b581492f/apm-collector/apm-collector-core/src/main/resources/application-default.yml#L7) **已经**配置如下：

``` JSON
queue:
  disruptor:
```

## 3.1 QueueModuleDisruptorProvider

`org.skywalking.apm.collector.queue.disruptor.CQueueModuleDisruptorProvider` ，实现 [ModuleProvider](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/ModuleProvider.java) **抽象类**，基于 Disruptor 的队列服务提供者。

[`#name()`](https://github.com/YunaiV/skywalking/blob/8b7205313e60e84d50579261992042c8b581492f/apm-collector/apm-collector-queue/collector-queue-disruptor-provider/src/main/java/org/skywalking/apm/collector/queue/disruptor/QueueModuleDisruptorProvider.java#L35) **实现**方法，返回组件服务提供者名为 `"disruptor"` 。

[`module()`](https://github.com/YunaiV/skywalking/blob/8b7205313e60e84d50579261992042c8b581492f/apm-collector/apm-collector-queue/collector-queue-disruptor-provider/src/main/java/org/skywalking/apm/collector/queue/disruptor/QueueModuleDisruptorProvider.java#L39) **实现**方法，返回组件类为 QueueModule 。

[`#requiredModules()`](https://github.com/YunaiV/skywalking/blob/8b7205313e60e84d50579261992042c8b581492f/apm-collector/apm-collector-queue/collector-queue-disruptor-provider/src/main/java/org/skywalking/apm/collector/queue/disruptor/QueueModuleDisruptorProvider.java#L55) **实现**方法，返回依赖组件为空。

-------

[`#prepare(Properties)`](https://github.com/YunaiV/skywalking/blob/8b7205313e60e84d50579261992042c8b581492f/apm-collector/apm-collector-queue/collector-queue-disruptor-provider/src/main/java/org/skywalking/apm/collector/queue/disruptor/QueueModuleDisruptorProvider.java#L43) **实现**方法，执行准备阶段逻辑。

* 第 44 行 ：创建 DisruptorQueueCreatorService 对象，并调用 `#registerServiceImplementation()` **父类**方法，注册到 `services` 。

[`#start()`](https://github.com/YunaiV/skywalking/blob/8b7205313e60e84d50579261992042c8b581492f/apm-collector/apm-collector-queue/collector-queue-disruptor-provider/src/main/java/org/skywalking/apm/collector/queue/disruptor/QueueModuleDisruptorProvider.java#L47) **实现**方法，方法为空。

[`#notifyAfterCompleted()`](https://github.com/YunaiV/skywalking/blob/8b7205313e60e84d50579261992042c8b581492f/apm-collector/apm-collector-queue/collector-queue-disruptor-provider/src/main/java/org/skywalking/apm/collector/queue/disruptor/QueueModuleDisruptorProvider.java#L51) **实现**方法，方法为空。

## 2.2 DisruptorQueueCreatorService

`org.skywalking.apm.collector.queue.disruptor.service.DisruptorQueueCreatorService` ，实现 QueueCreatorService **接口**，基于 Disruptor 的队列创建服务**实现类**。

[`#create(queueSize, executor)`](https://github.com/YunaiV/skywalking/blob/8b7205313e60e84d50579261992042c8b581492f/apm-collector/apm-collector-queue/collector-queue-disruptor-provider/src/main/java/org/skywalking/apm/collector/queue/disruptor/service/DisruptorQueueCreatorService.java#L42) **实现**方法，调用 `DisruptorQueueCreator#register(queueSize, executor)` 方法，创建队列处理器。

### 3.2.1 DisruptorQueueCreator

> 友情提示：如果胖友对 Disruptor 暂时不了解，建议先使用 Disruptor 写个小 Demo 。
>
> 如下是笔者阅读的文章：  
>
> * [《三步创建Disruptor应用》](http://colobu.com/2014/08/01/3-steps-to-create-a-disruptor-application/)
> * [《Disruptor入门》](http://ifeve.com/disruptor-getting-started/)
> * [《剖析Disruptor:为什么会这么快？（一）Ringbuffer的特别之处》](http://ifeve.com/dissecting-disruptor-whats-so-special/)

`org.skywalking.apm.collector.queue.disruptor.base.DisruptorQueueCreator` ，实现 QueueCreator **接口**，基于 Disruptor 的队列创建器**实现类**。

[`#create(queueSize, executor)`](https://github.com/YunaiV/skywalking/blob/8b7205313e60e84d50579261992042c8b581492f/apm-collector/apm-collector-queue/collector-queue-disruptor-provider/src/main/java/org/skywalking/apm/collector/queue/disruptor/base/DisruptorQueueCreator.java#L41) **实现**方法，代码如下：

* 第 42 至 45 行：**校验**队列大小为 2 的指数，否则创建 Disruptor 对象会报 `"bufferSize must be a power of 2"` 的异常，参见 [AbstractSequencer](https://github.com/LMAX-Exchange/disruptor/blob/master/src/main/java/com/lmax/disruptor/AbstractSequencer.java#L50) 的代码。
* 第 49 行：**创建** Disruptor 对象。
    * [`org.skywalking.apm.collector.queue.disruptor.base.MessageHolderFactory`](https://github.com/YunaiV/skywalking/blob/8b7205313e60e84d50579261992042c8b581492f/apm-collector/apm-collector-queue/collector-queue-disruptor-provider/src/main/java/org/skywalking/apm/collector/queue/disruptor/base/MessageHolderFactory.java#L29) ，MessageHolder **工厂**。
* 第 51 至 64 行：设置 Disruptor 对象的**默认异常处理器**。
* 第 67 至 70 行：创建 DisruptorEventHandler 对象，并设置为 Disruptor 对象的**事件处理器**。
* 第 74 行：**启动** Disruptor 对象。

**为什么 Disruptor 要求队列大小为 2 的指数呢**？如下是相关资料，感兴趣的同学可以看看( 可跳过 )：

* FROM [《环形缓冲器》](https://zh.wikipedia.org/wiki/%E7%92%B0%E5%BD%A2%E7%B7%A9%E8%A1%9D%E5%8D%80#Linux.E5.86.85.E6.A0.B8.E7.9A.84kfifo)

> ![](http://www.iocoder.cn/images/SkyWalking/2020_08_15/05.png)

* `SingleProducerSequencer#hasAvailableCapacity(requiredCapacity)` 方法，代码如下：![](http://www.iocoder.cn/images/SkyWalking/2020_08_15/06.png)

## 3.3 DisruptorEventHandler

`org.skywalking.apm.collector.queue.disruptor.base.DisruptorEventHandler` ，基于 Disruptor 的队列处理器**实现类**。

* [`ringBuffer`](https://github.com/YunaiV/skywalking/blob/0feb6bce291fa945eff5bf52100c82c960851143/apm-collector/apm-collector-queue/collector-queue-disruptor-provider/src/main/java/org/skywalking/apm/collector/queue/disruptor/base/DisruptorEventHandler.java#L43) 属性，Disruptor RingBuffer 对象。
* [`executor`](https://github.com/YunaiV/skywalking/blob/0feb6bce291fa945eff5bf52100c82c960851143/apm-collector/apm-collector-queue/c方法参数ollector-queue-disruptor-provider/src/main/java/org/skywalking/apm/collector/queue/disruptor/base/DisruptorEventHandler.java#L47) 属性，执行器。
* 实现 `org.skywalking.apm.collector.queue.base.QueueEventHandler` **接口** 的 [`#tell(message)`](https://github.com/YunaiV/skywalking/blob/0feb6bce291fa945eff5bf52100c82c960851143/apm-collector/apm-collector-queue/collector-queue-disruptor-provider/src/main/java/org/skywalking/apm/collector/queue/disruptor/base/DisruptorEventHandler.java#L80) 接口方法，标准的 Disruptor 发布事件的代码。
* 实现 [`com.lmax.disruptor.EventHandler`](https://github.com/LMAX-Exchange/disruptor/blob/master/src/main/java/com/lmax/disruptor/EventHandler.java) 的 [`#onEvent(event, sequence, endOfBatch)`](https://github.com/YunaiV/skywalking/blob/0feb6bce291fa945eff5bf52100c82c960851143/apm-collector/apm-collector-queue/collector-queue-disruptor-provider/src/main/java/org/skywalking/apm/collector/queue/disruptor/base/DisruptorEventHandler.java#L62) 接口方法，代码如下：
    * `endOfBatch` **方法参数**，标记该事件( 消息 )是否是 Disruptor **每次批处理**的最后一个事件。胖友可以参见 [《LMAX Disruptor - what determines the batch size?》](https://stackoverflow.com/questions/33716825/lmax-disruptor-what-determines-the-batch-size) 这篇文章，自己搭建一个 Demo 理解下该参数。
    * 第 66 行：调用 `MessageHolder#reset()` 方法，清空消息，因为在 Disruptor RingBuffer 里，事件( 消息 )对象是**重用**的，虽然后续发布事件( 消息 )可以进行**覆盖**，考虑到安全性进行清空。
    * 第 69 行：设置消息为该批量的结尾( 最后一条 )。**为什么**？在 [《SkyWalking 源码分析 —— Collector Streaming Computing 流式处理（二）》「3. AggregationWorker」](http://www.iocoder.cn/SkyWalking/collector-streaming-second/?self) 揭晓答案。
    * 第 72 行：调用 `QueueExecutor#execute(message)` 方法，执行处理消息。

# 4. collector-queue-datacarrier-provider

`collector-queue-datacarrier-provider` ：基于 [apm-datacarrier](https://github.com/YunaiV/skywalking/tree/master/apm-commons/apm-datacarrier) 的队列组件实现。

*目前暂未完成*。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

在地铁里，夹缝中写代码( 😈 当然有座位 )。

不容易，回家看《蜘蛛侠》。

胖友，分享一波朋友圈可好。


