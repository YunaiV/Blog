title: Kafka 解析之 topic 创建(1)
date: 2018-01-13
tag: 
categories: Kafka
permalink: Kafka/topic-create-1
author: 朱小厮
from_url: https://blog.csdn.net/u013256816/article/details/79303825
wechat_url: 

-------

摘要: 原创出处 https://blog.csdn.net/u013256816/article/details/79303825 「朱小厮」欢迎转载，保留摘要，谢谢！

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

在使用kafka发送消息和消费消息之前，必须先要创建topic，在kafka中创建topic的方式有以下2种：

1. 如果kafka broker中的config/server.properties配置文件中配置了auto.create.topics.enable参数为true（默认值就是true），那么当生产者向一个尚未创建的topic发送消息时，会自动创建一个num.partitions（默认值为1）个分区和default.replication.factor（默认值为1）个副本的对应topic。不过我们一般不建议将auto.create.topics.enable参数设置为true，因为这个参数会影响topic的管理与维护。
2. 通过kafka提供的kafka-topics.sh脚本来创建，并且我们也建议通过这种方式（或者相关的变种方式）来创建topic。

举个demo：通过kafka-topics.sh脚本来创建一个名为topic-test1并且副本数为2、分区数为4的topic。（如无特殊说明，本文所述都是基于1.0.0版本。）

```shell
bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test1 --replication-factor 2 --partitions 4
```

打开kafka-topics.sh脚本一探究竟，其内容只有一行，具体如下:

```shell
exec $(dirname $0)/kafka-run-class.sh kafka.admin.TopicCommand "$@"
```

这个脚本的主要作用就是运行kafka.admin.TopicCommand。在main方法中判断参数列表中是否包含有”create“，如果有，那么就实施创建topic的任务。创建topic时除了需要zookeeper的地址参数外，还需要指定topic的名称、副本因子replication-factor以及分区个数partitions等必选参数 ，还可以包括disable-rack-aware、config、if-not-exists等可选参数。

真正的创建过程是由createTopic这个方法中执行的，这个方法具体内容如下：

```java
def createTopic(zkUtils: ZkUtils, opts: TopicCommandOptions) {
  val topic = opts.options.valueOf(opts.topicOpt)//获取topic参数所对应的值，也就是Demo中的topic名称——topic-test
  val configs = parseTopicConfigsToBeAdded(opts)//将参数解析成Properties参数，config所指定的参数集
  val ifNotExists = opts.options.has(opts.ifNotExistsOpt)//对应if-not-exists
  if (Topic.hasCollisionChars(topic))
    println("WARNING: Due to limitations in metric names, topics with a period ('.') or underscore ('_') could collide. To avoid issues it is best to use either, but not both.")
  try {
    if (opts.options.has(opts.replicaAssignmentOpt)) {//检测是否有replica-assignment参数
      val assignment = parseReplicaAssignment(opts.options.valueOf(opts.replicaAssignmentOpt))
      AdminUtils.createOrUpdateTopicPartitionAssignmentPathInZK(zkUtils, topic, assignment, configs, update = false)
    } else {
      CommandLineUtils.checkRequiredArgs(opts.parser, opts.options, opts.partitionsOpt, opts.replicationFactorOpt)
      val partitions = opts.options.valueOf(opts.partitionsOpt).intValue
      val replicas = opts.options.valueOf(opts.replicationFactorOpt).intValue
      val rackAwareMode = if (opts.options.has(opts.disableRackAware)) RackAwareMode.Disabled
                          else RackAwareMode.Enforced
      AdminUtils.createTopic(zkUtils, topic, partitions, replicas, configs, rackAwareMode)
    }
    println("Created topic \"%s\".".format(topic))
  } catch  {
    case e: TopicExistsException => if (!ifNotExists) throw e
  }
}
```

createTopic方法中首先获取topic的名称，config参数集以及判断是否有if-not-exists参数。config参数集可以用来设置topic级别的配置以覆盖默认配置。如果创建的topic再现有的集群中存在，那么会报出异常：TopicExistsException，如果创建的时候带了if-not-exists参数，那么发现topic冲突的时候可以不做任何处理；如果topic不冲突，那么和不带if-not-exists参数的行为一样正常topic，下面demo中接着上面的demo继续创建同一个topic，不带有if-not-exists参数和带有if-not-exists参数的效果如下：

