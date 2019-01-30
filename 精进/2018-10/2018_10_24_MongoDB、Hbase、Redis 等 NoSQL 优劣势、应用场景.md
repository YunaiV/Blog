title: MongoDB、HBase、Redis 等 NoSQL 优劣势、应用场景
date: 2018-10-24
tags:
categories: 精进
permalink: Fight/Advantages-and-disadvantages-of-NoSQL-such-as-MongoDB-HBase-Redis-and-application-scenarios
author: 佚名
from_url: http://database.51cto.com/art/201808/582357.htm
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247485452&idx=2&sn=ac20250c11cfff7744ed877b665abcd2&chksm=fa4977bdcd3efeabcec16a46467a1073c59c2aa22db21055dab5b06d7a821a514999b9864901&token=982309024&lang=zh_CN#rd

-------

摘要: 原创出处 http://database.51cto.com/art/201808/582357.htm 「佚名」欢迎转载，保留摘要，谢谢！

- [NoSQL的四大种类](http://www.iocoder.cn/Fight/Advantages-and-disadvantages-of-NoSQL-such-as-MongoDB-HBase-Redis-and-application-scenarios/)
- [MongoDB](http://www.iocoder.cn/Fight/Advantages-and-disadvantages-of-NoSQL-such-as-MongoDB-HBase-Redis-and-application-scenarios/)
- [HBase](http://www.iocoder.cn/Fight/Advantages-and-disadvantages-of-NoSQL-such-as-MongoDB-HBase-Redis-and-application-scenarios/)
- [Redis](http://www.iocoder.cn/Fight/Advantages-and-disadvantages-of-NoSQL-such-as-MongoDB-HBase-Redis-and-application-scenarios/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# NoSQL的四大种类

NoSQL数据库在整个数据库领域的江湖地位已经不言而喻。在大数据时代，虽然RDBMS很优秀，但是面对快速增长的数据规模和日渐复杂的数据模型，RDBMS渐渐力不从心，无法应对很多数据库处理任务，这时NoSQL凭借易扩展、大数据量和高性能以及灵活的数据模型成功的在数据库领域站稳了脚跟。

目前大家基本认同将NoSQL数据库分为四大类：键值存储数据库，文档型数据库，列存储数据库和图形数据库，其中每一种类型的数据库都能够解决关系型数据不能解决的问题。在实际应用中，NoSQL数据库的分类界限其实没有那么明显，往往会是多种类型的组合体。

[![](http://static.iocoder.cn/3fd918cf4db2b7334c9a560e9fd57c4f)](http://static.iocoder.cn/3fd918cf4db2b7334c9a560e9fd57c4f)

主流nosql的详解：MongoDB、Hbase、Redis

[![](http://static.iocoder.cn/acf28e09c3beac8e59792b5984739dae)](http://static.iocoder.cn/acf28e09c3beac8e59792b5984739dae)

# MongoDB

MongoDB 是一个高性能，开源，无模式的文档型数据库，开发语言是C++。它在许多场景下可用于替代统的关系型数据库或键/值存储方式。

**1.MongoDB特点**

*  所用语言：C++
*  特点：保留了SQL一些友好的特性（查询，索引）。
*  使用许可： AGPL（发起者： Apache）
*  协议： Custom, binary（ BSON）
*  Master/slave复制（支持自动错误恢复，使用 sets 复制）
*  内建分片机制
*  支持 javascript表达式查询
*  可在服务器端执行任意的 javascript函数
*  update-in-place支持比CouchDB更好
*  在数据存储时采用内存到文件映射
*  对性能的关注超过对功能的要求
*  建议最好打开日志功能（参数 --journal）
*  在32位操作系统上，数据库大小限制在约2.5Gb
*  空数据库大约占 192Mb
*  采用 GridFS存储大数据或元数据（不是真正的文件系统）

**2.MongoDB优点：**

1）更高的写负载，MongoDB拥有更高的插入速度。

2）处理很大的规模的单表，当数据表太大的时候可以很容易的分割表。

3）高可用性，设置M-S不仅方便而且很快，MongoDB还可以快速、安全及自动化的实现节点 （数据中心）故障转移。

4）快速的查询，MongoDB支持二维空间索引，比如管道，因此可以快速及精确的从指定位置 获取数据。MongoDB在启动后会将数据库中的数据以文件映射的方式加载到内存中。如果内 存资源相当丰富的话，这将极大地提高数据库的查询速度。

5）非结构化数据的爆发增长，增加列在有些情况下可能锁定整个数据库，或者增加负载从而 导致性能下降，由于MongoDB的弱数据结构模式，添加1个新字段不会对旧表格有任何影响， 整个过程会非常快速。

**3.MongoDB缺点：**

1）不支持事务。

2）MongoDB占用空间过大 。

3）MongoDB没有成熟的维护工具。

**4.MongoDB应用场景**

1）适用于实时的插入、更新与查询的需求，并具备应用程序实时数据存储所需的复制及高度伸缩性；

2） 非常适合文档化格式的存储及查询；

3）高伸缩性的场景：MongoDB 非常适合由数十或者数百台服务器组成的数据库。

4）对性能的关注超过对功能的要求。

# HBase

HBase 是 Apache Hadoop 中的一个子项目，属于 bigtable 的开源版本，所实现的语言为Java（故依赖 Java SDK）。HBase 依托于 Hadoop 的 HDFS（分布式文件系统）作为最基本存储基础单元。

**1.HBase 特点：**

*  所用语言： Java
*  特点：支持数十亿行X上百万列
*  使用许可： Apache
*  协议：HTTP/REST （支持 Thrift，见编注4）
*  在 BigTable之后建模
*  采用分布式架构 Map/reduce
*  对实时查询进行优化
*  高性能 Thrift网关
*  通过在server端扫描及过滤实现对查询操作预判
*  支持 XML, Protobuf, 和binary的HTTP
*  Cascading, hive, and pig source and sink modules
*  基于 Jruby（ JIRB）的shell
*  对配置改变和较小的升级都会重新回滚
*  不会出现单点故障
*  堪比MySQL的随机访问性能

**3. HBase 优点**

1） 存储容量大，一个表可以容纳上亿行，上百万列；

2）可通过版本进行检索，能搜到所需的历史版本数据；

3）负载高时，可通过简单的添加机器来实现水平切分扩展，跟Hadoop的无缝集成保障了其数据可靠性（HDFS）和海量数据分析的高性能（MapReduce）；

4）在第3点的基础上可有效避免单点故障的发生。

**4.HBase 缺点**

1. 基于Java语言实现及Hadoop架构意味着其API更适用于Java项目；

2. node开发环境下所需依赖项较多、配置麻烦（或不知如何配置，如持久化配置），缺乏文档；

3. 占用内存很大，且鉴于建立在为批量分析而优化的HDFS上，导致读取性能不高；

4. API相比其它 NoSql 的相对笨拙。

**5.HBase 适用场景**

1）bigtable类型的数据存储；

2）对数据有版本查询需求；

3）应对超大数据量要求扩展简单的需求。

# Redis

Redis 是一个开源的使用ANSI C语言编写、支持网络、可基于内存亦可持久化的日志型、Key-Value数据库，并提供多种语言的API。目前由VMware主持开发工作。

1.Redis 特点：

*  所用语言：C/C++
*  特点：运行异常快
*  使用许可： BSD
*  协议：类 Telnet
*  有硬盘存储支持的内存数据库，
*  但自2.0版本以后可以将数据交换到硬盘（注意， 2.4以后版本不支持该特性！）
*  Master-slave复制（见编注3）
*  虽然采用简单数据或以键值索引的哈希表，但也支持复杂操作，例如 ZREVRANGEBYSCORE。
*  INCR & co （适合计算极限值或统计数据）
*  支持 sets（同时也支持 union/diff/inter）
*  支持列表（同时也支持队列；阻塞式 pop操作）
*  支持哈希表（带有多个域的对象）
*  支持排序 sets（高得分表，适用于范围查询）
*  Redis支持事务
*  支持将数据设置成过期数据（类似快速缓冲区设计）
*  Pub/Sub允许用户实现消息机制

**2. Redis 优势**

1）非常丰富的数据结构；

2）Redis提供了事务的功能，可以保证一串 命令的原子性，中间不会被任何操作打断；

3）数据存在内存中，读写非常的高速，可以达到10w/s的频率。

**3.Redis 缺点**

1） Redis3.0后才出来官方的集群方案，但仍存在一些架构上的问题；

2）持久化功能体验不佳——通过快照方法实现的话，需要每隔一段时间将整个数据库的数据写到磁盘上，代价非常高；而aof方法只追踪变化的数据，类似于mysql的binlog方法，但追加log可能过大，同时所有操作均要重新执行一遍，恢复速度慢；

3）由于是内存数据库，所以，单台机器，存储的数据量，跟机器本身的内存大小。虽然redis本身有key过期策略，但是还是需要提前预估和节约内存。如果内存增长过快，需要定期删除数据。

**4.Redis 应用场景：**

最佳应用场景：适用于数据变化快且数据库大小可遇见（适合内存容量）的应用程序。

例如：微博、数据分析、实时数据搜集、实时通讯等。
