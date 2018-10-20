title: 如何获取 Kafka 的消费者详情 —— 从 Scala 到 Java 的切换
date: 2018-01-18
tag: 
categories: Kafka
permalink: Kafka/from-scala-to-java
author: 朱小厮
from_url: https://blog.csdn.net/u013256816/article/details/79968647
wechat_url: 

-------

摘要: 原创出处 https://blog.csdn.net/u013256816/article/details/79968647 「朱小厮」欢迎转载，保留摘要，谢谢！


-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

### 前文摘要

在上一篇文章《[Kafka的Lag计算误区及正确实现](https://blog.csdn.net/u013256816/article/details/79955578)》中介绍了如何计算消费者的消费滞后量(Lag)，并且讲解了如何调用Kafka的kafka.admin.ConsumerGroupCommand文件中的KafkaConsumerGroupService来发送OffsetRequest和OffsetFetchRequest两个请求，进而通过两个请求结果之间的差值来获得结果。不过如果你不想修改kafka-core的代码并重新编译的话，这种实现方式无法成功，所以本文的主要目的就是通过调用更底层的API来实现不修改kafka-core的代码来实现KafkaConsumerGroupService的功能，即通过Java调用Scala的代码来实现获取Kafka的消费者详情的功能。

### 目标及实现

实现如同 bin/kafka-consumer-group.sh –describe –bootstrap-server localhost:9092 –group CONSUMER_GROUP_ID的效果：

```shell
[root@node2 kafka_2.12-1.0.0]# bin/kafka-consumer-groups.sh --describe --bootstrap-server localhost:9092 --group CONSUMER_GROUP_ID
TOPIC                PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG        CONSUMER-ID                                       HOST                   CLIENT-ID
topic-test1          0          1648            1648            0          CLIENT_ID-e2d41f8d-dbd2-4f0e-9239-efacb55c6261    /192.168.92.1          CLIENT_ID
topic-test1          1          1648            1648            0          CLIENT_ID-e2d41f8d-dbd2-4f0e-9239-efacb55c6261    /192.168.92.1          CLIENT_ID
topic-test1          2          1648            1648            0          CLIENT_ID-e2d41f8d-dbd2-4f0e-9239-efacb55c6261    /192.168.92.1          CLIENT_ID
topic-test1          3          1648            1648            0          CLIENT_ID-e2d41f8d-dbd2-4f0e-9239-efacb55c6261    /192.168.92.1          CLIENT_ID
```

KafkaConsumerGroupService的核心方法是CollectGroupAssignment，其方法参数为一个consumer group的groupId，方法输出为上面示例中的列表信息。CollectGroupAssignment方法主要有以下几个步骤：

1. 根据groupId调用describeConsumerGroup方法(内部原理是发送DescribeGroupsRequest请求)来获取consumer group的基本信息，参考上面示例中的CONSUMER-ID、HOST、CLIENT-ID以及TopicPartition信息，但是没有CURRENT-OFFSET、LOG-END-OFFSET、LAG信息。注意这里的LOG-END-OFFSET是消费者可见的LEO，不是生产者可见的LEO，也就是通俗意义上的HW。
2. 根据groupId调用listGroupOffsets方法(内部原理是发送OffsetFetchRequest请求)来获取各个分区(Partition)的对应的消费位移CURRENT-OFFSET。
3. 通过调用KafkaConsumer的endOffsets方法来获取TopicPartition对应的HW，即示例中的LOG-END-OFFSET。
4. 计算Lag并组合成信息列表List<PartitionAssignmentState>。

#### 改造

对应Java版的KafkaConsumerGroupService改造代码可以参见[代码](https://github.com/hiddenzzh/kafka/blob/master/src/main/java/com/hidden/custom/kafka/admin/KafkaConsumerGroupCustomService.java)，目录结构如下图所示：
![img](http://static.iocoder.cn/csdn/20180416234711534?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMyNTY4MTY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

其中model中的ConsumerGroupSummary、ConsumerSummary和PartitionAssignmentState是简单的JavaBean, PartitionAssignmentState是用来保存每个TopicPartition的消费者信息的，具体内容参考如下。KafkaConsumerGroupCustomService就是本文所要陈述的Java改造办的KafkaConsumerGroupSerivice，ConsumerGroupUtils用来存放一些公用的代码。

```java
@Data
@Builder
public class PartitionAssignmentState {
    private String group; // groupId
    private Node coordinator; // consumer coodinator节点信息
    private String topic;
    private int partition;
    private long offset;
    private long lag;
    private String consumerId;
    private String host;
    private String clientId;
    private long logEndOffset;
}
```

初始化KafkaConsumerGroupCustomService需要Kafka的服务端地址，然后初始化AdminClient和KafkaConsumer，AdminClient中包含了众多管理类方法，主要是通过发送各种自定义协议请求来完成，上面步骤中所说的describeConsumerGroup和listGroupOffsets方法也是通过AdminClient来实现的；KafkaConsumer主要是用来获取TopicPartition对应的HW(消费者可见的LogEndOffsets)的。

KafkaConsumerGroupCustomService中与scala版对应的collectGroupAssignment方法如下（详细步骤参考代码注释）：

```java
public List<PartitionAssignmentState> collectGroupAssignment(
        AdminClient adminClient, KafkaConsumer<String, String> consumer,
        String group) {
    //1. 获取consumer group的基本信息，包括CONSUMER-ID、HOST、
    // CLIENT-ID以及TopicPartition信息
    AdminClient.ConsumerGroupSummary consumerGroupSummary
            = adminClient.describeConsumerGroup(group, 0);
    List<TopicPartition> assignedTopicPartitions = new ArrayList<>();
    List<PartitionAssignmentState> rowsWithConsumer = new ArrayList<>();
    scala.collection.immutable.List<AdminClient.ConsumerSummary> consumers
            = consumerGroupSummary.consumers().get();
    if (consumers != null) {
        //2. 获取各个分区(Partition)的对应的消费位移CURRENT-OFFSET
        scala.collection.immutable.Map<TopicPartition, Object> offsets
                = adminClient.listGroupOffsets(group);
        if (offsets.nonEmpty()) {
            String state = consumerGroupSummary.state();
            // 3. 还有一个状态是Dead表示"group"对应的consumer group不存在
            if (state.equals("Stable") || state.equals("Empty")
                    || state.equals("PreparingRebalance")
                    || state.equals("AwaitingSync")) {
                List<ConsumerSummary> consumerList = changeToJavaList(consumers);
                // 4. 获取当前有消费者的消费信息，即包含CONSUMER-ID、HOST、CLIENT-ID
                rowsWithConsumer = getRowsWithConsumer(consumerGroupSummary, offsets,
                        consumer, consumerList, assignedTopicPartitions, group);
            }
        }
        //5. 获取当前没有消费者的消费信息
        List<PartitionAssignmentState> rowsWithoutConsumer =
                getRowsWithoutConsumer(consumerGroupSummary,
                offsets, consumer, assignedTopicPartitions, group);
        //6. 合并结果
        rowsWithConsumer.addAll(rowsWithoutConsumer);
    }
    return rowsWithConsumer;
}
```

KafkaConsumerGroupCustomService类中包含有getRowsWithConsumer()、getRowsWithoutConsumer()、changeToJavaList等私有方法也都是在Scala语言与Java语言之间进行切换，这样可以不需要修改kafka-core的原生代码而通过外部的封装调用既可以实现获取Kafka消费者详情的功能。光看代码比较抽象，建议对此感兴趣的同学可以亲自对比一下kafka-core包中kafka.admin.ConsumerGroupCommand的KafkaConsumerGroupSerivice与笔者自定义的KafkaConsumerGroupCustomService的实现来了解下Scala语言到Java语言的转换。

如果需要打印详情可以调用KafkaConsumerGroupCustomService同目录的ConsumerGroupUtils类中的printPasList(List list)方法。注意要运行这些代码需要JDK8的环境，笔者为了让代码显得“骚气”一点就用来一点Java8的语法，如果需要Java7的代码实现可以关注私聊。

或许有些同学对于Scala和Java交叉的代码并不感冒，想要寻求一种存Java式的实现方式，那么在这里怎么实现呢？答案是通过KafkaAdminClient，它是AdminClient的Java版实现，从Kafka0.11.0.0版本开始引入的，不过KafkaAdminClient本身并没有提供describeConsumerGroup、listGroupOffsets之类的方法给我们直接使用，扩展一下也很方便，由于篇幅限制，这部分的内容将在下一篇文章中进行介绍，如果想要先一睹为快，可以参考下[代码实现](https://github.com/hiddenzzh/kafka/blob/master/src/main/java/org/apache/kafka/clients/admin/app/KafkaConsumerGroupService.java)，详细的逻辑解析敬请期待….

# 666. 彩蛋

如果你对 Kafka 并发感兴趣，欢迎加入我的知识一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)