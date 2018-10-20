title: 【追光者系列】HikariCP 源码分析之从 validationTimeout 来讲讲 2.7.5 版本的那些故事
date: 2018-01-06
tags:
categories: HikariCP
permalink: HikariCP/zhazhawangzi/validationTimeout
author: 渣渣王子
from_url: https://mp.weixin.qq.com/s/zZCnM-IFwAwc6lQ_NvL-1A
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484852&idx=2&sn=5dd78e640d522cd678acbd656ed43199&chksm=fa497a05cd3ef313027bea6b0d464b2b32d2a7d95bafed2ed89d528379ba58ef6b54d02ee0d4#rd

-------

摘要: 原创出处 https://mp.weixin.qq.com/s/zZCnM-IFwAwc6lQ_NvL-1A 「渣渣王子」欢迎转载，保留摘要，谢谢！

- [概念](http://www.iocoder.cn/HikariCP/zhazhawangzi/validationTimeout/)
- [源码解析](http://www.iocoder.cn/HikariCP/zhazhawangzi/validationTimeout/)
  - [Write](http://www.iocoder.cn/HikariCP/zhazhawangzi/validationTimeout/)
    - [#PoolBase](http://www.iocoder.cn/HikariCP/zhazhawangzi/validationTimeout/)
    - [#HouseKeeper](http://www.iocoder.cn/HikariCP/zhazhawangzi/validationTimeout/)
  - [Read](http://www.iocoder.cn/HikariCP/zhazhawangzi/validationTimeout/)
    - [#getConnection](http://www.iocoder.cn/HikariCP/zhazhawangzi/validationTimeout/)
    - [#newConnection](http://www.iocoder.cn/HikariCP/zhazhawangzi/validationTimeout/)
- [Hikari 2.7.5的故事](http://www.iocoder.cn/HikariCP/zhazhawangzi/validationTimeout/)
  - [两个关键的Mbean](http://www.iocoder.cn/HikariCP/zhazhawangzi/validationTimeout/)
  - [2.7.5迎来了不可变设计](http://www.iocoder.cn/HikariCP/zhazhawangzi/validationTimeout/)
  - [且看大神论道](http://www.iocoder.cn/HikariCP/zhazhawangzi/validationTimeout/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnUhKeldQcsMJ8CYTzcdUN60oC4v5ib28NLzBhpUSqAOYaSicXtWtWiaOrpFEDzdDphlGgmsqO1Eniazkg/640)

今晚给大家讲一个故事，如上图所示，Hikari作者brettwooldridge先生非常无奈的在issue里回复了一句“阿门，兄弟”，到底发生了什么有趣的故事呢？这是一篇风格不同于以往的文章，就让我来带大家从源码validationTimeout分析角度一起揭开这个故事的面纱吧～



# 概念

此属性控制连接测试活动的最长时间。这个值必须小于connectionTimeout。最低可接受的验证超时时间为250 ms。 默认值：5000。

> **validationTimeout**
> This property controls the maximum amount of time that a connection will be tested for aliveness. This value must be less than the connectionTimeout. Lowest acceptable validation timeout is 250 ms. Default: 5000

更多配置大纲详见文章 《【追光者系列】HikariCP默认配置》

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnUHWZAZYwHyfEFsXkPlBaeooQ8LGAyAaibqveyYp9dnQKZ9H5ISm8Z3JrLyXUUic8Ya6rqNibgR1MjLg/640)

# 源码解析

我们首先来看一下validationTimeout用在了哪里的纲要图：

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnUhKeldQcsMJ8CYTzcdUN60Kpt8jfGswqdudx9ibKnmglWuAiaXXHqvdokISE2STaIrZwRaOyXI0hzg/640)

## Write

我们可以看到在两处看到validationTimeout的写入，一处是PoolBase构造函数，另一处是HouseKeeper线程。

### PoolBase

在com.zaxxer.hikari.pool.PoolBase中的构造函数声明了validationTimeout的初始值，而该值真正来自于com.zaxxer.hikari.HikariConfig的Default constructor，默认值为

```Java
private static final long VALIDATION_TIMEOUT = SECONDS.toMillis(5);
```

但是在HikariConfig的set方法中又做了处理

```Java
/** {@inheritDoc} */
   @Override
   public void setValidationTimeout(long validationTimeoutMs)
   {
      if (validationTimeoutMs < 250) {
         throw new IllegalArgumentException("validationTimeout cannot be less than 250ms");
      }
      this.validationTimeout = validationTimeoutMs;
   }
```

这就是概念一栏所说的**如果小于250毫秒，则会被重置回5秒**的原因。

### HouseKeeper

我们再来看一下com.zaxxer.hikari.pool.HikariPool这个代码,该线程尝试在池中维护的最小空闲连接数，并不断刷新的通过MBean调整的connectionTimeout和validationTimeout等值。
HikariCP有除了这个HouseKeeper线程之外，还有新建连接和关闭连接的线程。

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

## Read

### getConnection

在com.zaxxer.hikari.pool.HikariPool的核心方法getConnection中用到了validationTimeout，我们看一下源码，**borrow**到poolEntry之后，如果不是isMarkedEvicted，则会调用isConnectionAlive来判断连接的有效性，再强调一下**hikari是在borrow连接的时候校验连接的有效性**：

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
         PoolEntry poolEntry = null;
         try {
            do {
               poolEntry = connectionBag.borrow(timeout, MILLISECONDS);
               if (poolEntry == null) {
                  break; // We timed out... break and throw exception
               }
               final long now = currentTime();
               if (poolEntry.isMarkedEvicted() || (elapsedMillis(poolEntry.lastAccessed, now) > ALIVE_BYPASS_WINDOW_MS && !isConnectionAlive(poolEntry.connection))) {
                  closeConnection(poolEntry, "(connection is evicted or dead)"); // Throw away the dead connection (passed max age or failed alive test)
                  timeout = hardTimeout - elapsedMillis(startTime);
               }
               else {
                  metricsTracker.recordBorrowStats(poolEntry, startTime);
                  return poolEntry.createProxyConnection(leakTask.schedule(poolEntry), now);
               }
            } while (timeout > 0L);
            metricsTracker.recordBorrowTimeoutStats(startTime);
         }
         catch (InterruptedException e) {
            if (poolEntry != null) {
               poolEntry.recycle(startTime);
            }
            Thread.currentThread().interrupt();
            throw new SQLException(poolName + " - Interrupted during connection acquisition", e);
         }
      }
      finally {
         suspendResumeLock.release();
      }
      throw createTimeoutException(startTime);
   }
```

我们具体来看一下isConnectionAlive的实现：

```Java
   boolean isConnectionAlive(final Connection connection)
   {
      try {
         try {
            setNetworkTimeout(connection, validationTimeout);
            final int validationSeconds = (int) Math.max(1000L, validationTimeout) / 1000;
            if (isUseJdbc4Validation) {
               return connection.isValid(validationSeconds);
            }
            try (Statement statement = connection.createStatement()) {
               if (isNetworkTimeoutSupported != TRUE) {
                  setQueryTimeout(statement, validationSeconds);
               }
               statement.execute(config.getConnectionTestQuery());
            }
         }
         finally {
            setNetworkTimeout(connection, networkTimeout);
            if (isIsolateInternalQueries && !isAutoCommit) {
               connection.rollback();
            }
         }
         return true;
      }
      catch (Exception e) {
         lastConnectionFailure.set(e);
         LOGGER.warn("{} - Failed to validate connection {} ({})", poolName, connection, e.getMessage());
         return false;
      }
   }
   /**
    * Set the network timeout, if <code>isUseNetworkTimeout</code> is <code>true</code> and the
    * driver supports it.
    *
    * @param connection the connection to set the network timeout on
    * @param timeoutMs the number of milliseconds before timeout
    * @throws SQLException throw if the connection.setNetworkTimeout() call throws
    */
   private void setNetworkTimeout(final Connection connection, final long timeoutMs) throws SQLException
   {
      if (isNetworkTimeoutSupported == TRUE) {
         connection.setNetworkTimeout(netTimeoutExecutor, (int) timeoutMs);
      }
   }
/**
    * Set the query timeout, if it is supported by the driver.
    *
    * @param statement a statement to set the query timeout on
    * @param timeoutSec the number of seconds before timeout
    */
   private void setQueryTimeout(final Statement statement, final int timeoutSec)
   {
      if (isQueryTimeoutSupported != FALSE) {
         try {
            statement.setQueryTimeout(timeoutSec);
            isQueryTimeoutSupported = TRUE;
         }
         catch (Throwable e) {
            if (isQueryTimeoutSupported == UNINITIALIZED) {
               isQueryTimeoutSupported = FALSE;
               LOGGER.info("{} - Failed to set query timeout for statement. ({})", poolName, e.getMessage());
            }
         }
      }
   }
```

从如下代码可以看到，validationTimeout的默认值是5000毫秒，所以默认情况下validationSeconds的值应该在1-5毫秒之间，又由于validationTimeout的值必须小于connectionTimeout（默认值30000毫秒，如果小于250毫秒，则被重置回30秒），所以默认情况下，调整validationTimeout却不调整connectionTimeout情况下，validationSeconds的默认峰值应该是30毫秒。

```Java
final int validationSeconds = (int) Math.max(1000L, validationTimeout) / 1000;
```

如果是jdbc4的话，如果使用isUseJdbc4Validation(就是config.getConnectionTestQuery() == null的时候)

```Java
this.isUseJdbc4Validation = config.getConnectionTestQuery() == null;
```

用connection.isValid(validationSeconds)来验证连接的有效性，否则的话则用connectionTestQuery查询语句来查询验证。这里说一下java.sql.Connection的isValid()和isClosed()的区别：

isValid：如果连接尚未关闭并且仍然有效，则返回 true。驱动程序将提交一个关于该连接的查询，或者使用其他某种能确切验证在调用此方法时连接是否仍然有效的机制。由驱动程序提交的用来验证该连接的查询将在当前事务的上下文中执行。
参数：timeout - 等待用来验证连接是否完成的数据库操作的时间，以秒为单位。如果在操作完成之前超时期满，则此方法返回 false。0 值表示不对数据库操作应用超时值。
返回：如果连接有效，则返回 true，否则返回 false

isClosed：查询此 Connection 对象是否已经被关闭。如果在连接上调用了 close 方法或者发生某些严重的错误，则连接被关闭。只有在调用了Connection.close 方法之后被调用时，此方法才保证返回true。通常不能调用此方法确定到数据库的连接是有效的还是无效的。通过捕获在试图进行某一操作时可能抛出的异常，典型的客户端可以确定某一连接是无效的。
返回：如果此 Connection 对象是关闭的，则返回 true；如果它仍然处于打开状态，则返回 false。

```Java
/**
         * Returns true if the connection has not been closed and is still valid.
         * The driver shall submit a query on the connection or use some other
         * mechanism that positively verifies the connection is still valid when
         * this method is called.
         * <p>
         * The query submitted by the driver to validate the connection shall be
         * executed in the context of the current transaction.
         *
         * @param timeout -             The time in seconds to wait for the database operation
         *                                              used to validate the connection to complete.  If
         *                                              the timeout period expires before the operation
         *                                              completes, this method returns false.  A value of
         *                                              0 indicates a timeout is not applied to the
         *                                              database operation.
         * <p>
         * @return true if the connection is valid, false otherwise
         * @exception SQLException if the value supplied for <code>timeout</code>
         * is less then 0
         * @since 1.6
         *
         * @see java.sql.DatabaseMetaData#getClientInfoProperties
         */
         boolean isValid(int timeout) throws SQLException;
             /**
     * Retrieves whether this <code>Connection</code> object has been
     * closed.  A connection is closed if the method <code>close</code>
     * has been called on it or if certain fatal errors have occurred.
     * This method is guaranteed to return <code>true</code> only when
     * it is called after the method <code>Connection.close</code> has
     * been called.
     * <P>
     * This method generally cannot be called to determine whether a
     * connection to a database is valid or invalid.  A typical client
     * can determine that a connection is invalid by catching any
     * exceptions that might be thrown when an operation is attempted.
     *
     * @return <code>true</code> if this <code>Connection</code> object
     *         is closed; <code>false</code> if it is still open
     * @exception SQLException if a database access error occurs
     */
    boolean isClosed() throws SQLException;
```

```Java
   public void acquire()
   {
      acquisitionSemaphore.acquireUninterruptibly();
   }
```

### newConnection

在com.zaxxer.hikari.pool.PoolBase的newConnection#setupConnection（）中，对于validationTimeout超时时间也做了getAndSetNetworkTimeout等的处理

# Hikari 2.7.5的故事

从validationTimeout我们刚才讲到了有一个HouseKeeper线程干着不断刷新的通过MBean调整的connectionTimeout和validationTimeout等值的事情。这就是2.7.4到2.7.5版本的一个很重要的改变，为什么这么说？

## 两个关键的Mbean

首先Hikari有两个Mbean，分别是HikariPoolMXBean和HikariConfigMXBean，我们看一下代码，这两个代码的功能不言而喻：

```Java
/**
 * The javax.management MBean for a Hikari pool instance.
 *
 * @author Brett Wooldridge
 */
public interface HikariPoolMXBean
{
   int getIdleConnections();
   int getActiveConnections();
   int getTotalConnections();
   int getThreadsAwaitingConnection();
   void softEvictConnections();
   void suspendPool();
   void resumePool();
}
```

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnUhKeldQcsMJ8CYTzcdUN60Q1yEKLme0hGXlvHkgm0BOrZUo7QVP28Ll8UaSdTbRtKd3rqANZeaHw/640)

## 2.7.5迎来了不可变设计

作者在18年1月5日做了一次代码提交：

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnUhKeldQcsMJ8CYTzcdUN60lYl5ckhict4sx8Wribmq0nibdSjgibWhqzBevzLUEOgojicEhLxib3AdLkKw/640)

导致大多数方法都不允许动态更新了：

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnUhKeldQcsMJ8CYTzcdUN60srSVKMR9l6UvBMyWFIia6H6S6iakXwKIzrt1HdFiaJyt1T2LkQdyLh7Kg/640)

可以这么认为，2.7.4是支持的，2.7.5作者搞了一下就变成了不可变设计，sb2.0默认支持2.7.6。

这会带来什么影响呢？如果你想运行时动态更新Hikari的Config除非命中可修改参数，否则直接给你抛异常了；当然，你更新代码写得不好也可能命中作者的这段抛异常逻辑。作者非常推荐使用Mbean去修改，不过你自己重新创建一个数据源使用CAP（Compare And Swap）也是可行的，所以我就只能如下改了一下，顺应了一下SB 2.0的时代：

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnUhKeldQcsMJ8CYTzcdUN60kcxeibw7GvU3UJ4EwHPJDIZFEaJNCHOodY35cTqiajAumrynjBMdElfQ/640)

如上图，左侧的字段都是Hikari在2.7.5以前亲测过可以动态更改的，不过jdbcurl不在这个范围之内，所以这就是为什么作者要做这么一个比较安全的不可变模式的导火索。

## 且看大神论道

某用户在1.1日给作者提了一个issue，就是jdbcurl无法动态修改的事情：
https://github.com/brettwooldridge/HikariCP/issues/1053

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnUhKeldQcsMJ8CYTzcdUN60dzYrlia9YBwiam6OkVyNvPTJPeb41BtgW8fNWkoDUxZWAr7qiao0dtoDg/640)

作者予以了回复,意思就是运行时可以更改的唯一池配置是通过HikariConfigMXBean，并增强的抛出一个IllegalStateException异常。两人达成一致，Makes sense，觉得非常Perfect，另外会完善一下JavaDoc。So,Sealed configuration makes it much harder to configure Hikari。

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnUhKeldQcsMJ8CYTzcdUN60S8H1cicntfEyxyEvHmSHxoCngZbhsk7cWibUDp2JflEVjwgOnVv0uJ1g/640)

然后俩人又开了一个ISSUE：
https://github.com/brettwooldridge/HikariCP/issues/231
但是在这里，俩人产生了一些设计相关的分歧，很有意思。

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnUhKeldQcsMJ8CYTzcdUN60S650XlFaoIHuTqCoFVF6CE6NvNjn3yUjQY3JbcQMklA07DrCqpuxqA/640)

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnUhKeldQcsMJ8CYTzcdUN60iaUpoTmEzB6iaqnMWaf0vbgyoqdjLDicK9T8epu2kb8iciapMd7K0R4e9PQ/640)

作者表明他的一些改变增加代码的复杂性，而不是增加它的价值，而作者对于Hikari的初衷是追求极致性能、追求极简设计。

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnUhKeldQcsMJ8CYTzcdUN60e61FqKVHrXC2oibImhQdBmkibDibT7A20B05H6OxPLVwdba2vssChiausA/640)

该用户建议作者提供add the ability to copy the configuration of one HikariDataSource into another的能力。作者予以了反驳：

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnUhKeldQcsMJ8CYTzcdUN60gxgeibDIiaO1xzKLyibsQUrLiaRn7ia10mXnGbglXKytT64PKURlxOo6Riag/640)

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnUhKeldQcsMJ8CYTzcdUN60y05ep7ibv6KyicTP0qpmxTXibgkjb9fJDUCDa0kC3oMcvUFUEAYfDXbIQ/640)

作者还是一如既往得追求他大道至简的思想以及两个Mbean的主张。

该用户继续着他的观点，

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnUhKeldQcsMJ8CYTzcdUN60EibLFlL2KNaReFAREW6pBImMibbAr5PxOmpGib9yjGIh2lPUh1Yrr9iaZA/640)

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnUhKeldQcsMJ8CYTzcdUN60dTOkKYu0kfJ2qFTOCqqicoTWXdZ88vLqlxgAdVqLrVPutRG9iawbHBqw/640)

可是作者貌似还是很坚持他的Hikari观点，作为吃瓜群众，看着大神论道，还是非常有意思的。

最后说说我的观点吧，我觉得作者对于Hikari，既然取名为光，就是追求极致，那些过度设计什么的他都会尽量摈弃的，我使用Hikari以及阅读源码的过程中也能感觉到，所以我觉得作者不会继续做这个需求，后续请关注我的真情实感的从实战及源码分析角度的体会《为什么HikariCP这么快》（不同于网上的其他文章）。

接下来说，我作为Hikari的使用者，我也是有能力完成Hikari的wrapper工作，我也可以去写外层的HouseKeeper，所以我觉得这并不是什么太大的问题，这次2.7.5的更新，很鸡肋的一个功能，但是却让我，作为一名追光者，走近了作者一点，走近了Hikari一点 ：）

# 666. 彩蛋

如果你对 HikariCP 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)