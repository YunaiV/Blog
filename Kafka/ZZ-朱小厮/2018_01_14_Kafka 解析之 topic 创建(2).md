title: Kafka 解析之 topic 创建(2)
date: 2018-01-14
tag: 
categories: Kafka
permalink: Kafka/topic-create-2
author: 朱小厮
from_url: https://blog.csdn.net/u013256816/article/details/79303846
wechat_url: 

-------

摘要: 原创出处 https://blog.csdn.net/u013256816/article/details/79303846 「朱小厮」欢迎转载，保留摘要，谢谢！

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

### 前言

上一篇[Kafka解析之topic创建(1)](http://blog.csdn.net/u013256816/article/details/79303825)中的介绍了怎样创建一个topic以及对应的replica-assignment参数的一些使用细节，本文继续来讲述一下自动分配方案的具体算法实现，包括未指定机架的分配策略和指定机架的分配策略。

### 承接

如果在创建topic的时候并没有指定replica-assignment参数，那么就需要采用kafka默认的分区副本分配策略来创建topic。主要的是以下这6行代码：

```java
CommandLineUtils.checkRequiredArgs(opts.parser, opts.options, opts.partitionsOpt, opts.replicationFactorOpt)
val partitions = opts.options.valueOf(opts.partitionsOpt).intValue
val replicas = opts.options.valueOf(opts.replicationFactorOpt).intValue
val rackAwareMode = if (opts.options.has(opts.disableRackAware)) RackAwareMode.Disabled
                    else RackAwareMode.Enforced
AdminUtils.createTopic(zkUtils, topic, partitions, replicas, configs, rackAwareMode)
```

第一行的作用就是验证一下执行kafka-topics.sh时参数列表中是否包含有partitions和replication-factor这两个参数，如果没有包含则报出：Missing required argument “[partitions]”或者Missing required argument “[replication-factor]”，并给出参数的提示信息列表。

第2-5行的作用是获取paritions、replication-factor参数所对应的值以及验证是否包含disable-rack-aware这个参数。从0.10.x版本开始，kafka可以支持指定broker的机架信息，如果指定了机架信息则在副本分配时会尽可能地让分区的副本分不到不同的机架上。指定机架信息是通过kafka的配置文件config/server.properties中的broker.rack参数来配置的，比如配置当前broker所在的机架为“RACK1”：

```
broker.rack=RACK1
```

最后一行通过AdminUtils.createTopic方法来继续创建，至此代码流程又进入到下一个无底洞，不过暂时不用担心，下面是这个方法的详细内容，看上去只有几行而已：

```java
def createTopic(zkUtils: ZkUtils,
                topic: String,
                partitions: Int,
                replicationFactor: Int,
                topicConfig: Properties = new Properties,
                rackAwareMode: RackAwareMode = RackAwareMode.Enforced) {
  val brokerMetadatas = getBrokerMetadatas(zkUtils, rackAwareMode)
  val replicaAssignment = AdminUtils.assignReplicasToBrokers(brokerMetadatas, partitions, replicationFactor)
  AdminUtils.createOrUpdateTopicPartitionAssignmentPathInZK(zkUtils, topic, replicaAssignment, topicConfig)
}
```

总共只有三行，最后一行还是见过的，在使用replica-assignment参数解析验证之后调用的，主要用来在/brokers/topics路径下写入相应的节点。回过头来看第一句，它是用来获取集群中每个broker的brokerId和机架信息（Option[String]类型）信息的列表，为下面的 AdminUtils.assignReplicasToBrokers()方法做分区副本分配前的准备工作。AdminUtils.assignReplicasToBrokers()首先是做一些简单的验证工作：分区个数partitions不能小于等于0、副本个数replicationFactor不能小于等于0以及副本个数replicationFactor不能大于broker的节点个数，其后的步骤就是方法最重要的两大核心：assignReplicasToBrokersRackUnaware和assignReplicasToBrokersRackAware，看这个名字也应该猜出个一二来，前者用来针对不指定机架信息的情况，而后者是用来针对指定机架信息的情况，后者更加复杂一点。

### 未指定机架的分配策略

为了能够循序渐进的说明问题，这里先来讲解assignReplicasToBrokersRackUnaware，对应的代码如下：

```java
private def assignReplicasToBrokersRackUnaware(nPartitions: Int,
                                               replicationFactor: Int,
                                               brokerList: Seq[Int],
                                               fixedStartIndex: Int,
                                               startPartitionId: Int): Map[Int, Seq[Int]] = {
  val ret = mutable.Map[Int, Seq[Int]]()
  val brokerArray = brokerList.toArray
  val startIndex = if (fixedStartIndex >= 0) fixedStartIndex else rand.nextInt(brokerArray.length)
  var currentPartitionId = math.max(0, startPartitionId)
  var nextReplicaShift = if (fixedStartIndex >= 0) fixedStartIndex else rand.nextInt(brokerArray.length)
  for (_ <- 0 until nPartitions) {
    if (currentPartitionId > 0 && (currentPartitionId % brokerArray.length == 0))
      nextReplicaShift += 1
    val firstReplicaIndex = (currentPartitionId + startIndex) % brokerArray.length
    val replicaBuffer = mutable.ArrayBuffer(brokerArray(firstReplicaIndex))
    for (j <- 0 until replicationFactor - 1)
      replicaBuffer += brokerArray(replicaIndex(firstReplicaIndex, nextReplicaShift, j, brokerArray.length))
    ret.put(currentPartitionId, replicaBuffer)
    currentPartitionId += 1
  }
  ret
}
```

主构造函数参数列表中的fixedStartIndex和startPartitionId的值是从上游AdminUtils.assignReplicasToBrokers()方法调用传下来，都是-1，分别表示第一个副本分配的位置和起始分区编号。assignReplicasToBrokers这个方法的核心是遍历每个分区partition然后从brokerArray（brokerId的列表）中选取replicationFactor个brokerId分配给这个partition。

方法首先创建一个可变的Map用来存放本方法将要返回的结果，即分区partition和分配副本的映射关系。由于fixedStartIndex为-1，所以startIndex是一个随机数，用来计算一个起始分配的brokerId，同时由于startPartitionId为-1，所以currentPartitionId的值为0，可见默认创建topic时总是从编号为0的分区依次轮询进行分配。nextReplicaShift表示下一次副本分配相对于前一次分配的位移量，这个字面上理解有点绕，不如举个例子：假设集群中有3个broker节点，即代码中的brokerArray，创建某topic有3个副本和6个分区，那么首先从partitionId（partition的编号）为0的分区开始进行分配，假设第一次计算（由rand.nextInt(brokerArray.length)随机）到nextReplicaShift为1，第一次随机到的startIndex为2，那么partitionId为0的第一个副本的位置（这里指的是brokerArray的数组下标）firstReplicaIndex = (currentPartitionId + startIndex) % brokerArray.length = （0+2）%3 = 2，第二个副本的位置为replicaIndex(firstReplicaIndex, nextReplicaShift, j, brokerArray.length) = replicaIndex(2, nextReplicaShift+1,0, 3)=？，这里引入了一个新的方法replicaIndex，不过这个方法很简单，具体如下：

```java
private def replicaIndex(firstReplicaIndex: Int, secondReplicaShift: Int, replicaIndex: Int, nBrokers: Int): Int = {
  val shift = 1 + (secondReplicaShift + replicaIndex) % (nBrokers - 1)
  (firstReplicaIndex + shift) % nBrokers
}
```

继续计算 replicaIndex(2, nextReplicaShift+1,0, 3) = replicaIndex(2, 2,0, 3)= (2+(1+(2+0)%(3-1)))%3=0。继续计算下一个副本的位置replicaIndex(2, 2,1, 3) = (2+(1+(2+1)%(3-1)))%3=1。所以partitionId为0的副本分配位置列表为[2,0,1]，如果brokerArray正好是从0开始编号，也正好是顺序不间断的，即brokerArray为[0,1,2]的话，那么当前partitionId为0的副本分配策略为[2,0,1]。如果brokerId不是从零开始，也不是顺序的（有可能之前集群的其中broker几个下线了），最终的brokerArray为[2,5,8]，那么partitionId为0的分区的副本分配策略为[8,2,5]。为了便于说明问题，可以简单的假设brokerArray就是[0,1,2]。

同样计算下一个分区，即partitionId为1的副本分配策略。此时nextReplicaShift还是为2，没有满足自增的条件。这个分区的firstReplicaIndex = (1+2)%3=0。第二个副本的位置replicaIndex(0,2,0,3) = (0+(1+(2+0)%(3-1)))%3 = 1，第三个副本的位置replicaIndex(0,2,1,3) = 2，最终partitionId为2的分区分配策略为[0,1,2]。

以此类推，更多的分配细节可以参考下面的demo，topic-test4的分区分配策略和上面陈述的一致：

```java
[root@node3 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test4 --replication-factor 3 --partitions 6
Created topic "topic-test4".
[root@node3 kafka_2.12-1.0.0]# bin/kafka-topics.sh --describe --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test4
Topic:topic-test4   PartitionCount:6    ReplicationFactor:3 Configs:
    Topic: topic-test4  Partition: 0    Leader: 2   Replicas: 2,0,1 Isr: 2,0,1
    Topic: topic-test4  Partition: 1    Leader: 0   Replicas: 0,1,2 Isr: 0,1,2
    Topic: topic-test4  Partition: 2    Leader: 1   Replicas: 1,2,0 Isr: 1,2,0
    Topic: topic-test4  Partition: 3    Leader: 2   Replicas: 2,1,0 Isr: 2,1,0
    Topic: topic-test4  Partition: 4    Leader: 0   Replicas: 0,2,1 Isr: 0,2,1
    Topic: topic-test4  Partition: 5    Leader: 1   Replicas: 1,0,2 Isr: 1,0,2
```

我们无法预先获知startIndex和nextReplicaShift的值，因为都是随机产生的。startIndex和nextReplicaShift的值可以通过最终的分区分配方案来反推，比如上面的topic-test4，第一个分区（即partitionId=0的分区）的第一个副本为2，那么可由2 = (0+startIndex)%3推断出startIndex为2。之所以startIndex随机是因为这样可以在多个topic的情况下尽可能的均匀分布分区副本，如果这里固定为一个特定值，那么每次的第一个副本都是在这个broker上，进而就会导致少数几个broker所分配到的分区副本过多而其余broker分配到的过少，最终导致负载不均衡。尤其是某些topic的副本数和分区数都比较少，甚至都为1的情况下，所有的副本都落到了那个指定的broker上。与此同时，在分配时位移量nextReplicaShift也可以更好的使得分区副本分配的更加均匀。

### 指定机架的分配策略

下面我们再来看一下指定机架信息的副本分配情况，即方法assignReplicasToBrokersRackAware，注意assignReplicasToBrokersRackUnaware的执行前提是所有的broker都没有配置机架信息，而assignReplicasToBrokersRackAware的执行前提是所有的broker都配置了机架信息，如果出现部分broker配置了机架信息而另一部分没有配置的话，则会抛出AdminOperationException的异常，如果还想要顺利创建topic的话，此时需加上“–disable-rack-aware”，详细demo如下：

```java
[root@node2 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test5 --replication-factor 2 --partitions 4
Error while executing topic command : Not all brokers have rack information. Add --disable-rack-aware in command line to make replica assignment without rack information.
[2018-02-06 00:19:07,213] ERROR kafka.admin.AdminOperationException: Not all brokers have rack information. Add --disable-rack-aware in command line to make replica assignment without rack information.
    at kafka.admin.AdminUtils$.getBrokerMetadatas(AdminUtils.scala:443)
    at kafka.admin.AdminUtils$.createTopic(AdminUtils.scala:461)
    at kafka.admin.TopicCommand$.createTopic(TopicCommand.scala:110)
    at kafka.admin.TopicCommand$.main(TopicCommand.scala:63)
    at kafka.admin.TopicCommand.main(TopicCommand.scala)
 (kafka.admin.TopicCommand$)
[root@node2 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test5 --replication-factor 2 --partitions 4 --disable-rack-aware
Created topic "topic-test5".
[root@node2 kafka_2.12-1.0.0]#123456789101112
```

assignReplicasToBrokersRackAware方法的详细内容如下，这段代码内容偏多，仅供参考，看得辣眼睛的小伙伴可以习惯性的忽略，后面会做详细的文字介绍。

```java
private def assignReplicasToBrokersRackAware(nPartitions: Int,
                                             replicationFactor: Int,
                                             brokerMetadatas: Seq[BrokerMetadata],
                                             fixedStartIndex: Int,
                                             startPartitionId: Int): Map[Int, Seq[Int]] = {
  val brokerRackMap = brokerMetadatas.collect { case BrokerMetadata(id, Some(rack)) =>
    id -> rack
  }.toMap
  val numRacks = brokerRackMap.values.toSet.size//统计机架个数
  val arrangedBrokerList = getRackAlternatedBrokerList(brokerRackMap)//基于机架信息生成一个Broker列表，不同机架上的Broker交替出现

  val numBrokers = arrangedBrokerList.size
  val ret = mutable.Map[Int, Seq[Int]]()
  val startIndex = if (fixedStartIndex >= 0) fixedStartIndex else rand.nextInt(arrangedBrokerList.size)
  var currentPartitionId = math.max(0, startPartitionId)
  var nextReplicaShift = if (fixedStartIndex >= 0) fixedStartIndex else rand.nextInt(arrangedBrokerList.size)
  for (_ <- 0 until nPartitions) {
    if (currentPartitionId > 0 && (currentPartitionId % arrangedBrokerList.size == 0))
      nextReplicaShift += 1
    val firstReplicaIndex = (currentPartitionId + startIndex) % arrangedBrokerList.size
    val leader = arrangedBrokerList(firstReplicaIndex)
    val replicaBuffer = mutable.ArrayBuffer(leader)//每个分区的副本分配列表
    val racksWithReplicas = mutable.Set(brokerRackMap(leader))//每个分区中所分配的机架的列表集
    val brokersWithReplicas = mutable.Set(leader)//每个分区所分配的brokerId的列表集，和racksWithReplicas一起用来做一层筛选处理
    var k = 0
    for (_ <- 0 until replicationFactor - 1) {
      var done = false
      while (!done) {
        val broker = arrangedBrokerList(replicaIndex(firstReplicaIndex, nextReplicaShift * numRacks, k, arrangedBrokerList.size))
        val rack = brokerRackMap(broker)
        if ((!racksWithReplicas.contains(rack) || racksWithReplicas.size == numRacks)
            && (!brokersWithReplicas.contains(broker) || brokersWithReplicas.size == numBrokers)) {
          replicaBuffer += broker
          racksWithReplicas += rack
          brokersWithReplicas += broker
          done = true
        }
        k += 1
      }
    }
    ret.put(currentPartitionId, replicaBuffer)
    currentPartitionId += 1
  }
  ret
}
```

第一步获得brokerId和rack信息的映射关系列表brokerRackMap ，之后调用getRackAlternatedBrokerList()方法对brokerRackMap做进一步的处理生成一个brokerId的列表，这么解释比较拗口，不如举个demo。假设目前有3个机架rack1、rack2和rack3，以及9个broker，分别对应关系如下：

```
rack1: 0, 1, 2
rack2: 3, 4, 5
rack3: 6, 7, 8
```

那么经过getRackAlternatedBrokerList()方法处理过后就变成了[0, 3, 6, 1, 4, 7, 2, 5, 8]这样一个列表，显而易见的这是轮询各个机架上的broker而产生的，之后你可以简单的将这个列表看成是brokerId的列表，对应assignReplicasToBrokersRackUnaware()方法中的brokerArray，但是其中包含了简单的机架分配信息。之后的步骤也和未指定机架信息的算法类似，同样包含startIndex、currentPartiionId, nextReplicaShift的概念，循环为每一个分区分配副本。分配副本时处理第一个副本之外，其余的也调用replicaIndex方法来获得一个broker，但是这里和assignReplicasToBrokersRackUnaware()不同的是，这里不是简单的将这个broker添加到当前分区的副本列表之中，还要经过一层的筛选，满足以下任意一个条件的broker不能被添加到当前分区的副本列表之中：

1. 如果此broker所在的机架中已经存在一个broker拥有该分区的副本，并且还有其他的机架中没有任何一个broker拥有该分区的副本。对应代码中的(!racksWithReplicas.contains(rack) || racksWithReplicas.size == numRacks)
2. 如果此broker中已经拥有该分区的副本，并且还有其他broker中没有该分区的副本。对应代码中的(!brokersWithReplicas.contains(broker) || brokersWithReplicas.size == numBrokers))

