title: SkyWalking 源码分析 —— Collector Streaming Computing 流式处理（二）
date: 2020-09-01
tags:
categories: SkyWalking
permalink: SkyWalking/collector-streaming-second

-------

摘要: 原创出处 http://www.iocoder.cn/SkyWalking/collector-streaming-second/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 SkyWalking 3.2.6 正式版**

- [1. 概述](http://www.iocoder.cn/SkyWalking/collector-streaming-second/)
- [2. Data](http://www.iocoder.cn/SkyWalking/collector-streaming-second/)
  - [2.1 Collection](http://www.iocoder.cn/SkyWalking/collector-streaming-second/)
  - [2.2 DataCollection](http://www.iocoder.cn/SkyWalking/collector-streaming-second/)
  - [2.3 Window](http://www.iocoder.cn/SkyWalking/collector-streaming-second/)
  - [2.4 DataCache](http://www.iocoder.cn/SkyWalking/collector-streaming-second/)
- [3. AggregationWorker](http://www.iocoder.cn/SkyWalking/collector-streaming-second/)
- [4. PersistenceWorker](http://www.iocoder.cn/SkyWalking/collector-streaming-second/)
  - [4.1 WorkerCreateListener](http://www.iocoder.cn/SkyWalking/collector-streaming-second/)
  - [4.2 PersistenceTimer](http://www.iocoder.cn/SkyWalking/collector-streaming-second/)
- [666. 彩蛋](http://www.iocoder.cn/SkyWalking/collector-streaming-second/)

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

本文接 [《SkyWalking 源码分析 —— Collector Streaming Computing 流式处理（一）》](http://www.iocoder.cn/SkyWalking/collector-streaming-first/?self) ，主要分享 **Collector Streaming 流式处理的第二部分**。主要包含如下部分：![](http://www.iocoder.cn/images/SkyWalking/2020_09_01/01.png)

* AggregationWorker ：聚合处理数据，后提交 Data 到 Next 节点们处理。
* PersistenceWorker ：聚合处理数据，后存储 Data 。

# 2. Data

AggregationWorker 和 PersistenceWorker ，都先**聚合**处理数据，在进行各自的后续处理。那么聚合处理的数据结果，需要有**容器**进行缓存暂存：

* `org.skywalking.apm.collector.core.cache` ：接口
* `org.skywalking.apm.collector.stream.worker.impl.data` ：实现

类图如下：![](http://www.iocoder.cn/images/SkyWalking/2020_09_01/02.png)

* Collection ：数据采集，提供有读、写**两个状态**的数据容器。
* Window ：窗口( 😈这个解释怪怪的 )，内有**两个 Collection**。
    * 一个 Collection ，负责写入**数据**数据
    * 一个 Collection ，负责读出**处理**数据
    * 当写的 Collection **符合处理的条件**，读写 Collection 切换

## 2.1 Collection

[`org.skywalking.apm.collector.core.cache.Collection`](https://github.com/YunaiV/skywalking/blob/d6019b49d686f79d07528a19b09d4f0ccc4eca9b/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/cache/Collection.java) ，数据采集**接口**。

* 数据相关 ：`#collection()` / `#size()` / `#clear()`
* 读相关 ：`#reading()` / `#isReading()` / `#finishReading()`
* 写相关 ：`#writing()` / `#isWriting()` / `#finishWriting()`

## 2.2 DataCollection

[`org.skywalking.apm.collector.stream.worker.impl.data.DataCollection`](https://github.com/YunaiV/skywalking/blob/d6019b49d686f79d07528a19b09d4f0ccc4eca9b/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/data/DataCollection.java) ，实现 Collection 接口，数据采集实现类，使用 [`Map<String, Data>`](https://github.com/YunaiV/skywalking/blob/d6019b49d686f79d07528a19b09d4f0ccc4eca9b/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/data/DataCollection.java#L38) 作为数据容器。

## 2.3 Window

`org.skywalking.apm.collector.core.cache.Window` ，窗口**抽象类**。

[构造方法](https://github.com/YunaiV/skywalking/blob/d6019b49d686f79d07528a19b09d4f0ccc4eca9b/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/cache/Window.java#L50) ，代码如下：

* [`windowDataA`](https://github.com/YunaiV/skywalking/blob/d6019b49d686f79d07528a19b09d4f0ccc4eca9b/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/cache/Window.java#L44) 属性，窗口数据A 。
* [`windowDataB`](https://github.com/YunaiV/skywalking/blob/d6019b49d686f79d07528a19b09d4f0ccc4eca9b/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/cache/Window.java#L48) 属性，窗口数据B 。
* 通过 [`#collectionInstance()`](https://github.com/YunaiV/skywalking/blob/d6019b49d686f79d07528a19b09d4f0ccc4eca9b/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/cache/Window.java#L61) **抽象**方法，创建窗口数据( Collection )对象。
* [`pointer`](https://github.com/YunaiV/skywalking/blob/d6019b49d686f79d07528a19b09d4f0ccc4eca9b/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/cache/Window.java#L39) 属性，数据指向 `windowDataA` 或 `windowDataA`。
    * [`#getCurrent()`](https://github.com/YunaiV/skywalking/blob/d6019b49d686f79d07528a19b09d4f0ccc4eca9b/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/cache/Window.java#L111) 方法，获得**现**数据指向，即 `pointer` 。
    * [`#getLast()`](https://github.com/YunaiV/skywalking/blob/d6019b49d686f79d07528a19b09d4f0ccc4eca9b/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/cache/Window.java#L118) 方法，获得**原**数据指向，即**非** `pointer` 。
* [`windowSwitch`](https://github.com/YunaiV/skywalking/blob/d6019b49d686f79d07528a19b09d4f0ccc4eca9b/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/cache/Window.java#L33) 属性，窗口切换计数。

-------

**切换 Collection 相关**，方法如下：

* [`#trySwitchPointer()`](https://github.com/YunaiV/skywalking/blob/d6019b49d686f79d07528a19b09d4f0ccc4eca9b/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/cache/Window.java#L68) 方法，返回是否可以切换 Collection 。可以切换需要满足如下条件：
    * 只有**一个调用方**申请切换，通过 `windowSwitch` 属性进行计数。
    * **原**数据指向**不处于正在读取状态**。如果切换，一边读一边写，可能会有并发问题。
    * 无论是否可以切换 Collection ，需要调用 [`#trySwitchPointerFinally()`](https://github.com/YunaiV/skywalking/blob/d6019b49d686f79d07528a19b09d4f0ccc4eca9b/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/cache/Window.java#L75) 方法，**释放** `windowSwitch` 的计数。
* [`#switchPointer()`](https://github.com/YunaiV/skywalking/blob/d6019b49d686f79d07528a19b09d4f0ccc4eca9b/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/cache/Window.java#L82) 方法，切换数据指向，并标记**原**数据指向的 Collection **正在读取中**。
* [`#finishReadingLast()`](https://github.com/YunaiV/skywalking/blob/d6019b49d686f79d07528a19b09d4f0ccc4eca9b/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/cache/Window.java#L129) 方法，清空**原**数据指向的 Collection 数据，并标记**原**数据指向的 Collection **完成读取**( **不在正在读取中** )。

-------

**写 Collection 相关**，方法如下：

* [`#getCurrentAndWriting()`](https://github.com/YunaiV/skywalking/blob/d6019b49d686f79d07528a19b09d4f0ccc4eca9b/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/cache/Window.java#L98) 方法，获得**现**数据指向，并标记**正在写入中**。通过**正在写入**标记，切换 Collection 完成后，可以判断该 Collection 正在写入中，**若是，等待不在写入中，开始数据读取并处理**。

## 2.4 DataCache

`org.skywalking.apm.collector.stream.worker.impl.data.DataCache` ，实现 Window 抽象类，数据缓存。

* [`#collectionInstance()`](https://github.com/YunaiV/skywalking/blob/e6e1a7899e7b0e674a61f323447d4b66cffa8136/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/data/DataCache.java#L36) **实现**方法，创建 DataCollection 对象。
* [`#currentCollectionSize()`](https://github.com/YunaiV/skywalking/blob/e6e1a7899e7b0e674a61f323447d4b66cffa8136/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/data/DataCache.java#L82) 方法，获得当前数据指向( 写入 Collection )的数据数量。

-------

**写 Collection 相关**，方法如下：

* [`#writing()`](https://github.com/YunaiV/skywalking/blob/e6e1a7899e7b0e674a61f323447d4b66cffa8136/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/data/DataCache.java#L73) 方法，调用 `#getCurrentAndWriting()` 方法，开始写入。即，获得**现**数据指向，并标记**正在写入中**。
    * [`lockedDataCollection`](https://github.com/YunaiV/skywalking/blob/e6e1a7899e7b0e674a61f323447d4b66cffa8136/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/data/DataCache.java#L34) 属性，写入的窗口数据。
    * [`#put(id, data)`](https://github.com/YunaiV/skywalking/blob/e6e1a7899e7b0e674a61f323447d4b66cffa8136/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/data/DataCache.java#L66) 方法，向 `lockedDataCollection` 属性，写入 Data 。
    * [`#get(id)`](https://github.com/YunaiV/skywalking/blob/e6e1a7899e7b0e674a61f323447d4b66cffa8136/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/data/DataCache.java#L56) 方法，向 `lockedDataCollection` 属性，根据 ID 获得 Data 。
    * [`#containsKey(id)`](https://github.com/YunaiV/skywalking/blob/e6e1a7899e7b0e674a61f323447d4b66cffa8136/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/data/DataCache.java#L46) 方法，向 `lockedDataCollection` 属性，根据 ID 判断 Data 是否存在 。
* [`#finishWriting()`](https://github.com/YunaiV/skywalking/blob/e6e1a7899e7b0e674a61f323447d4b66cffa8136/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/data/DataCache.java#L89) 方法，完成写入。即，标记 `lockedDataCollection` **不在正在写入中**。

# 3. AggregationWorker

`org.skywalking.apm.collector.stream.worker.impl.AggregationWorker` ，实现 AbstractLocalAsyncWorker 抽象类，**异步聚合** Worker，负责聚合处理数据，后提交 Data 到 Next 节点们处理。

[**构造方法**](https://github.com/YunaiV/skywalking/blob/ccd030f6b449954fccc7be4e19c15290e8affc77/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/AggregationWorker.java#L47) ，代码如下：

* [`dataCache`](https://github.com/YunaiV/skywalking/blob/ccd030f6b449954fccc7be4e19c15290e8affc77/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/AggregationWorker.java#L41) 属性，数据缓存。
* [`messageNum`](https://github.com/YunaiV/skywalking/blob/ccd030f6b449954fccc7be4e19c15290e8affc77/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/AggregationWorker.java#L45) 属性，消息计数。当超过一定数量( 目前是 100 )，重置计数归零。

-------

[`#onWork(message)`](https://github.com/YunaiV/skywalking/blob/ccd030f6b449954fccc7be4e19c15290e8affc77/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/AggregationWorker.java#L52) **实现**方法，聚合处理数据，当满足条件时，提交 Data 到 Next 节点们处理。

* 第 53 行：`messageNum` 计数增加。
* 第 56 行：调用 [`#aggregate(message)`](https://github.com/YunaiV/skywalking/blob/ccd030f6b449954fccc7be4e19c15290e8affc77/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/AggregationWorker.java#L97) 方法，聚合消息到数据。
* 第 59 至 62 行：`messageNum >= 100` 时，调用 [`#sendToNext()`](https://github.com/YunaiV/skywalking/blob/ccd030f6b449954fccc7be4e19c15290e8affc77/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/AggregationWorker.java#L70) ，提交缓存数据的**读 Collection 的数据**给 Next 节点们继续处理。
* 第 65 至 67 行：`messageNum.endOfBatch == true` 时，当消息是批处理的最后一条时，调用 [`#sendToNext()`](https://github.com/YunaiV/skywalking/blob/ccd030f6b449954fccc7be4e19c15290e8affc77/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/AggregationWorker.java#L70) ，提交缓存数据的**读 Collection 的数据**给 Next 节点们继续处理。

-------

[`#sendToNext()`](https://github.com/YunaiV/skywalking/blob/ccd030f6b449954fccc7be4e19c15290e8affc77/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/AggregationWorker.java#L70) 方法，提交缓存数据的**读 Collection 的数据**给 Next 节点们继续处理。

* 第 72 行：**直接**调用 `Window#switchPointer()` 方法，切换数据指针，并标记**原**指向正在读取中。这里并未先调用 `Window#trySwitchPointer()` 方法，**是否会有并发问题**？目前这里是**异步单线程**，所以不会有问题，参见 [《SkyWalking 源码分析 —— Collector Queue 队列组件》](http://www.iocoder.cn/SkyWalking/collector-queue-module/?self) 。另外，在 [「4. PersistenceWorker」](#) 会看到并发的情况处理。
* 第 74 至 80 行：等待**原**指向不在读取中。
* 第 82 至 85 行：提交数据给 Next 节点们继续处理。
* 第 87 行：标记**原**指向完成读取。

# 4. PersistenceWorker

`org.skywalking.apm.collector.stream.worker.impl.PersistenceWorker` ，实现 AbstractLocalAsyncWorker 抽象类，**异步批量存储** Worker，负责聚合处理数据，后存储 Data 。

考虑到需要保证存储的时效性，PersistenceWorker 使用 PersistenceTimer ，定时存储 Data ，在 [「4.2 PersistenceWorker」](#) 详细解析。

-------

[**构造方法**](https://github.com/YunaiV/skywalking/blob/ccd030f6b449954fccc7be4e19c15290e8affc77/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/AggregationWorker.java#L47) ，代码如下：

* [`dataCache`](https://github.com/YunaiV/skywalking/blob/bf563461e75b6717d7b4589d7c3f579acf305541/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/PersistenceWorker.java#L49) 属性，数据缓存。
* [`batchDAO`](https://github.com/YunaiV/skywalking/blob/bf563461e75b6717d7b4589d7c3f579acf305541/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/PersistenceWorker.java#L53) 属性，批量操作 DAO ，在 [《SkyWalking 源码分析 —— Collector Storage 存储组件》](http://www.iocoder.cn/SkyWalking/collector-storage-module/?self) 有详细解析。

-------

[`#needMergeDBData()`](https://github.com/YunaiV/skywalking/blob/bf563461e75b6717d7b4589d7c3f579acf305541/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/PersistenceWorker.java#L186) **抽象**方法，存储时，是否需要合并数据。一些 Data 只有新增操作，没有更新操作。

[`#persistenceDAO()`](https://github.com/YunaiV/skywalking/blob/bf563461e75b6717d7b4589d7c3f579acf305541/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/PersistenceWorker.java#L181) **抽象**方法，获得 Data 对应的持久化 DAO 接口的**实现类**对象。

上述两个**抽象**方法，用于 [`#prepareBatch(dataMap)`](https://github.com/YunaiV/skywalking/blob/bf563461e75b6717d7b4589d7c3f579acf305541/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/PersistenceWorker.java#L121) 方法，生成批量操作对象数组，最终调用 `IBatchDAO#batchPersistence(List<?>)` 方法，通过执行批量操作对象数组，实现批量持久化数据，在 [《SkyWalking 源码分析 —— Collector Storage 存储组件》](http://www.iocoder.cn/SkyWalking/collector-storage-module/?self) 有详细解析。

-------

[`#onWork(message)`](https://github.com/YunaiV/skywalking/blob/bf563461e75b6717d7b4589d7c3f579acf305541/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/PersistenceWorker.java#L71) **实现**方法，当满足条件时存储 Data ，而后聚合数据。这点和 AggregationWorker 相反的，因为要考虑并发问题。代码如下：

* 第 72 行：调用 `DataCache#currentCollectionSize()` 方法，获得当前写入 Collection 的数据数量，判断是否超过 5000 。
    * 第 75 行：调用 `DataCache#trySwitchPointer()` 方法，**判断**是否可以切换 Collection 。通过该判断，保证和 PersistenceTimer 一起时，不会出现**并发问题**。
    * 第 77 行：调用 `Window#switchPointer()` 方法，切换数据指针，并标记**原**指向正在读取中。
    * 第 80 行：调用 [`#buildBatchCollection()`](https://github.com/YunaiV/skywalking/blob/bf563461e75b6717d7b4589d7c3f579acf305541/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/PersistenceWorker.java#L100) 方法，创建批量操作对象数组。该方法和 `AggregationWorker#sendToNext()` 方法**基本类似**。
    * 第 83 行：调用 `IBatchDAO#batchPersistence(List<?>)` 方法，通过执行批量操作对象数组，实现批量持久化数据。
    * 第 86 行：调用 `DataCache#trySwitchPointerFinally()` 方法，**释放** `DataCache.windowSwitch` 的计数。
* 第 91 行：调用 [`#aggregate(message)`](https://github.com/YunaiV/skywalking/blob/bf563461e75b6717d7b4589d7c3f579acf305541/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/PersistenceWorker.java#L164) 方法，聚合数据。该方法和 `AggregationWorker#aggregate(message)` 方法**基本相似**。

## 4.1 WorkerCreateListener

`org.skywalking.apm.collector.stream.worker.base.WorkerCreateListener` ，Worker 创建监听器。

Worker 在创建时，会调用 [`WorkerCreateListener#addWorker`](https://github.com/YunaiV/skywalking/blob/bf563461e75b6717d7b4589d7c3f579acf305541/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/base/WorkerCreateListener.java#L36) 方法，记录所有的 PersistenceWorker 对象。

**记录下来有什么用呢**？在 [AgentStreamBootStartup](https://github.com/YunaiV/skywalking/blob/bf563461e75b6717d7b4589d7c3f579acf305541/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/AgentStreamBootStartup.java#L47) 启动时，创建 PersistenceTimer 对象，并将 WorkerCreateListener 记录的 PersistenceWorker 对象集合**传递**给 PersistenceTimer 对象。这样，PersistenceTimer 能够"**访问**"到 PersistenceWorker 对象们的 DataCache ，**定时**存储数据。

## 4.2 PersistenceTimer

`org.skywalking.apm.collector.stream.timer.PersistenceTimer` ，持久化定时任务，负责**定时**、**批量**存储 PersistenceWorker 缓存的数据。

[`#start(IBatchDAO, List<PersistenceWorker>)`](https://github.com/YunaiV/skywalking/blob/ec95c9966b7e013c8a44ea2e0f748ad339363ce3/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/timer/PersistenceTimer.java#L43) 方法，创建延迟 1 秒，每 1 秒执行一次 `#extractDataAndSave()` 方法的定时任务，用于**定时**、**批量**存储 PersistenceWorker 缓存的数据。

[`#extractDataAndSave(IBatchDAO, List<PersistenceWorker>)`](https://github.com/YunaiV/skywalking/blob/ec95c9966b7e013c8a44ea2e0f748ad339363ce3/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/timer/PersistenceTimer.java#L52) 方法，代码如下：

* 第 55 至 68 行：获得所有 PersistenceWorker 读 Collection 缓存的数据。
    * 第 60 行：调用 [`PersistenceWorker#flushAndSwitch()`](https://github.com/YunaiV/skywalking/blob/ec95c9966b7e013c8a44ea2e0f748ad339363ce3/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/PersistenceWorker.java#L61) 切换数据指针，即切换读写 Collection 。
    * 第 62 行：调用 [`PersistenceWorker#buildBatchCollection()`](https://github.com/YunaiV/skywalking/blob/ec95c9966b7e013c8a44ea2e0f748ad339363ce3/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/PersistenceWorker.java#L100) 方法，创建批量操作对象数组。
    * **怎么保证并发安全**？通过 `Window#trySwitchPointer()` 方法，保证读 Collection **正在被读取中**时，PersistenceWorker 和 PersistenceTimer 有且仅有一个切换队列，读取数据。当读取完成后，调用 `Window#finishReadingLast()` 方法，清空原数据指向，并标记原数据指向完成正在读取中。

* 第 71 行：调用 `IBatchDAO#batchPersistence(List<?>)` 方法，执行批量操作，进行存储。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

呼呼，终于把流式处理给写完了，如果写的不详细或者不合适的，胖友看到麻烦告知笔者，谢谢。

胖友，分享一波朋友圈可好。


