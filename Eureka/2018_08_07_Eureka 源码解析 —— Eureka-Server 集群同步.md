title: Eureka 源码解析 —— Eureka-Server 集群同步
date: 2018-08-07
tags:
categories: Eureka
permalink: Eureka/server-cluster
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484882&idx=2&sn=4c0df739dac5912ff6b8d31f0573cc23&chksm=fa497a63cd3ef3759b4e205e1744f31b7ca07763c5b57534ed98792acce4ea0b85f5747f40c8#rd

---

摘要: 原创出处 http://www.iocoder.cn/Eureka/server-cluster/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Eureka 1.8.X 版本**

- [1. 概述](http://www.iocoder.cn/Eureka/server-cluster/)
- [2. 集群节点初始化与更新](http://www.iocoder.cn/Eureka/server-cluster/)
  - [2.1 集群节点启动](http://www.iocoder.cn/Eureka/server-cluster/)
  - [2.2 更新集群节点信息](http://www.iocoder.cn/Eureka/server-cluster/)
  - [2.3 集群节点](http://www.iocoder.cn/Eureka/server-cluster/)
- [3. 获取初始注册信息](http://www.iocoder.cn/Eureka/server-cluster/)
- [4. 同步注册信息](http://www.iocoder.cn/Eureka/server-cluster/)
  - [4.1 同步操作类型](http://www.iocoder.cn/Eureka/server-cluster/)
  - [4.2 发起 Eureka-Server 同步操作](http://www.iocoder.cn/Eureka/server-cluster/)
  - [4.3 接收 Eureka-Server 同步操作](http://www.iocoder.cn/Eureka/server-cluster/)
  - [4.4 处理 Eureka-Server 同步结果](http://www.iocoder.cn/Eureka/server-cluster/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2018_05_18.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

---

# 1. 概述

本文主要分享 **Eureka-Server 集群同步注册信息**。

Eureka-Server 集群如下图：

![](http://www.iocoder.cn/images/Eureka/2018_08_07/01.png)

* Eureka-Server 集群不区分**主从节点**或者 **Primary & Secondary 节点**，所有节点**相同角色( 也就是没有角色 )，完全对等**。
* Eureka-Client 可以向**任意** Eureka-Client 发起任意**读写**操作，Eureka-Server 将操作复制到另外的 Eureka-Server 以达到**最终一致性**。注意，Eureka-Server 是选择了 AP 的组件。

Eureka-Server 可以使用直接配置所有节点的服务地址，或者基于 DNS 配置。推荐阅读：[《Spring Cloud构建微服务架构（六）高可用服务注册中心》](http://blog.didispace.com/springcloud6/?from=http://www.iocoder.cn) 。

本文主要类在 `com.netflix.eureka.cluster` 包下。

OK，让我们开始愉快的遨游在代码的海洋。

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



ps ：**注意**，本文提到的**同步**，准确来说是**复制( Replication )**。

# 2. 集群节点初始化与更新

`com.netflix.eureka.cluster.PeerEurekaNodes` ，Eureka-Server 集群节点集合 。构造方法如下 ：

```Java
public class PeerEurekaNodes {

    private static final Logger logger = LoggerFactory.getLogger(PeerEurekaNodes.class);

    /**
     * 应用实例注册表
     */
    protected final PeerAwareInstanceRegistry registry;
    /**
     * Eureka-Server 配置
     */
    protected final EurekaServerConfig serverConfig;
    /**
     * Eureka-Client 配置
     */
    protected final EurekaClientConfig clientConfig;
    /**
     * Eureka-Server 编解码
     */
    protected final ServerCodecs serverCodecs;
    /**
     * 应用实例信息管理器
     */
    private final ApplicationInfoManager applicationInfoManager;

    /**
     * Eureka-Server 集群节点数组
     */
    private volatile List<PeerEurekaNode> peerEurekaNodes = Collections.emptyList();
    /**
     * Eureka-Server 服务地址数组
     */
    private volatile Set<String> peerEurekaNodeUrls = Collections.emptySet();

    /**
     * 定时任务服务
     */
    private ScheduledExecutorService taskExecutor;

    @Inject
    public PeerEurekaNodes(
            PeerAwareInstanceRegistry registry,
            EurekaServerConfig serverConfig,
            EurekaClientConfig clientConfig,
            ServerCodecs serverCodecs,
            ApplicationInfoManager applicationInfoManager) {
        this.registry = registry;
        this.serverConfig = serverConfig;
        this.clientConfig = clientConfig;
        this.serverCodecs = serverCodecs;
        this.applicationInfoManager = applicationInfoManager;
    }
}
```

* `peerEurekaNodes`, `peerEurekaNodeUrls`, `taskExecutor` 属性，在构造方法中**未设置和初始化**，而是在 `PeerEurekaNodes#start()` 方法，设置和初始化，下文我们会解析这个方法。
* Eureka-Server 在初始化时，调用 `EurekaBootStrap#getPeerEurekaNodes(...)` 方法，创建 PeerEurekaNodes ，点击 [链接](https://github.com/YunaiV/eureka/blob/94a4a1ebd35a3350893369a05757c01b2534b702/eureka-core/src/main/java/com/netflix/eureka/EurekaBootStrap.java#L245) 查看该方法的实现。

## 2.1 集群节点启动

调用 `PeerEurekaNodes#start()` 方法，集群节点启动，主要完成两个逻辑：

* 初始化集群节点信息
* 初始化固定周期( 默认：10 分钟，可配置 )更新集群节点信息的任务

代码如下：

```Java
  1: public void start() {
  2:     // 创建 定时任务服务
  3:     taskExecutor = Executors.newSingleThreadScheduledExecutor(
  4:             new ThreadFactory() {
  5:                 @Override
  6:                 public Thread newThread(Runnable r) {
  7:                     Thread thread = new Thread(r, "Eureka-PeerNodesUpdater");
  8:                     thread.setDaemon(true);
  9:                     return thread;
 10:                 }
 11:             }
 12:     );
 13:     try {
 14:         // 初始化 集群节点信息
 15:         updatePeerEurekaNodes(resolvePeerUrls());
 16:         // 初始化 初始化固定周期更新集群节点信息的任务
 17:         Runnable peersUpdateTask = new Runnable() {
 18:             @Override
 19:             public void run() {
 20:                 try {
 21:                     updatePeerEurekaNodes(resolvePeerUrls());
 22:                 } catch (Throwable e) {
 23:                     logger.error("Cannot update the replica Nodes", e);
 24:                 }
 25: 
 26:             }
 27:         };
 28:         taskExecutor.scheduleWithFixedDelay(
 29:                 peersUpdateTask,
 30:                 serverConfig.getPeerEurekaNodesUpdateIntervalMs(),
 31:                 serverConfig.getPeerEurekaNodesUpdateIntervalMs(),
 32:                 TimeUnit.MILLISECONDS
 33:         );
 34:     } catch (Exception e) {
 35:         throw new IllegalStateException(e);
 36:     }
 37:     // 打印 集群节点信息
 38:     for (PeerEurekaNode node : peerEurekaNodes) {
 39:         logger.info("Replica node URL:  " + node.getServiceUrl());
 40:     }
 41: }
```

* 第 15 行 && 第 21 行 ：调用 `#updatePeerEurekaNodes()` 方法，更新集群节点信息。

## 2.2 更新集群节点信息

调用 `#resolvePeerUrls()` 方法，获得 Eureka-Server 集群服务地址数组，代码如下：

```Java
  1: protected List<String> resolvePeerUrls() {
  2:     // 获得 Eureka-Server 集群服务地址数组
  3:     InstanceInfo myInfo = applicationInfoManager.getInfo();
  4:     String zone = InstanceInfo.getZone(clientConfig.getAvailabilityZones(clientConfig.getRegion()), myInfo);
  5:     List<String> replicaUrls = EndpointUtils.getDiscoveryServiceUrls(clientConfig, zone, new EndpointUtils.InstanceInfoBasedUrlRandomizer(myInfo));
  6: 
  7:     // 移除自己（避免向自己同步）
  8:     int idx = 0;
  9:     while (idx < replicaUrls.size()) {
 10:         if (isThisMyUrl(replicaUrls.get(idx))) {
 11:             replicaUrls.remove(idx);
 12:         } else {
 13:             idx++;
 14:         }
 15:     }
 16:     return replicaUrls;
 17: }
```

* 第 2 至 5 行 ：获得 Eureka-Server 集群服务地址数组。`EndpointUtils#getDiscoveryServiceUrls(...)` 方法，逻辑与 [《Eureka 源码解析 —— EndPoint 与 解析器》「3.4 ConfigClusterResolver」](http://www.iocoder.cn/Eureka/end-point-and-resolver/?self) 基本类似。EndpointUtils 正在逐步，猜测未来这里会替换。
* 第 7 至 15 行 ：移除自身节点，避免向自己同步。

-------

调用 `#updatePeerEurekaNodes()` 方法，更新集群节点信息，主要完成两部分逻辑：

* 添加新增的集群节点
* 关闭删除的集群节点

代码如下：

```Java
  1: protected void updatePeerEurekaNodes(List<String> newPeerUrls) {
  2:     if (newPeerUrls.isEmpty()) {
  3:         logger.warn("The replica size seems to be empty. Check the route 53 DNS Registry");
  4:         return;
  5:     }
  6: 
  7:     // 计算 新增的集群节点地址
  8:     Set<String> toShutdown = new HashSet<>(peerEurekaNodeUrls);
  9:     toShutdown.removeAll(newPeerUrls);
 10: 
 11:     // 计算 删除的集群节点地址
 12:     Set<String> toAdd = new HashSet<>(newPeerUrls);
 13:     toAdd.removeAll(peerEurekaNodeUrls);
 14: 
 15:     if (toShutdown.isEmpty() && toAdd.isEmpty()) { // No change
 16:         return;
 17:     }
 18: 
 19:     // 关闭删除的集群节点
 20:     // Remove peers no long available
 21:     List<PeerEurekaNode> newNodeList = new ArrayList<>(peerEurekaNodes);
 22:     if (!toShutdown.isEmpty()) {
 23:         logger.info("Removing no longer available peer nodes {}", toShutdown);
 24:         int i = 0;
 25:         while (i < newNodeList.size()) {
 26:             PeerEurekaNode eurekaNode = newNodeList.get(i);
 27:             if (toShutdown.contains(eurekaNode.getServiceUrl())) {
 28:                 newNodeList.remove(i);
 29:                 eurekaNode.shutDown(); // 关闭
 30:             } else {
 31:                 i++;
 32:             }
 33:         }
 34:     }
 35: 
 36:     // 添加新增的集群节点
 37:     // Add new peers
 38:     if (!toAdd.isEmpty()) {
 39:         logger.info("Adding new peer nodes {}", toAdd);
 40:         for (String peerUrl : toAdd) {
 41:             newNodeList.add(createPeerEurekaNode(peerUrl));
 42:         }
 43:     }
 44: 
 45:     // 赋值
 46:     this.peerEurekaNodes = newNodeList;
 47:     this.peerEurekaNodeUrls = new HashSet<>(newPeerUrls);
 48: }
```

* 第 7 至 9 行 ：**计算**新增的集群节点地址。
* 第 11 至 13 行 ：**计算**删除的集群节点地址。
* 第 19 至 34 行 ：**关闭**删除的集群节点。
* 第 36 至 43 行 ：**添加**新增的集群节点。调用 `#createPeerEurekaNode(peerUrl)` 方法，创建集群节点，代码如下：

    ```Java
      1: protected PeerEurekaNode createPeerEurekaNode(String peerEurekaNodeUrl) {
      2:     HttpReplicationClient replicationClient = JerseyReplicationClient.createReplicationClient(serverConfig, serverCodecs, peerEurekaNodeUrl);
      3:     String targetHost = hostFromUrl(peerEurekaNodeUrl);
      4:     if (targetHost == null) {
      5:         targetHost = "host";
      6:     }
      7:     return new PeerEurekaNode(registry, targetHost, peerEurekaNodeUrl, replicationClient, serverConfig);
      8: }
    ```
    * 第 2 行 ：创建 Eureka-Server 集群通信客户端，在 [《Eureka 源码解析 —— 网络通信》「4.2 JerseyReplicationClient」](http://www.iocoder.cn/Eureka/transport/?self) 有详细解析。
    * 第 7 行 ：创建 PeerEurekaNode ，在 [「2.3 PeerEurekaNode」](#) 有详细解析。

## 2.3 集群节点

`com.netflix.eureka.cluster.PeerEurekaNode` ，单个集群节点。

点击 [链接](https://github.com/YunaiV/eureka/blob/fcc9027a197783a23e7cb72ad0f617b7dc63d221/eureka-core/src/main/java/com/netflix/eureka/cluster/PeerEurekaNode.java#L111) 查看**构造方法**

* 第 129 行 ：创建 ReplicationTaskProcessor 。在 [「4.1.2 同步操作任务处理器」](#) 详细解析
* 第 131 至 140 行 ：创建**批量任务**分发器，在 [《Eureka 源码解析 —— 任务批处理》](http://www.iocoder.cn/Eureka/batch-tasks/?self) 有详细解析。
* 第 142 至 151 行 ：创建**单任务**分发器，用于 Eureka-Server 向亚马逊 AWS 的 ASG ( `Autoscaling Group` ) 同步状态。暂时跳过。

# 3. 获取初始注册信息

Eureka-Server 启动时，调用 `PeerAwareInstanceRegistryImpl#syncUp()` 方法，从集群的一个 Eureka-Server 节点获取**初始**注册信息，代码如下：

```Java
  1: @Override
  2: public int syncUp() {
  3:     // Copy entire entry from neighboring DS node
  4:     int count = 0;
  5: 
  6:     for (int i = 0; ((i < serverConfig.getRegistrySyncRetries()) && (count == 0)); i++) {
  7:         // 未读取到注册信息，sleep 等待
  8:         if (i > 0) {
  9:             try {
 10:                 Thread.sleep(serverConfig.getRegistrySyncRetryWaitMs());
 11:             } catch (InterruptedException e) {
 12:                 logger.warn("Interrupted during registry transfer..");
 13:                 break;
 14:             }
 15:         }
 16: 
 17:         // 获取注册信息
 18:         Applications apps = eurekaClient.getApplications();
 19:         for (Application app : apps.getRegisteredApplications()) {
 20:             for (InstanceInfo instance : app.getInstances()) {
 21:                 try {
 22:                     if (isRegisterable(instance)) { // 判断是否能够注册
 23:                         register(instance, instance.getLeaseInfo().getDurationInSecs(), true); // 注册
 24:                         count++;
 25:                     }
 26:                 } catch (Throwable t) {
 27:                     logger.error("During DS init copy", t);
 28:                 }
 29:             }
 30:         }
 31:     }
 32:     return count;
 33: }
```
* 第 7 至 15 行 ：未获取到注册信息，`sleep` 等待再次重试。
* 第 17 至 30 行 ：获取注册信息，若获取到，注册到自身节点。
    * 第 22 行 ：判断应用实例是否能够注册到自身节点。主要用于亚马逊 AWS 环境下的判断，若非部署在亚马逊里，都返回 `true` 。点击 [链接](https://github.com/YunaiV/eureka/blob/94a4a1ebd35a3350893369a05757c01b2534b702/eureka-core/src/main/java/com/netflix/eureka/registry/PeerAwareInstanceRegistryImpl.java#L593) 查看实现。
    * 第 23 行 ：调用 `#register()` 方法，注册应用实例到自身节点。在 [《Eureka 源码解析 —— 应用实例注册发现（一）之注册》](http://www.iocoder.cn/Eureka/instance-registry-register/) 有详细解析。

-------

若调用 `#syncUp()` 方法，未获取到应用实例，则 Eureka-Server 会有一段时间( 默认：5 分钟，可配 )不允许被 Eureka-Client 获取注册信息，避免影响 Eureka-Client 。

* 标记 Eureka-Server 启动时，未获取到应用实例，代码如下：

    ```Java
    // PeerAwareInstanceRegistryImpl.java
    
    private boolean peerInstancesTransferEmptyOnStartup = true;
    
    public void openForTraffic(ApplicationInfoManager applicationInfoManager, int count) {
        // ... 省略其他代码
        if (count > 0) {
            this.peerInstancesTransferEmptyOnStartup = false;
        }
        // ... 省略其他代码
    }
    ```
    
* 判断 Eureka-Server 是否允许被 Eureka-Client 获取注册信息，代码如下：

    ```Java
    // PeerAwareInstanceRegistryImpl.java
    public boolean shouldAllowAccess(boolean remoteRegionRequired) {
       if (this.peerInstancesTransferEmptyOnStartup) {
           // 设置启动时间
           this.startupTime = System.currentTimeMillis();
           if (!(System.currentTimeMillis() > this.startupTime + serverConfig.getWaitTimeInMsWhenSyncEmpty())) {
               return false;
           }
       }
       // ... 省略其他代码
       return true;
    }
    ```

# 4. 同步注册信息

Eureka-Server 集群同步注册信息如下图：

![](http://www.iocoder.cn/images/Eureka/2018_08_07/02.png)

* Eureka-Server 接收到 Eureka-Client 的 Register、Heartbeat、Cancel、StatusUpdate、DeleteStatusOverride 操作，固定间隔( 默认值 ：500 毫秒，可配 )向 Eureka-Server 集群内其他节点同步( **准实时，非实时** )。

## 4.1 同步操作类型

`com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl.Action` ，同步操作类型，代码如下：

```Java
public enum Action {
   Heartbeat, Register, Cancel, StatusUpdate, DeleteStatusOverride;

   // ... 省略监控相关属性
}
```

* Register ，在 [《Eureka 源码解析 —— 应用实例注册发现（一）之注册》](http://www.iocoder.cn/Eureka/instance-registry-register/?self) 有详细解析
* Heartbeat ，在 [《Eureka 源码解析 —— 应用实例注册发现（二）之续租》](http://www.iocoder.cn/Eureka/instance-registry-renew/?self) 有详细解析
* Cancel ，在 [《Eureka 源码解析 —— 应用实例注册发现（三）之下线》](http://www.iocoder.cn/Eureka/instance-registry-cancel/?self) 有详细解析
* StatusUpdate ，在 [《Eureka 源码解析 —— 应用实例注册发现（八）之覆盖状态》](http://www.iocoder.cn/Eureka/instance-registry-override-status/?self) 有详细解析
* DeleteStatusOverride ，在 [《Eureka 源码解析 —— 应用实例注册发现（八）之覆盖状态》](http://www.iocoder.cn/Eureka/instance-registry-override-status/?self) 有详细解析

## 4.2 发起 Eureka-Server 同步操作

Eureka-Server 在完成 Eureka-Client 发起的上述操作在**自身节点的执行后**，向集群内其他 Eureka-Server 发起同步操作。以 Register 操作举例子，代码如下：

```Java
// PeerAwareInstanceRegistryImpl.java
public void register(final InstanceInfo info, final boolean isReplication) {
   // 租约过期时间
   int leaseDuration = Lease.DEFAULT_DURATION_IN_SECS;
   if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() > 0) {
       leaseDuration = info.getLeaseInfo().getDurationInSecs();
   }
   // 注册应用实例信息
   super.register(info, leaseDuration, isReplication);
   // Eureka-Server 复制
   replicateToPeers(Action.Register, info.getAppName(), info.getId(), info, null, isReplication);
}
```

* 最后一行，调用 `#replicateToPeers(...)` 方法，传递**对应的同步操作类型**，发起同步操作。

-------

`#replicateToPeers(...)` 方法，代码如下：

```Java
  1: private void replicateToPeers(Action action, String appName, String id,
  2:                               InstanceInfo info /* optional */,
  3:                               InstanceStatus newStatus /* optional */, boolean isReplication) {
  4:     Stopwatch tracer = action.getTimer().start();
  5:     try {
  6:         if (isReplication) {
  7:             numberOfReplicationsLastMin.increment();
  8:         }
  9: 
 10:         // Eureka-Server 发起的请求 或者 集群为空
 11:         // If it is a replication already, do not replicate again as this will create a poison replication
 12:         if (peerEurekaNodes == Collections.EMPTY_LIST || isReplication) {
 13:             return;
 14:         }
 15: 
 16:         for (final PeerEurekaNode node : peerEurekaNodes.getPeerEurekaNodes()) {
 17:             // If the url represents this host, do not replicate to yourself.
 18:             if (peerEurekaNodes.isThisMyUrl(node.getServiceUrl())) {
 19:                 continue;
 20:             }
 21:             replicateInstanceActionsToPeers(action, appName, id, info, newStatus, node);
 22:         }
 23:     } finally {
 24:         tracer.stop();
 25:     }
 26: }
```

* 第 10 至 14 行 ：Eureka-Server 在处理上述操作( Action )，无论来自 Eureka-Client 发起请求，还是 Eureka-Server 发起同步，调用的内部方法相同，通过 `isReplication=true` 参数，避免死循环同步。
* 第 16 至 22 行 ：**循环**集群内**每个**节点，调用 `#replicateInstanceActionsToPeers(...)` 方法，发起同步操作。

-------

`#replicateInstanceActionsToPeers(...)` 方法，代码如下：

```Java
private void replicateInstanceActionsToPeers(Action action, String appName,
                                            String id, InstanceInfo info, InstanceStatus newStatus,
                                            PeerEurekaNode node) {
   try {
       InstanceInfo infoFromRegistry;
       CurrentRequestVersion.set(Version.V2);
       switch (action) {
           case Cancel:
               node.cancel(appName, id);
               break;
           case Heartbeat:
               InstanceStatus overriddenStatus = overriddenInstanceStatusMap.get(id);
               infoFromRegistry = getInstanceByAppAndId(appName, id, false);
               node.heartbeat(appName, id, infoFromRegistry, overriddenStatus, false);
               break;
           case Register:
               node.register(info);
               break;
           case StatusUpdate:
               infoFromRegistry = getInstanceByAppAndId(appName, id, false);
               node.statusUpdate(appName, id, newStatus, infoFromRegistry);
               break;
           case DeleteStatusOverride:
               infoFromRegistry = getInstanceByAppAndId(appName, id, false);
               node.deleteStatusOverride(appName, id, infoFromRegistry);
               break;
       }
   } catch (Throwable t) {
       logger.error("Cannot replicate information to {} for action {}", node.getServiceUrl(), action.name(), t);
   }
}
```

* **Cancel** ：调用 `PeerEurekaNode#cancel(...)` 方法，点击 [链接](https://github.com/YunaiV/eureka/blob/69993ad1e80d45c43ac8585921eca4efb88b09b9/eureka-core/src/main/java/com/netflix/eureka/cluster/PeerEurekaNode.java#L157) 查看实现。
* **Heartbeat** ：调用 `PeerEurekaNode#heartbeat(...)` 方法，点击 [链接](https://github.com/YunaiV/eureka/blob/69993ad1e80d45c43ac8585921eca4efb88b09b9/eureka-core/src/main/java/com/netflix/eureka/cluster/PeerEurekaNode.java#L194) 查看实现。
* **Register** ：调用 `PeerEurekaNode#register(...)` 方法，点击 [链接](https://github.com/YunaiV/eureka/blob/69993ad1e80d45c43ac8585921eca4efb88b09b9/eureka-core/src/main/java/com/netflix/eureka/cluster/PeerEurekaNode.java#L134) 查看实现。
* **StatusUpdate** ：调用 `PeerEurekaNode#statusUpdate(...)` 方法，点击 [链接](https://github.com/YunaiV/eureka/blob/69993ad1e80d45c43ac8585921eca4efb88b09b9/eureka-core/src/main/java/com/netflix/eureka/cluster/PeerEurekaNode.java#L243) 查看实现。
* **DeleteStatusOverride** ：调用 `PeerEurekaNode#deleteStatusOverride(...)` 方法，点击 [链接](https://github.com/YunaiV/eureka/blob/69993ad1e80d45c43ac8585921eca4efb88b09b9/eureka-core/src/main/java/com/netflix/eureka/cluster/PeerEurekaNode.java#L294) 查看实现。
* 上面的每个方法实现，我们**都**会看到类似这么一段代码 ：

    ```Java
    batchingDispatcher.process(
        taskId("${action}", appName, id), // id
        new InstanceReplicationTask(targetHost, Action.Cancel, appName, id) {
            
            @Override
            public EurekaHttpResponse<Void> execute() {
                return replicationClient.doString(...);
            }
            
            @Override
            public void handleFailure(int statusCode, Object responseEntity) throws Throwable {
                // do Something...
            }
            
        }, // ReplicationTask 子类
        expiryTime
    )
    ```
    * `#task(...)` 方法，生成同步操作任务**编号**。代码如下：

        ```Java
        private static String taskId(String requestType, String appName, String id) {
           return requestType + '#' + appName + '/' + id;
        }
        ```
        * **相同应用实例的相同同步操作使用相同任务编号**。在 [《Eureka 源码解析 —— 任务批处理》「2. 整体流程」](http://www.iocoder.cn/Eureka/batch-tasks/?self) 中，我们看到" 接收线程( Runner )合并任务，将相同任务编号的任务合并，只执行一次。 "，因此，相同应用实例的相同同步操作就能被合并，减少操作量。例如，Eureka-Server 同步某个应用实例的 Heartbeat 操作，接收同步的 Eureak-Server 挂了，一方面这个应用的这次操作会**重试**，另一方面，这个应用实例会发起**新的** Heartbeat 操作，通过任务编号合并，接收同步的 Eureka-Server 恢复后，减少收到**重复积压**的任务。
   
   * InstanceReplicationTask ，同步操作任务，在 [「4.1.1 同步操作任务」](#) 详细解析。
   * `expiryTime` ，任务过期时间。

### 4.1.1 同步操作任务

![](http://www.iocoder.cn/images/Eureka/2018_08_07/03.png)

* `com.netflix.eureka.cluster.ReplicationTask` ，同步任务**抽象类**
    * 点击 [链接](https://github.com/YunaiV/eureka/blob/69993ad1e80d45c43ac8585921eca4efb88b09b9/eureka-core/src/main/java/com/netflix/eureka/cluster/ReplicationTask.java) 查看 ReplicationTask 代码。
    * 定义了 `#getTaskName()` **抽象**方法。
    * 定义了 `#execute()` **抽象**方法，执行同步任务。
    * 实现了 `#handleSuccess()` 方法，处理成功执行同步结果。
    * 实现了 `#handleFailure(...)` 方法，处理失败执行同步结果。
* `com.netflix.eureka.cluster.InstanceReplicationTask` ，同步应用实例任务**抽象类**
    * 点击 [链接](https://github.com/YunaiV/eureka/blob/69993ad1e80d45c43ac8585921eca4efb88b09b9/eureka-core/src/main/java/com/netflix/eureka/cluster/InstanceReplicationTask.java) 查看 InstanceReplicationTask 代码。
    * 实现了父类 `#getTaskName()` **抽象**方法。
* `com.netflix.eureka.cluster.AsgReplicationTask` ，亚马逊 AWS 使用，暂时跳过。

从上面 `PeerEurekaNode#同步操作(...)` 方法，**全部**实现了 InstanceReplicationTask 类的 `#execute()` 方法，**部分**重写了 `#handleFailure(...)` 方法。

### 4.1.2 同步操作任务处理器

`com.netflix.eureka.cluster.InstanceReplicationTask` ，实现 TaskProcessor **接口**，同步操作任务处理器。

* TaskProcessor ，在 [《Eureka 源码解析 —— 任务批处理》「10. 任务执行器【执行任务】」](http://www.iocoder.cn/Eureka/batch-tasks/?self) 有详细解析。
* 点击 [链接](https://github.com/YunaiV/eureka/blob/69993ad1e80d45c43ac8585921eca4efb88b09b9/eureka-core/src/main/java/com/netflix/eureka/cluster/ReplicationTaskProcessor.java) 查看 InstanceReplicationTask 代码。

`ReplicationTaskProcessor#process(task)` ，**处理单任务**，用于 Eureka-Server 向亚马逊 AWS 的 ASG ( `Autoscaling Group` ) 同步状态，暂时跳过，感兴趣的同学可以点击 [链接](https://github.com/YunaiV/eureka/blob/94a4a1ebd35a3350893369a05757c01b2534b702/eureka-core/src/main/java/com/netflix/eureka/cluster/ReplicationTaskProcessor.java#L38) 查看方法代码。

`ReplicationTaskProcessor#process(tasks)` ，**处理批量任务**，用于 Eureka-Server 集群注册信息的同步操作任务，通过调用被同步的 Eureka-Server 的 `peerreplication/batch/` 接口，一次性将批量( 多个 )的同步操作任务发起请求，代码如下：

```Java
  1: @Override
  2: public ProcessingResult process(List<ReplicationTask> tasks) {
  3:     // 创建 批量提交同步操作任务的请求对象
  4:     ReplicationList list = createReplicationListOf(tasks);
  5:     try {
  6:         // 发起 批量提交同步操作任务的请求
  7:         EurekaHttpResponse<ReplicationListResponse> response = replicationClient.submitBatchUpdates(list);
  8:         // 处理 批量提交同步操作任务的响应
  9:         int statusCode = response.getStatusCode();
 10:         if (!isSuccess(statusCode)) {
 11:             if (statusCode == 503) {
 12:                 logger.warn("Server busy (503) HTTP status code received from the peer {}; rescheduling tasks after delay", peerId);
 13:                 return ProcessingResult.Congestion;
 14:             } else {
 15:                 // Unexpected error returned from the server. This should ideally never happen.
 16:                 logger.error("Batch update failure with HTTP status code {}; discarding {} replication tasks", statusCode, tasks.size());
 17:                 return ProcessingResult.PermanentError;
 18:             }
 19:         } else {
 20:             handleBatchResponse(tasks, response.getEntity().getResponseList());
 21:         }
 22:     } catch (Throwable e) {
 23:         if (isNetworkConnectException(e)) {
 24:             logNetworkErrorSample(null, e);
 25:             return ProcessingResult.TransientError;
 26:         } else {
 27:             logger.error("Not re-trying this exception because it does not seem to be a network exception", e);
 28:             return ProcessingResult.PermanentError;
 29:         }
 30:     }
 31:     return ProcessingResult.Success;
 32: }
```

* 第 4 行 ：创建批量提交同步操作任务的请求对象( ReplicationList ) 。比较易懂，咱就不啰嗦贴代码了。
    * ReplicationList ，点击 [链接](https://github.com/YunaiV/eureka/blob/94a4a1ebd35a3350893369a05757c01b2534b702/eureka-core/src/main/java/com/netflix/eureka/cluster/protocol/ReplicationList.java) 查看类。
    * ReplicationInstance ，点击 [链接](https://github.com/YunaiV/eureka/blob/94a4a1ebd35a3350893369a05757c01b2534b702/eureka-core/src/main/java/com/netflix/eureka/cluster/protocol/ReplicationInstance.java) 查看类。
    * `#createReplicationListOf(...)` ，点击 [链接](https://github.com/YunaiV/eureka/blob/69993ad1e80d45c43ac8585921eca4efb88b09b9/eureka-core/src/main/java/com/netflix/eureka/cluster/ReplicationTaskProcessor.java#L142) 查看方法。
    * `#createReplicationInstanceOf(...)` ，点击 [链接](https://github.com/YunaiV/eureka/blob/69993ad1e80d45c43ac8585921eca4efb88b09b9/eureka-core/src/main/java/com/netflix/eureka/cluster/ReplicationTaskProcessor.java#L173) 查看方法。
* 第 7 行 ：调用 `JerseyReplicationClient#submitBatchUpdates(...)` 方法，请求 `peerreplication/batch/` 接口，一次性将批量( 多个 )的同步操作任务发起请求。
    * `JerseyReplicationClient#submitBatchUpdates(...)` 方法，点击 [链接](https://github.com/YunaiV/eureka/blob/69993ad1e80d45c43ac8585921eca4efb88b09b9/eureka-core/src/main/java/com/netflix/eureka/transport/JerseyReplicationClient.java#L109) 查看方法。 
    * ReplicationListResponse ，点击 [链接](https://github.com/YunaiV/eureka/blob/69993ad1e80d45c43ac8585921eca4efb88b09b9/eureka-core/src/main/java/com/netflix/eureka/cluster/protocol/ReplicationListResponse.java) 查看类。
    * ReplicationInstanceResponse ，点击 [链接](https://github.com/YunaiV/eureka/blob/69993ad1e80d45c43ac8585921eca4efb88b09b9/eureka-core/src/main/java/com/netflix/eureka/cluster/protocol/ReplicationInstanceResponse.java) 查看类。
* 第 9 至 31 行 ：处理批量提交同步操作任务的响应，在 [「4.4 处理 Eureka-Server 同步结果」](#) 详细解析。

## 4.3 接收 Eureka-Server 同步操作

`com.netflix.eureka.resources.PeerReplicationResource` ，同步操作任务 Resource ( Controller )。

`peerreplication/batch/` 接口，映射 `PeerReplicationResource#batchReplication(...)` 方法，代码如下：

```Java
  1: @Path("batch")
  2: @POST
  3: public Response batchReplication(ReplicationList replicationList) {
  4:     try {
  5:         ReplicationListResponse batchResponse = new ReplicationListResponse();
  6:         // 逐个同步操作任务处理，并将处理结果( ReplicationInstanceResponse ) 合并到 ReplicationListResponse 。
  7:         for (ReplicationInstance instanceInfo : replicationList.getReplicationList()) {
  8:             try {
  9:                 batchResponse.addResponse(dispatch(instanceInfo));
 10:             } catch (Exception e) {
 11:                 batchResponse.addResponse(new ReplicationInstanceResponse(Status.INTERNAL_SERVER_ERROR.getStatusCode(), null));
 12:                 logger.error(instanceInfo.getAction() + " request processing failed for batch item "
 13:                         + instanceInfo.getAppName() + '/' + instanceInfo.getId(), e);
 14:             }
 15:         }
 16:         return Response.ok(batchResponse).build();
 17:     } catch (Throwable e) {
 18:         logger.error("Cannot execute batch Request", e);
 19:         return Response.status(Status.INTERNAL_SERVER_ERROR).build();
 20:     }
 21: }
 22: 
 23: private ReplicationInstanceResponse dispatch(ReplicationInstance instanceInfo) {
 24:     ApplicationResource applicationResource = createApplicationResource(instanceInfo);
 25:     InstanceResource resource = createInstanceResource(instanceInfo, applicationResource);
 26: 
 27:     String lastDirtyTimestamp = toString(instanceInfo.getLastDirtyTimestamp());
 28:     String overriddenStatus = toString(instanceInfo.getOverriddenStatus());
 29:     String instanceStatus = toString(instanceInfo.getStatus());
 30: 
 31:     Builder singleResponseBuilder = new Builder();
 32:     switch (instanceInfo.getAction()) {
 33:         case Register:
 34:             singleResponseBuilder = handleRegister(instanceInfo, applicationResource);
 35:             break;
 36:         case Heartbeat:
 37:             singleResponseBuilder = handleHeartbeat(serverConfig, resource, lastDirtyTimestamp, overriddenStatus, instanceStatus);
 38:             break;
 39:         case Cancel:
 40:             singleResponseBuilder = handleCancel(resource);
 41:             break;
 42:         case StatusUpdate:
 43:             singleResponseBuilder = handleStatusUpdate(instanceInfo, resource);
 44:             break;
 45:         case DeleteStatusOverride:
 46:             singleResponseBuilder = handleDeleteStatusOverride(instanceInfo, resource);
 47:             break;
 48:     }
 49:     return singleResponseBuilder.build();
 50: }
```

* 第 7 至 15 行 ：逐个处理**单个**同步操作任务，并将处理结果( ReplicationInstanceResponse ) 添加到 ReplicationListResponse 。
* 第 23 至 50 行 ：处理**单个**同步操作任务，返回处理结果( ReplicationInstanceResponse )。
    * 第 24 至 25 行 ：创建 ApplicationResource , InstanceResource 。我们看到，实际该方法是把**单个**同步操作任务提交到其他 Resource ( Controller ) 处理，Eureka-Server 收到 Eureka-Client 请求响应的 Resource ( Controller ) 是**相同的逻辑**。
    * Register ：点击 [链接](https://github.com/YunaiV/eureka/blob/69993ad1e80d45c43ac8585921eca4efb88b09b9/eureka-core/src/main/java/com/netflix/eureka/resources/PeerReplicationResource.java#L137) 查看 `#handleRegister(...)` 方法。
    * Heartbeat ：点击 [链接](https://github.com/YunaiV/eureka/blob/69993ad1e80d45c43ac8585921eca4efb88b09b9/eureka-core/src/main/java/com/netflix/eureka/resources/PeerReplicationResource.java#L147) 查看 `#handleHeartbeat(...)` 方法。
    * Cancel ：点击 [链接](https://github.com/YunaiV/eureka/blob/69993ad1e80d45c43ac8585921eca4efb88b09b9/eureka-core/src/main/java/com/netflix/eureka/resources/PeerReplicationResource.java#L142) 查看 `#handleCancel(...)` 方法。
    * StatusUpdate ：点击 [链接](https://github.com/YunaiV/eureka/blob/69993ad1e80d45c43ac8585921eca4efb88b09b9/eureka-core/src/main/java/com/netflix/eureka/resources/PeerReplicationResource.java#L165) 查看 `#handleStatusUpdate(...)` 方法。
    * DeleteStatusOverride ：点击 [链接](https://github.com/YunaiV/eureka/blob/69993ad1e80d45c43ac8585921eca4efb88b09b9/eureka-core/src/main/java/com/netflix/eureka/resources/PeerReplicationResource.java#L170) 查看 `#handleDeleteStatusOverride(...)` 方法。

## 4.4 处理 Eureka-Server 同步结果

😈 想想就有小激动，终于写到这里了。

接 `ReplicationTaskProcessor#process(tasks)` 方法，处理批量提交同步操作任务的响应，代码如下：

```Java
  1: @Override
  2: public ProcessingResult process(List<ReplicationTask> tasks) {
  3:     // 创建 批量提交同步操作任务的请求对象
  4:     ReplicationList list = createReplicationListOf(tasks);
  5:     try {
  6:         // 发起 批量提交同步操作任务的请求
  7:         EurekaHttpResponse<ReplicationListResponse> response = replicationClient.submitBatchUpdates(list);
  8:         // 处理 批量提交同步操作任务的响应
  9:         int statusCode = response.getStatusCode();
 10:         if (!isSuccess(statusCode)) {
 11:             if (statusCode == 503) {
 12:                 logger.warn("Server busy (503) HTTP status code received from the peer {}; rescheduling tasks after delay", peerId);
 13:                 return ProcessingResult.Congestion;
 14:             } else {
 15:                 // Unexpected error returned from the server. This should ideally never happen.
 16:                 logger.error("Batch update failure with HTTP status code {}; discarding {} replication tasks", statusCode, tasks.size());
 17:                 return ProcessingResult.PermanentError;
 18:             }
 19:         } else {
 20:             handleBatchResponse(tasks, response.getEntity().getResponseList());
 21:         }
 22:     } catch (Throwable e) {
 23:         if (isNetworkConnectException(e)) {
 24:             logNetworkErrorSample(null, e);
 25:             return ProcessingResult.TransientError;
 26:         } else {
 27:             logger.error("Not re-trying this exception because it does not seem to be a network exception", e);
 28:             return ProcessingResult.PermanentError;
 29:         }
 30:     }
 31:     return ProcessingResult.Success;
 32: }
```

* 第 10 行 ，调用 `#isSuccess(...)` 方法，判断请求是否成功，响应状态码是否在  [200, 300) 范围内。
* 第 11 至 13 行 ：状态码 503 ，目前 Eureka-Server 返回 503 的原因是被限流。在 [《Eureka 源码解析 —— 基于令牌桶算法的 RateLimiter》](http://www.iocoder.cn/Eureka/rate-limiter/?self) 详细解析。**该情况为瞬时错误，会重试该同步操作任务**，在 [《Eureka 源码解析 —— 任务批处理》「3. 任务处理器」](http://www.iocoder.cn/Eureka/batch-tasks/?self) 有详细解析。
* 第 14 至 18 行 ：非**预期**状态码，目前 Eureka-Server 在代码上看下来，不会返回这样的状态码。**该情况为永久错误，会重试该同步操作任务**，在 [《Eureka 源码解析 —— 任务批处理》「3. 任务处理器」](http://www.iocoder.cn/Eureka/batch-tasks/?self) 有详细解析。
* 第 20 行 ：请求成功，调用 `#handleBatchResponse(...)` 方法，逐个处理**每个** ReplicationTask 和 ReplicationInstanceResponse 。**这里有一点要注意下，请求成功指的是整个请求成功，实际每个 ReplicationInstanceResponse 可能返回的状态码不在 [200, 300) 范围内**。该方法下文详细解析。
* 第 23 至 25 行 ：请求发生网络异常，例如网络超时，打印网络异常日志。目前日志的打印为部分采样，条件为网络发生异常每间隔 10 秒打印一条，避免网络发生异常打印超级大量的日志。**该情况为永久错误，会重试该同步操作任务**，在 [《Eureka 源码解析 —— 任务批处理》「3. 任务处理器」](http://www.iocoder.cn/Eureka/batch-tasks/?self) 有详细解析。
    * `#isNetworkConnectException(...)` ，点击 [链接](https://github.com/YunaiV/eureka/blob/69993ad1e80d45c43ac8585921eca4efb88b09b9/eureka-core/src/main/java/com/netflix/eureka/cluster/ReplicationTaskProcessor.java#L163) 查看方法。
    * `#logNetworkErrorSample(...)` ，点击 [链接](https://github.com/YunaiV/eureka/blob/69993ad1e80d45c43ac8585921eca4efb88b09b9/eureka-core/src/main/java/com/netflix/eureka/cluster/ReplicationTaskProcessor.java#L103) 查看方法。

* 第 26 至 29 行 ：非**预期**异常，目前 Eureka-Server 在代码上看下来，不会抛出这样的异常。**该情况为永久错误，会重试该同步操作任务**，在 [《Eureka 源码解析 —— 任务批处理》「3. 任务处理器」](http://www.iocoder.cn/Eureka/batch-tasks/?self) 有详细解析。

-------

`#handleBatchResponse(...)` 方法，代码如下：

```Java
private void handleBatchResponse(List<ReplicationTask> tasks, List<ReplicationInstanceResponse> responseList) {
   if (tasks.size() != responseList.size()) {
       // This should ideally never happen unless there is a bug in the software.
       logger.error("Batch response size different from submitted task list ({} != {}); skipping response analysis", responseList.size(), tasks.size());
       return;
   }
   for (int i = 0; i < tasks.size(); i++) {
       handleBatchResponse(tasks.get(i), responseList.get(i));
   }
}

private void handleBatchResponse(ReplicationTask task, ReplicationInstanceResponse response) {
   // 执行成功
   int statusCode = response.getStatusCode();
   if (isSuccess(statusCode)) {
       task.handleSuccess();
       return;
   }

   // 执行失败
   try {
       task.handleFailure(response.getStatusCode(), response.getResponseEntity());
   } catch (Throwable e) {
       logger.error("Replication task " + task.getTaskName() + " error handler failure", e);
   }
}
```

* `ReplicationTask#handleSuccess()` 方法，无任务同步操作任务重写，是个**空方法**，代码如下：

    ```
    // ReplicationTask.java
    public void handleSuccess() {
    }
    ```

* `ReplicationTask#handleFailure()` 方法，有**两个**同步操作任务重写：

    * Cancel ：当 Eureka-Server 不存在下线的应用实例时，返回 404 状态码，此时打印错误日志，代码如下：

        ```Java
        // PeerEurekaNode#cancel(...)
        @Override
        public void handleFailure(int statusCode, Object responseEntity) throws Throwable {
            super.handleFailure(statusCode, responseEntity);
            if (statusCode == 404) {
                logger.warn("{}: missing entry.", getTaskName());
            }
        }   
        ```
        * x

    * Heartbeat ：情况较为复杂，我们换一行继续说，避免排版有问题，影响阅读。

噔噔噔恰，本文的重要头戏来啦！Last But Very Importment ！！！

Eureka-Server 是允许**同一时刻**允许在任意节点被 Eureka-Client 发起**写入**相关的操作，网络是不可靠的资源，Eureka-Client 可能向一个 Eureka-Server 注册成功，但是网络波动，导致 Eureka-Client 误以为失败，此时恰好 Eureka-Client 变更了应用实例的状态，重试向另一个 Eureka-Server 注册，那么两个 Eureka-Server 对该应用实例的状态产生冲突。

再例如...... 我们不要继续举例子，网络波动真的很复杂。我们来看看 Eureka 是怎么处理的。

应用实例( InstanceInfo ) 的 `lastDirtyTimestamp` 属性，使用**时间戳**，表示应用实例的**版本号**，当请求方( 不仅仅是 Eureka-Client ，也可能是同步注册操作的 Eureka-Server ) 向 Eureka-Server 发起注册时，若 Eureka-Server 已存在拥有更大 `lastDirtyTimestamp` 该实例( **相同应用并且相同应用实例编号被认为是相同实例** )，则请求方注册的应用实例( InstanceInfo ) 无法覆盖注册此 Eureka-Server 的该实例( 见 `AbstractInstanceRegistry#register(...)` 方法 )。例如我们上面举的例子，第一个 Eureka-Server 向 第二个 Eureka-Server 同步注册应用实例时，不会注册覆盖，反倒是第二个 Eureka-Server 同步注册应用到第一个 Eureka-Server ，注册覆盖成功，因为 `lastDirtyTimestamp` ( 应用实例状态变更时，可以设置 `lastDirtyTimestamp` 为当前时间，见 `ApplicationInfoManager#setInstanceStatus(status)` 方法 )。

但是光靠**注册**请求判断 `lastDirtyTimestamp` 显然是不够的，因为网络异常情况下时，同步操作任务多次执行失败到达过期时间后，此时在 Eureka-Server 集群同步起到最终一致性**最最最**关键性出现了：Heartbeat 。因为 Heartbeat 会周期性的执行，通过它一方面可以判断 Eureka-Server 是否存在心跳对应的应用实例，另外一方面可以比较应用实例的 `lastDirtyTimestamp` 。当满足下面任意条件，Eureka-Server 返回 404 状态码：

* 1）Eureka-Server 应用实例不存在，点击 [链接](https://github.com/YunaiV/eureka/blob/69993ad1e80d45c43ac8585921eca4efb88b09b9/eureka-core/src/main/java/com/netflix/eureka/registry/AbstractInstanceRegistry.java#L438) 查看触发条件代码位置。
* 2）Eureka-Server 应用实例状态为 `UNKNOWN`，点击 [链接](https://github.com/YunaiV/eureka/blob/69993ad1e80d45c43ac8585921eca4efb88b09b9/eureka-core/src/main/java/com/netflix/eureka/registry/AbstractInstanceRegistry.java#L450) 查看触发条件代码位置。为什么会是 `UNKNOWN` ，在 [《Eureka 源码解析 —— 应用实例注册发现（八）之覆盖状态》「 4.3 续租场景」](http://www.iocoder.cn/Eureka/instance-registry-override-status/?self) 有详细解析。
* **3）**请求的 `lastDirtyTimestamp` 更大，点击 [链接](https://github.com/YunaiV/eureka/blob/69993ad1e80d45c43ac8585921eca4efb88b09b9/eureka-core/src/main/java/com/netflix/eureka/resources/InstanceResource.java#L306) 查看触发条件代码位置。

请求方接收到 404 状态码返回后，**认为 Eureka-Server 应用实例实际是不存在的**，重新发起应用实例的注册。以本文的 Heartbeat 为例子，代码如下：

```Java
// PeerEurekaNode#heartbeat(...)
  1: @Override
  2: public void handleFailure(int statusCode, Object responseEntity) throws Throwable {
  3:     super.handleFailure(statusCode, responseEntity);
  4:     if (statusCode == 404) {
  5:         logger.warn("{}: missing entry.", getTaskName());
  6:         if (info != null) {
  7:             logger.warn("{}: cannot find instance id {} and hence replicating the instance with status {}",
  8:                     getTaskName(), info.getId(), info.getStatus());
  9:             register(info);
 10:         }
 11:     } else if (config.shouldSyncWhenTimestampDiffers()) {
 12:         InstanceInfo peerInstanceInfo = (InstanceInfo) responseEntity;
 13:         if (peerInstanceInfo != null) {
 14:             syncInstancesIfTimestampDiffers(appName, id, info, peerInstanceInfo);
 15:         }
 16:     }
 17: }
```

* 第 4 至 10 行 ：接收到 404 状态码，调用 `#register(...)` 方法，向该被心跳同步操作失败的 Eureka-Server 发起注册**本地的应用实例**的请求。
    * 上述 **3）** ，会使用请求参数 `overriddenStatus` 存储到 Eureka-Server 的应用实例覆盖状态集合( `AbstractInstanceRegistry.overriddenInstanceStatusMap` )，点击 [链接](https://github.com/YunaiV/eureka/blob/69993ad1e80d45c43ac8585921eca4efb88b09b9/eureka-core/src/main/java/com/netflix/eureka/resources/InstanceResource.java#L123) 查看触发条件代码位置。

* 第 11 至 16 行 ：恰好是 **3）** 反过来的情况，本地的应用实例的 `lastDirtyTimestamp` 小于 Eureka-Server 该应用实例的，此时 Eureka-Server 返回 409 状态码，点击 [链接](https://github.com/YunaiV/eureka/blob/69993ad1e80d45c43ac8585921eca4efb88b09b9/eureka-core/src/main/java/com/netflix/eureka/resources/InstanceResource.java#L314) 查看触发条件代码位置。调用 `#syncInstancesIfTimestampDiffers()` 方法，覆盖注册本地应用实例，点击 [链接](https://github.com/YunaiV/eureka/blob/7f868f9ca715a8862c0c10cac04e238bbf371db0/eureka-core/src/main/java/com/netflix/eureka/cluster/PeerEurekaNode.java#L387) 查看方法。

OK，撒花！记住：Eureka 通过 Heartbeat 实现 Eureka-Server 集群同步的最终一致性。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

写的比较嗨皮，所以就送胖友一只胖友

![](http://www.iocoder.cn/images/Eureka/2018_08_07/04.png)

胖友，分享我的公众号( **芋道源码** ) 给你的胖友可好？

以下是草稿，可以凑合看

eureka server 集群假定是 s1 s2  

1）client 向 s1 注册，有一个 lastDirtyTime ，正常情况下成功， s1 会向 s2 同步  
2）client 向 s1 注册（成功，但是网络波动），然后 client 发生状态的变化，lastDirtyTime 变化，向 s2 注册。  
这个时候，s1 s2 是冲突的，但是他们会互相同步，实际 s2 => s1 的注册会真正成功，s1 => s2 的注册不会返回失败，但是实际 s2 处理的时候，用的是自身的。  

心跳只是最终去校验。  

理论来说，心跳不应该带 lastDirtyTime 参数。带的原因就是为了做固定周期的比较。  

最优解是 注册 就处理掉数据不一致  
次优解是 心跳 处理掉数据不一致  

如果在类比，  

注册，相当于 insertOrUpdate   
心跳，附加了校验是否要发起【注册】  

