title: Java 分布式跟踪系统 Zipkin（三）：Brave 源码分析 —— Tracing
date: 2018-01-04
tag: 
categories: Zipkin
permalink: Zipkin/mozhu/brace-tracing
author: v墨竹v
from_url: https://blog.csdn.net/apei830/article/details/78722209
wechat_url: 

-------

摘要: 原创出处 https://blog.csdn.net/apei830/article/details/78722209 「v墨竹v」欢迎转载，保留摘要，谢谢！

- [Tracing.Builder](http://www.iocoder.cn/Zipkin/mozhu/brace-tracing/)
- [Sampler](http://www.iocoder.cn/Zipkin/mozhu/brace-tracing/)
- [TraceContext](http://www.iocoder.cn/Zipkin/mozhu/brace-tracing/)
- [TraceContextOrSamplingFlags](http://www.iocoder.cn/Zipkin/mozhu/brace-tracing/)
- [CurrentTraceContext](http://www.iocoder.cn/Zipkin/mozhu/brace-tracing/)
- [CurrentTraceContext.Scope](http://www.iocoder.cn/Zipkin/mozhu/brace-tracing/)
- [ThreadContextCurrentTraceContext](http://www.iocoder.cn/Zipkin/mozhu/brace-tracing/)
- [Propagation](http://www.iocoder.cn/Zipkin/mozhu/brace-tracing/)
- [ExtraFieldPropagation](http://www.iocoder.cn/Zipkin/mozhu/brace-tracing/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

上一篇博文中，我们了解了Brave框架的基本使用，并且分析了跟Tracer相关的部分源代码。这篇博文我们接着看看Tracing的初始化及相关类的源代码

```java
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
	// ...
    }
}
```

Brave中各个组件创建大量使用的builder设计模式，Tacing也不例外，先来看下Tracing.Builder

## Tracing.Builder

```java
public static final class Tracing.Builder {
    String localServiceName;
    Endpoint localEndpoint;
    Reporter<zipkin2.Span> reporter;
    Clock clock;
    Sampler sampler = Sampler.ALWAYS_SAMPLE;
    CurrentTraceContext currentTraceContext = CurrentTraceContext.Default.inheritable();
    boolean traceId128Bit = false;
    boolean supportsJoin = true;
    Propagation.Factory propagationFactory = Propagation.Factory.B3;

    public Tracing build() {
      if (clock == null) clock = Platform.get();
      if (localEndpoint == null) {
        localEndpoint = Platform.get().localEndpoint();
        if (localServiceName != null) {
          localEndpoint = localEndpoint.toBuilder().serviceName(localServiceName).build();
        }
      }
      if (reporter == null) reporter = Platform.get();
      return new Default(this);
    }

    Builder() {
    }
}
```

Tracing中依赖的几个重要类

- **Endpoint** - IP，端口和应用服务名等信息
- **Sampler** - 采样器，根据traceId来判断是否一条trace需要被采样，即上报到zipkin
- **TraceContext** - 包含TraceId，SpanId，是否采样等数据
- **CurrentTraceContext** - 是一个辅助类，可以用于获得当前线程的TraceContext
- **Propagation** - 是一个可以向数据携带的对象carrier上注入（inject）和提取（extract）数据的接口
- **Propagation.Factory** - Propagation的工厂类

前面TraceDemo例子中，我们初始化Tracing时设置了localServiceName，spanReporter，propagationFactory，currentTraceContext
其中spanReporter为AsyncReporter我们上一篇已经分析过其源代码了，在build方法中可以看到，其默认实现是Platform，默认会将Span信息用logger进行输出，而不是上报到zipkin中

```java
@Override public void report(zipkin2.Span span) {
  if (!logger.isLoggable(Level.INFO)) return;
  if (span == null) throw new NullPointerException("span == null");
  logger.info(span.toString());
}
```

## Sampler

采样器，根据traceId来判断是否一条trace需要被采样，即上报到zipkin

```java
public abstract class Sampler {

  public static final Sampler ALWAYS_SAMPLE = new Sampler() {
    @Override public boolean isSampled(long traceId) {
      return true;
    }

    @Override public String toString() {
      return "AlwaysSample";
    }
  };

  public static final Sampler NEVER_SAMPLE = new Sampler() {
    @Override public boolean isSampled(long traceId) {
      return false;
    }

    @Override public String toString() {
      return "NeverSample";
    }
  };

  /** Returns true if the trace ID should be measured. */
  public abstract boolean isSampled(long traceId);

  /**
   * Returns a sampler, given a rate expressed as a percentage.
   *
   * <p>The sampler returned is good for low volumes of traffic (<100K requests), as it is precise.
   * If you have high volumes of traffic, consider {@link BoundarySampler}.
   *
   * @param rate minimum sample rate is 0.01, or 1% of traces
   */
  public static Sampler create(float rate) {
    return CountingSampler.create(rate);
  }
}
```

Sampler.ALWAYS_SAMPLE 永远需要被采样
Sampler.NEVER_SAMPLE 永远不采样

Sampler还有一个实现类
CountingSampler可以指定采样率，如CountingSampler.create(0.5f)则对50%的请求数据进行采样，里面用到了一个算法，这里不展开分析了。

## TraceContext

包含TraceId，SpanId，是否采样等数据

在Tracer的newRootContext方法中有这样一段代码，通过newBuilder来构建TraceContext对象

```java
TraceContext newRootContext(SamplingFlags samplingFlags, List<Object> extra) {
  long nextId = Platform.get().randomLong();
  Boolean sampled = samplingFlags.sampled();
  if (sampled == null) sampled = sampler.isSampled(nextId);
  return TraceContext.newBuilder()
      .sampled(sampled)
      .traceIdHigh(traceId128Bit ? Platform.get().nextTraceIdHigh() : 0L).traceId(nextId)
      .spanId(nextId)
      .debug(samplingFlags.debug())
      .extra(extra).build();
}
```

TraceContext中有以下一些属性

- **traceIdHigh** - 唯一标识trace的16字节id，即128-bit
- **traceId** - 唯一标识trace的8字节id
- **parentId** - 父级Span的spanId
- **spanId** - 在某个trace中唯一标识span的8字节id
- **shared** - 如果为true，则表明需要从其他tracer上共享span信息
- **extra** - 在某个trace中相关的额外数据集

还有继承自SamplingFlags的两个属性

- **sampled** - 是否采样
- **debug** - 是否为调试，如果为true时，就算sampled为false，也表明该trace需要采样（即可以覆盖sampled的值）

TraceContext中还定义了两个接口Injector，Extractor

```java
public interface Injector<C> {
  void inject(TraceContext traceContext, C carrier);
}

public interface Extractor<C> {
  TraceContextOrSamplingFlags extract(C carrier);
}
```

- **Injector** - 用于将TraceContext中的各种数据注入到carrier中，这里的carrier一般在RPC中指的是类似于Http Headers的可以携带额外信息的对象
- **Extractor** - 用于在carrier中提取TraceContext相关信息或者采样标记信息TraceContextOrSamplingFlags

## TraceContextOrSamplingFlags

TraceContextOrSamplingFlags是三种数据的联合类型，即TraceContext，TraceIdContext，SamplingFlags，官方文档上说

- 当有traceId和spanId时，需用create(TraceContext)来创建
- 当只有spanId时，需用create(TraceIdContext)来创建
- 其他情况下，需用create(SamplingFlags)来创建
  TraceContextOrSamplingFlags里的代码比较简单，这里不展开分析了

## CurrentTraceContext

CurrentTraceContext是一个辅助类，可以用于获得当前线程的TraceContext，它的默认实现类是CurrentTraceContext.Default

```java
public static final class Default extends CurrentTraceContext {
    static final ThreadLocal<TraceContext> DEFAULT = new ThreadLocal<>();
    // Inheritable as Brave 3's ThreadLocalServerClientAndLocalSpanState was inheritable
    static final InheritableThreadLocal<TraceContext> INHERITABLE = new InheritableThreadLocal<>();

    final ThreadLocal<TraceContext> local;

    /** @deprecated prefer {@link #create()} as it isn't inheritable, so can't leak contexts. */
    @Deprecated
    public Default() {
      this(INHERITABLE);
    }

    /** Uses a non-inheritable static thread local */
    public static CurrentTraceContext create() {
      return new Default(DEFAULT);
    }

    /**
     * Uses an inheritable static thread local which allows arbitrary calls to {@link
     * Thread#start()} to automatically inherit this context. This feature is available as it is was
     * the default in Brave 3, because some users couldn't control threads in their applications.
     *
     * <p>This can be a problem in scenarios such as thread pool expansion, leading to data being
     * recorded in the wrong span, or spans with the wrong parent. If you are impacted by this,
     * switch to {@link #create()}.
     */
    public static CurrentTraceContext inheritable() {
      return new Default(INHERITABLE);
    }

    Default(ThreadLocal<TraceContext> local) {
      if (local == null) throw new NullPointerException("local == null");
      this.local = local;
    }

    @Override public TraceContext get() {
      return local.get();
    }
}
```

CurrentTraceContext.Default提供了两个静态方法，即create()和inheritable()
当使用create方法创建时，local对象为ThreadLocal类型
当使用inheritable方法创建时，local对象为InheritableThreadLocal类型
ThreadLocal可以理解为JVM为同一个线程开辟的一个共享内存空间，在同一个线程中不同方法调用，可以从该空间中取出放入的对象
而当使用InheritableThreadLocal获取线程绑定对象时，当前线程没有，则向当前线程的父线程的共享内存中获取

官方文档指出，inheritable方法在线程池的环境中需谨慎使用，可能会取出错误的TraceContext，这样会导致Span等信息会记录并关联到错误的traceId上

## CurrentTraceContext.Scope

```java
public abstract Scope newScope(@Nullable TraceContext currentSpan);

/** A span remains in the scope it was bound to until close is called. */
public interface Scope extends Closeable {
  /** No exceptions are thrown when unbinding a span scope. */
  @Override void close();
}
```

CurrentTraceContext中还定义了一个Scope接口，该接口继承自Closeable接口
自JDK7开始，凡是实现了Closeable接口的对象，只要在try语句中定义的，当finally执行的时候，JVM都会主动调用其close方法来回收资源，所以CurrentTraceContext中就提供了一个newScope方法，我们在代码里可以这样来用

```java
try (Scope scope = newScope(invocationContext)) {
  // do somthing
}
```

再来看看CurrentTraceContext.Default中是如何实现newScope的

```java
@Override public Scope newScope(@Nullable TraceContext currentSpan) {
      final TraceContext previous = local.get();
      local.set(currentSpan);
      class DefaultCurrentTraceContextScope implements Scope {
        @Override public void close() {
          local.set(previous);
        }
      }
      return new DefaultCurrentTraceContextScope();
    }
```

首先会将当前线程的TraceContext赋值给previous变量，然后设置新的TraceContext到当前线程，当Scope的close方法调用时，会还原previous的值到当前线程中

用两个嵌套的try代码块来演示下上面做法的意义

```java
TraceContext traceContext1;
TraceContext traceContext2;
try (Scope scope = newScope(traceContext1)) {
  // 1.此处CurrentTraceContext.get()能获得traceContext1
  try (Scope scope = newScope(traceContext2)) {
  // 2.此处CurrentTraceContext.get()能获得traceContext2
  }
  // 3.此处CurrentTraceContext.get()能获得traceContext1
}
```

1. 在进入内层try代码块前，通过CurrentTraceContext.get()获取到的traceContext1
2. 在进入内层try代码块后，通过CurrentTraceContext.get()获取到的traceContext2
3. 在运行完内层try代码块，通过CurrentTraceContext.get()获取到的traceContext1

这种处理方式确实比较灵活优雅，不过对使用的人来说，也有点过于隐晦，不知道JDK7新特性的同学刚开始看到这种用法可能会一脸茫然

当然这种用法必须得让使用的人将scope对象new在try语句中，每个人都能按照这种约定的规则来写，容易出错，所以CurrentTraceContext中提供了几个对Callable，Runnable的封装方法wrap方法

```java
/** Wraps the input so that it executes with the same context as now. */
public <C> Callable<C> wrap(Callable<C> task) {
  final TraceContext invocationContext = get();
  class CurrentTraceContextCallable implements Callable<C> {
    @Override public C call() throws Exception {
      try (Scope scope = newScope(invocationContext)) {
        return task.call();
      }
    }
  }
  return new CurrentTraceContextCallable();
}

/** Wraps the input so that it executes with the same context as now. */
public Runnable wrap(Runnable task) {
  final TraceContext invocationContext = get();
  class CurrentTraceContextRunnable implements Runnable {
    @Override public void run() {
      try (Scope scope = newScope(invocationContext)) {
        task.run();
      }
    }
  }
  return new CurrentTraceContextRunnable();
}
```

CurrentTraceContext还对Executor，及ExecuteService提供了包装方法

```java
/**
 * Decorates the input such that the {@link #get() current trace context} at the time a task is
 * scheduled is made current when the task is executed.
 */
public Executor executor(Executor delegate) {
  class CurrentTraceContextExecutor implements Executor {
    @Override public void execute(Runnable task) {
      delegate.execute(CurrentTraceContext.this.wrap(task));
    }
  }
  return new CurrentTraceContextExecutor();
}

/**
 * Decorates the input such that the {@link #get() current trace context} at the time a task is
 * scheduled is made current when the task is executed.
 */
public ExecutorService executorService(ExecutorService delegate) {
  class CurrentTraceContextExecutorService extends brave.internal.WrappingExecutorService {

    @Override protected ExecutorService delegate() {
      return delegate;
    }

    @Override protected <C> Callable<C> wrap(Callable<C> task) {
      return CurrentTraceContext.this.wrap(task);
    }

    @Override protected Runnable wrap(Runnable task) {
      return CurrentTraceContext.this.wrap(task);
    }
  }
  return new CurrentTraceContextExecutorService();
}
```

这几个方法都用的是装饰器设计模式，属于比较常用的设计模式，此处就不再展开分析了

## ThreadContextCurrentTraceContext

可以看到TraceDemo中，我们设置的CurrentTraceContext是ThreadContextCurrentTraceContext.create()

ThreadContextCurrentTraceContext是为log4j2封装的，是brave-context-log4j2包中的一个类，在ThreadContext中放置traceId和spanId两个属性，我们可以在log4j2的配置文件中配置日志打印的pattern，使用占位符%X{traceId}和%X{spanId}，让每行日志都能打印当前的traceId和spanId

zipkin-learning\Chapter1\servlet25\src\main\resources\log4j2.properties

```properties
appender.console.layout.pattern = %d{ABSOLUTE} [%X{traceId}/%X{spanId}] %-5p [%t] %C{2} - %m%n
```

pom.xml中需要添加日志相关的jar

```xml
<brave.version>4.9.1</brave.version>
<log4j.version>2.8.2</log4j.version>

<dependency>
  <groupId>io.zipkin.brave</groupId>
  <artifactId>brave-context-log4j2</artifactId>
  <version>${brave.version}</version>
</dependency>

<dependency>
  <groupId>org.apache.logging.log4j</groupId>
  <artifactId>log4j-core</artifactId>
  <version>${log4j.version}</version>
</dependency>
<dependency>
  <groupId>org.apache.logging.log4j</groupId>
  <artifactId>log4j-jul</artifactId>
  <version>${log4j.version}</version>
</dependency>
<dependency>
  <groupId>org.apache.logging.log4j</groupId>
  <artifactId>log4j-jcl</artifactId>
  <version>${log4j.version}</version>
</dependency>
<dependency>
  <groupId>org.apache.logging.log4j</groupId>
  <artifactId>log4j-slf4j-impl</artifactId>
  <version>${log4j.version}</version>
</dependency>
```

在Chapter1的例子中，如果你观察frontend和backend的控制台，会有如下输出0cabad9917e767ab为traceId，0cabad9917e767ab和e96a226ce75d30b4为spanId

```shell
10:11:05,731 [0cabad9917e767ab/0cabad9917e767ab] INFO  [qtp1441410416-17] servlet.FrontendServlet - frontend receive request
```

```shell
10:11:05,820 [0cabad9917e767ab/e96a226ce75d30b4] INFO  [qtp1441410416-15] servlet.BackendServlet - backend receive request
```

```java
public final class ThreadContextCurrentTraceContext extends CurrentTraceContext {
  public static ThreadContextCurrentTraceContext create() {
    return create(CurrentTraceContext.Default.inheritable());
  }

  public static ThreadContextCurrentTraceContext create(CurrentTraceContext delegate) {
    return new ThreadContextCurrentTraceContext(delegate);
  }

  final CurrentTraceContext delegate;

  ThreadContextCurrentTraceContext(CurrentTraceContext delegate) {
    if (delegate == null) throw new NullPointerException("delegate == null");
    this.delegate = delegate;
  }

  @Override public TraceContext get() {
    return delegate.get();
  }

  @Override public Scope newScope(@Nullable TraceContext currentSpan) {
    final String previousTraceId = ThreadContext.get("traceId");
    final String previousSpanId = ThreadContext.get("spanId");

    if (currentSpan != null) {
      ThreadContext.put("traceId", currentSpan.traceIdString());
      ThreadContext.put("spanId", HexCodec.toLowerHex(currentSpan.spanId()));
    } else {
      ThreadContext.remove("traceId");
      ThreadContext.remove("spanId");
    }

    Scope scope = delegate.newScope(currentSpan);
    class ThreadContextCurrentTraceContextScope implements Scope {
      @Override public void close() {
        scope.close();
        ThreadContext.put("traceId", previousTraceId);
        ThreadContext.put("spanId", previousSpanId);
      }
    }
    return new ThreadContextCurrentTraceContextScope();
  }
}
```

ThreadContextCurrentTraceContext继承了CurrentTraceContext，覆盖了其newScope方法，提取了currentSpan中的traceId和spanId放到log4j2的上下文对象ThreadContext中

在[https://github.com/openzipkin/brave/tree/master/context中还能找到对slf4j和log4j的支持](https://github.com/openzipkin/brave/tree/master/context%E4%B8%AD%E8%BF%98%E8%83%BD%E6%89%BE%E5%88%B0%E5%AF%B9slf4j%E5%92%8Clog4j%E7%9A%84%E6%94%AF%E6%8C%81)
brave-context-slf4j中的brave.context.slf4j.MDCCurrentTraceContext
brave-context-log4j12中的brave.context.log4j12.MDCCurrentTraceContext
代码都比较类似，这里不细说了

## Propagation

Propagation，英文翻译传播器，是一个可以向数据携带的对象carrier上注入（inject）和提取（extract）数据的接口。
对于Http协议来说，通常carrier就是指http request对象，它的http headers可以携带trace信息，一般来说http的客户端会在headers里注入（inject）trace信息，而服务端则会在headers提取（extract）trace信息
Propagation.Setter和Propagation.Getter可以在carrier中设置和获取值
另外还有injector和extractor方法分别返回TraceContext.Injector和TraceContext.Extractor

```java
interface Setter<C, K> {
  void put(C carrier, K key, String value);
}
interface Getter<C, K> {
  @Nullable String get(C carrier, K key);
}

<C> TraceContext.Injector<C> injector(Setter<C, K> setter);
<C> TraceContext.Extractor<C> extractor(Getter<C, K> getter);
```

Propagation中还有一个工厂类Propagation.Factory，有一个工厂方法create，通过KeyFactory来创建Propagation对象

```java
abstract class Factory {
    public static final Factory B3 = B3Propagation.FACTORY;

    public boolean supportsJoin() {
      return false;
    }

    public boolean requires128BitTraceId() {
      return false;
    }

    public abstract <K> Propagation<K> create(KeyFactory<K> keyFactory);
}

interface KeyFactory<K> {
    KeyFactory<String> STRING = name -> name;

    K create(String name);
}
```

Propagation的默认实现是B3Propagation
B3Propagation用下面这些http headers来传播trace信息

- **X-B3-TraceId** - 128位或者64位的traceId，被编码成32位和16位的小写16进制形式
- **X-B3-SpanId** - 64位的spanId，被编码成16位的小写16进制形式
- **X-B3-ParentSpanId** - 64位的父级spanId，被编码成16位的小写16进制形式
- **X-B3-Sampled** - 1代表采样，0代表不采样，如果没有这个key，则留给header接受端，即服务端自行判断
- **X-B3-Flags** - debug，如果为1代表采样

```java
@Override public <C> TraceContext.Injector<C> injector(Setter<C, K> setter) {
  if (setter == null) throw new NullPointerException("setter == null");
  return new B3Injector<>(this, setter);
}

static final class B3Injector<C, K> implements TraceContext.Injector<C> {
  final B3Propagation<K> propagation;
  final Setter<C, K> setter;

  B3Injector(B3Propagation<K> propagation, Setter<C, K> setter) {
    this.propagation = propagation;
    this.setter = setter;
  }

  @Override public void inject(TraceContext traceContext, C carrier) {
    setter.put(carrier, propagation.traceIdKey, traceContext.traceIdString());
    setter.put(carrier, propagation.spanIdKey, toLowerHex(traceContext.spanId()));
    if (traceContext.parentId() != null) {
      setter.put(carrier, propagation.parentSpanIdKey, toLowerHex(traceContext.parentId()));
    }
    if (traceContext.debug()) {
      setter.put(carrier, propagation.debugKey, "1");
    } else if (traceContext.sampled() != null) {
      setter.put(carrier, propagation.sampledKey, traceContext.sampled() ? "1" : "0");
    }
  }
}
```

inject方法中很简单，就是利用Setter将trace信息设置在carrier中

```java
@Override public <C> TraceContext.Extractor<C> extractor(Getter<C, K> getter) {
  if (getter == null) throw new NullPointerException("getter == null");
  return new B3Extractor(this, getter);
}

static final class B3Extractor<C, K> implements TraceContext.Extractor<C> {
  final B3Propagation<K> propagation;
  final Getter<C, K> getter;

  B3Extractor(B3Propagation<K> propagation, Getter<C, K> getter) {
    this.propagation = propagation;
    this.getter = getter;
  }

  @Override public TraceContextOrSamplingFlags extract(C carrier) {
    if (carrier == null) throw new NullPointerException("carrier == null");

    String traceId = getter.get(carrier, propagation.traceIdKey);
    String sampled = getter.get(carrier, propagation.sampledKey);
    String debug = getter.get(carrier, propagation.debugKey);
    if (traceId == null && sampled == null && debug == null) {
      return TraceContextOrSamplingFlags.EMPTY;
    }

    // Official sampled value is 1, though some old instrumentation send true
    Boolean sampledV = sampled != null
        ? sampled.equals("1") || sampled.equalsIgnoreCase("true")
        : null;
    boolean debugV = "1".equals(debug);

    String spanId = getter.get(carrier, propagation.spanIdKey);
    if (spanId == null) { // return early if there's no span ID
      return TraceContextOrSamplingFlags.create(
          debugV ? SamplingFlags.DEBUG : SamplingFlags.Builder.build(sampledV)
      );
    }

    TraceContext.Builder result = TraceContext.newBuilder().sampled(sampledV).debug(debugV);
    result.traceIdHigh(
        traceId.length() == 32 ? lowerHexToUnsignedLong(traceId, 0) : 0
    );
    result.traceId(lowerHexToUnsignedLong(traceId));
    result.spanId(lowerHexToUnsignedLong(spanId));
    String parentSpanIdString = getter.get(carrier, propagation.parentSpanIdKey);
    if (parentSpanIdString != null) result.parentId(lowerHexToUnsignedLong(parentSpanIdString));
    return TraceContextOrSamplingFlags.create(result.build());
  }
}
```

extract方法则利用Getter从carrier中获取trace信息

在TraceDemo中我们设置的propagationFactory是ExtraFieldPropagation.newFactory(B3Propagation.FACTORY, “user-name”)

## ExtraFieldPropagation

ExtraFieldPropagation可以用来传输额外的信息

运行Chapter1中的Frontend和Backend服务，在控制台输入

```shell
curl http://localhost:8081 --header "user-name: zhangsan"
```

可以看到控制台输出了user-name的值zhangsan

```shell
Wed Nov 15 11:42:02 GMT+08:00 2017 zhangsan
```

```java
static final class ExtraFieldInjector<C, K> implements Injector<C> {
  final Injector<C> delegate;
  final Propagation.Setter<C, K> setter;
  final Map<String, K> nameToKey;

  ExtraFieldInjector(Injector<C> delegate, Setter<C, K> setter, Map<String, K> nameToKey) {
    this.delegate = delegate;
    this.setter = setter;
    this.nameToKey = nameToKey;
  }

  @Override public void inject(TraceContext traceContext, C carrier) {
    for (Object extra : traceContext.extra()) {
      if (extra instanceof Extra) {
        ((Extra) extra).setAll(carrier, setter, nameToKey);
        break;
      }
    }
    delegate.inject(traceContext, carrier);
  }
}
```

ExtraFieldInjector的inject方法中，将traceContext的extra数据，set到carrier中，这里的Extra对象，其实就是key-value，有One和Many两种，Many时就相当于Map结构
在Extra中setAll方法中，先用extra的name去nameToKey里找，如果没有就不设置，如果找到就调用setter的put方法将值设置到carrier中。

```java
static final class One extends Extra {
  String name, value;

  @Override void put(String name, String value) {
    this.name = name;
    this.value = value;
  }

  @Override String get(String name) {
    return name.equals(this.name) ? value : null;
  }

  @Override <C, K> void setAll(C carrier, Setter<C, K> setter, Map<String, K> nameToKey) {
    K key = nameToKey.get(name);
    if (key == null) return;
    setter.put(carrier, key, value);
  }

  @Override public String toString() {
    return "ExtraFieldPropagation{" + name + "=" + value + "}";
  }
}

static final class Many extends Extra {
  final LinkedHashMap<String, String> fields = new LinkedHashMap<>();

  @Override void put(String name, String value) {
    fields.put(name, value);
  }

  @Override String get(String name) {
    return fields.get(name);
  }

  @Override <C, K> void setAll(C carrier, Setter<C, K> setter, Map<String, K> nameToKey) {
    for (Map.Entry<String, String> field : fields.entrySet()) {
      K key = nameToKey.get(field.getKey());
      if (key == null) continue;
      setter.put(carrier, nameToKey.get(field.getKey()), field.getValue());
    }
  }

  @Override public String toString() {
    return "ExtraFieldPropagation" + fields;
  }
}
```

ExtraFieldExtractor的extract方法中，循环names去carrier里找，然后构造Extra数据放入delegate执行extract方法后的结果中

```java
static final class ExtraFieldExtractor<C, K> implements Extractor<C> {
  final Extractor<C> delegate;
  final Propagation.Getter<C, K> getter;
  final Map<String, K> names;

  ExtraFieldExtractor(Extractor<C> delegate, Getter<C, K> getter, Map<String, K> names) {
    this.delegate = delegate;
    this.getter = getter;
    this.names = names;
  }

  @Override public TraceContextOrSamplingFlags extract(C carrier) {
    TraceContextOrSamplingFlags result = delegate.extract(carrier);

    Extra extra = null;
    for (Map.Entry<String, K> field : names.entrySet()) {
      String maybeValue = getter.get(carrier, field.getValue());
      if (maybeValue == null) continue;
      if (extra == null) {
        extra = new One();
      } else if (extra instanceof One) {
        One one = (One) extra;
        extra = new Many();
        extra.put(one.name, one.value);
      }
      extra.put(field.getKey(), maybeValue);
    }
    if (extra == null) return result;
    return result.toBuilder().addExtra(extra).build();
  }
}
```

至此，Tracing类相关的源代码已分析的差不多了，后续博文中，我们会继续分析Brave跟各大框架整合的源代码

# 666. 彩蛋

如果你对 Zipkin 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)