无论是带机架信息的策略还是不带机架信息的策略，上层调用方法AdminUtils.assignReplicasToBrokers()最后都是获得一个[Int, Seq[Int]]类型的副本分配列表，其最后作为kafka zookeeper节点/brokers/topics/{topic-name}节点数据。至此kafka的topic创建就讲解完了，有些同学会感到很疑问，全文通篇（包括上一篇）都是在讲述如何分配副本，最后得到的也不过是个分配的方案，并没有真正创建这些副本的环节，其实这个观点没有任何问题，对于通过kafka提供的kafka-topics.sh脚本创建topic的方法来说，它只是提供一个副本的分配方案，并在kafka zookeeper中创建相应的节点而已。kafka broker的服务会注册监听/brokers/topics/目录下是否有节点变化，如果有新节点创建就会监听到，然后根据其节点中的数据（即topic的分区副本分配方案）来创建对应的副本，具体的细节笔者会在后面的副本管理中有详细介绍。

既然整个kafka-topics.sh脚本的作用就只是创建一个zookeeper的节点，并且写上一些分配的方案数据而已，那么我们直接创建一个zookeeper节点来创建一个topic可不可以呢？答案是可以的。在开启的kafka broker的情况下（如果未开启kafka服务的情况下创建zk节点的话，待kafka启动之后是不会再创建实际副本的，只有watch到当前通知才可以），通过zkCli创建一个与topic-test1副本分配方案相同的topic-test6，详细如下：

