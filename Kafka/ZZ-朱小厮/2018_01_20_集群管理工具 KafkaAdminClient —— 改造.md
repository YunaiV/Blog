title: 集群管理工具 KafkaAdminClient —— 改造
date: 2018-01-20
tag: 
categories: Kafka
permalink: Kafka/
author: 朱小厮
from_url: https://blog.csdn.net/u013256816/article/details/79996138
wechat_url: 

-------

摘要: 原创出处 https://blog.csdn.net/u013256816/article/details/79996138 「朱小厮」欢迎转载，保留摘要，谢谢！

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

### 前文概述

在上一篇文章《[集群管理工具KafkaAdminClient——原理与示例](https://blog.csdn.net/u013256816/article/details/79996056)》中讲述了KafkaAdminClient的功能以及相应的原理，但是同时也提出了目前的KafkaAdminClient并没有非常的完善，还有许多功能还需要去丰富，这些功能可以自定义实现，在《[如何获取Kafka的消费者详情——从Scala到Java的切换](https://blog.csdn.net/u013256816/article/details/79968647)》一文中介绍了如何获取Kafka的消费详情，其原理是通过Java调用Kafka的Scala代码实现的，如果要使用纯Java的方式实现就需要用到了KafkaAdminClient，另外Scala版的AdminClient也被标注为：“This client is deprecated, and will be replaced by KafkaAdminClient.”，说明官方也推荐使用KafkaAdminClient。不过现在的版本（目前最新1.1.0）并没有提供类似describeConsumerGroup和listGroupOffsets的方法实现，这一点在前文《[集群管理工具KafkaAdminClient——原理与示例](https://blog.csdn.net/u013256816/article/details/79996056)》也有提及，所以如果要实现获取类似消费者详情的功能，那么就需要自己动手进行改造。

### 改造

参考Scala版的AdminClient，要实现获取Kafka的消费者详情的功能首先需要实现describeConsumerGroup和listGroupOffsets的方法，其中describeConsumerGroup方法内部还需要一个findCoordinator的方法用来提供消费者对应的coodinator节点，以便提供详细的消费者详情。describeConsumerGroup、listGroupOffsets和findCoordinator这三个方法都将在KafkaAdminClient类里提供自定义实现。KafkaAdminClient和XXXOptions、XXXResult的类都位于org.apache.kafka.clients.admin包下，笔者也将扩展的类也置于其同一包下，不过也进行了一些区分，如下图所示，新加入的XXXOptions、XXXResult类放入extend下，新加入的JavaBean放入model下，然后与具体应用功能对应的放在app下：
![img](http://static.iocoder.cn/csdn/2018041820172485?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMyNTY4MTY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

首先建立对应的XXXOptions、XXXResult类，就那简单的ListGroupOffsets来说，其ListGroupOffsetsOptions只是继承了AbstractOptions的空实现，而ListGroupOffsetsResult也很简单，提供了一个KafkaFuture的调用，代码参考如下：

```java
public class ListGroupOffsetsResult {
    private final KafkaFutureImpl<Map<TopicPartition, Long>> future;
    public ListGroupOffsetsResult(KafkaFutureImpl<Map<TopicPartition, Long>> future) {
        this.future = future;
    }
    public KafkaFutureImpl<Map<TopicPartition, Long>> values(){
        return this.future;
    }
}
```

model目录下的ConsumerGroupSummary是所要实现的describeConsumerGroup方法中所要获取的值类型，封装在DescribeConsumerGroupResult 中；ConsumerSummary在describeConsumerGroup方法内部使用，用来封装消费状态，包括consumerId、clientId、host(消费者主机)以及TopicPartition列表，最终被封装进ConsumerGroupSummary中。PartitionAssignmentState是服务于KafkaConsumerGroupService的，用来最后显示消费者详情列表。

KafkaAdminClient的父类是AdminClient(kafka-client中的抽象类)，describeConsumerGroup、listGroupOffsets和findCoordinator这三个方法也需要在AdminClient类中做申明，详细参考如下：

```java
public abstract DescribeConsumerGroupResult describeConsumerGroup(final String group,
                                                         final DescribeConsumerGroupOptions options);
public DescribeConsumerGroupResult describeConsumerGroup(final String group) {
    return describeConsumerGroup(group, new DescribeConsumerGroupOptions());
}
public abstract FindCoordinatorResult findCoordinator(final String group,
                                                      final FindCoordinatorOptions options);
public FindCoordinatorResult findCoordinator(final String group) {
    return findCoordinator(group, new FindCoordinatorOptions());
}
public abstract ListGroupOffsetsResult listGroupOffsets(final String group,
                                                        final ListGroupOffsetsOptions options);
public ListGroupOffsetsResult listGroupOffsets(final String group){
    return listGroupOffsets(group, new ListGroupOffsetsOptions());
}
```

在前面2篇文章《[集群管理工具KafkaAdminClient——原理与示例](https://blog.csdn.net/u013256816/article/details/79996056)》和《[如何获取Kafka的消费者详情——从Scala到Java的切换](https://blog.csdn.net/u013256816/article/details/79968647)》中都详细解释了describeConsumerGroup、listGroupOffsets方法，所以这里不在赘述，具体实现也很简单，可以参考笔者的[实现](https://github.com/hiddenzzh/kafka/blob/master/src/main/java/org/apache/kafka/clients/admin/KafkaAdminClient.java)。

最后来讲述一下org.apache.kafka.clients.admin.app包下的KafkaConsumerGroupService，具体代码地址在[这里](https://github.com/hiddenzzh/kafka/blob/master/src/main/java/org/apache/kafka/clients/admin/app/KafkaConsumerGroupService.java)，其内部通过上面改造的KafkaAdminClient和KafkaConsumer来实现，其内部逻辑和《[如何获取Kafka的消费者详情——从Scala到Java的切换](https://blog.csdn.net/u013256816/article/details/79968647)》一文中的KafkaConsumerGroupCustomService一样，这里就不在赘述了。

本篇以及《[Kafka的Lag计算误区及正确实现](https://blog.csdn.net/u013256816/article/details/79955578)》、《[如何获取Kafka的消费者详情——从Scala到Java的切换](https://blog.csdn.net/u013256816/article/details/79968647)》这三篇文章都是围绕如何获取消费者详情来做具体的陈述，回到问题的初衷：kafka.admin.ConsumerGroupCommand.PartitionAssignmentState无法被外部访问，那么真的需要这么复杂的转变过程么，详细请参考下一篇《Scala与Java语言的互操作》。

# 666. 彩蛋

如果你对 Kafka 并发感兴趣，欢迎加入我的知识一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)