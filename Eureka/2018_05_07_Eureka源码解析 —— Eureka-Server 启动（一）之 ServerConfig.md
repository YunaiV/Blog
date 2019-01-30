title: Eureka 源码解析 —— Eureka-Server 启动（一）之 ServerConfig
date: 2018-05-07
tags:
categories: Eureka
permalink: Eureka/eureka-server-init-first

---

摘要: 原创出处 http://www.iocoder.cn/Eureka/eureka-server-init-first/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Eureka 1.8.X 版本** 

- [1. 概述](http://www.iocoder.cn/Eureka/eureka-server-init-first/)
- [2. EurekaServerConfig](http://www.iocoder.cn/Eureka/eureka-server-init-first/)
  - [2.1 类关系图](http://www.iocoder.cn/Eureka/eureka-server-init-first/)
  - [2.2 配置属性](http://www.iocoder.cn/Eureka/eureka-server-init-first/)
  - [2.3 DefaultEurekaServerConfig](http://www.iocoder.cn/Eureka/eureka-server-init-first/)
- [666. 彩蛋](http://www.iocoder.cn/Eureka/eureka-server-init-first/)

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

本文主要分享 **Eureka-Server 启动的过程**。

考虑到整个初始化的过程中涉及的代码特别多，拆分成两两篇文章：

* 【本文】ServerConfig
* [EurekaBootStrap](http://www.iocoder.cn/Eureka/eureka-server-init-second/?self)

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



# 2. EurekaServerConfig

`com.netflix.eureka.EurekaServerConfig`，**Eureka-Server** 配置**接口**。

## 2.1 类关系图

![](http://www.iocoder.cn/images/Eureka/2018_05_07/01.png)

## 2.2 配置属性

点击 [EurekaServerConfig](https://github.com/YunaiV/eureka/blob/8b0f67ac33116ee05faad1ff5125034cfcf573bf/eureka-core/src/main/java/com/netflix/eureka/EurekaServerConfig.java) 查看配置属性简介，已经添加中文注释，可以对照着英文注释一起理解。这里笔者摘出部分较为重要的属性：

* **请求认证相关**
    * Eureka-Server 未实现认证。在 Spring-Cloud-Eureka-Server，通过 `spring-boot-starter-security` 模块支持。[《spring cloud-给Eureka Server加上安全的用户认证》](http://blog.csdn.net/liuchuanhong1/article/details/54729556)有详细解析。
    * `#shouldLogIdentityHeaders()` ：打印访问的客户端名和版本号，配合 [Netflix Servo](https://github.com/Netflix/servo) 实现监控信息采集。
* **请求限流相关**
    * [《Eureka 源码解析 —— 基于令牌桶算法的 RateLimiter》](http://www.iocoder.cn/Eureka/rate-limiter/?self) 有详细解析。
    * `#isRateLimiterEnabled()` ：请求限流是否开启。
    * `#isRateLimiterThrottleStandardClients()` ：是否对**标准客户端**判断是否限流。**标准客户端**通过请求头( `header` )的 `"DiscoveryIdentity-Name"` 来判断，是否在标准客户端名集合里。
    * `#getRateLimiterPrivilegedClients()` ：**标准**客户端名集合。默认包含`"DefaultClient"` 和 `"DefaultServer"` 。
    * `#getRateLimiterBurstSize()` ：速率限制的 burst size ，使用**令牌桶算法**。
    * `#getRateLimiterRegistryFetchAverageRate()` ：**增量**拉取注册信息的速率限制。
    * `#getRateLimiterFullFetchAverageRate()` ：**全量**拉取注册信息的速率限制。
* **获取注册信息请求相关**
    * [《Eureka 源码解析 —— 应用实例注册发现 （六）之全量获取》](http://www.iocoder.cn/Eureka/instance-registry-fetch-all/?self) 有详细解析。
    * [《Eureka 源码解析 —— 应用实例注册发现 （七）之增量获取》](http://www.iocoder.cn/Eureka/instance-registry-fetch-delta/?self) 有详细解析。
    * `#shouldUseReadOnlyResponseCache()` ：是否开启只读请求响应缓存。响应缓存 ( ResponseCache ) 机制目前使用两层缓存策略。优先读取**只读缓存**，读取不到后读取**固定过期**的**读写缓存**。
    * `#getResponseCacheUpdateIntervalMs()` ：**只读缓存**更新频率，单位：毫秒。**只读缓存**定时更新任务只更新读取过请求 (`com.netflix.eureka.registry.Key`)，因此虽然永不过期，也会存在读取不到的情况。
    * `#getResponseCacheAutoExpirationInSeconds()` ：**读写缓存**写入后过期时间，单位：秒。
    * `#getRetentionTimeInMSInDeltaQueue()`：租约变更记录过期时长，单位：毫秒。默认值 ： 3 * 60 * 1000 毫秒。
    * `#DeltaRetentionTimerIntervalInMs()`：移除队列里过期的租约变更记录的定时任务执行频率，单位：毫秒。默认值 ：30 * 1000 毫秒。

* **自我保护机制相关**
    * 在 [《Eureka 源码解析 —— 应用实例注册发现（四）之自我保护机制》](http://www.iocoder.cn/Eureka/instance-registry-self-preservation/?self) 有详细解析。 
    * `#shouldEnableSelfPreservation()` ：是否开启自我保护模式。
        
        > FROM [周立——《理解Eureka的自我保护模式》](http://www.itmuch.com/spring-cloud-sum/understanding-eureka-self-preservation/?from=www.iocoder.cn)  
        > 当Eureka Server节点在短时间内丢失过多客户端时（可能发生了网络分区故障），那么这个节点就会进入自我保护模式。  
        > 一旦进入该模式，Eureka Server就会保护服务注册表中的信息，不再删除服务注册表中的数据（也就是不会注销任何微服务）。  
        > 当网络故障恢复后，该Eureka Server节点会自动退出自我保护模式。
    
    * `#getRenewalPercentThreshold()` ：开启自我保护模式比例，超过该比例后开启自我保护模式。
    * `#getRenewalThresholdUpdateIntervalMs()` ：自我保护模式比例更新定时任务执行频率，单位：毫秒。

* **注册的应用实例的租约过期相关**
    * 在 [《Eureka 源码解析 —— 应用实例注册发现（五）之过期》](http://www.iocoder.cn/Eureka/instance-registry-evict/?self) 有详细解析。 
    
    * `#getEvictionIntervalTimerInMs()` ：租约过期定时任务执行频率，单位：毫秒。
* **Eureka-Server 远程节点( 非集群 )读取相关**
    * TODO[0009]：RemoteRegionRegistry
    * `#getRemoteRegionUrlsWithName()` ：TODO[0009]：RemoteRegionRegistry。
        * `key` ：Eureka-Server 区域( `region` )
        * `value` ：Eureka-Server 地址
    * `#getRemoteRegionAppWhitelist()` ：TODO[0009]：RemoteRegionRegistry。
    * `#getRemoteRegionRegistryFetchInterval()` ：TODO[0009]：RemoteRegionRegistry。
    * `#getRegistrySyncRetries()` ：Eureka-Server **启动**时，从远程 Eureka-Server 读取失败重试次数。
    * `#getRegistrySyncRetryWaitMs()` ：Eureka-Server **启动**时，从远程 Eureka-Server 读取失败等待( `sleep` )间隔，单位：毫秒。 
    * `#getRemoteRegionFetchThreadPoolSize()` ：TODO[0009]：RemoteRegionRegistry。
    * `#disableTransparentFallbackToOtherRegion()` ：是否禁用本地读取不到注册信息，从远程 Eureka-Server 读取。
* **Eureka-Server 集群同步相关**
    * 在 [《Eureka 源码解析 —— Eureka-Server 集群同步》](http://www.iocoder.cn/Eureka/instance-registry-register/)
    * `#getMaxThreadsForPeerReplication()` ：同步应用实例信息最大线程数。
    * `#getMaxElementsInPeerReplicationPool()` ：待执行同步应用实例信息事件缓冲最大数量。
    * `#getMaxTimeForReplication()` ：执行单个同步应用实例信息状态任务最大时间。
    * `#shouldSyncWhenTimestampDiffers()` ：是否同步应用实例信息，当应用实例信息最后更新时间戳( `lastDirtyTimestamp` )发生改变。
    * `#getWaitTimeInMsWhenSyncEmpty()` ：Eureka-Server **启动**时，从远程 Eureka-Server 读取不到注册信息时，多长时间不允许 Eureka-Client 访问。
    * `#getPeerEurekaNodesUpdateIntervalMs()` ：Eureka-Server 集群节点更新频率，单位：毫秒。

## 2.3 DefaultEurekaServerConfig

`com.netflix.eureka.DefaultEurekaServerConfig`，基于**配置文件**的 **Eureka-Server** 配置**实现类**，实现代码如下：

```Java
public class DefaultEurekaServerConfig implements EurekaServerConfig {

    // ... 省略部分方法和属性

    private static final String ARCHAIUS_DEPLOYMENT_ENVIRONMENT = "archaius.deployment.environment";
    private static final String TEST = "test";
    private static final String EUREKA_ENVIRONMENT = "eureka.environment";

    /**
     * 配置文件对象
     */
    private static final DynamicPropertyFactory configInstance = com.netflix.config.DynamicPropertyFactory.getInstance();
    /**
     * 配置文件
     */
    private static final DynamicStringProperty EUREKA_PROPS_FILE = DynamicPropertyFactory
            .getInstance().getStringProperty("eureka.server.props", "eureka-server");

    /**
     * 命名空间
     */
    private String namespace = "eureka.";

    public DefaultEurekaServerConfig() {
        init();
    }

    public DefaultEurekaServerConfig(String namespace) {
        // 设置 namespace，为 "." 结尾
        this.namespace = namespace;
        // 初始化 配置文件对象
        init();
    }

    private void init() {
        // 初始化 配置文件对象
        String env = ConfigurationManager.getConfigInstance().getString(EUREKA_ENVIRONMENT, TEST);
        ConfigurationManager.getConfigInstance().setProperty(ARCHAIUS_DEPLOYMENT_ENVIRONMENT, env);
        String eurekaPropsFile = EUREKA_PROPS_FILE.get();
        try {
            // ConfigurationManager
            // .loadPropertiesFromResources(eurekaPropsFile);
            ConfigurationManager.loadCascadedPropertiesFromResources(eurekaPropsFile);
        } catch (IOException e) {
            logger.warn("Cannot find the properties specified : {}. This may be okay if there are other environment "
                            + "specific properties or the configuration is installed with a different mechanism.", eurekaPropsFile);
        }
    }
}
```

* 调用 `#init()` 方法，初始化配置文件对象。类似 PropertiesInstanceConfig，点击[《Eureka 源码解析 —— Eureka-Client 初始化（一）之 EurekaInstanceConfig》「2.4 PropertiesInstanceConfig」](http://www.iocoder.cn/Eureka/eureka-client-init-first/?self)查看详细解析。**默认配置文件名**为 `eureka-server`。
* 无配置文件的每个属性 KEY 的枚举类。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

涉及到配置，内容初看起来会比较多，慢慢理解后，就会变得很“啰嗦”，请保持耐心。

胖友，分享一个朋友圈可好。

