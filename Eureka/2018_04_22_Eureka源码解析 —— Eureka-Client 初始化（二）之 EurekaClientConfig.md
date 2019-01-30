title: Eureka 源码解析 —— Eureka-Client 初始化（二）之 EurekaClientConfig
date: 2018-04-23
tags:
categories: Eureka
permalink: Eureka/eureka-client-init-second

---

摘要: 原创出处 http://www.iocoder.cn/Eureka/eureka-client-init-second/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Eureka 1.8.X 版本** 

- [1. 概述](http://www.iocoder.cn/Eureka/eureka-client-init-second/)
- [2. EurekaClientConfig](http://www.iocoder.cn/Eureka/eureka-client-init-second/)
	- [2.1 类关系图](http://www.iocoder.cn/Eureka/eureka-client-init-second/)
	- [2.2 配置属性](http://www.iocoder.cn/Eureka/eureka-client-init-second/)
	- [2.3 DefaultEurekaClientConfig](http://www.iocoder.cn/Eureka/eureka-client-init-second/)
	- [2.4 DefaultEurekaClientConfigProvider](http://www.iocoder.cn/Eureka/eureka-client-init-second/)
	- [2.5 小结](http://www.iocoder.cn/Eureka/eureka-client-init-second/)
- [3. EurekaTransportConfig](http://www.iocoder.cn/Eureka/eureka-client-init-second/)
	- [3.1 类关系图](http://www.iocoder.cn/Eureka/eureka-client-init-second/)
	- [3.2 配置属性](http://www.iocoder.cn/Eureka/eureka-client-init-second/)
	- [3.3 DefaultEurekaTransportConfig](http://www.iocoder.cn/Eureka/eureka-client-init-second/)
- [666. 彩蛋](http://www.iocoder.cn/Eureka/eureka-client-init-second/)

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

本文接[《Eureka 源码解析 —— Eureka-Client 初始化（一）之 EurekaInstanceConfig》](http://www.iocoder.cn/Eureka/eureka-client-init-first/?self)，主要分享 **Eureka-Client 自身初始化的过程**的第二部分 —— **EurekaClientConfig**，不包含 Eureka-Client 向 Eureka-Server 的注册过程( 🙂后面会另外文章分享 )。

Eureka-Client 自身初始化过程中，涉及到主要对象如下图：

![](http://www.iocoder.cn/images/Eureka/2018_04_15/01.png)

1. **创建** EurekaInstanceConfig对象
1. 使用 EurekaInstanceConfig对象 **创建** InstanceInfo对象
1. 使用 EurekaInstanceConfig对象 + InstanceInfo对象 **创建** ApplicationInfoManager对象
1. **创建** EurekaClientConfig对象
1. 使用 ApplicationInfoManager对象 + EurekaClientConfig对象 **创建** EurekaClient对象

考虑到整个初始化的过程中涉及的配置特别多，拆分成三篇文章：

1. （一）[EurekaInstanceConfig]((http://www.iocoder.cn/Eureka/eureka-client-init-first/))
2. **【本文】**（二）EurekaClientConfig
3. （三）[EurekaClient](http://www.iocoder.cn/Eureka/eureka-client-init-third/)

下面我们来看看每个**类**的实现。

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



# 2. EurekaClientConfig

`com.netflix.discovery.EurekaClientConfig`，**Eureka-Client** 配置**接口**。

## 2.1 类关系图

EurekaClientConfig 整体类关系如下图：

![](http://www.iocoder.cn/images/Eureka/2018_04_22/04.png)

* 本文只解析**红圈**部分类。
* EurekaArchaius2ClientConfig 基于 [Netflix Archaius 2.x](https://github.com/Netflix/archaius) 实现，目前还在开发中，因此暂不解析。

## 2.2 配置属性

点击 [EurekaClientConfig](https://github.com/YunaiV/eureka/blob/8b0f67ac33116ee05faad1ff5125034cfcf573bf/eureka-client/src/main/java/com/netflix/discovery/EurekaClientConfig.java) 查看配置属性简介，已经添加中文注释，可以对照着英文注释一起理解。这里笔者摘出部分较为重要的属性：

* **Region、Zone 相关**
    * `#getRegion()` ：Eureka-Client 所在区域( `region` )。
    * `#getAvailabilityZones()` ：Eureka-Client 所在地区( `region` ) 可用区( `zone` )集合。**该参数虽然是数组，第一个元素代表其所在的可用区**。实现代码如下：
    
        ```Java
        // InstanceInfo.java
        public static String getZone(String[] availZones, InstanceInfo myInfo) {
            String instanceZone = ((availZones == null || availZones.length == 0) ? "default"
                    : availZones[0]);
            if (myInfo != null
                    && myInfo.getDataCenterInfo().getName() == DataCenterInfo.Name.Amazon) {
    
                String awsInstanceZone = ((AmazonInfo) myInfo.getDataCenterInfo())
                        .get(AmazonInfo.MetaDataKey.availabilityZone);
                if (awsInstanceZone != null) {
                    instanceZone = awsInstanceZone;
                }
    
            }
            return instanceZone;
        }
        ```
        * x
    
   * 进步一步理解 Region、Zone 查看[《周立 —— Region、Zone解析》](http://www.itmuch.com/spring-cloud-1/?from=www.iocoder.cn)。
* **使用 DNS 获取 Eureka-Server URL 相关**
    * `#shouldUseDnsForFetchingServiceUrls()` ：是否使用 DNS 方式获取 Eureka-Server URL 地址。
    * `#getEurekaServerDNSName()` ：Eureka-Server 的 DNS 名。
    * `#getEurekaServerPort()` ：Eureka-Server 的端口。
    * `#getEurekaServerURLContext()` ：Eureka-Server 的 URL Context 。
    * `#getEurekaServiceUrlPollIntervalSeconds()` ：轮询获取 Eureka-Server 地址变更频率，单位：秒。默认：300 秒。
    * `#shouldPreferSameZoneEureka()` ：优先使用相同区( `zone` )的 Eureka-Server。
* **直接配合 Eureka-Server URL 相关**
    * `#getEurekaServerServiceUrls()` ： Eureka-Server 的 URL 集合。
* **发现：从 Eureka-Server 获取注册信息相关**
    * `#shouldFetchRegistry()` ：是否从 Eureka-Server 拉取注册信息。
    * `#getRegistryFetchIntervalSeconds()` ：从 Eureka-Server 拉取注册信息频率，单位：秒。默认：30 秒。
    * `#shouldFilterOnlyUpInstances()` ：是否过滤，只获取状态为开启( Up )的应用实例集合。
    * `#fetchRegistryForRemoteRegions()` ：TODO[0009]：RemoteRegionRegistry
    * `#getCacheRefreshExecutorThreadPoolSize()` ：注册信息缓存刷新线程池大小。
    * `#getCacheRefreshExecutorExponentialBackOffBound()` ：注册信息缓存刷新执行超时后的延迟重试的时间。
    * `#getRegistryRefreshSingleVipAddress()` ：只获得一个 `vipAddress` 对应的应用实例们的注册信息。
        * 实现逻辑和 [《Eureka 源码解析 —— 应用实例注册发现 （六）之全量获取》](http://www.iocoder.cn/Eureka/instance-registry-fetch-all/?self)
        * 本系列暂时写对它的源码解析，感兴趣的同学可以看 `com.netflix.discovery.shared.transport.EurekaHttpClient#getVip(String, String...)` 和 `com.netflix.eureka.resources.AbstractVIPResource` 。
* **注册：向 Eureka-Server 注册自身服务**
    * `#shouldRegisterWithEureka()` ：是否向 Eureka-Server 注册自身服务。
    * `#shouldUnregisterOnShutdown()` ：是否向 Eureka-Server 取消注册自身服务，当进程关闭时。
    * `#getInstanceInfoReplicationIntervalSeconds()` ：向 Eureka-Server 同步应用实例信息变化频率，单位：秒。
    * `#getInitialInstanceInfoReplicationIntervalSeconds()` ：向 Eureka-Server 同步应用信息变化初始化延迟，单位：秒。
    * `#getBackupRegistryImpl()` ：获取备份注册中心实现类。当 Eureka-Client 启动时，无法从 Eureka-Server 读取注册信息（可能挂了），从备份注册中心读取注册信息。目前 Eureka-Client 未提供合适的实现。
    * `#getHeartbeatExecutorThreadPoolSize()` ：心跳执行线程池大小。
    * `#getHeartbeatExecutorExponentialBackOffBound()` ：心跳执行超时后的延迟重试的时间。

## 2.3 DefaultEurekaClientConfig

`com.netflix.discovery.DefaultEurekaClientConfig`，基于**配置文件**的 **Eureka-Client** 配置**实现类**，实现代码如下：

```Java
public class DefaultEurekaClientConfig implements EurekaClientConfig {

    public static final String DEFAULT_ZONE = "defaultZone";

    /**
     * 命名空间
     */
    private final String namespace;
    /**
     * 配置文件对象
     */
    private final DynamicPropertyFactory configInstance;
    /**
     * HTTP 传输配置
     */
    private final EurekaTransportConfig transportConfig;

    public DefaultEurekaClientConfig(String namespace) {
        // 设置 namespace，为 "." 结尾
        this.namespace = namespace.endsWith(".")
                ? namespace
                : namespace + ".";
        // 初始化 配置文件对象
        this.configInstance = Archaius1Utils.initConfig(CommonConstants.CONFIG_FILE_NAME);
        // 创建 HTTP 传输配置
        this.transportConfig = new DefaultEurekaTransportConfig(namespace, configInstance);
    }
}
```

* 类似 PropertiesInstanceConfig，点击[《Eureka 源码解析 —— Eureka-Client 初始化（一）之 EurekaInstanceConfig》「2.4 PropertiesInstanceConfig」](http://www.iocoder.cn/Eureka/eureka-client-init-first/?self)查看详细解析。
* 在 `com.netflix.discovery.PropertyBasedClientConfigConstants` 可以看到配置文件的每个属性 KEY 。
* `transportConfig` 属性，在 [「3. EurekaTransportConfig」](#) 详细解析。

## 2.4 DefaultEurekaClientConfigProvider

`com.netflix.discovery.providers.DefaultEurekaClientConfigProvider`，创建 DefaultEurekaClientConfig 的工厂，实现代码如下：

```Java
public class DefaultEurekaClientConfigProvider implements Provider<EurekaClientConfig> {

    @Inject(optional = true)
    @EurekaNamespace
    private String namespace;

    private DefaultEurekaClientConfig config;
    
    @Override
    public synchronized EurekaClientConfig get() {
        if (config == null) {
            config = (namespace == null)
                    ? new DefaultEurekaClientConfig()
                    : new DefaultEurekaClientConfig(namespace);
                    
            // TODO: Remove this when DiscoveryManager is finally no longer used
            DiscoveryManager.getInstance().setEurekaClientConfig(config);
        }

        return config;
    }
}
```

## 2.5 小结

推荐参考阅读：

* [程序猿DD —— 《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi) Spring Cloud Eureka —— 配置详解
* [风中程序猿 —— 《微服务架构：Eureka参数配置项详解》](http://www.cnblogs.com/fangfuhai/p/7070325.html)

# 3. EurekaTransportConfig

## 3.1 类关系图

EurekaTransportConfig 整体类关系如下图：

![](http://www.iocoder.cn/images/Eureka/2018_04_22/05.png)

* 本文只解析**红圈**部分类。
* EurekaArchaius2TransportConfig 基于 [Netflix Archaius 2.x](https://github.com/Netflix/archaius) 实现，目前还在开发中，因此暂不解析。


## 3.2 配置属性

点击 [EurekaTransportConfig](https://github.com/YunaiV/eureka/blob/3a65b471526e4912829bbfedc29822ba93ef42bb/eureka-client/src/main/java/com/netflix/discovery/shared/transport/EurekaTransportConfig.java) 查看配置属性简介，已经添加中文注释，可以对照着英文注释一起理解。这里笔者摘出部分较为重要的属性：

* `#getSessionedClientReconnectIntervalSeconds()` ：EurekaHttpClient 会话周期性重连时间，单位：秒。在 [《Eureka 源码解析 —— 网络通信》「5.4 SessionedEurekaHttpClient」》](http://www.iocoder.cn/Eureka/transport/?self) 有详细解析。
* `#getRetryableClientQuarantineRefreshPercentage()` ：重试 EurekaHttpClient ，请求失败的 Eureka-Server 隔离集合占比 Eureka-Server 全量集合占比，超过该比例，进行清空。在 [《Eureka 源码解析 —— 网络通信》「5.3 RetryableEurekaHttpClient」》](http://www.iocoder.cn/Eureka/transport/?self) 有详细解析。
* **异步 EndPoint 集群解析器** ：
    * 在 [《Eureka 源码解析 —— EndPoint 与 解析器》「3.6 AsyncResolver」》](http://www.iocoder.cn/Eureka/end-point-and-resolver/?self) 有详细解析。
    * `#getAsyncResolverRefreshIntervalMs()` ：异步解析 EndPoint 集群频率，单位：毫秒。
    * `#getAsyncResolverWarmUpTimeoutMs()` ：异步解析器预热解析 EndPoint 集群超时时间，单位：毫秒。
    * `#getAsyncExecutorThreadPoolSize()` ：异步解析器线程池大小。
* TODO[0028]：写入集群和读取集群。Eureka 2.x 兼容 ：
    * `#getApplicationsResolverDataStalenessThresholdSeconds()`
    * `#applicationsResolverUseIp()`
    * `#getWriteClusterVip()`
    * `#getReadClusterVip()`
    * `#getBootstrapResolverStrategy()`
    * `#useBootstrapResolverForQuery()`

## 3.3 DefaultEurekaTransportConfig

`com.netflix.discovery.shared.transport.DefaultEurekaTransportConfig`，基于**配置文件**的**网络传输**配置**实现类**，实现代码如下：

```Java
public class DefaultEurekaTransportConfig implements EurekaTransportConfig {

    private static final String SUB_NAMESPACE = TRANSPORT_CONFIG_SUB_NAMESPACE + ".";

    /**
     * 命名空间
     */
    private final String namespace;
    /**
     * 配置文件对象
     */
    private final DynamicPropertyFactory configInstance;

    public DefaultEurekaTransportConfig(String parentNamespace, DynamicPropertyFactory configInstance) {
        // 命名空间
        this.namespace = parentNamespace == null
                ? SUB_NAMESPACE
                : (parentNamespace.endsWith(".")
                    ? parentNamespace + SUB_NAMESPACE
                    : parentNamespace + "." + SUB_NAMESPACE);
        // 配置文件对象
        this.configInstance = configInstance;
    }
}
```

* 类似 PropertiesInstanceConfig，点击[《Eureka 源码解析 —— Eureka-Client 初始化（一）之 EurekaInstanceConfig》「2.4 PropertiesInstanceConfig」](http://www.iocoder.cn/Eureka/eureka-client-init-first/?self)查看详细解析。
* 在 `com.netflix.discovery.shared.transport.PropertyBasedTransportConfigConstants` 可以看到配置文件的每个属性 KEY 。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

涉及到配置，内容初看起来会比较多，慢慢理解后，就会变得很“啰嗦”，请保持耐心。

胖友，分享一个朋友圈可好。

