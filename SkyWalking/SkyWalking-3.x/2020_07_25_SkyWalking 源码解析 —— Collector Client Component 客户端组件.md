title: SkyWalking 源码分析 —— Collector Client Component 客户端组件
date: 2020-07-25
tags:
categories: SkyWalking
permalink: SkyWalking/collector-client-component
wechat_url:
toutiao_url:

-------

摘要: 原创出处 http://www.iocoder.cn/SkyWalking/collector-client-component/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 SkyWalking 3.2.6 正式版**

- [1. 概述](http://www.iocoder.cn/SkyWalking/collector-client-component/)
- [2. Client](http://www.iocoder.cn/SkyWalking/collector-client-component/)
- [3. ElasticSearchClient](http://www.iocoder.cn/SkyWalking/collector-client-component/)
- [4. GRPCClient](http://www.iocoder.cn/SkyWalking/collector-client-component/)
- [5. H2Client](http://www.iocoder.cn/SkyWalking/collector-client-component/)
- [6. RedisClient](http://www.iocoder.cn/SkyWalking/collector-client-component/)
- [7. ZookeeperClient](http://www.iocoder.cn/SkyWalking/collector-client-component/)
- [666. 彩蛋](http://www.iocoder.cn/SkyWalking/collector-client-component/)

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

本文主要分享 **SkyWalking Collector Client Component 客户端组件**。Collector 通过客户端，和其他服务进行通信，例如 Elastic Search 、Zookeeper 、H2 等等。

Client Component 在 SkyWalking 架构图处于如下位置( **红框** ) ：

> FROM https://github.com/apache/incubating-skywalking  
> ![](http://www.iocoder.cn/images/SkyWalking/2020_07_25/01.jpeg)

下面我们来看看整体的项目结构，如下图所示 ：

![](http://www.iocoder.cn/images/SkyWalking/2020_07_25/02.png)

OK，我们从**接口到实现**的顺序进行分享。

# 2. Client

[`org.skywalking.apm.collector.client.Client`](https://github.com/YunaiV/skywalking/blob/c546a9a4d4588d99bf532da519ae721ef60b918e/apm-collector/apm-collector-component/client-component/src/main/java/org/skywalking/apm/collector/client/Client.java) ，客户端**接口**。其定义接口方法如下：

* `#initialize()` 方法，初始化客户端。
* `#shutdown()` 方法，关闭客户端。

Client 的实现类，如下类图：

![](http://www.iocoder.cn/images/SkyWalking/2020_07_25/03.png)

# 3. ElasticSearchClient

[`org.skywalking.apm.collector.client.elasticsearch.ElasticSearchClient`](https://github.com/YunaiV/skywalking/blob/001f700612ad52bc1eb1a278bc0e2ff9e5330df8/apm-collector/apm-collector-component/client-component/src/main/java/org/skywalking/apm/collector/client/elasticsearch/ElasticSearchClient.java) ，Elastic Search 客户端。

基于 `org.elasticsearch.client.transport` 的 `5.5.0` 版本，封装 SkyWalking 需要的 Elastic Search 操作。目前用于 [`collector-storage-es-provider`](https://github.com/YunaiV/skywalking/blob/001f700612ad52bc1eb1a278bc0e2ff9e5330df8/apm-collector/apm-collector-storage/collector-storage-es-provider) 模块。

# 4. GRPCClient

[`org.skywalking.apm.collector.client.grpc.GRPCClient`](https://github.com/YunaiV/skywalking/blob/001f700612ad52bc1eb1a278bc0e2ff9e5330df8/apm-collector/apm-collector-component/client-component/src/main/java/org/skywalking/apm/collector/client/grpc/GRPCClient.java) ，gRPC 客户端。

基于 `io.grpc.grpc-core` 的 `1.8.0` 版本，封装 SkyWalking 需要的 gRPC 操作。目前用于 [`collector-remote-grpc-provider`](https://github.com/YunaiV/skywalking/tree/001f700612ad52bc1eb1a278bc0e2ff9e5330df8/apm-collector/apm-collector-remote/collector-remote-grpc-provider) 模块。

# 5. H2Client

[`org.skywalking.apm.collector.client.h2.H2Client`](https://github.com/YunaiV/skywalking/blob/001f700612ad52bc1eb1a278bc0e2ff9e5330df8/apm-collector/apm-collector-component/client-component/src/main/java/org/skywalking/apm/collector/client/h2/H2Client.java) ，H2 数据库客户端。

基于 `com.h2database.h2` 的 `1.4.196` 版本，封装 SkyWalking 需要的 H2 数据库操作。目前用于 [`collector-storage-h2-provider`](https://github.com/YunaiV/skywalking/tree/001f700612ad52bc1eb1a278bc0e2ff9e5330df8/apm-collector/apm-collector-storage/collector-storage-h2-provider) / [`collector-cluster-standalone-provider`](https://github.com/YunaiV/skywalking/tree/001f700612ad52bc1eb1a278bc0e2ff9e5330df8/apm-collector/apm-collector-cluster/collector-cluster-standalone-provider) 模块。

# 6. RedisClient

[`org.skywalking.apm.collector.client.redis.RedisClient`](https://github.com/YunaiV/skywalking/blob/001f700612ad52bc1eb1a278bc0e2ff9e5330df8/apm-collector/apm-collector-component/client-component/src/main/java/org/skywalking/apm/collector/client/redis/RedisClient.java) ，Redis 客户端。

基于 `redis.clients.jedis` 的 `2.9.0` 版本，封装 SkyWalking 需要的 Reids 操作。预计未来用于 [`collector-cluster-redis-provider`](https://github.com/YunaiV/skywalking/tree/001f700612ad52bc1eb1a278bc0e2ff9e5330df8/apm-collector/apm-collector-cluster/collector-cluster-redis-provider) 模块。

# 7. ZookeeperClient

[`org.skywalking.apm.collector.client.zookeeper.ZookeeperClient`](https://github.com/YunaiV/skywalking/blob/001f700612ad52bc1eb1a278bc0e2ff9e5330df8/apm-collector/apm-collector-component/client-component/src/main/java/org/skywalking/apm/collector/client/zookeeper/ZookeeperClient.java) ，Zookeeper 客户端。

基于 `org.apache.zookeeper.zookeeper` 的 `3.4.10` 版本，封装 SkyWalking 需要的 Zookeeper 操作。预计未来用于 [`collector-cluster-zookeeper-provider`](https://github.com/YunaiV/skywalking/tree/001f700612ad52bc1eb1a278bc0e2ff9e5330df8/apm-collector/apm-collector-cluster/collector-cluster-zookeeper-provider) 模块。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

哈哈哈，是不是看的很有成就感( 笔者又在水更了 )。

不要方，下面还有一篇水更。

胖友，分享个朋友圈可好？


