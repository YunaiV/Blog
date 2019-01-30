title: Eureka 源码解析 —— Eureka-Client 初始化（三）之 EurekaClient
date: 2018-04-29
tags:
categories: Eureka
permalink: Eureka/eureka-client-init-third

---

摘要: 原创出处 http://www.iocoder.cn/Eureka/eureka-client-init-third/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Eureka 1.8.X 版本** 

- [1. 概述](http://www.iocoder.cn/Eureka/eureka-client-init-third/)
- [2. EurekaClient](http://www.iocoder.cn/Eureka/eureka-client-init-third/)
  - [2.1 LookupService](http://www.iocoder.cn/Eureka/eureka-client-init-third/)
- [3. DiscoveryClient](http://www.iocoder.cn/Eureka/eureka-client-init-third/)
  - [3.1 构造方法参数](http://www.iocoder.cn/Eureka/eureka-client-init-third/)
  - [3.2 构造方法](http://www.iocoder.cn/Eureka/eureka-client-init-third/)
- [666. 彩蛋](http://www.iocoder.cn/Eureka/eureka-client-init-third/)

---

![](http://www.iocoder.cn/images/common/wechat_mp_2018_05_18.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

---

# 1. 概述

本文接[《Eureka 源码解析 —— Eureka-Client 初始化（二）之 EurekaClientConfig》](http://www.iocoder.cn/Eureka/eureka-client-init-second/?self)，主要分享 **Eureka-Client 自身初始化的过程**的第三部分 —— **EurekaClient**，不包含 Eureka-Client 向 Eureka-Server 的注册过程( 🙂后面会另外文章分享 )。

Eureka-Client 自身初始化过程中，涉及到主要对象如下图：

![](http://www.iocoder.cn/images/Eureka/2018_04_15/01.png)

1. **创建** EurekaInstanceConfig对象
1. 使用 EurekaInstanceConfig对象 **创建** InstanceInfo对象
1. 使用 EurekaInstanceConfig对象 + InstanceInfo对象 **创建** ApplicationInfoManager对象
1. **创建** EurekaClientConfig对象
1. 使用 ApplicationInfoManager对象 + EurekaClientConfig对象 **创建** EurekaClient对象

考虑到整个初始化的过程中涉及的配置特别多，拆分成三篇文章：

1. （一）[EurekaInstanceConfig]((http://www.iocoder.cn/Eureka/eureka-client-init-first/))
2. （二）[EurekaClientConfig](http://www.iocoder.cn/Eureka/eureka-client-init-second/)
3. **【本文】**（三）EurekaClient

下面我们来看看每个**类**的实现。

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



# 2. EurekaClient

[`com.netflix.discovery.EurekaClient`](https://github.com/YunaiV/eureka/blob/3ef162f20a28c75de84321b69412c4ef138ad55a/eureka-client/src/main/java/com/netflix/discovery/EurekaClient.java)，Eureka-Client **接口**，声明如下方法：

* 提供**多种**方法获取应用集合(`com.netflix.discovery.shared.Applications`) 和 应用实例信息集合( `com.netflix.appinfo.InstanceInfo` )。
* 提供方法获取**本地**客户端信息，例如，应用管理器( `com.netflix.appinfo.ApplicationInfoManager` )和 Eureka-Client 配置( `com.netflix.discovery.EurekaClientConfig` )。
* 提供方法**注册**本地客户端的健康检查和 Eureka 事件监听器。

另外，Eureka 2.X 版本正在开发，该接口为 Eureka 1.X 和 2.X 提供平滑过渡接口。

> This interface does NOT try to clean up the current client interface for eureka 1.x. Rather it tries to provide an easier transition path from eureka 1.x to eureka 2.x.

## 2.1 LookupService

[`com.netflix.discovery.shared.LookupService`](https://github.com/YunaiV/eureka/blob/3ef162f20a28c75de84321b69412c4ef138ad55a/eureka-client/src/main/java/com/netflix/discovery/shared/LookupService.java)，查找服务**接口**，提供**简单单一**的方式获取应用集合(`com.netflix.discovery.shared.Applications`) 和 应用实例信息集合( `com.netflix.appinfo.InstanceInfo` )。

![](http://www.iocoder.cn/images/Eureka/2018_04_29/01.png)

* 在 Eureka-Client 里，EurekaClient 继承该接口。
* 在 Eureka-Server 里，`com.netflix.eureka.registry.InstanceRegistry` 继承该接口。

# 3. DiscoveryClient

`com.netflix.discovery.DiscoveryClient`，实现 EurekaClient **接口**，用于与 Eureka-Server 交互。实现如下方法：

* 向 Eureka-Server **注册**自身服务
* 向 Eureka-Server **续约**自身服务
* 向 Eureka-Server **取消**自身服务，当关闭时
* 从 Eureka-Server **查询**应用集合和应用实例信息
* *简单来理解，对 Eureka-Server 服务的增删改查*

## 3.1 构造方法参数

DiscoveryClient **完整**构造方法需要传入四个参数，实现代码如下：

```Java
DiscoveryClient(ApplicationInfoManager applicationInfoManager, EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args,
                    Provider<BackupRegistry> backupRegistryProvider) {
     // .... 省略代码
}
```

* ApplicationInfoManager，在[《Eureka 源码解析 —— Eureka-Client 初始化（一）之 EurekaInstanceConfig》](ttp://www.iocoder.cn/Eureka/eureka-client-init-first/)有详细解析。
* EurekaClientConfig，在[《Eureka 源码解析 —— Eureka-Client 初始化（二）之 EurekaClientConfig》](http://www.iocoder.cn/Eureka/eureka-client-init-second/)有详细解析。
* `com.netflix.discovery.BackupRegistry`，备份注册中心**接口**。当 Eureka-Client 启动时，无法从 Eureka-Server 读取注册信息（可能挂了），从备份注册中心读取注册信息。实现代码如下：

    ```Java
    // BackupRegistry.java
    public interface BackupRegistry {

        Applications fetchRegistry();
    
        Applications fetchRegistry(String[] includeRemoteRegions);
    }
    
    // NotImplementedRegistryImpl.java
    public class NotImplementedRegistryImpl implements BackupRegistry {

        @Override
        public Applications fetchRegistry() {
            return null;
        }
    
        @Override
        public Applications fetchRegistry(String[] includeRemoteRegions) {
            return null;
        }
    }
    ```
    * 从 `com.netflix.discovery.NotImplementedRegistryImpl` 可以看出，目前 Eureka-Client 未提供合适的默认实现。

* `com.netflix.discovery.AbstractDiscoveryClientOptionalArgs`，DiscoveryClient 可选参数抽象基类。不同于上面三个**必填**参数，该参数是**选填**参数，实际生产下使用较少。实现代码如下：

    ```Java
    public abstract class AbstractDiscoveryClientOptionalArgs<T> {
    
        /**
         * 生成健康检查回调的工厂
         */
        Provider<HealthCheckCallback> healthCheckCallbackProvider;
        /**
         * 生成健康检查处理器的工厂
         */
        Provider<HealthCheckHandler> healthCheckHandlerProvider;
        /**
         * 向 Eureka-Server 注册之前的处理器
         */
        PreRegistrationHandler preRegistrationHandler;
        /**
         * Jersey 过滤器集合
         */
        Collection<T> additionalFilters;
        /**
         * Jersey 客户端
         */
        EurekaJerseyClient eurekaJerseyClient;
        /**
         * 生成 Jersey 客户端的工厂的工厂
         */
        TransportClientFactories transportClientFactories;
        /**
         * Eureka 事件监听器集合
         */
        private Set<EurekaEventListener> eventListeners;
    }
    ```
    * `com.netflix.appinfo.HealthCheckCallback`，健康检查回调**接口**，目前已经废弃，使用 HealthCheckHandler 替代，**你可以不关注该参数**。
    * `com.netflix.appinfo.HealthCheckHandler`，健康检查处理器**接口**，目前暂未提供合适的**默认**实现，唯一提供的 `com.netflix.appinfo.HealthCheckCallbackToHandlerBridge`，用于将 HealthCheckCallback **桥接**成 HealthCheckHandler，实现代码如下：
    
        ```Java
        // HealthCheckHandler.java
        public interface HealthCheckHandler {
    
            InstanceInfo.InstanceStatus getStatus(InstanceInfo.InstanceStatus currentStatus);
        
        }
        
        // HealthCheckCallbackToHandlerBridge.java
        public class HealthCheckCallbackToHandlerBridge implements HealthCheckHandler {
        
            private final HealthCheckCallback callback;
        
            public HealthCheckCallbackToHandlerBridge() {
                callback = null;
            }
        
            public HealthCheckCallbackToHandlerBridge(HealthCheckCallback callback) {
                this.callback = callback;
            }
        
            @Override
            public InstanceInfo.InstanceStatus getStatus(InstanceInfo.InstanceStatus currentStatus) {
                if (null == callback || InstanceInfo.InstanceStatus.STARTING == currentStatus
                        || InstanceInfo.InstanceStatus.OUT_OF_SERVICE == currentStatus) { // Do not go to healthcheck handler if the status is starting or OOS.
                    return currentStatus;
                }
        
                return callback.isHealthy() ? InstanceInfo.InstanceStatus.UP : InstanceInfo.InstanceStatus.DOWN;
            }
        
        }
        ```
        * 在 Spring-Cloud-Eureka-Client，提供了默认实现 [`org.springframework.cloud.netflix.eureka.EurekaHealthCheckHandler`](https://github.com/spring-cloud/spring-cloud-netflix/blob/82991a7fc2859b6345b7f67e2461dbf5d7663836/spring-cloud-netflix-eureka-client/src/main/java/org/springframework/cloud/netflix/eureka/EurekaHealthCheckHandler.java)，需要结合 [`spirng-boot-actuate`](https://github.com/spring-projects/spring-boot/tree/c79568886406662736dcdce78f65e7f46dd62696/spring-boot-actuator/) 使用，感兴趣的同学可以看看。本文暂不拓展开，后面另开文章分享。（TODO[0004]：健康检查）

    * `com.netflix.discovery.PreRegistrationHandler`，向 Eureka-Server 注册之前的处理器**接口**，目前暂未提供默认实现。通过实现该接口，可以在注册前做一些自定义的处理。实现代码如下：

        ```Java
        public interface PreRegistrationHandler {
    
            void beforeRegistration();
        
        }
        ```
        * x
   
   * `additionalFilters`，Jersey 过滤器集合。这里声明泛型 `<T>` 的原因，Jersey 1.X 和 Jersey 2.X 的过滤器接口**不同**，通过泛型来支持。实现代码如下：

       ```Java
       // Jersey1DiscoveryClientOptionalArgs.java
       public class Jersey1DiscoveryClientOptionalArgs extends AbstractDiscoveryClientOptionalArgs<ClientFilter> {
        }
        
       // Jersey2DiscoveryClientOptionalArgs.java
       public class Jersey2DiscoveryClientOptionalArgs extends AbstractDiscoveryClientOptionalArgs<ClientRequestFilter> {
        }
        
       // DiscoveryClientOptionalArgs.java
       public static class DiscoveryClientOptionalArgs extends Jersey1DiscoveryClientOptionalArgs {

       }
       ```
       * Jersey 1.X 使用 ClientFilter 。ClientFilter 目前有两个过滤器实现：EurekaIdentityHeaderFilter 、DynamicGZIPContentEncodingFilter 。
       * Jersey 2.X 使用 ClientRequestFilter 。
       * DiscoveryClient 使用 DiscoveryClientOptionalArgs，即 Jersey 1.X 。

  * `eurekaJerseyClient`，Jersey 客户端。该**参数**目前废弃，使用下面 TransportClientFactories 参数来进行生成。
  * `com.netflix.discovery.shared.transport.jersey.TransportClientFactories`，生成 Jersey 客户端**工厂的工厂**接口。目前有 Jersey1TransportClientFactories 、Jersey2TransportClientFactories 两个实现。TransportClientFactories 实现代码如下：

      ```Java
      // TransportClientFactories.java
      public interface TransportClientFactories<F> {
        
            @Deprecated
            TransportClientFactory newTransportClientFactory(final Collection<F> additionalFilters,
                                                             final EurekaJerseyClient providedJerseyClient);
        
            TransportClientFactory newTransportClientFactory(final EurekaClientConfig clientConfig,
                                                             final Collection<F> additionalFilters,
                                                             final InstanceInfo myInstanceInfo);
      }
      
      // TransportClientFactory.java
      public interface TransportClientFactory {

            EurekaHttpClient newClient(EurekaEndpoint serviceUrl);
    
            void shutdown();
    
      }
      ```
      * 第一个方法已经废弃，这就是为什么说上面的 `eurekaJerseyClient` **参数**( 不是 EurekaJerseyClient 类)已经废弃，被第二个方法取代。相比来说，第二个方法对 EurekaJerseyClient 创建封装会更好。

 * `com.netflix.discovery.EurekaEventListener`，Eureka 事件监听器。实现代码如下：

     ```Java
     // EurekaEventListener.java
     public interface EurekaEventListener {
     }
     
     // EurekaEvent.java
     public interface EurekaEvent {
     }
     
     // DiscoveryEvent.java
     public abstract class DiscoveryEvent implements EurekaEvent {
     
            private final long timestamp;
     
     }
     ```
     * `com.netflix.discovery.StatusChangeEvent`，应用实例状态变更事件，在[《Eureka 源码解析 —— 应用实例注册发现 （一）之注册》「2.1 应用实例信息复制器」](http://www.iocoder.cn/Eureka/instance-registry-register/?self) 有详细解析。
     * `com.netflix.discovery.CacheRefreshedEvent`，在[《Eureka 源码解析 —— 应用实例注册发现 （六）之全量获取》「2.4 发起获取注册信息」](http://www.iocoder.cn/Eureka/instance-registry-fetch-all/?self) 有详细解析。

## 3.2 构造方法

DiscoveryClient 的构造方法实现代码相对较多，已经将代码**切块** + **中文注册**，点击 [DiscoveryClient](https://github.com/YunaiV/eureka/blob/3ef162f20a28c75de84321b69412c4ef138ad55a/eureka-client/src/main/java/com/netflix/discovery/DiscoveryClient.java#L298) 链接，对照下面每个小结阅读理解。

### 3.2.1 赋值 AbstractDiscoveryClientOptionalArgs

```Java
// DiscoveryClient.java 构造方法
if (args != null) {
  this.healthCheckHandlerProvider = args.healthCheckHandlerProvider;
  this.healthCheckCallbackProvider = args.healthCheckCallbackProvider;
  this.eventListeners.addAll(args.getEventListeners());
  this.preRegistrationHandler = args.preRegistrationHandler;
} else {
  this.healthCheckCallbackProvider = null;
  this.healthCheckHandlerProvider = null;
  this.preRegistrationHandler = null;
}
```

### 3.2.2 赋值 ApplicationInfoManager、EurekaClientConfig

```Java
// DiscoveryClient.java 构造方法
this.applicationInfoManager = applicationInfoManager;
InstanceInfo myInfo = applicationInfoManager.getInfo();

clientConfig = config;
staticClientConfig = clientConfig;
transportConfig = config.getTransportConfig();
instanceInfo = myInfo;
if (myInfo != null) {
  appPathIdentifier = instanceInfo.getAppName() + "/" + instanceInfo.getId(); // 无实际业务用途，用于打 logger
} else {
  logger.warn("Setting instanceInfo to a passed in null value");
}
```

### 3.2.3 赋值 BackupRegistry

```Java
this.backupRegistryProvider = backupRegistryProvider;
```

### 3.2.4 初始化 InstanceInfoBasedUrlRandomizer

TODO[0016]：InstanceInfoBasedUrlRandomizer

```Java
this.urlRandomizer = new EndpointUtils.InstanceInfoBasedUrlRandomizer(instanceInfo);
```

### 3.2.5 初始化 Applications 在本地的缓存

```Java
// DiscoveryClient.java 变量
/**
* Applications 在本地的缓存
*/
private final AtomicReference<Applications> localRegionApps = new AtomicReference<Applications>();
/**
* 拉取注册信息次数
* monotonically increasing generation counter to ensure stale threads do not reset registry to an older version
*/
private final AtomicLong fetchRegistryGeneration;

// DiscoveryClient.java 构造方法
localRegionApps.set(new Applications());

fetchRegistryGeneration = new AtomicLong(0);
```

* 在创建 DiscoveryClient 时，`localRegionApps` 为空。
* 定时任务**间隔**从 Eureka-Server 拉取注册应用信息到本地缓存，在 [《Eureka 源码解析 —— 应用实例注册发现 （六）之全量获取》](http://www.iocoder.cn/Eureka/instance-registry-fetch-all/?self) 有详细解析。

### 3.2.6 获取哪些 Region 集合的注册信息

```Java
// DiscoveryClient.java 变量
/**
* 获取哪些区域( Region )集合的注册信息
*/
private final AtomicReference<String> remoteRegionsToFetch;
/**
* 获取哪些区域( Region )集合的注册信息
*/
private final AtomicReference<String[]> remoteRegionsRef;

// DiscoveryClient.java 构造方法
remoteRegionsToFetch = new AtomicReference<>(clientConfig.fetchRegistryForRemoteRegions());
remoteRegionsRef = new AtomicReference<>(remoteRegionsToFetch.get() == null ? null : remoteRegionsToFetch.get().split(","));
```

### 3.2.7 初始化拉取、心跳的监控

```Java
// DiscoveryClient.java 变量
/**
* 最后成功从 Eureka-Server 拉取注册信息时间戳
*/
private volatile long lastSuccessfulRegistryFetchTimestamp = -1;
/**
* 最后成功向 Eureka-Server 心跳时间戳
*/
private volatile long lastSuccessfulHeartbeatTimestamp = -1;
/**
* 心跳监控
*/
private final ThresholdLevelsMetric heartbeatStalenessMonitor;
/**
* 拉取监控
*/
private final ThresholdLevelsMetric registryStalenessMonitor;

// DiscoveryClient.java 构造方法
if (config.shouldFetchRegistry()) {
  this.registryStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRY_PREFIX + "lastUpdateSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
} else {
  this.registryStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
}

if (config.shouldRegisterWithEureka()) {
  this.heartbeatStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRATION_PREFIX + "lastHeartbeatSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
} else {
  this.heartbeatStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
}
```

* 每次成功向 Eureka-Serve 心跳或者从从 Eureka-Server 拉取注册信息后，都会更新相应时间戳。
* 配合 [Netflix Servo](https://github.com/Netflix/servo) 实现监控信息采集。
* 对 [`com.netflix.discovery.util.ThresholdLevelsMetric`](https://github.com/YunaiV/eureka/blob/3ef162f20a28c75de84321b69412c4ef138ad55a/eureka-client/src/main/java/com/netflix/discovery/util/ThresholdLevelsMetric.java) 感兴趣的同学可以点击链接查看。本文暂不拓展开，后面另开文章分享。（TODO[0012]：监控相关）

### 3.2.8 结束初始化，当无需和 Eureka-Server 交互

```Java
// DiscoveryClient.java 构造方法
if (!config.shouldRegisterWithEureka() && !config.shouldFetchRegistry()) {
  logger.info("Client configured to neither register nor query for data.");
  scheduler = null;
  heartbeatExecutor = null;
  cacheRefreshExecutor = null;
  eurekaTransport = null;
  instanceRegionChecker = new InstanceRegionChecker(new PropertyBasedAzToRegionMapper(config), clientConfig.getRegion());

  // This is a bit of hack to allow for existing code using DiscoveryManager.getInstance()
  // to work with DI'd DiscoveryClient
  DiscoveryManager.getInstance().setDiscoveryClient(this);
  DiscoveryManager.getInstance().setEurekaClientConfig(config);

  initTimestampMs = System.currentTimeMillis();
  logger.info("Discovery Client initialized at timestamp {} with initial instances count: {}",
          initTimestampMs, this.getApplications().size());

  return;  // no need to setup up an network tasks and we are done
}
```

### 3.2.9 初始化线程池

```Java
// DiscoveryClient.java 变量
/**
* 线程池
*
* A scheduler to be used for the following 3 tasks: 【目前只有两个】
* - updating service urls
* - scheduling a TimedSuperVisorTask
*/
private final ScheduledExecutorService scheduler;
// additional executors for supervised subtasks
/**
* 心跳执行器
*/
private final ThreadPoolExecutor heartbeatExecutor;
/**
* {@link #localRegionApps} 刷新执行器
*/
private final ThreadPoolExecutor cacheRefreshExecutor;

// DiscoveryClient.java 构造方法
// default size of 2 - 1 each for heartbeat and cacheRefresh
scheduler = Executors.newScheduledThreadPool(2,
     new ThreadFactoryBuilder()
             .setNameFormat("DiscoveryClient-%d")
             .setDaemon(true)
             .build());

heartbeatExecutor = new ThreadPoolExecutor(
     1, clientConfig.getHeartbeatExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
     new SynchronousQueue<Runnable>(),
     new ThreadFactoryBuilder()
             .setNameFormat("DiscoveryClient-HeartbeatExecutor-%d")
             .setDaemon(true)
             .build()
);  // use direct handoff

cacheRefreshExecutor = new ThreadPoolExecutor(
     1, clientConfig.getCacheRefreshExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
     new SynchronousQueue<Runnable>(),
     new ThreadFactoryBuilder()
             .setNameFormat("DiscoveryClient-CacheRefreshExecutor-%d")
             .setDaemon(true)
             .build()
);  // use direct handoff
```

* `scheduler`，**定时任务**线程池，初始化大小为 2，一个给 `heartbeatExecutor`，一个给 `cacheRefreshExecutor`。
* `heartbeatExecutor`、`cacheRefreshExecutor` 在提交给 `scheduler` 才声明具体的**任务**。

### 3.2.10 初始化 Eureka 网络通信相关

```Java
// DiscoveryClient.java 构造方法
eurekaTransport = new EurekaTransport();
scheduleServerEndpointTask(eurekaTransport, args);
```

* 本文暂不拓展开，在 [《Eureka 源码解析 —— EndPoint 与 解析器》](http://www.iocoder.cn/Eureka/end-point-and-resolver/?self) 和 [《Eureka 源码解析 —— 网络通信》](http://www.iocoder.cn/Eureka/transport/?self) 详细解析。

### 3.2.11 初始化 InstanceRegionChecker

```Java
// DiscoveryClient.java 构造方法
AzToRegionMapper azToRegionMapper;
if (clientConfig.shouldUseDnsForFetchingServiceUrls()) {
    azToRegionMapper = new DNSBasedAzToRegionMapper(clientConfig);
} else {
    azToRegionMapper = new PropertyBasedAzToRegionMapper(clientConfig);
}
if (null != remoteRegionsToFetch.get()) {
    azToRegionMapper.setRegionsToFetch(remoteRegionsToFetch.get().split(","));
}
instanceRegionChecker = new InstanceRegionChecker(azToRegionMapper, clientConfig.getRegion());
```

* `com.netflix.discovery.AzToRegionMapper`，主要用于亚马逊 AWS，跳过。
* `com.netflix.discovery.InstanceRegionChecker`，应用实例信息区域( `region` )校验，实现代码如下：

    ```Java
    public class InstanceRegionChecker {
    
        // ... 省略和亚马逊 AWS 相关的属性和方法
    
        /**
         * 本地区域( Region )
         */
        private final String localRegion;
    
        public boolean isLocalRegion(@Nullable String instanceRegion) {
            return null == instanceRegion || instanceRegion.equals(localRegion); // no region == local
        }
    
        public String getLocalRegion() {
            return localRegion;
        }
    
    }
    ```

### 3.2.12 从 Eureka-Server 拉取注册信息

```Java
// DiscoveryClient.java 构造方法
if (clientConfig.shouldFetchRegistry() && !fetchRegistry(false)) {
  fetchRegistryFromBackup();
}
```

* 调用 `#fetchRegistry(false)` 方法，从 Eureka-Server **初始**拉取注册信息。在（TO后文链接）详细解析。
* 调用 `#fetchRegistryFromBackup()` 方法，若**初始**拉取注册信息失败，从备份注册中心获取。实现代码如下：

    ```Java
    // DiscoveryClient.java
    private void fetchRegistryFromBackup() {
        try {
            @SuppressWarnings("deprecation")
            BackupRegistry backupRegistryInstance = newBackupRegistryInstance();
            if (null == backupRegistryInstance) { // backward compatibility with the old protected method, in case it is being used.
                backupRegistryInstance = backupRegistryProvider.get();
            }
            if (null != backupRegistryInstance) {
                Applications apps = null;
                if (isFetchingRemoteRegionRegistries()) {
                    String remoteRegionsStr = remoteRegionsToFetch.get();
                    if (null != remoteRegionsStr) {
                        apps = backupRegistryInstance.fetchRegistry(remoteRegionsStr.split(","));
                    }
                } else {
                    apps = backupRegistryInstance.fetchRegistry();
                }
                if (apps != null) {
                    final Applications applications = this.filterAndShuffle(apps);
                    applications.setAppsHashCode(applications.getReconcileHashCode());
                    localRegionApps.set(applications);
                    logTotalInstances();
                    logger.info("Fetched registry successfully from the backup");
                }
            } else {
                logger.warn("No backup registry instance defined & unable to find any discovery servers.");
            }
        } catch (Throwable e) {
            logger.warn("Cannot fetch applications from apps although backup registry was specified", e);
        }
    }
    ```
    * BackupRegistry 目前暂未提供默认实现，需要自行相关逻辑。

### 3.2.13 执行向 Eureka-Server 注册之前的处理器

```Java
// DiscoveryClient.java 构造方法
// call and execute the pre registration handler before all background tasks (inc registration) is started
if (this.preRegistrationHandler != null) {
  this.preRegistrationHandler.beforeRegistration();
}
```

### 3.2.14 初始化定时任务

```Java
// DiscoveryClient.java 构造方法
initScheduledTasks();

// DiscoveryClient.java
private void initScheduledTasks() {
   // 从 Eureka-Server 拉取注册信息执行器
   if (clientConfig.shouldFetchRegistry()) {
       // registry cache refresh timer
       int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
       int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
       scheduler.schedule(
               new TimedSupervisorTask(
                       "cacheRefresh",
                       scheduler,
                       cacheRefreshExecutor,
                       registryFetchIntervalSeconds,
                       TimeUnit.SECONDS,
                       expBackOffBound,
                       new CacheRefreshThread()
               ),
               registryFetchIntervalSeconds, TimeUnit.SECONDS);
   }

   // 向 Eureka-Server 心跳（续租）执行器
   if (clientConfig.shouldRegisterWithEureka()) {
       int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
       int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
       logger.info("Starting heartbeat executor: " + "renew interval is: " + renewalIntervalInSecs);

       // Heartbeat timer
       scheduler.schedule(
               new TimedSupervisorTask(
                       "heartbeat",
                       scheduler,
                       heartbeatExecutor,
                       renewalIntervalInSecs,
                       TimeUnit.SECONDS,
                       expBackOffBound,
                       new HeartbeatThread()
               ),
               renewalIntervalInSecs, TimeUnit.SECONDS);

       // InstanceInfo replicator
       instanceInfoReplicator = new InstanceInfoReplicator(
               this,
               instanceInfo,
               clientConfig.getInstanceInfoReplicationIntervalSeconds(),
               2); // burstSize

       statusChangeListener = new ApplicationInfoManager.StatusChangeListener() {
           @Override
           public String getId() {
               return "statusChangeListener";
           }

           @Override
           public void notify(StatusChangeEvent statusChangeEvent) {
               if (InstanceStatus.DOWN == statusChangeEvent.getStatus() ||
                       InstanceStatus.DOWN == statusChangeEvent.getPreviousStatus()) {
                   // log at warn level if DOWN was involved
                   logger.warn("Saw local status change event {}", statusChangeEvent);
               } else {
                   logger.info("Saw local status change event {}", statusChangeEvent);
               }
               instanceInfoReplicator.onDemandUpdate();
           }
       };

       if (clientConfig.shouldOnDemandUpdateStatusChange()) {
           applicationInfoManager.registerStatusChangeListener(statusChangeListener);
       }

       instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
   } else {
       logger.info("Not registering with Eureka server per configuration");
   }
}
```

* **初始化**从 Eureka-Server 拉取注册信息执行器，在 [《Eureka 源码解析 —— 应用实例注册发现 （六）之全量获取》](http://www.iocoder.cn/Eureka/instance-registry-fetch-all/?self) 详细解析。
* **初始化**向 Eureka-Server 心跳（续租）执行器，在 [《Eureka 源码解析 —— 应用实例注册发现（二）之续租》](http://www.iocoder.cn/Eureka/instance-registry-renew/?self) 详细解析。

### 3.2.15 向 Servo 注册监控

```Java
// DiscoveryClient.java 构造方法
try {
  Monitors.registerObject(this);
} catch (Throwable e) {
  logger.warn("Cannot register timers", e);
}
```
* 配合 [Netflix Servo](https://github.com/Netflix/servo) 实现监控信息采集。

### 3.2.16 初始化完成

```Java
// DiscoveryClient.java 变量
/**
* 初始化完成时间戳
*/
private final long initTimestampMs;

// DiscoveryClient.java 构造方法
// 【3.2.16】初始化完成
// This is a bit of hack to allow for existing code using DiscoveryManager.getInstance()
// to work with DI'd DiscoveryClient
DiscoveryManager.getInstance().setDiscoveryClient(this);
DiscoveryManager.getInstance().setEurekaClientConfig(config);

initTimestampMs = System.currentTimeMillis();
logger.info("Discovery Client initialized at timestamp {} with initial instances count: {}",
      initTimestampMs, this.getApplications().size());
```

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

由于笔者是边理解源码边输出博客内容，如果有错误或者不清晰的地方，**欢迎**微笑给我的微信公众号( **芋道源码** ) 留言，我会**仔细**回复。感谢 + 1024。

后面文章不断更新，会慢慢完善本文中的。

推荐参考阅读：

* [程序猿DD —— 《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi) Spring Cloud Eureka —— 源码分析
* **买盗版书，等于编写一个初级 BUG**