```shell
[zk: localhost:2181(CONNECTED) 8] create /kafka100/brokers/topics/topic-test6 {"version":1,"partitions":{"2":[0,1],"1":[1,0],"3":[1,0],"0":[0,1]}}
Created /kafka100/brokers/topics/topic-test6
```

这里再来进一步check下topic-test1和topic-test6是否完全相同：

```shell
[root@node1 kafka_2.12-1.0.0]# bin/kafka-topics.sh --describe --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test1,topic-test6
Topic:topic-test1   PartitionCount:4    ReplicationFactor:2 Configs:
    Topic: topic-test1  Partition: 0    Leader: 0   Replicas: 0,1   Isr: 0,1
    Topic: topic-test1  Partition: 1    Leader: 1   Replicas: 1,0   Isr: 1,0
    Topic: topic-test1  Partition: 2    Leader: 0   Replicas: 0,1   Isr: 0,1
    Topic: topic-test1  Partition: 3    Leader: 1   Replicas: 1,0   Isr: 1,0
Topic:topic-test6   PartitionCount:4    ReplicationFactor:2 Configs:
    Topic: topic-test6  Partition: 0    Leader: 0   Replicas: 0,1   Isr: 0,1
    Topic: topic-test6  Partition: 1    Leader: 1   Replicas: 1,0   Isr: 1,0
    Topic: topic-test6  Partition: 2    Leader: 0   Replicas: 0,1   Isr: 0,1
    Topic: topic-test6  Partition: 3    Leader: 1   Replicas: 1,0   Isr: 1,0
```

