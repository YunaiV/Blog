title: Kafka 解惑之 Old Producer（1）—— Beginning
date: 2018-01-09
tag: 
categories: Kafka
permalink: Kafka/old-producer-beginning
author: 朱小厮
from_url: https://blog.csdn.net/u013256816/article/details/79193847
wechat_url: 

-------

摘要: 原创出处 https://blog.csdn.net/u013256816/article/details/79193847 「朱小厮」欢迎转载，保留摘要，谢谢！

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

众所周知，目前Kafka的最新版本已经到达1.0.0，很多公司运行的kafka也大多升级到了0.10.x版本，Kafka的Producer客户端早已不再使用0.8.2.x就已基本停止维护的Scala版本的Producer了，那么我们还有必要了解它么？当然很有必要，通过Kafka Old Producer我们可以了解Kafka变迁升级的历史：旧版的Old Producer模型相对简单利于初始了解，通过对Old Producer的了解也可以慢慢的发现隐患的问题，这样进一步可以研究探讨解决方法，最后再通过对新版Producer的学习来提升对Kafka的认知，与此同时也可以让读者在遇到相似问题的时候可以借鉴Kafka的优化过来来优化自己的应用。以铜为鉴，可以正衣冠。

在使用Scala版本的Kafka生产者客户端kafka.javaapi.producer.Producer时，实际上调用的是kafka.producer.Producer类。

```java
package kafka.javaapi.producer
class Producer[K, V](private val underlying : kafka.producer.Producer[K, V]) extends scala.AnyRef {
  def this(config : kafka.producer.ProducerConfig) = { /* compiled code */ }
  def send(message : kafka.producer.KeyedMessage[K, V]) : scala.Unit = { /* compiled code */ }
  def send(messages : java.util.List[kafka.producer.KeyedMessage[K, V]]) : scala.Unit = { /* compiled code */ }
  def close : scala.Unit = { /* compiled code */ }
}
```

包括kafka-console-producer.sh的脚本（常用来测试发送消息之用）中，对于0.8.2.x版本如果不指定“– new-producer”参数；或者对于.0.0版本如果指定“– old-producer”参数的话，实际上内部调用的都是kafka.producer.Producer这个类。

对于kafka-console-producer.sh脚本的内容如下：

```shell
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx512M"
fi
exec $(dirname $0)/kafka-run-class.sh kafka.tools.ConsoleProducer "$@"
```

我们看到实际上kafka-console-producer.sh的内容就是运行kafka.tools.ConsoleProducer而已，可以看到main函数代码块中的config.useOldProducer，这个笔者看的是1.0.0版本的代码，而0.8.2.2版本中的ConsoleProducer对应的是config.useNewProducer，稍有不同而已，不过如果都指定使用旧版的Scala的Producer，那么都是指kafka.producer.OldProducer。

```java
object ConsoleProducer {
  def main(args: Array[String]) {
    try {
        val config = new ProducerConfig(args)
        val reader = Class.forName(config.readerClass).newInstance().asInstanceOf[MessageReader]
        reader.init(System.in, getReaderProps(config))

        val producer =
          if(config.useOldProducer) {
            new OldProducer(getOldProducerProps(config))
          } else {
            new NewShinyProducer(getNewProducerProps(config))
          }
```

进一步剖析，kafka.producer.OldProducer的内部构造很简单，关键代码如下：

```java
class OldProducer(producerProps: Properties) extends BaseProducer {
  // default to byte array partitioner
  if (producerProps.getProperty("partitioner.class") == null)
    producerProps.setProperty("partitioner.class", classOf[kafka.producer.ByteArrayPartitioner].getName)
  val producer = new kafka.producer.Producer[Array[Byte], Array[Byte]](new ProducerConfig(producerProps))
```

可以看到内部的producer最终还是实例化的kafka.producer.Producer。最终验证了开篇所述的旧版的Kafka生产者客户端即为Kafka.producer.Producer。

> 新版的Java版的Kafka客户端是：org.apache.kafka.clients.producer.KafkaProducer，读者请注意区分。对于新版的KafkaProducer在以后的文章中会有详细介绍。

下面就来深入了解下Kafka.producer.Producer（下面如无特殊说明都将Kafka.producer.Producer此简称为Producer）了。当实例化Producer的时候，首先要读取、解析以及校验配置信息的合法性，根据配置信息来实例化Producer。Producer的配置项有18个，比如设置分区器、消息压缩方式等，这些都比较好理解，而最主要的配置就是request.required.acks和producer.type这两个配置。

