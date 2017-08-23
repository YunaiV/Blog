title: Sharding-JDBC 源码分析 —— 分布式事务（一）之最大努力型
date: 2017-08-20
tags:
categories: Sharding-JDBC
permalink: Sharding-JDBC/transaction-bed
keywords: Sharding-JDBC,ShardingJDBC,Sharding-JDBC 源码,JDBC,事务,分布式事务,柔性事务

-------

![](https://www.yunai.me/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

- [1. 概述](#)
- [2. 最大努力送达型](#)
- [3. 柔性事务管理器](#)
	- [3.1 概念](#)
	- [3.2 柔性事务配置](#)
	- [3.3 柔性事务](#)
		- [3.3.1 创建柔性事务](#)
- [4. 事务日志存储器](#)
	- [4.1 #add()](#)
	- [4.2 #remove()](#)
	- [4.3 #findEligibleTransactionLogs()](#)
	- [4.4 #increaseAsyncDeliveryTryTimes()](#)
	- [4.5 #processData()](#)
- [5. 最大努力送达型事务监听器](#)
- [6. 最大努力送达型异步作业](#)
	- [6.1 BestEffortsDeliveryJob](#)
	- [6.2 AsyncSoftTransactionJobConfiguration](#)
	- [6.3 Elastic-Job 是否必须？](#)
- [7. 适用场景](#)
- [8. 开发指南 & 开发示例](#)
- [666. 彩蛋](#)

-------

# 1. 概述

数据库表**分库**后，业务场景下的**单库本地事务**可能变成**跨库分布式事务**。虽然我们可以通过合适的**分库规则**让操作的数据在同库下，继续保证**单库本地事务**，这也是非常推崇的，但不是所有场景下都能适用。如果这些场景对事务的一致性有要求，我们就不得不解决分布式事务的“麻烦”。

**分布式事务**是个很大的话题，我们来看看 Sharding-JDBC 对她的权衡：

> Sharding-JDBC由于性能方面的考量，决定不支持强一致性分布式事务。我们已明确规划线路图，未来会支持最终一致性的柔性事务。

Sharding-JDBC 提供了两种 **柔性事务**：

* 最大努力送达型 BED ：已经实现
* 事务补偿型 TCC ：计划中

**本文分享 最大努力送达型 的实现**。建议前置阅读：[《Sharding-JDBC 源码分析 —— SQL 执行》](http://www.yunai.me/Sharding-JDBC/sql-execute/?self)。

> **Sharding-JDBC 正在收集使用公司名单：[传送门](https://github.com/dangdangdotcom/sharding-jdbc/issues/234)。  
> 🙂 你的登记，会让更多人参与和使用 Sharding-JDBC。[传送门](https://github.com/dangdangdotcom/sharding-jdbc/issues/234)  
> Sharding-JDBC 也会因此，能够覆盖更多的业务场景。[传送门](https://github.com/dangdangdotcom/sharding-jdbc/issues/234)  
> 登记吧，骚年！[传送门](https://github.com/dangdangdotcom/sharding-jdbc/issues/234)**

# 2. 最大努力送达型

**概念**

> 在分布式数据库的场景下，相信对于该数据库的操作最终一定可以成功，所以通过最大努力反复尝试送达操作。

从概念看，可能不是很直白的理解是什么意思，本文会**最大努力**让你干净理解。

**架构图**

> ![](http://www.yunai.me/images/Sharding-JDBC/2017_08_20/01.jpeg)

执行过程有 **四种** 情况：

1. 【红线】执行成功
2. 【棕线】执行失败，同步重试成功
3. 【粉线】执行失败，同步重试失败，异步重试成功
4. 【绿线】执行失败，同步重试失败，异步重试失败，事务日志保留

整体成漏斗倒三角，上一个阶段失败，交给下一个阶段重试：

![](http://www.yunai.me/images/Sharding-JDBC/2017_08_20/02.png)

整个过程通过如下 **组件** 完成：

* 柔性事务管理器
* 最大努力送达型柔性事务
* 最大努力送达型事务监听器
* 事务日志存储器
* 最大努力送达型异步作业

下面，我们逐节分享每个组件。

# 3. 柔性事务管理器

## 3.1 概念

柔性事务管理器，SoftTransactionManager 实现，负责对柔性事务配置( SoftTransactionConfiguration ) 、柔性事务( AbstractSoftTransaction )的管理。

## 3.2 柔性事务配置

调用 `#init()` 初始化柔性管理器：

```Java
// SoftTransactionManager.java
/**
* 柔性事务配置对象
*/
@Getter
private final SoftTransactionConfiguration transactionConfig;  

// SoftTransactionManager.java
/**
* 初始化事务管理器.
*/
public void init() throws SQLException {
   // 初始化 最大努力送达型事务监听器
   EventBusInstance.getInstance().register(new BestEffortsDeliveryListener());
   // 初始化 事务日志数据库存储表
   if (TransactionLogDataSourceType.RDB == transactionConfig.getStorageType()) {
       Preconditions.checkNotNull(transactionConfig.getTransactionLogDataSource());
       createTable();
   }
   // 初始化 内嵌的最大努力送达型异步作业
   if (transactionConfig.getBestEffortsDeliveryJobConfiguration().isPresent()) {
       new NestedBestEffortsDeliveryJobFactory(transactionConfig).init();
   }
}
```

* 将最大努力送达型事务监听器( BestEffortsDeliveryListener )注册到事务总线 ( EventBus )。在『最大努力送达型事务监听器』小节会详细分享
* 当使用**数据库**存储事务日志( TransactionLog ) 时，若**事务日志表( `transaction_log` )**不存在则进行创建。在『事务日志存储器』小节会详细分享
* 当配置使用**内嵌的**最大努力送达型异步作业( NestedBestEffortsDeliveryJob ) 时，进行初始化。在『最大努力送达型异步作业』小节会详细分享

**SoftTransactionConfiguration**

SoftTransactionConfiguration，柔性事务配置对象。

```Java
public class SoftTransactionConfiguration {
    /**
     * 事务管理器管理的数据源.
     */
    @Getter(AccessLevel.NONE)
    private final DataSource targetDataSource;
    
    /**
     * 同步的事务送达的最大尝试次数.
     */
    private int syncMaxDeliveryTryTimes = 3;
    
    /**
     * 事务日志存储类型.
     */
    private TransactionLogDataSourceType storageType = RDB;
    /**
     * 存储事务日志的数据源.
     */
    private DataSource transactionLogDataSource;
    
    /**
     * 内嵌的最大努力送达型异步作业配置对象.
     */
    private Optional<NestedBestEffortsDeliveryJobConfiguration> bestEffortsDeliveryJobConfiguration = Optional.absent();
}
```

## 3.3 柔性事务

在 Sharding-JDBC 里，目前柔性事务分成两种：

* BEDSoftTransaction ：最大努力送达型柔性事务
* TCCSoftTransaction ：TCC型柔性事务

**继承 AbstractSoftTransaction**

```Java
public abstract class AbstractSoftTransaction {
    /**
     * 分片连接原自动提交状态
     */
    private boolean previousAutoCommit;
    /**
     * 分片连接
     */
    @Getter
    private ShardingConnection connection;
    /**
     * 事务类型
     */
    @Getter
    private SoftTransactionType transactionType;
    /**
     * 事务编号
     */
    @Getter
    private String transactionId;
}
```



AbstractSoftTransaction 实现了开启柔性事务、关闭柔性事务两个方法提供给子类调用：

* `#beginInternal()`

    ```Java
    /**
    * 开启柔性
    *
    * @param conn 分片连接
    * @param type 事务类型
    * @throws SQLException
    */
    protected final void beginInternal(final Connection conn, final SoftTransactionType type) throws SQLException {
       // TODO 判断如果在传统事务中，则抛异常
       Preconditions.checkArgument(conn instanceof ShardingConnection, "Only ShardingConnection can support eventual consistency transaction.");
       // 设置执行错误，不抛出异常
       ExecutorExceptionHandler.setExceptionThrown(false);
       connection = (ShardingConnection) conn;
       transactionType = type;
       // 设置自动提交状态
       previousAutoCommit = connection.getAutoCommit();
       connection.setAutoCommit(true);
       // 生成事务编号
       // TODO 替换UUID为更有效率的id生成器
       transactionId = UUID.randomUUID().toString();
    }
    ```


    * 调用 `ExecutorExceptionHandler.setExceptionThrown(false)` 设置执行 SQL 错误时，也不抛出异常。
        * 对异常处理的代码：[ExecutorExceptionHandler#setExceptionThrown()](https://github.com/dangdangdotcom/sharding-jdbc/blob/884b38f4c2402e31464d15b444f4b405e07fe211/sharding-jdbc-core/src/main/java/com/dangdang/ddframe/rdb/sharding/executor/threadlocal/ExecutorExceptionHandler.java#L59) 
        * 对于其他 SQL，不会因为 SQL 错误不执行，会继续执行
        * 对于上层业务，不会因为 SQL 错误终止逻辑，会继续执行。这里有一点要注意下，上层业务不能对该 SQL 执行结果有强依赖，因为 SQL 错误需要重试达到数据最终一致性
        * 对于**最大努力型事务**( TCC暂未实现 )，会对执行错误的 SQL 进行重试

    * 调用 `connection.setAutoCommit(true);`，设置执行自动提交。**使用最大努力型事务时，上层业务执行 SQL 会马上提交，即使调用  `Connection#rollback()` 也是无法回滚的，这点一定要注意。**
   
* `#end()`

    ```Java
    /**
    * 结束柔性事务.
    */
    public final void end() throws SQLException {
      if (connection != null) {
          ExecutorExceptionHandler.setExceptionThrown(true);
          connection.setAutoCommit(previousAutoCommit);
          SoftTransactionManager.closeCurrentTransactionManager();
      }
    }
        
    // SoftTransactionManager.java
    /**
    * 关闭当前的柔性事务管理器.
    */
    static void closeCurrentTransactionManager() {
       ExecutorDataMap.getDataMap().put(TRANSACTION, null);
       ExecutorDataMap.getDataMap().put(TRANSACTION_CONFIG, null);
    }
    ```

    * 事务结束后，一定要记得调用 `#end()` 清理线程变量。否则，下次请求使用到该线程，会继续在这个柔性事务内。

    
**BEDSoftTransaction**    

BEDSoftTransaction，最大努力送达型柔性事务。

```Java
public class BEDSoftTransaction extends AbstractSoftTransaction {
    
    /**
     * 开启柔性事务.
     * 
     * @param connection 数据库连接对象
     */
    public void begin(final Connection connection) throws SQLException {
        beginInternal(connection, SoftTransactionType.BestEffortsDelivery);
    }
}
```

**TCCSoftTransaction**

TCCSoftTransaction，TCC 型柔性事务，暂未实现。实现后，会更新到 [《Sharding-JDBC 源码分析 —— 分布式事务（二）之事务补偿型》](http://www.yunai.me/Sharding-JDBC/transaction-tcc/?self)。

-------

### 3.3.1 创建柔性事务

通过调用 `SoftTransactionManager#getTransaction()` 创建柔性事务对象：

```Java
/**
* {@link ExecutorDataMap#dataMap} 柔性事务对象 key
*/
private static final String TRANSACTION = "transaction";
/**
* {@link ExecutorDataMap#dataMap} 柔性事务配置 key
*/
private static final String TRANSACTION_CONFIG = "transactionConfig";

// SoftTransactionManager.java
/**
* 创建柔性事务.
* 
* @param type 柔性事务类型
* @return 柔性事务
*/
public AbstractSoftTransaction getTransaction(final SoftTransactionType type) {
   AbstractSoftTransaction result;
   switch (type) {
       case BestEffortsDelivery: 
           result = new BEDSoftTransaction();
           break;
       case TryConfirmCancel:
           result = new TCCSoftTransaction();
           break;
       default: 
           throw new UnsupportedOperationException(type.toString());
   }
   // TODO 目前使用不支持嵌套事务，以后这里需要可配置
   if (getCurrentTransaction().isPresent()) {
       throw new UnsupportedOperationException("Cannot support nested transaction.");
   }
   ExecutorDataMap.getDataMap().put(TRANSACTION, result);
   ExecutorDataMap.getDataMap().put(TRANSACTION_CONFIG, transactionConfig);
   return result;
}
```

* 后续可以从 [ExecutorDataMap](https://github.com/dangdangdotcom/sharding-jdbc/blob/884b38f4c2402e31464d15b444f4b405e07fe211/sharding-jdbc-core/src/main/java/com/dangdang/ddframe/rdb/sharding/executor/threadlocal/ExecutorDataMap.java) 中获取当前线程的柔性事务和柔性事务配置：

    ```Java
    // SoftTransactionManager.java
    /**
    * 获取当前线程的柔性事务配置.
    * 
    * @return 当前线程的柔性事务配置
    */
    public static Optional<SoftTransactionConfiguration> getCurrentTransactionConfiguration() {
       Object transactionConfig = ExecutorDataMap.getDataMap().get(TRANSACTION_CONFIG);
       return (null == transactionConfig)
               ? Optional.<SoftTransactionConfiguration>absent()
               : Optional.of((SoftTransactionConfiguration) transactionConfig);
    }
        
    /**
    * 获取当前的柔性事务.
    * 
    * @return 当前的柔性事务
    */
    public static Optional<AbstractSoftTransaction> getCurrentTransaction() {
       Object transaction = ExecutorDataMap.getDataMap().get(TRANSACTION);
       return (null == transaction)
               ? Optional.<AbstractSoftTransaction>absent()
               : Optional.of((AbstractSoftTransaction) transaction);
    }
    ```

# 4. 事务日志存储器

柔性事务执行过程中，会通过事务日志( TransactionLog ) 记录每条 SQL 执行状态：

* SQL 执行前，记录一条事务日志
* SQL 执行成功，移除对应的事务日志 

通过实现事务日志存储器接口( TransactionLogStorage )，提供存储功能。目前有两种实现：

* MemoryTransactionLogStorage ：基于**内存**的事务日志存储器。主要用于开发测试，**生产环境下不要使用**。
* RdbTransactionLogStorage ：基于**数据库**的事务日志存储器。

本节只分析 RdbTransactionLogStorage。对 [MemoryTransactionLogStorage](https://github.com/dangdangdotcom/sharding-jdbc/blob/884b38f4c2402e31464d15b444f4b405e07fe211/sharding-jdbc-transaction-parent/sharding-jdbc-transaction-storage/src/main/java/com/dangdang/ddframe/rdb/transaction/soft/storage/impl/RdbTransactionLogStorage.java) 感兴趣的同学可以点击链接传送到达。

**TransactionLogStorage 有五个接口方法，下文每个小标题都是一个方法。**

## 4.1 #add()

```Java
// TransactionLogStorage.java
/**
* 存储事务日志.
* 
* @param transactionLog 事务日志
*/
void add(TransactionLog transactionLog);

// RdbTransactionLogStorage.java
@Override
public void add(final TransactionLog transactionLog) {
   String sql = "INSERT INTO `transaction_log` (`id`, `transaction_type`, `data_source`, `sql`, `parameters`, `creation_time`) VALUES (?, ?, ?, ?, ?, ?);";
   try (
    // ... 省略你熟悉的代码
   } catch (final SQLException ex) {
       throw new TransactionLogStorageException(ex);
   }
}
```

* **注意**：如果插入事务日志**失败**，SQL 会继续执行，如果此时 SQL 执行失败，则该 SQL 会不见了。建议：`#add()` 和下文的 `#remove()` 异常时，都打印下异常日志都文件系统

TransactionLog (transaction_log) 数据库表结构如下：

| 字段 | 名字 | 数据库类型 | 备注 |
| --- | --- | --- | --- |
| id | 事件编号 | VARCHAR(40) | EventBus 事件编号，**非事务编号** |
| transaction_type | 柔性事务类型 | VARCHAR(30) |
| data_source | 真实数据源名 | VARCHAR(255) |  |
| sql | 执行 SQL | TEXT | 已经改写过的 SQL |
| parameters | 占位符参数 | TEXT | JSON 字符串存储 |
| creation_time | 记录时间 | LONG |
| async_delivery_try_times | 已异步重试次数 | INT |


## 4.2 #remove()

```Java
// TransactionLogStorage.java
/**
* 根据主键删除事务日志.
* 
* @param id 事务日志主键
*/
void remove(String id);
    
// RdbTransactionLogStorage.java    
@Override
public void remove(final String id) {
   String sql = "DELETE FROM `transaction_log` WHERE `id`=?;";
   try (
          // ... 省略你熟悉的代码
   } catch (final SQLException ex) {
       throw new TransactionLogStorageException(ex);
   }
}
```

## 4.3 #findEligibleTransactionLogs()

```Java
// TransactionLogStorage.java
/**
* 读取需要处理的事务日志.
* 
* <p>需要处理的事务日志为: </p>
* <p>1. 异步处理次数小于最大处理次数.</p>
* <p>2. 异步处理的事务日志早于异步处理的间隔时间.</p>
* 
* @param size 获取日志的数量
* @param maxDeliveryTryTimes 事务送达的最大尝试次数
* @param maxDeliveryTryDelayMillis 执行送达事务的延迟毫秒数.
*/
List<TransactionLog> findEligibleTransactionLogs(int size, int maxDeliveryTryTimes, long maxDeliveryTryDelayMillis);

// RdbTransactionLogStorage.java
@Override
public List<TransactionLog> findEligibleTransactionLogs(final int size, final int maxDeliveryTryTimes, final long maxDeliveryTryDelayMillis) {
   List<TransactionLog> result = new ArrayList<>(size);
   String sql = "SELECT `id`, `transaction_type`, `data_source`, `sql`, `parameters`, `creation_time`, `async_delivery_try_times` "
       + "FROM `transaction_log` WHERE `async_delivery_try_times`<? AND `transaction_type`=? AND `creation_time`<? LIMIT ?;";
   try (Connection conn = dataSource.getConnection()) {
       // ... 省略你熟悉的代码
   } catch (final SQLException ex) {
       throw new TransactionLogStorageException(ex);
   }
   return result;
}
```

## 4.4 #increaseAsyncDeliveryTryTimes()

```Java
// TransactionLogStorage.java
/**
* 增加事务日志异步重试次数.
* 
* @param id 事务主键
*/
void increaseAsyncDeliveryTryTimes(String id);

// RdbTransactionLogStorage.java
@Override
public void increaseAsyncDeliveryTryTimes(final String id) {
   String sql = "UPDATE `transaction_log` SET `async_delivery_try_times`=`async_delivery_try_times`+1 WHERE `id`=?;";
   try (
       // ... 省略你熟悉的代码
   } catch (final SQLException ex) {
       throw new TransactionLogStorageException(ex);
   }
}
```

## 4.5 #processData()

```Java
// TransactionLogStorage.java
/**
* 处理事务数据.
*
* @param connection 业务数据库连接
* @param transactionLog 事务日志
* @param maxDeliveryTryTimes 事务送达的最大尝试次数
*/
boolean processData(Connection connection, TransactionLog transactionLog, int maxDeliveryTryTimes);

// RdbTransactionLogStorage.java
@Override
public boolean processData(final Connection connection, final TransactionLog transactionLog, final int maxDeliveryTryTimes) {
   // 重试执行失败 SQL
   try (
       Connection conn = connection;
       PreparedStatement preparedStatement = conn.prepareStatement(transactionLog.getSql())) {
       for (int parameterIndex = 0; parameterIndex < transactionLog.getParameters().size(); parameterIndex++) {
           preparedStatement.setObject(parameterIndex + 1, transactionLog.getParameters().get(parameterIndex));
       }
       preparedStatement.executeUpdate();
   } catch (final SQLException ex) {
       // 重试失败，更新事务日志，增加已异步重试次数
       increaseAsyncDeliveryTryTimes(transactionLog.getId());
       throw new TransactionCompensationException(ex);
   }
   // 移除重试执行成功 SQL 对应的事务日志
   remove(transactionLog.getId());
   return true;
}
```

* 不同于前四个**增删改查**接口方法的实现，`#processData()` 是带有一些逻辑的。根据事务日志( TransactionLog )重试执行失败的 SQL，若成功，移除事务日志；若失败，更新事务日志，增加已异步重试次数
* 该方法会被**最大努力送达型异步作业**调用到

# 5. 最大努力送达型事务监听器

最大努力送达型事务监听器，BestEffortsDeliveryListener，负责记录事务日志、同步重试执行失败 SQL。

```Java
// BestEffortsDeliveryListener.java
@Subscribe
@AllowConcurrentEvents
public void listen(final DMLExecutionEvent event) {
   if (!isProcessContinuously()) {
       return;
   }
   SoftTransactionConfiguration transactionConfig = SoftTransactionManager.getCurrentTransactionConfiguration().get();
   TransactionLogStorage transactionLogStorage = TransactionLogStorageFactory.createTransactionLogStorage(transactionConfig.buildTransactionLogDataSource());
   BEDSoftTransaction bedSoftTransaction = (BEDSoftTransaction) SoftTransactionManager.getCurrentTransaction().get();
   switch (event.getEventExecutionType()) {
       case BEFORE_EXECUTE: // 执行前，插入事务日志
           //TODO 对于批量执行的SQL需要解析成两层列表
           transactionLogStorage.add(new TransactionLog(event.getId(), bedSoftTransaction.getTransactionId(), bedSoftTransaction.getTransactionType(), 
                   event.getDataSource(), event.getSql(), event.getParameters(), System.currentTimeMillis(), 0));
           return;
       case EXECUTE_SUCCESS: // 执行成功，移除事务日志
           transactionLogStorage.remove(event.getId());
           return;
       case EXECUTE_FAILURE: // 执行失败，同步重试
           boolean deliverySuccess = false;
           for (int i = 0; i < transactionConfig.getSyncMaxDeliveryTryTimes(); i++) { // 同步【多次】重试
               if (deliverySuccess) {
                   return;
               }
               boolean isNewConnection = false;
               Connection conn = null;
               PreparedStatement preparedStatement = null;
               try {
                   // 获得数据库连接
                   conn = bedSoftTransaction.getConnection().getConnection(event.getDataSource(), SQLType.DML);
                   if (!isValidConnection(conn)) { // 因为可能执行失败是数据库连接异常，所以判断一次，如果无效，重新获取数据库连接
                       bedSoftTransaction.getConnection().release(conn);
                       conn = bedSoftTransaction.getConnection().getConnection(event.getDataSource(), SQLType.DML);
                       isNewConnection = true;
                   }
                   preparedStatement = conn.prepareStatement(event.getSql());
                   // 同步重试
                   //TODO 对于批量事件需要解析成两层列表
                   for (int parameterIndex = 0; parameterIndex < event.getParameters().size(); parameterIndex++) {
                       preparedStatement.setObject(parameterIndex + 1, event.getParameters().get(parameterIndex));
                   }
                   preparedStatement.executeUpdate();
                   deliverySuccess = true;
                   // 同步重试成功，移除事务日志
                   transactionLogStorage.remove(event.getId());
               } catch (final SQLException ex) {
                   log.error(String.format("Delivery times %s error, max try times is %s", i + 1, transactionConfig.getSyncMaxDeliveryTryTimes()), ex);
               } finally {
                   close(isNewConnection, conn, preparedStatement);
               }
           }
           return;
       default: 
           throw new UnsupportedOperationException(event.getEventExecutionType().toString());
   }
}
```

* BestEffortsDeliveryListener 通过 EventBus 实现监听 SQL 的执行。Sharding-JDBC 如何实现 EventBus 的，请看[《Sharding-JDBC 源码分析 —— SQL 执行》](http://www.yunai.me/Sharding-JDBC/sql-execute/?self)
* 调用 `#isProcessContinuously()` 方法判断是否处于**最大努力送达型事务**中，当且仅当处于该状态才进行监听事件处理
* SQL 执行**前**，插入事务日志
* SQL 执行**成功**，移除事务日志
* SQL 执行**失败**，根据柔性事务配置( SoftTransactionConfiguration )同步的事务送达的最大尝试次数( `syncMaxDeliveryTryTimes` )进行多次重试**直到成功**。总体逻辑和 `RdbTransactionLogStorage#processData()` 方法逻辑类似，区别在于**获取分片数据库连接**的特殊处理：此处调用失败，数据库连接可能是异常无效的，因此调用了 `#isValidConnection()` 判断连接的**有效性**。若无效，则重新获取分片数据库连接。另外，若是重新获取分片数据库连接，需要进行关闭释放 (`Connection#close()`)：

    ```Java
    // BestEffortsDeliveryListener.java
    /**
    * 通过 SELECT 1 校验数据库连接是否有效
    *
    * @param conn 数据库连接
    * @return 是否有效
    */
    private boolean isValidConnection(final Connection conn) {
       try (PreparedStatement preparedStatement = conn.prepareStatement("SELECT 1")) {
           try (ResultSet rs = preparedStatement.executeQuery()) {
               return rs.next() && 1 == rs.getInt("1");
           }
       } catch (final SQLException ex) {
           return false;
       }
    }
    
    /**
    * 关闭释放预编译SQL对象和数据库连接
    *
    * @param isNewConnection 是否新创建的数据库连接，是的情况下才释放
    * @param conn 数据库连接
    * @param preparedStatement 预编译SQL
    */
    private void close(final boolean isNewConnection, final Connection conn, final PreparedStatement preparedStatement) {
       if (null != preparedStatement) {
           try {
               preparedStatement.close();
           } catch (final SQLException ex) {
               log.error("PreparedStatement closed error:", ex);
           }
       }
       if (isNewConnection && null != conn) {
           try {
               conn.close();
           } catch (final SQLException ex) {
               log.error("Connection closed error:", ex);
           }
       }
    }
    ```

# 6. 最大努力送达型异步作业

当最大努力送达型事务监听器( BestEffortsDeliveryListener )**多次同步**重试失败后，交给**最大努力送达型异步作业**进行**多次异步**重试，并且多次执行有**固定间隔**。

Sharding-JDBC 提供了两个最大努力送达型异步作业实现：

* NestedBestEffortsDeliveryJob ：内嵌的最大努力送达型异步作业
* BestEffortsDeliveryJob ：最大努力送达型异步作业

两者实现代码逻辑**基本一致**。前者相比后者，用于开发测试，去除对 Zookeeper 依赖，无法实现**高可用**，因此**生产环境下不适合使用**。

## 6.1 BestEffortsDeliveryJob

BestEffortsDeliveryJob 所在 Maven 项目为 `sharding-jdbc-transaction-async-job`，基于当当开源的 [Elastic-Job](https://github.com/dangdangdotcom/elastic-job) 实现。如下是官方对该 Maven 项目的简要说明：

> 由于柔性事务采用异步尝试，需要部署独立的作业和Zookeeper。sharding-jdbc-transaction采用elastic-job实现的sharding-jdbc-transaction-async-job，通过简单配置即可启动高可用作业异步送达柔性事务，启动脚本为start.sh。

**BestEffortsDeliveryJob**

```Java
public class BestEffortsDeliveryJob extends AbstractIndividualThroughputDataFlowElasticJob<TransactionLog> {

    /**
     * 最大努力送达型异步作业配置对象
     */
    @Setter
    private BestEffortsDeliveryConfiguration bedConfig;
    /**
     * 事务日志存储器对象
     */
    @Setter
    private TransactionLogStorage transactionLogStorage;

    @Override
    public List<TransactionLog> fetchData(final JobExecutionMultipleShardingContext context) {
        return transactionLogStorage.findEligibleTransactionLogs(context.getFetchDataCount(),
            bedConfig.getJobConfig().getMaxDeliveryTryTimes(), bedConfig.getJobConfig().getMaxDeliveryTryDelayMillis());
    }

    @Override
    public boolean processData(final JobExecutionMultipleShardingContext context, final TransactionLog data) {
        try (
            Connection conn = bedConfig.getTargetDataSource(data.getDataSource()).getConnection()) {
            transactionLogStorage.processData(conn, data, bedConfig.getJobConfig().getMaxDeliveryTryTimes());
        } catch (final SQLException | TransactionCompensationException ex) {
            log.error(String.format("Async delivery times %s error, max try times is %s, exception is %s", data.getAsyncDeliveryTryTimes() + 1, 
                bedConfig.getJobConfig().getMaxDeliveryTryTimes(), ex.getMessage()));
            return false;
        }
        return true;
    }
    
    @Override
    public boolean isStreamingProcess() {
        return false;
    }
}
```

* 调用 `#fetchData()` 方法获取需要处理的事务日志 (TransactionLog)，内部调用了 `TransactionLogStorage#findEligibleTransactionLogs()` 方法
* 调用 `#processData()` 方法处理事务日志，重试执行失败的 SQL，内部调用了 `TransactionLogStorage#processData()`
* `#fetchData()` 和 `#processData()` 调用是 Elastic-Job 控制的。每一轮定时调度，**每条**事务日志只执行**一次**。当**超过**最大异步调用次数后，该条事务日志不再处理，所以**生产使用时，最好增加下相应监控超过最大异步重试次数的事务日志**。

## 6.2 AsyncSoftTransactionJobConfiguration

AsyncSoftTransactionJobConfiguration，异步柔性事务作业配置对象。

```Java
public class AsyncSoftTransactionJobConfiguration {
    
    /**
     * 作业名称.
     */
    private String name = "bestEffortsDeliveryJob";
    
    /**
     * 触发作业的cron表达式.
     */
    private String cron = "0/5 * * * * ?";
    
    /**
     * 每次作业获取的事务日志最大数量.
     */
    private int transactionLogFetchDataCount = 100;
    
    /**
     * 事务送达的最大尝试次数.
     */
    private int maxDeliveryTryTimes = 3;
    
    /**
     * 执行事务的延迟毫秒数.
     *
     * <p>早于此间隔时间的入库事务才会被作业执行.</p>
     */
    private long maxDeliveryTryDelayMillis = 60  * 1000L;
}
```

## 6.3 Elastic-Job 是否必须？

Sharding-JDBC 提供的最大努力送达型异步作业实现( BestEffortsDeliveryJob )，通过与 Elastic-Job 集成，可以很便捷并且有质量保证的**高可用**、**高性能**使用。一部分团队，可能已经引入或自研了类似 Elastic-Job 的分布式作业中间件解决方案，每多一个中间件，就是多一个学习与运维成本。那么是否可以使用自己的分布式作业解决方案？答案是，可以的。参考 BestEffortsDeliveryJob 的实现，通过调用 TransactionLogStorage 来实现：

```Java
// 伪代码(不考虑性能、异常)
List<TransactionLog> transactionLogs = transactionLogStorage.findEligibleTransactionLogs(....);
for (TransactionLog transactionLog : transactionLogs) {
       transactionLogStorage.processData(conn, log, maxDeliveryTryTimes);
}
```

当然，个人还是很推荐 Elastic-Job。  

😈 **笔者要开始写[《Elastic-Job 源码分析》](http://www.yunai.me/images/common/wechat_mp_2017_07_31_bak.jpg)**。

-------

另外，如果有支持**事务消息**的分布式队列系统，可以通过 TransactionLogStorage 实现存储事务消息存储成消息。为什么要支持**事务消息**？如果 SQL 执行是成功的，需要回滚（删除）事务消息。

# 7. 适用场景

见[《官方文档 - 事务支持》](http://dangdangdotcom.github.io/sharding-jdbc/02-guide/transaction/)。

# 8. 开发指南 & 开发示例

见[《官方文档 - 事务支持》](http://dangdangdotcom.github.io/sharding-jdbc/02-guide/transaction/)。

# 666. 彩蛋

哈哈哈

算是坚持把这个系列写完了，给自己 32 個赞。

满足！

[《Elastic-Job 源码分析》](http://www.yunai.me/images/common/wechat_mp_2017_07_31_bak.jpg) 走起！不 High 不结束！


