title: Kafka 分区分配计算(分区器 Partitions )
date: 2018-01-06
tag: 
categories: Kafka
permalink: Kafka/partitions
author: 朱小厮
from_url: https://blog.csdn.net/u013256816/article/details/78703111
wechat_url: 

-------

摘要: 原创出处 https://blog.csdn.net/u013256816/article/details/78703111 「朱小厮」欢迎转载，保留摘要，谢谢！

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

KafkaProducer在调用send方法发送消息至broker的过程中，首先是经过拦截器Inteceptors处理，然后是经过序列化Serializer处理，之后就到了Partitions阶段，即分区分配计算阶段。在某些应用场景下，业务逻辑需要控制每条消息落到合适的分区中，有些情形下则只要根据默认的分配规则即可。在KafkaProducer计算分配时，首先根据的是ProducerRecord中的partition字段指定的序号计算分区。读者有可能刚睡醒，看到这个ProducerRecord似曾相识，没有关系，先看段Kafka生产者的示例片段：

```java
Producer<String,String> producer = new KafkaProducer<String,String>(properties);
String message = "kafka producer demo";
ProducerRecord<String, String> producerRecord = new ProducerRecord<String, String>(topic,message);
try {
    producer.send(producerRecord).get();
} catch (InterruptedException e) {
    e.printStackTrace();
} catch (ExecutionException e) {
    e.printStackTrace();
}
```

没错，ProducerRecord只是一个封装了消息的对象而已，ProducerRecord一共有5个成员变量，即：

```java
private final String topic;//所要发送的topic
private final Integer partition;//指定的partition序号
private final Headers headers;//一组键值对，与RabbitMQ中的headers类似，kafka0.11.x版本才引入的一个属性
private final K key;//消息的key
private final V value;//消息的value,即消息体
private final Long timestamp;//消息的时间戳，可以分为Create_Time和LogAppend_Time之分，这个以后的文章中再表。123456
```

在KafkaProducer的源码（1.0.0）中，计算分区时调用的是下面的partition()方法：

```java
/**
 * computes partition for given record.
 * if the record has partition returns the value otherwise
 * calls configured partitioner class to compute the partition.
 */
private int partition(ProducerRecord<K, V> record, byte[] serializedKey, byte[] serializedValue, Cluster cluster) {
    Integer partition = record.partition();
    return partition != null ?
            partition :
            partitioner.partition(record.topic(), record.key(), serializedKey, record.value(), serializedValue, cluster);
}
```

可以看出的确是先判断有无指明ProducerRecord的partition字段，如果没有指明，则再进一步计算分区。上面这段代码中的partitioner在默认情况下是指Kafka默认实现的org.apache.kafka.clients.producer.DefaultPartitioner，其partition()方法实现如下：

```java
/**
 * Compute the partition for the given record.
 *
 * @param topic The topic name
 * @param key The key to partition on (or null if no key)
 * @param keyBytes serialized key to partition on (or null if no key)
 * @param value The value to partition on or null
 * @param valueBytes serialized value to partition on or null
 * @param cluster The current cluster metadata
 */
public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
    List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
    int numPartitions = partitions.size();
    if (keyBytes == null) {
        int nextValue = nextValue(topic);
        List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
        if (availablePartitions.size() > 0) {
            int part = Utils.toPositive(nextValue) % availablePartitions.size();
            return availablePartitions.get(part).partition();
        } else {
            // no partitions are available, give a non-available partition
            return Utils.toPositive(nextValue) % numPartitions;
        }
    } else {
        // hash the keyBytes to choose a partition
        return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
    }
}

private int nextValue(String topic) {
    AtomicInteger counter = topicCounterMap.get(topic);
    if (null == counter) {
        counter = new AtomicInteger(ThreadLocalRandom.current().nextInt());
        AtomicInteger currentCounter = topicCounterMap.putIfAbsent(topic, counter);
        if (currentCounter != null) {
            counter = currentCounter;
        }
    }
    return counter.getAndIncrement();
}
```

由上源码可以看出partition的计算方式：

1. 如果key为null，则按照一种轮询的方式来计算分区分配
2. 如果key不为null则使用称之为murmur的Hash算法（非加密型Hash函数，具备高运算性能及低碰撞率）来计算分区分配。

KafkaProducer中还支持自定义分区分配方式，与org.apache.kafka.clients.producer.internals.DefaultPartitioner一样首先实现org.apache.kafka.clients.producer.Partitioner接口，然后在KafkaProducer的配置中指定partitioner.class为对应的自定义分区器（Partitioners）即可，即：

```java
properties.put("partitioner.class","com.hidden.partitioner.DemoPartitioner");
```

自定义DemoPartitioner主要是实现Partitioner接口的public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster)的方法。DemoPartitioner稍微修改了下DefaultPartitioner的计算方式，详细参考如下：

```Java
public class DemoPartitioner implements Partitioner {

    private final AtomicInteger atomicInteger = new AtomicInteger(0);

    @Override
    public void configure(Map<String, ?> configs) {}

    @Override
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();
        if (null == keyBytes || keyBytes.length<1) {
            return atomicInteger.getAndIncrement() % numPartitions;
        }
        //借用String的hashCode的计算方式
        int hash = 0;
        for (byte b : keyBytes) {
            hash = 31 * hash + b;
        }
        return hash % numPartitions;
    }

    @Override
    public void close() {}
}
```

这个自定义分区器的实现比较简单，读者可以根据自身业务的需求来灵活实现分配分区的计算方式，比如：一般大型电商都有多个仓库，可以将仓库的名称或者ID作为Key来灵活的记录商品信息。

# 666. 彩蛋

如果你对 Kafka 并发感兴趣，欢迎加入我的知识一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)