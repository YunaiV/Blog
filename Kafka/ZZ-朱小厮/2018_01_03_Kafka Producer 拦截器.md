title: Kafka Producer 拦截器
date: 2018-01-03
tag: 
categories: Kafka
permalink: Kafka/producer-producer
author: 朱小厮
from_url: https://blog.csdn.net/u013256816/article/details/78573425
wechat_url: 

-------

摘要: 原创出处 https://blog.csdn.net/u013256816/article/details/78573425 「朱小厮」欢迎转载，保留摘要，谢谢！

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

Kafka中的拦截器（Interceptor）是0.10.x.x版本引入的一个功能，一共有两种：Kafka Producer端的拦截器和Kafka Consumer端的拦截器。本篇主要讲述的是Kafka Producer端的拦截器，它主要用来对消息进行拦截或者修改，也可以用于Producer的Callback回调之前进行相应的预处理。

使用Kafka Producer端的拦截器非常简单，主要是实现ProducerInterceptor接口，此接口包含4个方法：

1. ProducerRecord<K, V> onSend(ProducerRecord<K, V> record)：Producer在将消息序列化和分配分区之前会调用拦截器的这个方法来对消息进行相应的操作。一般来说最好不要修改消息ProducerRecord的topic、key以及partition等信息，如果要修改，也需确保对其有准确的判断，否则会与预想的效果出现偏差。比如修改key不仅会影响分区的计算，同样也会影响Broker端日志压缩（Log Compaction）的功能。
2. void onAcknowledgement(RecordMetadata metadata, Exception exception)：在消息被应答（Acknowledgement）之前或者消息发送失败时调用，优先于用户设定的Callback之前执行。这个方法运行在Producer的IO线程中，所以这个方法里实现的代码逻辑越简单越好，否则会影响消息的发送速率。
3. void close()：关闭当前的拦截器，此方法主要用于执行一些资源的清理工作。
4. configure(Map<String, ?> configs)：用来初始化此类的方法，这个是ProducerInterceptor接口的父接口Configurable中的方法。

一般情况下只需要关注并实现onSend或者onAcknowledgement方法即可。下面我们来举个案例，通过onSend方法来过滤消息体为空的消息以及通过onAcknowledgement方法来计算发送消息的成功率。

```Java
public class ProducerInterceptorDemo implements ProducerInterceptor<String,String> {
    private volatile long sendSuccess = 0;
    private volatile long sendFailure = 0;

    @Override
    public ProducerRecord<String, String> onSend(ProducerRecord<String, String> record) {
        if(record.value().length()<=0)
            return null;
        return record;
    }

    @Override
    public void onAcknowledgement(RecordMetadata metadata, Exception exception) {
        if (exception == null) {
            sendSuccess++;
        } else {
            sendFailure ++;
        }
    }

    @Override
    public void close() {
        double successRatio = (double)sendSuccess / (sendFailure + sendSuccess);
        System.out.println("[INFO] 发送成功率="+String.format("%f", successRatio * 100)+"%");
    }

    @Override
    public void configure(Map<String, ?> configs) {}
}
```

自定义的ProducerInterceptorDemo类实现之后就可以在Kafka Producer的主程序中指定，示例代码如下：

```Java
public class ProducerMain {
    public static final String brokerList = "localhost:9092";
    public static final String topic = "hidden-topic";

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        Properties properties = new Properties();
        properties.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        properties.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        properties.put("bootstrap.servers", brokerList);
        properties.put("interceptor.classes", "com.hidden.producer.ProducerInterceptorDemo");

        Producer<String, String> producer = new KafkaProducer<String, String>(properties);

        for(int i=0;i<100;i++) {
            ProducerRecord<String, String> producerRecord = new ProducerRecord<String, String>(topic, "msg-" + i);
            producer.send(producerRecord).get();
        }
        producer.close();
    }
}
```

Kafka Producer不仅可以指定一个拦截器，还可以指定多个拦截器以形成拦截链，这个拦截链会按照其中的拦截器的加入顺序一一执行。比如上面的程序多添加一个拦截器，示例如下：

```
properties.put("interceptor.classes", "com.hidden.producer.ProducerInterceptorDemo,com.hidden.producer.ProducerInterceptorDemoPlus");1
```

这样Kafka Producer会先执行拦截器ProducerInterceptorDemo，之后再执行ProducerInterceptorDemoPlus。

有关interceptor.classes参数，在kafka 1.0.0版本中的定义如下：

| NAME                 | DESCRIPTION                                                  | TYPE | DEFAULT | VALID VALUES | IMPORTANCE |
| -------------------- | ------------------------------------------------------------ | ---- | ------- | ------------ | ---------- |
| interceptor.calssses | A list of classes to use as interceptors. Implementing the org.apache.kafka.clients.producer.ProducerInterceptor interface allows you to intercept (and possibly mutate) the records received by the producer before they are published to the Kafka cluster. By default, there no interceptors. | list | null    |              | low        |

# 666. 彩蛋

如果你对 Kafka 并发感兴趣，欢迎加入我的知识一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)