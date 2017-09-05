title: Elastic-Job-Cloud 源码分析 —— 作业调度（二）
date: 2017-12-28
tags:
categories: Elastic-Job-Cloud
permalink: Elastic-Job/cloud-job-scheduler-and-executor-second

-------

**本文基于 Elastic-Job V2.1.5 版本分享**

- [1. 概述](#)
- [2. 云作业操作](#)
	- [2.1 注册云作业配置](#)
	- [2.2 禁用云作业](#)
	- [2.3 启动云作业](#)
	- [2.4 更新云作业配置](#)
	- [2.5 注销云作业](#)
	- [2.6 触发一次云作业](#)
- [3. 云作业应用操作](#)
	- [3.1 注册云作业应用](#)
	- [3.2 更新云作业应用配置](#)
	- [3.3 禁用云作业应用](#)
	- [3.4 启用云作业应用](#)
	- [3.5 注销云作业应用](#)
- [666. 彩蛋](#)

-------

# 1. 概述

本文主要分享 **Elastic-Job-Cloud 云作业应用配置和云作业配置变更对作业调度的影响**，作为[《Elastic-Job-Cloud 源码分析 —— 作业调度（一）》](http://www.yunai.me/Elastic-Job/cloud-job-scheduler-and-executor-first/?self)的补充内容。所以需要你对**作业调度**已经有一定了解的基础上。

🙂 如果你做作业调度有任何想交流，欢迎加我的公众号( 芋道源码 ) 或 微信( wangwenbin-server ) 交流。

> 你行好事会因为得到赞赏而愉悦  
> 同理，开源项目贡献者会因为 Star 而更加有动力  
> 为 Elastic-Job 点赞！[传送门](https://github.com/dangdangdotcom/elastic-job/stargazers)

# 2. 云作业操作

我们可以使用**运维平台**或 Restful API 对云作业进行操作。前者是对后者的界面包装，如下图所示：

![](http://www.yunai.me/images/Elastic-Job/2017_12_28/01.png)

## 2.1 注册云作业配置

[《Elastic-Job-Cloud 源码分析 —— 作业配置》「3.1.1 操作云作业配置」](http://www.yunai.me/Elastic-Job/cloud-job-config/?self)有详细解析。

## 2.2 禁用云作业

调用 `CloudJobRestfulApi#disable(...)` 方法，禁用云作业，实现代码如下：

```Java
@POST
@Path("/{jobName}/disable")
public void disable(@PathParam("jobName") final String jobName) {
   if (configService.load(jobName).isPresent()) {
       // 将作业放入禁用队列
       facadeService.disableJob(jobName);
       // 停止调度作业
       producerManager.unschedule(jobName);
   }
}
```

* 调用 `FacadeService#disableJob(...)` 方法，将作业放入**禁用作业队列**。实现代码如下：

    ```Java
    // FacadeService.java
    public void disableJob(final String jobName) {
       disableJobService.add(jobName);
    }
    
    // DisableJobService.java
    public void add(final String jobName) {
       if (regCenter.getNumChildren(DisableJobNode.ROOT) > env.getFrameworkConfiguration().getJobStateQueueSize()) {
           log.warn("Cannot add disable job, caused by read state queue size is larger than {}.", env.getFrameworkConfiguration().getJobStateQueueSize());
           return;
       }
       // 将作业放入禁用队列
       String disableJobNodePath = DisableJobNode.getDisableJobNodePath(jobName);
       if (!regCenter.isExisted(disableJobNodePath)) {
           regCenter.persist(disableJobNodePath, jobName);
       }
    }
    
    // DisableJobNode.java
    final class DisableJobNode {
        
        static final String ROOT = StateNode.ROOT + "/disable/job";
        
        private static final String DISABLE_JOB = ROOT + "/%s"; // %s = ${JOB_NAME}
    }
    ```
    * DisableJobService，禁用作业队列服务。
    * 禁用作业队列存储在注册中心( Zookeeper )的**持久**数据节点 `/${NAMESPACE}/state/disable/job/${JOB_NAME}`，存储值为作业名称。使用 zkClient 查看如下： 

        ```Shell
        [zk: localhost:2181(CONNECTED) 6] ls /elastic-job-cloud/state/disable/job
    [test_job_simple]
        ```
* 调用 `ProducerManager#unschedule(...)` 方法，停止调度作业。实现代码如下：

    ```Java
    public void unschedule(final String jobName) {
       // 杀死作业对应的 Mesos 任务们
       for (TaskContext each : runningService.getRunningTasks(jobName)) {
           schedulerDriver.killTask(Protos.TaskID.newBuilder().setValue(each.getId()).build());
       }
       // 将作业从运行时队列删除
       runningService.remove(jobName);
       // 将作业从待运行队列删除
       readyService.remove(Lists.newArrayList(jobName));
       // 停止作业调度
       Optional<CloudJobConfiguration> jobConfig = configService.load(jobName);
       if (jobConfig.isPresent()) {
           transientProducerScheduler.deregister(jobConfig.get());
       }
    }
    ```
    * 调用 `SchedulerDriver#killTask(...)` 方法，杀死作业对应的 Mesos 任务们，适用**常驻作业**。Elastic-Job-Cloud-Scheduler 会接收到 Mesos 杀死任务的请求，调用 `TaskExecutor#killTask(...)` 方法，停止任务调度。实现代码如下：

        ```Java
        // TaskExecutor.java
        public final class TaskExecutor implements Executor {
            
            @Override
            public void killTask(final ExecutorDriver executorDriver, final Protos.TaskID taskID) {
                // 更新 Mesos 任务状态，已杀死。
                executorDriver.sendStatusUpdate(Protos.TaskStatus.newBuilder().setTaskId(taskID).setState(Protos.TaskState.TASK_KILLED).build());
                // 关闭该 Mesos 任务的调度
                DaemonTaskScheduler.shutdown(taskID);
            }
        }
        
        // DaemonTaskScheduler.java
        /**
        * 停止任务调度.
        * 
        * @param taskID 任务主键
        */
        public static void shutdown(final Protos.TaskID taskID) {
           // 移除任务的 Quartz Scheduler
           Scheduler scheduler = RUNNING_SCHEDULERS.remove(taskID.getValue());
           // 关闭任务的 Quartz Scheduler
           if (null != scheduler) {
               try {
                   scheduler.shutdown();
               } catch (final SchedulerException ex) {
                   throw new JobSystemException(ex);
               }
           }
        }
        ```
        * 
    
    * 调用 `RunningService#remove(...)` 方法，将作业从**运行时队列**删除。
    * 调用 `ReadyService#remove(...)` 方法，将作业从**待运行队列**删除。
    * 调用 `TransientProducerScheduler#deregister(...)` 方法，停止作业调度，适用**瞬时作业**。实现代码如下：

        ```Java
        synchronized void deregister(final CloudJobConfiguration jobConfig) {
            // 移除作业
            repository.remove(jobConfig.getJobName());
            // 若 cron 不再对应有作业调度，移除 Quartz Scheduler 对 cron 对应的 Quartz Job
            String cron = jobConfig.getTypeConfig().getCoreConfig().getCron();
            if (!repository.containsKey(buildJobKey(cron))) {
                try {
                    scheduler.unscheduleJob(TriggerKey.triggerKey(cron));
                } catch (final SchedulerException ex) {
                    throw new JobSystemException(ex);
                }
            }
        }
        ```

## 2.3 启动云作业

调用 `CloudJobRestfulApi#enable(...)` 方法，启用云作业，实现代码如下：

```Java
@DELETE
@Path("/{jobName}/disable")
public void enable(@PathParam("jobName") final String jobName) throws JSONException {
   Optional<CloudJobConfiguration> configOptional = configService.load(jobName);
   if (configOptional.isPresent()) {
       // 将作业移出禁用队列
       facadeService.enableJob(jobName);
       // 重新调度作业
       producerManager.reschedule(jobName);
   }
}
```

* 调用 `FacadeService#enableJob(...)` 方法，将作业移出**禁用作业队列**。实现代码如下：

    ```Java
    // FacadeService.java
    public void enableJob(final String jobName) {
       disableJobService.remove(jobName);
    }
    
    // DisableJobService.java
    public void remove(final String jobName) {
       regCenter.remove(DisableJobNode.getDisableJobNodePath(jobName));
    }
    ```

* 调用 `ProducerManager#reschedule(...)` 方法，将作业重新调度。实现代码如下：

    ```Java
    public void reschedule(final String jobName) {
       // 停止调度作业
       unschedule(jobName);
       // 调度作业
       Optional<CloudJobConfiguration> jobConfig = configService.load(jobName);
       if (jobConfig.isPresent()) {
           schedule(jobConfig.get());
       }
    }
    ```
    * 调用 `#unschedule(...)` 方法，停止调度作业。
    * 调用 `#schedule(...)` 方法，调度作业，在[《Elastic-Job-Cloud 源码分析 —— 作业调度（一）》](http://www.iocoder.cn/Elastic-Job/cloud-job-scheduler-and-executor-first/?self)有详细解析。

## 2.4 更新云作业配置

调用 `CloudJobRestfulApi#update(...)` 方法，更新云作业配置，实现代码如下：

```Java
// CloudJobRestfulApi.java
@PUT
@Path("/update")
@Consumes(MediaType.APPLICATION_JSON)
public void update(final CloudJobConfiguration jobConfig) {
   producerManager.update(jobConfig);
}

// ProducerManager.java
public void update(final CloudJobConfiguration jobConfig) {
   Optional<CloudJobConfiguration> jobConfigFromZk = configService.load(jobConfig.getJobName());
   if (!jobConfigFromZk.isPresent()) {
       throw new JobConfigurationException("Cannot found job '%s', please register first.", jobConfig.getJobName());
   }
   // 修改云作业配置
   configService.update(jobConfig);
   // 重新调度作业
   reschedule(jobConfig.getJobName());
}
```

* 调用 `ConfigService#update(jobConfig)` 方法，修改云作业配置。实现代码如下：

    ```Java
    public void update(final CloudJobConfiguration jobConfig) {
       regCenter.update(CloudJobConfigurationNode.getRootNodePath(jobConfig.getJobName()), CloudJobConfigurationGsonFactory.toJson(jobConfig));
    }
    ```

* 调用 `#reschedule(...)` 方法，重新调度作业。

存储在注册中心( Zookeeper )的 云作业配置被更新时，云作业配置变更监听( CloudJobConfigurationListener )会监听到，并执行更新相应逻辑，实现代码如下：

```Java
public final class CloudJobConfigurationListener implements TreeCacheListener {

    @Override
    public void childEvent(final CuratorFramework client, final TreeCacheEvent event) throws Exception {
        String path = null == event.getData() ? "" : event.getData().getPath();
        if (isJobConfigNode(event, path, Type.NODE_ADDED)) {
            // .... 省略无关代码
        } else if (isJobConfigNode(event, path, Type.NODE_UPDATED)) {
            CloudJobConfiguration jobConfig = getJobConfig(event);
            if (null == jobConfig) {
                return;
            }
            // 从待执行队列中删除相关作业
            if (CloudJobExecutionType.DAEMON == jobConfig.getJobExecutionType()) {
                readyService.remove(Collections.singletonList(jobConfig.getJobName()));
            }
            // 设置禁用错过重执行
            if (!jobConfig.getTypeConfig().getCoreConfig().isMisfire()) {
                readyService.setMisfireDisabled(jobConfig.getJobName());
            }
            // 重新调度作业
            producerManager.reschedule(jobConfig.getJobName());
        } else if (isJobConfigNode(event, path, Type.NODE_REMOVED)) {
            // .... 省略无关代码
        }
    }

}
```

* CloudJobConfigurationListener 实现 TreeCacheListener 实现对 Zookeeper 数据变更的监听。对 TreeCacheListener 感兴趣的同学，可以查看 [Apache Curator](https://curator.apache.org/) 相关知识。
* 调用 `ReadyService#remove(...)` 方法，将作业从**待运行队列**删除。TODO，为啥要删除？
* 调用 `ReadyService#setMisfireDisabled(...)` 方法，设置禁用错过重执行。TODO misfired
* 调用 `ProducerManager#reschedule(...)` 方法，重新调度作业。TODO，为啥要重复调用这个方法？

## 2.5 注销云作业

调用 `CloudJobRestfulApi#deregister(...)` 方法，注销云作业，实现代码如下：

```Java
// CloudJobRestfulApi.java
@DELETE
@Path("/deregister")
@Consumes(MediaType.APPLICATION_JSON)
public void deregister(final String jobName) {
   producerManager.deregister(jobName);
}

// ProducerManager.java
public void deregister(final String jobName) {
   Optional<CloudJobConfiguration> jobConfig = configService.load(jobName);
   if (jobConfig.isPresent()) {
       //  从作业禁用队列中删除作业
       disableJobService.remove(jobName);
       // 删除云作业
       configService.remove(jobName);
   }
   // 停止调度作业
   unschedule(jobName);
}
```

* 调用 `DisableJobService#remove(...)` 方法，从**作业禁用队列**中删除作业，实现代码如下：

    ```Java
    public void remove(final String jobName) {
       regCenter.remove(DisableJobNode.getDisableJobNodePath(jobName));
    }
    ```
    
* 调用 `CloudJobConfigurationService#remove(...)` 方法，删除云作业配置。 

* 调用 `#reschedule(...)` 方法，重新调度作业。

存储在注册中心( Zookeeper )的 云作业配置被删除时，**云作业配置变更监听器**( CloudJobConfigurationListener )会监听到，并执行删除相应逻辑，实现代码如下：

```Java
public final class CloudJobConfigurationListener implements TreeCacheListener {
    
    private final CoordinatorRegistryCenter regCenter;
    
    private final ProducerManager producerManager;
    
    private final ReadyService readyService;
    
    public CloudJobConfigurationListener(final CoordinatorRegistryCenter regCenter, final ProducerManager producerManager) {
        this.regCenter = regCenter;
        readyService = new ReadyService(regCenter);
        this.producerManager = producerManager;
    }
    
    @Override
    public void childEvent(final CuratorFramework client, final TreeCacheEvent event) throws Exception {
        String path = null == event.getData() ? "" : event.getData().getPath();
        if (isJobConfigNode(event, path, Type.NODE_ADDED)) {
            // ... 省略无关代码
        } else if (isJobConfigNode(event, path, Type.NODE_UPDATED)) {
            // ... 省略无关代码
        } else if (isJobConfigNode(event, path, Type.NODE_REMOVED)) {
            String jobName = path.substring(CloudJobConfigurationNode.ROOT.length() + 1, path.length());
            producerManager.unschedule(jobName);
        }
    }
}
```

* 调用 `#reschedule(...)` 方法，重新调度作业。

## 2.6 触发一次云作业

调用 `CloudJobRestfulApi#trigger(...)` 方法，触发一次云作业，实现代码如下：

```Java
@POST
@Path("/trigger")
@Consumes(MediaType.APPLICATION_JSON)
public void trigger(final String jobName) {
   // 常驻作业不允许触发一次作业
   Optional<CloudJobConfiguration> config = configService.load(jobName);
   if (config.isPresent() && CloudJobExecutionType.DAEMON == config.get().getJobExecutionType()) {
       throw new JobSystemException("Daemon job '%s' cannot support trigger.", jobName);
   }
   // 将瞬时作业放入待执行队列
   facadeService.addTransient(jobName);
}
```

* 目前**常驻**作业**不支持**触发一次云作业。如果想实现该功能，需要 Elastic-Job-Cloud-Scheduler 通过 Mesos 发送自定义消息，通知 Elastic-Job-Cloud-Executor 触发该作业对应的任务们。
* 调用 `FacadeService#addTransient(...)` 方法，将瞬时作业放入**待执行队列**。当且仅当云作业配置 `JobCoreConfiguration.misfire = true` 时，该作业在**待执行队列**的执行次数不断累积加一。在[《Elastic-Job-Cloud 源码分析 —— 作业调度（一）》「3.2.3 ProducerJob」](http://www.iocoder.cn/Elastic-Job/cloud-job-scheduler-and-executor-first/?self)有详细解析。

# 3. 云作业应用操作

我们可以使用**运维平台**或 Restful API 对云作业应用进行操作。前者是对后者的界面包装，如下图所示：

![](http://www.yunai.me/images/Elastic-Job/2017_12_28/02.png)

## 3.1 注册云作业应用

[《Elastic-Job-Cloud 源码分析 —— 作业配置》「2.2 操作云作业App配置」](http://www.yunai.me/Elastic-Job/cloud-job-config/?self)有详细解析。

## 3.2 更新云作业应用配置

调用 `CloudAppRestfulApi#update(...)` 方法，更新云作业应用配置，实现代码如下：

```Java
// CloudAppRestfulApi.java
@PUT
@Consumes(MediaType.APPLICATION_JSON)
public void update(final CloudAppConfiguration appConfig) {
   appConfigService.update(appConfig);
}

// CloudAppConfigurationService.java
public void update(final CloudAppConfiguration appConfig) {
   regCenter.update(CloudAppConfigurationNode.getRootNodePath(appConfig.getAppName()), CloudAppConfigurationGsonFactory.toJson(appConfig));
}
```

## 3.3 禁用云作业应用

调用 `CloudAppRestfulApi#disable(...)` 方法，禁用云作业应用，实现代码如下：

```Java
@POST
@Path("/{appName}/disable")
public void disable(@PathParam("appName") final String appName) {
   if (appConfigService.load(appName).isPresent()) {
       // 将应用放入禁用队列
       disableAppService.add(appName);
       // 停止应用对应所有作业的调度
       for (CloudJobConfiguration each : jobConfigService.loadAll()) {
           if (appName.equals(each.getAppName())) {
               producerManager.unschedule(each.getJobName());
           }
       }
   }
}
```

* 调用 `DisableAppService#add(...)` 方法，将应用放入**禁用应用队列**，实现代码如下：

    ```Java
    // DisableAppService.java
    public class DisableAppService {
    
        public void add(final String appName) {
            if (regCenter.getNumChildren(DisableAppNode.ROOT) > env.getFrameworkConfiguration().getJobStateQueueSize()) {
                log.warn("Cannot add disable app, caused by read state queue size is larger than {}.", env.getFrameworkConfiguration().getJobStateQueueSize());
                return;
            }
            String disableAppNodePath = DisableAppNode.getDisableAppNodePath(appName);
            if (!regCenter.isExisted(disableAppNodePath)) {
                regCenter.persist(disableAppNodePath, appName);
            }
        }
    }
    
    // DisableAppNode.java
    final class DisableAppNode {
        
        static final String ROOT = StateNode.ROOT + "/disable/app";
        
        private static final String DISABLE_APP = ROOT + "/%s"; // %s = ${APP_NAME}
    }
    ```
    * DisableAppService，禁用应用队列服务。
    * 禁用应用队列存储在注册中心( Zookeeper )的**持久**数据节点 `/${NAMESPACE}/state/disable/app/${APP_NAME}`，存储值为应用名。使用 zkClient 查看如下： 

        ```Shell
        [zk: localhost:2181(CONNECTED) 6] ls /elastic-job-cloud/state/disable/app
    [example_app]
        ```
* 遍历应用对应所有作业，调用 `ProducerManager#unschedule(...)` 方法，停止作业调度。

## 3.4 启用云作业应用

调用 `CloudAppRestfulApi#enable(...)` 方法，启用云作业应用，实现代码如下：

```Java
// CloudAppRestfulApi.java
@DELETE
@Path("/{appName}/disable")
public void enable(@PathParam("appName") final String appName) throws JSONException {
   if (appConfigService.load(appName).isPresent()) {
       // 从禁用应用队列中删除应用
       disableAppService.remove(appName);
       // 重新开启应用对应所有作业的调度
       for (CloudJobConfiguration each : jobConfigService.loadAll()) {
           if (appName.equals(each.getAppName())) {
               producerManager.reschedule(each.getJobName());
           }
       }
   }
}

// DisableAppService.java
public void remove(final String appName) {
   regCenter.remove(DisableAppNode.getDisableAppNodePath(appName));
}
```

## 3.5 注销云作业应用

调用 `CloudAppRestfulApi#deregister(...)` 方法，注销云作业应用，实现代码如下：

```Java
@DELETE
@Path("/{appName}")
@Consumes(MediaType.APPLICATION_JSON)
public void deregister(@PathParam("appName") final String appName) {
   if (appConfigService.load(appName).isPresent()) {
       // 移除应用和应用对应所有作业的配置
       removeAppAndJobConfigurations(appName);
       // 停止应用对应的执行器( Elastic-Job-Cloud-Scheduler )
       stopExecutors(appName);
   }
}
```

* 调用 `#removeAppAndJobConfigurations(...)` 方法，移除应用和应用对应所有作业的配置，实现代码如下：

    ```Java
    private void removeAppAndJobConfigurations(final String appName) {
       // 注销(移除)应用对应所有作业的配置
       for (CloudJobConfiguration each : jobConfigService.loadAll()) {
           if (appName.equals(each.getAppName())) {
               producerManager.deregister(each.getJobName());
           }
       }
       // 从禁用应用队列中删除应用
       disableAppService.remove(appName);
       // 删除云作业App配置
       appConfigService.remove(appName);
    }
    ```

* 调用 `#stopExecutors(...)` 方法，停止应用对应的执行器( Elastic-Job-Cloud-Scheduler )，实现代码如下：

    ```Java
    // CloudAppRestfulApi.java
    private void stopExecutors(final String appName) {
       try {
           Collection<ExecutorStateInfo> executorBriefInfo = mesosStateService.executors(appName);
           for (ExecutorStateInfo each : executorBriefInfo) {
               producerManager.sendFrameworkMessage(ExecutorID.newBuilder().setValue(each.getId()).build(),
                       SlaveID.newBuilder().setValue(each.getSlaveId()).build(), "STOP".getBytes());
           }
       } catch (final JSONException ex) {
           throw new JobSystemException(ex);
       }
    }
    
    // ProducerManager.java 
    public void sendFrameworkMessage(final ExecutorID executorId, final SlaveID slaveId, final byte[] data) {
        schedulerDriver.sendFrameworkMessage(executorId, slaveId, data);
    }
    ```
    * 调用 `SchedulerDriver#sendFrameworkMessage(...)` 方法，通过 Mesos 向云作业应用对应的执行器们( Elastic-Job-Cloud-Executor ) 发送消息为 `"STOP"` 从而关闭执行器。Elastic-Job-Cloud-Executor 会接收到 Mesos 消息，调用 `TaskExecutor#frameworkMessage(...)` 方法，关闭自己。实现代码如下：

    ```Java
    @Override
    public void frameworkMessage(final ExecutorDriver executorDriver, final byte[] bytes) {
        if (null != bytes && "STOP".equals(new String(bytes))) {
            log.error("call frameworkMessage executor stopped.");
            executorDriver.stop();
        }
    }
    ```

# 666. 彩蛋

Elastic-Job-Cloud 作业调度两篇内容到此就结束啦。后续我们会更新大家关心的[《Elastic-Job-Cloud 源码分析 —— 高可用》](http://www.yunai.me?todo)是如何实现的噢。

![](http://www.yunai.me/images/Elastic-Job/2017_12_28/03.png)

道友，赶紧上车，分享一波朋友圈！