request.required.acks是用来配置生产端消息确认的方式，在0.8.x这个系列的版本之中，可以配置为0,1，-1的值，也可以配置为其他的整数值，用来控制一条消息经由多少个ISR中的副本所在的Broker确认之后才向客户端发送确认信息，这个参数在之后的版本，比如1.0.0版本中就只能设置0,1，-1(all)这3（4）种取值，分别表示：

1. 当request.required.acks=0时，这意味着producer无需等待来自broker的确认而继续发送下一批消息。这种情况下数据传输效率最高，但是数据可靠性确是最低的。
2. 当request.required.acks=1（默认）时，这意味着producer在ISR中的leader已成功收到数据并得到确认。如果leader宕机了，则会丢失数据。
3. 当request.required.acks=-1时，producer需要等待ISR中的所有follower都确认接收到数据后才算一次发送完成，可靠性最高。但是这样也不能保证数据不丢失，比如当ISR中只有leader时，这样就变成了acks=1的情况。为了提高数据的可靠性，可以通过min.insync.replicas参数来辅助作用，当同步副本数不足时，生产者会跑出异常。

有关kafka的消息可靠性的更深层次的讲解可以参考我2017年初的一篇博客：[kafka数据可靠性深度解读](http://blog.csdn.net/u013256816/article/details/71091774)，这篇博客主要是针对0.8.2.x版本的kafka做深层次的探讨，后续会对1.0.0版本做进一步的说明。

Producer的发送模式分为同步（sync）和异步（async）两种情况，这一点可以通过参数producer.type来配置。同步模式会将消息直接发往broker中，而异步模式则会将消息存入LinkedBlockingQueue中，然后通过一个ProducerSendThread来专门发送消息。为了便于说明，笔者这里先对同步模式的情况来做说明，而异步模式只是在同步模式的基础上做了一些封装而已。

```java
class Producer[K,V](val config: ProducerConfig,
                    private val eventHandler: EventHandler[K,V])  // only for unit testing
  extends Logging {

  private val hasShutdown = new AtomicBoolean(false)
  private val queue = new LinkedBlockingQueue[KeyedMessage[K,V]](config.queueBufferingMaxMessages)

  private var sync: Boolean = true
  private var producerSendThread: ProducerSendThread[K,V] = null
  private val lock = new Object()

  config.producerType match {
    case "sync" =>
    case "async" =>
      sync = false
      producerSendThread = new ProducerSendThread[K,V]("ProducerSendThread-" + config.clientId,
                                                       queue,
                                                       eventHandler,
                                                       config.queueBufferingMaxMs,
                                                       config.batchNumMessages,
                                                       config.clientId)
      producerSendThread.start()
  }
```

在讲述Producer的具体行为之前先来看一个发送方的Demo:

```java
public class ProducerScalaDemo {
    public static final String brokerList = "xxx.xxx.xxx.xxx:9092";
    public static final String topic = "topic-zzh";

    public static void main(String[] args) {
        Properties properties = new Properties();
        properties.put("serializer.class", "kafka.serializer.StringEncoder");
        properties.put("metadata.broker.list", brokerList);
        properties.put("producer.type", "sync");
        properties.put("request.required.acks", "1");

        Producer<String, String> producer = new Producer<String, String>(new ProducerConfig(properties));

        String message = "kafka_message-" + new Date().getTime() + " edited by hidden.zhu";
        KeyedMessage<String, String> keyedMessage = new KeyedMessage<String, String>(topic,null, message);
        producer.send(keyedMessage);
    }
}
```

我们可以看到再初始化Producer的时候之用了ProducerConfig这一个类型的参数，而在Producer的类定义中还用到了EventHandler这个类型的参数。在Scala语言中只有一个主构造函数，这个主构造函数的参数列表就是跟在类名后面括号中的各个的参数，如果要重载的话就需要自定义辅助构造函数，辅助构造函数必须调用主构造函数（this方法）。如此上面这个Demo中很显然的就调用了辅助构造函数来进行实例化，那么我们再来看下其对应的辅助构造函数：

```java
def this(config: ProducerConfig) =
  this(config,
       new DefaultEventHandler[K,V](config,
                                    CoreUtils.createObject[Partitioner](config.partitionerClass, config.props),
                                    CoreUtils.createObject[Encoder[V]](config.serializerClass, config.props),
                                    CoreUtils.createObject[Encoder[K]](config.keySerializerClass, config.props),
                                    new ProducerPool(config)))
```

这里又引入了两个新的东西：DefaultEventHandler和ProducerPool，这个DefaultEventHandler继承了EventHandler这个类，这个是消息发送的关键。而ProducerPool内部是一个HashMap，其中的key是broker的id，而value就是每个broker对应的SyncProducer，这个SyncProducer就是真正的消息发送者。

# 666. 彩蛋

如果你对 Kafka 并发感兴趣，欢迎加入我的知识一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)