```shell
[root@node1 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test1 --replication-factor 2 --partitions 4
Error while executing topic command : Topic 'topic-test1' already exists.
[2018-01-30 17:52:32,425] ERROR org.apache.kafka.common.errors.TopicExistsException: Topic 'topic-test1' already exists.
 (kafka.admin.TopicCommand$)
[root@node1 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test1 --replication-factor 2 --partitions 4 --if-not-exists
[root@node1 kafka_2.12-1.0.0]#
```

接下去还会进一步检测topic名称中是否包含有“.”或者“_”字符的，这一个步骤在AdminUtils.createOrUpdateTopicPartitionAssignmentPathInZK()中调用validateCreateOrUpdateTopic()方法实现的。为什么要检测这两个字符呢？因为在Kafka的内部做埋点时会根据topic的名称来命名metrics的名称，并且会将句点号“.”改成下划线”_”。假设遇到一个topic的名称为“topic.1_2”，还有一个topic的名称为“topic_1.2”，那么最后的metrics的名称都为“topic_1_2”，所以就会发生名称冲突。举例如下，首先创建一个以”topic.1_2”名称的topic，提示WARNING警告，之后在创建一个“topic.1_2”时发生InvalidTopicException异常。

```shell
[root@node2 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic.1_2 --replication-factor 2 --partitions 4
WARNING: Due to limitations in metric names, topics with a period ('.') or underscore ('_') could collide. To avoid issues it is best to use either, but not both.
Created topic "topic.1_2".
[root@node2 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic_1.2 --replication-factor 2 --partitions 4
WARNING: Due to limitations in metric names, topics with a period ('.') or underscore ('_') could collide. To avoid issues it is best to use either, but not both.
Error while executing topic command : Topic 'topic_1.2' collides with existing topics: topic.1_2
[2018-01-31 20:27:28,449] ERROR org.apache.kafka.common.errors.InvalidTopicException: Topic 'topic_1.2' collides with existing topics: topic.1_2
 (kafka.admin.TopicCommand$)
```

> 补充：topic的命名同样不推荐（虽然可以这样做）使用双下划线“__”开头，因为以双下划线开头的topic一般看作是kafka的内部topic，比如__consumer_offsets和__transaction_state。topic的名称必须由大小写字母、数字、“.”、“-”、“_”组成，不能为空、不能为“.”、不能为“..”，且长度不能超过249。

接下去createTopic方法的主体就分为两个部分了，如果检测出有replica-assignment参数，那么就是制定了副本的分配方式。这个在前面都没有提及，那么这个又指的是什么呢？如果包含了replica-assignment参数，那么就可以通过指定的分区副本分配方式创建topic，这个有点绕，不妨再来一个demo开拓下思路：

```shell
[root@node1 kafka_2.12-1.0.0]# bin/kafka-topics.sh --describe --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test1
Topic:topic-test1   PartitionCount:4    ReplicationFactor:2 Configs:
    Topic: topic-test   Partition: 0    Leader: 0   Replicas: 0,1   Isr: 0,1
    Topic: topic-test   Partition: 1    Leader: 1   Replicas: 1,0   Isr: 1,0
    Topic: topic-test   Partition: 2    Leader: 0   Replicas: 0,1   Isr: 0,1
    Topic: topic-test   Partition: 3    Leader: 1   Replicas: 1,0   Isr: 1,0
[root@node1 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test1 --replication-factor 2 --partitions 4 --if-not-exists
[root@node1 kafka_2.12-1.0.0]# bin/kafka-topics.sh --describe --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test1
Topic:topic-test    PartitionCount:4    ReplicationFactor:2 Configs:
    Topic: topic-test   Partition: 0    Leader: 0   Replicas: 0,1   Isr: 0,1
    Topic: topic-test   Partition: 1    Leader: 1   Replicas: 1,0   Isr: 1,0
    Topic: topic-test   Partition: 2    Leader: 0   Replicas: 0,1   Isr: 0,1
    Topic: topic-test   Partition: 3    Leader: 1   Replicas: 1,0   Isr: 1,0
[root@node1 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test2 --replica-assignment 0:1,1:0,0:1,1:0
Created topic "topic-test2".
[root@node1 kafka_2.12-1.0.0]# bin/kafka-topics.sh --describe --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test2
Topic:topic-test2   PartitionCount:4    ReplicationFactor:2 Configs:
    Topic: topic-test2  Partition: 0    Leader: 0   Replicas: 0,1   Isr: 0,1
    Topic: topic-test2  Partition: 1    Leader: 1   Replicas: 1,0   Isr: 1,0
    Topic: topic-test2  Partition: 2    Leader: 0   Replicas: 0,1   Isr: 0,1
    Topic: topic-test2  Partition: 3    Leader: 1   Replicas: 1,0   Isr: 1,
```

