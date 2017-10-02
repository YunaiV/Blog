title: TCC-Transaction 源码分析 —— 事务存储器
date: 2018-02-15
tags:
categories: TCC-Transaction
permalink: TCC-Transaction/transaction-repository

---

**本文主要基于 TCC-Transaction 1.2.3.3 正式版**  

- [1. 概述](#)
- [2. 序列化](#)
	- [2.1 JDK 序列化实现](#)
	- [2.2 Kyro 序列化实现](#)
	- [2.3 JSON 序列化实现](#)
- [3. 存储器](#)
	- [3.1 可缓存的事务存储器抽象类](#)
	- [3.2 JDBC 事务存储器](#)
	- [3.3 Redis 事务存储器](#)
	- [3.4 Zookeeper 事务存储器](#)
	- [3.5 File 事务存储器](#)
- [666. 彩蛋](#)

---

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

---

# 1. 概述

本文分享 **事务存储器**。主要涉及如下 Maven 项目：

* `tcc-transaction-core` ：tcc-transaction 底层实现。

在 TCC 的过程中，根据应用内存中的事务信息完成整个事务流程。But 实际业务场景中，将事务信息只放在应用内存中是远远不够可靠的。例如：

1. 应用进程异常崩溃，未完成的事务信息将丢失。
2. 应用进程集群，当提供远程服务调用时，事务信息需要集群内共享。
3. 发起事务的应用需要重启部署新版本，因为各种原因，有未完成的事务。

因此，TCC-Transaction 将事务信息添加到内存中的同时，会使用外部存储进行持久化。目前提供四种外部存储：

* JdbcTransactionRepository，JDBC 事务存储器
* RedisTransactionRepository，Redis 事务存储器
* ZooKeeperTransactionRepository，Zookeeper 事务存储器
* FileSystemTransactionRepository，File 事务存储器

本文涉及到的类关系如下图( [打开大图](http://www.iocoder.cn/images/TCC-Transaction/2018_02_15/01.png) )：

![](http://www.iocoder.cn/images/TCC-Transaction/2018_02_15/01.png)

> 你行好事会因为得到赞赏而愉悦  
> 同理，开源项目贡献者会因为 Star 而更加有动力  
> 为 TCC-Transaction 点赞！[传送门](https://github.com/changmingxie/tcc-transaction)

ps：笔者假设你已经阅读过[《tcc-transaction 官方文档 —— 使用指南1.2.x》](https://github.com/changmingxie/tcc-transaction/wiki/%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%971.2.x)。

# 2. 序列化

在[《TCC-Transaction 源码分析 —— TCC 实现》「4. 事务与参与者」](http://www.iocoder.cn/TCC-Transaction/tcc-core/?self)，可以看到 Transaction 是一个比较复杂的对象，内嵌 Participant 数组，而 Participant 本身也是复杂的对象，内嵌了更多的其他对象，因此，存储器在持久化 Transaction 时，需要序列化后才能存储。

`org.mengyun.tcctransaction.serializer.ObjectSerializer`，对象序列化**接口**。实现代码如下：

```Java
public interface ObjectSerializer<T> {

    byte[] serialize(T t);
    
    T deserialize(byte[] bytes);

}
```

目前提供 **JDK自带序列化** 和 **Kyro序列化** 两种实现。

## 2.1 JDK 序列化实现

`org.mengyun.tcctransaction.serializer.JdkSerializationSerializer`，JDK 序列化实现。比较易懂，点击[链接](https://github.com/changmingxie/tcc-transaction/blob/70130d12004456fd4b97510c210c24502a1b3acb/tcc-transaction-core/src/main/java/org/mengyun/tcctransaction/serializer/JdkSerializationSerializer.java)直接查看。

**TCC-Transaction 使用的默认的序列化**。

## 2.2 Kyro 序列化实现

`org.mengyun.tcctransaction.serializer.KryoTransactionSerializer`，Kyro 序列化实现。比较易懂，点击[链接](https://github.com/changmingxie/tcc-transaction/blob/70130d12004456fd4b97510c210c24502a1b3acb/tcc-transaction-core/src/main/java/org/mengyun/tcctransaction/serializer/KryoTransactionSerializer.java)直接查看。

## 2.3 JSON 序列化实现

JDK 和 Kyro 的序列化实现，肉眼无法直观具体存储事务的信息，你可以通过实现 ObjectSerializer 接口，实现自定义的 JSON 序列化。

# 3. 存储器

`org.mengyun.tcctransaction.TransactionRepository`，事务存储器**接口**。实现代码如下：

```Java
public interface TransactionRepository {

    /**
     * 新增事务
     *
     * @param transaction 事务
     * @return 新增数量
     */
    int create(Transaction transaction);

    /**
     * 更新事务
     *
     * @param transaction 事务
     * @return 更新数量
     */
    int update(Transaction transaction);

    /**
     * 删除事务
     *
     * @param transaction 事务
     * @return 删除数量
     */
    int delete(Transaction transaction);

    /**
     * 获取事务
     *
     * @param xid 事务编号
     * @return 事务
     */
    Transaction findByXid(TransactionXid xid);

    /**
     * 获取超过指定时间的事务集合
     *
     * @param date 指定时间
     * @return 事务集合
     */
    List<Transaction> findAllUnmodifiedSince(Date date);
}
```

不同的存储器通过实现该接口，提供事务的增删改查功能。

## 3.1 可缓存的事务存储器抽象类

`org.mengyun.tcctransaction.repository.CachableTransactionRepository`，**可缓存**的事务存储器**抽象类**，实现增删改查事务时，同时缓存事务信息。在上面类图，我们也可以看到 TCC-Transaction 自带的多种存储器都继承该抽象类。

**CachableTransactionRepository 构造方法**实现代码如下：

```Java
public abstract class CachableTransactionRepository implements TransactionRepository {

    /**
     * 缓存过期时间
     */
    private int expireDuration = 120;
    /**
     * 缓存
     */
    private Cache<Xid, Transaction> transactionXidCompensableTransactionCache;

    public CachableTransactionRepository() {
        transactionXidCompensableTransactionCache = CacheBuilder.newBuilder().expireAfterAccess(expireDuration, TimeUnit.SECONDS).maximumSize(1000).build();
    }
}
```

* 使用 [Guava Cache](https://github.com/google/guava/wiki/CachesExplained) 内存缓存事务信息，设置最大缓存个数为 1000 个，缓存过期时间为最后访问时间 120 秒。

-------

**`#create(...)`** 实现代码如下：

```Java
@Override
public int create(Transaction transaction) {
   int result = doCreate(transaction);
   if (result > 0) {
       putToCache(transaction);
   }
   return result;
}

/**
* 添加到缓存
*
* @param transaction 事务
*/
protected void putToCache(Transaction transaction) {
   transactionXidCompensableTransactionCache.put(transaction.getXid(), transaction);
}

/**
* 新增事务
*
* @param transaction 事务
* @return 新增数量
*/
protected abstract int doCreate(Transaction transaction);
```

* 调用 `#doCreate(...)` 方法，新增事务。新增成功后，调用 `#putToCache(...)` 方法，添加事务到缓存。
* `#doCreate(...)` 为抽象方法，子类实现该方法，提供新增事务功能。

-------

**`#update(...)`** 实现代码如下：

```Java
@Override
public int update(Transaction transaction) {
   int result = 0;
   try {
       result = doUpdate(transaction);
       if (result > 0) {
           putToCache(transaction);
       } else {
           throw new OptimisticLockException();
       }
   } finally {
       if (result <= 0) { // 更新失败，移除缓存。下次访问，从存储器读取
           removeFromCache(transaction);
       }
   }
   return result;
}

/**
* 移除事务从缓存
*
* @param transaction 事务
*/
protected void removeFromCache(Transaction transaction) {
   transactionXidCompensableTransactionCache.invalidate(transaction.getXid());
}

/**
* 更新事务
*
* @param transaction 事务
* @return 更新数量
*/
protected abstract int doUpdate(Transaction transaction);
```

* 调用 `#doUpdate(...)` 方法，更新事务。
    * 若更新成功后，调用 `#putToCache(...)` 方法，添加事务到缓存。
    * 若更新失败后，抛出 OptimisticLockException 异常。有两种情况会导致更新失败：(1) 该事务已经被提交，被删除；(2) 乐观锁更新时，缓存的事务的版本号( `Transaction.version` )和存储器里的事务的版本号不同，更新失败。**为什么**？在[《TCC-Transaction 源码分析 —— 事务恢复》](http://www.iocoder.cn/TCC-Transaction/transaction-recovery/)详细解析。更新失败，意味着缓存已经不不一致，调用 `#removeFromCache(...)` 方法，移除事务从缓存中。
* `#doUpdate(...)` 为抽象方法，子类实现该方法，提供更新事务功能。
    

-------

**`#delete(...)`** 实现代码如下：

```Java
@Override
public int delete(Transaction transaction) {
   int result;
   try {
       result = doDelete(transaction);
   } finally {
       removeFromCache(transaction);
   }
   return result;
}

/**
* 删除事务
*
* @param transaction 事务
* @return 删除数量
*/
protected abstract int doDelete(Transaction transaction);
```

* 调用 `#doDelete(...)` 方法，删除事务。
* 调用 `#removeFromCache(...)` 方法，移除事务从缓存中。
* `#doDelete(...)` 为抽象方法，子类实现该方法，提供删除事务功能。

-------

**`#findByXid(...)`** 实现代码如下：

```Java
@Override
public Transaction findByXid(TransactionXid transactionXid) {
   Transaction transaction = findFromCache(transactionXid);
   if (transaction == null) {
       transaction = doFindOne(transactionXid);
       if (transaction != null) {
           putToCache(transaction);
       }
   }
   return transaction;
}

/**
* 获得事务从缓存中
*
* @param transactionXid 事务编号
* @return 事务
*/
protected Transaction findFromCache(TransactionXid transactionXid) {
   return transactionXidCompensableTransactionCache.getIfPresent(transactionXid);
}

/**
* 查询事务
*
* @param xid 事务编号
* @return 事务
*/
protected abstract Transaction doFindOne(Xid xid);
```

* 调用 `#findFromCache()` 方法，优先从缓存中获取事务。
* 调用 `#doFindOne()` 方法，缓存中事务不存在，从存储器中获取。获取到后，调用 `#putToCache()` 方法，添加事务到缓存中。
* `#doFindOne(...)` 为抽象方法，子类实现该方法，提供查询事务功能。

-------

**`#findAllUnmodifiedSince(...)`** 实现代码如下：

```Java
@Override
public List<Transaction> findAllUnmodifiedSince(Date date) {
   List<Transaction> transactions = doFindAllUnmodifiedSince(date);
   // 添加到缓存
   for (Transaction transaction : transactions) {
       putToCache(transaction);
   }
   return transactions;
}

/**
* 获取超过指定时间的事务集合
*
* @param date 指定时间
* @return 事务集合
*/
protected abstract List<Transaction> doFindAllUnmodifiedSince(Date date);
```

* 调用 `#findAllUnmodifiedSince(...)` 方法，从存储器获取超过指定时间的事务集合。调用 `#putToCache(...)` 方法，循环事务集合添加到缓存。
* `#doFindAllUnmodifiedSince(...)` 为抽象方法，子类实现该方法，提供获取超过指定时间的事务集合功能。

## 3.2 JDBC 事务存储器

`org.mengyun.tcctransaction.repository.JdbcTransactionRepository`，JDBC 事务存储器，通过 JDBC 驱动，将 Transaction 存储到 MySQL / Oracle / PostgreSQL / SQLServer 等关系数据库。实现代码如下：

```Java
public class JdbcTransactionRepository extends CachableTransactionRepository {

    /**
     * 领域
     */
    private String domain;
    /**
     * 表后缀
     */
    private String tbSuffix;
    /**
     * 数据源
     */
    private DataSource dataSource;
    /**
     * 序列化
     */
    private ObjectSerializer serializer = new JdkSerializationSerializer();
}
```

* `domain`，领域，或者也可以称为模块名，应用名，**用于唯一标识一个资源**。例如，Maven 模块 `xxx-order`，我们可以配置该属性为 `ORDER`。
* `tbSuffix`，表后缀。默认存储表名为 `TCC_TRANSACTION`，配置表名后，为 `TCC_TRANSACTION${tbSuffix}`。
* `dataSource`，存储数据的数据源。
* `serializer`，序列化。**当数据库里已经有数据的情况下，不要更换别的序列化，否则会导致反序列化报错。**建议：TCC-Transaction 存储时，新增字段，记录序列化的方式。

表结构如下：

```Java
CREATE TABLE `TCC_TRANSACTION` (
  `TRANSACTION_ID` int(11) NOT NULL AUTO_INCREMENT,
  `DOMAIN` varchar(100) DEFAULT NULL,
  `GLOBAL_TX_ID` varbinary(32) NOT NULL,
  `BRANCH_QUALIFIER` varbinary(32) NOT NULL,
  `CONTENT` varbinary(8000) DEFAULT NULL,
  `STATUS` int(11) DEFAULT NULL,
  `TRANSACTION_TYPE` int(11) DEFAULT NULL,
  `RETRIED_COUNT` int(11) DEFAULT NULL,
  `CREATE_TIME` datetime DEFAULT NULL,
  `LAST_UPDATE_TIME` datetime DEFAULT NULL,
  `VERSION` int(11) DEFAULT NULL,
  PRIMARY KEY (`TRANSACTION_ID`),
  UNIQUE KEY `UX_TX_BQ` (`GLOBAL_TX_ID`,`BRANCH_QUALIFIER`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

* `TRANSACTION_ID`，仅仅数据库自增，无实际用途。
* `CONTENT`，Transaction 序列化。

ps：点击[链接](https://github.com/YunaiV/tcc-transaction/blob/c164ff5ab29d31e08bc7061de5bc7403f3e40f1d/tcc-transaction-core/src/main/java/org/mengyun/tcctransaction/repository/JdbcTransactionRepository.java)查看 JdbcTransactionRepository 代码实现，已经添加完整中文注释。

## 3.3 Redis 事务存储器

`org.mengyun.tcctransaction.repository.RedisTransactionRepository`，Redis 事务存储器，将 Transaction 存储到 Redis。实现代码如下：

```Java
public class RedisTransactionRepository extends CachableTransactionRepository {

    /**
     * Jedis Pool
     */
    private JedisPool jedisPool;
    /**
     * key 前缀
     */
    private String keyPrefix = "TCC:";
    /**
     * 序列化
     */
    private ObjectSerializer serializer = new JdkSerializationSerializer();
    
}
```

* `keyPrefix`，key 前缀。类似 JdbcTransactionRepository 的 `domain` 属性。

一个事务存储到 Reids，使用 Redis 的数据结构为 [HASHES](https://redis.io/commands#hash)。

* key : 使用 `keyPrefix` + `xid`，实现代码如下：

    ```Java
    /**
    * 创建事务的 Redis Key
    *
    * @param keyPrefix key 前缀
    * @param xid 事务
    * @return Redis Key
    */
    public static byte[] getRedisKey(String keyPrefix, Xid xid) {
       byte[] prefix = keyPrefix.getBytes();
       byte[] globalTransactionId = xid.getGlobalTransactionId();
       byte[] branchQualifier = xid.getBranchQualifier();
       // 拼接 key
       byte[] key = new byte[prefix.length + globalTransactionId.length + branchQualifier.length];
       System.arraycopy(prefix, 0, key, 0, prefix.length);
       System.arraycopy(globalTransactionId, 0, key, prefix.length, globalTransactionId.length);
       System.arraycopy(branchQualifier, 0, key, prefix.length + globalTransactionId.length, branchQualifier.length);
       return key;
    }
    ```

* HASHES 的 key ：使用 `version`。
    * 添加和更新 Transaction 时，使用 Redis [HSETNX](https://redis.io/commands/hsetnx)，不存在当前版本的值时，进行设置，重而实现类似乐观锁的更新。
    * 读取 Transaction 时，使用 Redis [HGETALL](https://redis.io/commands/hgetall)，将 Transaction 所有 `version` 对应的值读取到内存后，取 `version` 值最大的对应的值。

* HASHES 的 value ：调用 `TransactionSerializer#serialize(...)` 方法，序列化 Transaction。实现代码如下：

    ```Java
    public static byte[] serialize(ObjectSerializer serializer, Transaction transaction) {
       Map<String, Object> map = new HashMap<String, Object>();
       map.put("GLOBAL_TX_ID", transaction.getXid().getGlobalTransactionId());
       map.put("BRANCH_QUALIFIER", transaction.getXid().getBranchQualifier());
       map.put("STATUS", transaction.getStatus().getId());
       map.put("TRANSACTION_TYPE", transaction.getTransactionType().getId());
       map.put("RETRIED_COUNT", transaction.getRetriedCount());
       map.put("CREATE_TIME", transaction.getCreateTime());
       map.put("LAST_UPDATE_TIME", transaction.getLastUpdateTime());
       map.put("VERSION", transaction.getVersion());
       // 序列化
       map.put("CONTENT", serializer.serialize(transaction));
       return serializer.serialize(map);
    }
    ```
    * TODO 为什么序列化两次

在实现 `#doFindAllUnmodifiedSince(date)` 方法，无法像数据库使用时间条件进行过滤，因此，加载所有事务后在内存中过滤。实现代码如下：

```Java
@Override
protected List<Transaction> doFindAllUnmodifiedSince(Date date) {
   // 获得所有事务
   List<Transaction> allTransactions = doFindAll();
   // 过滤时间
   List<Transaction> allUnmodifiedSince = new ArrayList<Transaction>();
   for (Transaction transaction : allTransactions) {
       if (transaction.getLastUpdateTime().compareTo(date) < 0) {
           allUnmodifiedSince.add(transaction);
       }
   }
   return allUnmodifiedSince;
}
```

ps：点击[链接](https://github.com/YunaiV/tcc-transaction/blob/c164ff5ab29d31e08bc7061de5bc7403f3e40f1d/tcc-transaction-core/src/main/java/org/mengyun/tcctransaction/repository/RedisTransactionRepository.java)查看 RedisTransactionRepository 代码实现，已经添加完整中文注释。

> FROM [《TCC-Transaction 官方文档 —— 使用指南1.2.x》](https://github.com/changmingxie/tcc-transaction/wiki/%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%971.2.x#%E9%85%8D%E7%BD%AEtcc-transaction)  
> 使用 RedisTransactionRepository 需要配置 Redis 服务器如下：  
> appendonly yes  
> appendfsync always

## 3.4 Zookeeper 事务存储器

`org.mengyun.tcctransaction.repository.ZooKeeperTransactionRepository`，Zookeeper 事务存储器，将 Transaction 存储到 Zookeeper。实现代码如下：

```Java
public class ZooKeeperTransactionRepository extends CachableTransactionRepository {

    /**
     * Zookeeper 服务器地址数组
     */
    private String zkServers;
    /**
     * Zookeeper 超时时间
     */
    private int zkTimeout;
    /**
     * TCC 存储 Zookeeper 根目录
     */
    private String zkRootPath = "/tcc";
    /**
     * Zookeeper 连接
     */
    private volatile ZooKeeper zk;
    /**
     * 序列化
     */
    private ObjectSerializer serializer = new JdkSerializationSerializer();
}
```
* `zkRootPath`，存储 Zookeeper 根目录，类似 JdbcTransactionRepository 的 `domain` 属性。


一个事务存储到 Zookeeper，使用 Zookeeper 的**持久数据节点**。

* path：`${zkRootPath}` + `/` + `${xid}`。实现代码如下：

    ```Java
    // ZooKeeperTransactionRepository.java
    private String getTxidPath(Xid xid) {
       return String.format("%s/%s", zkRootPath, xid);
    }
    
    // TransactionXid.java
    @Override
    public String toString() {
       StringBuilder stringBuilder = new StringBuilder();
       stringBuilder.append("globalTransactionId:").append(UUID.nameUUIDFromBytes(globalTransactionId).toString());
       stringBuilder.append(",").append("branchQualifier:").append(UUID.nameUUIDFromBytes(branchQualifier).toString());
       return stringBuilder.toString();
    }
    ```

* data：调用 `TransactionSerializer#serialize(...)` 方法，序列化 Transaction。
* version：使用 Zookeeper 数据节点自带版本功能。这里要注意下，Transaction 的版本从 1 开始，而 Zookeeper 数据节点版本从 0 开始。

ps：点击[链接](https://github.com/YunaiV/tcc-transaction/blob/c164ff5ab29d31e08bc7061de5bc7403f3e40f1d/tcc-transaction-core/src/main/java/org/mengyun/tcctransaction/repository/ZooKeeperTransactionRepository.java)查看 ZooKeeperTransactionRepository 代码实现，已经添加完整中文注释。

另外，在生产上暂时不建议使用 ZooKeeperTransactionRepository，原因有两点：

* 不支持 Zookeeper 安全认证。
* 使用 Zookeeper 时，未考虑断网重连等情况。

如果你要使用 Zookeeper 进行事务的存储，可以考虑使用 [Apache Curator](https://curator.apache.org/) 操作 Zookeeper，重写 ZooKeeperTransactionRepository 部分代码。

## 3.5 File 事务存储器

`org.mengyun.tcctransaction.repository.FileSystemTransactionRepository`，File 事务存储器，将 Transaction 存储到文件系统。

实现上和 ZooKeeperTransactionRepository，区别主要在于不支持乐观锁更新。有兴趣的同学点击[链接](https://github.com/YunaiV/tcc-transaction/blob/c164ff5ab29d31e08bc7061de5bc7403f3e40f1d/tcc-transaction-core/src/main/java/org/mengyun/tcctransaction/repository/FileSystemTransactionRepository.java)查看，这里就不拓展开来。

另外，在生产上不建议使用 FileSystemTransactionRepository，因为不支持多节点共享。用分布式存储挂载文件另说，当然还是不建议，因为不支持乐观锁并发更新。

# 666. 彩蛋

这篇略( 超 )微( 级 )水更，哈哈哈，为[《TCC-Transaction 源码分析 —— 事务恢复》](http://www.iocoder.cn/TCC-Transaction/transaction-recover/?self)做铺垫啦。

使用 RedisTransactionRepository 和 ZooKeeperTransactionRepository 存储事务还是 Get 蛮多点的。

![](http://www.iocoder.cn/images/TCC-Transaction/2018_02_15/02.png)

胖友，分享一个朋友圈可好？

