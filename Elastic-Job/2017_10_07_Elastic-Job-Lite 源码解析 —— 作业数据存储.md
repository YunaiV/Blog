title: Elastic-Job-Lite 源码分析 —— 作业数据存储
date: 2017-10-07
tags:
categories: Elastic-Job-Lite
permalink: Elastic-Job/job-storage

-------

**本文基于 Elastic-Job V2.1.5 版本分享**

- [1. 概述](#1-%E6%A6%82%E8%BF%B0)
- [2. JobNodePath](#2-jobnodepath)
- [3. JobNodeStorage](#3-jobnodestorage)
- [4. ConfigurationNode](#4-configurationnode)
- [5. ServerNode](#5-servernode)
- [6. InstanceNode](#6-instancenode)
- [7. ShardingNode](#7-shardingnode)
- [8. LeaderNode](#8-leadernode)
- [9. FailoverNode](#9-failovernode)
- [10. GuaranteeNode](#10-guaranteenode)
- [666. 彩蛋](#666-%E5%BD%A9%E8%9B%8B)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

# 1. 概述

本文主要分享 **Elastic-Job-Lite 作业数据存储**。

涉及到主要类的类图如下( [打开大图](http://www.iocoder.cn/images/Elastic-Job/2017_10_07/01.png) )：

![](http://www.iocoder.cn/images/Elastic-Job/2017_10_07/01.png)

> 你行好事会因为得到赞赏而愉悦  
> 同理，开源项目贡献者会因为 Star 而更加有动力  
> 为 Elastic-Job 点赞！[传送门](https://github.com/dangdangdotcom/elastic-job/stargazers)

# 2. JobNodePath

JobNodePath，作业节点路径类。**作业节点是在普通的节点前加上作业名称的前缀**。

在 Zookeeper 看一个作业的数据存储：

``` bash
[zk: localhost:2181(CONNECTED) 65] ls /elastic-job-example-lite-java/javaSimpleJob
[leader, servers, config, instances, sharding]
```

* `elastic-job-example-lite-java`：作业节点集群名，使用 `ZookeeperConfiguration.namespace` 属性配置。
* `javaSimpleJob`：作业名字，使用 `JobCoreConfiguration.jobName` 属性配置。
* `config` / `servers` / `instances` / `sharding` / `leader`：不同服务的数据存储节点路径。

JobNodePath，注释很易懂，点击[链接](https://github.com/dangdangdotcom/elastic-job/blob/7dc099541a16de49f024fc59e46377a726be7f6b/elastic-job-lite/elastic-job-lite-core/src/main/java/com/dangdang/ddframe/job/lite/internal/storage/JobNodePath.java)查看。这里我们梳理下 JobNodePath 和**其它节点路径类**的关系：

| Zookeeper 路径 | JobNodePath 静态属性 | JobNodePath 方法 | 节点路径类 |
| :--- | :--- | :--- | :--- |
| `config` | CONFIG_NODE | `#getConfigNodePath()` | ConfigurationNode |
| `servers` | SERVERS_NODE | `#getServerNodePath()` | ServerNode |
| `instances` | INSTANCES_NODE | `#getInstancesNodePath()` | InstanceNode |
| `sharding` | SHARDING_NODE | `#getShardingNodePath()` | ShardingNode |
| `leader` | / | `#getFullPath(node)` | LeaderNode |
| `leader/failover` | / | `#getFullPath(node)` | FailoverNode |
| `guarantee` | / | `#getFullPath(node)` | GuaranteeNode |

# 3. JobNodeStorage

JobNodeStorage，作业节点数据访问类。

Elastic-Job-Lite 使用**注册中心**存储作业节点数据，JobNodeStorage 对注册中心提供的方法做下简单的封装提供调用。举个例子：

```Java
// JobNodeStorage.java
private final CoordinatorRegistryCenter regCenter;
private final JobNodePath jobNodePath;

/**
* 判断作业节点是否存在.
* 
* @param node 作业节点名称
* @return 作业节点是否存在
*/
public boolean isJobNodeExisted(final String node) {
   return regCenter.isExisted(jobNodePath.getFullPath(node));
}

// JobNodePath.java
/**
* 获取节点全路径.
* 
* @param node 节点名称
* @return 节点全路径
*/
public String getFullPath(final String node) {
   return String.format("/%s/%s", jobName, node);
}
```

* 传递的参数 `node` 只是简单的**作业节点名称**，通过调用 `JobNodePath#getFullPath(...)` 方法获取节点全路径。
* 其它方法类似，有兴趣的同学点击[链接](https://github.com/dangdangdotcom/elastic-job/blob/7dc099541a16de49f024fc59e46377a726be7f6b/elastic-job-lite/elastic-job-lite-core/src/main/java/com/dangdang/ddframe/job/lite/internal/storage/JobNodePath.java)查看。

# 4. ConfigurationNode

ConfigurationNode，配置节点路径。

在 Zookeeper 看一个作业的**配置**节点数据存储：

``` bash
[zk: localhost:2181(CONNECTED) 67] get /elastic-job-example-lite-java/javaSimpleJob/config
{"jobName":"javaSimpleJob","jobClass":"com.dangdang.ddframe.job.example.job.simple.JavaSimpleJob","jobType":"SIMPLE","cron":"0/5 * * * * ?","shardingTotalCount":3,"shardingItemParameters":"0\u003dBeijing,1\u003dShanghai,2\u003dGuangzhou","jobParameter":"","failover":true,"misfire":true,"description":"","jobProperties":{"job_exception_handler":"com.dangdang.ddframe.job.executor.handler.impl.DefaultJobExceptionHandler","executor_service_handler":"com.dangdang.ddframe.job.executor.handler.impl.DefaultExecutorServiceHandler"},"monitorExecution":true,"maxTimeDiffSeconds":-1,"monitorPort":-1,"jobShardingStrategyClass":"","reconcileIntervalMinutes":10,"disabled":false,"overwrite":true}
```

* `/config` 是**持久**节点，存储Lite作业配置( LiteJobConfiguration ) JSON化字符串。

ConfigurationNode 代码如下：

```Java
public final class ConfigurationNode {
    
    static final String ROOT = "config";
}
```

ConfigurationNode 如何读取、存储，在[《Elastic-Job-Lite 源码分析 —— 作业配置》的「3.」作业配置服务](http://www.iocoder.cn/Elastic-Job/job-config/?self)已经详细解析。

# 5. ServerNode

ServerNode，服务器节点路径。

在 Zookeeper 看一个作业的**服务器**节点数据存储： 

``` bash
[zk: localhost:2181(CONNECTED) 72] ls /elastic-job-example-lite-java/javaSimpleJob/servers
[192.168.16.164, 169.254.93.156, 192.168.252.57, 192.168.16.137, 192.168.3.2, 192.168.43.31]
[zk: localhost:2181(CONNECTED) 73] get /elastic-job-example-lite-java/javaSimpleJob/servers/192.168.16.164

```

* `/servers/` 目录下以 `IP` 为数据节点路径存储每个服务器节点。如果**相同IP**服务器有多个服务器节点，只存储一个 `IP` 数据节点。
* `/servers/${IP}` 是**持久**节点，不存储任何信息，只是空串( `""`);

ServerNode 代码如下：

```Java
public final class ServerNode {
    
    /**
     * 服务器信息根节点.
     */
    public static final String ROOT = "servers";
    
    private static final String SERVERS = ROOT + "/%s";
}
```

ServerNode 如何存储，在[《Elastic-Job-Lite 源码分析 —— 作业初始化》的「3.2.4」注册作业启动信息](http://www.iocoder.cn/Elastic-Job/job-init/?self)已经详细解析。

# 6. InstanceNode

InstanceNode，运行实例节点路径。

在 Zookeeper 看一个作业的**运行实例**节点数据存储： 

```bash
[zk: localhost:2181(CONNECTED) 81] ls /elastic-job-example-lite-java/javaSimpleJob/instances
[192.168.16.137@-@56010]
[zk: localhost:2181(CONNECTED) 82] get /elastic-job-example-lite-java/javaSimpleJob/instances

```

* `/instances` 目录下以作业实例主键( `JOB_INSTANCE_ID` ) 为数据节点路径存储每个运行实例节点。
* `/instances/${JOB_INSTANCE_ID}` 是**临时**节点，不存储任何信息，只是空串( `""`);
* `JOB_INSTANCE_ID` 生成方式：

    ```Java
    // JobInstance.java
    
    jobInstanceId = IpUtils.getIp()
                    + DELIMITER
                    + ManagementFactory.getRuntimeMXBean().getName().split("@")[0]; // PID
    ```

InstanceNode 代码如下：

```Java
public final class InstanceNode {
    
    /**
     * 运行实例信息根节点.
     */
    public static final String ROOT = "instances";
    
    private static final String INSTANCES = ROOT + "/%s";
    
    /**
     * 获取当前运行实例节点路径
     *
     * @return 当前运行实例节点路径
     */
    String getLocalInstanceNode() {
        return String.format(INSTANCES, JobRegistry.getInstance().getJobInstance(jobName).getJobInstanceId());
    }
}
```

InstanceNode 如何存储，在[《Elastic-Job-Lite 源码分析 —— 作业初始化》的「3.2.4」注册作业启动信息](http://www.iocoder.cn/Elastic-Job/job-init/?self)已经详细解析。

# 7. ShardingNode

ShardingNode，分片节点路径。

在 Zookeeper 看一个作业的**分片**节点数据存储： 

``` bash
[zk: localhost:2181(CONNECTED) 1] ls /elastic-job-example-lite-java/javaSimpleJob/sharding
[0, 1, 2]
[zk: localhost:2181(CONNECTED) 2] ls /elastic-job-example-lite-java/javaSimpleJob/sharding/0
[running, instance, misfire]
[zk: localhost:2181(CONNECTED) 3] get /elastic-job-example-lite-java/javaSimpleJob/sharding/0/instance
192.168.16.137@-@56010
```

* `/sharding/${ITEM_ID}` 目录下以作业分片项序号( `ITEM_ID` ) 为数据节点路径存储作业分片项的 `instance` / `running` / `misfire` / `disable` **数据节点**信息。
* `/sharding/${ITEM_ID}/instance` 是**临时**节点，存储该作业分片项**分配到的作业实例主键**( `JOB_INSTANCE_ID` )。在[《Elastic-Job-Lite 源码分析 —— 作业分片》](http://www.iocoder.cn/Elastic-Job/job-sharding/?self)详细解析。
* `/sharding/${ITEM_ID}/running` 是**临时**节点，当该作业分片项**正在运行**，存储空串( `""` )；当该作业分片项**不在运行**，移除该数据节点。[《Elastic-Job-Lite 源码分析 —— 作业执行》的「4.6」执行普通触发的作业](http://www.iocoder.cn/Elastic-Job/job-init/?self)已经详细解析。
* `/sharding/${ITEM_ID}/misfire` 是**永久节点**，当该作业分片项**被错过执行**，存储空串( `""` )；当该作业分片项重新执行，移除该数据节点。[《Elastic-Job-Lite 源码分析 —— 作业执行》的「4.7」执行被错过触发的作业](http://www.iocoder.cn/Elastic-Job/job-init/?self)已经详细解析。
* `/sharding/${ITEM_ID}/disable` 是**永久节点**，当该作业分片项**被禁用**，存储空串( `""` )；当该作业分片项**被开启**，移除数据节点。

ShardingNode，代码如下：

```Java
public final class ShardingNode {
    
    /**
     * 执行状态根节点.
     */
    public static final String ROOT = "sharding";
    
    static final String INSTANCE_APPENDIX = "instance";
    
    public static final String INSTANCE = ROOT + "/%s/" + INSTANCE_APPENDIX;
    
    static final String RUNNING_APPENDIX = "running";
    
    static final String RUNNING = ROOT + "/%s/" + RUNNING_APPENDIX;
    
    static final String MISFIRE = ROOT + "/%s/misfire";
    
    static final String DISABLED = ROOT + "/%s/disabled";
    
    static final String LEADER_ROOT = LeaderNode.ROOT + "/" + ROOT;
    
    static final String NECESSARY = LEADER_ROOT + "/necessary";
    
    static final String PROCESSING = LEADER_ROOT + "/processing";
}
```

* LEADER_ROOT / NECESSARY / PROCESSING 放在「4.7」**LeaderNode** 解析。

# 8. LeaderNode

LeaderNode，主节点路径。

在 `leader` 目录下一共有三个存储子节点：

* `election`：主节点选举。
* `sharding`：作业分片项分配。
* `failover`：作业失效转移。

**主节点选举**

在 Zookeeper 看一个作业的 **`leader/election`** 节点数据存储： 

```bash
[zk: localhost:2181(CONNECTED) 1] ls /elastic-job-example-lite-java/javaSimpleJob/leader/election
[latch, instance]
[zk: localhost:2181(CONNECTED) 2] get /elastic-job-example-lite-java/javaSimpleJob/leader/election/instance
192.168.16.137@-@1910
```

* `/leader/election/instance` 是**临时**节点，当作业集群完成选举后，存储主作业实例主键( `JOB_INSTANCE_ID` )。
* `/leader/election/latch` 主节点选举分布式锁，是 Apache Curator 针对 Zookeeper 实现的**分布式锁**的一种，笔者暂未了解存储形式，无法解释。在[《Elastic-Job-Lite 源码分析 —— 注册中心》的「3.1」在主节点执行操作](http://www.iocoder.cn/Elastic-Job/reg-center-zookeeper/?self)进行了简单解析。

**作业分片项分配**

在 Zookeeper 看一个作业的 **`leader/sharding`** 节点数据存储： 

```bash
[zk: localhost:2181(CONNECTED) 1] ls /elastic-job-example-lite-java/javaSimpleJob/leader/sharding
[necessary, processing]
[zk: localhost:2181(CONNECTED) 2] 个get /elastic-job-example-lite-java/javaSimpleJob/leader/sharding

[zk: localhost:2181(CONNECTED) 3] 个get /elastic-job-example-lite-java/javaSimpleJob/leader/processing

```

* `/leader/sharding/necessary` 是**永久节点**，当**相同作业**有新的作业节点加入或者移除时，存储空串( `""` )，标记需要进行作业分片项重新分配；当重新分配完成后，移除该数据节点。
* `/leader/sharding/processing` 是**临时节点**，当开始重新分配作业分片项时，存储空串( `""` )，标记正在进行重新分配；当重新分配完成后，移除该数据节点。
* 当且仅当作业节点为主节点时，才可以执行作业分片项分配，[《Elastic-Job-Lite 源码分析 —— 作业分片》](http://www.iocoder.cn/Elastic-Job/job-sharding/?self)详细解析。

**作业失效转移**

作业失效转移数据节点在 FailoverNode，放在「9」**FailoverNode** 解析。

这里大家可能会和我一样比较疑惑，为什么 `/leader/failover` 放在 `/leader` 目录下，而不独立成为一个根目录？经过确认，**作业失效转移** 设计到分布式锁，统一存储在 `/leader` 目录下。

-------

LeaderNode，代码如下：

```Java
public final class LeaderNode {
    
    /**
     * 主节点根路径.
     */
    public static final String ROOT = "leader";
    
    static final String ELECTION_ROOT = ROOT + "/election";
    
    static final String INSTANCE = ELECTION_ROOT + "/instance";
    
    static final String  LATCH = ELECTION_ROOT + "/latch";
}
```

# 9. FailoverNode

FailoverNode，失效转移节点路径。

在 Zookeeper 看一个作业的**失效转移**节点数据存储： 

```Java
[zk: localhost:2181(CONNECTED) 2] ls /elastic-job-example-lite-java/javaSimpleJob/leader/failover
[latch, items]
[zk: localhost:2181(CONNECTED) 4] ls /elastic-job-example-lite-java/javaSimpleJob/leader/failover/items
[0]
```

* `/leader/failover/latch` 作业失效转移分布式锁，和 `/leader/failover/latch` 是一致的。
* `/leader/items/${ITEM_ID}` 是**永久节点**，当某台作业节点 CRASH 时，其分配的作业分片项标记需要进行失效转移，存储其分配的作业分片项的 `/leader/items/${ITEM_ID}` 为空串( `""` )；当失效转移标记，移除 `/leader/items/${ITEM_ID}`，存储 `/sharding/${ITEM_ID}/failover` 为空串( `""` )，**临时**节点，需要进行失效转移执行。[《Elastic-Job-Lite 源码分析 —— 作业失效转移》](http://www.iocoder.cn/Elastic-Job/job-failover/?self)详细解析。

FailoverNode 代码如下：

```Java
public final class FailoverNode {
    
    static final String FAILOVER = "failover";
    
    static final String LEADER_ROOT = LeaderNode.ROOT + "/" + FAILOVER;
    
    static final String ITEMS_ROOT = LEADER_ROOT + "/items";
        
    static final String ITEMS = ITEMS_ROOT + "/%s";
        
    static final String LATCH = LEADER_ROOT + "/latch";
        
    private static final String EXECUTION_FAILOVER = ShardingNode.ROOT + "/%s/" + FAILOVER;
    
    static String getItemsNode(final int item) {
        return String.format(ITEMS, item);
    }
    
    static String getExecutionFailoverNode(final int item) {
        return String.format(EXECUTION_FAILOVER, item);
    }
}
```

# 10. GuaranteeNode

GuaranteeNode，保证分布式任务全部开始和结束状态节点路径。在[《Elastic-Job-Lite 源码分析 —— 作业监听器》](http://www.iocoder.cn/Elastic-Job/job-listener/?self)详细解析。

# 666. 彩蛋

旁白君：芋道君，你又水更了！  
芋道君：屁屁屁，劳资怼死你！如下是作业数据存储整理，哼哼哈兮！

![](http://www.iocoder.cn/images/Elastic-Job/2017_10_07/02.png)

道友，赶紧上车，分享一波朋友圈！

