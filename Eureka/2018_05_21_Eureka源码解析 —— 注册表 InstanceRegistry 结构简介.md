title: Eureka 源码解析 —— 注册表 InstanceRegistry 类关系
date: 2018-05-21
tags:
categories: Eureka
permalink: Eureka/instance-registry-class-diagram

---

摘要: 原创出处 http://www.iocoder.cn/Eureka/instance-registry-class-diagram/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Eureka 1.8.X 版本** 

- [1. 概述](http://www.iocoder.cn/Eureka/instance-registry-class-diagram/?toc)
- [2. 类图](http://www.iocoder.cn/Eureka/instance-registry-class-diagram/?toc)
- [3. LookupService](http://www.iocoder.cn/Eureka/instance-registry-class-diagram/?toc)
- [4. LeaseManager](http://www.iocoder.cn/Eureka/instance-registry-class-diagram/?toc)
- [5. InstanceRegistry](http://www.iocoder.cn/Eureka/instance-registry-class-diagram/?toc)
- [6. AbstractInstanceRegistry](http://www.iocoder.cn/Eureka/instance-registry-class-diagram/?toc)
- [7. PeerAwareInstanceRegistry](http://www.iocoder.cn/Eureka/instance-registry-class-diagram/?toc)
- [8. PeerAwareInstanceRegistryImpl](http://www.iocoder.cn/Eureka/instance-registry-class-diagram/?toc)
- [666. 彩蛋](http://www.iocoder.cn/Eureka/instance-registry-class-diagram/?toc)

---

# 1. 概述

本文主要简介 **注册表 InstanceRegistry 的类关系**，为后文的**应用实例注册发现**、**Eureka-Server 集群复制**做整体的铺垫。

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。

# 2. 类图

![](http://www.iocoder.cn/images/Eureka/2018_05_21/01.png)

* `com.netflix.eureka.registry.AwsInstanceRegistry`，主要用于亚马逊 AWS，跳过。
* `com.netflix.eureka.registry.RemoteRegionRegistry`，笔者暂时不太理解它的用途。目前猜测 Eureka-Server 集群和集群之间的注册信息的交互方式。查阅官方资料，[《Add ability to retrieve instances from any remote region》](https://github.com/Netflix/eureka/issues/29) 在做了简单介绍。翻看目前网络上的博客、书籍、项目实战，暂时都没提及此块。估摸和亚马逊 AWS 跨区域( `region` ) 机制有一定关系，先暂时跳过。有了解此块的同学，麻烦告知下笔者，万分感谢。TODO[0009]：RemoteRegionRegistry。
* **蓝框**部分，本文主角。

# 3. LookupService

[`com.netflix.discovery.shared.LookupService`](https://github.com/YunaiV/eureka/blob/3ef162f20a28c75de84321b69412c4ef138ad55a/eureka-client/src/main/java/com/netflix/discovery/shared/LookupService.java)，查找服务**接口**，提供**简单单一**的方式获取应用集合(`com.netflix.discovery.shared.Applications`) 和 应用实例信息集合( `com.netflix.appinfo.InstanceInfo` )。接口代码如下：

```Java
public interface LookupService<T> {

    Application getApplication(String appName);
    
    Applications getApplications();
    
    List<InstanceInfo> getInstancesById(String id);
    
    InstanceInfo getNextServerFromEureka(String virtualHostname, boolean secure);

}
```

![](http://www.iocoder.cn/images/Eureka/2018_04_29/01.png)

* 在 Eureka-Client 里，EurekaClient 继承该接口。
* 在 Eureka-Server 里，`com.netflix.eureka.registry.InstanceRegistry` 继承该接口。

# 4. LeaseManager

`com.netflix.eureka.lease.LeaseManager`，租约管理器**接口**，提供租约的注册、续租、取消( 主动下线 )、过期( 过期下线 )。接口代码如下：

```Java
public interface LeaseManager<T> {

    void register(T r, int leaseDuration, boolean isReplication);
    
    boolean cancel(String appName, String id, boolean isReplication);
    
    boolean renew(String appName, String id, boolean isReplication);
    
    void evict();
    
}
```

# 5. InstanceRegistry

`com.netflix.eureka.registry.InstanceRegistry`，**应用实例**注册表**接口**。它继承了 LookupService 、LeaseManager 接口，提供应用实例的**注册**与**发现**服务。另外，它结合实际业务场景，定义了**更加丰富**的接口方法。接口代码如下：

```Java
public interface InstanceRegistry extends LeaseManager<InstanceInfo>, LookupService<String> {

    // ====== 开启与关闭相关 ======

    void openForTraffic(ApplicationInfoManager applicationInfoManager, int count);
    
    void shutdown();
    
    void clearRegistry();

    // ====== 应用实例状态变更相关 ======
    
    void storeOverriddenStatusIfRequired(String appName, String id, InstanceStatus overriddenStatus);

    boolean statusUpdate(String appName, String id, InstanceStatus newStatus,
                         String lastDirtyTimestamp, boolean isReplication);

    boolean deleteStatusOverride(String appName, String id, InstanceStatus newStatus,
                                 String lastDirtyTimestamp, boolean isReplication);

    Map<String, InstanceStatus> overriddenInstanceStatusesSnapshot();

    // ====== 响应缓存相关 ======

    void initializedResponseCache();

    ResponseCache getResponseCache();
    
    // ====== 自我保护模式相关 ======
    
    long getNumOfRenewsInLastMin();

    int getNumOfRenewsPerMinThreshold();

    int isBelowRenewThresold();
    
    boolean isSelfPreservationModeEnabled();
    
    public boolean isLeaseExpirationEnabled();
    
    // ====== 调试/监控相关 ======
    List<Pair<Long, String>> getLastNRegisteredInstances();

    List<Pair<Long, String>> getLastNCanceledInstances();
}
```

# 6. AbstractInstanceRegistry

`com.netflix.eureka.registry.AbstractInstanceRegistry`，应用对象注册表**抽象实现**。

这里先不拓展开，[《Eureka 源码解析 —— 应用实例注册发现》系列](http://www.iocoder.cn/categories/Eureka/?self) 逐篇分享。

# 7. PeerAwareInstanceRegistry

`com.netflix.eureka.registry.PeerAwareInstanceRegistry`，PeerAware ( 暂时找不到合适的翻译 ) 应用对象注册表**接口**，提供 Eureka-Server 集群内注册信息的**同步**服务。接口代码如下：

```Java
public interface PeerAwareInstanceRegistry extends InstanceRegistry {

    void init(PeerEurekaNodes peerEurekaNodes) throws Exception;
    
    int syncUp();
    
    boolean shouldAllowAccess(boolean remoteRegionRequired);

    void register(InstanceInfo info, boolean isReplication);

    void statusUpdate(final String asgName, final ASGResource.ASGStatus newStatus, final boolean isReplication);
}
```

# 8. PeerAwareInstanceRegistryImpl

`com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl`，PeerAware ( 暂时找不到合适的翻译 ) 应用对象注册表**实现类**。

这里先不拓展开，[《Eureka 源码解析 —— Eureka-Server 集群》系列](http://www.iocoder.cn/categories/Eureka/?self) 逐篇分享。

# 666. 彩蛋

本文是一篇**简介**( 啪啪啪，打脸 )，如果胖友比较着急想了解原理，可以阅读 [携程 ——《深度剖析服务发现组件Netflix Eureka》](http://techshow.ctrip.com/archives/1699.html?from=www.iocoder.cn) 先，写的非常非常非常不错。

快马加鞭，更新 [《Eureka 源码解析 —— 应用实例注册发现 （一）之注册》](http://www.iocoder.cn/Eureka/instance-registry-register/?self) **ing** ...

胖友，分享我的公众号( **芋道源码** ) 给你的胖友可好？

