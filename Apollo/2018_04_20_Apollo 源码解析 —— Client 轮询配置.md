title: Apollo 源码解析 —— Client 轮询配置
date: 2018-04-20
tags:
categories: Apollo
permalink: Apollo/client-polling-config

---

摘要: 原创出处 http://www.iocoder.cn/Apollo/client-polling-config/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Apollo/client-polling-config/)
- [2. ConfigRepository](http://www.iocoder.cn/Apollo/client-polling-config/)
  - [2.1 AbstractConfigRepository](http://www.iocoder.cn/Apollo/client-polling-config/)
  - [2.2 RemoteConfigRepository](http://www.iocoder.cn/Apollo/client-polling-config/)
- [3. RemoteConfigLongPollService](http://www.iocoder.cn/Apollo/client-polling-config/)
  - [3.1 构造方法](http://www.iocoder.cn/Apollo/client-polling-config/)
  - [3.2 getConfigServices](http://www.iocoder.cn/Apollo/client-polling-config/)
  - [3.3 assembleLongPollRefreshUrl](http://www.iocoder.cn/Apollo/client-polling-config/)
  - [3.4 submit](http://www.iocoder.cn/Apollo/client-polling-config/)
  - [3.5 startLongPolling](http://www.iocoder.cn/Apollo/client-polling-config/)
  - [3.6 doLongPollingRefresh](http://www.iocoder.cn/Apollo/client-polling-config/)
  - [3.7 updateNotifications](http://www.iocoder.cn/Apollo/client-polling-config/)
  - [3.8 updateRemoteNotifications](http://www.iocoder.cn/Apollo/client-polling-config/)
  - [3.9 notify](http://www.iocoder.cn/Apollo/client-polling-config/)
- [4. SchedulePolicy](http://www.iocoder.cn/Apollo/client-polling-config/)
  - [4.1 ExponentialSchedulePolicy](http://www.iocoder.cn/Apollo/client-polling-config/)
- [666. 彩蛋](http://www.iocoder.cn/Apollo/client-polling-config/)

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

本文分享 Client 如何通过**轮询**的方式，从 Config Service 读取配置。Client 的**轮询**包括两部分：

1. RemoteConfigRepository ，**定时**轮询 Config Service 的配置读取 `/configs/{appId}/{clusterName}/{namespace:.+}` 接口。
    * 接口的逻辑，在 [《Apollo 源码解析 —— Config Service 配置读取接口》](http://www.iocoder.cn/Apollo/config-service-config-query-api/?self)  有详细解析。
2. RemoteConfigLongPollService ，**长**轮询 Config Service 的配置变更通知 `/notifications/v2` 接口。
    * 当有**新的**通知时，触发 RemoteConfigRepository ，**立即**轮询 Config Service 的配置读取 `/configs/{appId}/{clusterName}/{namespace:.+}` 接口。
    * 接口的逻辑，在 [《Apollo 源码解析 —— Config Service 通知配置变化》](http://www.iocoder.cn/Apollo/config-service-notifications/?self) 有详细解析。

整体流程如下图：![流程](http://www.iocoder.cn/images/Apollo/2018_04_20/01.png)

* 一个 Namespace 对应一个 RemoteConfigRepository 。
* **多个** RemoteConfigRepository ，注册到**全局唯一**的 RemoteConfigLongPollService 中。

为什么是这样的设计呢？老艿艿请教了 Apollo 的作者宋老师。聊天内容如下，非常感谢：

> * 老艿艿：
>   * https://github.com/ctripcorp/apollo/issues/652  
>   * 看这个 issue 问了 
>   * 问题：非常感谢。为什么不在 long polling 的返回结果中直接返回更新的结果呢？
>  * 回答：这样推送消息就是有状态了，做不到幂等了，会带来很多问题。
>   * [呲牙] 周末打扰哈。带来的问题主要是哪些哈？
> 
> * 张大佬：
>   * 可能会有数据的丢失，如果使用只推送变更通知的话，即使丢失了，还是能在下一次变更的时候达到一个一致的状态
> 
> * 宋老师：
>   * @老艿艿 主要是幂等考虑
>   * 加载配置接口是幂等的，推送配置的话就做不到了，因为和推送顺序相关
> 
> * 老艿艿：
>   * 推送顺序指的是？
> 
> * 张大佬：
>   * 我想的是，比如两次修改操作的通知，由于网络问题，客户端接收到的顺序可能是反的，如果这两次修改的是同一个key，就有问题了。
> 
> * 老艿艿：
>   * 从目前代码看下来，长轮询发起，同时只会有一个，应该不会同时两个通知呀。
>   * 现在 client 是一个长轮询+定时轮询，是不是这两个会有互相的影响。
> 
> * 张大佬：
>   * @宋老师 嗯嗯
> 
> * 宋老师：
>   * 目前推送是单连接走http的，所以问题可能不大，不过在设计上而言是有这个问题的，比如如果推送是走的tcp长连接的话。另外，长轮询和推送之间也会有冲突，如果连续两次配置变化，就可能造成双写。还有一点，就是保持逻辑的简单，目前的做法推送只负责做简单的通知，不需要去计算客户端的配置应该是什么，因为计算逻辑挺复杂的，需要考虑集群，关联，灰度等。
> 
> * 老艿艿：
>   * 现在的设计思路上，是不是可以理解.
>   * 1、client 的定时轮询，可以保持最终一致。
>   * 2、client 的长轮询，是定时轮训的“实时”补充，通过这样的方式，让流程统一？
> 
> * 宋老师：
>   * 总而言之，就是在满足幂等性，实时性的基础上保持设计的简单
>   * 是的，推拉结合
> 
> * 张大佬：
>   * 长轮询个推送直接的冲突没太理解
>   * 有没有可能有这种问题，一次长轮询中消息丢失了，但是长轮询还在
> 
> * 老艿艿：
>   * 1. 长轮询的通知里面，带有配置信息
>   * 2. 定时轮训，里面也拿到配置信息
>   * 这个时候，client 是没办法判断哪个配置信息是新的。
>   * @张大佬 我认为是可能的，这个时候，client 可以发起新的长轮询。
> 
> * 宋大佬：
>   * @张大佬 长轮询和推送的冲突，这个更正为定时轮询和推送的冲突
> 
> * 老艿艿：
>   * 通知是定时轮询配置的补充。有了通知，立马轮询。不用在定时了
> 
> * 张大佬：
>   * get
> 
> * 老艿艿：
>   * 谢谢宋老师和张大佬[坏笑][坏笑]

# 2. ConfigRepository

`com.ctrip.framework.apollo.internals.ConfigRepository` ，配置 Repository 接口。代码如下：

```Java
public interface ConfigRepository {

    /**
     * Get the config from this repository.
     *
     * @return config
     */
    Properties getConfig();

    /**
     * Set the fallback repo for this repository.
     *
     * @param upstreamConfigRepository the upstream repo
     */
    void setUpstreamRepository(ConfigRepository upstreamConfigRepository);

    /**
     * Add change listener.
     *
     * @param listener the listener to observe the changes
     */
    void addChangeListener(RepositoryChangeListener listener);

    /**
     * Remove change listener.
     *
     * @param listener the listener to remove
     */
    void removeChangeListener(RepositoryChangeListener listener);

}
```

* ConfigRepository ，作为 Client 的 **Repository** ( 类似 DAO ) ，读取配置。
* `#getConfig()` 方法，读取配置。
* `#setUpstreamRepository(ConfigRepository)` 方法，设置**上游**的 Repository 。主要用于 LocalFileConfigRepository ，从 Config Service 读取配置，缓存在本地文件。
* RepositoryChangeListener ，监听 Repository 的配置的**变化**。🙂
    * `#addChangeListener(RepositoryChangeListener)`
    * `#removeChangeListener(RepositoryChangeListener)`

-------

子类如下图所示：![类图](http://www.iocoder.cn/images/Apollo/2018_04_20/02.png)

* 本文不分享 LocalFileConfigRepository 。

## 2.1 AbstractConfigRepository

`com.ctrip.framework.apollo.internals.AbstractConfigRepository` ，实现 ConfigRepository 接口，配置 Repository **抽象类**。

### 2.1.1 同步配置

```Java
/**
 * 尝试同步
 *
 * @return 是否同步成功
 */
protected boolean trySync() {
    try {
        // 同步
        sync();
        // 返回同步成功
        return true;
    } catch (Throwable ex) {
        // 【TODO 6001】Tracer 日志
        Tracer.logEvent("ApolloConfigException", ExceptionUtil.getDetailMessage(ex));
        logger.warn("Sync config failed, will retry. Repository {}, reason: {}", this.getClass(), ExceptionUtil.getDetailMessage(ex));
    }
    // 返回同步失败
    return false;
}

/**
 * 同步配置
 */
protected abstract void sync();
```

* `#sync()` **抽象**方法，同步配置。

### 2.1.2 监听器

```Java
/**
 * RepositoryChangeListener 数组
 */
private List<RepositoryChangeListener> m_listeners = Lists.newCopyOnWriteArrayList();

@Override
public void addChangeListener(RepositoryChangeListener listener) {
    if (!m_listeners.contains(listener)) {
        m_listeners.add(listener);
    }
}

@Override
public void removeChangeListener(RepositoryChangeListener listener) {
    m_listeners.remove(listener);
}

/**
 * 触发监听器们
 *
 * @param namespace Namespace 名字
 * @param newProperties 配置
 */
protected void fireRepositoryChange(String namespace, Properties newProperties) {
    // 循环 RepositoryChangeListener 数组
    for (RepositoryChangeListener listener : m_listeners) {
        try {
            // 触发监听器
            listener.onRepositoryChange(namespace, newProperties);
        } catch (Throwable ex) {
            // 【TODO 6001】Tracer 日志
            Tracer.logError(ex);
            logger.error("Failed to invoke repository change listener {}", listener.getClass(), ex);
        }
    }
}
```

## 2.2 RemoteConfigRepository

`com.ctrip.framework.apollo.internals.RemoteConfigRepository` ，实现 AbstractConfigRepository 抽象类，远程配置 Repository 。实现从 Config Service 拉取配置，并**缓存**在内存中。并且，**定时 + 实时**刷新缓存。

### 2.2.1 构造方法

```Java
private static final Logger logger = LoggerFactory.getLogger(RemoteConfigRepository.class);
private static final Joiner STRING_JOINER = Joiner.on(ConfigConsts.CLUSTER_NAMESPACE_SEPARATOR);
private static final Joiner.MapJoiner MAP_JOINER = Joiner.on("&").withKeyValueSeparator("=");

private static final Escaper pathEscaper = UrlEscapers.urlPathSegmentEscaper();
private static final Escaper queryParamEscaper = UrlEscapers.urlFormParameterEscaper();

/**
 * 远程配置长轮询服务
 */
private RemoteConfigLongPollService remoteConfigLongPollService;
/**
 * 指向 ApolloConfig 的 AtomicReference ，缓存配置
 */
private volatile AtomicReference<ApolloConfig> m_configCache;
/**
 * Namespace 名字
 */
private final String m_namespace;
/**
 * ScheduledExecutorService 对象
 */
private final static ScheduledExecutorService m_executorService;
/**
 * 指向 ServiceDTO( Config Service 信息) 的 AtomicReference
 */
private AtomicReference<ServiceDTO> m_longPollServiceDto;
/**
 * 指向 ApolloNotificationMessages 的 AtomicReference
 */
private AtomicReference<ApolloNotificationMessages> m_remoteMessages;
/**
 * 加载配置的 RateLimiter
 */
private RateLimiter m_loadConfigRateLimiter;
/**
 * 是否强制拉取缓存的标记
 *
 * 若为 true ，则多一轮从 Config Service 拉取配置
 * 为 true 的原因，RemoteConfigRepository 知道 Config Service 有配置刷新
 */
private AtomicBoolean m_configNeedForceRefresh;
/**
 * 失败定时重试策略，使用 {@link ExponentialSchedulePolicy}
 */
private SchedulePolicy m_loadConfigFailSchedulePolicy;
private Gson gson;
private ConfigUtil m_configUtil;
private HttpUtil m_httpUtil;
private ConfigServiceLocator m_serviceLocator;

static {
    // 单线程池
    m_executorService = Executors.newScheduledThreadPool(1, ApolloThreadFactory.create("RemoteConfigRepository", true));
}

/**
 * Constructor.
 *
 * @param namespace the namespace
 */
public RemoteConfigRepository(String namespace) {
    m_namespace = namespace;
    m_configCache = new AtomicReference<>();
    m_configUtil = ApolloInjector.getInstance(ConfigUtil.class);
    m_httpUtil = ApolloInjector.getInstance(HttpUtil.class);
    m_serviceLocator = ApolloInjector.getInstance(ConfigServiceLocator.class);
    remoteConfigLongPollService = ApolloInjector.getInstance(RemoteConfigLongPollService.class);
    m_longPollServiceDto = new AtomicReference<>();
    m_remoteMessages = new AtomicReference<>();
    m_loadConfigRateLimiter = RateLimiter.create(m_configUtil.getLoadConfigQPS());
    m_configNeedForceRefresh = new AtomicBoolean(true);
    m_loadConfigFailSchedulePolicy = new ExponentialSchedulePolicy(m_configUtil.getOnErrorRetryInterval(), m_configUtil.getOnErrorRetryInterval() * 8);
    gson = new Gson();
    // 尝试同步配置
    this.trySync();
    // 初始化定时刷新配置的任务
    this.schedulePeriodicRefresh();
    // 注册自己到 RemoteConfigLongPollService 中，实现配置更新的实时通知
    this.scheduleLongPollingRefresh();
}
```

* 基础属性
    * `m_namespace` 属性，Namespace 名字。一个 RemoteConfigRepository 对应一个 Namespace 。
    * `m_configCache` 属性，指向 ApolloConfig 的 AtomicReference ，**缓存配置**。
* 轮询属性
    * `m_remoteMessages` 属性，指向 **ApolloNotificationMessages** 的 AtomicReference 。
    * `m_executorService` 属性，ScheduledExecutorService 对象，线程大小为 **1** 。
    * `m_loadConfigRateLimiter` 属性，加载配置的 RateLimiter 。
    * `m_loadConfigFailSchedulePolicy` ，失败定时重试策略，使用 ExponentialSchedulePolicy 实现类，区间范围是 `[1, 8]` 秒。详细解析，见 [「4. SchedulePolicy」](#) 。
* 通知属性
    * `remoteConfigLongPollService` 属性，远程配置长轮询服务。
    * `m_longPollServiceDto` 属性，长轮询到通知的 Config Service 信息。在下一次轮询配置时，优先从该 Config Service 请求。
    * `m_configNeedForceRefresh` 属性，是否强制拉取缓存的标记。
        * 若为 `true` ，则多一轮从 Config Service 拉取配置。
        * 为 `true` 的原因，RemoteConfigRepository 知道 Config Service 有配置刷新，例如有**新的**通知的情况下。
        * 比较绕，下面看了代码会更加好理解。
* **构造方法**
    * 调用 `#trySync()` 方法，尝试同步配置，作为**初次**的配置缓存初始化。
    * 调用 `#schedulePeriodicRefresh()` 方法，初始化定时刷新配置的任务。代码如下：

        ```Java
        private void schedulePeriodicRefresh() {
            logger.debug("Schedule periodic refresh with interval: {} {}", m_configUtil.getRefreshInterval(), m_configUtil.getRefreshIntervalTimeUnit());
            // 创建定时任务，定时刷新配置
            m_executorService.scheduleAtFixedRate(new Runnable() {
                @Override
                public void run() {
                    // 【TODO 6001】Tracer 日志
                    Tracer.logEvent("Apollo.ConfigService", String.format("periodicRefresh: %s", m_namespace));
                    logger.debug("refresh config for namespace: {}", m_namespace);
                    // 尝试同步配置
                    trySync();
                    // 【TODO 6001】Tracer 日志
                    Tracer.logEvent("Apollo.Client.Version", Apollo.VERSION);
                }
            }, m_configUtil.getRefreshInterval(), m_configUtil.getRefreshInterval(), m_configUtil.getRefreshIntervalTimeUnit());
        }
        ```
        * 每 **5 分钟**，调用 `#trySync()` 方法，同步配置。
    * 调用 `#scheduleLongPollingRefresh()` 方法，将**自己**注册到 RemoteConfigLongPollService 中，实现配置更新的实时通知。代码如下：

        ```Java
        private void scheduleLongPollingRefresh() {
            remoteConfigLongPollService.submit(m_namespace, this);
        }
        ```
        * 当 RemoteConfigLongPollService 长轮询到该 RemoteConfigRepository 的 **Namespace** 下的配置更新时，会回调 `#onLongPollNotified(ServiceDTO, ApolloNotificationMessages)` 方法，在 [「2.2.4 onLongPollNotified」](#) 中，详细解析。

### 2.2.2 getConfigServices

`#getConfigServices()` 方法，获得**所有** Config Service 信息。代码如下：

```Java
private List<ServiceDTO> getConfigServices() {
    List<ServiceDTO> services = m_serviceLocator.getConfigServices();
    if (services.size() == 0) {
        throw new ApolloConfigException("No available config service");
    }
    return services;
}
```

* 通过 ConfigServiceLocator ，可获得 Config Service 集群的地址们。在后续和注册发现相关的文章，详细解析 ConfigServiceLocator 。这里，我们只需要有这么一回事即可。

### 2.2.3 assembleQueryConfigUrl

`#assembleQueryConfigUrl()` 方法，组装轮询 Config Service 的配置读取 `/configs/{appId}/{clusterName}/{namespace:.+}` 接口的 URL ，代码如下：

```Java
String assembleQueryConfigUrl(String uri, String appId, String cluster, String namespace,
                              String dataCenter, ApolloNotificationMessages remoteMessages, ApolloConfig previousConfig) {
    String path = "configs/%s/%s/%s"; // /configs/{appId}/{clusterName}/{namespace:.+}
    List<String> pathParams = Lists.newArrayList(pathEscaper.escape(appId), pathEscaper.escape(cluster), pathEscaper.escape(namespace));
    Map<String, String> queryParams = Maps.newHashMap();
    // releaseKey
    if (previousConfig != null) {
        queryParams.put("releaseKey", queryParamEscaper.escape(previousConfig.getReleaseKey()));
    }
    // dataCenter
    if (!Strings.isNullOrEmpty(dataCenter)) {
        queryParams.put("dataCenter", queryParamEscaper.escape(dataCenter));
    }
    // ip
    String localIp = m_configUtil.getLocalIp();
    if (!Strings.isNullOrEmpty(localIp)) {
        queryParams.put("ip", queryParamEscaper.escape(localIp));
    }
    // messages
    if (remoteMessages != null) {
        queryParams.put("messages", queryParamEscaper.escape(gson.toJson(remoteMessages)));
    }
    // 格式化 URL
    String pathExpanded = String.format(path, pathParams.toArray());
    // 拼接 Query String
    if (!queryParams.isEmpty()) {
        pathExpanded += "?" + MAP_JOINER.join(queryParams);
    }
    // 拼接最终的请求 URL
    if (!uri.endsWith("/")) {
        uri += "/";
    }
    return uri + pathExpanded;
}
```

###2.2.4 onLongPollNotified

`#onLongPollNotified(ServiceDTO longPollNotifiedServiceDto, ApolloNotificationMessages remoteMessages)` 方法，当长轮询到配置更新时，发起同步配置的任务。代码如下：

```Java
  1: public void onLongPollNotified(ServiceDTO longPollNotifiedServiceDto, ApolloNotificationMessages remoteMessages) {
  2:     // 设置长轮询到配置更新的 Config Service 。下次同步配置时，优先读取该服务
  3:     m_longPollServiceDto.set(longPollNotifiedServiceDto);
  4:     // 设置 m_remoteMessages
  5:     m_remoteMessages.set(remoteMessages);
  6:     // 提交同步任务
  7:     m_executorService.submit(new Runnable() {
  8:
  9:         @Override
 10:         public void run() {
 11:             // 设置 m_configNeedForceRefresh 为 true
 12:             m_configNeedForceRefresh.set(true);
 13:             // 尝试同步配置
 14:             trySync();
 15:         }
 16:
 17:     });
 18: }
```

* 第 3 行：设置长轮询到配置更新的 Config Service 。下次同步配置时，优先读取该服务。
* 第 5 行：设置 `m_remoteMessages` 。
* 第 6 至 17 行：提交配置同步任务。
    * 第 12 行：设置 `m_configNeedForceRefresh` 为 `true` 。
    * 第 14 行：调用 `#trySync()` 方法，同步配置。

### 2.2.5 sync

`#sync()` **实现**方法，从 Config Service 同步配置。代码如下：

```Java
  1: @Override
  2: protected synchronized void sync() {
  3:     // 【TODO 6001】Tracer 日志
  4:     Transaction transaction = Tracer.newTransaction("Apollo.ConfigService", "syncRemoteConfig");
  5:     try {
  6:         // 获得缓存的 ApolloConfig 对象
  7:         ApolloConfig previous = m_configCache.get();
  8:         // 从 Config Service 加载 ApolloConfig 对象
  9:         ApolloConfig current = loadApolloConfig();
 10:
 11:         // reference equals means HTTP 304
 12:         // 若不相等，说明更新了，设置到缓存中
 13:         if (previous != current) {
 14:             logger.debug("Remote Config refreshed!");
 15:             // 设置到缓存
 16:             m_configCache.set(current);
 17:             // 发布 Repository 的配置发生变化，触发对应的监听器们
 18:             super.fireRepositoryChange(m_namespace, this.getConfig());
 19:         }
 20:         // 【TODO 6001】Tracer 日志
 21:         if (current != null) {
 22:             Tracer.logEvent(String.format("Apollo.Client.Configs.%s", current.getNamespaceName()), current.getReleaseKey());
 23:         }
 24:         // 【TODO 6001】Tracer 日志
 25:         transaction.setStatus(Transaction.SUCCESS);
 26:     } catch (Throwable ex) {
 27:         // 【TODO 6001】Tracer 日志
 28:         transaction.setStatus(ex);
 29:         throw ex;
 30:     } finally {
 31:         // 【TODO 6001】Tracer 日志
 32:         transaction.complete();
 33:     }
 34: }
```

* 第 7 行：获得缓存 `m_configCache` 的 ApolloConfig 对象。
* 第 9 行：调用 `#loadApolloConfig()` 方法，从 Config Service 加载 ApolloConfig 对象。详细解析，见
* 第 13 至 19 行：若缓存的和加载的 ApolloConfig 对象不同，说明更新了，设置缓存中。
    * 第 16 行：设置加载到的  ApolloConfig 到缓存 `m_configCache` 中。
    * 第 18 行：调用 `#fireRepositoryChange(m_namespace, ApolloConfig)` 方法，发布 Repository 的**配置发生变化**，触发对应的监听器们。。详细解析，见 [「2.2.6  loadApolloConfig」](#) 。

### 2.2.6 loadApolloConfig

`#loadApolloConfig()` 方法，从 Config Service 加载 ApolloConfig 对象。代码如下：

```Java
  1: private ApolloConfig loadApolloConfig() {
  2:     // 限流
  3:     if (!m_loadConfigRateLimiter.tryAcquire(5, TimeUnit.SECONDS)) {
  4:         // wait at most 5 seconds
  5:         try {
  6:             TimeUnit.SECONDS.sleep(5);
  7:         } catch (InterruptedException e) {
  8:         }
  9:     }
 10:     // 获得 appId cluster dataCenter 配置信息
 11:     String appId = m_configUtil.getAppId();
 12:     String cluster = m_configUtil.getCluster();
 13:     String dataCenter = m_configUtil.getDataCenter();
 14:     Tracer.logEvent("Apollo.Client.ConfigMeta", STRING_JOINER.join(appId, cluster, m_namespace));
 15:     // 计算重试次数
 16:     int maxRetries = m_configNeedForceRefresh.get() ? 2 : 1;
 17:     long onErrorSleepTime = 0; // 0 means no sleep
 18:     Throwable exception = null;
 19:     // 获得所有的 Config Service 的地址
 20:     List<ServiceDTO> configServices = getConfigServices();
 21:     String url = null;
 22:     // 循环读取配置重试次数直到成功。每一次，都会循环所有的 ServiceDTO 数组。
 23:     for (int i = 0; i < maxRetries; i++) {
 24:         // 随机所有的 Config Service 的地址
 25:         List<ServiceDTO> randomConfigServices = Lists.newLinkedList(configServices);
 26:         Collections.shuffle(randomConfigServices);
 27:         // 优先访问通知配置变更的 Config Service 的地址。并且，获取到时，需要置空，避免重复优先访问。
 28:         // Access the server which notifies the client first
 29:         if (m_longPollServiceDto.get() != null) {
 30:             randomConfigServices.add(0, m_longPollServiceDto.getAndSet(null));
 31:         }
 32:         // 循环所有的 Config Service 的地址
 33:         for (ServiceDTO configService : randomConfigServices) {
 34:             // sleep 等待，下次从 Config Service 拉取配置
 35:             if (onErrorSleepTime > 0) {
 36:                 logger.warn("Load config failed, will retry in {} {}. appId: {}, cluster: {}, namespaces: {}", onErrorSleepTime, m_configUtil.getOnErrorRetryIntervalTimeUnit(), appId, cluster, m_namespace);
 37:                 try {
 38:                     m_configUtil.getOnErrorRetryIntervalTimeUnit().sleep(onErrorSleepTime);
 39:                 } catch (InterruptedException e) {
 40:                     //ignore
 41:                 }
 42:             }
 43:             // 组装查询配置的地址
 44:             url = assembleQueryConfigUrl(configService.getHomepageUrl(), appId, cluster, m_namespace, dataCenter, m_remoteMessages.get(), m_configCache.get());
 45:
 46:             logger.debug("Loading config from {}", url);
 47:             // 创建 HttpRequest 对象
 48:             HttpRequest request = new HttpRequest(url);
 49:
 50:             // 【TODO 6001】Tracer 日志
 51:             Transaction transaction = Tracer.newTransaction("Apollo.ConfigService", "queryConfig");
 52:             transaction.addData("Url", url);
 53:             try {
 54:                 // 发起请求，返回 HttpResponse 对象
 55:                 HttpResponse<ApolloConfig> response = m_httpUtil.doGet(request, ApolloConfig.class);
 56:                 // 设置 m_configNeedForceRefresh = false
 57:                 m_configNeedForceRefresh.set(false);
 58:                 // 标记成功
 59:                 m_loadConfigFailSchedulePolicy.success();
 60:
 61:                 // 【TODO 6001】Tracer 日志
 62:                 transaction.addData("StatusCode", response.getStatusCode());
 63:                 transaction.setStatus(Transaction.SUCCESS);
 64:
 65:                 // 无新的配置，直接返回缓存的 ApolloConfig 对象
 66:                 if (response.getStatusCode() == 304) {
 67:                     logger.debug("Config server responds with 304 HTTP status code.");
 68:                     return m_configCache.get();
 69:                 }
 70:
 71:                 // 有新的配置，进行返回新的 ApolloConfig 对象
 72:                 ApolloConfig result = response.getBody();
 73:                 logger.debug("Loaded config for {}: {}", m_namespace, result);
 74:                 return result;
 75:             } catch (ApolloConfigStatusCodeException ex) {
 76:                 ApolloConfigStatusCodeException statusCodeException = ex;
 77:                 // 若返回的状态码是 404 ，说明查询配置的 Config Service 不存在该 Namespace 。
 78:                 // config not found
 79:                 if (ex.getStatusCode() == 404) {
 80:                     String message = String.format("Could not find config for namespace - appId: %s, cluster: %s, namespace: %s, " +
 81:                                     "please check whether the configs are released in Apollo!", appId, cluster, m_namespace);
 82:                     statusCodeException = new ApolloConfigStatusCodeException(ex.getStatusCode(), message);
 83:                 }
 84:                 // 【TODO 6001】Tracer 日志
 85:                 Tracer.logEvent("ApolloConfigException", ExceptionUtil.getDetailMessage(statusCodeException));
 86:                 transaction.setStatus(statusCodeException);
 87:                 // 设置最终的异常
 88:                 exception = statusCodeException;
 89:             } catch (Throwable ex) {
 90:                 // 【TODO 6001】Tracer 日志
 91:                 Tracer.logEvent("ApolloConfigException", ExceptionUtil.getDetailMessage(ex));
 92:                 transaction.setStatus(ex);
 93:                 // 设置最终的异常
 94:                 exception = ex;
 95:             } finally {
 96:                 // 【TODO 6001】Tracer 日志
 97:                 transaction.complete();
 98:             }
 99:             // 计算延迟时间
100:             // if force refresh, do normal sleep, if normal config load, do exponential sleep
101:             onErrorSleepTime = m_configNeedForceRefresh.get() ? m_configUtil.getOnErrorRetryInterval() : m_loadConfigFailSchedulePolicy.fail();
102:         }
103:
104:     }
105:     // 若查询配置失败，抛出 ApolloConfigException 异常
106:     String message = String.format("Load Apollo Config failed - appId: %s, cluster: %s, namespace: %s, url: %s", appId, cluster, m_namespace, url);
107:     throw new ApolloConfigException(message, exception);
108: }
```
* 第 3 至 9 行：调用 `RateLimiter#tryAcquire(long timeout, TimeUnit unit)` 方法，判断是否被限流。若限流，**sleep** 5 秒，避免对 Config Service 请求过于频繁。
* 第 10 至 13 行：获得 `appId` `cluster` `dataCenter` 配置信息。
* 第 16 行：计算重试次数。若 `m_configNeedForceRefresh` 为 `true` ，代表**强制**刷新配置，会多重试一次。
* 第 20 行：调用 `#getConfigServices()` 方法，获得**所有**的 Config Service 的地址。
* ========== **第一层**循环 ==========
* 第 23 至 104 行：**循环**读取配置重试次数直到成功。**每一次，都会循环所有的 ServiceDTO 数组**。
* 第 24 至 26 行：创建新的 Config Service 数组，并随机打乱。因为【第 29 至 31 行】可能会添加新的 Config Service 元素，如果不创建新的数组，会修改了原有数组。
* 第 27 至 31 行：若 `m_longPollServiceDto` 非空，**优先**访问通知配置变更的 Config Service 的地址。并且，获取到时，需要置空，避免重复优先访问。
* ========== **第二层**循环 ==========
* 第 33 至 102 行：**循环**所有的 Config Service 的地址，读取配置重试次数直到成功。
* 第 34 至 42 行：若 `onErrorSleepTime` 大于零，**sleep** 等待，下次从 Config Service 拉取配置。在【第 101 行】，若请求失败一次 Config Service 时，会计算一次下一次请求的**延迟时间**。因为是**每次**请求失败一次 Config Service 时就计算一次，所以**延迟时间**的上限为 8 秒，比较短。
* 第 44 行：调用 `#assembleQueryConfigUrl(...)` 方法，组装查询配置的地址。
* 第 48 行：创建 HttpRequest 对象。
* 第 55 行：调用 `HttpUtil#doGet(request, Class)` 方法，发起请求，返回 HttpResponse 对象。
* 第 57 行：设置 `m_configNeedForceRefresh` 为 `false` 。
* 第 59 行：调用 `SchedulePolicy#success()` 方法，标记成功。
* 第 65 至 69 行：若返回的状态码是 **304** ，**无**新的配置，直接返回**缓存的** ApolloConfig 对象。
* 第 71 至 74 行：**有**新的配置，创建 ApolloConfig 对象，并返回。
* 第 75 至 94 行：异常相关的处理，胖友自己看注释。
* 第 101 行：计算延迟时间。若 `m_configNeedForceRefresh` 为：
    *  `true` 时，调用 `ConfigUtil#getOnErrorRetryInterval()` 方法，返回 2 **秒**。因为已经知道有配置更新，所以减短重试间隔。
    * `false` 时，调用 `SchedulePolicy#fail()` 方法，计算下次重试延迟时间。
* ========== **最外层** ==========
* 第 105 至 107 行：若查询配置失败，抛出 ApolloConfigException 异常。

### 2.2.7 getConfig

`#getConfig()` **实现**方法，获得配置。代码如下：

```Java
  1: @Override
  2: public Properties getConfig() {
  3:     // 如果缓存为空，强制从 Config Service 拉取配置
  4:     if (m_configCache.get() == null) {
  5:         this.sync();
  6:     }
  7:     // 转换成 Properties 对象，并返回
  8:     return transformApolloConfigToProperties(m_configCache.get());
  9: }
```

* 第 3 至 6 行：若果缓存为空，调用 `#sync()` 方法，强制从 Config Service 拉取配置。
* 第 8 行：调用 `#transformApolloConfigToProperties(ApolloConfig)` 对象，转换成 Properties 对象，并返回。代码如下：

    ```Java
    private Properties transformApolloConfigToProperties(ApolloConfig apolloConfig) {
        Properties result = new Properties();
        result.putAll(apolloConfig.getConfigurations());
        return result;
    }
    ```

# 3. RemoteConfigLongPollService

`com.ctrip.framework.apollo.internals.RemoteConfigLongPollService` ，远程配置长轮询服务。负责长轮询 Config Service 的配置变更通知 `/notifications/v2` 接口。当有**新的**通知时，触发 RemoteConfigRepository ，**立即**轮询 Config Service 的配置读取 `/configs/{appId}/{clusterName}/{namespace:.+}` 接口。

## 3.1 构造方法

```Java
private static final Logger logger = LoggerFactory.getLogger(RemoteConfigLongPollService.class);

private static final Joiner STRING_JOINER = Joiner.on(ConfigConsts.CLUSTER_NAMESPACE_SEPARATOR);
private static final Joiner.MapJoiner MAP_JOINER = Joiner.on("&").withKeyValueSeparator("=");
private static final Escaper queryParamEscaper = UrlEscapers.urlFormParameterEscaper();

private static final long INIT_NOTIFICATION_ID = ConfigConsts.NOTIFICATION_ID_PLACEHOLDER;
// 90 seconds, should be longer than server side's long polling timeout, which is now 60 seconds
private static final int LONG_POLLING_READ_TIMEOUT = 90 * 1000;

/**
 * 长轮询 ExecutorService
 */
private final ExecutorService m_longPollingService;
/**
 * 是否停止长轮询的标识
 */
private final AtomicBoolean m_longPollingStopped;
/**
 * 失败定时重试策略，使用 {@link ExponentialSchedulePolicy}
 */
private SchedulePolicy m_longPollFailSchedulePolicyInSecond;
/**
 * 长轮询的 RateLimiter
 */
private RateLimiter m_longPollRateLimiter;
/**
 * 是否长轮询已经开始的标识
 */
private final AtomicBoolean m_longPollStarted;
/**
 * 长轮询的 Namespace Multimap 缓存
 *
 * 通过 {@link #submit(String, RemoteConfigRepository)} 添加 RemoteConfigRepository 。
 *
 * KEY：Namespace 的名字
 * VALUE：RemoteConfigRepository 集合
 */
private final Multimap<String, RemoteConfigRepository> m_longPollNamespaces;
/**
 * 通知编号 Map 缓存
 *
 * KEY：Namespace 的名字
 * VALUE：最新的通知编号
 */
private final ConcurrentMap<String, Long> m_notifications;
/**
 * 通知消息 Map 缓存
 *
 * KEY：Namespace 的名字
 * VALUE：ApolloNotificationMessages 对象
 */
private final Map<String, ApolloNotificationMessages> m_remoteNotificationMessages;//namespaceName -> watchedKey -> notificationId
private Type m_responseType;
private Gson gson;
private ConfigUtil m_configUtil;
private HttpUtil m_httpUtil;
private ConfigServiceLocator m_serviceLocator;

public RemoteConfigLongPollService() {
    m_longPollFailSchedulePolicyInSecond = new ExponentialSchedulePolicy(1, 120); //in second
    m_longPollingStopped = new AtomicBoolean(false);
    m_longPollingService = Executors.newSingleThreadExecutor(ApolloThreadFactory.create("RemoteConfigLongPollService", true));
    m_longPollStarted = new AtomicBoolean(false);
    m_longPollNamespaces = Multimaps.synchronizedSetMultimap(HashMultimap.<String, RemoteConfigRepository>create());
    m_notifications = Maps.newConcurrentMap();
    m_remoteNotificationMessages = Maps.newConcurrentMap();
    m_responseType = new TypeToken<List<ApolloConfigNotification>>() {}.getType();
    gson = new Gson();
    m_configUtil = ApolloInjector.getInstance(ConfigUtil.class);
    m_httpUtil = ApolloInjector.getInstance(HttpUtil.class);
    m_serviceLocator = ApolloInjector.getInstance(ConfigServiceLocator.class);
    m_longPollRateLimiter = RateLimiter.create(m_configUtil.getLongPollQPS());
}
```

* 基础属性
    * `m_longPollNamespaces` 属性，**注册的**长轮询的 Namespace Multimap 缓存。
    * `m_notifications` 属性，通知**编号** Map 缓存。
    * `m_remoteNotificationMessages` 属性，通知**消息** Map 缓存。
* 轮询属性
    * `m_longPollingService` 属性，长轮询 ExecutorService ，线程大小为 **1** 。
    * `m_longPollingStopped` 属性，是否**停止**长轮询的标识。
    * `m_longPollStarted` 属性，是否长轮询已经**开始**的标识。
    * `m_loadConfigRateLimiter` 属性，加载配置的 RateLimiter 。
    * `m_longPollFailSchedulePolicyInSecond` ，失败定时重试策略，使用 ExponentialSchedulePolicy 实现类，区间范围是 `[1, 120]` 秒。详细解析，见 [「4. SchedulePolicy」](#)  。

## 3.2 getConfigServices

同 [「2.2 getConfigServices」](#)  的代码。

## 3.3 assembleLongPollRefreshUrl

`#assembleLongPollRefreshUrl(...)` 方法，**长**轮询 Config Service 的配置变更通知 `/notifications/v2` 接口的 URL ，代码如下：

```Java
String assembleLongPollRefreshUrl(String uri, String appId, String cluster, String dataCenter, Map<String, Long> notificationsMap) {
    Map<String, String> queryParams = Maps.newHashMap();
    queryParams.put("appId", queryParamEscaper.escape(appId));
    queryParams.put("cluster", queryParamEscaper.escape(cluster));
    // notifications
    queryParams.put("notifications", queryParamEscaper.escape(assembleNotifications(notificationsMap)));
    // dataCenter
    if (!Strings.isNullOrEmpty(dataCenter)) {
        queryParams.put("dataCenter", queryParamEscaper.escape(dataCenter));
    }
    // ip
    String localIp = m_configUtil.getLocalIp();
    if (!Strings.isNullOrEmpty(localIp)) {
        queryParams.put("ip", queryParamEscaper.escape(localIp));
    }
    // 创建 Query String
    String params = MAP_JOINER.join(queryParams);
    // 拼接 URL
    if (!uri.endsWith("/")) {
        uri += "/";
    }
    return uri + "notifications/v2?" + params;
}

String assembleNotifications(Map<String, Long> notificationsMap) {
    // 创建 ApolloConfigNotification 数组
    List<ApolloConfigNotification> notifications = Lists.newArrayList();
    // 循环，添加 ApolloConfigNotification 对象
    for (Map.Entry<String, Long> entry : notificationsMap.entrySet()) {
        ApolloConfigNotification notification = new ApolloConfigNotification(entry.getKey(), entry.getValue());
        notifications.add(notification);
    }
    // JSON 化成字符串
    return gson.toJson(notifications);
}
```

## 3.4 submit

`#submit(namespace, RemoteConfigRepository)` 方法，提交 **RemoteConfigRepository** 到长轮询任务。代码如下：

```Java
  1: public boolean submit(String namespace, RemoteConfigRepository remoteConfigRepository) {
  2:     // 添加到 m_longPollNamespaces 中
  3:     boolean added = m_longPollNamespaces.put(namespace, remoteConfigRepository);
  4:     // 添加到 m_notifications 中
  5:     m_notifications.putIfAbsent(namespace, INIT_NOTIFICATION_ID);
  6:     // 若未启动长轮询定时任务，进行启动
  7:     if (!m_longPollStarted.get()) {
  8:         startLongPolling();
  9:     }
 10:     return added;
 11: }
```

* 第 3 行：添加到 `m_longPollNamespaces.put` 中。
* 第 5 行：添加到 `m_notifications` 中。
* 第 6 至 9 行：若**未启动**长轮询定时任务，调用 `#startLongPolling()` 方法，进行启动。

## 3.5 startLongPolling

`#startLongPolling()` 方法，启动长轮询任务。代码如下：

```Java
  1: private void startLongPolling() {
  2:     // CAS 设置长轮询任务已经启动。若已经启动，不重复启动。
  3:     if (!m_longPollStarted.compareAndSet(false, true)) {
  4:         //already started
  5:         return;
  6:     }
  7:     try {
  8:         // 获得 appId cluster dataCenter 配置信息
  9:         final String appId = m_configUtil.getAppId();
 10:         final String cluster = m_configUtil.getCluster();
 11:         final String dataCenter = m_configUtil.getDataCenter();
 12:         // 获得长轮询任务的初始化延迟时间，单位毫秒。
 13:         final long longPollingInitialDelayInMills = m_configUtil.getLongPollingInitialDelayInMills();
 14:         // 提交长轮询任务。该任务会持续且循环执行。
 15:         m_longPollingService.submit(new Runnable() {
 16:             @Override
 17:             public void run() {
 18:                 // 初始等待
 19:                 if (longPollingInitialDelayInMills > 0) {
 20:                     try {
 21:                         logger.debug("Long polling will start in {} ms.", longPollingInitialDelayInMills);
 22:                         TimeUnit.MILLISECONDS.sleep(longPollingInitialDelayInMills);
 23:                     } catch (InterruptedException e) {
 24:                         //ignore
 25:                     }
 26:                 }
 27:                 // 执行长轮询
 28:                 doLongPollingRefresh(appId, cluster, dataCenter);
 29:             }
 30:         });
 31:     } catch (Throwable ex) {
 32:         // 设置 m_longPollStarted 为 false
 33:         m_longPollStarted.set(false);
 34:         // 【TODO 6001】Tracer 日志
 35:         ApolloConfigException exception = new ApolloConfigException("Schedule long polling refresh failed", ex);
 36:         Tracer.logError(exception);
 37:         logger.warn(ExceptionUtil.getDetailMessage(exception));
 38:     }
 39: }
```

* 第 2 至 6 行：**CAS** 设置长轮询任务已经启动。若已经启动，不重复启动。
* 第 8 至 11 行：获得 `appId` `cluster` `dataCenter` 配置信息。
* 第 13 行：调用 `ConfigUtil#getLongPollingInitialDelayInMills()` 方法，获得长轮询任务的初始化延迟时间，单位毫秒。默认，2000 毫秒。
* 第 14 至 30 行：提交长轮询任务。该任务会持续且循环执行。
    * 第 18 至 26 行：**sleep** ，初始等待。
    * 第 28 行：调用 `#doLongPollingRefresh(appId, cluster, dataCenter)` 方法，执行长轮询任务。
* 第 31 至 38 行：初始化失败的异常处理，胖友自己看代码注释。

## 3.6 doLongPollingRefresh

`#doLongPollingRefresh()` 方法，**持续**执行长轮询。代码如下：

```Java
  1: private void doLongPollingRefresh(String appId, String cluster, String dataCenter) {
  2:     final Random random = new Random();
  3:     ServiceDTO lastServiceDto = null;
  4:     // 循环执行，直到停止或线程中断
  5:     while (!m_longPollingStopped.get() && !Thread.currentThread().isInterrupted()) {
  6:         // 限流
  7:         if (!m_longPollRateLimiter.tryAcquire(5, TimeUnit.SECONDS)) {
  8:             // wait at most 5 seconds
  9:             try {
 10:                 TimeUnit.SECONDS.sleep(5);
 11:             } catch (InterruptedException e) {
 12:             }
 13:         }
 14:         // 【TODO 6001】Tracer 日志
 15:         Transaction transaction = Tracer.newTransaction("Apollo.ConfigService", "pollNotification");
 16:         String url = null;
 17:         try {
 18:             // 获得 Config Service 的地址
 19:             if (lastServiceDto == null) {
 20:                 // 获得所有的 Config Service 的地址
 21:                 List<ServiceDTO> configServices = getConfigServices();
 22:                 lastServiceDto = configServices.get(random.nextInt(configServices.size()));
 23:             }
 24:             // 组装长轮询通知变更的地址
 25:             url = assembleLongPollRefreshUrl(lastServiceDto.getHomepageUrl(), appId, cluster, dataCenter, m_notifications);
 26: 
 27:             logger.debug("Long polling from {}", url);
 28:             // 创建 HttpRequest 对象，并设置超时时间
 29:             HttpRequest request = new HttpRequest(url);
 30:             request.setReadTimeout(LONG_POLLING_READ_TIMEOUT);
 31: 
 32:             // 【TODO 6001】Tracer 日志
 33:             transaction.addData("Url", url);
 34: 
 35:             // 发起请求，返回 HttpResponse 对象
 36:             final HttpResponse<List<ApolloConfigNotification>> response = m_httpUtil.doGet(request, m_responseType);
 37:             logger.debug("Long polling response: {}, url: {}", response.getStatusCode(), url);
 38: 
 39:             // 有新的通知，刷新本地的缓存
 40:             if (response.getStatusCode() == 200 && response.getBody() != null) {
 41:                 // 更新 m_notifications
 42:                 updateNotifications(response.getBody());
 43:                 // 更新 m_remoteNotificationMessages
 44:                 updateRemoteNotifications(response.getBody());
 45:                 // 【TODO 6001】Tracer 日志
 46:                 transaction.addData("Result", response.getBody().toString());
 47:                 // 通知对应的 RemoteConfigRepository 们
 48:                 notify(lastServiceDto, response.getBody());
 49:             }
 50: 
 51:             // 无新的通知，重置连接的 Config Service 的地址，下次请求不同的 Config Service ，实现负载均衡。
 52:             // try to load balance
 53:             if (response.getStatusCode() == 304 && random.nextBoolean()) { // 随机
 54:                 lastServiceDto = null;
 55:             }
 56:             // 标记成功
 57:             m_longPollFailSchedulePolicyInSecond.success();
 58:             // 【TODO 6001】Tracer 日志
 59:             transaction.addData("StatusCode", response.getStatusCode());
 60:             transaction.setStatus(Transaction.SUCCESS);
 61:         } catch (Throwable ex) {
 62:             // 重置连接的 Config Service 的地址，下次请求不同的 Config Service
 63:             lastServiceDto = null;
 64:             // 【TODO 6001】Tracer 日志
 65:             Tracer.logEvent("ApolloConfigException", ExceptionUtil.getDetailMessage(ex));
 66:             transaction.setStatus(ex);
 67:             // 标记失败，计算下一次延迟执行时间
 68:             long sleepTimeInSecond = m_longPollFailSchedulePolicyInSecond.fail();
 69:             logger.warn("Long polling failed, will retry in {} seconds. appId: {}, cluster: {}, namespaces: {}, long polling url: {}, reason: {}",
 70:                     sleepTimeInSecond, appId, cluster, assembleNamespaces(), url, ExceptionUtil.getDetailMessage(ex));
 71:             // 等待一定时间，下次失败重试
 72:             try {
 73:                 TimeUnit.SECONDS.sleep(sleepTimeInSecond);
 74:             } catch (InterruptedException ie) {
 75:                 //ignore
 76:             }
 77:         } finally {
 78:             transaction.complete();
 79:         }
 80:     }
 81: }
```

* 第 5 至 80 行：**循环**执行，直到停止或线程中断。
* 第 7 至 13 行：调用 `RateLimiter#tryAcquire(long timeout, TimeUnit unit)` 方法，判断是否被限流。若限流，**sleep** 5 秒，避免对 Config Service 请求过于频繁。
* 第 19 至 23 行：若无 `lastServiceDto` 对象，随机获得 Config Service 的地址。
* 第 25 行：调用 `#assembleLongPollRefreshUrl(...)` 方法，组装长轮询通知变更的地址。
* 第 29 至 30 行：创建 HttpRequest 对象，并设置超时时间。默认超时时间为 90 秒，**大于** Config Service 的通知接口的 60 秒。
* 第 36 行：调用 `HttpUtil#doGet(request, Class)` 方法，发起请求，返回 HttpResponse 对象。
* 第 40 至 49 行：若返回状态码为 **200** ，说明有**新的**通知，刷新本地的缓存。
    * 第 42 行：调用 `#updateNotifications(List<ApolloConfigNotification>)` 方法，更新 `m_notifications` 。详细解析，在 [「3.7 updateNotifications」](#) 。
    * 第 44 行：调用 `#updateRemoteNotifications(List<ApolloConfigNotification>)` 方法，更新 `m_remoteNotificationMessages` 。详细解析，在 [「3.8 updateRemoteNotifications」](#) 。
    * 第 48 行：调用 `#notify(ServiceDTO, List<ApolloConfigNotification>)` 方法，通知对应的 **RemoteConfigRepository** 们。详细解析，在 [「3.9 notify」](#) 。
* 第 51 至 55 行：若返回状态码为 **304** ，说明无**新的**通知，**随机**，重置连接的 Config Service 的地址，下次请求不同的 Config Service ，实现负载均衡。
* 第 57 行：调用 `SchedulePolicy#success()` 方法，标记成功。
* 第 61 至 76 行：处理异常。
    * 第 63 行：重置连接的 Config Service 的地址 `lastServiceDto` ，下次请求不同的 Config Service。
    * 第 67 至 70 行：调用 `SchedulePolicy#fail()` 方法，标记失败，计算下一次延迟执行时间。
    * 第 71 至 76 行：**sleep**，等待一定时间，下次失败重试。

## 3.7 updateNotifications

`#updateNotifications(List<ApolloConfigNotification>)` 方法，更新 `m_notifications` 。代码如下：

```Java
private void updateNotifications(List<ApolloConfigNotification> deltaNotifications) {
    // 循环 ApolloConfigNotification
    for (ApolloConfigNotification notification : deltaNotifications) {
        if (Strings.isNullOrEmpty(notification.getNamespaceName())) {
            continue;
        }
        // 更新 m_notifications
        String namespaceName = notification.getNamespaceName();
        if (m_notifications.containsKey(namespaceName)) {
            m_notifications.put(namespaceName, notification.getNotificationId());
        }
        // 因为 .properties 在默认情况下被过滤掉，所以我们需要检查是否有 .properties 后缀的通知。如有，更新 m_notifications
        // since .properties are filtered out by default, so we need to check if there is notification with .properties suffix
        String namespaceNameWithPropertiesSuffix = String.format("%s.%s", namespaceName, ConfigFileFormat.Properties.getValue());
        if (m_notifications.containsKey(namespaceNameWithPropertiesSuffix)) {
            m_notifications.put(namespaceNameWithPropertiesSuffix, notification.getNotificationId());
        }
    }
}
```

## 3.8 updateRemoteNotifications

`#updateRemoteNotifications(List<ApolloConfigNotification>)` 方法，更新 `m_remoteNotificationMessages` 。代码如下：

```Java
private void updateRemoteNotifications(List<ApolloConfigNotification> deltaNotifications) {
    // 循环 ApolloConfigNotification
    for (ApolloConfigNotification notification : deltaNotifications) {
        if (Strings.isNullOrEmpty(notification.getNamespaceName())) {
            continue;
        }
        if (notification.getMessages() == null || notification.getMessages().isEmpty()) {
            continue;
        }
        // 若不存在 Namespace 对应的 ApolloNotificationMessages ，进行创建
        ApolloNotificationMessages localRemoteMessages = m_remoteNotificationMessages.get(notification.getNamespaceName());
        if (localRemoteMessages == null) {
            localRemoteMessages = new ApolloNotificationMessages();
            m_remoteNotificationMessages.put(notification.getNamespaceName(), localRemoteMessages);
        }
        // 合并通知消息到 ApolloNotificationMessages 中
        localRemoteMessages.mergeFrom(notification.getMessages());
    }
}
```

## 3.9 notify

`#notify(ServiceDTO, List<ApolloConfigNotification>)` 方法，更新 `m_remoteNotificationMessages` 。代码如下：

```Java
private void notify(ServiceDTO lastServiceDto, List<ApolloConfigNotification> notifications) {
    if (notifications == null || notifications.isEmpty()) {
        return;
    }
    // 循环 ApolloConfigNotification
    for (ApolloConfigNotification notification : notifications) {
        String namespaceName = notification.getNamespaceName(); // Namespace 的名字
        // 创建 RemoteConfigRepository 数组，避免并发问题
        // create a new list to avoid ConcurrentModificationException
        List<RemoteConfigRepository> toBeNotified = Lists.newArrayList(m_longPollNamespaces.get(namespaceName));
        // 因为 .properties 在默认情况下被过滤掉，所以我们需要检查是否有监听器。若有，添加到 RemoteConfigRepository 数组
        // since .properties are filtered out by default, so we need to check if there is any listener for it
        toBeNotified.addAll(m_longPollNamespaces.get(String.format("%s.%s", namespaceName, ConfigFileFormat.Properties.getValue())));
        // 获得远程的 ApolloNotificationMessages 对象，并克隆
        ApolloNotificationMessages originalMessages = m_remoteNotificationMessages.get(namespaceName);
        ApolloNotificationMessages remoteMessages = originalMessages == null ? null : originalMessages.clone();
        // 循环 RemoteConfigRepository ，进行通知
        for (RemoteConfigRepository remoteConfigRepository : toBeNotified) {
            try {
                // 进行通知
                remoteConfigRepository.onLongPollNotified(lastServiceDto, remoteMessages);
            } catch (Throwable ex) {
                // 【TODO 6001】Tracer 日志
                Tracer.logError(ex);
            }
        }
    }
}
```



# 4. SchedulePolicy

`com.ctrip.framework.apollo.core.schedule.SchedulePolicy` ，定时策略接口。在 Apollo 中，用于执行失败，计算下一次执行的延迟时间。代码如下：

```Java
public interface SchedulePolicy {

    /**
     * 执行失败
     *
     * @return 下次执行延迟
     */
    long fail();

    /**
     * 执行成功
     */
    void success();

}
```

## 4.1 ExponentialSchedulePolicy

`com.ctrip.framework.apollo.core.schedule.ExponentialSchedulePolicy` ，实现 SchedulePolicy 接口，基于**指数级计算**的定时策略实现类。代码如下：

```Java
public class ExponentialSchedulePolicy implements SchedulePolicy {

    /**
     * 延迟时间下限
     */
    private final long delayTimeLowerBound;
    /**
     * 延迟时间上限
     */
    private final long delayTimeUpperBound;
    /**
     * 最后延迟执行时间
     */
    private long lastDelayTime;

    public ExponentialSchedulePolicy(long delayTimeLowerBound, long delayTimeUpperBound) {
        this.delayTimeLowerBound = delayTimeLowerBound;
        this.delayTimeUpperBound = delayTimeUpperBound;
    }

    @Override
    public long fail() {
        long delayTime = lastDelayTime;
        // 设置初始时间
        if (delayTime == 0) {
            delayTime = delayTimeLowerBound;
        // 指数级计算，直到上限
        } else {
            delayTime = Math.min(lastDelayTime << 1, delayTimeUpperBound);
        }
        // 最后延迟执行时间
        lastDelayTime = delayTime;
        // 返回
        return delayTime;
    }

    @Override
    public void success() {
        lastDelayTime = 0;
    }

    public static void main(String[] args) {
        ExponentialSchedulePolicy policy = new ExponentialSchedulePolicy(1, 120);
        for (int i = 0; i < 10; i++) {
            System.out.println(policy.fail());
        }
    }

}
```

* 每次执行失败，调用 `#fail()` 方法，指数级计算新的延迟执行时间。
* 举例如下：

    ```Java
    delayTimeLowerBound, delayTimeUpperBound= [1, 120] 执行 10 轮
    1 2 4 8 16 32 64 120 120 120
    delayTimeLowerBound, delayTimeUpperBound= [30, 120] 执行 10 轮
    30 60 120 120 120 120 120 120 120 120 120 120
    ```

# 666. 彩蛋

😈 整个配置加载的流程完成！！！我的天！！！

充实的周天。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)



