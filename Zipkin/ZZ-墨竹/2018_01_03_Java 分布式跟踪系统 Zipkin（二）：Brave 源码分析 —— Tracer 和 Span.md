title: Java 分布式跟踪系统 Zipkin（二）：Brave 源码分析 —— Tracer 和 Span
date: 2018-01-03
tag: 
categories: Zipkin
permalink: Zipkin/mozhu/brave-tracer-and-span
author: v墨竹v
from_url: https://blog.csdn.net/apei830/article/details/78722180
wechat_url: 

-------

摘要: 原创出处 https://blog.csdn.net/apei830/article/details/78722180 「v墨竹v」欢迎转载，保留摘要，谢谢！

- [Span](http://www.iocoder.cn/Zipkin/mozhu/brave-tracer-and-span/)
- [BoundedAsyncReporter](http://www.iocoder.cn/Zipkin/mozhu/brave-tracer-and-span/)
- [ByteBoundedQueue](http://www.iocoder.cn/Zipkin/mozhu/brave-tracer-and-span/)
- [BufferNextMessage](http://www.iocoder.cn/Zipkin/mozhu/brave-tracer-and-span/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

Brave是Java版的Zipkin客户端，它将收集的跟踪信息，以Span的形式上报给Zipkin系统。

（Zipkin是基于Google的一篇论文，名为Dapper，Dapper在荷兰语里是“勇敢的”的意思，这也是Brave的命名的原因）

Brave目前版本为4.9.1，兼容zipkin1和2的协议，github地址：<https://github.com/openzipkin/brave>

我们一般不会手动编写Trace相关的代码，Brave提供了一些开箱即用的库，来帮助我们对某些特定的库类来进行追踪，比如servlet，springmvc，mysql，okhttp3，httpclient等，这些都可以在下面页面中找到：

<https://github.com/openzipkin/brave/tree/master/instrumentation>

我们先来看看一个简单的Demo来演示下Brave的基本使用，这对我们后续分析Brave的原理和其他类库的使用有很大帮助

TraceDemo

```java
package tracing;

import brave.Span;
import brave.Tracer;
import brave.Tracing;
import brave.context.log4j2.ThreadContextCurrentTraceContext;
import brave.propagation.B3Propagation;
import brave.propagation.ExtraFieldPropagation;
import zipkin2.codec.SpanBytesEncoder;
import zipkin2.reporter.AsyncReporter;
import zipkin2.reporter.Sender;
import zipkin2.reporter.okhttp3.OkHttpSender;

import java.util.concurrent.TimeUnit;

public class TraceDemo {

    public static void main(String[] args) {
        Sender sender = OkHttpSender.create("http://localhost:9411/api/v2/spans");
        AsyncReporter asyncReporter = AsyncReporter.builder(sender)
                .closeTimeout(500, TimeUnit.MILLISECONDS)
                .build(SpanBytesEncoder.JSON_V2);

        Tracing tracing = Tracing.newBuilder()
                .localServiceName("tracer-demo")
                .spanReporter(asyncReporter)
                .propagationFactory(ExtraFieldPropagation.newFactory(B3Propagation.FACTORY, "user-name"))
                .currentTraceContext(ThreadContextCurrentTraceContext.create())
                .build();

        Tracer tracer = tracing.tracer();
        Span span = tracer.newTrace().name("encode").start();
        try {
            doSomethingExpensive();
        } finally {
            span.finish();
        }


        Span twoPhase = tracer.newTrace().name("twoPhase").start();
        try {
            Span prepare = tracer.newChild(twoPhase.context()).name("prepare").start();
            try {
                prepare();
            } finally {
                prepare.finish();
            }
            Span commit = tracer.newChild(twoPhase.context()).name("commit").start();
            try {
                commit();
            } finally {
                commit.finish();
            }
        } finally {
            twoPhase.finish();
        }


        sleep(1000);

    }

    private static void doSomethingExpensive() {
        sleep(500);
    }

    private static void commit() {
        sleep(500);
    }

    private static void prepare() {
        sleep(500);
    }

    private static void sleep(long milliseconds) {
        try {
            TimeUnit.MILLISECONDS.sleep(milliseconds);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

启动Zipkin，然后运行TraceDemo，在Zipkin的UI界面中能查到两条跟踪信息
[![img](http://static.blog.mozhu.org/images/zipkin/2_1.png)](http://static.blog.mozhu.org/images/zipkin/2_1.png)

点击第一条跟踪信息，可以看到有一条Span(encode)，耗时500ms左右
[![encode跟踪信息](http://static.blog.mozhu.org/images/zipkin/2_2.png)](http://static.blog.mozhu.org/images/zipkin/2_2.png)encode跟踪信息

本条跟踪信息对应的代码片段为：

```java
Tracer tracer = tracing.tracer();
Span span = tracer.newTrace().name("encode").start();
try {
    doSomethingExpensive();
} finally {
    span.finish();
}
```

由Tracer创建一个新的Span，名为encode，然后调用start方法开始计时，之后运行一个比较耗时的方法doSomethingExpensive，最后调用finish方法结束计时，完成并记录一条跟踪信息。

这段代码实际上向Zipkin上报的数据为：

```json
[
  {
    "traceId": "16661f6cb5d58903",
    "id": "16661f6cb5d58903",
    "name": "encode",
    "timestamp": 1510043590522358,
    "duration": 499867,
    "binaryAnnotations": [
      {
        "key": "lc",
        "value": "",
        "endpoint": {
          "serviceName": "tracer-demo",
          "ipv4": "192.168.99.1"
        }
      }
    ]
  }
]
```

然后我们再来看第二条稍微复杂的跟踪信息，可以看到一条名为twoPhase的Span，总耗时为1000ms，它有2个子Span，分别名为prepare和commit，两者分别耗时500ms

[![twoPhase跟踪信息](http://static.blog.mozhu.org/images/zipkin/2_3.png)](http://static.blog.mozhu.org/images/zipkin/2_3.png)twoPhase跟踪信息

这条跟踪信息对应的代码片段为

```json
Span twoPhase = tracer.newTrace().name("twoPhase").start();
try {
    Span prepare = tracer.newChild(twoPhase.context()).name("prepare").start();
    try {
	prepare();
    } finally {
	prepare.finish();
    }
    Span commit = tracer.newChild(twoPhase.context()).name("commit").start();
    try {
	commit();
    } finally {
	commit.finish();
    }
} finally {
    twoPhase.finish();
}
```

这段代码实际上向Zipkin上报的数据为：

```json
[
  {
    "traceId": "89e051d5394b90b1",
    "id": "89e051d5394b90b1",
    "name": "twophase",
    "timestamp": 1510043591038983,
    "duration": 1000356,
    "binaryAnnotations": [
      {
        "key": "lc",
        "value": "",
        "endpoint": {
          "serviceName": "tracer-demo",
          "ipv4": "192.168.99.1"
        }
      }
    ]
  },
  {
    "traceId": "89e051d5394b90b1",
    "id": "60568c4903793b8d",
    "name": "prepare",
    "parentId": "89e051d5394b90b1",
    "timestamp": 1510043591039919,
    "duration": 499246,
    "binaryAnnotations": [
      {
        "key": "lc",
        "value": "",
        "endpoint": {
          "serviceName": "tracer-demo",
          "ipv4": "192.168.99.1"
        }
      }
    ]
  },
  {
    "traceId": "89e051d5394b90b1",
    "id": "ce14448169d01d2f",
    "name": "commit",
    "parentId": "89e051d5394b90b1",
    "timestamp": 1510043591539304,
    "duration": 499943,
    "binaryAnnotations": [
      {
        "key": "lc",
        "value": "",
        "endpoint": {
          "serviceName": "tracer-demo",
          "ipv4": "192.168.99.1"
        }
      }
    ]
  }
]
```

## Span

首先看下Span的实现类RealSpan

该类依赖几个核心类

**Recorder**，用于记录Span

**Reporter**，用于上报Span给Zipkin

**MutableSpan**，Span的包装类，提供各种API操作Span

**MutableSpanMap**，以TraceContext为Key，MutableSpan为Value的Map结构，用于内存中存放所有的Span

RealSpan两个核心方法start, finish

```java
public Span start(long timestamp) {
  recorder().start(context(), timestamp);
  return this;
}

public void finish(long timestamp) {
  recorder().finish(context(), timestamp);
}
```

分别调用Recorder的start和finish方法，获取跟TraceContext绑定的Span信息，记录开始时间和结束时间，并在结束时，调用reporter的report方法，上报给Zipkin

```java
public void start(TraceContext context, long timestamp) {
  if (noop.get()) return;
  spanMap.getOrCreate(context).start(timestamp);
}

public void finish(TraceContext context, long finishTimestamp) {
  MutableSpan span = spanMap.remove(context);
  if (span == null || noop.get()) return;
  synchronized (span) {
    span.finish(finishTimestamp);
    reporter.report(span.toSpan());
  }
}
```

## BoundedAsyncReporter

Reporter的实现类AsyncReporter，而AsyncReporter的实现类是BoundedAsyncReporter

```
static final class BoundedAsyncReporter<S> extends AsyncReporter<S> {
    static final Logger logger = Logger.getLogger(BoundedAsyncReporter.class.getName());
    final AtomicBoolean closed = new AtomicBoolean(false);
    final BytesEncoder<S> encoder;
    final ByteBoundedQueue pending;
    final Sender sender;
    final int messageMaxBytes;
    final long messageTimeoutNanos;
    final long closeTimeoutNanos;
    final CountDownLatch close;
    final ReporterMetrics metrics;

    BoundedAsyncReporter(Builder builder, BytesEncoder<S> encoder) {
      this.pending = new ByteBoundedQueue(builder.queuedMaxSpans, builder.queuedMaxBytes);
      this.sender = builder.sender;
      this.messageMaxBytes = builder.messageMaxBytes;
      this.messageTimeoutNanos = builder.messageTimeoutNanos;
      this.closeTimeoutNanos = builder.closeTimeoutNanos;
      this.close = new CountDownLatch(builder.messageTimeoutNanos > 0 ? 1 : 0);
      this.metrics = builder.metrics;
      this.encoder = encoder;
    }
}
```

BoundedAsyncReporter中的几个重要的类：

- **BytesEncoder** - Span的编码器，将Span编码成二进制，便于sender发送给Zipkin
- **ByteBoundedQueue** - 类似于BlockingQueue，是一个既有数量限制，又有字节数限制的阻塞队列
- **Sender** - 将编码后的二进制数据，发送给Zipkin
- **ReporterMetrics** - Span的report相关的统计信息
- **BufferNextMessage** - Consumer，Span信息的消费者，依靠Sender上报Span信息

```java
public <S> AsyncReporter<S> build(BytesEncoder<S> encoder) {
  if (encoder == null) throw new NullPointerException("encoder == null");

  if (encoder.encoding() != sender.encoding()) {
    throw new IllegalArgumentException(String.format(
        "Encoder doesn't match Sender: %s %s", encoder.encoding(), sender.encoding()));
  }

  final BoundedAsyncReporter<S> result = new BoundedAsyncReporter<>(this, encoder);

  if (messageTimeoutNanos > 0) { // Start a thread that flushes the queue in a loop.
    final BufferNextMessage consumer =
        new BufferNextMessage(sender, messageMaxBytes, messageTimeoutNanos);
    final Thread flushThread = new Thread(() -> {
      try {
        while (!result.closed.get()) {
          result.flush(consumer);
        }
      } finally {
        for (byte[] next : consumer.drain()) result.pending.offer(next);
        result.close.countDown();
      }
    }, "AsyncReporter(" + sender + ")");
    flushThread.setDaemon(true);
    flushThread.start();
  }
  return result;
}
```

当messageTimeoutNanos大于0时，启动一个守护线程flushThread，一直循环调用BoundedAsyncReporter的flush方法，将内存中的Span信息上报给Zipkin
而当messageTimeoutNanos等于0时，客户端需要手动调用flush方法来上报Span信息

再来看下BoundedAsyncReporter中的close方法

```java
@Override public void close() {
  if (!closed.compareAndSet(false, true)) return; // already closed
  try {
    // wait for in-flight spans to send
    if (!close.await(closeTimeoutNanos, TimeUnit.NANOSECONDS)) {
      logger.warning("Timed out waiting for in-flight spans to send");
    }
  } catch (InterruptedException e) {
    logger.warning("Interrupted waiting for in-flight spans to send");
    Thread.currentThread().interrupt();
  }
  int count = pending.clear();
  if (count > 0) {
    metrics.incrementSpansDropped(count);
    logger.warning("Dropped " + count + " spans due to AsyncReporter.close()");
  }
}
```

这个close方法和FlushThread中while循环相呼应，在close方法中，首先将closed变量置为true，然后调用close.await()，等待close信号量(CountDownLatch)的释放，此处代码会阻塞，一直到FlushThread中finally中调用result.close.countDown();
而在close方法中将closed变量置为true后，FlushThread中的while循环将结束执行，然后执行finally代码块，系统会将内存中还未上报的Span，添加到queue（result.pending）中，然后调用result.close.countDown(); close方法中阻塞的代码会继续执行，将调用metrics.incrementSpansDropped(count)将这些Span的数量添加到metrics统计信息中

```java
@Override public void report(S span) {
  if (span == null) throw new NullPointerException("span == null");

  metrics.incrementSpans(1);
  byte[] next = encoder.encode(span);
  int messageSizeOfNextSpan = sender.messageSizeInBytes(Collections.singletonList(next));
  metrics.incrementSpanBytes(next.length);
  if (closed.get() ||
      // don't enqueue something larger than we can drain
      messageSizeOfNextSpan > messageMaxBytes ||
      !pending.offer(next)) {
    metrics.incrementSpansDropped(1);
  }
}
```

前面看到在Recorder的finish方法中，会调用Reporter的report方法，此处report方法，将span转化成字节数组，然后计算出messageSize，添加到queue(pending)中，并记录相应的统计信息

接下来看看两个flush方法，其中flush()方法，是public的，供外部手动调用，而flush(BufferNextMessage bundler)是在FlushThread中循环调用

```java
@Override public final void flush() {
  flush(new BufferNextMessage(sender, messageMaxBytes, 0));
}

void flush(BufferNextMessage bundler) {
  if (closed.get()) throw new IllegalStateException("closed");

  //将队列中的数据，全部提取到BufferNextMessage中，直到buffer(bundler)满为止
  pending.drainTo(bundler, bundler.remainingNanos());

  // record after flushing reduces the amount of gauge events vs on doing this on report
  metrics.updateQueuedSpans(pending.count);
  metrics.updateQueuedBytes(pending.sizeInBytes);

  // loop around if we are running, and the bundle isn't full
  // if we are closed, try to send what's pending
  if (!bundler.isReady() && !closed.get()) return;

  // Signal that we are about to send a message of a known size in bytes
  metrics.incrementMessages();
  metrics.incrementMessageBytes(bundler.sizeInBytes());
  List<byte[]> nextMessage = bundler.drain();

  try {
    sender.sendSpans(nextMessage).execute();
  } catch (IOException | RuntimeException | Error t) {
    // In failure case, we increment messages and spans dropped.
    int count = nextMessage.size();
    Call.propagateIfFatal(t);
    metrics.incrementMessagesDropped(t);
    metrics.incrementSpansDropped(count);
    if (logger.isLoggable(FINE)) {
      logger.log(FINE,
          format("Dropped %s spans due to %s(%s)", count, t.getClass().getSimpleName(),
              t.getMessage() == null ? "" : t.getMessage()), t);
    }
    // Raise in case the sender was closed out-of-band.
    if (t instanceof IllegalStateException) throw (IllegalStateException) t;
  }
}
```

flush中大致分下面几步

1. 先将队列pending中的数据，全部提取到BufferNextMessage（bundler）中，直到bundler满为止
2. 当bundler准备好，即isReady()返回true，将bundler中的message全部取出来
3. 将取出来的所有message，调用Sender的sendSpans方法，发送到Zipkin

## ByteBoundedQueue

类似于BlockingQueue，是一个既有数量限制，又有字节数限制的阻塞队列，提供了offer，drainTo，clear三个方法，供调用者向queue里存放，提取和清空数据

```java
final class ByteBoundedQueue {

  final ReentrantLock lock = new ReentrantLock(false);
  final Condition available = lock.newCondition();

  final int maxSize;
  final int maxBytes;

  final byte[][] elements;
  int count;
  int sizeInBytes;
  int writePos;
  int readPos;

  ByteBoundedQueue(int maxSize, int maxBytes) {
    this.elements = new byte[maxSize][];
    this.maxSize = maxSize;
    this.maxBytes = maxBytes;
  }
}
```

ByteBoundedQueue接受两个int参数，maxSize是queue接受的最大数量，maxBytes是queue接受的最大字节数
ByteBoundedQueue中使用一个二维byte数组elements来存储message，并使用writePos和readPos两个游标，分别记录写和读的位置
ByteBoundedQueue中使用了最典型的可重入锁ReentrantLock，使offer，drainTo，clear等方法是线程安全的

```java
/**
 * Returns true if the element could be added or false if it could not due to its size.
 */
boolean offer(byte[] next) {
  lock.lock();
  try {
    if (count == elements.length) return false;
    if (sizeInBytes + next.length > maxBytes) return false;

    elements[writePos++] = next;

    if (writePos == elements.length) writePos = 0; // circle back to the front of the array

    count++;
    sizeInBytes += next.length;

    available.signal(); // alert any drainers
    return true;
  } finally {
    lock.unlock();
  }
}
```

offer方法是添加message到queue中，使用了标准的try-lock结构，即先获取锁，然后finally里释放锁，在获取锁以后
当count等于elements.length时，意味着queue是满的，则不能继续添加
当sizeInBytes + next.length > maxBytes时，意味着该消息加进队列会超出队列字节大小限制，也不能添加新message
如果上面两个条件都不满足，则表明可以继续添加message，将writePos+1，并将message放于writePos+1处
当writePos到达数组尾部，则将writePos置为0，让下一次添加从数组头部开始
然后将count计数器加1，并更新字节总数
最后调用available.signal()来通知其他在lock上等待的线程（在drainTo方法中阻塞的线程）继续竞争线程资源

```java
/** Blocks for up to nanosTimeout for elements to appear. Then, consume as many as possible. */
int drainTo(Consumer consumer, long nanosTimeout) {
  try {
    // This may be called by multiple threads. If one is holding a lock, another is waiting. We
    // use lockInterruptibly to ensure the one waiting can be interrupted.
    lock.lockInterruptibly();
    try {
      long nanosLeft = nanosTimeout;
      while (count == 0) {
        if (nanosLeft <= 0) return 0;
        nanosLeft = available.awaitNanos(nanosLeft);
      }
      return doDrain(consumer);
    } finally {
      lock.unlock();
    }
  } catch (InterruptedException e) {
    return 0;
  }
}
```

drainTo方法是提取message到Consumer中消费，如果当时queue里没有消息，则每次等待nanosTimeout，直到queue里存入消息为止
当while循环退出，表明queue中已经有新的message添加进来，可以消费，则调用doDrain方法。

```java
int doDrain(Consumer consumer) {
  int drainedCount = 0;
  int drainedSizeInBytes = 0;
  while (drainedCount < count) {
    byte[] next = elements[readPos];

    if (next == null) break;
    if (consumer.accept(next)) {
      drainedCount++;
      drainedSizeInBytes += next.length;

      elements[readPos] = null;
      if (++readPos == elements.length) readPos = 0; // circle back to the front of the array
    } else {
      break;
    }
  }
  count -= drainedCount;
  sizeInBytes -= drainedSizeInBytes;
  return drainedCount;
}
```

doDrain里依然是一个while循环，当drainedCount小于count，即提取的message数量总数小于queue里消息总数时，尝试调用consumer.accept方法
如果accept方法返回true，则将drainedCount加1，并且drainedSizeInBytes加上当前消息的字节数
如果accept方法返回false，则跳出循环，将queue的count减掉提取的总消息数drainedCount，sizeInBytes减去提取的总字节数drainedSizeInBytes

```java
int clear() {
  lock.lock();
  try {
    int result = count;
    count = sizeInBytes = readPos = writePos = 0;
    Arrays.fill(elements, null);
    return result;
  } finally {
    lock.unlock();
  }
}
```

clear方法，清空队列，这个方法比较简单，就是将所有东西清零，该方法在Reporter的close方法中会被使用

## BufferNextMessage

BufferNextMessage是ByteBoundedQueue.Consumer的默认实现

```
final class BufferNextMessage implements ByteBoundedQueue.Consumer {
  private final Sender sender;
  private final int maxBytes;
  private final long timeoutNanos;
  private final List<byte[]> buffer = new LinkedList<>();

  long deadlineNanoTime;
  int sizeInBytes;
  boolean bufferFull;

  BufferNextMessage(Sender sender, int maxBytes, long timeoutNanos) {
    this.sender = sender;
    this.maxBytes = maxBytes;
    this.timeoutNanos = timeoutNanos;
  }
}
```

BufferNextMessage中使用一个LinkedList来存储接收的messages

```java
@Override
public boolean accept(byte[] next) {
  buffer.add(next); // speculatively add to the buffer so we can size it
  int x = sender.messageSizeInBytes(buffer);
  int y = maxBytes;
  int includingNextVsMaxBytes = (x < y) ? -1 : ((x == y) ? 0 : 1);

  // If we can fit queued spans and the next into one message...
  if (includingNextVsMaxBytes <= 0) {
    sizeInBytes = x;

    if (includingNextVsMaxBytes == 0) {
      bufferFull = true;
    }
    return true;
  } else {
    buffer.remove(buffer.size() - 1);
    return false; // we couldn't fit the next message into this buffer
  }
}
```

accept方法，先将message放入buffer，然后调用sender的messageSizeInBytes方法统计下所有buffer消息的总字节数includingNextVsMaxBytes
当includingNextVsMaxBytes大于该buffer的最大字节数maxBytes，则将加入到buffer的message移除
当includingNextVsMaxBytes等于该buffer的最大字节数maxBytes，则将该buffer标记为已满状态，即bufferFull = true

```java
long remainingNanos() {
  if (buffer.isEmpty()) {
    deadlineNanoTime = System.nanoTime() + timeoutNanos;
  }
  return Math.max(deadlineNanoTime - System.nanoTime(), 0);
}

boolean isReady() {
  return bufferFull || remainingNanos() <= 0;
}
```

remainingNanos方法中，当buffer为空，则重置一个deadlineNanoTime，其值为当前系统时间加上timeoutNanos，当系统时间超过这个时间或者buffer满了的时候， isReady会返回true，即buffer为准备就绪状态

```java
List<byte[]> drain() {
  if (buffer.isEmpty()) return Collections.emptyList();
  ArrayList<byte[]> result = new ArrayList<>(buffer);
  buffer.clear();
  sizeInBytes = 0;
  bufferFull = false;
  deadlineNanoTime = 0;
  return result;
}
```

drain方法返回buffer里的所有数据，并将buffer清空

isReady方法和drain方法，在BoundedAsyncReporter的flush方法中会被使用

```java
void flush(BufferNextMessage bundler) {
	// ...
	if (!bundler.isReady() && !closed.get()) return;
	// ...
	List<byte[]> nextMessage = bundler.drain();
	// ...
	sender.sendSpans(nextMessage).execute();
}
```

因为flush是会一直不间断被调用，而这里先调用bundler.isReady()方法，当返回true后才取出所有堆积的消息，一起打包发送给zipkin提高效率

再回过头来看看BoundedAsyncReporter里手动flush方法

```java
@Override public final void flush() {
  flush(new BufferNextMessage(sender, messageMaxBytes, 0));
}
```

在我们分析完BufferNextMessage源代码后，我们很容易得出结论：这里构造BufferNextMessage传入的timeoutNanos为0，所以BufferNextMessage的isReady()方法会永远返回true。
这意味着每次我们手动调用flush方法，会立即将queue的数据用BufferNextMessage填满，并打包发送给Zipkin，至于queue里剩下的数据，需要等到下次FlushThread循环执行flush方法的时候被发送

至此，我们已经分析过Tracer和Span相关的源代码，这对我们后续看Brave和其他框架整合有很大帮助：
Span/RealSpan
Recorder
Reporter/AsyncReporter/BoundedAsyncReporter
BufferNextMessage
ByteBoundedQueue

在下一篇博文中，会继续分析Tracing的初始化过程，以及相关源代码

# 666. 彩蛋

如果你对 Zipkin 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)