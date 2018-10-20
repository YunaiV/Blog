title: Java 分布式跟踪系统 Zipkin（八）：Zipkin 源码分析 —— KafkaCollector
date: 2018-01-09
tag: 
categories: Zipkin
permalink: Zipkin/mozhu/KafkaCollector
author: v墨竹v
from_url: https://blog.csdn.net/apei830/article/details/78722430
wechat_url: 

-------

摘要: 原创出处 https://blog.csdn.net/apei830/article/details/78722430 「v墨竹v」欢迎转载，保留摘要，谢谢！

  - [KafkaSender](http://www.iocoder.cn/Zipkin/mozhu/KafkaCollector/)
  - [KafkaCollector](http://www.iocoder.cn/Zipkin/mozhu/KafkaCollector/)
  - [ZipkinKafkaCollectorAutoConfiguration](http://www.iocoder.cn/Zipkin/mozhu/KafkaCollector/)
  - [KafkaZooKeeperSetCondition](http://www.iocoder.cn/Zipkin/mozhu/KafkaCollector/)
  - [LazyConnector](http://www.iocoder.cn/Zipkin/mozhu/KafkaCollector/)
  - [LazyStreams](http://www.iocoder.cn/Zipkin/mozhu/KafkaCollector/)
  - [KafkaStreamProcessor](http://www.iocoder.cn/Zipkin/mozhu/KafkaCollector/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

前面几篇博文中，都是使用OkHttpSender来上报Trace信息给Zipkin，这在生产环境中，当业务量比较大的时候，可能会成为一个性能瓶颈，这一篇博文我们来使用KafkaSender将Trace信息先写入到Kafka中，然后Zipkin使用KafkaCollector从Kafka中收集Span信息。
在Brave配置中需要将Sender设置为KafkaSender，而zipkin的collector组件配置为KafkaCollector

相关代码在Chapter8/zipkin-kafka中
pom.xml中添加依赖

```xml
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-sender-kafka11</artifactId>
    <version>${zipkin-reporter2.version}</version>
</dependency>
```

TracingConfiguration中，我们修改Sender为KafkaSender，指定Kafka的地址，以及topic

```java
@Bean
Sender sender() {
	return KafkaSender.newBuilder().bootstrapServers("localhost:9091,localhost:9092,localhost:9093").topic("zipkin").encoding(Encoding.JSON).build();
}
```

我们先启动zookeeper（默认端口号为2181），再依次启动一个本地的3个broker的kafka集群（端口号分别为9091、9092、9093），最后启动一个KafkaManager（默认端口号9000），KafkaManager是Kafka的UI管理工具
关于如何搭建本地Kafka伪集群，请自行上网搜索教程，本文使用的Kafka版本为0.10.0.0。

kafka启动完毕后，我们创建名为zipkin的topic，因为我们有3个broker，我这里设置replication-factor=3

```shell
bin/windows/kafka-topics.bat --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic zipkin
```

打开KafkaManager界面
<http://localhost:9000/clusters/localhost/topics/zipkin>
[![KafkaManager](http://static.blog.mozhu.org/images/zipkin/7_1.png)](http://static.blog.mozhu.org/images/zipkin/7_1.png)KafkaManager
可以看到topic zipkin中暂时没有消息。

我们使用如下命令启动zipkin，带上Kafka的Zookeeper地址参数，这样zipkin就会从kafka中消费我们上报的trace信息。

```shell
java -jar zipkin-server-2.2.1-exec.jar --KAFKA_ZOOKEEPER=localhost:2181
```

然后分别运行，主意我们这里将backend的端口改为9001，目的是为了避免和KafkaManager端口号冲突。

```shell
mvn spring-boot:run -Drun.jvmArguments="-Dserver.port=9001 -Dzipkin.service=backend"
```

```shell
mvn spring-boot:run -Drun.jvmArguments="-Dserver.port=8081 -Dzipkin.service=frontend"
```

浏览器访问 <http://localhost:8081/> 会显示当前时间

我们再次刷新KafkaManager界面
<http://localhost:9000/clusters/localhost/topics/zipkin>
[![KafkaManager](http://static.blog.mozhu.org/images/zipkin/7_2.png)](http://static.blog.mozhu.org/images/zipkin/7_2.png)KafkaManager
可以看到topic zipkin中有两条消息。

为了看到这两条消息的具体内容，我们可以在kafka安装目录使用如下命令

```shell
bin/windows/kafka-console-consumer.bat --zookeeper localhost:2181 --topic zipkin --from-beginning
```

在控制台会打印出最近的两条消息

```json
[{"traceId":"802bd09f480b5faa","parentId":"802bd09f480b5faa","id":"bb3c70909ea3ee3c","kind":"SERVER","name":"get","timestamp":1510891296426607,"duration":10681,"localEndpoint":{"serviceName":"backend","ipv4":"10.200.170.137"},"remoteEndpoint":{"ipv4":"127.0.0.1","port":64421},"tags":{"http.path":"/api"},"shared":true}]
[{"traceId":"802bd09f480b5faa","parentId":"802bd09f480b5faa","id":"bb3c70909ea3ee3c","kind":"CLIENT","name":"get","timestamp":1510891296399882,"duration":27542,"localEndpoint":{"serviceName":"frontend","ipv4":"10.200.170.137"},"tags":{"http.path":"/api"}},{"traceId":"802bd09f480b5faa","id":"802bd09f480b5faa","kind":"SERVER","name":"get","timestamp":1510891296393252,"duration":39514,"localEndpoint":{"serviceName":"frontend","ipv4":"10.200.170.137"},"remoteEndpoint":{"ipv6":"::1","port":64420},"tags":{"http.path":"/"}}]
```

这说明我们的应用frontend和backend已经将trace信息写入kafka成功了！

在Zipkin的Web界面中，也能查询到这次跟踪信息
在zipkin的控制台，我们也看到跟Kafka相关的类ConsumerFetcherThread启动，我们在后续专门分析zipkin的源代码再来看看这个类。

```shell
2017-11-17 11:25:00.477  INFO 9292 --- [49-8e18eab0-0-1] kafka.consumer.ConsumerFetcherThread     : [ConsumerFetcherThread-zipkin_LT290-1510889099649-8e18eab0-0-1], Starting
2017-11-17 11:25:00.482  INFO 9292 --- [r-finder-thread] kafka.consumer.ConsumerFetcherManager    : [ConsumerFetcherManager-1510889099800] Added fetcher for partitions ArrayBuffer([[zipkin,0], initOffset 0 to broker id:1,host:10.200.170.137,port:9091] )
```

## KafkaSender

```java
public abstract class KafkaSender extends Sender {
  public static Builder newBuilder() {
    // Settings below correspond to "Producer Configs"
    // http://kafka.apache.org/0102/documentation.html#producerconfigs
    Properties properties = new Properties();
    properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class.getName());
    properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
        ByteArraySerializer.class.getName());
    properties.put(ProducerConfig.ACKS_CONFIG, "0");
    return new zipkin2.reporter.kafka11.AutoValue_KafkaSender.Builder()
        .encoding(Encoding.JSON)
        .properties(properties)
        .topic("zipkin")
        .overrides(Collections.EMPTY_MAP)
        .messageMaxBytes(1000000);
  }

  @Override public zipkin2.Call<Void> sendSpans(List<byte[]> encodedSpans) {
    if (closeCalled) throw new IllegalStateException("closed");
    byte[] message = encoder().encode(encodedSpans);
    return new KafkaCall(message);
  }

}
```

KafkaSender中通过KafkaProducer客户端来发送消息给Kafka，在newBuilder方法中，设置了一些默认值，比如topic默认为zipkin，编码默认用JSON，消息最大字节数1000000，还可以通过overrides来覆盖默认的配置来定制KafkaProducer。

在sendSpans方法中返回KafkaCall，这个对象的execute方法，在AsyncReporter中的flush方法中会被调用：

```java
void flush(BufferNextMessage bundler) {
	// ...
	sender.sendSpans(nextMessage).execute();
	// ...
}
```

KafkaCall的父类BaseCall方法execute会调用doExecute，而在doExecute方法中使用了一个AwaitableCallback将KafkaProducer的异步发送消息的方法，强制转为了同步发送，这里也确实处理的比较优雅。

```java
class KafkaCall extends BaseCall<Void> { // KafkaFuture is not cancelable
  private final byte[] message;

  KafkaCall(byte[] message) {
    this.message = message;
  }

  @Override protected Void doExecute() throws IOException {
    final AwaitableCallback callback = new AwaitableCallback();
    get().send(new ProducerRecord<>(topic(), message), (metadata, exception) -> {
      if (exception == null) {
        callback.onSuccess(null);
      } else {
        callback.onError(exception);
      }
    });
    callback.await();
    return null;
  }

  @Override protected void doEnqueue(Callback<Void> callback) {
    get().send(new ProducerRecord<>(topic(), message), (metadata, exception) -> {
      if (exception == null) {
        callback.onSuccess(null);
      } else {
        callback.onError(exception);
      }
    });
  }

  @Override public Call<Void> clone() {
    return new KafkaCall(message);
  }
}
```

这里还有一个知识点，get方法每次都会返回一个新的KafkaProducer，我在第一眼看到这段代码时也曾怀疑，难道这里没有性能问题？
原来这里用到了google的插件autovalue里的标签@Memoized，结合@AutoValue标签，它会在自动生成的类里，给我们添加一些代码，可以看到get方法里作了一层缓存，所以我们的担心是没有必要的

```java
@Memoized KafkaProducer<byte[], byte[]> get() {
  KafkaProducer<byte[], byte[]> result = new KafkaProducer<>(properties());
  provisioned = true;
  return result;
}
```

AutoValue_KafkaSender

```java
final class AutoValue_KafkaSender extends $AutoValue_KafkaSender {
  private volatile KafkaProducer<byte[], byte[]> get;

  AutoValue_KafkaSender(Encoding encoding$, int messageMaxBytes$, BytesMessageEncoder encoder$,
      String topic$, Properties properties$) {
    super(encoding$, messageMaxBytes$, encoder$, topic$, properties$);
  }

  @Override
  KafkaProducer<byte[], byte[]> get() {
    if (get == null) {
      synchronized (this) {
        if (get == null) {
          get = super.get();
          if (get == null) {
            throw new NullPointerException("get() cannot return null");
          }
        }
      }
    }
    return get;
  }
}
```

## KafkaCollector

我们再来看下Zipkin中的KafkaCollector，我们打开zipkin-server的源代码，在目录resources/zipkin-server-shared.yml文件中，发现关于kafka的配置片段
而我们在本文前面使用–KAFKA_ZOOKEEPER启动了zipkin，将kafka的zookeeper参数传递给了KafkaServer的main方法，也就是说，我们制定了zipkin.collector.kafka.zookeeper的值为localhost:2181

```shell
java -jar zipkin-server-2.2.1-exec.jar --KAFKA_ZOOKEEPER=localhost:2181
```

zipkin-server-shared.yml

```yaml
zipkin:
  collector:
    kafka:
      # ZooKeeper host string, comma-separated host:port value.
      zookeeper: ${KAFKA_ZOOKEEPER:}
      # Name of topic to poll for spans
      topic: ${KAFKA_TOPIC:zipkin}
      # Consumer group this process is consuming on behalf of.
      group-id: ${KAFKA_GROUP_ID:zipkin}
      # Count of consumer threads consuming the topic
      streams: ${KAFKA_STREAMS:1}
      # Maximum size of a message containing spans in bytes
      max-message-size: ${KAFKA_MAX_MESSAGE_SIZE:1048576}
```

在pom.xml中，有如下依赖

```xml
<!-- Kafka Collector -->
<dependency>
  <groupId>${project.groupId}</groupId>
  <artifactId>zipkin-autoconfigure-collector-kafka</artifactId>
  <optional>true</optional>
</dependency>
```

## ZipkinKafkaCollectorAutoConfiguration

我们找到zipkin-autoconfigure/collector-kafka的ZipkinKafkaCollectorAutoConfiguration类，使用了@Conditional注解，当KafkaZooKeeperSetCondition条件满足时，ZipkinKafkaCollectorAutoConfiguration类会被SpringBoot加载。当加载时，会配置KafkaCollector到spring容器中。

```java
@Configuration
@EnableConfigurationProperties(ZipkinKafkaCollectorProperties.class)
@Conditional(KafkaZooKeeperSetCondition.class)
public class ZipkinKafkaCollectorAutoConfiguration {

  /**
   * This launches a thread to run start. This prevents a several second hang, or worse crash if
   * zookeeper isn't running, yet.
   */
  @Bean KafkaCollector kafka(ZipkinKafkaCollectorProperties kafka, CollectorSampler sampler,
      CollectorMetrics metrics, StorageComponent storage) {
    final KafkaCollector result =
        kafka.toBuilder().sampler(sampler).metrics(metrics).storage(storage).build();

    // don't use @Bean(initMethod = "start") as it can crash the process if zookeeper is down
    Thread start = new Thread("start " + result.getClass().getSimpleName()) {
      @Override public void run() {
        result.start();
      }
    };
    start.setDaemon(true);
    start.start();

    return result;
  }
}
```

## KafkaZooKeeperSetCondition

KafkaZooKeeperSetCondition继承了SpringBootCondition，实现了getMatchOutcome方法，当上下文的环境变量中有配置zipkin.collector.kafka.zookeeper的时候，则条件满足，即ZipkinKafkaCollectorAutoConfiguration会被加载

```java
final class KafkaZooKeeperSetCondition extends SpringBootCondition {
  static final String PROPERTY_NAME = "zipkin.collector.kafka.zookeeper";

  @Override
  public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata a) {
    String kafkaZookeeper = context.getEnvironment().getProperty(PROPERTY_NAME);
    return kafkaZookeeper == null || kafkaZookeeper.isEmpty() ?
        ConditionOutcome.noMatch(PROPERTY_NAME + " isn't set") :
        ConditionOutcome.match();
  }
}
```

在ZipkinKafkaCollectorAutoConfiguration中，启动了一个守护线程来运行KafkaCollector的start方法，避免zookeeper连不上，阻塞zipkin的启动过程。

```java
public final class KafkaCollector implements CollectorComponent {
  final LazyConnector connector;
  final LazyStreams streams;

  KafkaCollector(Builder builder) {
    connector = new LazyConnector(builder);
    streams = new LazyStreams(builder, connector);
  }

  @Override public KafkaCollector start() {
    connector.get();
    streams.get();
    return this;
  }
}
```

KafkaCollector中初始化了两个对象，LazyConnector，和LazyStreams，在start方法中调用了2个对象的get方法

## LazyConnector

LazyConnector继承了Lazy，当get方法被调用的时候，compute方法会被调用

```java
static final class LazyConnector extends LazyCloseable<ZookeeperConsumerConnector> {

  final ConsumerConfig config;

  LazyConnector(Builder builder) {
    this.config = new ConsumerConfig(builder.properties);
  }

  @Override protected ZookeeperConsumerConnector compute() {
    return (ZookeeperConsumerConnector) createJavaConsumerConnector(config);
  }

  @Override
  public void close() {
    ZookeeperConsumerConnector maybeNull = maybeNull();
    if (maybeNull != null) maybeNull.shutdown();
  }
}
```

Lazy的get方法中，使用了典型的懒汉式单例模式，并使用了double-check，方式多线程构造多个实例，而真正构造对象是委派给compute方法

```java
public abstract class Lazy<T> {

  volatile T instance = null;

  /** Remembers the result, if the operation completed unexceptionally. */
  protected abstract T compute();

  /** Returns the same value, computing as necessary */
  public final T get() {
    T result = instance;
    if (result == null) {
      synchronized (this) {
        result = instance;
        if (result == null) {
          instance = result = tryCompute();
        }
      }
    }
    return result;
  }

  /**
   * This is called in a synchronized block when the value to memorize hasn't yet been computed.
   *
   * <p>Extracted only for LazyCloseable, hence package protection.
   */
  T tryCompute() {
    return compute();
  }
}
```

在LazyConnector的compute方法中根据ConsumerConfig构造出了ZookeeperConsumerConnector，这个是kafka 0.8版本一种重要的对象，基于zookeeper的ConsumerConnector。

## LazyStreams

在LazyStreams的compute中，新建了一个线程池，线程池大小可以由参数streams（即zipkin.collector.kafka.streams）来指定，默认为一个线程的线程池。
然后通过topicCountMap设置zipkin的kafka消费使用的线程数，再使用ZookeeperConsumerConnector的createMessageStreams方法来创建KafkaStream，然后使用线程池执行KafkaStreamProcessor。

```java
static final class LazyStreams extends LazyCloseable<ExecutorService> {
  final int streams;
  final String topic;
  final Collector collector;
  final CollectorMetrics metrics;
  final LazyCloseable<ZookeeperConsumerConnector> connector;
  final AtomicReference<CheckResult> failure = new AtomicReference<>();

  LazyStreams(Builder builder, LazyCloseable<ZookeeperConsumerConnector> connector) {
    this.streams = builder.streams;
    this.topic = builder.topic;
    this.collector = builder.delegate.build();
    this.metrics = builder.metrics;
    this.connector = connector;
  }

  @Override protected ExecutorService compute() {
    ExecutorService pool = streams == 1
        ? Executors.newSingleThreadExecutor()
        : Executors.newFixedThreadPool(streams);

    Map<String, Integer> topicCountMap = new LinkedHashMap<>(1);
    topicCountMap.put(topic, streams);

    for (KafkaStream<byte[], byte[]> stream : connector.get().createMessageStreams(topicCountMap)
        .get(topic)) {
      pool.execute(guardFailures(new KafkaStreamProcessor(stream, collector, metrics)));
    }
    return pool;
  }

  Runnable guardFailures(final Runnable delegate) {
    return () -> {
      try {
        delegate.run();
      } catch (RuntimeException e) {
        failure.set(CheckResult.failed(e));
      }
    };
  }

  @Override
  public void close() {
    ExecutorService maybeNull = maybeNull();
    if (maybeNull != null) maybeNull.shutdown();
  }
}
```

## KafkaStreamProcessor

在KafkaStreamProcessor的run方法中，迭代stream对象，取出获得的流数据，然后调用Collector的acceptSpans方法，即使用storage组件来接收并存储span数据。

```java
final class KafkaStreamProcessor implements Runnable {
  final KafkaStream<byte[], byte[]> stream;
  final Collector collector;
  final CollectorMetrics metrics;

  KafkaStreamProcessor(
      KafkaStream<byte[], byte[]> stream, Collector collector, CollectorMetrics metrics) {
    this.stream = stream;
    this.collector = collector;
    this.metrics = metrics;
  }

  @Override
  public void run() {
    ConsumerIterator<byte[], byte[]> messages = stream.iterator();
    while (messages.hasNext()) {
      byte[] bytes = messages.next().message();
      metrics.incrementMessages();

      if (bytes.length == 0) {
        metrics.incrementMessagesDropped();
        continue;
      }

      // If we received legacy single-span encoding, decode it into a singleton list
      if (bytes[0] <= 16 && bytes[0] != 12 /* thrift, but not a list */) {
        try {
          metrics.incrementBytes(bytes.length);
          Span span = SpanDecoder.THRIFT_DECODER.readSpan(bytes);
          collector.accept(Collections.singletonList(span), NOOP);
        } catch (RuntimeException e) {
          metrics.incrementMessagesDropped();
        }
      } else {
        collector.acceptSpans(bytes, DETECTING_DECODER, NOOP);
      }
    }
  }
}
```

这里的kafka消费方式还是kafka0.8版本的，如果你想用kafka0.10+的版本，可以更改zipkin-server的pom，将collector-kafka10加入到依赖中，其原理跟kafka0.8的差不多，此处不再展开分析了。

```xml
<!-- Kafka10 Collector -->
<dependency>
  <groupId>io.zipkin.java</groupId>
  <artifactId>zipkin-autoconfigure-collector-kafka10</artifactId>
  <optional>true</optional>
</dependency>

<dependency>
  <groupId>io.zipkin.java</groupId>
  <artifactId>zipkin-collector-kafka10</artifactId>
</dependency>
```

在生产环境中，我们可以将zipkin的日志收集器改为kafka来提高系统的吞吐量，而且也可以让客户端和zipkin服务端解耦，客户端将不依赖zipkin服务端，只依赖kafka集群。

当然我们也可以将zipkin的collector替换为RabbitMQ来提高日志收集的效率，zipkin对scribe也作了支持，这里就不展开篇幅细说了。

# 666. 彩蛋

如果你对 Zipkin 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)