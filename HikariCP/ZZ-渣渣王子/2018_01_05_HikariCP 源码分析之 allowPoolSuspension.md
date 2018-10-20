title: 【追光者系列】HikariCP 源码分析之 allowPoolSuspension
date: 2018-01-05
tags:
categories: HikariCP
permalink: HikariCP/zhazhawangzi/allowPoolSuspension
author: 渣渣王子
from_url: https://mp.weixin.qq.com/s/-WGg22lUQU41c_8lx6kyQA
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484805&idx=2&sn=08277c59afda370be885e9e41ed6ea98&chksm=fa497a34cd3ef322f63a09bcfbad0a09cdbc4ea29c3b2de39ccfbb4273b0f81ea95709b54f03#rd

-------

摘要: 原创出处 https://mp.weixin.qq.com/s/-WGg22lUQU41c_8lx6kyQA 「渣渣王子」欢迎转载，保留摘要，谢谢！

- [概念](http://www.iocoder.cn/HikariCP/zhazhawangzi/allowPoolSuspension/)
- [用途及实战思考](http://www.iocoder.cn/HikariCP/zhazhawangzi/allowPoolSuspension/)
- [源码解析](http://www.iocoder.cn/HikariCP/zhazhawangzi/allowPoolSuspension/)
  - [suspendPool](http://www.iocoder.cn/HikariCP/zhazhawangzi/allowPoolSuspension/)
  - [Construct for isAllowPoolSuspension](http://www.iocoder.cn/HikariCP/zhazhawangzi/allowPoolSuspension/)
  - [SuspendResumeLock](http://www.iocoder.cn/HikariCP/zhazhawangzi/allowPoolSuspension/)
  - [getConnection](http://www.iocoder.cn/HikariCP/zhazhawangzi/allowPoolSuspension/)
  - [resumePool](http://www.iocoder.cn/HikariCP/zhazhawangzi/allowPoolSuspension/)
  - [softEvictConnections](http://www.iocoder.cn/HikariCP/zhazhawangzi/allowPoolSuspension/)
  - [ConcurrentBag](http://www.iocoder.cn/HikariCP/zhazhawangzi/allowPoolSuspension/)
- [参考资料](http://www.iocoder.cn/HikariCP/zhazhawangzi/allowPoolSuspension/)

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

该属性控制池是否可以通过JMX暂停和恢复。这对于某些故障转移自动化方案很有用。当池被暂停时，调用 getConnection()将不会超时，并将一直保持到池恢复为止。 默认值：false。

> **allowPoolSuspension**
> This property controls whether the pool can be suspended and resumed through JMX. This is useful for certain failover automation scenarios. When the pool is suspended, calls to getConnection() will not timeout and will be held until the pool is resumed. Default: false

这里要特别说明一下，必须开启 **allowPoolSuspension: true** 且在 **registerMbeans: true**的情况下才能通过MBean Proxy调节softEvictConnections()和suspendPool()/resumePool() methods.

使用方式如下：

```Java
MBeanServer mBeanServer = ManagementFactory.getPlatformMBeanServer();
ObjectName poolName = new ObjectName("com.zaxxer.hikari:type=Pool (foo)");
HikariPoolMXBean poolProxy = JMX.newMXBeanProxy(mBeanServer, poolName, HikariPoolMXBean.class);
int idleConnections = poolProxy.getIdleConnections();
poolProxy.suspendPool();
poolProxy.softEvictConnections();
poolProxy.resumePool();
```

# 用途及实战思考

作者是这么说的：
https://github.com/brettwooldridge/HikariCP/issues/1060

All of the suspend use cases I have heard have centered around a pattern of:

- Suspend the pool.
- Alter the pool configuration, or alter DNS configuration (to point to a new master).
- Soft-evict existing connections.
- Resume the pool.

我做过试验，Suspend期间getConnection确实不会超时，SQL执行都会被保留下来，软驱除现有连接之后，一直保持到池恢复Resume时，这些SQL依然会继续执行，也就是说用户并不会丢数据。
但是在实际生产中，不影响业务很难，即使继续执行，业务也可能超时了。
故障注入是中间件开发应该要做的，这个点的功能在实现chaosmonkey以模拟数据库连接故障，但是监控过程中我发现hikaricp_pending_threads指标并没有提升、MBean的threadAwaitingConnections也没有改变，所以包括故障演练以后也可以不用搞得那么复杂，收拢在中间件内部做可能更好，前提是对于这个参数，中间件还需要自研以增加模拟抛异常或是一些监控指标进行加强。
另外，**长期阻塞该参数存在让微服务卡死的风险**。

# 源码解析

本文基于hikariCP 2.7.3的源码进行分析

## suspendPool

首先我们观察com.zaxxer.hikari.pool.HikariPool#suspendPool方法，

```Java
   @Override
   public synchronized void suspendPool()
   {
      if (suspendResumeLock == SuspendResumeLock.FAUX_LOCK) {
         throw new IllegalStateException(poolName + " - is not suspendable");
      }
      else if (poolState != POOL_SUSPENDED) {
         suspendResumeLock.suspend();
         poolState = POOL_SUSPENDED;
      }
   }
```

如果suspendResumeLock是FAUX_LOCK的话，就直接抛异常；否则，如果当前连接池状态并不是POOL_SUSPENDED（1）状态——还有POOL_NORMAL（0）及POOL_SHUTDOWN（2）状态，调用java.util.concurrent.Semaphore.SuspendResumeLock的suspend方法，从此信号量获取给定数目10000的许可，在提供这些许可前一直将线程阻塞。

```Java
private static final int MAX_PERMITS = 10000;
public void suspend()
   {
      acquisitionSemaphore.acquireUninterruptibly(MAX_PERMITS);
   }
```

## Construct for isAllowPoolSuspension

我前文提及的为什么**必须开启allowPoolSuspension: true且在 registerMbeans: true的情况下才能通过MBean Proxy调节softEvictConnections()和suspendPool()/resumePool() methods**，我之前的大纲文章[【追光者系列】HikariCP默认配置](http://mp.weixin.qq.com/s?__biz=MzUzNTY4NTYxMA==&mid=2247483722&idx=1&sn=1a48000871c2ef79b748969b588b5fc3&chksm=fa80f1cfcdf778d9412df737791274b531cf89a9d4e449bcf872d629ee9a1671799ec83ada82&scene=21#wechat_redirect)也有提及，现在我带大家从源码角度看一下：
我们看一下com.zaxxer.hikari.pool.HikariPool的构造函数

```Java
/**
    * Construct a HikariPool with the specified configuration.
    *
    * @param config a HikariConfig instance
    */
   public HikariPool(final HikariConfig config)
   {
      super(config);
      this.connectionBag = new ConcurrentBag<>(this);
      this.suspendResumeLock = config.isAllowPoolSuspension() ? new SuspendResumeLock() : SuspendResumeLock.FAUX_LOCK;
      this.houseKeepingExecutorService = initializeHouseKeepingExecutorService();
      checkFailFast();
      if (config.getMetricsTrackerFactory() != null) {
         setMetricsTrackerFactory(config.getMetricsTrackerFactory());
      }
      else {
         setMetricRegistry(config.getMetricRegistry());
      }
      setHealthCheckRegistry(config.getHealthCheckRegistry());
      registerMBeans(this);
      ThreadFactory threadFactory = config.getThreadFactory();
      LinkedBlockingQueue<Runnable> addConnectionQueue = new LinkedBlockingQueue<>(config.getMaximumPoolSize());
      this.addConnectionQueue = unmodifiableCollection(addConnectionQueue);
      this.addConnectionExecutor = createThreadPoolExecutor(addConnectionQueue, poolName + " connection adder", threadFactory, new ThreadPoolExecutor.DiscardPolicy());
      this.closeConnectionExecutor = createThreadPoolExecutor(config.getMaximumPoolSize(), poolName + " connection closer", threadFactory, new ThreadPoolExecutor.CallerRunsPolicy());
      this.leakTaskFactory = new ProxyLeakTaskFactory(config.getLeakDetectionThreshold(), houseKeepingExecutorService);
      this.houseKeeperTask = houseKeepingExecutorService.scheduleWithFixedDelay(new HouseKeeper(), 100L, HOUSEKEEPING_PERIOD_MS, MILLISECONDS);
   }
```

在这里我们可以看到

```Java
this.suspendResumeLock = config.isAllowPoolSuspension() ? new SuspendResumeLock() : SuspendResumeLock.FAUX_LOCK;
```

isAllowPoolSuspension默认值是false的，构造函数直接会创建SuspendResumeLock.FAUX_LOCK；只有isAllowPoolSuspension为true时，才会真正创建SuspendResumeLock。

## SuspendResumeLock

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

由于Hikari的isAllowPoolSuspension默认值是false的，FAUX_LOCK只是一个空方法，acquisitionSemaphore对象也是空的；如果isAllowPoolSuspension值调整为true，当收到MBean的suspend调用时将会一次性acquisitionSemaphore.acquireUninterruptibly从此信号量获取给定数目MAX_PERMITS 10000的许可，在提供这些许可前一直将线程阻塞。之后HikariPool的getConnection方法获取不到连接，阻塞在suspendResumeLock.acquire()，除非resume方法释放给定数目MAX_PERMITS 10000的许可，将其返回到信号量。

## getConnection

我们看一下com.zaxxer.hikari.pool.HikariPool的getConnection核心方法

```Java
 /**
    * Get a connection from the pool, or timeout after connectionTimeout milliseconds.
    *
    * @return a java.sql.Connection instance
    * @throws SQLException thrown if a timeout occurs trying to obtain a connection
    */
   public Connection getConnection() throws SQLException
   {
      return getConnection(connectionTimeout);
   }
```

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

我们可以看到在getConnection的方法最前面和finally最后的时候分别进行了suspendResumeLock.acquire()和suspendResumeLock.release的操作，hardTimeout就是connectionTimeout，默认值SECONDS.toMillis(30) = 30000（如果小于250毫秒，则被重置回30秒），代表the maximum time to wait for a connection from the pool（等待来自池的连接的最大毫秒数,补充一下，在**acquire之后**如果在没有可用连接的情况下超过此时间，则会抛出SQLException）。
suspendPool之后的每次getConnection方法，其实都会卡在上面代码第一行suspendResumeLock.acquire()中在SuspendResumeLock的具体实现

```Java
   public void acquire()
   {
      acquisitionSemaphore.acquireUninterruptibly();
   }
```

## resumePool

resumePool只针对当前是POOL_SUSPENDED状态的连接池置为POOL_NORMAL,然后fillPool,最终resume实际调用SuspendResumeLock的acquisitionSemaphore.release(MAX_PERMITS)方法释放给定数目MAX_PERMITS 10000的许可，将其返回到信号量。

```Java
  @Override
   public synchronized void resumePool()
   {
      if (poolState == POOL_SUSPENDED) {
         poolState = POOL_NORMAL;
         fillPool();
         suspendResumeLock.resume();
      }
   }
```

fillPool

从当前的空闲连接(在执行时被感知到的)填充到minimumIdle（HikariCP尝试在池中维护的最小空闲连接数，如果空闲连接低于此值并且池中的总连接数少于maximumPoolSize，HikariCP将尽最大努力快速高效地添加其他连接）。

```Java
   /**
    * Fill pool up from current idle connections (as they are perceived at the point of execution) to minimumIdle connections.
    */
   private synchronized void fillPool()
   {
      final int connectionsToAdd = Math.min(config.getMaximumPoolSize() - getTotalConnections(), config.getMinimumIdle() - getIdleConnections())
                                   - addConnectionQueue.size();
      for (int i = 0; i < connectionsToAdd; i++) {
         addConnectionExecutor.submit((i < connectionsToAdd - 1) ? POOL_ENTRY_CREATOR : POST_FILL_POOL_ENTRY_CREATOR);
      }
   }
```

com.zaxxer.hikari.util.SuspendResumeLock#resume

```Java
   public void resume()
   {
      acquisitionSemaphore.release(MAX_PERMITS);
   }
```

## softEvictConnections

Mbean的softEvictConnections方法真正执行的是com.zaxxer.hikari.pool.HikariPool中softEvictConnections方法，这是一种“软”驱逐池中连接的方法，如果调用方是owner身份，或者连接处于空闲状态，可以立即关闭连接。否则，我们将其“标记”为驱逐，以便下次有人试图从池中获取它时将其逐出。

```Java
public void softEvictConnections()
   {
      connectionBag.values().forEach(poolEntry -> softEvictConnection(poolEntry, "(connection evicted)", false /* not owner */));
   }
```

```Java
/**
    * "Soft" evict a Connection (/PoolEntry) from the pool.  If this method is being called by the user directly
    * through {@link com.zaxxer.hikari.HikariDataSource#evictConnection(Connection)} then {@code owner} is {@code true}.
    *
    * If the caller is the owner, or if the Connection is idle (i.e. can be "reserved" in the {@link ConcurrentBag}),
    * then we can close the connection immediately.  Otherwise, we leave it "marked" for eviction so that it is evicted
    * the next time someone tries to acquire it from the pool.
    *
    * @param poolEntry the PoolEntry (/Connection) to "soft" evict from the pool
    * @param reason the reason that the connection is being evicted
    * @param owner true if the caller is the owner of the connection, false otherwise
    * @return true if the connection was evicted (closed), false if it was merely marked for eviction
    */
   private boolean softEvictConnection(final PoolEntry poolEntry, final String reason, final boolean owner)
   {
      poolEntry.markEvicted();
      if (owner || connectionBag.reserve(poolEntry)) {
         closeConnection(poolEntry, reason);
         return true;
      }
      return false;
   }
```

执行此方法时我们的owner默认传false(not owner),调用com.zaxxer.hikari.util.ConcurrentBag的reserve对方进行保留

```Java
 /**
    * The method is used to make an item in the bag "unavailable" for
    * borrowing.  It is primarily used when wanting to operate on items
    * returned by the <code>values(int)</code> method.  Items that are
    * reserved can be removed from the bag via <code>remove(T)</code>
    * without the need to unreserve them.  Items that are not removed
    * from the bag can be make available for borrowing again by calling
    * the <code>unreserve(T)</code> method.
    *
    * @param bagEntry the item to reserve
    * @return true if the item was able to be reserved, false otherwise
    */
   public boolean reserve(final T bagEntry)
   {
      return bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_RESERVED);
   }
```

## ConcurrentBag

说到ConcurrentBag这个不得不提的类，我这里引用一下文章做一下简要介绍,本系列后面会专题系统分析：
http://www.cnblogs.com/taisenki/p/7699667.html
HikariCP连接池是基于自主实现的ConcurrentBag完成的数据连接的多线程共享交互，是HikariCP连接管理快速的其中一个关键点。
ConcurrentBag是一个专门的并发包裹，在连接池（多线程数据交互）的实现上具有比LinkedBlockingQueue和LinkedTransferQueue更优越的性能。
ConcurrentBag通过拆分 CopyOnWriteArrayList、ThreadLocal和SynchronousQueue
进行并发数据交互。

- CopyOnWriteArrayList：负责存放ConcurrentBag中全部用于出借的资源
- ThreadLocal：用于加速线程本地化资源访问
- SynchronousQueue：用于存在资源等待线程时的第一手资源交接

ConcurrentBag中全部的资源均只能通过add方法进行添加，只能通过remove方法进行移出。

```Java
public void add(final T bagEntry)
{
   if (closed) {
      LOGGER.info("ConcurrentBag has been closed, ignoring add()");
      throw new IllegalStateException("ConcurrentBag has been closed, ignoring add()");
   }
   sharedList.add(bagEntry); //新添加的资源优先放入CopyOnWriteArrayList
   // 当有等待资源的线程时，将资源交到某个等待线程后才返回（SynchronousQueue）
   while (waiters.get() > 0 && !handoffQueue.offer(bagEntry)) {
      yield();
   }
}
public boolean remove(final T bagEntry)
{
   // 如果资源正在使用且无法进行状态切换，则返回失败
   if (!bagEntry.compareAndSet(STATE_IN_USE, STATE_REMOVED) && !bagEntry.compareAndSet(STATE_RESERVED, STATE_REMOVED) && !closed) {
      LOGGER.warn("Attempt to remove an object from the bag that was not borrowed or reserved: {}", bagEntry);
      return false;
   }
   final boolean removed = sharedList.remove(bagEntry); // 从CopyOnWriteArrayList中移出
   if (!removed && !closed) {
      LOGGER.warn("Attempt to remove an object from the bag that does not exist: {}", bagEntry);
   }
   return removed;
}
```

ConcurrentBag中通过borrow方法进行数据资源借用，通过requite方法进行资源回收，注意其中borrow方法只提供对象引用，不移除对象，因此使用时通过borrow取出的对象必须通过requite方法进行放回，否则容易导致内存泄露！

```Java
public T borrow(long timeout, final TimeUnit timeUnit) throws InterruptedException
{
   // 优先查看有没有可用的本地化的资源
   final List<Object> list = threadList.get();
   for (int i = list.size() - 1; i >= 0; i--) {
      final Object entry = list.remove(i);
      @SuppressWarnings("unchecked")
      final T bagEntry = weakThreadLocals ? ((WeakReference<T>) entry).get() : (T) entry;
      if (bagEntry != null && bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
         return bagEntry;
      }
   }
   final int waiting = waiters.incrementAndGet();
   try {
      // 当无可用本地化资源时，遍历全部资源，查看是否存在可用资源
      // 因此被一个线程本地化的资源也可能被另一个线程“抢走”
      for (T bagEntry : sharedList) {
         if (bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
            if (waiting > 1) {
                // 因为可能“抢走”了其他线程的资源，因此提醒包裹进行资源添加
               listener.addBagItem(waiting - 1);
            }
            return bagEntry;
         }
      }
      listener.addBagItem(waiting);
      timeout = timeUnit.toNanos(timeout);
      do {
         final long start = currentTime();
         // 当现有全部资源全部在使用中，等待一个被释放的资源或者一个新资源
         final T bagEntry = handoffQueue.poll(timeout, NANOSECONDS);
         if (bagEntry == null || bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
            return bagEntry;
         }
         timeout -= elapsedNanos(start);
      } while (timeout > 10_000);
      return null;
   }
   finally {
      waiters.decrementAndGet();
   }
}
public void requite(final T bagEntry)
{
   // 将状态转为未在使用
   bagEntry.setState(STATE_NOT_IN_USE);
   // 判断是否存在等待线程，若存在，则直接转手资源
   for (int i = 0; waiters.get() > 0; i++) {
      if (bagEntry.getState() != STATE_NOT_IN_USE || handoffQueue.offer(bagEntry)) {
         return;
      }
      else if ((i & 0xff) == 0xff) {
         parkNanos(MICROSECONDS.toNanos(10));
      }
      else {
         yield();
      }
   }
   // 否则，进行资源本地化
   final List<Object> threadLocalList = threadList.get();
   threadLocalList.add(weakThreadLocals ? new WeakReference<>(bagEntry) : bagEntry);
}
```

上述代码中的 weakThreadLocals 是用来判断是否使用弱引用，通过下述方法初始化：

```Java
private boolean useWeakThreadLocals()
{
   try {
      // 人工指定是否使用弱引用，但是官方不推荐进行自主设置。
      if (System.getProperty("com.dareway.concurrent.useWeakReferences") != null) {
         return Boolean.getBoolean("com.dareway.concurrent.useWeakReferences");
      }
      // 默认通过判断初始化的ClassLoader是否是系统的ClassLoader来确定
      return getClass().getClassLoader() != ClassLoader.getSystemClassLoader();
   }
   catch (SecurityException se) {
      return true;
   }
}
```

# 参考资料

[【追光者系列】Hikari连接池配多大合适？](http://mp.weixin.qq.com/s?__biz=MzUzNTY4NTYxMA==&mid=2247483731&idx=1&sn=b81013c6af5e6e62d5ac8d8a26e2b848&chksm=fa80f1d6cdf778c03aec5b9db0e539b5ad0efae471324a8735828762998b6703a8bb8b226ac4&scene=21#wechat_redirect)

[【追光者系列】HikariCP连接池监控指标实战](http://mp.weixin.qq.com/s?__biz=MzUzNTY4NTYxMA==&mid=2247483699&idx=1&sn=5efd1f9d872688eba0029c71b3668662&chksm=fa80f1b6cdf778a0bbd12ce5e97507d697058d7bc070082a8ca828c9910c1379ead43c7b9a05&scene=21#wechat_redirect)

[【追光者系列】HikariCP默认配置](http://mp.weixin.qq.com/s?__biz=MzUzNTY4NTYxMA==&mid=2247483722&idx=1&sn=1a48000871c2ef79b748969b588b5fc3&chksm=fa80f1cfcdf778d9412df737791274b531cf89a9d4e449bcf872d629ee9a1671799ec83ada82&scene=21#wechat_redirect)

[【追光者系列】Can you Read Aloud The Word Hikari Properly?](http://mp.weixin.qq.com/s?__biz=MzUzNTY4NTYxMA==&mid=2247483715&idx=1&sn=0946cfe08aac4bcc6b27633a6b3e3b78&chksm=fa80f1c6cdf778d03e2de5056f1749aaf63a9673599d8ec9043f224649a64c6d6a66d1dc9efb&scene=21#wechat_redirect)

https://github.com/brettwooldridge/HikariCP
https://github.com/brettwooldridge/HikariCP/issues/1060
https://github.com/brettwooldridge/HikariCP/wiki/MBean-(JMX)-Monitoring-and-Management
https://segmentfault.com/a/1190000013062326
http://www.cnblogs.com/taisenki/p/7699667.html

# 666. 彩蛋

如果你对 HikariCP 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)