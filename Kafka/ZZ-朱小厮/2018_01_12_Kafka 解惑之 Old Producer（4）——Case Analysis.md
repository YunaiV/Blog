title: Kafka 解惑之 Old Producer（4）——Case Analysis
date: 2018-01-12
tag: 
categories: Kafka
permalink: Kafka/old-producer-case-analysis
author: 朱小厮
from_url: https://blog.csdn.net/u013256816/article/details/79237769
wechat_url: 

-------

摘要: 原创出处 https://blog.csdn.net/u013256816/article/details/79237769 「朱小厮」欢迎转载，保留摘要，谢谢！

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

在前面三篇文章中详细了解了Old Producer的内容，本文主要通过一个实际应用案例来加深各位对Old Producer的理解。

问题描述：线上由多台Kafka Broker组成的集群（Producer的metadata.broker.list参数设置的是所有broker的地址+端口号的列表），版本号为0.8.2.x，当一台Kafka Broker的硬盘发生故障导致系统崩溃，由于Kafka的HA的作用线上业务无明显异常，发送方和消费方的流量也与之前持稳，但是后面监测到每隔10分钟左右就有少量的消息发送的时延很大，而且有ERROR告警报出。

```java
2018-01-30 00:53:20 -[ERROR] - [fetching topic metadata for topics [Set(hidden-topic)] from broker [ArrayBuffer(id:0,host:xxx.xxx.xxx.xxx,port:9092)] failed] - [kafka.utils.Logging$$anonfun$swallowError$1:106]
kafka.common.KafkaException: fetching topic metadata for topics [Set(hidden-topic)] from broker [ArrayBuffer(id:0,host:xxx.xxx.xxx.xxx,port:9092)] failed
	at kafka.client.ClientUtils$.fetchTopicMetadata(ClientUtils.scala:72)
	at kafka.producer.BrokerPartitionInfo.updateInfo(BrokerPartitionInfo.scala:82)
	at kafka.producer.async.DefaultEventHandler$$anonfun$handle$1.apply$mcV$sp(DefaultEventHandler.scala:67)
    at kafka.utils.Utils$.swallow(Utils.scala:172)
    at kafka.utils.Logging$class.swallowError(Logging.scala:106)
    at kafka.utils.Utils$.swallowError(Utils.scala:45)
    at kafka.producer.async.DefaultEventHandler.handle(DefaultEventHandler.scala:67)
    at kafka.producer.Producer.send(Producer.scala:77)
    at kafka.javaapi.producer.Producer.send(Producer.scala:33)
```

通过上面的异常栈我们可以发现在获取元数据的时候（kafka.client.ClientUtils$.fetchTopicMetadata）发生了异常，其实如果你对Kafka的配置参数足够了解的话，看到10分钟这个数值就可以联想到600*1000的某个参数，也就是topic.metadata.refresh.interval.ms。在DefaultEventHandler的handle()方法中，在调用dispatchSerializedData()方法预处理并发送消息之前就会有下面的一个if判断语言用来判断当前是否需要更新元数据：

```java
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
```

上面代码中的topicMetadataRefreshInterval指的就是topic.metadata.refresh.interval.ms参数。如果topic.metadata.refresh.interval.ms这个参数设置为0，那么就意味着每次发送消息之前都需要拉取并更新元数据信息，更新元数据信息之后还要更新ProducerPool的内容，包括重建SyncProducer，那么这一番操作必然会有一定的延迟，当然这个延迟没有本文中所述问题中的延迟那么大。如果topic.metadata.refresh.interval.ms这个参数设置为负数，那么这个if条件语句就不能成立，也就是不会有定时更新元数据的操作，只有在获取元数据信息失败是才会请求完整的元数据信息。

本文中所述问题中采用的topic.metadata.refresh.interval.ms参数设置的是默认值大小，那么问题中还有一个细节：只有少量消息发送超时。为了进一步的一探究竟，我们来继续深究。当定时更新元数据信息条件满足时，就会调用brokerPartitionInfo.updateInfo的方法，更进取之后，实际上内部调用的是：ClientUtils.fetchTopicMetadata(topics, brokers, producerConfig, correlationId)来请求元数据信息，并获得TopicMetadataResponse来做更新。

TopicMetadataResponse中包含有broker的id、host、port，还有topic、topic中的partition、partition所对应的leader、AR和ISR等等，有兴趣的同学可以进一步翻阅kafka.api.TopicMetadataResponse这个类，主体代码只有30行左右，只要学一点Scala的构造函数相关的只是就能看懂。

我们回过头来再来进一步分析ClientUtils.fetchTopicMetadata这个方法，详细代码如下：

