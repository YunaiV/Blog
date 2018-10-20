title: Apollo 源码解析 —— Config Service 记录 Instance
date: 2018-04-25
tags:
categories: Apollo
permalink: Apollo/config-service-audit-instance

-------

摘要: 原创出处 http://www.iocoder.cn/Apollo/config-service-audit-instance/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Apollo/config-service-audit-instance/)
- [2. 实体](http://www.iocoder.cn/Apollo/config-service-audit-instance/)
  - [2.1 Instance](http://www.iocoder.cn/Apollo/config-service-audit-instance/)
  - [2.2 InstanceConfig](http://www.iocoder.cn/Apollo/config-service-audit-instance/)
- [3. InstanceConfigAuditUtil](http://www.iocoder.cn/Apollo/config-service-audit-instance/)
  - [3.1 构造方法](http://www.iocoder.cn/Apollo/config-service-audit-instance/)
  - [3.2 初始化任务](http://www.iocoder.cn/Apollo/config-service-audit-instance/)
  - [3.3 audit](http://www.iocoder.cn/Apollo/config-service-audit-instance/)
  - [3.4 doAudit](http://www.iocoder.cn/Apollo/config-service-audit-instance/)
- [666. 彩蛋](http://www.iocoder.cn/Apollo/config-service-audit-instance/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2018_05_18.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

# 1. 概述

> 老艿艿：本系列假定胖友已经阅读过 [《Apollo 官方 wiki 文档》](https://github.com/ctripcorp/apollo/wiki/)  。

在 Portal 的应用详情页，我们可以看到每个 Namespace 下的**实例**列表。如下图所示：![实例列表](http://www.iocoder.cn/images/Apollo/2018_04_25/01.png)

* 实例( Instance )，实际就是 Apollo 的**客户端**。

**本文分享实例相关的实体和如何存储的**。

# 2. 实体

## 2.1 Instance

`com.ctrip.framework.apollo.biz.entity.Instance` ，Instance **实体**。代码如下：

```Java
@Entity
@Table(name = "Instance")
public class Instance {

    /**
     * 编号
     */
    @Id
    @GeneratedValue
    @Column(name = "Id")
    private long id;
    /**
     * App 编号
     */
    @Column(name = "AppId", nullable = false)
    private String appId;
    /**
     * Cluster 名字
     */
    @Column(name = "ClusterName", nullable = false)
    private String clusterName;
    /**
     * 数据中心的 Cluster 名字
     */
    @Column(name = "DataCenter", nullable = false)
    private String dataCenter;
    /**
     * 客户端 IP
     */
    @Column(name = "Ip", nullable = false)
    private String ip;
    /**
     * 数据创建时间
     */
    @Column(name = "DataChange_CreatedTime", nullable = false)
    private Date dataChangeCreatedTime;
    /**
     * 数据最后更新时间
     */
    @Column(name = "DataChange_LastTime")
    private Date dataChangeLastModifiedTime;

    @PrePersist
    protected void prePersist() {
        if (this.dataChangeCreatedTime == null) {
            dataChangeCreatedTime = new Date();
        }
        if (this.dataChangeLastModifiedTime == null) {
            dataChangeLastModifiedTime = dataChangeCreatedTime;
        }
    }
}
```

* `id` 字段，编号，自增。
* `appId` + `clusterName` + `dataCenter` + `ip` 组成**唯一索引**，通过这四个字段**唯一一个实例( 客户端 )**。

## 2.2 InstanceConfig

`com.ctrip.framework.apollo.biz.entity.InstanceConfig` ，Instance Config **实体**，记录 Instance 对 Namespace 的配置的**获取**情况。如果一个 Instance 使用了**多个** Namespace ，则会记录**多条** InstanceConfig 。

代码如下：

```Java
@Entity
@Table(name = "InstanceConfig")
public class InstanceConfig {

    /**
     * 编号
     */
    @Id
    @GeneratedValue
    @Column(name = "Id")
    private long id;
    /**
     * Instance 编号，指向 {@link Instance#id}
     */
    @Column(name = "InstanceId")
    private long instanceId;
    /**
     * App 编号
     */
    @Column(name = "ConfigAppId", nullable = false)
    private String configAppId;
    /**
     * Cluster 名字
     */
    @Column(name = "ConfigClusterName", nullable = false)
    private String configClusterName;
    /**
     * Namespace 名字
     */
    @Column(name = "ConfigNamespaceName", nullable = false)
    private String configNamespaceName;
    /**
     * Release Key ，对应 {@link Release#releaseKey}
     */
    @Column(name = "ReleaseKey", nullable = false)
    private String releaseKey;
    /**
     * 配置下发时间
     */
    @Column(name = "ReleaseDeliveryTime", nullable = false)
    private Date releaseDeliveryTime;
    /**
     * 数据创建时间
     */
    @Column(name = "DataChange_CreatedTime", nullable = false)
    private Date dataChangeCreatedTime;
    /**
     * 数据最后更新时间
     */
    @Column(name = "DataChange_LastTime")
    private Date dataChangeLastModifiedTime;

    @PrePersist
    protected void prePersist() {
        if (this.dataChangeCreatedTime == null) {
            dataChangeCreatedTime = new Date();
        }
        if (this.dataChangeLastModifiedTime == null) {
            dataChangeLastModifiedTime = dataChangeCreatedTime;
        }
    }
}
```

* `id` 字段，编号，自增。
* `instanceId` + `configAppId`  + `ConfigNamespaceName` 组成**唯一索引**，因为一个 Instance 可以使用**多个** Namespace 。
* `releaseKey` 字段，Release Key ，对应 `Release.releaseKey` 字段。
* `releaseDeliveryTime` 字段，配置下发时间。
* 通过 `releaseKey` + `releaseDeliveryTime` 字段，可以很容易判断 Instance 在当前 Namespace 获取**配置的情况**。
* `configClusterName` 字段，Cluster 名字。

# 3. InstanceConfigAuditUtil

在 [《Apollo 源码解析 —— Config Service 配置读取接口》](http://www.iocoder.cn/Apollo/config-service-config-query-api/?self) 中，我们看到，客户端读取配置时，会调用 Config Service 的 **GET `/configs/{appId}/{clusterName}/{namespace:.+}` 接口**。在接口中，会调用 `InstanceConfigAuditUtil#audit(...)` 的方法，代码如下：

```Java
private void auditReleases(String appId, String cluster, String dataCenter, String clientIp,
                           List<Release> releases) {
    if (Strings.isNullOrEmpty(clientIp)) {
        //no need to audit instance config when there is no ip
        return;
    }
    // 循环 Release 数组
    for (Release release : releases) {
        // 记录 InstanceConfig
        instanceConfigAuditUtil.audit(appId, cluster, dataCenter, clientIp, release.getAppId(),
                release.getClusterName(),
                release.getNamespaceName(), release.getReleaseKey());
    }
}
```

下面我们来看看 InstanceConfigAuditUtil 的具体实现。

`com.ctrip.framework.apollo.configservice.util.InstanceConfigAuditUtil` ，实现 InitializingBean 接口，InstanceConfig 审计工具类。

## 3.1 构造方法

```Java
/**
 * {@link #audits} 大小
 */
private static final int INSTANCE_CONFIG_AUDIT_MAX_SIZE = 10000;
/**
 * {@link #instanceCache} 大小
 */
private static final int INSTANCE_CACHE_MAX_SIZE = 50000;
/**
 * {@link #instanceConfigReleaseKeyCache} 大小
 */
private static final int INSTANCE_CONFIG_CACHE_MAX_SIZE = 50000;
private static final long OFFER_TIME_LAST_MODIFIED_TIME_THRESHOLD_IN_MILLI = TimeUnit.MINUTES.toMillis(10);//10 minutes

private static final Joiner STRING_JOINER = Joiner.on(ConfigConsts.CLUSTER_NAMESPACE_SEPARATOR);

/**
 * ExecutorService 对象。队列大小为 1 。
 */
private final ExecutorService auditExecutorService;
/**
 * 是否停止
 */
private final AtomicBoolean auditStopped;
/**
 * 队列
 */
private BlockingQueue<InstanceConfigAuditModel> audits = Queues.newLinkedBlockingQueue(INSTANCE_CONFIG_AUDIT_MAX_SIZE);
/**
 * Instance 的编号的缓存
 *
 * KEY：{@link #assembleInstanceKey(String, String, String, String)}
 * VALUE：{@link Instance#id}
 */
private Cache<String, Long> instanceCache;
/**
 * InstanceConfig 的 ReleaseKey 的缓存
 *
 * KEY：{@link #assembleInstanceConfigKey(long, String, String)}
 * VALUE：{@link InstanceConfig#id}
 */
private Cache<String, String> instanceConfigReleaseKeyCache;

@Autowired
private InstanceService instanceService;

public InstanceConfigAuditUtil() {
    auditExecutorService = Executors.newSingleThreadExecutor(ApolloThreadFactory.create("InstanceConfigAuditUtil", true));
    auditStopped = new AtomicBoolean(false);
    instanceCache = CacheBuilder.newBuilder().expireAfterAccess(1, TimeUnit.HOURS).maximumSize(INSTANCE_CACHE_MAX_SIZE).build();
    instanceConfigReleaseKeyCache = CacheBuilder.newBuilder().expireAfterWrite(1, TimeUnit.DAYS).maximumSize(INSTANCE_CONFIG_CACHE_MAX_SIZE).build();
}
```

* 基础属性
    * `instanceCache` 属性，Instance 的**编号**的**缓存**。其中：
        * KEY ，使用  `appId` + `clusterName` + `dataCenter` + `ip` ，恰好是 Instance 的唯一索引的字段。
        * VALUE ，使用 `id` 。
    * `instanceConfigReleaseKeyCache` 属性，InstanceConfig 的 **ReleaseKey** 的**缓存**。其中：
        * KEY ，使用 `instanceId` + `configAppId`  + `ConfigNamespaceName` ，恰好是 InstanceConfig 的唯一索引的字段。
        * VALUE ，使用 `releaseKey` 。
* 线程相关
    * InstanceConfigAuditUtil 记录 Instance 和 InstanceConfig 是提交到队列，使用线程池异步处理。
    * `auditExecutorService` 属性，ExecutorService 对象。队列大小为 **1** 。
    * `auditStopped` 属性，是否停止。
    * `audits` 属性，队列。

## 3.2 初始化任务

`#afterPropertiesSet()` 方法，通过 Spring 调用，**初始化任务**。代码如下：

```Java
  1: @Override
  2: public void afterPropertiesSet() {
  3:     // 提交任务
  4:     auditExecutorService.submit(() -> {
  5:         // 循环，直到停止或线程打断
  6:         while (!auditStopped.get() && !Thread.currentThread().isInterrupted()) {
  7:             try {
  8:                 // 获得队首 InstanceConfigAuditModel 元素，非阻塞
  9:                 InstanceConfigAuditModel model = audits.poll();
 10:                 // 若获取不到，sleep 等待 1 秒
 11:                 if (model == null) {
 12:                     TimeUnit.SECONDS.sleep(1);
 13:                     continue;
 14:                 }
 15:                 // 若获取到，记录 Instance 和 InstanceConfig
 16:                 doAudit(model);
 17:             } catch (Throwable ex) {
 18:                 Tracer.logError(ex);
 19:             }
 20:         }
 21:     });
 22: }
```

* 第 4 至 21 行：提交任务到 `auditExecutorService` 中。
    * 第 6 至 20 行：循环，直到停止或线程打断。
    * 第 9 行：调用 `BlockingQueue#poll()` 方法，获得**队首** InstanceConfigAuditModel 元素，**非阻塞**。
    * 第 10 至 14 行：若获取不到，**sleep** 等待 1 秒。
    * 第 16 行：若获取到，调用 `#doAudit(InstanceConfigAuditModel)` 方法，记录 Instance 和 InstanceConfig 。详细解析，见 [「3.4 doAudit」](#) 。

## 3.3 audit

`#audit(...)` 方法，添加到队列中。代码如下：

```Java
public boolean audit(String appId, String clusterName, String dataCenter, String
        ip, String configAppId, String configClusterName, String configNamespace, String releaseKey) {
    return this.audits.offer(new InstanceConfigAuditModel(appId, clusterName, dataCenter, ip,
            configAppId, configClusterName, configNamespace, releaseKey));
}
```

* 创建 InstanceConfigAuditModel 对象，代码如下：

    ```Java
    public static class InstanceConfigAuditModel {
    
        private String appId;
        private String clusterName;
        private String dataCenter;
        private String ip;
        private String configAppId;
        private String configClusterName;
        private String configNamespace;
        private String releaseKey;
        /**
         * 入队时间
         */
        private Date offerTime;
    
        public InstanceConfigAuditModel(String appId, String clusterName, String dataCenter, String
                clientIp, String configAppId, String configClusterName, String configNamespace, String
                                                releaseKey) {
            this.offerTime = new Date(); // 当前时间
            this.appId = appId;
            this.clusterName = clusterName;
            this.dataCenter = Strings.isNullOrEmpty(dataCenter) ? "" : dataCenter;
            this.ip = clientIp;
            this.configAppId = configAppId;
            this.configClusterName = configClusterName;
            this.configNamespace = configNamespace;
            this.releaseKey = releaseKey;
        }
    }
    ```
    * `offerTime` 属性，入队时间，取得当前时间，**避免异步处理的时间差**。

* 调用 `BlockingQueue#offset(InstanceConfigAuditModel)` 方法，添加到队列 `audits` 中。

## 3.4 doAudit

 `#doAudit(InstanceConfigAuditModel)` 方法，记录 Instance 和 InstanceConfig 。代码如下：
 
 ```Java
  1: void doAudit(InstanceConfigAuditModel auditModel) {
  2:     // 拼接 instanceCache 的 KEY
  3:     String instanceCacheKey = assembleInstanceKey(auditModel.getAppId(), auditModel.getClusterName(),
  4:             auditModel.getIp(), auditModel.getDataCenter());
  5:     // 获得 Instance 编号
  6:     Long instanceId = instanceCache.getIfPresent(instanceCacheKey);
  7:     // 查询不到，从 DB 加载或者创建，并添加到缓存中。
  8:     if (instanceId == null) {
  9:         instanceId = prepareInstanceId(auditModel);
 10:         instanceCache.put(instanceCacheKey, instanceId);
 11:     }
 12: 
 13:     // 获得 instanceConfigReleaseKeyCache 的 KEY
 14:     // load instance config release key from cache, and check if release key is the same
 15:     String instanceConfigCacheKey = assembleInstanceConfigKey(instanceId, auditModel.getConfigAppId(),
 16:             auditModel.getConfigNamespace());
 17:     // 获得缓存的 cacheReleaseKey
 18:     String cacheReleaseKey = instanceConfigReleaseKeyCache.getIfPresent(instanceConfigCacheKey);
 19:     // 若相等，跳过
 20:     // if release key is the same, then skip audit
 21:     if (cacheReleaseKey != null && Objects.equals(cacheReleaseKey, auditModel.getReleaseKey())) {
 22:         return;
 23:     }
 24:     // 更新对应的 instanceConfigReleaseKeyCache 缓存
 25:     instanceConfigReleaseKeyCache.put(instanceConfigCacheKey, auditModel.getReleaseKey());
 26:     // 获得 InstanceConfig 对象
 27:     // if release key is not the same or cannot find in cache, then do audit
 28:     InstanceConfig instanceConfig = instanceService.findInstanceConfig(instanceId, auditModel.getConfigAppId(),
 29:             auditModel.getConfigNamespace());
 30: 
 31:     // 若 InstanceConfig 已经存在，进行更新
 32:     if (instanceConfig != null) {
 33:         // ReleaseKey 发生变化
 34:         if (!Objects.equals(instanceConfig.getReleaseKey(), auditModel.getReleaseKey())) {
 35:             instanceConfig.setConfigClusterName(auditModel.getConfigClusterName());
 36:             instanceConfig.setReleaseKey(auditModel.getReleaseKey());
 37:             instanceConfig.setReleaseDeliveryTime(auditModel.getOfferTime()); // 配置下发时间，使用入队时间
 38:         // 时间过近，例如 Client 先请求的 Config Service A 节点，再请求 Config Service B 节点的情况。
 39:         } else if (offerTimeAndLastModifiedTimeCloseEnough(auditModel.getOfferTime(), instanceConfig.getDataChangeLastModifiedTime())) {
 40:             //when releaseKey is the same, optimize to reduce writes if the record was updated not long ago
 41:             return;
 42:         }
 43:         // 更新
 44:         //we need to update no matter the release key is the same or not, to ensure the
 45:         //last modified time is updated each day
 46:         instanceConfig.setDataChangeLastModifiedTime(auditModel.getOfferTime());
 47:         instanceService.updateInstanceConfig(instanceConfig);
 48:         return;
 49:     }
 50: 
 51:     // 若 InstanceConfig 不存在，创建 InstanceConfig 对象
 52:     instanceConfig = new InstanceConfig();
 53:     instanceConfig.setInstanceId(instanceId);
 54:     instanceConfig.setConfigAppId(auditModel.getConfigAppId());
 55:     instanceConfig.setConfigClusterName(auditModel.getConfigClusterName());
 56:     instanceConfig.setConfigNamespaceName(auditModel.getConfigNamespace());
 57:     instanceConfig.setReleaseKey(auditModel.getReleaseKey());
 58:     instanceConfig.setReleaseDeliveryTime(auditModel.getOfferTime());
 59:     instanceConfig.setDataChangeCreatedTime(auditModel.getOfferTime());
 60:     // 保存 InstanceConfig 对象到数据库中
 61:     try {
 62:         instanceService.createInstanceConfig(instanceConfig);
 63:     } catch (DataIntegrityViolationException ex) {
 64:         // concurrent insertion, safe to ignore
 65:     }
 66: }
 ```
 
 * ============ Instance 相关 ============
 * 第 2 至 4 行：拼接 `instanceCache` 的 KEY 。
 * 第 6 行：调用 `Cache#getIfPresent(key)` 从缓存 `instanceCache` 中获得 Instance 编号。
 * 第 7 至 11 行：查询不到，从 DB 加载或者创建，并添加到缓存中。`#prepareInstanceId(InstanceConfigAuditModel)` 方法，代码如下：

     ```Java
     private long prepareInstanceId(InstanceConfigAuditModel auditModel) {
        // 查询 Instance 对象
        Instance instance = instanceService.findInstance(auditModel.getAppId(), auditModel
                .getClusterName(), auditModel.getDataCenter(), auditModel.getIp());
        // 已存在，返回 Instance 编号
        if (instance != null) {
            return instance.getId();
        }
        // 若 Instance 不存在，创建 Instance 对象
        instance = new Instance();
        instance.setAppId(auditModel.getAppId());
        instance.setClusterName(auditModel.getClusterName());
        instance.setDataCenter(auditModel.getDataCenter());
        instance.setIp(auditModel.getIp());
        // 保存 Instance 对象到数据库中
        try {
            return instanceService.createInstance(instance).getId();
        } catch (DataIntegrityViolationException ex) {
            // 发生唯一索引冲突，意味着已经存在，进行查询 Instance 对象，并返回
            // return the one exists
            return instanceService.findInstance(instance.getAppId(), instance.getClusterName(),
                    instance.getDataCenter(), instance.getIp()).getId();
        }
    }
     ```
     * 🙂 代码比较简单，胖友看下注释。
 * ============ InstanceConfig 相关 ============
 * 第 15 行：拼接 `instanceConfigReleaseKeyCache` 的 KEY 。
 * 第 18 行：调用 `Cache#getIfPresent(key)` 从缓存 `instanceConfigReleaseKeyCache` 中获得 `cacheReleaseKey` 。
 * 第 19 至 23 行：若 `releaseKey` 相当，说明**无更新**，跳过。
 * 第 25 行：更新对应的 `instanceConfigReleaseKeyCache` 缓存。
 * 第 26 至 29 行：调用 `InstanceService#findInstanceConfig(...)` 方法，获得 InstanceConfig 对象。相比 Instance 来说，InstanceConfig 存在**更新**逻辑。
 * 第 31 至 49 行：若 InstanceConfig 已经存在，进行更新。
    * 第 34 至 37 行：若 `releaseKey` 发生变化，设置需要更新的字段 `configClusterName` `releaseKey` `releaseDeliveryTime` 。**注意**，`releaseDeliveryTime` 配置下发时间，使用入队时间。
    * 第 38 至 42 行：调用 `#offerTimeAndLastModifiedTimeCloseEnough(Date offerTime, Date lastModifiedTime)` 方法，时间过近，仅相差 **10** 分钟。例如，Client 先请求的 Config Service A 节点，再请求 Config Service B 节点的情况。此时，InstanceConfig 在 DB 中是已经更新了，但是在 Config Service B 节点的缓存是未更新的。`#offerTimeAndLastModifiedTimeCloseEnough(...)` 方法，代码如下：

        ```Java
        private boolean offerTimeAndLastModifiedTimeCloseEnough(Date offerTime, Date lastModifiedTime) {
            return (offerTime.getTime() - lastModifiedTime.getTime()) < OFFER_TIME_LAST_MODIFIED_TIME_THRESHOLD_IN_MILLI;
        }
        ```
        * x
    * 第 43 至 48 行：调用 `InstanceService#updateInstanceConfig(InstanceConfig)` 方法，更新 InstanceConfig。**结束处理**。
* 第 51 至 65 行：若 InstanceConfig 不存在，创建 InstanceConfig 对象。
    * 第 52 至 59 行：创建 InstanceConfig 对象。
    * 第 60 至 62 行：调用 `InstanceService#createInstanceConfig(InstanceConfig)` 方法，保存 InstanceConfig 对象到数据库中。

# 666. 彩蛋

Instance 和 InstanceConfig 相关的 Service 和 Controller 类的代码，胖友可以自己查看源码，比较好理解。老艿艿就不瞎比比啦。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)