可以看到手动指定“–replica-assignment 0:1,1:0,0:1,1:0”后副本的分配方式和自动分配的效果一样。createTopic方法中如果判断opts.options.has(opts.replicaAssignmentOpt)满足条件，那么接下去的工作就是解析并验证指定的副本是否有重复、每个分区的副本个数是否相同等等。如果指定0:0,1:1这种（副本重复）就会报出AdminCommandFailedException异常。详细demo如下：

```shell
[root@node1 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test3 --replica-assignment 0:0,1:1
Error while executing topic command : Partition replica lists may not contain duplicate entries: 0
[2018-02-01 20:23:40,435] ERROR kafka.common.AdminCommandFailedException: Partition replica lists may not contain duplicate entries: 0
    at kafka.admin.TopicCommand$.$anonfun$parseReplicaAssignment$1(TopicCommand.scala:286)
    at scala.collection.immutable.Range.foreach$mVc$sp(Range.scala:156)
    at kafka.admin.TopicCommand$.parseReplicaAssignment(TopicCommand.scala:282)
    at kafka.admin.TopicCommand$.createTopic(TopicCommand.scala:102)
    at kafka.admin.TopicCommand$.main(TopicCommand.scala:63)
    at kafka.admin.TopicCommand.main(TopicCommand.scala)
 (kafka.admin.TopicCommand$)
```

如果指定0:1, 0, 1:0这种（分区副本个数不同）就会报出AdminOperationException异常。详细demo如下：

```shell
[root@node2 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test3 --replica-assignment 0:1,1,0:1,1:0
Error while executing topic command : Partition 1 has different replication factor: [I@159f197
[2018-01-31 20:37:49,136] ERROR kafka.admin.AdminOperationException: Partition 1 has different replication factor: [I@159f197
    at kafka.admin.TopicCommand$.$anonfun$parseReplicaAssignment$1(TopicCommand.scala:289)
    at scala.collection.immutable.Range.foreach$mVc$sp(Range.scala:156)
    at kafka.admin.TopicCommand$.parseReplicaAssignment(TopicCommand.scala:282)
    at kafka.admin.TopicCommand$.createTopic(TopicCommand.scala:102)
    at kafka.admin.TopicCommand$.main(TopicCommand.scala:63)
    at kafka.admin.TopicCommand.main(TopicCommand.scala)
 (kafka.admin.TopicCommand$)
```

当然，像0:1,,0:1,1:0这种企图跳过一个partition连续序号的行为也是不被允许的，详细demo如下：

```shell
[root@node2 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test3 --replica-assignment 0:1,,0:1,1:0
Error while executing topic command : For input string: ""
[2018-02-04 22:14:26,948] ERROR java.lang.NumberFormatException: For input string: ""
    at java.lang.NumberFormatException.forInputString(NumberFormatException.java:65)
    at java.lang.Integer.parseInt(Integer.java:592)
    at java.lang.Integer.parseInt(Integer.java:615)
    at scala.collection.immutable.StringLike.toInt(StringLike.scala:301)
    at scala.collection.immutable.StringLike.toInt$(StringLike.scala:301)
    at scala.collection.immutable.StringOps.toInt(StringOps.scala:29)
    at kafka.admin.TopicCommand$.$anonfun$parseReplicaAssignment$2(TopicCommand.scala:283)
    ......
 (kafka.admin.TopicCommand$)
```

验证之后在zookeeper中创建/brokers/topics/topic-test持久化节点，对应节点的数据就是以json格式呈现的分区分配的结果集，格式参考：{“version”:1,”partitions”:{“2”:[0,1],”1”:[1,0],”3”:[1,0],”0”:[0,1]}}。如果配置了config参数的话，同样先进行验证，如若无误就创建/config/topics/topic-test节点，并写入config对应的数据，格式参考：{“version”:1,”config”:{“max.message.bytes”:”1000013”}}。详细demo如下：

```shell
[root@node2 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test3 --replication-factor 2 --partitions 4 --config key=value
Error while executing topic command : Unknown topic config name: key
[2018-01-31 20:43:23,208] ERROR org.apache.kafka.common.errors.InvalidConfigurationException: Unknown topic config name: key
 (kafka.admin.TopicCommand$)
[root@node2 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test3 --replication-factor 2 --partitions 4 --config max.message.bytes=1000013
Created topic "topic-test3".
```

# 666. 彩蛋

如果你对 Kafka 并发感兴趣，欢迎加入我的知识一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)