答案显而易见。前面的篇幅也提到了通过kafka-topics.sh脚本的创建方式会对副本的分配有大堆的合格性的校验，但是直接创建zk节点的方式没有这些校验，比如创建一个topic-test7，这个topic节点的数据为：{“version”:1,”partitions”:{“2”:[0,1],”1”:[1],”3”:[1,0],”0”:[0,1]}}，可以看出paritionId为1的分区只有一个副本，我们来检测下是否创建成功：

```shell
[root@node1 kafka_2.12-1.0.0]# bin/kafka-topics.sh --describe --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test7
Topic:topic-test7   PartitionCount:4    ReplicationFactor:2 Configs:
    Topic: topic-test7  Partition: 0    Leader: 0   Replicas: 0,1   Isr: 0,1
    Topic: topic-test7  Partition: 1    Leader: 1   Replicas: 1 Isr: 1
    Topic: topic-test7  Partition: 2    Leader: 0   Replicas: 0,1   Isr: 0,1
    Topic: topic-test7  Partition: 3    Leader: 1   Replicas: 1,0   Isr: 1,0
```

结果也是显而易见的，不过如果没有特殊需求不宜用这种方式，这种方式把控不好就像断了线的风筝一样难以把控。如果又不想用auto.create.topics.enable=true的这种方式，也不想用kafka-topics.sh的这种方式，就像用类似java的编程语言在代码中内嵌创建topic，以便更好的与公司内部的系统结合怎么办？

