title: Elastic-Job-Lite 源码分析 —— 注册中心
date: 2017-09-30
tags:
categories: Elastic-Job-Lite
permalink: Elastic-Job/reg-center-zookeeper

-------

**本文基于 Elastic-Job V2.1.5 版本分享**

- [1. 概述](#)
- [1. 概述](#)
- [2. 基于 Zookeeper 注册中心](#)
	- [2.1 初始化](#)
	- [2.2 缓存](#)
	- [2.3 关闭](#)
	- [2.4 获得数据](#)
	- [2.5 获得注册子节点](#)
	- [2.6 存储注册数据](#)
	- [2.7 存储顺序注册数据](#)
	- [2.8 移除注册数据](#)
	- [2.9 获取注册中心当前时间](#)
	- [2.10 注册中心异常处理器](#)
- [3. 作业节点数据访问类](#)
	- [3.1 在主节点执行操作](#)
	- [3.2 在事务中执行操作](#)
- [666. 彩蛋](#)

-------

![](http://www.yunai.me/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

# 1. 概述

本文主要分享 **Elastic-Job-Lite 注册中心**。

涉及到主要类的类图如下( [打开大图](http://www.yunai.me/images/Elastic-Job/2017_09_30/01.png) )：

![](http://www.yunai.me/images/Elastic-Job/2017_09_30/01.png)

* **黄色**的类在 `elastic-job-common-core` 项目里，为 Elastic-Job-Lite、Elastic-Job-Cloud **公用**注册中心类。
* 作业节点数据访问类( JobNodeStorage )的**在主节点执行操作**、**在事务中执行操作**两个方法和注册中心**协调分布式服务**有关系，从[《Elastic-Job-Lite 源码解析 —— 作业数据存储》](http://www.yunai.me/Elastic-Job/job-storage/?self)摘出来，放本文解析。

> 你行好事会因为得到赞赏而愉悦  
> 同理，开源项目贡献者会因为 Star 而更加有动力  
> 为 Elastic-Job 点赞！[传送门](https://github.com/dangdangdotcom/elastic-job/stargazers)

# 2. 基于 Zookeeper 注册中心
 
ZookeeperRegistryCenter，基于 Zookeeper 注册中心。从上面的类图可以看到，ZookeeperRegistryCenter 实现 CoordinatorRegistryCenter 接口，CoordinatorRegistryCenter 继承 RegistryCenter 接口。

* RegistryCenter，注册中心，定义了简单的增删改查注册数据和查询时间的接口方法。
* CoordinatorRegistryCenter，用于协调分布式服务的注册中心，定义了持久节点、临时节点、持久顺序节点、临时顺序节点等**目录服务**接口方法，隐性的要求提供**事务**、**分布式锁**、**数据订阅**等特性。
  
ZookeeperRegistryCenter 使用 [Apache Curator](https://curator.apache.org/) 进行 Zookeeper 注册中心。
 
## 2.1 初始化

ZookeeperConfiguration，基于 Zookeeper 的注册中心配置，注释完整，点击[链接](https://github.com/dangdangdotcom/elastic-job/blob/7dc099541a16de49f024fc59e46377a726be7f6b/elastic-job-common/elastic-job-common-core/src/main/java/com/dangdang/ddframe/job/reg/zookeeper/ZookeeperConfiguration.java)直接查看。

```Java
@Override
public void init() {
   log.debug("Elastic job: zookeeper registry center init, server lists is: {}.", zkConfig.getServerLists());
   CuratorFrameworkFactory.Builder builder = CuratorFrameworkFactory.builder()
           .connectString(zkConfig.getServerLists())
           .retryPolicy(new ExponentialBackoffRetry(zkConfig.getBaseSleepTimeMilliseconds(), zkConfig.getMaxRetries(), zkConfig.getMaxSleepTimeMilliseconds()))
           .namespace(zkConfig.getNamespace()); // 命名空间
   if (0 != zkConfig.getSessionTimeoutMilliseconds()) {
       builder.sessionTimeoutMs(zkConfig.getSessionTimeoutMilliseconds()); // 会话超时时间，默认 60 * 1000 毫秒
   }
   if (0 != zkConfig.getConnectionTimeoutMilliseconds()) {
       builder.connectionTimeoutMs(zkConfig.getConnectionTimeoutMilliseconds()); // 连接超时时间，默认 15 * 1000 毫秒
   }
   // 认证
   if (!Strings.isNullOrEmpty(zkConfig.getDigest())) {
       builder.authorization("digest", zkConfig.getDigest().getBytes(Charsets.UTF_8))
               .aclProvider(new ACLProvider() {
               
                   @Override
                   public List<ACL> getDefaultAcl() {
                       return ZooDefs.Ids.CREATOR_ALL_ACL;
                   }
               
                   @Override
                   public List<ACL> getAclForPath(final String path) {
                       return ZooDefs.Ids.CREATOR_ALL_ACL;
                   }
               });
   }
   client = builder.build();
   client.start();
   // 连接 Zookeeper
   try {
       if (!client.blockUntilConnected(zkConfig.getMaxSleepTimeMilliseconds() * zkConfig.getMaxRetries(), TimeUnit.MILLISECONDS)) {
           client.close();
           throw new KeeperException.OperationTimeoutException();
       }
   } catch (final Exception ex) {
       RegExceptionHandler.handleException(ex);
   }
}
```

* ExponentialBackoffRetry，当 Zookeeper 失去链接后重新连接的**一种**策略：动态计算每次计算重连的间隔，时间间隔 = `baseSleepTimeMs * Math.max(1, random.nextInt(1 << (retryCount + 1)))`。如果对其它重连策略感兴趣，可以看 [RetryPolicy](https://github.com/apache/curator/blob/abaabb5f65c2161f77527165a15d2420f6c88219/curator-client/src/main/java/org/apache/curator/RetryPolicy.java) 的实现类，本文就不展开了。
* **相同**的作业集群使用**相同**的 Zookeeper 命名空间( `ZookeeperConfiguration.namespace` )。

## 2.2 缓存

通过 Curator TreeCache 实现监控整个树( Zookeeper目录 )的数据订阅和缓存，包括节点的状态，子节点的状态。

**初始化作业缓存**

作业初始化注册时，初始化缓存。

```JAVA
// JobRegistry.java
public void registerJob(final String jobName, final JobScheduleController jobScheduleController, final CoordinatorRegistryCenter regCenter) {
   schedulerMap.put(jobName, jobScheduleController);
   regCenterMap.put(jobName, regCenter);
   // 添加注册中心缓存
   regCenter.addCacheData("/" + jobName);
}
    
// ZookeeperRegistryCenter.java
/**
* 缓存
* key：/作业名/
*/
private final Map<String, TreeCache> caches = new HashMap<>();
```

**作业服务订阅数据**

每个不同的服务，都会订阅数据实现功能逻辑。在后续不同服务的文章，我们会详细解析。🙂

```Java
public void addDataListener(final TreeCacheListener listener) {
   TreeCache cache = (TreeCache) regCenter.getRawCache("/" + jobName);
   cache.getListenable().addListener(listener);
}
```

**关闭作业缓存**
 
```Java   
@Override
public void evictCacheData(final String cachePath) {
   TreeCache cache = caches.remove(cachePath + "/");
   if (null != cache) {
       cache.close();
   }
}
```

对 Curator TreeCache 感兴趣的同学，可以点击[链接](http://colobu.com/2014/12/15/zookeeper-recipes-by-example-5/)继续了解。

## 2.3 关闭

```Java
public void close() {
   for (Entry<String, TreeCache> each : caches.entrySet()) {
       each.getValue().close();
   }
   waitForCacheClose();
   CloseableUtils.closeQuietly(client);
}
    
/* TODO 等待500ms, cache先关闭再关闭client, 否则会抛异常
* 因为异步处理, 可能会导致client先关闭而cache还未关闭结束.
* 等待Curator新版本解决这个bug.
* BUG地址：https://issues.apache.org/jira/browse/CURATOR-157
*/
private void waitForCacheClose() {
   try {
       Thread.sleep(500L);
   } catch (final InterruptedException ex) {
       Thread.currentThread().interrupt();
   }
}
```

## 2.4 获得数据
  
```Java
@Override
public String get(final String key) {
   TreeCache cache = findTreeCache(key); // 获取缓存
   if (null == cache) {
       return getDirectly(key);
   }
   ChildData resultInCache = cache.getCurrentData(key); // 缓存中获取 value
   if (null != resultInCache) {
       return null == resultInCache.getData() ? null : new String(resultInCache.getData(), Charsets.UTF_8);
   }
   return getDirectly(key);
}

@Override
public String getDirectly(final String key) {
   try {
       return new String(client.getData().forPath(key), Charsets.UTF_8);
   } catch (final Exception ex) {
       RegExceptionHandler.handleException(ex);
       return null;
   }
}
```

* `#get(...)` 先从 **TreeCache缓存** 获取，后从 Zookeeper 获取。
* `#getDirectly(...)` **直接**从 Zookeeper 获取。
* `#findTreeCache(...)` 代码如下：

    ```Java
    private TreeCache findTreeCache(final String key) {
       for (Entry<String, TreeCache> entry : caches.entrySet()) {
           if (key.startsWith(entry.getKey())) {
               return entry.getValue();
           }
       }
       return null;
    }
    ```

## 2.5 获得注册子节点

**获取子节点名称集合(降序)**

```Java
@Override
public List<String> getChildrenKeys(final String key) {
   try {
       List<String> result = client.getChildren().forPath(key);
       Collections.sort(result, new Comparator<String>() {
           
           @Override
           public int compare(final String o1, final String o2) {
               return o2.compareTo(o1);
           }
       });
       return result;
   } catch (final Exception ex) {
       RegExceptionHandler.handleException(ex);
       return Collections.emptyList();
   }
}
```

**获取子节点数量**    

```Java
@Override
public int getNumChildren(final String key) {
   try {
       Stat stat = client.checkExists().forPath(key);
       if (null != stat) {
           return stat.getNumChildren();
       }
   } catch (final Exception ex) {
       RegExceptionHandler.handleException(ex);
   }
   return 0;
}
```
    
## 2.6 存储注册数据
    
```Java
@Override
public void persist(final String key, final String value) {
   try {
       if (!isExisted(key)) {
           client.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath(key, value.getBytes(Charsets.UTF_8));
       } else {
           update(key, value);
       }
   } catch (final Exception ex) {
       RegExceptionHandler.handleException(ex);
   }
}

@Override
public void persistEphemeral(final String key, final String value) {
   try {
       if (isExisted(key)) {
           client.delete().deletingChildrenIfNeeded().forPath(key);
       }
       client.create().creatingParentsIfNeeded().withMode(CreateMode.EPHEMERAL).forPath(key, value.getBytes(Charsets.UTF_8));
   } catch (final Exception ex) {
       RegExceptionHandler.handleException(ex);
   }
}
```

* `#persist(...)` 存储**持久**节点数据。逻辑等价于 insertOrUpdate 操作。
* `persistEphemeral(...)` 存储**临时**节点数据。节点类型无法变更，因此如果数据已存在，需要先进行删除。
* `#isExisted(...)`、`#update(...)` 代码如下：

    ```Java
    @Override
    public boolean isExisted(final String key) {
       try {
           return null != client.checkExists().forPath(key);
       } catch (final Exception ex) {
           RegExceptionHandler.handleException(ex);
           return false;
       }
    }
       
    @Override
    public void update(final String key, final String value) {
       try {
           client.inTransaction().check().forPath(key).and().setData().forPath(key, value.getBytes(Charsets.UTF_8)).and().commit();
       } catch (final Exception ex) {
           RegExceptionHandler.handleException(ex);
       }
    }
    ```
    * `#update(...)` 使用**事务**校验键( key )存在才进行更新。

## 2.7 存储顺序注册数据

实现逻辑和**存储注册数据**类似。Elastic-Job 未使用该方法，跳过。
 
## 2.8 移除注册数据

```Java
@Override
public void remove(final String key) {
   try {
       client.delete().deletingChildrenIfNeeded().forPath(key);
   } catch (final Exception ex) {
       RegExceptionHandler.handleException(ex);
   }
}
```
   
## 2.9 获取注册中心当前时间
    
```Java
@Override
public long getRegistryCenterTime(final String key) {
   long result = 0L;
   try {
       persist(key, "");
       result = client.checkExists().forPath(key).getMtime();
   } catch (final Exception ex) {
       RegExceptionHandler.handleException(ex);
   }
   Preconditions.checkState(0L != result, "Cannot get registry center time.");
   return result;
}
```

* 通过更新节点，获得该节点的最后更新时间( `mtime` )获得 Zookeeper 的时间。six six six。

## 2.10 注册中心异常处理器

RegExceptionHandler，注册中心异常处理器。在上面的操作 Zookeeper 发生异常时，都会调用 `RegExceptionHandler.handleException(...)` 处理异常：

```Java
public static void handleException(final Exception cause) {
   if (null == cause) {
       return;
   }
   if (isIgnoredException(cause) || null != cause.getCause() && isIgnoredException(cause.getCause())) {
       log.debug("Elastic job: ignored exception for: {}", cause.getMessage());
   } else if (cause instanceof InterruptedException) {
       Thread.currentThread().interrupt();
   } else {
       throw new RegException(cause);
   }
}
    
private static boolean isIgnoredException(final Throwable cause) {
   return cause instanceof ConnectionLossException || cause instanceof NoNodeException || cause instanceof NodeExistsException;
}
```

* 部分异常会被无视，仅打印异常。例如调用 `#getDirectly(...)` 获得注册数据时，可能节点不存在，抛出 NodeExistsException，这种异常可以无视。

# 3. 作业节点数据访问类

JobNodeStorage，作业节点数据访问类。

## 3.1 在主节点执行操作

```Java
// JobNodeStorage.java
/**
* 在主节点执行操作.
* 
* @param latchNode 分布式锁使用的节点，例如：leader/election/latch
* @param callback 执行操作的回调
*/
public void executeInLeader(final String latchNode, final LeaderExecutionCallback callback) {
   try (LeaderLatch latch = new LeaderLatch(getClient(), jobNodePath.getFullPath(latchNode))) {
       latch.start();
       latch.await();
       callback.execute();
   } catch (final Exception ex) {
       handleException(ex);
   }
}
```

[Apache Curator](https://curator.apache.org/) 使用 Zookeeper 实现了两种分布式锁，LeaderLatch 是其中的一种。使用**一个** Zookeeper 节点路径创建**一个** LeaderLatch，`#start()` 后，调用 `#await()` 等待拿到这把**锁**。如果有多个线程执行了**相同节点路径**的 LeaderLatch 的 `#await()` 后，同一时刻有且仅有一个线程可以继续执行，其他线程需要等待。当该线程释放( `LeaderLatch#close()` )后，下一个线程可以拿到该**锁**继续执行。用 Java 并发包 Lock 举例子：

``` Java
public void executeInLeader(Lock lock) {
    try {
        lock.lock();
        // doSomething();
    } finally {
        lock.unlock();
    }
}
```

[《官方文档 —— LeaderLatch》](https://github.com/Netflix/curator/wiki/Leader-Latch)，有兴趣的同学可以看看。在[《Elastic-Job-Lite 源码解析 —— 主节点选举》](http://www.yunai.me/Elastic-Job/election/?self)中，我们会看到 `#executeInLeader(...)` 的使用。

另一种分布式锁实现，[《官方文档 —— LeaderElection》](https://github.com/Netflix/curator/wiki/Leader-Election)，有兴趣也可以看看。在 Elastic-Job-Cloud 中使用到了，后续进行解析。

## 3.2 在事务中执行操作

```Java
// JobNodeStorage.java
public void executeInTransaction(final TransactionExecutionCallback callback) {
   try {
       CuratorTransactionFinal curatorTransactionFinal = getClient().inTransaction().check().forPath("/").and();
       callback.execute(curatorTransactionFinal);
       curatorTransactionFinal.commit();
   } catch (final Exception ex) {
       RegExceptionHandler.handleException(ex);
   }
}
```

* 开启事务，执行 TransactionExecutionCallback 回调逻辑，提交事务。

# 666. 彩蛋
   
旁白君：煞笔芋道君，又在水更  
芋道君：人艰不拆，好不好。

道友，赶紧上车，分享一波朋友圈！


