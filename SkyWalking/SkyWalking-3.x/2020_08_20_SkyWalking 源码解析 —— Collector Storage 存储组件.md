title: SkyWalking 源码分析 —— Collector Storage 存储组件
date: 2020-08-20
tags:
categories: SkyWalking
permalink: SkyWalking/collector-storage-module

-------

摘要: 原创出处 http://www.iocoder.cn/SkyWalking/collector-storage-module/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 SkyWalking 3.2.6 正式版**

- [1. 概述](http://www.iocoder.cn/SkyWalking/collector-storage-module/)
- [2. apm-collector-core](http://www.iocoder.cn/SkyWalking/collector-storage-module/)
  - [2.1 Table](http://www.iocoder.cn/SkyWalking/collector-storage-module/)
  - [2.2 TableDefine](http://www.iocoder.cn/SkyWalking/collector-storage-module/)
  - [2.3 Data](http://www.iocoder.cn/SkyWalking/collector-storage-module/)
- [3. collector-storage-define](http://www.iocoder.cn/SkyWalking/collector-storage-module/)
  - [3.1 StorageModule](http://www.iocoder.cn/SkyWalking/collector-storage-module/)
  - [3.2 table 包](http://www.iocoder.cn/SkyWalking/collector-storage-module/)
  - [3.3 StorageInstaller](http://www.iocoder.cn/SkyWalking/collector-storage-module/)
  - [3.4 dao 包](http://www.iocoder.cn/SkyWalking/collector-storage-module/)
- [4. collector-storage-h2-provider](http://www.iocoder.cn/SkyWalking/collector-storage-module/)
- [5. collector-storage-es-provider](http://www.iocoder.cn/SkyWalking/collector-storage-module/)
  - [5.1 StorageModuleEsProvider](http://www.iocoder.cn/SkyWalking/collector-storage-module/)
  - [5.2 define 包](http://www.iocoder.cn/SkyWalking/collector-storage-module/)
  - [5.3 dao 包](http://www.iocoder.cn/SkyWalking/collector-storage-module/)
  - [5.4 DataTTLKeeperTimer](http://www.iocoder.cn/SkyWalking/collector-storage-module/)
- [666. 彩蛋](http://www.iocoder.cn/SkyWalking/collector-storage-module/)

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

本文主要分享 **SkyWalking Collector Storage 存储组件**。顾名思义，负责将调用链路、应用、应用实例等等信息存储到存储器，例如，ES 、H2 。

> 友情提示：建议先阅读 [《SkyWalking 源码分析 —— Collector 初始化》](http://www.iocoder.cn/SkyWalking/collector-init/?self) ，以了解 Collector 组件体系。

> FROM https://github.com/apache/incubating-skywalking  
> ![](http://www.iocoder.cn/images/SkyWalking/2020_08_20/01.jpeg)

下面我们来看看整体的项目结构，如下图所示 ：

![](http://www.iocoder.cn/images/SkyWalking/2020_08_20/02.png)

* `apm-collector-core` 的 `data` 和 `define` **包** ：数据的抽象。
* `collector-storage-define` ：定义存储组件接口。
* `collector-storage-h2-provider` ：基于 H2 的 存储组件实现。**该实现是单机版，建议仅用于 SkyWalking 快速上手，生产环境不建议使用**。
* `collector-storage-es-provider` ：基于 Elasticsearch 的集群管理实现。**生产环境推荐使用**。

下面，我们从**接口到实现**的顺序进行分享。

# 2. apm-collector-core

`apm-collector-core` 的 `data` 和 `define` **包**，如下图所示：![](http://www.iocoder.cn/images/SkyWalking/2020_08_20/03.png)

我们对类进行梳理分类，如下图：![](http://www.iocoder.cn/images/SkyWalking/2020_08_20/04.png)

* Table ：Data 和 TableDefine 之间的桥梁，每个 Table 定义了该表的**表名**，**字段名们**。
* TableDefine ：Table 的详细定义，包括**表名**，**字段定义**( ColumnDefine )们。在下文中，[StorageInstaller](https://github.com/apache/incubator-skywalking/blob/15328202b8b7df89a609885d9110361ff29ce668/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/apache/skywalking/apm/collector/storage/StorageInstaller.java) 会基于 TableDefine 初始化表的相关信息。
* Data ：数据，包括**一条**数据的数据值们和数据字段( Column )们。在下文中，[Dao](https://github.com/apache/incubator-skywalking/blob/15328202b8b7df89a609885d9110361ff29ce668/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/apache/skywalking/apm/collector/storage/base/dao/DAO.java) 会存储 Data 到存储器中。另外，在 [《SkyWalking 源码分析 —— Collector Streaming Computing 流式处理（一）》](http://www.iocoder.cn/SkyWalking/collector-streaming-first/?self) 中，我们也会看到对 Data 的流式处理**通用**封装。

## 2.1 Table

[`org.skywalking.apm.collector.core.data.CommonTable`](https://github.com/YunaiV/skywalking/blob/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/CommonTable.java) ，通用表。

* [`TABLE_TYPE`](https://github.com/YunaiV/skywalking/blob/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/CommonTable.java#L33) **静态**属性，表类型。目前只有 ES 存储组件使用到，下文详细解析。
* [`COLUMN_`](https://github.com/YunaiV/skywalking/blob/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/CommonTable.java#L37) 前缀的**静态**属性，通用的字段名。

在 `collector-storage-define` 的 [`table`](https://github.com/YunaiV/skywalking/tree/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table) **包**下，我们可以看到所有 Table 类，以 `"Table"` 结尾。每个 Table 的表名，在每个实现类里，例如 [ApplicationTable](https://github.com/YunaiV/skywalking/blob/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/register/ApplicationTable.java) 。

## 2.2 TableDefine

`org.skywalking.apm.collector.core.data.TableDefine` ，表定义**抽象类**。

* [`name`](https://github.com/YunaiV/skywalking/blob/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/TableDefine.java#L34) 属性，表名。
* [`columnDefines`](https://github.com/YunaiV/skywalking/blob/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/TableDefine.java#L38) 属性，ColumnDefine数组。
* [`#initialize()`](https://github.com/YunaiV/skywalking/blob/578ea4f66f11bdfe5dcda25f574a1ed57ca47d24/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/TableDefine.java#L48) **抽象**方法，初始化表定义。例如：[ApplicationEsTableDefine](https://github.com/YunaiV/skywalking/blob/578ea4f66f11bdfe5dcda25f574a1ed57ca47d24/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/define/ApplicationEsTableDefine.java#L38) 。

不同的存储组件实现，有不同的 TableDefine 实现类，如下图：![](http://www.iocoder.cn/images/SkyWalking/2020_08_20/05.png)

* ElasticSearchTableDefine ：基于 Elasticsearch 的表定义**抽象类**，在 `collector-storage-es-provider` 的 [`define`](https://github.com/YunaiV/skywalking/tree/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/define) **包**下，我们可以看到**所有** ES 的 TableDefine 类。

* H2TableDefine ：基于 H2 的表定义**抽象类**，在 `collector-storage-h2-provider` 的 [`define`](https://github.com/YunaiV/skywalking/tree/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-storage/collector-storage-h2-provider/src/main/java/org/skywalking/apm/collector/storage/h2/define) **包**下，我们可以看到**所有** H2 的 TableDefine 类。

### 2.2.1 ColumnDefine

 [`org.skywalking.apm.collector.core.data.ColumnDefine`](https://github.com/YunaiV/skywalking/blob/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/ColumnDefine.java) ，字段定义**抽象类**。

* [`name`](https://github.com/YunaiV/skywalking/blob/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/ColumnDefine.java#L31) 属性，字段名。
* [`type`](https://github.com/YunaiV/skywalking/blob/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/ColumnDefine.java#L35) 属性，字段类型。

在 `collector-storage-xxx-provider` 模块中，H2ColumnDefine 、ElasticSearchColumnDefine 实现 ColumnDefine 。

### 2.2.2 Loader

涉及到的类如下图所示：![](http://www.iocoder.cn/images/SkyWalking/2020_08_20/06.png)

[`org.skywalking.apm.collector.core.data.StorageDefineLoader`](https://github.com/YunaiV/skywalking/blob/0aa5e6a49c1f29b43824ebabf6bb7d76b80e3eb7/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/StorageDefineLoader.java) ，调用 [`org.skywalking.apm.collector.core.define.DefinitionLoader`](https://github.com/YunaiV/skywalking/blob/0aa5e6a49c1f29b43824ebabf6bb7d76b80e3eb7/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/define/DefinitionLoader.java) ，从 [`org.skywalking.apm.collector.core.data.StorageDefinitionFile`](https://github.com/YunaiV/skywalking/blob/0aa5e6a49c1f29b43824ebabf6bb7d76b80e3eb7/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/StorageDefinitionFile.java) 中，加载 TableDefine 实现类数组。

另外，在 `collector-storage-es-provider` 和 `collector-storage-h2-provider` 里都有 `storage.define` 文件，如下图：![](http://www.iocoder.cn/images/SkyWalking/2020_08_20/07.png)

* StorageDefinitionFile 声明了读取该文件。
* **注意**，DefinitionLoader 在加载时，两个文件都会被读取，最终在 `StorageInstaller#defineFilter(List<TableDefine>)` 方法，进行过滤。

代码比较简单，中文注释已加，胖友自己阅读理解下。

## 2.3 Data

`org.skywalking.apm.collector.core.data.Data`  ，数据**抽象类**。

* [`dataXXX`]() **前缀**的属性，字段值们。
    * [`dataStrings`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/Data.java#L30) 属性的第一位，是 **ID** 属性。参见 [**构造方法**](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/Data.java#L51)的【第 51 行】 或者 [`#setId(id)`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/Data.java#L142) 方法。
* [`xxxColumns`]() **后缀**的属性，字段( Column )们。
* 通过上述两种属性 + 自身类，可以确定一条数据记录的表、字段类型、字段名、字段值。
* **继承** [`org.skywalking.apm.collector.core.data.EndOfBatchQueueMessage`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/EndOfBatchQueueMessage.java) ，带是否消息批处理的最后一条标记的**消息抽象类**，[`endOfBatch`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/EndOfBatchQueueMessage.java#L31) 属性，在 [《SkyWalking 源码分析 —— Collector Streaming Computing 流式处理（二）》「3. AggregationWorker」](http://www.iocoder.cn/SkyWalking/collector-streaming-second/?self) 详细解析。
    * **继承** [`org.skywalking.apm.collector.core.data.AbstractHashMessage`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/AbstractHashMessage.java) ，带哈希码的**消息抽象类**，[`hashCode`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/AbstractHashMessage.java#L38) 属性，在 [《SkyWalking 源码分析 —— Collector Streaming Computing 流式处理（二）》「3. AggregationWorker」](http://www.iocoder.cn/SkyWalking/collector-streaming-second/?self) 详细解析。
* [`#mergeData(Data)`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/Data.java#L154) 方法，合并传入的数据到自身。该方法被 [`AggregationWorker#aggregate(message)`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/AggregationWorker.java#L94) 调用，在 [《SkyWalking 源码分析 —— Collector Streaming Computing 流式处理（二）》「3. AggregationWorker」](http://www.iocoder.cn/SkyWalking/collector-streaming-second/?self) 详细解析。

在 `collector-storage-define` 的 [`table`](https://github.com/YunaiV/skywalking/tree/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table) **包**下，我们可以看到所有 Data 类，**非** `"Table"` 结尾，例如 [Application](https://github.com/YunaiV/skywalking/blob/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/register/Application.java) 。

### 2.3.1 Column

[`org.skywalking.apm.collector.core.data.Column`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/Column.java) ，字段。

* [`name`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/Column.java#L31) 属性，字段名。
* [`operation`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/Column.java#L35) 属性，操作( Operation )。

### 2.3.2 Operation

[`org.skywalking.apm.collector.core.data.Operation`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/Operation.java) ，操作**接口**。用于两个值之间的操作，例如，相加等等。目前实现类有：

* [AddOperation](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/operator/AddOperation.java) ：值相加操作。
* [CoverOperation](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/operator/CoverOperation.java) ：值覆盖操作，即以新值为返回。
* [NonOperation](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/operator/NonOperation.java) ：空操作，即以老值为返回。

# 3. collector-storage-define

`collector-cluster-define` ：定义存储组件接口。项目结构如下 ：

![](http://www.iocoder.cn/images/SkyWalking/2020_08_20/08.png)

## 3.1 StorageModule

`org.skywalking.apm.collector.storage.StorageModule` ，实现 [Module](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Module.java) 抽象类，集群管理 Module 。

[`#name()`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/StorageModule.java#L67) **实现**方法，返回模块名为 `"storage"` 。

[`#services()`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/StorageModule.java#L71) **实现**方法，返回 Service 类名：在 [org.skywalking.apm.collector.storage.dao](https://github.com/YunaiV/skywalking/tree/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/dao) **包**下的所有类 和 IBatchDAO。

## 3.2 table 包

在 [`org.skywalking.apm.collector.storage.table`](https://github.com/YunaiV/skywalking/tree/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table) 包下，定义了存储模块所有的 Table 和 Data 实现类。

## 3.3 StorageInstaller

`org.skywalking.apm.collector.storage.StorageInstaller` ，存储安装器**抽象类**，基于 TableDefine ，初始化存储组件的表。

* [`#defineFilter(List<TableDefine>)`](https://github.com/YunaiV/skywalking/blob/2b700457124e7d4f788343d8bcd9a03d2e273aca/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/StorageInstaller.java#L71) **抽象**方法，过滤 TableDefine 数组中，非自身需要的。例如说，ElasticSearchStorageInstaller 过滤后，只保留 ElasticSearchTableDefine 对象。
* [`#isExists(Client, TableDefine)`](https://github.com/YunaiV/skywalking/blob/2b700457124e7d4f788343d8bcd9a03d2e273aca/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/StorageInstaller.java#L81) **抽象**方法，判断表是否存在。
* [`#deleteTable(Client, TableDefine)`](https://github.com/YunaiV/skywalking/blob/2b700457124e7d4f788343d8bcd9a03d2e273aca/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/StorageInstaller.java#L81) **抽象**方法，删除表。
* [`#createTable(Client, TableDefine)`](https://github.com/YunaiV/skywalking/blob/2b700457124e7d4f788343d8bcd9a03d2e273aca/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/StorageInstaller.java#L81) **抽象**方法，创建表。
* [`#install(Client)`](https://github.com/YunaiV/skywalking/blob/2b700457124e7d4f788343d8bcd9a03d2e273aca/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/StorageInstaller.java#L39) 方法，基于 TableDefine ，初始化存储组件的表。
    * 该方法会被 StorageModuleH2Provider 或 StorageModuleEsProvider 启动时调用。

## 3.4 dao 包

在 `collector-storage-define` 项目结构图，我们看到一共有**两**个 `bao` 包：

* `org.skywalking.apm.collector.storage.base.dao` ，**系统**的 DAO 接口。
* `org.skywalking.apm.collector.storage.dao` ，**业务**的 DAO 接口。
    * **继承**系统的 DAO 接口。
    * 被 `collector-storage-xxx-provider` 的 `dao` 包**实现**。

![](http://www.iocoder.cn/images/SkyWalking/2020_08_20/09.png)

### 3.4.1 系统 DAO

[`org.skywalking.apm.collector.storage.base.dao.DAO`](https://github.com/YunaiV/skywalking/blob/2b700457124e7d4f788343d8bcd9a03d2e273aca/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/base/dao/DAO.java) ，继承 [Service](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Service.java) 接口，DAO **接口**。

无任何方法。

#### 3.4.1.1 AbstractDAO

[`org.skywalking.apm.collector.storage.base.dao.AbstractDAO`](https://github.com/YunaiV/skywalking/blob/2b700457124e7d4f788343d8bcd9a03d2e273aca/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/base/dao/AbstractDAO.java) ，实现 DAO 接口，DAO 抽象基类。

* [`client`](https://github.com/YunaiV/skywalking/blob/2b700457124e7d4f788343d8bcd9a03d2e273aca/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/base/dao/AbstractDAO.java#L33) 属性，数据操作客户端。例如，H2Client 、ElasticSearchClient 。

在 `collector-storage-xxx-provider` 模块中，H2DAO 、EsDAO 实现 AbstractDAO 。

#### 3.4.1.2 IPersistenceDAO

`org.skywalking.apm.collector.storage.base.dao.IPersistenceDAO` ，实现 DAO 接口，持久化 DAO **接口**，定义了 Data 的增删改查操作。

* [`#get(id)`](https://github.com/YunaiV/skywalking/blob/2b700457124e7d4f788343d8bcd9a03d2e273aca/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/base/dao/IPersistenceDAO.java#L38) **接口**方法，根据 ID 查询一条 Data 。
* [`#deleteHistory(startTimestamp, endTimestamp)`](https://github.com/YunaiV/skywalking/blob/2b700457124e7d4f788343d8bcd9a03d2e273aca/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/base/dao/IPersistenceDAO.java#L68) **接口**方法，删除时间范围内的 Data 们。
* [`#prepareBatchInsert(data)`](https://github.com/YunaiV/skywalking/blob/2b700457124e7d4f788343d8bcd9a03d2e273aca/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/base/dao/IPersistenceDAO.java#L50) **接口**方法，准备批量插入操作对象。例如：[`CpuMetricEsPersistenceDAO#prepareBatchInsert(CpuMetric)`](https://github.com/YunaiV/skywalking/blob/2b700457124e7d4f788343d8bcd9a03d2e273aca/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/CpuMetricEsPersistenceDAO.java) 方法，返回的是 `org.elasticsearch.action.index.IndexRequestBuilder` 对象。注意：
    * 该方法不会发起具体的 DAO 操作，仅仅是创建插入操作对象，最终的执行在 `IBatchDAO#batchPersistence(List<?>)`。
    * 该方法创建的是批量插入操作对象们中的一个。
* [`#prepareBatchUpdate(data)`](https://github.com/YunaiV/skywalking/blob/2b700457124e7d4f788343d8bcd9a03d2e273aca/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/base/dao/IPersistenceDAO.java#L60) **接口**方法，准备批量更新操作对象。类似 `#prepareBatchInsert(data)` 方法。

#### 3.4.1.3 IBatchDAO

`org.skywalking.apm.collector.storage.base.dao.IBatchDAO` ，实现 DAO 接口，批量操作 DAO **接口**。

* [`#batchPersistence(List<?> batchCollection)`](https://github.com/YunaiV/skywalking/blob/2b700457124e7d4f788343d8bcd9a03d2e273aca/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/base/dao/IBatchDAO.java#L39) **接口**方法，通过执行批量操作对象数组，实现批量持久化数据。
    * `batchCollection` **方法参数**，通过 `IPersistenceDAO#prepareBatchInsert` 或 `IPersistenceDAO#prepareBatchUpdate` 方法，生成**每个**操作数组元素。
    * 该方法会被 `PersistenceTimer#extractDataAndSave(...)` 或 `PersistenceWorker#onWork(...)` 方法调用，在 [《SkyWalking 源码分析 —— Collector Streaming Computing 流式处理（二）》「4. PersistenceWorker」](http://www.iocoder.cn/SkyWalking/collector-streaming-second/?self) 详细解析。

在 `collector-storage-xxx-provider` 模块中，BatchH2DAO 、BatchEsDAO 实现 IBatchDAO 。

### 3.4.2 业务 DAO

在 [`StorageModule#services()`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/StorageModule.java#L71) 方法里，我们可以看到，业务 DAO 按照**用途**可以拆分成**四种**：

* Cache ：缓存应用、应用实例、服务名
* Register ：注册应用、应用实例、服务名
* Persistence ：持久化，实际可以理解成批量持久化
* UI ：SkyWaling UI 查询使用。

那么整理如下：

| Package | Data | Cache / Register | Persistence | UI | 关联文章 |
| --- | --- | --- | --- | ---  | --- |
| register | Application | √ |  |  |  |
| register | Instance | √ | √ | √ |  |
| register | ServiceName | √ |  |  |  |
| jvm | CpuMetric |  | √ | √ |  |
| jvm | CMetric |  | √ | √ |  |
| jvm | MemoryMetric |  | √ | √ |  |
| jvm | MemoryPoolMetric |  | √ | √ |  |
| global | GlobalTrace |  | √ | √ |  |
| instance | InstPerformance |  | √ | √ |  |
| node | NodeComponent |  | √ | √ |  |
| node | NodeMapping |  | √ | √ |  |
| noderef | NodeReference |  | √ | √ |  |
| segment | SegmentCost |  | √ | √ |  |
| segment | Segment |  | √ | √ |  |
| service | ServiceEntry |  | √ | √ |  |
| serviceref | ServiceReference |  | √ | √ |  |

# 4. collector-storage-h2-provider

`collector-storage-h2-provider` ，基于 H2 的存储组件实现。项目结构如下 ：![](http://www.iocoder.cn/images/SkyWalking/2020_08_20/12.png)

**该实现是单机版，建议仅用于 SkyWalking 快速上手，生产环境不建议使用**。

由于生产环境主要使用 ES 的存储组件实现，所以本文暂不解析相关实现，感兴趣的胖友自己嗨起来。

# 5. collector-storage-es-provider

`collector-storage-es-provider` ，基于 ES 的存储组件实现。项目结构如下 ：![](http://www.iocoder.cn/images/SkyWalking/2020_08_20/10.png)

实际使用时，通过 `application.yml` 配置如下：

``` JSON
storage:
  elasticsearch:
    cluster_name: elasticsearch
    cluster_transport_sniffer: true
    cluster_nodes: 127.0.0.1:9300
    index_shards_number: 2
    index_replicas_number: 0
    ttl: 7
```

* 生产环境下，推荐 Elasticsearch 配置成集群。
* `cluster_name` 、`cluster_transport_sniffer` 、`cluster_nodes` 、`index_shards_number` 、`index_replicas_number` 参数，Elasticsearch 相关参数。
* `ttl` ：保留 N 天内的数据。超过 N 天的数据，将被自动滚动删除。
    * 该功能目前版本**暂未发布**，需要等到 5.0 版本后。
* [《部署集群collector》](https://github.com/apache/incubator-skywalking/blob/master/docs/cn/Deploy-collector-in-cluster-mode-CN.md)

## 5.1 StorageModuleEsProvider

`org.skywalking.apm.collector.storage.es.StorageModuleEsProvider` ，实现 [ModuleProvider](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/ModuleProvider.java) **抽象类**，基于 ES 的存储组件服务提供者。

[`#name()`](https://github.com/YunaiV/skywalking/blob/7eecc004e69685fe6f5ce3b3dc4822475d6aa713/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/StorageModuleEsProvider.java#L62) **实现**方法，返回组件服务提供者名为 `"elasticsearch"` 。

[`module()`](https://github.com/YunaiV/skywalking/blob/7eecc004e69685fe6f5ce3b3dc4822475d6aa713/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/StorageModuleEsProvider.java#L66) **实现**方法，返回组件类为 StorageModule 。

[`#requiredModules()`](https://github.com/YunaiV/skywalking/blob/7eecc004e69685fe6f5ce3b3dc4822475d6aa713/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/StorageModuleEsProvider.java#L119) **实现**方法，返回依赖组件为 `"cluster"` 。

-------

[`#prepare(Properties)`](https://github.com/YunaiV/skywalking/blob/7eecc004e69685fe6f5ce3b3dc4822475d6aa713/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/StorageModuleEsProvider.java#L70) **实现**方法，执行准备阶段逻辑。

* 第 71 至 75 行 ：创建 [`org.skywalking.apm.collector.client.elasticsearch.ElasticSearchClient`](https://github.com/YunaiV/skywalking/blob/7eecc004e69685fe6f5ce3b3dc4822475d6aa713/apm-collector/apm-collector-component/client-component/src/main/java/org/skywalking/apm/collector/client/elasticsearch/ElasticSearchClient.java) 对象。
* 第 77 至 82 行 ：创建 DAO 对象们，并调用 `#registerServiceImplementation()` **父类**方法，注册到 `services` 。

[`#start()`](https://github.com/YunaiV/skywalking/blob/7eecc004e69685fe6f5ce3b3dc4822475d6aa713/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/StorageModuleEsProvider.java#L85) **实现**方法，执行启动阶段逻辑。

* 第 90 行 ：调用 `ElasticSearchClient#initialize()` 方法，初始化 ZookeeperClient 。
* 第 93 至 94 行 ：创建 ElasticSearchStorageInstaller 对象，初始化存储组件的表。在 [「5.2.4 ElasticSearchStorageInstaller」](#) 详细解析。
* 第 100 至 102 行 ：创建 [`org.skywalking.apm.collector.storage.es.StorageModuleEsRegistration`](https://github.com/YunaiV/skywalking/blob/7eecc004e69685fe6f5ce3b3dc4822475d6aa713/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/StorageModuleEsRegistration.java) 对象，并注册信息到集群管理。在 [《SkyWalking 源码分析 —— Collector Cluster 集群管理》](http://www.iocoder.cn/SkyWalking/collector-cluster-module/?self) 有详细解析。
* 第 105 至 107 行 ：创建 [`org.skywalking.apm.collector.storage.es.StorageModuleEsNamingListener`](https://github.com/YunaiV/skywalking/blob/7eecc004e69685fe6f5ce3b3dc4822475d6aa713/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/StorageModuleEsNamingListener.java) 对象，并注册信息到集群管理。在 [《SkyWalking 源码分析 —— Collector Cluster 集群管理》](http://www.iocoder.cn/SkyWalking/collector-cluster-module/?self) 有详细解析。
* 第 110 至 111 行 ：创建 DataTTLKeeperTimer 对象。在 [「5.4 DataTTLKeeperTimer」](#) 详细解析。

[`#notifyAfterCompleted()`](https://github.com/YunaiV/skywalking/blob/7eecc004e69685fe6f5ce3b3dc4822475d6aa713/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/StorageModuleEsProvider.java#L114) **实现**方法，执行启动完成逻辑。

* 第 115 行 ：调用 `DataTTLKeeperTimer#start()` 方法，启动 DataTTLKeeperTimer 。在本文 [「5.4 DataTTLKeeperTimer」](#) 详细解析。

## 5.2 define 包

在 `collector-storage-es-provider` 项目结构图，我们看到一共有**两**个 `define` 包：

* `org.skywalking.apm.collector.storage.es.base.define` ，**系统**的 TableDefine 抽象类。
* `org.skywalking.apm.collector.storage.es.define` ，**业务**的 TableDefine 实现类。
    * **继承**系统的 TableDefine 抽象类。

### 5.2.1 ElasticSearchTableDefine

`org.skywalking.apm.collector.storage.es.base.define.ElasticSearchTableDefine` ，实现 TableDefine 接口，基于 Elasticsearch 的表定义**抽象类**。

* [`#type()`](https://github.com/YunaiV/skywalking/blob/7a4a409e266b953e523dca14a7ba88af07039f57/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/base/define/ElasticSearchTableDefine.java#L39) 方法，文档元数据 `_type` 字段，参见  [《Elasticsearch学习笔记》「_type」](http://geosmart.github.io/2016/07/22/Elasticsearch%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/#type) 。
* [`#refreshInterval()`](https://github.com/YunaiV/skywalking/blob/7a4a409e266b953e523dca14a7ba88af07039f57/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/base/define/ElasticSearchTableDefine.java#L46) **抽象**方法，文档索引刷新频率，参见 [《Elasticsearch: 权威指南 » 基础入门 » 分片内部原理 » 近实时搜索》「refresh API」](https://www.elastic.co/guide/cn/elasticsearch/guide/current/near-real-time.html#refresh-api)。

### 5.2.2 ElasticSearchColumnDefine

`org.skywalking.apm.collector.storage.es.base.define.ElasticSearchColumnDefine` ，实现 ColumnDefine 抽象类，基于 ES 的字段定义。

* [Type](https://github.com/peng-yongsheng/incubator-skywalking/blob/89c601ba386d30acb04b3713a90b52e6c0d501d8/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/apache/skywalking/apm/collector/storage/es/base/define/ElasticSearchColumnDefine.java#L32) **枚举**类：枚举 ES 字段类型。

### 5.2.3 业务 TableDefine 实现类

在 [`org.apache.skywalking.apm.collector.storage.es.define`](https://github.com/peng-yongsheng/incubator-skywalking/tree/89c601ba386d30acb04b3713a90b52e6c0d501d8/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/apache/skywalking/apm/collector/storage/es/define) **包**里，我们可以看到，**所有**基于 ES 的业务 TableDefine 实现类。例如：[ApplicationEsTableDefine](https://github.com/peng-yongsheng/incubator-skywalking/blob/89c601ba386d30acb04b3713a90b52e6c0d501d8/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/apache/skywalking/apm/collector/storage/es/define/ApplicationEsTableDefine.java) 。

整体 `#refreshInterval()` 方法返回的结果如下：

* 1 s
    * CpuMetricEsTableDefine
    * GCMetricEsTableDefine
    * MemoryMetricEsTableDefine
    * MemoryPoolMetricEsTableDefine
* 2 s
    * InstPerformanceEsTableDefine
    * NodeComponentEsTableDefine
    * NodeMappingEsTableDefine
    * NodeReferenceEsTableDefine
    * ServiceEntryEsTableDefine
    * ServiceReferenceEsTableDefine
* 2 s && [WriteRequest.RefreshPolicy.IMMEDIATE](https://static.javadoc.io/org.elasticsearch/elasticsearch/5.0.0/org/elasticsearch/action/support/WriteRequest.RefreshPolicy.html#IMMEDIATE)
    * 【WriteRequest.RefreshPolicy.IMMEDIATE】参见 [`ApplicationEsRegisterDAO#save(Application)`](https://github.com/YunaiV/skywalking/blob/6f925c180fbd1bb543fbf5bbf6fafe118f031d11/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/ApplicationEsRegisterDAO.java#L60) 方法
    * ApplicationEsTableDefine
    * InstanceEsTableDefine
    * ServiceNameEsTableDefine
* 5 s
    * GlobalTraceEsTableDefine
    * SegmentCostEsTableDefine
* 10 s
    * SegmentEsTableDefine

### 5.2.4 ElasticSearchStorageInstaller

> 友情提示：ElasticSearchStorageInstaller 主要是对 Elasticsearch Java API 的使用，所以不熟悉的胖友，可以 Google 下。

`org.skywalking.apm.collector.storage.es.base.define.ElasticSearchStorageInstaller` ，实现 StorageInstaller 抽象类， 基于 ES 存储安装器实现类。

* [`#defineFilter(List<TableDefine>)`](https://github.com/YunaiV/skywalking/blob/222024defff2d1a7647dbeb5811cad146c49a604/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/base/define/ElasticSearchStorageInstaller.java#L53) **实现**方法，过滤数组中，非 ElasticSearchTableDefine 的元素。
* [`#createTable(Client, TableDefine)`](https://github.com/YunaiV/skywalking/blob/222024defff2d1a7647dbeb5811cad146c49a604/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/base/define/ElasticSearchStorageInstaller.java#L63) **实现**方法，创建 Elasticsearch 索引。
    * 文档数据结构如下：
        * `_id` ：数据编号，String 类型。
        * `_type` ：`"type"` 。
        * `_index` ：TableDefine 定义的**表名**。
        * `source`  ：Data 数据。
    * 了解 Elasticsearch 的胖友可能有和笔者一样的疑惑，网络上很多文章把 `_index` 类比成关系数据库的 DB ，`_type` 类比成关系数据库的 Table ，和 SkyWalking 目前使用的方式**不一致**？
        * SkyWalking [彭勇升](https://github.com/peng-yongsheng) ：`_index`和 `_type` 是 ES 特有的，考虑其他数据库接入，所以没有用他这个特性。
        * SkyWalking QQ交流群( 392443393 ) ，[小心](#) 群友 ：`_type` 本来就没做物理隔离，Lucene 层面也不存在，ES 6.x 已经废弃了。
        * [《Elasticsearch 6.0 将移除 Type》](https://elasticsearch.cn/article/158)
* [`#deleteTable(Client, TableDefine)`](https://github.com/YunaiV/skywalking/blob/222024defff2d1a7647dbeb5811cad146c49a604/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/base/define/ElasticSearchStorageInstaller.java#L131) **实现**方法，删除 Elasticsearch 索引。
* [`#isExists(Client, TableDefine)`](https://github.com/YunaiV/skywalking/blob/222024defff2d1a7647dbeb5811cad146c49a604/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/base/define/ElasticSearchStorageInstaller.java#L141) **实现**方法，判断 Elasticsearch 索引是否存在。
* 在方法里，笔者添加了一些 API 的说明，不熟悉的胖友，可以仔细阅读理解。


## 5.3 dao 包

在 `collector-storage-es-provider` 项目结构图，我们看到一共有**两**个 `dao` 包：

* `org.skywalking.apm.collector.storage.es.base.dao` ，**系统**的 DAO 抽象类。
* `org.skywalking.apm.collector.storage.es.dao` ，**业务**的 DAO 实现类。
    * **继承**系统的 DAO 抽象类。

### 5.3.1 EsDAO

`org.skywalking.apm.collector.storage.es.base.dao.EsDAO` ，实现 AbstractDAO 抽象类，基于 ES 的 DAO **抽象类**。

* [`#getMaxId(indexName, columnName)`](https://github.com/YunaiV/skywalking/blob/6f925c180fbd1bb543fbf5bbf6fafe118f031d11/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/base/dao/EsDAO.java#L49) 方法，获得索引名的指定字段的**最大值**。
* [`#getMinId(indexName, columnName)`](https://github.com/YunaiV/skywalking/blob/6f925c180fbd1bb543fbf5bbf6fafe118f031d11/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/base/dao/EsDAO.java#L75) 方法，获得索引名的指定字段的**最小值**。

### 5.3.2 BatchEsDAO

`org.skywalking.apm.collector.storage.es.base.dao.BatchEsDAO` ，实现 IBatchDAO 接口，继承 EsDAO 抽象类，基于 ES 批量操作 DAO 实现类。

* [`#batchPersistence(List<?>)`](https://github.com/YunaiV/skywalking/blob/6f925c180fbd1bb543fbf5bbf6fafe118f031d11/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/base/dao/BatchEsDAO.java#L46) **实现**方法，将 `org.elasticsearch.action.index.IndexRequestBuilder` 和 `org.elasticsearch.action.index.UpdateRequestBuilder` 数组，创建成 `org.elasticsearch.action.bulk.BulkRequestBuilder` 对象，批量持久化。
    * IndexRequestBuilder 和 UpdateRequestBuilder 的创建，在 [「5.3.3 业务 DAO 实现类」](#) 会看到。

### 5.3.3 业务 DAO 实现类

在 [`org.apache.skywalking.apm.collector.storage.es.dao`](https://github.com/YunaiV/skywalking/tree/6f925c180fbd1bb543fbf5bbf6fafe118f031d11/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao) **包**里，我们可以看到，**所有**基于 ES 的业务 DAO 实现类。

实现代码易懂，胖友可以自己阅读。良心如我们，按照 DAO 的业务用途，推荐例子如下：

* Cache ：[ApplicationEsCacheDAO](https://github.com/YunaiV/skywalking/blob/6f925c180fbd1bb543fbf5bbf6fafe118f031d11/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/ApplicationEsCacheDAO.java)
* Register ：[ApplicationEsRegisterDAO](https://github.com/YunaiV/skywalking/blob/6f925c180fbd1bb543fbf5bbf6fafe118f031d11/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/ApplicationEsRegisterDAO.java)
* Persistence ：[SegmentEsPersistenceDAO](https://github.com/YunaiV/skywalking/blob/6f925c180fbd1bb543fbf5bbf6fafe118f031d11/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/SegmentEsPersistenceDAO.java)
    * 此处可见 IndexRequestBuilder 和 UpdateRequestBuilder 的创建。
* UI ：[SegmentEsUIDAO](https://github.com/YunaiV/skywalking/blob/6f925c180fbd1bb543fbf5bbf6fafe118f031d11/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/SegmentEsUIDAO.java)

## 5.4 DataTTLKeeperTimer

`org.skywalking.apm.collector.storage.es.DataTTLKeeperTimer` ，过期数据删除**定时器**。通过该定时器，只保留 N 天内的数据。

* [`#start()`](https://github.com/YunaiV/skywalking/blob/efe2967813b706ae97901d7620fef9d7f975e745/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/DataTTLKeeperTimer.java#L47) 方法，启动定时任务。
    * 第 49 行：创建延迟 1 小时，每 8 小时执行一次 `#delete()` 方法的定时任务。目前该行代码被注释，胖友可以等待 SkyWallking 5.0 版本的发布。
* [`#delete()`](https://github.com/YunaiV/skywalking/blob/efe2967813b706ae97901d7620fef9d7f975e745/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/DataTTLKeeperTimer.java#L53) 方法，删除过期数据。
    * 第 54 至 66 行：计算删除的开始与结束时间，即指定时间的**前一天**。例如，2017-12-23 执行时，删除 2017-12-16 那天的数据。
    * 第 69 行：调用 [`#deleteJVMRelatedData(startTimestamp, endTimestamp)`](https://github.com/YunaiV/skywalking/blob/efe2967813b706ae97901d7620fef9d7f975e745/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/DataTTLKeeperTimer.java#L73) 方法，删除 JVM 相关的数据。
    * 第 70 行：调用 [`#deleteTraceRelatedData(startTimestamp, endTimestamp)`](https://github.com/YunaiV/skywalking/blob/efe2967813b706ae97901d7620fef9d7f975e745/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/DataTTLKeeperTimer.java#L87) 方法，删除 Trace 相关的数据。

如下是**不会删除**的数据的表：

* Application
* Instance
* ServiceName
* ServiceEntry

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

😈 有种自己把简单的东西写的太复杂了，悲伤。

胖友望见谅。

胖友，分享一波朋友圈可好。


