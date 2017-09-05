title: Elastic-Job-Lite 源码分析 —— 注册中心监听器
date: 2017-10-14
tags:
categories: Elastic-Job-Lite
permalink: Elastic-Job/reg-center-zookeeper-listener

-------

**本文基于 Elastic-Job V2.1.5 版本分享**

- [1. 概述](#)
- [2. ListenerManager](#)
- [3. AbstractListenerManager](#)
- [4. AbstractJobListener](#)
- [5. RegistryCenterConnectionStateListener](#)
- [666. 彩蛋](#)

-------

![](http://www.yunai.me/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

# 1. 概述

本文主要分享 **Elastic-Job-Lite 注册中心监听器**。

建议前置阅读：

* [《Elastic-Job-Lite 源码分析 —— 注册中心》](http://www.yunai.me/Elastic-Job/reg-center-zookeeper/?self)

涉及到主要类的类图如下( [打开大图](http://www.yunai.me/images/Elastic-Job/2017_10_14/01.png) )：

![](http://www.yunai.me/images/Elastic-Job/2017_10_14/01.png)

> 你行好事会因为得到赞赏而愉悦  
> 同理，开源项目贡献者会因为 Star 而更加有动力  
> 为 Elastic-Job 点赞！[传送门](https://github.com/dangdangdotcom/elastic-job/stargazers)

# 2. ListenerManager

ListenerManager，作业注册中心的监听器管理者。管理者**两类**组件：

* 监听管理器
* 注册中心连接状态监听器

其中**监听管理器**管理着自己的作业注册中心监听器。

一起从代码层面看看：

```Java
public final class ListenerManager {
    
    private final JobNodeStorage jobNodeStorage;
    
    private final ElectionListenerManager electionListenerManager;
    
    private final ShardingListenerManager shardingListenerManager;
    
    private final FailoverListenerManager failoverListenerManager;
    
    private final MonitorExecutionListenerManager monitorExecutionListenerManager;
    
    private final ShutdownListenerManager shutdownListenerManager;
    
    private final TriggerListenerManager triggerListenerManager;
    
    private final RescheduleListenerManager rescheduleListenerManager;

    private final GuaranteeListenerManager guaranteeListenerManager;
    
    private final RegistryCenterConnectionStateListener regCenterConnectionStateListener;
}
```

* 第一类：`electionListenerManager` / `shardingListenerManager` / `failoverListenerManager` / `MonitorExecutionListenerManager` / `shutdownListenerManager` / `triggerListenerManager` / `rescheduleListenerManager` / `guaranteeListenerManager` 是不同服务的**监听管理器**，都继承**作业注册中心的监听器管理者的抽象类**( AbstractListenerManager )。我们以下一篇文章会涉及到的**分片监听管理器**( ShardingListenerManager ) 来瞅瞅内部整体实现：

    ```Java
    public final class ShardingListenerManager extends AbstractListenerManager {
        @Override
        public void start() {
            addDataListener(new ShardingTotalCountChangedJobListener());
            addDataListener(new ListenServersChangedJobListener());
        }
        
        class ShardingTotalCountChangedJobListener extends AbstractJobListener {
            // .... 省略方法
        }
        
        class ListenServersChangedJobListener extends AbstractJobListener {
            // .... 省略方法
        }
    }
    ```
    * ShardingListenerManager 内部管理了 ShardingTotalCountChangedJobListener / ListenServersChangedJobListener 两个作业注册中心监听器。具体作业注册中心监听器是什么，有什么用途，下文会详细解析。
* 第二类：`regCenterConnectionStateListener` 是注册中心连接状态监听器。下文也会详细解析。

在[《Elastic-Job-Lite 源码分析 —— 作业初始化》「3.2.4」注册作业启动信息](http://www.yunai.me/Elastic-Job/job-init?self)，我们看到作业初始化时，会开启所有注册中心监听器：

```Java
// SchedulerFacade.java
/**
* 注册作业启动信息.
* 
* @param enabled 作业是否启用
*/
public void registerStartUpInfo(final boolean enabled) {
   // 开启 所有监听器
   listenerManager.startAllListeners();
   // .... 省略方法
}

// ListenerManager.java
/**
* 开启所有监听器.
*/
public void startAllListeners() {
   // 开启 不同服务监听管理器
   electionListenerManager.start();
   shardingListenerManager.start();
   failoverListenerManager.start();
   monitorExecutionListenerManager.start();
   shutdownListenerManager.start();
   triggerListenerManager.start();
   rescheduleListenerManager.start();
   guaranteeListenerManager.start();
   // 开启 注册中心连接状态监听器
   jobNodeStorage.addConnectionStateListener(regCenterConnectionStateListener);
}
```

# 3. AbstractListenerManager

AbstractListenerManager，作业注册中心的监听器管理者的**抽象类**。

```Java
public abstract class AbstractListenerManager {
    
    private final JobNodeStorage jobNodeStorage;
    
    protected AbstractListenerManager(final CoordinatorRegistryCenter regCenter, final String jobName) {
        jobNodeStorage = new JobNodeStorage(regCenter, jobName);
    }

    /**
     * 开启监听器.
     */
    public abstract void start();

    /**
     * 添加注册中心监听器
     *
     * @param listener 注册中心监听器
     */
    protected void addDataListener(final TreeCacheListener listener) {
        jobNodeStorage.addDataListener(listener);
    }
}
```

* `#addDataListener()`，将作业注册中心的监听器添加到注册中心 TreeCache 的监听者里。`JobNodeStorage#addDataListener(...)` 在[《Elastic-Job-Lite 源码分析 —— 作业初始化》「2.2」缓存](http://www.yunai.me/Elastic-Job/reg-center-zookeeper/?self)已经详细解析。
* 子类实现 `#start()` 方法实现监听器初始化。目前所有子类的实现都是将自己管理的注册中心监听器调用 `#addDataListener(...)`，还是以 ShardingListenerManager 举例子：

    ```Java
    public final class ShardingListenerManager extends AbstractListenerManager {
    
        @Override
        public void start() {
            addDataListener(new ShardingTotalCountChangedJobListener());
            addDataListener(new ListenServersChangedJobListener());
        }
    
    }
    ```

# 4. AbstractJobListener

AbstractJobListener，作业注册中心的监听器**抽象类**。

```Java
public abstract class AbstractJobListener implements TreeCacheListener {
    
    @Override
    public final void childEvent(final CuratorFramework client, final TreeCacheEvent event) throws Exception {
        ChildData childData = event.getData();
        // 忽略掉非数据变化的事件，例如 event.type 为 CONNECTION_SUSPENDED、CONNECTION_RECONNECTED、CONNECTION_LOST、INITIALIZED 事件
        if (null == childData) {
            return;
        }
        String path = childData.getPath();
        if (path.isEmpty()) {
            return;
        }
        dataChanged(path, event.getType(), null == childData.getData() ? "" : new String(childData.getData(), Charsets.UTF_8));
    }

    /**
     * 节点数据变化
     *
     * @param path 节点路径
     * @param eventType 事件类型
     * @param data 数据
     */
    protected abstract void dataChanged(final String path, final Type eventType, final String data);
}
```

* 作业注册中心的监听器**实现类**实现 `#dataChanged(...)`，对节点数据变化进行处理。
* `#childEvent(...)` 屏蔽掉非节点数据变化事件，例如：CONNECTION_SUSPENDED、CONNECTION_RECONNECTED、CONNECTION_LOST、INITIALIZED 事件，只处理 NODE_ADDED、NODE_UPDATED、NODE_REMOVED 事件。

我们再拿 ShardingListenerManager 举例子：

```Java
public final class ShardingListenerManager extends AbstractListenerManager {

    class ShardingTotalCountChangedJobListener extends AbstractJobListener {
        
        @Override
        protected void dataChanged(final String path, final Type eventType, final String data) {
            if (configNode.isConfigPath(path) && 0 != JobRegistry.getInstance().getCurrentShardingTotalCount(jobName)) {
                int newShardingTotalCount = LiteJobConfigurationGsonFactory.fromJson(data).getTypeConfig().getCoreConfig().getShardingTotalCount();
                if (newShardingTotalCount != JobRegistry.getInstance().getCurrentShardingTotalCount(jobName)) {
                    shardingService.setReshardingFlag();
                    JobRegistry.getInstance().setCurrentShardingTotalCount(jobName, newShardingTotalCount);
                }
            }
        }
    }
    
    class ListenServersChangedJobListener extends AbstractJobListener {
        
        @Override
        protected void dataChanged(final String path, final Type eventType, final String data) {
            if (!JobRegistry.getInstance().isShutdown(jobName) && (isInstanceChange(eventType, path) || isServerChange(path))) {
                shardingService.setReshardingFlag();
            }
        }
        
        private boolean isInstanceChange(final Type eventType, final String path) {
            return instanceNode.isInstancePath(path) && Type.NODE_UPDATED != eventType;
        }
        
        private boolean isServerChange(final String path) {
            return serverNode.isServerPath(path);
        }
    }

}
```

* 在[《Elastic-Job-Lite 源码解析 —— 任务分片》](http://www.yunai.me/Elastic-Job/job-sharding/?self)详细解析。

# 5. RegistryCenterConnectionStateListener

RegistryCenterConnectionStateListener，实现 Curator ConnectionStateListener 接口，注册中心连接状态监听器。

```Java
public final class RegistryCenterConnectionStateListener implements ConnectionStateListener {

    @Override
    public void stateChanged(final CuratorFramework client, final ConnectionState newState) {
        if (JobRegistry.getInstance().isShutdown(jobName)) {
            return;
        }
        JobScheduleController jobScheduleController = JobRegistry.getInstance().getJobScheduleController(jobName);
        if (ConnectionState.SUSPENDED == newState || ConnectionState.LOST == newState) { // Zookeeper 连接终端 或 连接丢失
            // 暂停作业调度
            jobScheduleController.pauseJob();
        } else if (ConnectionState.RECONNECTED == newState) { // Zookeeper 重新连上
            // 持久化作业服务器上线信息
            serverService.persistOnline(serverService.isEnableServer(JobRegistry.getInstance().getJobInstance(jobName).getIp()));
            // 持久化作业运行实例上线相关信息
            instanceService.persistOnline();
            // 清除本地分配的作业分片项运行中的标记
            executionService.clearRunningInfo(shardingService.getLocalShardingItems());
            // 恢复作业调度
            jobScheduleController.resumeJob();
        }
    }
    
}
```

* 当注册中心连接 SUSPENDED 或 LOST 时，暂停**本地**作业调度：

    ```Java
    // JobScheduleController.java
    public synchronized void pauseJob() {
       try {
           if (!scheduler.isShutdown()) {
               scheduler.pauseAll();
           }
       } catch (final SchedulerException ex) {
           throw new JobSystemException(ex);
       }
    }
    ```
* 当注册中心重新连接成功( RECONNECTED )，恢复**本地**作业调度：

    ```Java
    /**
    * 恢复作业.
    */
    public synchronized void resumeJob() {
      try {
          if (!scheduler.isShutdown()) {
              scheduler.resumeAll();
          }
      } catch (final SchedulerException ex) {
          throw new JobSystemException(ex);
      }
    }
    ```

# 666. 彩蛋

旁白君：芋道君，你又水更了！  
芋道君：是是是，是是是！

![](http://www.yunai.me/images/Elastic-Job/2017_10_14/02.png)

道友，赶紧上车，分享一波朋友圈！