我们上篇文章中知道kafka-topics.sh内部就是调用了一下kafka.admin.TopicCommand而已，那么我们也调用一下这个可不可以？Of course，下面举一个简单的demo，创建一个副本数为2，分区数为4的topic-test8：

```java
public class CreateTopicDemo {
    public static void main(String[] args) {
        //demo: 创建一个副本数为2，分区数为4的主题：topic-test8
        String[] options = new String[]{
                "--create",
                "--zookeeper","192.168.0.2:2181/kafka100",
                "--replication-factor", "2",
                "--partitions", "4",
                "--topic", "topic-test8"
        };
        kafka.admin.TopicCommand.main(options);
    }
}
```

可以看到这种方式和kafka-topics.sh的方式如出一辙，可以用这种方式继承到自动化系统中以创建topic，当然对于topic的删、改、查等都可以通过这种方法来实现，具体的篇幅限制就不一一细表了。

有关kafka的topic的创建细节其实并没有介绍完全，比如create.topic.policy.class.name参数的具体含义与用法，这个会在后面介绍KafkaApis的时候再做具体的介绍，所以为了不迷路，为了涨知识不如关注一波公众号，然后watch。。。

# 666. 彩蛋

如果你对 Kafka 并发感兴趣，欢迎加入我的知识一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)