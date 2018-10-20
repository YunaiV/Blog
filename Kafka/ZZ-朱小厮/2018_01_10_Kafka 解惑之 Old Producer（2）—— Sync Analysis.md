title: Kafka 解惑之 Old Producer（2）—— Sync Analysis
date: 2018-01-10
tag: 
categories: Kafka
permalink: Kafka/old-producer-sync-analysis
author: 朱小厮
from_url: https://blog.csdn.net/u013256816/article/details/79202691
wechat_url: 

-------

摘要: 原创出处 https://blog.csdn.net/u013256816/article/details/79202691 「朱小厮」欢迎转载，保留摘要，谢谢！

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

一下子扩展的有点多，我们还是先回到DefaultEventHandler上来，当调用producer.send方法发送消息的时候，紧接着就是调用DefaultEventHandler的handle方法。下面是handle方法的主要内容，虽然行数有点多，但是这是Producer中最最核心的一块，需要反复研磨，方能一探究竟：

```java
def handle(events: Seq[KeyedMessage[K,V]]) {
  val serializedData = serialize(events)
  var outstandingProduceRequests = serializedData
  var remainingRetries = config.messageSendMaxRetries + 1
  val correlationIdStart = correlationId.get()
  while (remainingRetries > 0 && outstandingProduceRequests.nonEmpty) {
    topicMetadataToRefresh ++= outstandingProduceRequests.map(_.topic)
    if (topicMetadataRefreshInterval >= 0 &&
        Time.SYSTEM.milliseconds - lastTopicMetadataRefreshTime > topicMetadataRefreshInterval) {
      CoreUtils.swallowError(brokerPartitionInfo.updateInfo(topicMetadataToRefresh.toSet, correlationId.getAndIncrement))
      sendPartitionPerTopicCache.clear()
      topicMetadataToRefresh.clear
      lastTopicMetadataRefreshTime = Time.SYSTEM.milliseconds
    }
    outstandingProduceRequests = dispatchSerializedData(outstandingProduceRequests)
    if (outstandingProduceRequests.nonEmpty) {
      Thread.sleep(config.retryBackoffMs)
      CoreUtils.swallowError(brokerPartitionInfo.updateInfo(outstandingProduceRequests.map(_.topic).toSet, correlationId.getAndIncrement))
      sendPartitionPerTopicCache.clear()
      remainingRetries -= 1
      producerStats.resendRate.mark()
    }
  }
}
```

注意handle方法的参数是个Seq[KeyedMessage]类型的，而不是KeyedMessage。虽然Demo中用的只是单个KeyedMessage，最后调用底层的handle方法都是转换为Seq类型，你可以把Seq看成是java中的List，在Scala中表示序列，指的是一类具有一定长度的可迭代访问的对象，其中每个元素均带有一个从0开始计数的固定索引位置。

这个handle方法中首先是调用serialize(events)方法对消息进行序列化操作，这个容易理解，就是通过serializer.class参数指定的序列化类进行序列化。

其次获取所发送消息对应的元数据信息，然后将一坨消息（也有可能是一条）转换为HashMap[Int, collection.mutable.Map[TopicAndPartition, Seq[KeyedMessage[K,Message]]]]格式，其中key:Int表示broker的id，value是TopicAndPartition与消息集的Map，对应的方法为dispatchSerializedData()。因为客户端发消息是发到对应的broker上，所以要对每个消息找出对应的leader副本所在的broker的位置，然后将要发送的消息集分类，每个broker对应其各自所要接收的消息。而TopicAndPartition是针对broker上的存储层的，每个TopicAndPartition对应特定的当前的存储文件（Segment文件），将消息写入到存储文件中。

获取元数据信息并不是每次发送消息都要向metadata.broker.list所指定地址中的服务索要拉取，而是向缓存中的元数据进行拉取，拉取失败后才向metadata.broker.list所指定地址中的服务发送元数据更新的请求进行拉取。很多朋友会把metadata.broker.list看成是broker的地址，这个不完全正确，官网解释：

> This is for bootstrapping and the producer will only use it for getting metadata (topics, partitions and replicas). The socket connections for sending the actual data will be established based on the broker information returned in the metadata. The format is host1:port1,host2:port2, and the list can be a subset of brokers or a VIP pointing to a subset of brokers.

因为这个地址只提供给客户端拉取元数据信息之用，而剩下的动作比如发送消息是通过与元数据信息中的broker地址建立连接之后再进行操作，这也就意味着metadata.broker.list可以和broker的真正地址没有任何交集。你完全可以为metadata.broker.list配置一个“伪装”接口地址，这个接口配合kafka的传输格式并提供相应的元数据信息，这样方便集中式的配置管理（可以集成到配置中心中）。为了简化说明，我们姑且可以狭义的认为metadata.broker.list指的就是kafka broker的地址。

