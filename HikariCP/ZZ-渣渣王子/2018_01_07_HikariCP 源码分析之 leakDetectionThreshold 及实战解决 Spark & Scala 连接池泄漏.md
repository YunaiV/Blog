title: 【追光者系列】HikariCP 源码分析之 leakDetectionThreshold 及实战解决 Spark/Scala 连接池泄漏
date: 2018-01-07
tags:
categories: HikariCP
permalink: HikariCP/zhazhawangzi/leakDetectionThreshold
author: 渣渣王子
from_url: https://mp.weixin.qq.com/s/_ghOnuwbLHOkqGKgzWdLVw
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484878&idx=2&sn=e56bb959879ec76ed26202c86f4d568b&chksm=fa497a7fcd3ef369174c22a9d36bfef9ca6ebfc09468bd55f974f0edf884045de6e84780e25b#rd
memo: todo 图片有点问题

-------

摘要: 原创出处 https://mp.weixin.qq.com/s/_ghOnuwbLHOkqGKgzWdLVw 「渣渣王子」欢迎转载，保留摘要，谢谢！

- [概念](http://www.iocoder.cn/HikariCP/zhazhawangzi/leakDetectionThreshold/)
- [源码解析](http://www.iocoder.cn/HikariCP/zhazhawangzi/leakDetectionThreshold/)
  - [Write](http://www.iocoder.cn/HikariCP/zhazhawangzi/leakDetectionThreshold/)
    - [HikariConfig](http://www.iocoder.cn/HikariCP/zhazhawangzi/leakDetectionThreshold/)
    - [HouseKeeper](http://www.iocoder.cn/HikariCP/zhazhawangzi/leakDetectionThreshold/)
    - [小结](http://www.iocoder.cn/HikariCP/zhazhawangzi/leakDetectionThreshold/)
  - [Read](http://www.iocoder.cn/HikariCP/zhazhawangzi/leakDetectionThreshold/)
    - [getConnection](http://www.iocoder.cn/HikariCP/zhazhawangzi/leakDetectionThreshold/)
    - [leakTaskFactory、ProxyLeakTaskFactory、ProxyLeakTask](http://www.iocoder.cn/HikariCP/zhazhawangzi/leakDetectionThreshold/)
    - [close](http://www.iocoder.cn/HikariCP/zhazhawangzi/leakDetectionThreshold/)
- [测试模拟](http://www.iocoder.cn/HikariCP/zhazhawangzi/leakDetectionThreshold/)
- [Spark/Scala连接池泄漏问题排查](http://www.iocoder.cn/HikariCP/zhazhawangzi/leakDetectionThreshold/)
- [参考资料](http://www.iocoder.cn/HikariCP/zhazhawangzi/leakDetectionThreshold/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# 概念

此属性控制在记录消息之前连接可能离开池的时间量，单位毫秒，默认为0，表明可能存在连接泄漏。
如果大于0且不是单元测试，则进一步判断：(leakDetectionThreshold < SECONDS.toMillis(2) or (leakDetectionThreshold > maxLifetime && maxLifetime > 0)，会被重置为0。即如果要生效则必须>0，而且不能小于2秒，而且当maxLifetime > 0时不能大于maxLifetime（默认值1800000毫秒=30分钟）。

> **leakDetectionThreshold**
> This property controls the amount of time that a connection can be out of the pool before a message is logged indicating a possible connection leak. A value of 0 means leak detection is disabled. Lowest acceptable value for enabling leak detection is 2000 (2 seconds). Default: 0

更多配置大纲详见文章 [【追光者系列】HikariCP默认配置](http://mp.weixin.qq.com/s?__biz=MzUzNTY4NTYxMA==&mid=2247483722&idx=1&sn=1a48000871c2ef79b748969b588b5fc3&chksm=fa80f1cfcdf778d9412df737791274b531cf89a9d4e449bcf872d629ee9a1671799ec83ada82&scene=21#wechat_redirect)

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnUHWZAZYwHyfEFsXkPlBaeooQ8LGAyAaibqveyYp9dnQKZ9H5ISm8Z3JrLyXUUic8Ya6rqNibgR1MjLg/640)

# 源码解析

我们首先来看一下leakDetectionThreshold用在了哪里的纲要图：

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

## Write

还记得上一篇文章[【追光者系列】HikariCP源码分析之从validationTimeout来讲讲Hikari 2.7.5版本的那些故事](http://mp.weixin.qq.com/s?__biz=MzUzNTY4NTYxMA==&mid=2247483754&idx=1&sn=e8929409d902d972a63372db9f3c7bb6&chksm=fa80f1efcdf778f9b4fa9ae746e347c4e918f31bf509a276b94106e2672060bcce8061cfdfb5&scene=21#wechat_redirect)提到：我们可以看到在两处看到validationTimeout的写入，一处是PoolBase构造函数，另一处是HouseKeeper线程。
leakDetectionThreshold的用法可以说是异曲同工，除了构造函数之外，也用了HouseKeeper线程去处理。

### HikariConfig

在com.zaxxer.hikari.HikariConfig中进行了leakDetectionThreshold初始化工作，

```Java
@Override
   public void setLeakDetectionThreshold(long leakDetectionThresholdMs)
   {
      this.leakDetectionThreshold = leakDetectionThresholdMs;
   }
```

validateNumerics方法中则是解释了上文及官方文档中该值validate的策略

```Java
if (leakDetectionThreshold > 0 && !unitTest) {
         if (leakDetectionThreshold < SECONDS.toMillis(2) || (leakDetectionThreshold > maxLifetime && maxLifetime > 0)) {
            LOGGER.warn("{} - leakDetectionThreshold is less than 2000ms or more than maxLifetime, disabling it.", poolName);
            leakDetectionThreshold = 0;
         }
      }
```

该方法会被HikariConfig#validate所调用，而HikariConfig#validate会在HikariDataSource的specified configuration的构造函数使用到

```Java
   /**
    * Construct a HikariDataSource with the specified configuration.  The
    * {@link HikariConfig} is copied and the pool is started by invoking this
    * constructor.
    *
    * The {@link HikariConfig} can be modified without affecting the HikariDataSource
    * and used to initialize another HikariDataSource instance.
    *
    * @param configuration a HikariConfig instance
    */
   public HikariDataSource(HikariConfig configuration)
   {
      configuration.validate();
      configuration.copyStateTo(this);
      LOGGER.info("{} - Starting...", configuration.getPoolName());
      pool = fastPathPool = new HikariPool(this);
      LOGGER.info("{} - Start completed.", configuration.getPoolName());
      this.seal();
   }
```

也在每次getConnection的时候用到了，

```Java
 // ***********************************************************************
   //                          DataSource methods
   // ***********************************************************************
   /** {@inheritDoc} */
   @Override
   public Connection getConnection() throws SQLException
   {
      if (isClosed()) {
         throw new SQLException("HikariDataSource " + this + " has been closed.");
      }
      if (fastPathPool != null) {
         return fastPathPool.getConnection();
      }
      // See http://en.wikipedia.org/wiki/Double-checked_locking#Usage_in_Java
      HikariPool result = pool;
      if (result == null) {
         synchronized (this) {
            result = pool;
            if (result == null) {
               validate();
               LOGGER.info("{} - Starting...", getPoolName());
               try {
                  pool = result = new HikariPool(this);
                  this.seal();
               }
               catch (PoolInitializationException pie) {
                  if (pie.getCause() instanceof SQLException) {
                     throw (SQLException) pie.getCause();
                  }
                  else {
                     throw pie;
                  }
               }
               LOGGER.info("{} - Start completed.", getPoolName());
            }
         }
      }
      return result.getConnection();
   }
```

这里要特别提一下一个很牛逼的Double-checked_locking的实现，大家可以看一下这篇文章 https://en.wikipedia.org/wiki/Double-checked_locking#Usage_in_Java

```Java
// Works with acquire/release semantics for volatile in Java 1.5 and later
// Broken under Java 1.4 and earlier semantics for volatile
class Foo {
    private volatile Helper helper;
    public Helper getHelper() {
        Helper localRef = helper;
        if (localRef == null) {
            synchronized(this) {
                localRef = helper;
                if (localRef == null) {
                    helper = localRef = new Helper();
                }
            }
        }
        return localRef;
    }
    // other functions and members...
}
```

### HouseKeeper

我们再来看一下com.zaxxer.hikari.pool.HikariPool这个代码,该线程尝试在池中维护的最小空闲连接数，并不断刷新的通过MBean调整的connectionTimeout和validationTimeout等值，leakDetectionThreshold这个值也是通过这个HouseKeeper的leakTask.updateLeakDetectionThreshold(config.getLeakDetectionThreshold())去管理的。

```Java
/**
    * The house keeping task to retire and maintain minimum idle connections.
    */
   private final class HouseKeeper implements Runnable
   {
      private volatile long previous = plusMillis(currentTime(), -HOUSEKEEPING_PERIOD_MS);
      @Override
      public void run()
      {
         try {
            // refresh timeouts in case they changed via MBean
            connectionTimeout = config.getConnectionTimeout();
            validationTimeout = config.getValidationTimeout();
            leakTask.updateLeakDetectionThreshold(config.getLeakDetectionThreshold());
            final long idleTimeout = config.getIdleTimeout();
            final long now = currentTime();
            // Detect retrograde time, allowing +128ms as per NTP spec.
            if (plusMillis(now, 128) < plusMillis(previous, HOUSEKEEPING_PERIOD_MS)) {
               LOGGER.warn("{} - Retrograde clock change detected (housekeeper delta={}), soft-evicting connections from pool.",
                           poolName, elapsedDisplayString(previous, now));
               previous = now;
               softEvictConnections();
               fillPool();
               return;
            }
            else if (now > plusMillis(previous, (3 * HOUSEKEEPING_PERIOD_MS) / 2)) {
               // No point evicting for forward clock motion, this merely accelerates connection retirement anyway
               LOGGER.warn("{} - Thread starvation or clock leap detected (housekeeper delta={}).", poolName, elapsedDisplayString(previous, now));
            }
            previous = now;
            String afterPrefix = "Pool ";
            if (idleTimeout > 0L && config.getMinimumIdle() < config.getMaximumPoolSize()) {
               logPoolState("Before cleanup ");
               afterPrefix = "After cleanup  ";
               final List<PoolEntry> notInUse = connectionBag.values(STATE_NOT_IN_USE);
               int removed = 0;
               for (PoolEntry entry : notInUse) {
                  if (elapsedMillis(entry.lastAccessed, now) > idleTimeout && connectionBag.reserve(entry)) {
                     closeConnection(entry, "(connection has passed idleTimeout)");
                     if (++removed > config.getMinimumIdle()) {
                        break;
                     }
                  }
               }
            }
            logPoolState(afterPrefix);
            fillPool(); // Try to maintain minimum connections
         }
         catch (Exception e) {
            LOGGER.error("Unexpected exception in housekeeping task", e);
         }
      }
   }
```

这里补充说一下这个HouseKeeper，它是在com.zaxxer.hikari.pool.HikariPool的构造函数中初始化的：this.houseKeepingExecutorService = initializeHouseKeepingExecutorService();

```Java
 /**
    * Create/initialize the Housekeeping service {@link ScheduledExecutorService}.  If the user specified an Executor
    * to be used in the {@link HikariConfig}, then we use that.  If no Executor was specified (typical), then create
    * an Executor and configure it.
    *
    * @return either the user specified {@link ScheduledExecutorService}, or the one we created
    */
   private ScheduledExecutorService initializeHouseKeepingExecutorService()
   {
      if (config.getScheduledExecutor() == null) {
         final ThreadFactory threadFactory = Optional.ofNullable(config.getThreadFactory()).orElse(new DefaultThreadFactory(poolName + " housekeeper", true));
         final ScheduledThreadPoolExecutor executor = new ScheduledThreadPoolExecutor(1, threadFactory, new ThreadPoolExecutor.DiscardPolicy());
         executor.setExecuteExistingDelayedTasksAfterShutdownPolicy(false);
         executor.setRemoveOnCancelPolicy(true);
         return executor;
      }
      else {
         return config.getScheduledExecutor();
      }
   }
```

这里简要说明一下，ScheduledThreadPoolExecutor是ThreadPoolExecutor类的子类，因为继承了ThreadPoolExecutor类所有的特性。但是，Java推荐仅在开发定时任务程序时采用ScheduledThreadPoolExecutor类。
在调用shutdown()方法而仍有待处理的任务需要执行时，可以配置ScheduledThreadPoolExecutor的行为。默认的行为是不论执行器是否结束，待处理的任务仍将被执行。但是，通过调用ScheduledThreadPoolExecutor类的setExecuteExistingDelayedTasksAfterShutdownPolicy()方法则可以改变这个行为。**传递false参数给这个方法，执行shutdown()方法之后，待处理的任务将不会被执行。**
取消任务后，判断是否需要从阻塞队列中移除任务。其中removeOnCancel参数通过setRemoveOnCancelPolicy()设置。之所以要在取消任务后移除阻塞队列中任务，**是为了防止队列中积压大量已被取消的任务**。
从这两个参数配置大家可以了解到作者的对于HouseKeeper的配置初衷。

### 小结

Hikari通过构造函数和HouseKeeper对于一些配置参数进行初始化及动态赋值，动态赋值依赖于HikariConfigMXbean以及使用任务调度线程池ScheduledThreadPoolExecutor来不断刷新配置的。

我们仅仅以com.zaxxer.hikari.HikariConfig来做下小结，允许在运行时进行动态修改的主要有：

```Java
   // Properties changeable at runtime through the HikariConfigMXBean
   //
   private volatile long connectionTimeout;
   private volatile long validationTimeout;
   private volatile long idleTimeout;
   private volatile long leakDetectionThreshold;
   private volatile long maxLifetime;
   private volatile int maxPoolSize;
   private volatile int minIdle;
   private volatile String username;
   private volatile String password;
```

不允许在运行时进行改变的主要有

```Java
  // Properties NOT changeable at runtime
   //
   private long initializationFailTimeout;
   private String catalog;
   private String connectionInitSql;
   private String connectionTestQuery;
   private String dataSourceClassName;
   private String dataSourceJndiName;
   private String driverClassName;
   private String jdbcUrl;
   private String poolName;
   private String schema;
   private String transactionIsolationName;
   private boolean isAutoCommit;
   private boolean isReadOnly;
   private boolean isIsolateInternalQueries;
   private boolean isRegisterMbeans;
   private boolean isAllowPoolSuspension;
   private DataSource dataSource;
   private Properties dataSourceProperties;
   private ThreadFactory threadFactory;
   private ScheduledExecutorService scheduledExecutor;
   private MetricsTrackerFactory metricsTrackerFactory;
   private Object metricRegistry;
   private Object healthCheckRegistry;
   private Properties healthCheckProperties;
```

## Read

### getConnection

在com.zaxxer.hikari.pool.HikariPool的核心方法getConnection返回的时候调用了poolEntry.createProxyConnection(leakTaskFactory.schedule(poolEntry), now)
注意，创建代理连接的时候关联了ProxyLeakTask。
连接泄漏检测的原理就是：**连接有借有还，hikari是每借用一个connection则会创建一个延时的定时任务，在归还或者出异常的或者用户手动调用evictConnection的时候cancel掉这个task**

```Java
   /**
    * Get a connection from the pool, or timeout after the specified number of milliseconds.
    *
    * @param hardTimeout the maximum time to wait for a connection from the pool
    * @return a java.sql.Connection instance
    * @throws SQLException thrown if a timeout occurs trying to obtain a connection
    */
   public Connection getConnection(final long hardTimeout) throws SQLException
   {
      suspendResumeLock.acquire();
      final long startTime = currentTime();
      try {
         long timeout = hardTimeout;
         do {
            PoolEntry poolEntry = connectionBag.borrow(timeout, MILLISECONDS);
            if (poolEntry == null) {
               break; // We timed out... break and throw exception
            }
            final long now = currentTime();
            if (poolEntry.isMarkedEvicted() || (elapsedMillis(poolEntry.lastAccessed, now) > ALIVE_BYPASS_WINDOW_MS && !isConnectionAlive(poolEntry.connection))) {
               closeConnection(poolEntry, poolEntry.isMarkedEvicted() ? EVICTED_CONNECTION_MESSAGE : DEAD_CONNECTION_MESSAGE);
               timeout = hardTimeout - elapsedMillis(startTime);
            }
            else {
               metricsTracker.recordBorrowStats(poolEntry, startTime);
               return poolEntry.createProxyConnection(leakTaskFactory.schedule(poolEntry), now);
            }
         } while (timeout > 0L);
         metricsTracker.recordBorrowTimeoutStats(startTime);
         throw createTimeoutException(startTime);
      }
      catch (InterruptedException e) {
         Thread.currentThread().interrupt();
         throw new SQLException(poolName + " - Interrupted during connection acquisition", e);
      }
      finally {
         suspendResumeLock.release();
      }
   }
```

### leakTaskFactory、ProxyLeakTaskFactory、ProxyLeakTask

在HikariPool构造函数里，初始化了leakTaskFactory，以及houseKeepingExecutorService。

```Java
this.houseKeepingExecutorService = initializeHouseKeepingExecutorService();
this.leakTaskFactory = new ProxyLeakTaskFactory(config.getLeakDetectionThreshold(), houseKeepingExecutorService);
this.houseKeeperTask = houseKeepingExecutorService.scheduleWithFixedDelay(new HouseKeeper(), 100L, HOUSEKEEPING_PERIOD_MS, MILLISECONDS);
```

com.zaxxer.hikari.pool.ProxyLeakTaskFactory是作者惯用的设计，我们看一下源码：

```Java
/**
 * A factory for {@link ProxyLeakTask} Runnables that are scheduled in the future to report leaks.
 *
 * @author Brett Wooldridge
 * @author Andreas Brenk
 */
class ProxyLeakTaskFactory
{
   private ScheduledExecutorService executorService;
   private long leakDetectionThreshold;
   ProxyLeakTaskFactory(final long leakDetectionThreshold, final ScheduledExecutorService executorService)
   {
      this.executorService = executorService;
      this.leakDetectionThreshold = leakDetectionThreshold;
   }
   ProxyLeakTask schedule(final PoolEntry poolEntry)
   {
      return (leakDetectionThreshold == 0) ? ProxyLeakTask.NO_LEAK : scheduleNewTask(poolEntry);
   }
   void updateLeakDetectionThreshold(final long leakDetectionThreshold)
   {
      this.leakDetectionThreshold = leakDetectionThreshold;
   }
   private ProxyLeakTask scheduleNewTask(PoolEntry poolEntry) {
      ProxyLeakTask task = new ProxyLeakTask(poolEntry);
      task.schedule(executorService, leakDetectionThreshold);
      return task;
   }
}
```

如果leakDetectionThreshold=0，即禁用连接泄露检测，schedule返回的是ProxyLeakTask.NO_LEAK，否则则新建一个ProxyLeakTask，在leakDetectionThreshold时间后触发

再看一下com.zaxxer.hikari.pool.ProxyLeakTask的源码

```Java
/**
 * A Runnable that is scheduled in the future to report leaks.  The ScheduledFuture is
 * cancelled if the connection is closed before the leak time expires.
 *
 * @author Brett Wooldridge
 */
class ProxyLeakTask implements Runnable
{
   private static final Logger LOGGER = LoggerFactory.getLogger(ProxyLeakTask.class);
   static final ProxyLeakTask NO_LEAK;
   private ScheduledFuture<?> scheduledFuture;
   private String connectionName;
   private Exception exception;
   private String threadName;
   private boolean isLeaked;
   static
   {
      NO_LEAK = new ProxyLeakTask() {
         @Override
         void schedule(ScheduledExecutorService executorService, long leakDetectionThreshold) {}
         @Override
         public void run() {}
         @Override
         public void cancel() {}
      };
   }
   ProxyLeakTask(final PoolEntry poolEntry)
   {
      this.exception = new Exception("Apparent connection leak detected");
      this.threadName = Thread.currentThread().getName();
      this.connectionName = poolEntry.connection.toString();
   }
   private ProxyLeakTask()
   {
   }
   void schedule(ScheduledExecutorService executorService, long leakDetectionThreshold)
   {
      scheduledFuture = executorService.schedule(this, leakDetectionThreshold, TimeUnit.MILLISECONDS);
   }
   /** {@inheritDoc} */
   @Override
   public void run()
   {
      isLeaked = true;
      final StackTraceElement[] stackTrace = exception.getStackTrace();
      final StackTraceElement[] trace = new StackTraceElement[stackTrace.length - 5];
      System.arraycopy(stackTrace, 5, trace, 0, trace.length);
      exception.setStackTrace(trace);
      LOGGER.warn("Connection leak detection triggered for {} on thread {}, stack trace follows", connectionName, threadName, exception);
   }
   void cancel()
   {
      scheduledFuture.cancel(false);
      if (isLeaked) {
         LOGGER.info("Previously reported leaked connection {} on thread {} was returned to the pool (unleaked)", connectionName, threadName);
      }
   }
}
```

NO_LEAK类里头的方法都是空操作
一旦该task被触发，则抛出Exception("Apparent connection leak detected")

我们想起了什么，是不是想起了[【追光者系列】HikariCP源码分析之allowPoolSuspension](http://mp.weixin.qq.com/s?__biz=MzUzNTY4NTYxMA==&mid=2247483735&idx=1&sn=d8ed8446ebc5e3c3df02afb2c6c3ed77&chksm=fa80f1d2cdf778c4da61d53d37aa7123d603fc1a4abe0804cc7c03c20f83d91436301deb3fa6&scene=21#wechat_redirect)那篇文章里有着一摸一样的设计？

```Java
this.suspendResumeLock = config.isAllowPoolSuspension() ? new SuspendResumeLock() : SuspendResumeLock.FAUX_LOCK;
```

isAllowPoolSuspension默认值是false的，构造函数直接会创建SuspendResumeLock.FAUX_LOCK；只有isAllowPoolSuspension为true时，才会真正创建SuspendResumeLock。

com.zaxxer.hikari.util.SuspendResumeLock内部实现了一虚一实两个java.util.concurrent.Semaphore

```Java
/**
 * This class implements a lock that can be used to suspend and resume the pool.  It
 * also provides a faux implementation that is used when the feature is disabled that
 * hopefully gets fully "optimized away" by the JIT.
 *
 * @author Brett Wooldridge
 */
public class SuspendResumeLock
{
   public static final SuspendResumeLock FAUX_LOCK = new SuspendResumeLock(false) {
      @Override
      public void acquire() {}
      @Override
      public void release() {}
      @Override
      public void suspend() {}
      @Override
      public void resume() {}
   };
   private static final int MAX_PERMITS = 10000;
   private final Semaphore acquisitionSemaphore;
   /**
    * Default constructor
    */
   public SuspendResumeLock()
   {
      this(true);
   }
   private SuspendResumeLock(final boolean createSemaphore)
   {
      acquisitionSemaphore = (createSemaphore ? new Semaphore(MAX_PERMITS, true) : null);
   }
   public void acquire()
   {
      acquisitionSemaphore.acquireUninterruptibly();
   }
   public void release()
   {
      acquisitionSemaphore.release();
   }
   public void suspend()
   {
      acquisitionSemaphore.acquireUninterruptibly(MAX_PERMITS);
   }
   public void resume()
   {
      acquisitionSemaphore.release(MAX_PERMITS);
   }
}
```

由于Hikari的isAllowPoolSuspension默认值是false的，FAUX_LOCK只是一个空方法，acquisitionSemaphore对象也是空的；如果isAllowPoolSuspension值调整为true，当收到MBean的suspend调用时将会一次性acquisitionSemaphore.acquireUninterruptibly从此信号量获取给定数目MAX_PERMITS 10000的许可，在提供这些许可前一直将线程阻塞。之后HikariPool的getConnection方法获取不到连接，阻塞在suspendResumeLock.acquire()，除非resume方法释放给定数目MAX_PERMITS 10000的许可，将其返回到信号量

### close

连接有借有还，连接检测的task也是会关闭的。
我们看一下com.zaxxer.hikari.pool.ProxyConnection源码，

```Java
 // **********************************************************************
   //              "Overridden" java.sql.Connection Methods
   // **********************************************************************
   /** {@inheritDoc} */
   @Override
   public final void close() throws SQLException
   {
      // Closing statements can cause connection eviction, so this must run before the conditional below
      closeStatements();
      if (delegate != ClosedConnection.CLOSED_CONNECTION) {
         leakTask.cancel();
         try {
            if (isCommitStateDirty && !isAutoCommit) {
               delegate.rollback();
               lastAccess = currentTime();
               LOGGER.debug("{} - Executed rollback on connection {} due to dirty commit state on close().", poolEntry.getPoolName(), delegate);
            }
            if (dirtyBits != 0) {
               poolEntry.resetConnectionState(this, dirtyBits);
               lastAccess = currentTime();
            }
            delegate.clearWarnings();
         }
         catch (SQLException e) {
            // when connections are aborted, exceptions are often thrown that should not reach the application
            if (!poolEntry.isMarkedEvicted()) {
               throw checkException(e);
            }
         }
         finally {
            delegate = ClosedConnection.CLOSED_CONNECTION;
            poolEntry.recycle(lastAccess);
         }
      }
   }
```

在connection的close的时候,delegate != ClosedConnection.CLOSED_CONNECTION时会调用leakTask.cancel();取消检测连接泄露的task。

在closeStatements中也会关闭：

```Java
   @SuppressWarnings("EmptyTryBlock")
   private synchronized void closeStatements()
   {
      final int size = openStatements.size();
      if (size > 0) {
         for (int i = 0; i < size && delegate != ClosedConnection.CLOSED_CONNECTION; i++) {
            try (Statement ignored = openStatements.get(i)) {
               // automatic resource cleanup
            }
            catch (SQLException e) {
               LOGGER.warn("{} - Connection {} marked as broken because of an exception closing open statements during Connection.close()",
                           poolEntry.getPoolName(), delegate);
               leakTask.cancel();
               poolEntry.evict("(exception closing Statements during Connection.close())");
               delegate = ClosedConnection.CLOSED_CONNECTION;
            }
         }
         openStatements.clear();
      }
   }
```

在checkException中也会关闭

```Java
   final SQLException checkException(SQLException sqle)
   {
      SQLException nse = sqle;
      for (int depth = 0; delegate != ClosedConnection.CLOSED_CONNECTION && nse != null && depth < 10; depth++) {
         final String sqlState = nse.getSQLState();
         if (sqlState != null && sqlState.startsWith("08") || ERROR_STATES.contains(sqlState) || ERROR_CODES.contains(nse.getErrorCode())) {
            // broken connection
            LOGGER.warn("{} - Connection {} marked as broken because of SQLSTATE({}), ErrorCode({})",
                        poolEntry.getPoolName(), delegate, sqlState, nse.getErrorCode(), nse);
            leakTask.cancel();
            poolEntry.evict("(connection is broken)");
            delegate = ClosedConnection.CLOSED_CONNECTION;
         }
         else {
            nse = nse.getNextException();
         }
      }
      return sqle;
   }
```

在com.zaxxer.hikari.pool.HikariPool的evictConnection中，也会关闭任务

```Java
   /**
    * Evict a Connection from the pool.
    *
    * @param connection the Connection to evict (actually a {@link ProxyConnection})
    */
   public void evictConnection(Connection connection)
   {
      ProxyConnection proxyConnection = (ProxyConnection) connection;
      proxyConnection.cancelLeakTask();
      try {
         softEvictConnection(proxyConnection.getPoolEntry(), "(connection evicted by user)", !connection.isClosed() /* owner */);
      }
      catch (SQLException e) {
         // unreachable in HikariCP, but we're still forced to catch it
      }
   }
```

小结关闭任务如下图所示：

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

# 测试模拟

我们可以根据本文对于leakDetectionThreshold的分析用测试包里的com.zaxxer.hikari.pool.MiscTest代码进行适当参数调整模拟连接泄漏情况，测试代码如下：

```Java
/**
 * @author Brett Wooldridge
 */
public class MiscTest
{
   @Test
   public void testLogWriter() throws SQLException
   {
      HikariConfig config = newHikariConfig();
      config.setMinimumIdle(0);
      config.setMaximumPoolSize(4);
      config.setDataSourceClassName("com.zaxxer.hikari.mocks.StubDataSource");
      setConfigUnitTest(true);
      try (HikariDataSource ds = new HikariDataSource(config)) {
         PrintWriter writer = new PrintWriter(System.out);
         ds.setLogWriter(writer);
         assertSame(writer, ds.getLogWriter());
         assertEquals("testLogWriter", config.getPoolName());
      }
      finally
      {
         setConfigUnitTest(false);
      }
   }
   @Test
   public void testInvalidIsolation()
   {
      try {
         getTransactionIsolation("INVALID");
         fail();
      }
      catch (Exception e) {
         assertTrue(e instanceof IllegalArgumentException);
      }
   }
   @Test
   public void testCreateInstance()
   {
      try {
         createInstance("invalid", null);
         fail();
      }
      catch (RuntimeException e) {
         assertTrue(e.getCause() instanceof ClassNotFoundException);
      }
   }
   @Test
   public void testLeakDetection() throws Exception
   {
      ByteArrayOutputStream baos = new ByteArrayOutputStream();
      try (PrintStream ps = new PrintStream(baos, true)) {
         setSlf4jTargetStream(Class.forName("com.zaxxer.hikari.pool.ProxyLeakTask"), ps);
         setConfigUnitTest(true);
         HikariConfig config = newHikariConfig();
         config.setMinimumIdle(0);
         config.setMaximumPoolSize(4);
         config.setThreadFactory(Executors.defaultThreadFactory());
         config.setMetricRegistry(null);
         config.setLeakDetectionThreshold(TimeUnit.SECONDS.toMillis(4));
         config.setDataSourceClassName("com.zaxxer.hikari.mocks.StubDataSource");
         try (HikariDataSource ds = new HikariDataSource(config)) {
            setSlf4jLogLevel(HikariPool.class, Level.DEBUG);
            getPool(ds).logPoolState();
            try (Connection connection = ds.getConnection()) {
               quietlySleep(SECONDS.toMillis(4));
               connection.close();
               quietlySleep(SECONDS.toMillis(1));
               ps.close();
               String s = new String(baos.toByteArray());
               assertNotNull("Exception string was null", s);
               assertTrue("Expected exception to contain 'Connection leak detection' but contains *" + s + "*", s.contains("Connection leak detection"));
            }
         }
         finally
         {
            setConfigUnitTest(false);
            setSlf4jLogLevel(HikariPool.class, Level.INFO);
         }
      }
   }
}
```

当代码执行到了quietlySleep(SECONDS.toMillis(4));时直接按照预期抛异常Apparent connection leak detected。

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

紧接着在close的过程中执行到了delegate != ClosedConnection.CLOSED_CONNECTION来进行leakTask.cancel()

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

完整的测试输出模拟过程如下所示：

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

# Spark/Scala连接池泄漏问题排查

金融中心大数据决策数据组的同学找到反馈了一个问题：

> 我们在同一个jvm 需要连接多个数据库时，发现总体上 从连接池borrow 的 connection 多于 归还的，一段时间后 连接池就会报出
> Caused by: java.sql.SQLTransientConnectionException: HikariPool-0 - Connection is not available, request timed out after 30000ms的异常。

用户使用的spark的场景有点特殊，单机上开的链接很小，但是有很多机器都会去连。用户在一个jvm中就只会并发1个链接。

```Java
maximumPoolSize: 5
minimumIdle: 2
```

程序也会出现block的情况，发现是执行mysql时出现的，
mysql show processlist；发现大多停留在query end的情况，程序 thread dump 进程 持有monitor的线程。

DBA介入之后发现存在slow sql。

当然，这个问题出了是写频繁导致的，一次写入的量有点大，每一个sql都巨大走的batch，写入的 records 数在每秒 30-50条，一个record 有70多个字段。一个解决方式是把 binlog 移到 ssd 盘；还有一个方式是innodb_flush_log_at_trx_commit把这个参数改成0了，估计可能会提高20%~30%。

修复了如上一些问题之后，又发现用户反馈的问题，加了leakDetectionThreshold，得出的结论是存在连接泄漏（从池中借用后连接没有关闭）。

针对这个问题，我们怀疑的连接池泄漏的点要么在hikari中，要么在spark/scala中。采用排除法使用了druid，依然存在这个问题；于是我们就去翻spark这块的代码，仔细分析之后定位到了问题：

因为scala map懒加载，一开始mapPartitions都落在一个stage中，我们调整代码toList之后result.iterator就分在独立的stage中，连接池泄漏问题就不再存在。

根本原因可以参见《Spark : How to use mapPartition and create/close connection per partition
》：
https://stackoverflow.com/questions/36545579/spark-how-to-use-mappartition-and-create-close-connection-per-partition/36545821#36545821

一开始以为这是一个连接池问题，或者是spark问题，但是实际上通过leakDetectionThreshold的定位，我们得知实际上这是一个scala问题 ：）

# 参考资料

https://segmentfault.com/a/1190000013092894

# 666. 彩蛋

如果你对 HikariCP 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)