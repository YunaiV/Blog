title: Kafka 解析之 topic 创建(3) —— 合法性验证
date: 2018-01-15
tag: 
categories: Kafka
permalink: Kafka/topic-create-3
author: 朱小厮
from_url: https://blog.csdn.net/u013256816/article/details/79996208
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

在《[Kafka解析之Topic创建(1)](https://blog.csdn.net/u013256816/article/details/79303825)》这篇文章中，我们讲述了创建Topic的方式有两种：

1. 如果kafka broker中的config/server.properties配置文件中配置了auto.create.topics.enable参数为true（默认值就是true），那么当生产者向一个尚未创建的topic发送消息时，会自动创建一个num.partitions（默认值为1）个分区和default.replication.factor（默认值为1）个副本的对应topic。
2. 通过kafka提供的kafka-topics.sh脚本来创建，或者相关的变种方式(比如在zookeeper中的/config/topics/路径下直接创建)。

在学习了KafkaAdminClient之后我们发现它也可以用来创建Topic，即通过发送CreateTopicsRequest请求的方式来创建。KafkaAdminClient的详细内容可以参考：《[集群管理工具KafkaAdminClient——原理与示例](https://blog.csdn.net/u013256816/article/details/79996056)》和《[集群管理工具KafkaAdminClient——改造](https://blog.csdn.net/u013256816/article/details/79996138)》。

------

一般情况下，Kafka生产环境中的 auto.create.topics.enable参数会被修改为false，即自动创建Topic这条路会被堵住。kafka-topics.sh脚本创建的方式一般由运维人员操作，普通用户无权过问。那么KafkaAdminClient就为普通用户提供了一个口子，或者将其集成到公司内部的资源申请、审核系统中更加的方便。普通用户在创建Topic的时候，有可能由于误操作或者其他原因而创建了不符合运维规范的Topic，比如命名不规范，副本因子数太低等，这些都会影响后期的系统运维。如果创建Topic的操作是封装在资源申请、审核系统中的话，那么可以在前端就可以根据规则过滤掉不符合规范的申请操作。然而如果用户就是用了KafkaAdminClient或者类似的工具来创建了一个错误的Topic，我们有什么办法可以做相应的规范处理呢?

在Kafka服务端中提供了这样一个参数：create.topic.policy.class.name，其提供了一个入口用来验证Topic创建的合法性。使用方式是自定义实现org.apache.kafka.server.policy.CreateTopicPolicy接口，比如下面的PolicyDemo，然后在kafka broker中的config/server.properties配置文件中配置参数create.topic.policy.class.name=org.apache.kafka.server.policy.PolicyDemo，然后启动Kafka服务即可。PolicyDemo的代码参考如下，主要实现接口中的configure、close以及validate方法，configure方法会在Kafka服务启动的时候执行，validate方法用来鉴定Topic参数的合法性，其在创建Topic的时候执行，close方法在关闭Kafka服务的时候执行。

```java
public class PolicyDemo implements CreateTopicPolicy{
   public void configure(Map<String, ?> configs) {}
   public void close() throws Exception {}
   public void validate(RequestMetadata requestMetadata)
         throws PolicyViolationException {
      if(requestMetadata.numPartitions()!=null || requestMetadata.replicationFactor()!=null){
         if(requestMetadata.numPartitions()< 5){
            throw new PolicyViolationException("Topic should have at least 5 partitions, received: "
                  + requestMetadata.numPartitions());
         }
         if(requestMetadata.replicationFactor()<= 1){
            throw new PolicyViolationException("Topic should have at least 2 replication factor, recevied: "
                  + requestMetadata.replicationFactor());
         }}}}
```

采用文章《[集群管理工具KafkaAdminClient——原理与示例](https://blog.csdn.net/u013256816/article/details/79996056)》中的所提及的关于KafkaAdminClient来创建Topic，测试代码如下，创建一个分区数为4，副本数为1的Topic：

```java
@Test
public void createTopics() {
    NewTopic newTopic = new NewTopic(NEW_TOPIC,4, (short) 1);
    Collection<NewTopic> newTopicList = new ArrayList<>();
    newTopicList.add(newTopic);
    CreateTopicsResult result = adminClient.createTopics(newTopicList);
    try {
        result.all().get();
    } catch (InterruptedException e) {
        e.printStackTrace();
    } catch (ExecutionException e) {
        e.printStackTrace();
    }
}
```

测试结果如期报错：

```java
java.util.concurrent.ExecutionException: org.apache.kafka.common.errors.PolicyViolationException: Topic should have at least 5 partitions, received: 4
```

相应的Kafka服务端的日志如下：

```java
CreateTopicPolicy.RequestMetadata(topic=topic-test2, numPartitions=4, replicationFactor=1, replicasAssignments=null, configs={})
[2018-04-18 19:52:02,747] INFO [Admin Manager on Broker 0]: Error processing create topic request for topic topic-test2 with arguments (numPartitions=4, replicationFactor=1, replicasAssignments={}, configs={}) (kafka.server.AdminManager)
org.apache.kafka.common.errors.PolicyViolationException: Topic should have at least 5 partitions, received: 4
```

客户端向Kafka服务端发送了CreateTopicsRequest请求之后，会经过KafkaApis：

```Java
case ApiKeys.CREATE_TOPICS => handleCreateTopicsRequest(request)
```

然后调用handleCreateTopicsRequest()方法，Topic最终在服务端的创建是在AdminManager中的createTopics方法中实现的。而CreateTopicPolicy的作用域也限定在这个createTopics方法之内，故只有通过CreateTopicsRequest请求的方式才能促使CreateTopicPolicy有效，而对于类似于kafka-topics.sh脚本的创建方式无效。不过在正文开头就提及了在运维规范的情况下，一般是通过KafkaAdminClient进行操作，或者更加规范的话直接通过申请页面来创建，这样就可以在前端规避风险，这样显得更加的专业。

# 666. 彩蛋

如果你对 Kafka 并发感兴趣，欢迎加入我的知识一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)