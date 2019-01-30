title: SkyWalking 源码分析 —— Collector Cluster 集群管理
date: 2020-07-20
tags:
categories: SkyWalking
permalink: SkyWalking/collector-cluster-module
wechat_url:
toutiao_url:

-------

摘要: 原创出处 http://www.iocoder.cn/SkyWalking/collector-cluster-module/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 SkyWalking 3.2.6 正式版**

- [1. 概述](http://www.iocoder.cn/SkyWalking/collector-cluster-module/)
- [2. collector-cluster-define](http://www.iocoder.cn/SkyWalking/collector-cluster-module/)
  - [2.1 ClusterModule](http://www.iocoder.cn/SkyWalking/collector-cluster-module/)
  - [2.2 ModuleRegisterService](http://www.iocoder.cn/SkyWalking/collector-cluster-module/)
  - [2.3 ModuleListenerService](http://www.iocoder.cn/SkyWalking/collector-cluster-module/)
  - [2.4 DataMonitor](http://www.iocoder.cn/SkyWalking/collector-cluster-module/)
- [3. collector-cluster-zookeeper-provider](http://www.iocoder.cn/SkyWalking/collector-cluster-module/)
  - [3.1 ClusterModuleZookeeperProvider](http://www.iocoder.cn/SkyWalking/collector-cluster-module/)
  - [3.2 ZookeeperModuleRegisterService](http://www.iocoder.cn/SkyWalking/collector-cluster-module/)
  - [3.3 ZookeeperModuleListenerService](http://www.iocoder.cn/SkyWalking/collector-cluster-module/)
  - [3.4 ClusterZKDataMonitor](http://www.iocoder.cn/SkyWalking/collector-cluster-module/)
  - [3.5 ZookeeperClient](http://www.iocoder.cn/SkyWalking/collector-cluster-module/)
- [4. collector-cluster-standalone-provider](http://www.iocoder.cn/SkyWalking/collector-cluster-module/)
- [5. collector-cluster-redis-provider](http://www.iocoder.cn/SkyWalking/collector-cluster-module/)
- [666. 彩蛋](http://www.iocoder.cn/SkyWalking/collector-cluster-module/)

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

本文主要分享 **SkyWalking Collector Cluster Module**，负责集群的管理，即 Collector 节点的注册于发现。

> 友情提示：建议先阅读 [《SkyWalking 源码分析 —— Collector 初始化》](http://www.iocoder.cn/SkyWalking/collector-init/?self) ，以了解 Collector 组件体系。

Cluster Module 在 SkyWalking 架构图处于如下位置( **红框** ) ：

> FROM https://github.com/apache/incubating-skywalking  
> ![](http://www.iocoder.cn/images/SkyWalking/2020_07_20/01.jpeg)

下面我们来看看整体的项目结构，如下图所示 ：

![](http://www.iocoder.cn/images/SkyWalking/2020_07_20/02.png)

* `collector-cluster-define` ：定义集群管理接口。
* `collector-cluster-standalone-provider` ：基于 H2 的 集群管理实现。**该实现是单机版，建议仅用于 SkyWalking 快速上手，生产环境不建议使用**。
* `collector-cluster-redis-provider` ：基于 Redis 的集群管理实现。*目前暂未完成*。
* `collector-cluster-zookeeper-provider` ：基于 Zookeeper 的集群管理实现。**生产环境推荐使用**

下面，我们从**接口到实现**的顺序进行分享。

# 2. collector-cluster-define

`collector-cluster-define` ：定义集群管理接口。项目结构如下 ：

![](http://www.iocoder.cn/images/SkyWalking/2020_07_20/03.png)

* 交互如下图 ：![](http://www.iocoder.cn/images/SkyWalking/2020_07_20/04.png)

* ModuleListenerService 暴露给其他 Module 注册监听器 ( ClusterModuleListener ) 到 DataMonitor 。
* ModuleRegisterService 暴露给其他 Module 注册组件登记( ModuleRegistration ) 到 DataMonitor 。
* 通过实现 DataMonitor 接口，基于不同的存储器实现注册发现。

## 2.1 ClusterModule

`org.skywalking.apm.collector.cluster.ClusterModule` ，实现 [Module](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Module.java) 抽象类，集群管理 Module 。

[`#name()`](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-cluster/collector-cluster-define/src/main/java/org/skywalking/apm/collector/cluster/ClusterModule.java#L33) **实现**方法，返回模块名为 `"cluster"` 。

[`#services()`](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-cluster/collector-cluster-define/src/main/java/org/skywalking/apm/collector/cluster/ClusterModule.java#L37) **实现**方法，返回 Service 类名：ModuleListenerService / ModuleRegisterService 。

## 2.2 ModuleRegisterService

`org.skywalking.apm.collector.cluster.service.ModuleRegisterService` ，继承 [Service](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Service.java) 接口，模块注册服务**接口**。

[`#register(moduleName, providerName, registration)`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-define/src/main/java/org/skywalking/apm/collector/cluster/service/ModuleRegisterService.java#L38) **接口**方法，注册模块注册信息。一般情况下，实现该接口方法，调用 `DataMonitor#register(path, registration)` 方法。

### 2.2.1 ModuleRegistration

`org.skywalking.apm.collector.cluster.ModuleRegistration` ，模块注册信息**抽象类**。不同 Module 通过实现 ModuleRegistration ，将它们注册到 ModuleRegisterService。目前子类如下 ：![](http://www.iocoder.cn/images/SkyWalking/2020_07_20/05.png)

[`#buildValue()`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-define/src/main/java/org/skywalking/apm/collector/cluster/ModuleRegistration.java) **抽象**方法，获得模块注册信息( [Value](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-define/src/main/java/org/skywalking/apm/collector/cluster/ModuleRegistration.java#L30) )。

## 2.3 ModuleListenerService

`org.skywalking.apm.collector.cluster.service.ModuleListenerService` ，继承 [Service](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Service.java) 接口，注册监听器服务**接口**。

[`#addListener(listener)`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-define/src/main/java/org/skywalking/apm/collector/cluster/service/ModuleListenerService.java#L36) **接口**方法，添加监听器。一般情况下，实现该接口方法，调用 `DataMonitor#addListener(listener)` 方法。

### 2.3.1 ClusterModuleListener

`org.skywalking.apm.collector.cluster.ClusterModuleListener` ，集群组件监听器**抽象类**。目前子类如下 ：![](http://www.iocoder.cn/images/SkyWalking/2020_07_20/11.png)

[**构造方法**](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-define/src/main/java/org/skywalking/apm/collector/cluster/ClusterModuleListener.java#L36)，创建地址数组( `addresses` )。该数组的读写方法如下：

* `#addAddress(address)`
* `#removeAddress(address)`
* `#getAddresses()`

[`#path()`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-define/src/main/java/org/skywalking/apm/collector/cluster/ClusterModuleListener.java#L43) **抽象**方法，返回路径。该路径即为 ClusterModuleListener 监听的**"事件"**。多个 Collector 节点的相同 Module ，**通过路径分组形成集群**。

[`#serverJoinNotify(serverAddress)`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-define/src/main/java/org/skywalking/apm/collector/cluster/ClusterModuleListener.java#L62) / [`#serverQuitNotify(serverAddress)`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-define/src/main/java/org/skywalking/apm/collector/cluster/ClusterModuleListener.java#L69) **抽象**方法，通知服务的加入 / 下线。目前只有 GRPCRemoteSenderService **真正**( 其它都是空方法 )实现该方法，在 [《SkyWalking 源码分析 —— Collector Remote 远程通信服务》「3.2 GRPCRemoteSenderService」](http://www.iocoder.cn/SkyWalking/collector-remote-module) 详细解析。

## 2.4 DataMonitor

`org.skywalking.apm.collector.cluster.DataMonitor` ，数据监**视**器**接口**。通过实现 DataMonitor 接口，基于不同的存储器实现注册发现。目前子类如下 ：![](http://www.iocoder.cn/images/SkyWalking/2020_07_20/06.png)

[`#register(path, registration)`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-define/src/main/java/org/skywalking/apm/collector/cluster/DataMonitor.java#L40) **接口**方法，注册模块注册信息。

[`#addListener(ClusterModuleListener)`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-define/src/main/java/org/skywalking/apm/collector/cluster/DataMonitor.java#L38) **接口**方法，添加监听器。

* [`#getListener(path)`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-define/src/main/java/org/skywalking/apm/collector/cluster/DataMonitor.java#L42) **接口**方法，获得监听**指定路径**的监听器。

[`#setClient(Client)`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-define/src/main/java/org/skywalking/apm/collector/cluster/DataMonitor.java#L36) **接口**方法，设置 [Client](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-component/client-component/src/main/java/org/skywalking/apm/collector/client/Client.java) 。在 [`client-component`](https://github.com/YunaiV/skywalking/tree/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-component/client-component/src/main/java/org/skywalking/apm/collector/client) 有 ZookeeperClient / H2Client / ElasticSearchClient 等多种实现。

* [`BASE_CATALOG`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-define/src/main/java/org/skywalking/apm/collector/cluster/DataMonitor.java#L34) 属性，基础目录为 `"/skywalking"` 。例如说，在 Zookeeper 为根节点的路径。
* [`#createPath(path)`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-define/src/main/java/org/skywalking/apm/collector/cluster/DataMonitor.java#L44) **接口**方法，使用 Client 创建路径。
* [`#setData(path)`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-define/src/main/java/org/skywalking/apm/collector/cluster/DataMonitor.java#L46) **接口**方法，使用 Client 设置路径的值。

# 3. collector-cluster-zookeeper-provider

`collector-cluster-zookeeper-provider` ，基于 Zookeeper 的集群管理实现。项目结构如下 ：![](http://www.iocoder.cn/images/SkyWalking/2020_07_20/07.png)

实际使用时，通过 `application.yml` 配置如下：

``` JSON
cluster:
  zookeeper:
    hostPort: localhost:2181
    sessionTimeout: 100000
```

* 生产环境下，推荐 Zookeeper 配置成集群。

## 3.1 ClusterModuleZookeeperProvider

`org.skywalking.apm.collector.cluster.zookeeper.ClusterModuleZookeeperProvider` ，实现 [ModuleProvider](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/ModuleProvider.java) **抽象类**，基于 Zookeeper 的集群管理服务提供者。

[`#name()`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-zookeeper-provider/src/main/java/org/skywalking/apm/collector/cluster/zookeeper/ClusterModuleZookeeperProvider.java#L53) **实现**方法，返回组件服务提供者名为 `"zookeeper"` 。

[`module()`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-zookeeper-provider/src/main/java/org/skywalking/apm/collector/cluster/zookeeper/ClusterModuleZookeeperProvider.java#L57) **实现**方法，返回组件类为 ClusterModule 。

[`#requiredModules()`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-zookeeper-provider/src/main/java/org/skywalking/apm/collector/cluster/zookeeper/ClusterModuleZookeeperProvider.java#L94) **实现**方法，返回依赖组件为空。

-------

[`#prepare(Properties)`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-zookeeper-provider/src/main/java/org/skywalking/apm/collector/cluster/zookeeper/ClusterModuleZookeeperProvider.java#L61) **实现**方法，执行准备阶段逻辑。

* 第 63 行 ：创建 ClusterZKDataMonitor 对象。
* 第 69 行 ：创建 ZookeeperClient 对象。**注意，此时并未连接 Zookeeper** 。
* 第 71 至 73 行 ：创建 ZookeeperModuleListenerService / ZookeeperModuleRegisterService 对象，并调用 `#registerServiceImplementation()` **父类**方法，注册到 `services` 。

[`#start()`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-zookeeper-provider/src/main/java/org/skywalking/apm/collector/cluster/zookeeper/ClusterModuleZookeeperProvider.java#L76) **实现**方法，执行启动阶段逻辑。

* 第 79 行 ：调用 `ZookeeperClient#initialize()` 方法，初始化 ZookeeperClient ，**此时会连接 Zookeeper**。

[`#notifyAfterCompleted()`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-zookeeper-provider/src/main/java/org/skywalking/apm/collector/cluster/zookeeper/ClusterModuleZookeeperProvider.java#L85) **实现**方法，执行启动完成逻辑。

* 第 88 行 ：调用 `ClusterZKDataMonitor#start()` 方法，启动 ClusterZKDataMonitor 。在本文 [「3.4 ClusterZKDataMonitor」](#) 详细解析。

## 3.2 ZookeeperModuleRegisterService

`org.skywalking.apm.collector.cluster.zookeeper.service.ZookeeperModuleRegisterService` ，基于 Zookeeper 的模块注册服务**实现类**。

[`#register(moduleName, providerName, registration)`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-zookeeper-provider/src/main/java/org/skywalking/apm/collector/cluster/zookeeper/service/ZookeeperModuleRegisterService.java#L38) **实现**方法，调用 `ClusterZKDataMonitor#register(path, registration)` 方法，注册模块注册信息。

## 3.3 ZookeeperModuleListenerService

`org.skywalking.apm.collector.cluster.zookeeper.service.ZookeeperModuleListenerService` ，基于 Zookeeper 的注册监听器服务**实现类**。

[`#addListener(ClusterModuleListener)`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-zookeeper-provider/src/main/java/org/skywalking/apm/collector/cluster/zookeeper/service/ZookeeperModuleListenerService.java#L38) **实现**方法，调用 `ClusterZKDataMonitor#addListener(ClusterModuleListener)` 方法，注册模块注册信息。

## 3.4 ClusterZKDataMonitor

`org.skywalking.apm.collector.cluster.zookeeper.ClusterZKDataMonitor` ，基于 Zookeeper 的数据监视器**实现类**。

在看具体代码实现之前，我们先来看看 Zookeeper 是如何存储数据的，如下图所示 ：![](http://www.iocoder.cn/images/SkyWalking/2020_07_20/08.png)

* 紫色部分，通过调用 [`#createPath(path)`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-zookeeper-provider/src/main/java/org/skywalking/apm/collector/cluster/zookeeper/ClusterZKDataMonitor.java#L172) 方法，顺着路径，逐层创建**持久**节点。
* 黄色部分，通过调用 [`#setData(path)`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-zookeeper-provider/src/main/java/org/skywalking/apm/collector/cluster/zookeeper/ClusterZKDataMonitor.java#L184) 方法，创建**临时**节点，设置 Collector 模块地址。若 Collector 集群有 N 个节点，则此处会有 N 个**临时**节点。
* 打开 `zkClient.sh` ，我们来看一个例子 ：

    ```BASH
    [zk: localhost:2181(CONNECTED) 1] ls /skywalking
    [remote, ui, agent_jetty, agent_gRPC]

    [zk: localhost:2181(CONNECTED) 2] ls /skywalking/ui
    [jetty]

    [zk: localhost:2181(CONNECTED) 3] ls /skywalking/ui/jetty
    [localhost:12800]

    [zk: localhost:2181(CONNECTED) 4] get /skywalking/ui/jetty/localhost:12800
    /
    cZxid = 0x24
    ctime = Thu Dec 14 16:05:25 CST 2017
    mZxid = 0x24
    mtime = Thu Dec 14 16:05:25 CST 2017
    pZxid = 0x24
    cversion = 0
    dataVersion = 0
    aclVersion = 0
    ephemeralOwner = 0x16052d8b9f40006
    dataLength = 1
    numChildren = 0
    ```

-------

[`#register(path, registration)`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-zookeeper-provider/src/main/java/org/skywalking/apm/collector/cluster/zookeeper/ClusterZKDataMonitor.java#L163) **实现**方法，添加到组件注册信息集合( `registrations` )。

[`#start()`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-zookeeper-provider/src/main/java/org/skywalking/apm/collector/cluster/zookeeper/ClusterZKDataMonitor.java#L123) 方法，启动 ClusterZKDataMonitor ，将组件注册信息( `registrations` ) 写到 Zookeeper 中。

-------

[`#addListener(listener)`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-zookeeper-provider/src/main/java/org/skywalking/apm/collector/cluster/zookeeper/ClusterZKDataMonitor.java#L157) **实现**方法，添加到监听器集合( `listeners` )。

[`#process(WatchedEvent)`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-cluster/collector-cluster-zookeeper-provider/src/main/java/org/skywalking/apm/collector/cluster/zookeeper/ClusterZKDataMonitor.java#L123) **实现**方法，处理有 Collector 节点的组件加入或下线。总体逻辑是，从 Zookeeper 获取变更的路径下的地址数组，和本地的地址( `ClusterModuleListener.addresses` )比较，处理加入或移除逻辑的地址。

* ClusterZKDataMonitor 实现 `org.apache.zookeeper.Watcher` **接口**，所以实现该方法。
* 该方法是 `synchronized` 方法，以保证不会出现并发问题。

## 3.5 ZookeeperClient

[`org.skywalking.apm.collector.client.zookeeper.ZookeeperClient`](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-component/client-component/src/main/java/org/skywalking/apm/collector/client/zookeeper/ZookeeperClient.java) ，实现 [Client](https://github.com/YunaiV/skywalking/blob/20d38d7fcbbaac65e10eb8d256881fc9c0cedd87/apm-collector/apm-collector-component/client-component/src/main/java/org/skywalking/apm/collector/client/Client.java) **接口**，Zookeeper 客户端。

代码比较简单，胖友自己阅读理解。

# 4. collector-cluster-standalone-provider

`collector-cluster-standalone-provider.ClusterStandaloneDataMonitor` ，基于 H2 的 集群管理实现。**该实现是单机版，建议仅用于 SkyWalking 快速上手，生产环境不建议使用**。项目结构如下 ：![](http://www.iocoder.cn/images/SkyWalking/2020_07_20/09.png)

大体实现和 `collector-cluster-zookeeper-provider` 差不多，差异在对 DataMonitor 的实现类 ClusterStandaloneDataMonitor 上。

在 ClusterStandaloneDataMonitor 里，实际并未使用 H2Client ，而是基于内存，胖友可以自己查看下。

# 5. collector-cluster-redis-provider

`collector-cluster-redis-provider` ：基于 Redis 的集群管理实现。*目前暂未完成*。

【TODO 4003】等实现后来写写，基于 Redis Pub Sub 保证实时性

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

有一种硬生生把很简单的东西，写的很复杂的感觉。

胖友，分享个朋友圈可好？


