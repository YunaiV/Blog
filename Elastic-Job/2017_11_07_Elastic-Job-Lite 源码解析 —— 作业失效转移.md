title: Elastic-Job-Lite 源码分析 —— 作业失效转移
date: 2017-11-07
tags:
categories: Elastic-Job-Lite
permalink: Elastic-Job/job-failover

-------

![](http://www.yunai.me/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

- [1. 概述](#)
- [2. 作业节点崩溃监听](#)
- [3. 作业失效转移](#)
- [4. 获取作业分片上下文集合](#)
- [5. 监听作业失效转移功能关闭](#)
- [666. 彩蛋](#)

-------

# 1. 概述

本文主要分享 **Elastic-Job-Lite 作业失效转移**。

当作业节点执行作业异常崩溃时，其所分配的作业分片项在下次重新分片之前不会被重新执行。开启失效转移功能后，这部分作业分片项将被其他作业节点抓取后“执行”。为什么此处的执行打引号呢？😈下文我们会分享到噢，卖个关子。

笔者对**失效转移**理解了蛮久时间，因此引用官方对它的解释，让你能更好的理解：

> 来源地址：https://my.oschina.net/u/719192/blog/506062  
> 失效转移： 运行中的作业服务器崩溃不会导致重新分片，只会在下次作业启动时分片。启用失效转移功能可以在本次作业执行过程中，监测其他作业服务器空闲，抓取未完成的孤儿分片项执行。  
> -- 分隔符 --  
> 来源地址：http://dangdangdotcom.github.io/elastic-job/elastic-job-lite/03-design/lite-design/  
> 实现失效转移功能，在某台服务器执行完毕后主动抓取未分配的分片，并且在某台服务器下线后主动寻找可用的服务器执行任务。

这样看概念可能还是比较难理解，代码搞起来！

涉及到主要类的类图如下( [打开大图](http://www.yunai.me/images/Elastic-Job/2017_11_07/01.png) )：

![](http://www.yunai.me/images/Elastic-Job/2017_11_07/01.png)

* 粉色的类在 `com.dangdang.ddframe.job.lite.internal.failover` 包下，实现了 Elastic-Job-Lite 作业失效转移。
* FailoverService，作业失效转移服务。
* FailoverNode，作业失效转移数据存储路径。
* FailoverListenerManager，作业失效转移监听管理器。

> 你行好事会因为得到赞赏而愉悦  
> 同理，开源项目贡献者会因为 Star 而更加有动力  
> 为 Elastic-Job 点赞！[传送门](https://github.com/dangdangdotcom/elastic-job/stargazers)

# 2. 作业节点崩溃监听

当作业节点崩溃时，监听器 JobCrashedJobListener 会监听到该情况，进行作业失效转移处理。

```Java
// JobCrashedJobListener.java
class JobCrashedJobListener extends AbstractJobListener {
   
   @Override
   protected void dataChanged(final String path, final Type eventType, final String data) {
       if (isFailoverEnabled() && Type.NODE_REMOVED == eventType
               && instanceNode.isInstancePath(path)) { // /${JOB_NAME}/instances/${INSTANCE_ID}
           String jobInstanceId = path.substring(instanceNode.getInstanceFullPath().length() + 1);
           if (jobInstanceId.equals(JobRegistry.getInstance().getJobInstance(jobName).getJobInstanceId())) {
               return;
           }
           List<Integer> failoverItems = failoverService.getFailoverItems(jobInstanceId); // /${JOB_NAME}/sharding/${ITEM_ID}/failover
           if (!failoverItems.isEmpty()) {
               for (int each : failoverItems) {
                   failoverService.setCrashedFailoverFlag(each);
                   failoverService.failoverIfNecessary();
               }
           } else {
               for (int each : shardingService.getShardingItems(jobInstanceId)) { // /${JOB_NAME}/sharding/${ITEM_ID}/instance
                   failoverService.setCrashedFailoverFlag(each);
                   failoverService.failoverIfNecessary();
               }
           }
       }
   }
}
```

* 通过判断 `/${JOB_NAME}/instances/${INSTANCE_ID}` 被移除，执行作业失效转移逻辑。❓说好的作业节点**崩溃**呢？经过确认，目前这块存在 BUG，未判断作业节点是否为奔溃。**所以在当前版本，作业失效转移面向的是所有作业节点关闭逻辑，不仅限于作业崩溃关闭。**
* 优先调用 `FailoverService#getFailoverItems(...)` 方法，获得关闭作业节点( `${JOB_INSTANCE_ID}` )对应的 `${JOB_NAME}/sharding/${ITEM_ID}/failover` 作业分片项。

  若该作业分片项为空，再调用 `ShardingService#getShardingItems(...)` 方法，获得关闭作业节点( `${JOB_INSTANCE_ID}` )对应的 `/${JOB_NAME}/sharding/${ITEM_ID}/instance` 作业分片项。
  
  为什么是这样的顺序呢？放在 `FailoverService#failoverIfNecessary()` 一起讲。这里先看下 `FailoverService#getFailoverItems(...)` 方法的实现：

    ```Java
    // FailoverService
    public List<Integer> getFailoverItems(final String jobInstanceId) {
       List<String> items = jobNodeStorage.getJobNodeChildrenKeys(ShardingNode.ROOT);
       List<Integer> result = new ArrayList<>(items.size());
       for (String each : items) {
           int item = Integer.parseInt(each);
           String node = FailoverNode.getExecutionFailoverNode(item); // `${JOB_NAME}/sharding/${ITEM_ID}/failover`
           if (jobNodeStorage.isJobNodeExisted(node) && jobInstanceId.equals(jobNodeStorage.getJobNodeDataDirectly(node))) {
               result.add(item);
           }
       }
       Collections.sort(result);
       return result;
    }
    ```
* 调用 `FailoverService#setCrashedFailoverFlag(...)` 方法，设置失效的分片项标记 `/${JOB_NAME}/leader/failover/items/${ITEM_ID}`。该数据节点为**永久**节点，存储空串( `""` )。

    ```Java
    // FailoverService.java
    public void setCrashedFailoverFlag(final int item) {
       if (!isFailoverAssigned(item)) {
           jobNodeStorage.createJobNodeIfNeeded(FailoverNode.getItemsNode(item)); // /${JOB_NAME}/leader/failover/items/${ITEM_ID}
       }
    }
        
    private boolean isFailoverAssigned(final Integer item) {
       return jobNodeStorage.isJobNodeExisted(FailoverNode.getExecutionFailoverNode(item));
    }
    ```
* 调用 `FailoverService#failoverIfNecessary()` 方法，如果需要失效转移, 则执行作业失效转移。

# 3. 作业失效转移

调用 `FailoverService#failoverIfNecessary()` 方法，如果需要失效转移, 则执行作业失效转移。

```Java
// FailoverService.java
public void failoverIfNecessary() {
   if (needFailover()) {
       jobNodeStorage.executeInLeader(FailoverNode.LATCH, new FailoverLeaderExecutionCallback());
   }
}
```

* 调用 `#needFailover()` 方法，判断是否满足失效转移条件。

    ```Java
    private boolean needFailover() {
                // `${JOB_NAME}/leader/failover/items/${ITEM_ID}` 有失效转移的作业分片项
        return jobNodeStorage.isJobNodeExisted(FailoverNode.ITEMS_ROOT) && !jobNodeStorage.getJobNodeChildrenKeys(FailoverNode.ITEMS_ROOT).isEmpty()
                // 当前作业不在运行中
                && !JobRegistry.getInstance().isJobRunning(jobName);
    }
    ```
    * 条件一：`${JOB_NAME}/leader/failover/items/${ITEM_ID}` 有失效转移的作业分片项。
    * 条件二：当前作业不在运行中。此条件即是上文提交的作业节点**空闲**的定义。
    
        > 失效转移： 运行中的作业服务器崩溃不会导致重新分片，只会在下次作业启动时分片。启用失效转移功能可以在本次作业执行过程中，监测其他作业服务器【空闲】，抓取未完成的孤儿分片项执行

* 调用 `JobNodeStorage#executeInLeader(...)` 方法，使用 `FailoverNode.LATCH`( `/${JOB_NAME}/leader/failover/latch` ) 路径构成的**分布式锁**，保证 FailoverLeaderExecutionCallback 的回调方法同一时间，即使多个作业节点调用，有且仅有一个作业节点进行执行。另外，虽然 `JobNodeStorage#executeInLeader(...)` 方法上带有 `Leader` 关键字，实际非必须在主节点的操作，任何一个拿到**分布式锁**的作业节点都可以调用。目前和**分布式锁**相关的逻辑，在 Elastic-Job-Lite 里，都会调用 `JobNodeStorage#executeInLeader(...)` 方法，数据都存储在 `/leader/` 节点目录下。关于**分布式锁**相关的，在[《Elastic-Job-Lite 源码分析 —— 注册中心》「3.1 在主节点执行操作」](http://www.yunai.me/Elastic-Job/reg-center-zookeeper/?self)有详细分享。

-------

FailoverLeaderExecutionCallback 回调逻辑如下：

```Java
class FailoverLeaderExecutionCallback implements LeaderExecutionCallback {
   
   @Override
   public void execute() {
       // 判断需要失效转移
       if (JobRegistry.getInstance().isShutdown(jobName) || !needFailover()) {
           return;
       }
       // 获得一个 `${JOB_NAME}/leader/failover/items/${ITEM_ID}` 作业分片项
       int crashedItem = Integer.parseInt(jobNodeStorage.getJobNodeChildrenKeys(FailoverNode.ITEMS_ROOT).get(0));
       log.debug("Failover job '{}' begin, crashed item '{}'", jobName, crashedItem);
       // 设置这个 `${JOB_NAME}/sharding/${ITEM_ID}/failover` 作业分片项 为 当前作业节点
       jobNodeStorage.fillEphemeralJobNode(FailoverNode.getExecutionFailoverNode(crashedItem), JobRegistry.getInstance().getJobInstance(jobName).getJobInstanceId());
       // 移除这个 `${JOB_NAME}/leader/failover/items/${ITEM_ID}` 作业分片项
       jobNodeStorage.removeJobNodeIfExisted(FailoverNode.getItemsNode(crashedItem));
       // TODO 不应使用triggerJob, 而是使用executor统一调度 疑问：为什么要用executor统一，后面研究下
       // 触发作业执行
       JobScheduleController jobScheduleController = JobRegistry.getInstance().getJobScheduleController(jobName);
       if (null != jobScheduleController) {
           jobScheduleController.triggerJob();
       }
   }
}
```
* 再次调用 `#needFailover()` 方法，确保经过分布式锁获取等待过程中，仍然需要失效转移。因为可能多个作业节点调用了该回调，第一个作业节点执行了失效转移，可能第二个作业节点就不需要执行失效转移了。
* 调用 `JobNodeStorage#getJobNodeChildrenKeys(FailoverNode.ITEMS_ROOT)#get(0)` 方法，获得**一个** `${JOB_NAME}/leader/failover/items/${ITEM_ID}` 作业分片项。  
  
  调用 `JobNodeStorage#fillEphemeralJobNode(...)` 方法，设置这个**临时**数据节点 `${JOB_NAME}/sharding/${ITEM_ID}failover` 作业分片项为当前作业节点( `${JOB_INSTANCE_ID}` )。  
  
  调用 `JobNodeStorage#removeJobNodeIfExisted(...)` 方法，移除这个`${JOB_NAME}/leader/failover/items/${ITEM_ID}` 作业分片项。
  
* 调用 `JobScheduleController#triggerJob()` 方法，立即启动作业。调用该方法，实际作业不会立即执行，而仅仅是进行触发。如果有多个失效转移的作业分片项，多次调用 `JobScheduleController#triggerJob()` 方法会不会导致作业是**并行执行**的？答案是不会，因为一个作业的 Quartz 线程数设置为 1。

    ```Java
    // JobScheduler.java
    private Properties getBaseQuartzProperties() {
       Properties result = new Properties();
       // ... 省略无关代码
       result.put("org.quartz.threadPool.threadCount", "1"); // Quartz 线程数：1
       // ... 省略无关代码
       return result;
    }
    ```

-------

如果说作业分片项实现转移时，每个作业节点都不处于非空闲状态，岂不是 FailoverLeaderExecutionCallback 一直无法被回调？答案当然不是的。作业在执行完分配给自己的作业分片项，会调用 `LiteJobFacade#failoverIfNecessary()` 方法，进行失效转移的作业分片项抓取：

```Java
public final void execute() {
   // ...  省略无关代码
   
   // 执行 普通触发的作业
   execute(shardingContexts, JobExecutionEvent.ExecutionSource.NORMAL_TRIGGER);
   // 执行 被跳过触发的作业
   while (jobFacade.isExecuteMisfired(shardingContexts.getShardingItemParameters().keySet())) {
       jobFacade.clearMisfire(shardingContexts.getShardingItemParameters().keySet());
       execute(shardingContexts, JobExecutionEvent.ExecutionSource.MISFIRE);
   }
   
   // 执行 作业失效转移
   jobFacade.failoverIfNecessary();
   
   // ...  省略无关代码
}

// LiteJobFacade.java
@Override
public void failoverIfNecessary() {
   if (configService.load(true).isFailover()) {
       failoverService.failoverIfNecessary();
   }
}

// FailoverService.java
public void failoverIfNecessary() {
   if (needFailover()) {
       jobNodeStorage.executeInLeader(FailoverNode.LATCH, new FailoverLeaderExecutionCallback());
   }
}
```

让我们在翻回 JobCrashedJobListener 处代码，为什么获取失效转移的作业分片项是这样的优先顺序？一个作业节点拥有 `${JOB_NAME}/sharding/${ITEM_ID}/failover` 数据分片项，意味着分配给它的作业分片项已经执行完成，否则怎么回调 FailoverLeaderExecutionCallback 方法，抓取失效转移的作业分片项呢？！

![](http://www.yunai.me/images/Elastic-Job/2017_11_07/02.png)

旁白君：双击666，关注笔者公众号一波。

# 4. 获取作业分片上下文集合

在[《Elastic-Job-Lite 源码分析 —— 作业执行》「4.2 获取当前作业服务器的分片上下文」](http://www.yunai.me/Elastic-Job/job-execute/?self)中，我们可以看到作业执行器( AbstractElasticJobExecutor ) 执行作业时，会获取当前作业服务器的分片上下文进行执行。获取过程总体如下顺序图( [打开大图](http://www.yunai.me/images/Elastic-Job/2017_11_07/03.png) )：

![](http://www.yunai.me/images/Elastic-Job/2017_11_07/03.png)

* 红色叉叉在[《Elastic-Job-Lite 源码解析 —— 作业分片》](http://www.yunai.me/Elastic-Job/job-sharding/?self)有详细分享。

实现代码如下：

```Java
// LiteJobFacade.java
@Override
public ShardingContexts getShardingContexts() {
   // 获得 失效转移的作业分片项
   boolean isFailover = configService.load(true).isFailover();
   if (isFailover) {
       List<Integer> failoverShardingItems = failoverService.getLocalFailoverItems();
       if (!failoverShardingItems.isEmpty()) {
           // 【忽略，作业分片详解】获取当前作业服务器分片上下文
           return executionContextService.getJobShardingContext(failoverShardingItems);
       }
   }
   // 【忽略，作业分片详解】作业分片，如果需要分片且当前节点为主节点
   shardingService.shardingIfNecessary();
   // 【忽略，作业分片详解】获得 分配在本机的作业分片项
   List<Integer> shardingItems = shardingService.getLocalShardingItems();
   // 移除 分配在本机的失效转移的作业分片项目
   if (isFailover) {
       shardingItems.removeAll(failoverService.getLocalTakeOffItems());
   }
   // 移除 被禁用的作业分片项
   shardingItems.removeAll(executionService.getDisabledItems(shardingItems));
   // 【忽略，作业分片详解】获取当前作业服务器分片上下文
   return executionContextService.getJobShardingContext(shardingItems);
}
```

* 调用 `FailoverService#getLocalFailoverItems()` 方法，获取运行在本作业节点的失效转移分片项集合。

    ```Java
    // FailoverService.java
    public List<Integer> getLocalFailoverItems() {
       if (JobRegistry.getInstance().isShutdown(jobName)) {
           return Collections.emptyList();
       }
       return getFailoverItems(JobRegistry.getInstance().getJobInstance(jobName).getJobInstanceId()); // `${JOB_NAME}/sharding/${ITEM_ID}/failover`
    }
    ```

* 调用 `ExecutionContextService#getJobShardingContext()` 方法，获取当前作业服务器分片上下文。在[《Elastic-Job-Lite 源码解析 —— 作业分片》「4. 获取作业分片上下文集合」](http://www.yunai.me/Elastic-Job/job-sharding/?self)有详细解析。
* 当本作业节点不存在抓取的失效转移分片项，则获得分配给本作业分解的作业分片项。此时你会看到略奇怪的方法调用，`shardingItems.removeAll(failoverService.getLocalTakeOffItems())`。为什么呢？举个例子，作业节点A持有作业分片项[0, 1]，此时异常断网，导致[0, 1]被作业节点B失效转移抓取，此时若作业节点A恢复，作业分片项[0, 1]依然属于作业节点A，但是可能已经在作业节点B执行，因此需要进行移除，避免多节点运行相同的作业分片项。`FailoverService#getLocalTakeOffItems()` 方法实现代码如下：

    ```Java
    // FailoverService.java
    /**
    * 获取运行在本作业服务器的被失效转移的序列号.
    * 
    * @return 运行在本作业服务器的被失效转移的序列号
    */
    public List<Integer> getLocalTakeOffItems() {
       List<Integer> shardingItems = shardingService.getLocalShardingItems();
       List<Integer> result = new ArrayList<>(shardingItems.size());
       for (int each : shardingItems) {
           if (jobNodeStorage.isJobNodeExisted(FailoverNode.getExecutionFailoverNode(each))) {
               result.add(each);
           }
       }
       return result;
    }
    ```

# 5. 监听作业失效转移功能关闭

```Java
class FailoverSettingsChangedJobListener extends AbstractJobListener {
        
   @Override
   protected void dataChanged(final String path, final Type eventType, final String data) {
       if (configNode.isConfigPath(path) && Type.NODE_UPDATED == eventType
               && !LiteJobConfigurationGsonFactory.fromJson(data).isFailover()) { // 关闭失效转移功能
           failoverService.removeFailoverInfo();
       }
   }
}
```

# 666. 彩蛋

旁白君：啊啊啊，有点绕。  
芋道君：耐心，耐心，耐心。

![](http://www.yunai.me/images/Elastic-Job/2017_11_07/04.png)

道友，赶紧上车，分享一波朋友圈！