```java
def fetchTopicMetadata(topics: Set[String], brokers: Seq[BrokerEndPoint], producerConfig: ProducerConfig, correlationId: Int): TopicMetadataResponse = {
  var fetchMetaDataSucceeded: Boolean = false
  var i: Int = 0
  val topicMetadataRequest = new TopicMetadataRequest(TopicMetadataRequest.CurrentVersion, correlationId, producerConfig.clientId, topics.toSeq)
  var topicMetadataResponse: TopicMetadataResponse = null
  var t: Throwable = null
  // shuffle the list of brokers before sending metadata requests so that most requests don't get routed to the
  // same broker
  val shuffledBrokers = Random.shuffle(brokers)
  while(i < shuffledBrokers.size && !fetchMetaDataSucceeded) {
    val producer: SyncProducer = ProducerPool.createSyncProducer(producerConfig, shuffledBrokers(i))
    info("Fetching metadata from broker %s with correlation id %d for %d topic(s) %s".format(shuffledBrokers(i), correlationId, topics.size, topics))
    try {
      topicMetadataResponse = producer.send(topicMetadataRequest)
      fetchMetaDataSucceeded = true
    }
    catch {
      case e: Throwable =>
        warn("Fetching topic metadata with correlation id %d for topics [%s] from broker [%s] failed"
          .format(correlationId, topics, shuffledBrokers(i).toString), e)
        t = e
    } finally {
      i = i + 1
      producer.close()
    }
  }
  if(!fetchMetaDataSucceeded) {
    throw new KafkaException("fetching topic metadata for topics [%s] from broker [%s] failed".format(topics, shuffledBrokers), t)
  } else {
    debug("Successfully fetched metadata for %d topic(s) %s".format(topics.size, topics))
  }
  topicMetadataResponse
}
```

fetchTopicMetadata方法参数列表中brokers代表metadata.broker.list所配置的地址列表。可以看到方法中首先建立TopicMetadataRequest的请求，然后从brokers中随机挑选（做一个shuffle，然后从列表中的第一个开始取，也就是相当于随机）一个broker建立SyncProducer并发送TopicMetadataRequest请求，问题的关键就在这个随机挑选一个broker之上，如果正好随机到的是那台磁盘损毁而崩溃的机器，那么这个请求必定要等到设定的超时时间之后才能捕获异常：[ERROR] - [fetching topic metadata for topics [Set(hidden-topic)] from broker [ArrayBuffer(id:0,host:xxx.xxx.xxx.xxx,port:9092)] failed] ，进而再找到下一个broker重新发送TopicMetadataRequest请求。

上面提到了超时时间，这个超时时间是通过request.timeout.ms参数设定的，默认值为10000，也就是10s。具体指的是kafka.network.BlockingChannel中的channel.socket.connect(new InetSocketAddress(host, port), connectTimeoutMs)这段代码，参数connectTimeoutMs就是指request.timeout.ms。如果元数据的请求时打到那台崩溃的broker上的话，那么元数据的请求就要耗时10s以上，待元数据刷新后才能发送消息。这个request.timeout.ms参数才是导致文中开头有少量消息发送时延很大的原因。为了进一步验证结论是否正确，笔者将相关的类SyncProducer、BlockingChannel用Java重写了一遍，并测试出请求的耗时，当访问一个不存在的ip地址时，从发送请求到异常报出的耗时在10194ms。不过如果访问一个存在的ip地址时，但是没有kafka服务的话，从发送请求到返回的耗时只有1248ms，基本上减少了一个数量级，如果读者在遇到同样的问题时，不妨上线一台（随意一台能建立TCP连接的机器就好）与崩溃宕机的那台broker一样的ip地址的机器，上面无需运行kafka的服务，就能大大的降低发送消息的时延。这样可以流出更多的时间去定位、修复、重新上线那台崩溃的broker。

当然如果对时延过敏，还有一些其它的方法可以参考。比如metadata.broker.list配置的是一个类似虚IP（VIP）的话，可以在VIP的下游剔除掉这台broker，让VIP过来的请求不会落到崩溃的broker上。或者也可以通过topic.metadata.refresh.interval.ms参数来调节，比如设置为负数就可以免去定时刷新元数据的烦恼，不过在元数据变动的时候很难有效的感知其变化，通过定时重连之类的方法又显得有点奇葩。当然直接上线一台服务器，安装运行kafka服务，且设置这台kafka服务器的ip为崩溃的那台服务器的ip地址，不过这番操作比直接拉一台空机器的时效性要低一些，凡事在于抉择。

Old Producer拉取元数据信息以及发送消息是在同一个线程中的，这必然会引起局部消息的时延增大。不过在新版的KafkaProducer中，这些问题都已经迎刃而解，具体怎么处理，且看后面的文章分析。

# 666. 彩蛋

如果你对 Kafka 并发感兴趣，欢迎加入我的知识一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)