缓存中的元数据每隔topic.metadata.refresh.interval.ms才去broker拉取元数据信息，可以参考上面大段源码中的if语句：

```java
 if (topicMetadataRefreshInterval >= 0 &&
        Time.SYSTEM.milliseconds - lastTopicMetadataRefreshTime > topicMetadataRefreshInterval)
```

topic.metadata.refresh.interval.ms参数的默认值是600*1000ms，也就是10分钟。如果设置为0，则每次发送消息时都要先向broker拉取元数据信息；如果设置为负数，那么只有在元数据获取失败的情况下才会请求元数据信息。由于这个老版的Scala的Producer请求元数据和发送消息是在同一个线程中完成的，所以此处会有延迟的隐患，具体的笔者会在后面的案例分析环节为大家详细介绍。

接下去所要做的工作就是查看是否需要压缩，如果客户端设置了压缩，则根据compression.type参数配置的压缩方式对消息进行压缩处理。0.8.2.x版本支持gzip和snappy的压缩方式，1.0.0版本还支持lz4的压缩方式。compression.type参数的默认值值none，即不需要压缩。

最后根据brokerId分组发送消息。这个分组发送的过程就与ProducerPool有关了，我们前面提到在实例化Producer的时候引入了DefaultEventHandler和ProducerPool。这个ProducerPool保存的是生产者和broker的连接，每个连接对应一个SyncProducer对象。SyncProducer包装了NIO网络层的操作，每个SyncProducer都是一个与对应broker的socket连接，是真正发送消息至broker中的执行者。

```java
@deprecated("This class has been deprecated and will be removed in a future release.", "0.10.0.0")
class ProducerPool(val config: ProducerConfig) extends Logging {
  private val syncProducers = new HashMap[Int, SyncProducer]
```

当调用最上层的send方法发送消息的时候，下面的执行顺序为DefaultEventHandler.handle()->DefaultEventHandler.dispatchSerializedData()->DefaultEventHandler.send()。在底层的DefaultEventHandler.send方法定义为：

```java
private def send(brokerId: Int, messagesPerTopic: collection.mutable.Map[TopicAndPartition, ByteBufferMessageSet])
```

这个方法就需要根据brokerId从ProducerPool中的HashMap中找到对应SyncProducer，然后在将“messagesPerTopic: collection.mutable.Map[TopicAndPartition, ByteBufferMessageSet]”这个消息发送到SyncProducer对应的broker上。如果在获取缓存中的元数据失败的时候就需要重新向broker拉取元数据，或者定时（topic.metadata.refresh.interval.ms）向broker端请求元数据的数据，都会有可能更新ProducerPool的信息，对应的方法为ProducerPool.updateProducer()：

```java
def updateProducer(topicMetadata: Seq[TopicMetadata]) {
  val newBrokers = new collection.mutable.HashSet[BrokerEndPoint]
  topicMetadata.foreach(tmd => {
    tmd.partitionsMetadata.foreach(pmd => {
      if(pmd.leader.isDefined) {
        newBrokers += pmd.leader.get
      }
    })
  })
  lock synchronized {
    newBrokers.foreach(b => {
      if(syncProducers.contains(b.id)){
        syncProducers(b.id).close()
        syncProducers.put(b.id, ProducerPool.createSyncProducer(config, b))
      } else
        syncProducers.put(b.id, ProducerPool.createSyncProducer(config, b))
    })
  }
}
```

会Java的读者看这段代码的时候应该能看出来个90%以上，解释下这段代码：首先是找到更新的元数据中所有的brorker（更具体的来说是broker的id、主机地址host和端口号port三元组信息）；之后在查到原有的ProducerPool中是否有相应的SyncProducer，如果有则关闭之后再重新建立；如果没有则新建。SyncProducer底层是阻塞式的NIO，所以关闭再建立会有一定程度上的开销，相关细节如下：

```java
channel = SocketChannel.open()
if(readBufferSize > 0)
  channel.socket.setReceiveBufferSize(readBufferSize)
if(writeBufferSize > 0)
  channel.socket.setSendBufferSize(writeBufferSize)
channel.configureBlocking(true)
channel.socket.setSoTimeout(readTimeoutMs)
channel.socket.setKeepAlive(true)
channel.socket.setTcpNoDelay(true)
channel.socket.connect(new InetSocketAddress(host, port), connectTimeoutMs)
```

玩过NIO的读者对这段代码相比很是熟络，虽然是scala版的。如果没有接触过NIO，那么可以先看看这一篇：[攻破JAVA NIO技术壁垒](http://blog.csdn.net/u013256816/article/details/51457215)。

说道这里我们用一副结构图来说明下Old Producer的大致脉络（注：图中的所有操作都是在一个线程中执行的）：
![这里写图片描述](http://static.iocoder.cn/csdn/20180130101841386?)

# 666. 彩蛋

如果你对 Kafka 并发感兴趣，欢迎加入我的知识一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)