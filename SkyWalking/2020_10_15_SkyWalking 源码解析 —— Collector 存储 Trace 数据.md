title: SkyWalking 源码分析 —— Collector 存储 Trace 数据
date: 2020-10-15
tags:
categories: SkyWalking
permalink: SkyWalking/collector-store-trace

-------

摘要: 原创出处 http://www.iocoder.cn/SkyWalking/collector-store-trace/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 SkyWalking 3.2.6 正式版**

- [1. 概述](http://www.iocoder.cn/SkyWalking/collector-store-trace/)
- [2. SpanListener](http://www.iocoder.cn/SkyWalking/collector-store-trace/)
- [3. GlobalTrace](http://www.iocoder.cn/SkyWalking/collector-store-trace/)
- [4. InstPerformance](http://www.iocoder.cn/SkyWalking/collector-store-trace/)
- [5. SegmentCost](http://www.iocoder.cn/SkyWalking/collector-store-trace/)
- [6. NodeComponent](http://www.iocoder.cn/SkyWalking/collector-store-trace/)
- [7. NodeMapping](http://www.iocoder.cn/SkyWalking/collector-store-trace/)
- [8. NodeReference](http://www.iocoder.cn/SkyWalking/collector-store-trace/)
- [9. ServiceEntry](http://www.iocoder.cn/SkyWalking/collector-store-trace/)
- [10. ServiceReference](http://www.iocoder.cn/SkyWalking/collector-store-trace/)
- [11. Segment](http://www.iocoder.cn/SkyWalking/collector-store-trace/)
- [666. 彩蛋](http://www.iocoder.cn/SkyWalking/collector-store-trace/)

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

分布式链路追踪系统，链路的追踪大体流程如下：

1. Agent 收集 Trace 数据。
2. Agent 发送 Trace 数据给 Collector 。
3. Collector 接收 Trace 数据。
4. **Collector 存储 Trace 数据到存储器，例如，数据库**。

本文主要分享【第四部分】 **SkyWalking Collector 存储 Trace 数据**。

> 友情提示：Collector 接收到 TraceSegment 的数据，对应的类是 Protobuf 生成的。考虑到更加易读易懂，本文使用 TraceSegment 相关的**原始类**。

Collector 在接收到 Trace 数据后，经过**流式处理**，最终**存储**到存储器。如下图，**红圈部分**，为本文分享的内容：![](http://www.iocoder.cn/images/SkyWalking/2020_10_15/01.jpeg)

# 2. SpanListener

在 [《SkyWalking 源码分析 —— Collector 接收 Trace 数据》](http://www.iocoder.cn/SkyWalking/collector-receive-trace/) 一文中，我们看到 [`SegmentParse#parse(UpstreamSegment, Source)`](https://github.com/YunaiV/skywalking/blob/428190e783d887c8240546f321e76e0a6b5f5d18/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/parser/SegmentParse.java#L86) 方法中：

* 在 [`#preBuild(List<UniqueId>, SegmentDecorator)`](https://github.com/YunaiV/skywalking/blob/428190e783d887c8240546f321e76e0a6b5f5d18/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/parser/SegmentParse.java#L118) 方法中，预构建的过程中，使用 Span 监听器们，从 TraceSegment 解析出不同的数据。
* 在**预构建**成功后，通知 Span 监听器们，去构建各自的数据，经过**流式处理**，最终**存储**到存储器。

[`org.skywalking.apm.collector.agent.stream.parser.SpanListener`](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/parser/SpanListener.java) ，Span 监听器**接口**。

* 定义了 [`#build()`](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/parser/SpanListener.java#L28) 方法，构建数据，执行流式处理，最终存储到存储器。

SpanListener 的子类如下图：![](http://www.iocoder.cn/images/SkyWalking/2020_10_15/02.png)

* 第一层，通用接口层，定义了从 TraceSegment 解析数据的方法。
    * ① [GlobalTraceSpanListener](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/parser/GlobalTraceIdsListener.java) ：解析链路追踪全局编号数组( `TraceSegment.relatedGlobalTraces` )。
    * ② [RefsListener](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/parser/RefsListener.java) ：解析父 Segment 指向数组( `TraceSegment.refs` )。
    * ③ [FirstSpanListener](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/parser/FirstSpanListener.java) ：解析**第一个** Span (`TraceSegment.spans[0]`) 。
    * ③ [EntrySpanListener](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/parser/EntrySpanListener.java) ：解析 EntrySpan (`TraceSegment.spans`)。
    * ③ [LocalSpanListener](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/parser/LocalSpanListener.java) ：解析 LocalSpan (`TraceSegment.spans`)。
    * ③ [ExitSpanListener](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/parser/ExitSpanListener.java) ：解析 ExitSpan (`TraceSegment.spans`)。
* 第二层，业务实现层，每个实现类对应一个数据实体类，一个 Graph 对象。如下图所示：![](http://www.iocoder.cn/images/SkyWalking/2020_10_15/03.png)

下面，我们以每个数据实体类为中心，逐个分享。

# 3. GlobalTrace

[`org.skywalking.apm.collector.storage.table.global.GlobalTrace`](https://github.com/YunaiV/skywalking/blob/3d1d1f5219205d38f58f1b59f0e81d81c038d2f1/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/global/GlobalTrace.java) ，全局链路追踪，记录一次分布式链路追踪，包括的 TraceSegment 编号。

* GlobalTrace : TraceSegment = N : M ，一个 GlobalTrace 可以有多个 TraceSegment ，一个 TraceSegment 可以关联多个 GlobalTrace 。参见 [《SkyWalking 源码分析 —— Agent 收集 Trace 数据》「2. Trace」](http://www.iocoder.cn/SkyWalking/agent-collect-trace/?self) 。
* [`org.skywalking.apm.collector.storage.table.global.GlobalTraceTable`](https://github.com/YunaiV/skywalking/blob/3d1d1f5219205d38f58f1b59f0e81d81c038d2f1/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/global/GlobalTraceTable.java) ， GlobalTrace 表( `global_trace` )。字段如下：
    * `global_trace_id` ：全局链路追踪编号。
    * `segment_id` ：TraceSegment 链路编号。
    * `time_bucket` ：时间。
* [`org.skywalking.apm.collector.storage.es.dao.GlobalTraceEsPersistenceDAO`](https://github.com/YunaiV/skywalking/blob/3d1d1f5219205d38f58f1b59f0e81d81c038d2f1/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/GlobalTraceEsPersistenceDAO.java) ，GlobalTrace 的 EsDAO 。
* 在 ES 存储例子如下图： ![](http://www.iocoder.cn/images/SkyWalking/2020_10_15/04.png)

-------

[`org.skywalking.apm.collector.agent.stream.worker.trace.global.GlobalTraceSpanListener`](https://github.com/YunaiV/skywalking/blob/3d1d1f5219205d38f58f1b59f0e81d81c038d2f1/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/global/GlobalTraceSpanListener.java) ，**GlobalTrace 的 SpanListener** ，实现了 FirstSpanListener 、GlobalTraceIdsListener 接口，代码如下：

* `globalTraceIds` 属性，全局链路追踪编号**数组**。
* `segmentId` 属性，TraceSegment 链路编号。
* `timeBucket` 属性，时间。
* [`#parseFirst(SpanDecorator, applicationId, instanceId, segmentId)`](https://github.com/YunaiV/skywalking/blob/3d1d1f5219205d38f58f1b59f0e81d81c038d2f1/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/global/GlobalTraceSpanListener.java#L60) 方法，从 Span 中解析到 `segmentId` ，`timeBucket` 。
* [`#parseGlobalTraceId(UniqueId)`](https://github.com/YunaiV/skywalking/blob/3d1d1f5219205d38f58f1b59f0e81d81c038d2f1/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/global/GlobalTraceSpanListener.java#L66) 方法，解析全局链路追踪编号，添加到 `globalTraceIds` 数组。
* [`#build()`](https://github.com/YunaiV/skywalking/blob/3d1d1f5219205d38f58f1b59f0e81d81c038d2f1/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/global/GlobalTraceSpanListener.java#L80) 方法，构建，代码如下：
    * 第 84 行：获取 GlobalTrace 对应的 `Graph<GlobalTrace>` 对象。
    * 第 86 至 92 行：循环 `globalTraceIds` 数组，创建 GlobalTrace 对象，逐个调用 `Graph#start(application)` 方法，进行流式处理。在这过程中，会保存 GlobalTrace 到存储器。

-------

在 [`TraceStreamGraph#createGlobalTraceGraph()`](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/graph/TraceStreamGraph.java#L93) 方法中，我们可以看到 GlobalTrace 对应的 `Graph<GlobalTrace>` 对象的创建。

* [`org.skywalking.apm.collector.agent.stream.worker.trace.global.GlobalTracePersistenceWorker`](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/global/GlobalTracePersistenceWorker.java) ，继承 PersistenceWorker 抽象类，GlobalTrace **批量**保存 Worker 。

    * [Factory](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/global/GlobalTracePersistenceWorker.java#L51) 内部类，实现 AbstractLocalAsyncWorkerProvider 抽象类，在 [《SkyWalking 源码分析 —— Collector Streaming Computing 流式处理（一）》「3.2.2 AbstractLocalAsyncWorker」](http://www.iocoder.cn/SkyWalking/collector-streaming-first/?self) 有详细解析。
    * **PersistenceWorker** ，在 [《SkyWalking 源码分析 —— Collector Streaming Computing 流式处理（二）》「4. PersistenceWorker」](http://www.iocoder.cn/SkyWalking/collector-streaming-second/?self) 有详细解析。
    * [`#id()`](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/global/GlobalTracePersistenceWorker.java#L39) 实现方法，返回 120 。
    * [`#needMergeDBData()`](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/global/GlobalTracePersistenceWorker.java#L43) 实现方法，返回 `false` ，存储时，不需要合并数据。GlobalTrace 只有新增操作，没有更新操作，因此无需合并数据。

# 4. InstPerformance

> 旁白君：InstPerformance 和 GlobalTrace 整体比较相似，分享的会比较简洁一些。

[`org.skywalking.apm.collector.storage.table.instance.InstPerformance`](https://github.com/YunaiV/skywalking/blob/0f8c38a8a35cbda777fa9a9bc5f51c8651ae4051/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/instance/InstPerformance.java) ，应用实例性能，记录应用实例**每秒**的请求总次数，请求总时长。

* [`org.skywalking.apm.collector.storage.table.instance.InstPerformanceTable`](https://github.com/YunaiV/skywalking/blob/0f8c38a8a35cbda777fa9a9bc5f51c8651ae4051/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/instance/InstPerformanceTable.java) ， GlobalTrace 表( `global_trace` )。字段如下：
    * `application_id` ：应用编号。
    * `instance_id` ：应用实例编号。
    * `calls` ：调用总次数。
    * `cost_total` ：消耗总时长。
    * `time_bucket` ：时间。
* [`org.skywalking.apm.collector.storage.es.dao.InstPerformanceEsPersistenceDAO`](https://github.com/YunaiV/skywalking/blob/3d1d1f5219205d38f58f1b59f0e81d81c038d2f1/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/InstPerformanceEsPersistenceDAO.java) ，InstPerformance 的 EsDAO 。
* 在 ES 存储例子如下图：![](http://www.iocoder.cn/images/SkyWalking/2020_10_15/05.png)

-------

[`org.skywalking.apm.collector.agent.stream.worker.trace.instance.InstPerformanceSpanListener`](https://github.com/YunaiV/skywalking/blob/0f8c38a8a35cbda777fa9a9bc5f51c8651ae4051/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/instance/InstPerformanceSpanListener.java) ，**InstPerformance 的 SpanListener** ，实现了 FirstSpanListener 、EntrySpanListener 接口。

-------

在 [`TraceStreamGraph#createInstPerformanceGraph()`](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/graph/TraceStreamGraph.java#L101) 方法中，我们可以看到 InstPerformance 对应的 `Graph<InstPerformance>` 对象的创建。

* [`org.skywalking.apm.collector.agent.stream.worker.trace.instance.InstPerformancePersistenceWorker`](https://github.com/YunaiV/skywalking/blob/0f8c38a8a35cbda777fa9a9bc5f51c8651ae4051/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/instance/InstPerformancePersistenceWorker.java) ，继承 PersistenceWorker 抽象类，InstPerformance **批量**保存 Worker 。

    * 类似 GlobalTracePersistenceWorker ，... 省略其它类和方法。
    * [`#needMergeDBData()`](https://github.com/YunaiV/skywalking/blob/0f8c38a8a35cbda777fa9a9bc5f51c8651ae4051/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/instance/InstPerformancePersistenceWorker.java#L46) 实现方法，返回 `true` ，存储时，需要合并数据。`calls` 、`cost_total` 需要累加合并。

# 5. SegmentCost

> 旁白君：SegmentCost 和 GlobalTrace 整体比较相似，分享的会比较简洁一些。

[`org.skywalking.apm.collector.storage.table.segment.SegmentCost`](https://github.com/YunaiV/skywalking/blob/b8be916ac2bf7817544d6ff77a95a624bcc3efe6/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/segment/SegmentCost.java) ，TraceSegment 消耗时长，记录 TraceSegment 开始时间，结束时间，花费时长等等。

* SegmentCost : TraceSegment = 1 : 1 。
* [`org.skywalking.apm.collector.storage.table.instance.SegmentCostTable`](https://github.com/YunaiV/skywalking/blob/b8be916ac2bf7817544d6ff77a95a624bcc3efe6/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/segment/SegmentCostTable.java) ， SegmentCostTable 表( `segment_cost` )。字段如下：
    * `segment_id` ：TraceSegment 编号。
    * `application_id` ：应用编号。
    * `start_time` ：开始时间。
    * `end_time` ：结束时间。
    * `service_name` ：操作名。
    * `cost` ：消耗时长。
    * `time_bucket` ：时间( `yyyyMMddHHmm` )。
* [`org.skywalking.apm.collector.storage.es.dao.SegmentCostEsPersistenceDAO`](https://github.com/YunaiV/skywalking/blob/3d1d1f5219205d38f58f1b59f0e81d81c038d2f1/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/SegmentCostEsPersistenceDAO.java) ，SegmentCost 的 EsDAO 。
* 在 ES 存储例子如下图：![](http://www.iocoder.cn/images/SkyWalking/2020_10_15/06.png)

-------

[`org.skywalking.apm.collector.agent.stream.worker.trace.segment.SegmentCostSpanListener`](https://github.com/YunaiV/skywalking/blob/3d1d1f5219205d38f58f1b59f0e81d81c038d2f1/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/segment/SegmentCostSpanListener.java) ，**SegmentCost 的 SpanListener** ，实现了 FirstSpanListener 、EntrySpanListener 、ExitSpanListener 、LocalSpanListener 接口。

-------

在 [`TraceStreamGraph#createSegmentCostGraph()`](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/graph/TraceStreamGraph.java#L172) 方法中，我们可以看到 SegmentCost 对应的 `Graph<SegmentCost>` 对象的创建。

* [`org.skywalking.apm.collector.agent.stream.worker.trace.segment.SegmentCostPersistenceWorker`](https://github.com/YunaiV/skywalking/blob/b8be916ac2bf7817544d6ff77a95a624bcc3efe6/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/segment/SegmentCostPersistenceWorker.java) ，继承 PersistenceWorker 抽象类，InstPerformance **批量**保存 Worker 。

    * 类似 GlobalTracePersistenceWorker ，... 省略其它类和方法。

# 6. NodeComponent

[`org.skywalking.apm.collector.storage.table.node.NodeComponent`](https://github.com/YunaiV/skywalking/blob/297c693e9e91200860a147ca41473f68d48d5955/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/node/NodeComponent.java) ，节点组件。

* [`org.skywalking.apm.collector.storage.table.node.NodeComponentTable`](https://github.com/YunaiV/skywalking/blob/297c693e9e91200860a147ca41473f68d48d5955/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/node/NodeComponentTable.java) ， NodeComponentTable 表( `node_component` )。字段如下：
    * `component_id` ：组件编号，参见 [ComponentsDefine](https://github.com/YunaiV/skywalking/blob/0f8c38a8a35cbda777fa9a9bc5f51c8651ae4051/apm-network/src/main/java/org/skywalking/apm/network/trace/component/ComponentsDefine.java) 的枚举。
    * `peer_id` ：对等编号。每个组件，或是服务提供者，有服务地址；又或是服务消费者，有调用服务地址。这两者都脱离不开**服务地址**。SkyWalking 将**服务地址**作为 `applicationCode` ，注册到 Application 。因此，此处的 `peer_id` 实际上是，**服务地址**对应的应用编号。
    * `time_bucket` ：时间( `yyyyMMddHHmm` )。
* [`org.skywalking.apm.collector.storage.es.dao.NodeComponentEsPersistenceDAO`](https://github.com/YunaiV/skywalking/blob/3d1d1f5219205d38f58f1b59f0e81d81c038d2f1/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/NodeComponentEsPersistenceDAO.java) ，NodeComponent 的 EsDAO 。
* 在 ES 存储例子如下图：![](http://www.iocoder.cn/images/SkyWalking/2020_10_15/07.png)

-------

[`org.skywalking.apm.collector.agent.stream.worker.trace.node.NodeComponentSpanListener`](https://github.com/YunaiV/skywalking/blob/297c693e9e91200860a147ca41473f68d48d5955/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/node/NodeComponentSpanListener.java) ，**NodeComponent 的 SpanListener** ，实现了 FirstSpanListener 、EntrySpanListener 、ExitSpanListener 接口，代码如下：

* `nodeComponents` 属性，节点组件**数组**，一次 TraceSegment 可以经过个节点组件，例如 SpringMVC => MongoDB 。
* `segmentId` 属性，TraceSegment 链路编号。
* `timeBucket` 属性，时间( `yyyyMMddHHmm` )。
* [`#parseEntry(SpanDecorator, applicationId, instanceId, segmentId)`](https://github.com/YunaiV/skywalking/blob/297c693e9e91200860a147ca41473f68d48d5955/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/node/NodeComponentSpanListener.java#L65) 方法，从 EntrySpan 中解析到 `segmentId` ，`applicationId` ，创建 NodeComponent 对象，添加到 `nodeComponents` 。**注意**，EntrySpan 使用 `applicationId` 作为 `peerId` 。
* [`#parseExit(SpanDecorator, applicationId, instanceId, segmentId)`](https://github.com/YunaiV/skywalking/blob/297c693e9e91200860a147ca41473f68d48d5955/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/node/NodeComponentSpanListener.java#L53) 方法，从 ExitSpan 中解析到 `segmentId` ，`peerId` ，创建 NodeComponent 对象，添加到 `nodeComponents` 。**注意**，ExitSpan 使用 `peerId` 作为 `peerId` 。
* [`#parseFirst(SpanDecorator, applicationId, instanceId, segmentId)`](https://github.com/YunaiV/skywalking/blob/297c693e9e91200860a147ca41473f68d48d5955/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/node/NodeComponentSpanListener.java#L78) 方法，从**首个** Span 中解析到 `timeBucket` 。
* [`#build()`](https://github.com/YunaiV/skywalking/blob/297c693e9e91200860a147ca41473f68d48d5955/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/node/NodeComponentSpanListener.java#L83) 方法，构建，代码如下：
    * 第 84 行：获取 NodeComponent 对应的 `Graph<NodeComponent>` 对象。
    * 第 86 至 92 行：循环 `nodeComponents` 数组，逐个调用 `Graph#start(nodeComponent)` 方法，进行流式处理。在这过程中，会保存 NodeComponent 到存储器。

-------

在 [`TraceStreamGraph#createNodeComponentGraph()`](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/graph/TraceStreamGraph.java#L109) 方法中，我们可以看到 NodeComponent 对应的 `Graph<NodeComponent>` 对象的创建。

* [`org.skywalking.apm.collector.agent.stream.worker.trace.node.NodeComponentAggregationWorker`](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/service/ServiceEntryAggregationWorker.java) ，继承 AggregationWorker 抽象类，NodeComponent 聚合 Worker 。
    * NodeComponent 的编号生成规则为 `${timeBucket}_${componentId}_${peerId}` ，并且 `timeBucket` 是**分钟级** ，可以使用 AggregationWorker 进行聚合，合并相同操作，减小 Collector 和 ES 的压力。
    * [Factory](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/service/ServiceEntryAggregationWorker.java#L40) 内部类，实现 AbstractLocalAsyncWorkerProvider 抽象类，在 [《SkyWalking 源码分析 —— Collector Streaming Computing 流式处理（一）》「3.2.2 AbstractLocalAsyncWorker」](http://www.iocoder.cn/SkyWalking/collector-streaming-first/?self) 有详细解析。
    * **AggregationWorker** ，在 [《SkyWalking 源码分析 —— Collector Streaming Computing 流式处理（二）》「3. AggregationWorker」](http://www.iocoder.cn/SkyWalking/collector-streaming-second/?self) 有详细解析。
    * [`#id()`](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/service/ServiceEntryAggregationWorker.java#L36) 实现方法，返回 106 。
* [`org.skywalking.apm.collector.agent.stream.worker.trace.service.ServiceEntryRemoteWorker`](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/service/ServiceEntryRemoteWorker.java) ，继承 AbstractRemoteWorker 抽象类，应用注册远程 Worker 。
    * [Factory](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/service/ServiceEntryRemoteWorker.java#L50) 内部类，实现 AbstractRemoteWorkerProvider 抽象类，在 [《SkyWalking 源码分析 —— Collector Streaming Computing 流式处理（一）》「3.2.3 AbstractRemoteWorker」](http://www.iocoder.cn/SkyWalking/collector-streaming-first/?self) 有详细解析。
    * **AbstractRemoteWorker** ，在 [《SkyWalking 源码分析 —— Collector Streaming Computing 流式处理（一）》「3.2.3 AbstractRemoteWorker」](http://www.iocoder.cn/SkyWalking/collector-streaming-first/?self) 有详细解析。
    * [`#id()`](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/service/ServiceEntryRemoteWorker.java#L38) 实现方法，返回 10002 。
    * [`#selector`](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/service/ServiceEntryRemoteWorker.java#L46) 实现方法，返回 `Selector.HashCode` 。将**相同编号**的 NodeComponent 发给同一个 Collector 节点，统一处理。在 [《SkyWalking 源码分析 —— Collector Remote 远程通信服务》](http://www.iocoder.cn/SkyWalking/collector-remote-module/?self) 有详细解析。
* [`org.skywalking.apm.collector.agent.stream.worker.trace.service.ServiceEntryPersistenceWorker`](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/service/ServiceEntryPersistenceWorker.java) ，继承 PersistenceWorker 抽象类，NodeComponent **批量**保存 Worker 。

    * 类似 GlobalTracePersistenceWorker ，... 省略其它类和方法。
    * [`#needMergeDBData()`](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/service/ServiceEntryPersistenceWorker.java#L43) 实现方法，返回 `true` ，存储时，需要合并数据。

# 7. NodeMapping

[`org.skywalking.apm.collector.storage.table.node.NodeComponent`](https://github.com/YunaiV/skywalking/blob/ee9400fc51d6ae21b1053bb0d8cca7ad4d51efe5/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/node/NodeComponent.java) ，节点匹配，用于匹配服务消费者与提供者。

* [`org.skywalking.apm.collector.storage.table.node.NodeMappingTable`](https://github.com/YunaiV/skywalking/blob/ee9400fc51d6ae21b1053bb0d8cca7ad4d51efe5/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/node/NodeMappingTable.java) ， NodeMappingTable 表( `node_mapping` )。字段如下：
    * `application_id` ：服务消费者应用编号。
    * `address_id` ：服务提供者应用编号。
    * `time_bucket` ：时间( `yyyyMMddHHmm` )。
* [`org.skywalking.apm.collector.storage.es.dao.NodeMappingEsPersistenceDAO`](https://github.com/YunaiV/skywalking/blob/3d1d1f5219205d38f58f1b59f0e81d81c038d2f1/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/NodeMappingEsPersistenceDAO.java) ，NodeMapping 的 EsDAO 。
* 在 ES 存储例子如下图：![](http://www.iocoder.cn/images/SkyWalking/2020_10_15/08.png)

-------

[`org.skywalking.apm.collector.agent.stream.worker.trace.node.NodeMappingSpanListener`](https://github.com/YunaiV/skywalking/blob/ee9400fc51d6ae21b1053bb0d8cca7ad4d51efe5/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/node/NodeMappingSpanListener.java) ，**NodeMapping 的 SpanListener** ，实现了 FirstSpanListener 、RefsListener 接口，代码如下：

* `nodeMappings` 属性，节点匹配**数组**，一次 TraceSegment 可以经过个节点组件，例如调用多次远程服务，或者数据库。
* `timeBucket` 属性，时间( `yyyyMMddHHmm` )。
* [`#parseRef(SpanDecorator, applicationId, instanceId, segmentId)`](https://github.com/YunaiV/skywalking/blob/ee9400fc51d6ae21b1053bb0d8cca7ad4d51efe5/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/node/NodeMappingSpanListener.java#L52) 方法，从 TraceSegmentRef 中解析到 `applicationId` ，`peerId` ，创建 NodeMapping 对象，添加到 `nodeMappings` 。
* [`#parseFirst(SpanDecorator, applicationId, instanceId, segmentId)`](https://github.com/YunaiV/skywalking/blob/ee9400fc51d6ae21b1053bb0d8cca7ad4d51efe5/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/node/NodeMappingSpanListener.java#L66) 方法，从**首个** Span 中解析到`timeBucket` 。
* [`#build()`](https://github.com/YunaiV/skywalking/blob/ee9400fc51d6ae21b1053bb0d8cca7ad4d51efe5/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/node/NodeMappingSpanListener.java#L71) 方法，构建，代码如下：
    * 第 84 行：获取 NodeMapping 对应的 `Graph<NodeMapping>` 对象。
    * 第 86 至 92 行：循环 `nodeMappings` 数组，逐个调用 `Graph#start(nodeMapping)` 方法，进行流式处理。在这过程中，会保存 NodeMapping 到存储器。

-------

在 [`TraceStreamGraph#createNodeMappingGraph()`](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/graph/TraceStreamGraph.java#L120) 方法中，我们可以看到 NodeMapping 对应的 `Graph<NodeMapping>` 对象的创建。

* 和 NodeComponent 的 `Graph<NodeComponent>` 基本一致，胖友自己看下源码。
* [`org.skywalking.apm.collector.agent.stream.worker.trace.node.NodeMappingAggregationWorker`](https://github.com/YunaiV/skywalking/blob/ee9400fc51d6ae21b1053bb0d8cca7ad4d51efe5/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/node/NodeMappingAggregationWorker.java)
* [`org.skywalking.apm.collector.agent.stream.worker.trace.node.NodeMappingRemoteWorker`](https://github.com/YunaiV/skywalking/blob/ee9400fc51d6ae21b1053bb0d8cca7ad4d51efe5/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/node/NodeMappingRemoteWorker.java)
* [`org.skywalking.apm.collector.agent.stream.worker.trace.node.NodeMappingPersistenceWorker`](https://github.com/YunaiV/skywalking/blob/ee9400fc51d6ae21b1053bb0d8cca7ad4d51efe5/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/node/NodeMappingPersistenceWorker.java)

# 8. NodeReference

[`org.skywalking.apm.collector.storage.table.noderef.NodeReference`](https://github.com/YunaiV/skywalking/blob/112bcba1a7543e3c86fcbfb49718f7e4f3f4638f/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/noderef/NodeReference.java) ，节点调用统计，用于记录服务消费者对服务提供者的调用，基于**应用**级别的，以**分钟**为时间最小粒度的聚合统计。

* [`org.skywalking.apm.collector.storage.table.noderef.NodeReference`](https://github.com/YunaiV/skywalking/blob/112bcba1a7543e3c86fcbfb49718f7e4f3f4638f/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/noderef/NodeReferenceTable.java) ， NodeReference 表( `node_reference` )。字段如下：
    * `front_application_id` ：服务消费者应用编号。
    * `behind_application_id` ：服务提供者应用编号。
    * `s1_lte` ：( 0, 1000 ms ] 的调用次数。
    * `s3_lte` ：( 1000, 3000 ms ] 的调用次数。
    * `s5_lte` ：( 3000, 5000ms ] 的调用次数
    * `s5_gt` ：( 5000, +∞ ] 的调用次数。
    * `error` ：发生异常的调用次数。
    * `summary` ：总共的调用次数。
    * `time_bucket` ：时间( `yyyyMMddHHmm` )。
* [`org.skywalking.apm.collector.storage.es.dao.NodeReferenceEsPersistenceDAO`](https://github.com/YunaiV/skywalking/blob/3d1d1f5219205d38f58f1b59f0e81d81c038d2f1/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/NodeReferenceEsPersistenceDAO.java) ，NodeReference 的 EsDAO 。
* 在 ES 存储例子如下图：![](http://www.iocoder.cn/images/SkyWalking/2020_10_15/09.png)

-------

[`org.skywalking.apm.collector.agent.stream.worker.trace.noderef.NodeReferenceSpanListener`](https://github.com/YunaiV/skywalking/blob/112bcba1a7543e3c86fcbfb49718f7e4f3f4638f/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/noderef/NodeReferenceSpanListener.java) ，**NodeReference 的 SpanListener** ，实现了 EntrySpanListener 、ExitSpanListener 、RefsListener 接口，代码如下：

* `references` 属性，**父 TraceSegment** 调用产生的 NodeReference 数组。
* `nodeReferences` 属性，NodeReference 数组，最终会包含 `references` 数组。
* `timeBucket` 属性，时间( `yyyyMMddHHmm` )。
* [`#parseRef(SpanDecorator, applicationId, instanceId, segmentId)`](https://github.com/YunaiV/skywalking/blob/112bcba1a7543e3c86fcbfb49718f7e4f3f4638f/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/noderef/NodeReferenceSpanListener.java#L103) 方法，代码如下：
    * 第 106 至 109 行：使用父 TraceSegment 的应用编号作为服务**消费者**编号，自己的应用编号作为服务**提供者**应用编号，创建 NodeReference 对象。
    * 第 111 行：将 NodeReference 对象，添加到 `references` 。**注意**，是 `references` ，而不是 `nodeReference` 。
* [`#parseEntry(SpanDecorator, applicationId, instanceId, segmentId)`](https://github.com/YunaiV/skywalking/blob/112bcba1a7543e3c86fcbfb49718f7e4f3f4638f/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/noderef/NodeReferenceSpanListener.java#L77) 方法，代码如下：
    * 作为服务提供者，**接受**调用。
    * ------- **父 TraceSegment 存在** --------
    * 第 79 至 85 行：`references` 非空，说明被父 TraceSegment 调用。因此，循环 `references` 数组，设置 `id` ，`timeBucket` 属性( 因为 `timeBucket` 需要从 EntrySpan 中获取，所以 `#parseRef(...)` 的目的，就是临时存储父 TraceSegment 的应用编号到 `references` 中 )。
    * 第 87 行：调用 [`#buildserviceSum(...)`](https://github.com/YunaiV/skywalking/blob/112bcba1a7543e3c86fcbfb49718f7e4f3f4638f/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/noderef/NodeReferenceSpanListener.java#L122) 方法，设置调用次数，然后添加到 `nodeReferences` 中。
    * ------- **父 TraceSegment 不存在** --------
    * 第 91 至 97 行：使用 `USER_ID` 的应用编号( 特殊，代表 "**用户**" )作为服务**消费者**编号，自己的应用编号作为服务**提供者**应用编号，创建 NodeReference 对象。
    * 第 99 行：调用 [`#buildserviceSum(...)`](https://github.com/YunaiV/skywalking/blob/112bcba1a7543e3c86fcbfb49718f7e4f3f4638f/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/noderef/NodeReferenceSpanListener.java#L122) 方法，设置调用次数，然后添加到 `nodeReferences` 中。
* [`#parseExit(SpanDecorator, applicationId, instanceId, segmentId)`](https://github.com/YunaiV/skywalking/blob/112bcba1a7543e3c86fcbfb49718f7e4f3f4638f/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/noderef/NodeReferenceSpanListener.java#L62) 方法，代码如下：
    * 作为服务消费者，**发起**调用。  
    * 第 64 至 71 行：使用自己的应用编号作为服务**消费者**编号，`peerId` 作为服务**提供者**应用编号，创建 NodeReference 对象。
    * 第 73 行：调用 [`#buildserviceSum(...)`](https://github.com/YunaiV/skywalking/blob/112bcba1a7543e3c86fcbfb49718f7e4f3f4638f/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/noderef/NodeReferenceSpanListener.java#L122) 方法，设置调用次数，然后添加到 `nodeReferences` 中。
* [`#build()`](https://github.com/YunaiV/skywalking/blob/112bcba1a7543e3c86fcbfb49718f7e4f3f4638f/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/noderef/NodeReferenceSpanListener.java#L114) 方法，构建，代码如下：
    * 第 84 行：获取 NodeReference 对应的 `Graph<NodeReference>` 对象。
    * 第 86 至 92 行：循环 `nodeReferences` 数组，逐个调用 `Graph#start(nodeReference)` 方法，进行流式处理。在这过程中，会保存 NodeReference 到存储器。

-------

在 [`TraceStreamGraph#createNodeReferenceGraph()`](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/graph/TraceStreamGraph.java#L131) 方法中，我们可以看到 NodeReference 对应的 `Graph<NodeReference>` 对象的创建。

* 和 NodeComponent 的 `Graph<NodeComponent>` 基本一致，胖友自己看下源码。
* [`org.skywalking.apm.collector.agent.stream.worker.trace.noderef.NodeReferenceAggregationWorker`](https://github.com/YunaiV/skywalking/blob/ee9400fc51d6ae21b1053bb0d8cca7ad4d51efe5/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/noderef/NodeReferenceAggregationWorker.java)
* [`org.skywalking.apm.collector.agent.stream.worker.trace.noderef.NodeReferenceRemoteWorker`](https://github.com/YunaiV/skywalking/blob/ee9400fc51d6ae21b1053bb0d8cca7ad4d51efe5/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/noderef/NodeReferenceRemoteWorker.java)
* [`org.skywalking.apm.collector.agent.stream.worker.trace.noderef.NodeReferencePersistenceWorker`](https://github.com/YunaiV/skywalking/blob/ee9400fc51d6ae21b1053bb0d8cca7ad4d51efe5/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/noderef/NodeReferencePersistenceWorker.java)

# 9. ServiceEntry

[`org.skywalking.apm.collector.storage.table.service.ServiceEntry`](https://github.com/YunaiV/skywalking/blob/d8e7d053381e7317d413188c4248be1dacf4e85a/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/service/ServiceEntry.java) ，入口操作。

* ServiceEntry **只保存分布式链路的入口操作**，不同于 ServiceName **保存所有操作**，即 ServiceEntry 是 ServiceName 的**子集**。
    * **注意，子 TraceSegment 的入口操作也不记录**。
* [`org.skywalking.apm.collector.storage.table.service.ServiceEntryTable`](https://github.com/YunaiV/skywalking/blob/d8e7d053381e7317d413188c4248be1dacf4e85a/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/service/ServiceEntryTable.java) ， ServiceEntry 表( `service_entry` )。字段如下：
    * `application_id` ：应用编号。
    * `entry_service_id` ：入口操作编号。
    * `entry_service_name` ：入口操作名。
    * `register_time` ：注册时间。
    * `newest_time` ：最后调用时间。
* [`org.skywalking.apm.collector.storage.es.dao.ServiceEntryEsPersistenceDAO`](https://github.com/YunaiV/skywalking/blob/3d1d1f5219205d38f58f1b59f0e81d81c038d2f1/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/ServiceEntryEsPersistenceDAO.java) ，ServiceEntry 的 EsDAO 。
* 在 ES 存储例子如下图：![](http://www.iocoder.cn/images/SkyWalking/2020_10_15/10.png)

-------

[`org.skywalking.apm.collector.agent.stream.worker.trace.service.ServiceEntrySpanListener`](https://github.com/YunaiV/skywalking/blob/112bcba1a7543e3c86fcbfb49718f7e4f3f4638f/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/service/ServiceEntrySpanListener.java) ，**ServiceEntry 的 SpanListener** ，实现了 EntrySpanListener 、FirstSpanListener 、RefsListener 接口，代码如下：

* `hasReference` 属性， 是否有 TraceSegmentRef 。
* `applicationId` 属性，应用编号。
* `entryServiceId` 属性，入口操作编号。
* `entryServiceName` 属性，入口操作名。
* `hasEntry` 属性，是否有 EntrySpan 。
* `timeBucket` 属性，时间( `yyyyMMddHHmm` )。
* [`#parseRef(SpanDecorator, applicationId, instanceId, segmentId)`](https://github.com/YunaiV/skywalking/blob/d8e7d053381e7317d413188c4248be1dacf4e85a/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/service/ServiceEntrySpanListener.java#L75) 方法，是否有 TraceSegmentRef 。
* [`#parseFirst(SpanDecorator, applicationId, instanceId, segmentId)`](https://github.com/YunaiV/skywalking/blob/d8e7d053381e7317d413188c4248be1dacf4e85a/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/service/ServiceEntrySpanListener.java#L89) 方法，从**首个** Span 中解析到 `timeBucket` 。
* [`#parseEntry(SpanDecorator, applicationId, instanceId, segmentId)`](https://github.com/YunaiV/skywalking/blob/d8e7d053381e7317d413188c4248be1dacf4e85a/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/service/ServiceEntrySpanListener.java#L75) 方法，从 EntrySpan 中解析到 `applicationId` 、`entryServiceId` 、`entryServiceName` 、`hasEntry` 。
* [`#build()`](https://github.com/YunaiV/skywalking/blob/d8e7d053381e7317d413188c4248be1dacf4e85a/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/service/ServiceEntrySpanListener.java#L94) 方法，构建，代码如下：
    * 第 96 行：只保存分布式链路的入口操作。
    * 第 98 至 103 行：创建 ServiceEntry 对象。
    * 第 107 行：获取 ServiceEntry 对应的 `Graph<ServiceEntry>` 对象。
    * 第 108 行：调用 `Graph#start(serviceEntry)` 方法，进行流式处理。在这过程中，会保存 ServiceEntry 到存储器。

-------

在 [`TraceStreamGraph#createServiceEntryGraph()`](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/graph/TraceStreamGraph.java#L142) 方法中，我们可以看到 ServiceEntry 对应的 `Graph<ServiceEntry>` 对象的创建。

* 和 NodeComponent 的 `Graph<NodeComponent>` 基本一致，胖友自己看下源码。
* [`org.skywalking.apm.collector.agent.stream.worker.trace.service.ServiceEntryAggregationWorker`](https://github.com/YunaiV/skywalking/blob/ee9400fc51d6ae21b1053bb0d8cca7ad4d51efe5/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/service/ServiceEntryAggregationWorker.java)
* [`org.skywalking.apm.collector.agent.stream.worker.trace.service.ServiceEntryRemoteWorker`](https://github.com/YunaiV/skywalking/blob/ee9400fc51d6ae21b1053bb0d8cca7ad4d51efe5/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/service/ServiceEntryRemoteWorker.java)
* [`org.skywalking.apm.collector.agent.stream.worker.trace.service.ServiceEntryPersistenceWorker`](https://github.com/YunaiV/skywalking/blob/ee9400fc51d6ae21b1053bb0d8cca7ad4d51efe5/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/service/ServiceEntryPersistenceWorker.java)

# 10. ServiceReference

[`org.skywalking.apm.collector.storage.table.serviceref.ServiceReference`](https://github.com/YunaiV/skywalking/blob/288d70975ed1c5f1ecfb7d51e2233ec75ad8d12a/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/serviceref/ServiceReference.java) ，入口操作调用统计，用于记录入口操作的调用，基于**入口操作**级别的，以**分钟**为时间最小粒度的聚合统计。

* 和 NodeReference 类似。
* **注意**，此处的 "**入口操作**" 不同于 ServiceEntry ，**包含**每一条 TraceSegment 的入口操作。
* [`org.skywalking.apm.collector.storage.table.serviceref.ServiceReferenceTable`](https://github.com/YunaiV/skywalking/blob/288d70975ed1c5f1ecfb7d51e2233ec75ad8d12a/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/serviceref/ServiceReferenceTable.java) ， ServiceReference 表( `service_reference` )。字段如下：
    * `entry_service_id` ：入口操作编号。
    * `front_service_id` ：服务消费者操作编号。
    * `behind_service_id` ：服务提供者操作编号。
    * `s1_lte` ：( 0, 1000 ms ] 的调用次数。
    * `s3_lte` ：( 1000, 3000 ms ] 的调用次数。
    * `s5_lte` ：( 3000, 5000ms ] 的调用次数
    * `s5_gt` ：( 5000, +∞ ] 的调用次数。
    * `error` ：发生异常的调用次数。
    * `summary` ：总共的调用次数。
    * `cost_summary` ：总共的花费时间。
    * `time_bucket` ：时间( `yyyyMMddHHmm` )。
* [`org.skywalking.apm.collector.storage.es.dao.ServiceReference`](https://github.com/YunaiV/skywalking/blob/3d1d1f5219205d38f58f1b59f0e81d81c038d2f1/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/ServiceReference.java) ，ServiceReference 的 EsDAO 。
* 在 ES 存储例子如下图：![](http://www.iocoder.cn/images/SkyWalking/2020_10_15/11.png)

-------

[`org.skywalking.apm.collector.agent.stream.worker.trace.segment.ServiceReferenceSpanListener`](https://github.com/YunaiV/skywalking/blob/33a9634fff31a299b0baab5ffba603a58d6ff371/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/serviceref/ServiceReferenceSpanListener.java) ，**ServiceReference 的 SpanListener** ，实现了 EntrySpanListener 、FirstSpanListener 、RefsListener 接口，代码如下：

* `referenceServices` 属性，ReferenceDecorator 数组，记录 TraceSegmentRef 数组。
* `serviceId` 属性，入口操作编号。
* `startTime` 属性，开始时间。
* `endTime` 属性，结束时间。
* `isError` 属性，是否有错误。
* `hasEntry` 属性，是否有 SpanEntry 。
* `timeBucket` 属性，时间( `yyyyMMddHHmm` )。
* [`#parseRef(SpanDecorator, applicationId, instanceId, segmentId)`](https://github.com/YunaiV/skywalking/blob/33a9634fff31a299b0baab5ffba603a58d6ff371/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/serviceref/ServiceReferenceSpanListener.java#L79) 方法，将 TraceSegmentRef 添加到 `referenceServices` 。
* [`#parseFirst(SpanDecorator, applicationId, instanceId, segmentId)`](https://github.com/YunaiV/skywalking/blob/33a9634fff31a299b0baab5ffba603a58d6ff371/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/serviceref/ServiceReferenceSpanListener.java#L74) 方法，从**首个** Span 中解析到 `timeBucket` 。
* [`#parseEntry(SpanDecorator, applicationId, instanceId, segmentId)`](https://github.com/YunaiV/skywalking/blob/33a9634fff31a299b0baab5ffba603a58d6ff371/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/serviceref/ServiceReferenceSpanListener.java#L85) 方法，从 EntrySpan 中解析 `serviceId` 、`startTime` 、`endTime` 、`isError` 、`hasEntry` 。
* [`#build()`](https://github.com/YunaiV/skywalking/blob/33a9634fff31a299b0baab5ffba603a58d6ff371/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/serviceref/ServiceReferenceSpanListener.java#L112) 方法，构建，代码如下：
    * 第 114 行：判断 `hasEntry = true` ，存在 EntrySpan 。
    * --------- **有 TraceSegmentRef** ---------
    * 第 117 至 120 行：创建 ServiceReference 对象，其中：
        * `entryServiceId` ：TraceSegmentRef 的入口编号。
        * `frontServiceId` ：TraceSegmentRef 的操作编号。
        * `behindServiceId` ： 自己 EntrySpan 的操作编号。
    * 第 121 行：调用 [`#calculateCost(...)`](https://github.com/YunaiV/skywalking/blob/33a9634fff31a299b0baab5ffba603a58d6ff371/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/serviceref/ServiceReferenceSpanListener.java#L94) 方法，设置调用次数。
    * 第 126 行：调用 [`#sendToAggregationWorker(...)`](https://github.com/YunaiV/skywalking/blob/33a9634fff31a299b0baab5ffba603a58d6ff371/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/serviceref/ServiceReferenceSpanListener.java#L141) 方法，发送 ServiceReference 给 AggregationWorker ，执行流式处理。
    * --------- **无 TraceSegmentRef** ---------
    * 第 117 至 120 行：创建 ServiceReference 对象，其中：
        * `entryServiceId` ：自己 EntrySpan 的操作编号。
        * `frontServiceId` ：`Const.NONE_SERVICE_ID` 对应的操作编号( 系统内置，代表【空】 )。
        * `behindServiceId` ： 自己 EntrySpan 的操作编号。
    * 第 121 行：调用 [`#calculateCost(...)`](https://github.com/YunaiV/skywalking/blob/33a9634fff31a299b0baab5ffba603a58d6ff371/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/serviceref/ServiceReferenceSpanListener.java#L94) 方法，设置调用次数。
    * 第 126 行：调用 [`#sendToAggregationWorker(...)`](https://github.com/YunaiV/skywalking/blob/33a9634fff31a299b0baab5ffba603a58d6ff371/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/serviceref/ServiceReferenceSpanListener.java#L141) 方法，发送 ServiceReference 给 AggregationWorker ，执行流式处理。

-------

在 [`TraceStreamGraph#createServiceReferenceGraph()`](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/graph/TraceStreamGraph.java#L153) 方法中，我们可以看到 ServiceReference 对应的 `Graph<ServiceReference>` 对象的创建。

* 和 NodeComponent 的 `Graph<NodeComponent>` 基本一致，胖友自己看下源码。
* [`org.skywalking.apm.collector.agent.stream.worker.trace.noderef.ServiceEntryAggregationWorker`](https://github.com/YunaiV/skywalking/blob/ee9400fc51d6ae21b1053bb0d8cca7ad4d51efe5/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/serviceref/ServiceReferenceAggregationWorker.java)
* [`org.skywalking.apm.collector.agent.stream.worker.trace.noderef.ServiceEntryRemoteWorker`](https://github.com/YunaiV/skywalking/blob/ee9400fc51d6ae21b1053bb0d8cca7ad4d51efe5/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/serviceref/ServiceReferenceRemoteWorker.java)
* [`org.skywalking.apm.collector.agent.stream.worker.trace.noderef.ServiceEntryPersistenceWorker`](https://github.com/YunaiV/skywalking/blob/ee9400fc51d6ae21b1053bb0d8cca7ad4d51efe5/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/serviceref/ServiceReferencePersistenceWorker.java)

# 11. Segment

不同于上述所有数据实体，Segment 无需**解析**，直接使用 TraceSegment 构建，参见如下方法：

* [`SegmentParse#parse(UpstreamSegment, Source)`](https://github.com/YunaiV/skywalking/blob/0f8c38a8a35cbda777fa9a9bc5f51c8651ae4051/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/parser/SegmentParse.java#L109)
* [`SegmentParse#buildSegment(id, dataBinary)`](https://github.com/YunaiV/skywalking/blob/0f8c38a8a35cbda777fa9a9bc5f51c8651ae4051/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/parser/SegmentParse.java#L177)

[`org.skywalking.apm.collector.storage.table.segment.Segment`](https://github.com/YunaiV/skywalking/blob/3d1d1f5219205d38f58f1b59f0e81d81c038d2f1/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/global/GlobalTrace.java) ，全局链路追踪，记录一次分布式链路追踪，包括的 TraceSegment 编号。

* [`org.skywalking.apm.collector.storage.table.global.GlobalTraceTable`](https://github.com/YunaiV/skywalking/blob/25a830d4fed13ee2d62adb7ad7b35ada436bd6f6/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/segment/Segment.java) ， Segment 表( `segment` )。字段如下：
    * `_id` ：TraceSegment 编号。
    * `data_binary` ：TraceSegment 链路编号。
    * `time_bucket` ：时间( `yyyyMMddHHmm` )。
* [`org.skywalking.apm.collector.storage.es.dao.SegmentEsPersistenceDAO`](https://github.com/YunaiV/skywalking/blob/3d1d1f5219205d38f58f1b59f0e81d81c038d2f1/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/SegmentEsPersistenceDAO.java) ，GlobalTrace 的 EsDAO 。
* 在 ES 存储例子如下图： ![](http://www.iocoder.cn/images/SkyWalking/2020_10_15/12.png)

-------

在 [`TraceStreamGraph#createSegmentGraph()`](https://github.com/YunaiV/skywalking/blob/52e41bc200857d2eb1b285d046cb9d2dd646fb7b/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/graph/TraceStreamGraph.java#L164) 方法中，我们可以看到 Segment 对应的 `Graph<Segment>` 对象的创建。

* [`org.skywalking.apm.collector.agent.stream.worker.trace.segment.SegmentPersistenceWorker`](https://github.com/YunaiV/skywalking/blob/ee9400fc51d6ae21b1053bb0d8cca7ad4d51efe5/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/trace/segment/SegmentPersistenceWorker.java)

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

抱歉，本文可能会存在一些错误或者细节没有扣到，烦请见谅。  
胖友如果有疑惑，请给我的公众号留言，一起探讨。

大量类似的内容，笔者一天都处于昏昏沉沉的状态，中间有一块不小心替换错误，实在是苦闷而又几分枯燥，不得不佩服 SkyWalking 开发者的耐心。

胖友，分享个朋友圈可好？


