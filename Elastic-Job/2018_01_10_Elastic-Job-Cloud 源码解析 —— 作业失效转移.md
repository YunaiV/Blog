title: Elastic-Job-Cloud 源码分析 —— 作业失效转移
date: 2018-01-10
tags:
categories: Elastic-Job-Cloud
permalink: Elastic-Job/cloud-job-failover

-------

**本文基于 Elastic-Job V2.1.5 版本分享**

- [1. 概述](#)
- [2. 记录作业失效转移](#)
- [3. 提交失效转移作业](#)
- [666. 彩蛋](#)

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

本文主要分享 **Elastic-Job-Cloud 作业失效转移**。对应到 Elastic-Job-Lite 源码解析文章为[《Elastic-Job-Lite 作业作业失效转移》](http://www.iocoder.cn/Elastic-Job/job-failover/?self)。

你需要对[《Elastic-Job-Cloud 源码分析 —— 作业调度（一）》](http://www.iocoder.cn/Elastic-Job/cloud-job-scheduler-and-executor-first/?self)有一定的了解。

当作业任务在 Elastic-Job-Cloud-Executor 异常崩溃时，该任务在下次调度之前不会被重新执行。开启失效转移功能后，该作业任务会立即被 Elastic-Job-Cloud-Scheduler 重新调度，提交 Elastic-Job-Cloud-Executor **立即**执行。

在 Elastic-Job-Cloud 里，我们了解到作业分成**瞬时**作业和**常驻**作业。实际上面失效转移的定义暂时只适用于**瞬时**作业。对于**常驻**作业，作业任务异常崩溃后，无论你是否开启失效转移功能，Elastic-Job-Cloud-Scheduler 会立刻提交 Elastic-Job-Cloud-Executor **重新调度**执行。

**为什么此处使用的是“重新调度”，而不是“立即执行”呢**？目前版本 Elasitc-Job-Cloud 暂时不支持**常驻**作业的失效转移，当作业任务异常崩溃，本次执行**不会重新执行**，但是为了作业任务后续能够调度执行，所以再次提交 Elastic-Job-Cloud-Scheduler。

> 你行好事会因为得到赞赏而愉悦  
> 同理，开源项目贡献者会因为 Star 而更加有动力  
> 为 Elastic-Job 点赞！[传送门](https://github.com/dangdangdotcom/elastic-job/stargazers)

OK，下面我们来看看作业失效转移的实现方式和作业任务异常崩溃的多重场景。

# 2. 记录作业失效转移

当作业任务异常崩溃时，Elastic-Job-Cloud-Scheduler 通过 Mesos 任务状态变更接口( `#statusUpdate()` )实现对任务状态的监听处理，实现代码如下：

```Java
public final class SchedulerEngine implements Scheduler {
    @Override
    public void statusUpdate(final SchedulerDriver schedulerDriver, final Protos.TaskStatus taskStatus) {
        String taskId = taskStatus.getTaskId().getValue();
        TaskContext taskContext = TaskContext.from(taskId);
        String jobName = taskContext.getMetaInfo().getJobName();
        log.trace("call statusUpdate task state is: {}, task id is: {}", taskStatus.getState(), taskId);
        jobEventBus.post(new JobStatusTraceEvent(jobName, taskContext.getId(), taskContext.getSlaveId(), Source.CLOUD_SCHEDULER, 
                taskContext.getType(), String.valueOf(taskContext.getMetaInfo().getShardingItems()), State.valueOf(taskStatus.getState().name()), taskStatus.getMessage()));
        switch (taskStatus.getState()) {
            case TASK_RUNNING:
                // ... 省略无关代码
                break;
            case TASK_FINISHED:
                // ... 省略无关代码
                break;
            case TASK_KILLED:
                // ... 省略无关代码
                break;
            case TASK_LOST:
            case TASK_DROPPED:
            case TASK_GONE:
            case TASK_GONE_BY_OPERATOR:
            case TASK_FAILED: // 执行作业任务被错误终止
            case TASK_ERROR: // 任务错误
                log.warn("task id is: {}, status is: {}, message is: {}, source is: {}", taskId, taskStatus.getState(), taskStatus.getMessage(), taskStatus.getSource());
                // 将任务从运行时队列删除
                facadeService.removeRunning(taskContext);
                // 记录失效转移队列
                facadeService.recordFailoverTask(taskContext);
                // 通知 TaskScheduler 任务不分配在对应主机上
                unAssignTask(taskId);
                // 统计
                statisticManager.taskRunFailed();
                break;
            case TASK_UNKNOWN:
            case TASK_UNREACHABLE:
                log.error("task id is: {}, status is: {}, message is: {}, source is: {}", taskId, taskStatus.getState(), taskStatus.getMessage(), taskStatus.getSource());
                statisticManager.taskRunFailed();
                break;
            default:
                break;
        }
    }
}
```

一共有 6 种状态判定为作业任务崩溃，我们来一个一个看看：

* TASK_DROPPED / TASK_GONE / TASK_GONE_BY_OPERATOR
    
    这三个状态，笔者暂时不太了解，这里先引用一些资料，欢迎有了解的同学指教一下。    
    
    > FROM http://mesos.apache.org/api/latest/java/org/apache/mesos/Protos.TaskState.html  
    > **TASK_DROPPED**：The task failed to launch because of a transient error.  
    > **TASK_GONE**：The task is no longer running.  
    > **TASK_GONE_BY_OPERATOR**：The task was running on an agent that the master cannot contact; the operator has asserted that the agent has been shutdown, but this has not been directly confirmed by the master.  
    >   
    > FROM http://mesos.apache.org/blog/mesos-1-1-0-released/  
    > [MESOS-5344] - Experimental support for partition-aware Mesos frameworks. In previous Mesos releases, when an agent is partitioned from the master and then reregisters with the cluster, all tasks running on the agent are terminated and the agent is shutdown. In Mesos 1.1, partitioned agents will no longer be shutdown when they reregister with the master. By default, tasks running on such agents will still be killed (for backward compatibility); however, frameworks can opt-in to the new PARTITION_AWARE capability. If they do this, their tasks will not be killed when a partition is healed. This allows frameworks to define their own policies for how to handle partitioned tasks. Enabling the PARTITION_AWARE capability also introduces a new set of task states: TASK_UNREACHABLE, TASK_DROPPED, TASK_GONE, TASK_GONE_BY_OPERATOR, and TASK_UNKNOWN. **These new states are intended to eventually replace the TASK_LOST state**.

* TASK_FAILED

    执行作业任务被**错误**终止。例如，执行器( Elastic-Job-Cloud-Executor )异常崩溃，或者被杀死。

* TASK_ERROR

    任务启动尝试失败错误。例如，执行器( Elastic-Job-Cloud-Executor ) 接收到的任务的作业配置不正确。实现代码如下：
    
    ```Java
       @Override
   public void run() {
       // 更新 Mesos 任务状态，运行中。
       executorDriver.sendStatusUpdate(Protos.TaskStatus.newBuilder().setTaskId(taskInfo.getTaskId()).setState(Protos.TaskState.TASK_RUNNING).build());
       //
       Map<String, Object> data = SerializationUtils.deserialize(taskInfo.getData().toByteArray());
       ShardingContexts shardingContexts = (ShardingContexts) data.get("shardingContext");
       @SuppressWarnings("unchecked")
       JobConfigurationContext jobConfig = new JobConfigurationContext((Map<String, String>) data.get("jobConfigContext"));
       try {
           // 获得 分布式作业
           ElasticJob elasticJob = getElasticJobInstance(jobConfig);
           // 调度器提供内部服务的门面对象
           final CloudJobFacade jobFacade = new CloudJobFacade(shardingContexts, jobConfig, jobEventBus);
           // 执行作业
           if (jobConfig.isTransient()) {
               // 执行作业
               JobExecutorFactory.getJobExecutor(elasticJob, jobFacade).execute();
               // 更新 Mesos 任务状态，已完成。
               executorDriver.sendStatusUpdate(Protos.TaskStatus.newBuilder().setTaskId(taskInfo.getTaskId()).setState(Protos.TaskState.TASK_FINISHED).build());
           } else {
               // 初始化 常驻作业调度器
               new DaemonTaskScheduler(elasticJob, jobConfig, jobFacade, executorDriver, taskInfo.getTaskId()).init();
           }
           // CHECKSTYLE:OFF
       } catch (final Throwable ex) {
           // CHECKSTYLE:ON
           log.error("Elastic-Job-Cloud-Executor error", ex);
           // 更新 Mesos 任务状态，错误。
           executorDriver.sendStatusUpdate(Protos.TaskStatus.newBuilder().setTaskId(taskInfo.getTaskId()).setState(Protos.TaskState.TASK_ERROR).setMessage(ExceptionUtil.transform(ex)).build());
           // 停止自己
           executorDriver.stop();
           throw ex;
       }
   }
    ```
    * 调用 `#getElasticJobInstance()` 方法，因为任务的作业配置不正确抛出**异常**。例如，任务类不存在；Spring 的 配置文件不存在；Spring 容器初始化出错；Spring Bean 对象初始化或获取出错；以及等等。
    * **瞬时**作业，调用 `AbstractElasticJobExecutor#execute(...)` 方法，发生**异常**，并且**异常被抛出**。默认情况下，AbstractElasticJobExecutor 内部使用 DefaultJobExceptionHandler 处理发生的异常，**不会抛出异常**，实现代码如下：

        ```Java
        public final class DefaultJobExceptionHandler implements JobExceptionHandler {
        
            @Override
            public void handleException(final String jobName, final Throwable cause) {
                log.error(String.format("Job '%s' exception occur in job processing", jobName), cause);
            }
        }
        ```
        * 
    
    * **常驻**作业，调用 `DaemonTaskScheduler#(...)` 方法，初始化发生**异常**。
    * 因为上述的种种异常，调用 `ExecutorDriver#sendStatusUpdate(...)`，更新 Mesos 任务状态为 TASK_ERROR。另外，调用 `ExecutorDriver#stop()` 方法，关闭自己。**这意味着，一个执行器上如果存在一个作业任务发生 TASK_ERROR，其他作业任务即使是正常的，也会更新作业任务状态为 TASK_FAILED**。这块千万要注意。

* TASK_LOST

    执行作业任务的 Elastic-Job-Cloud-Executor 所在的 Mesos Slave 与 Mesos Master 因为**网络问题或 Mesos Slave 崩溃**引起丢失连接，**可能**导致其上的所有作业任务状态变为 TASK_LOST。

    **当 Slave 宕机后重启，导致 TASK_LOST 时，Mesos又是怎么来处理的呢？**
    
    > FROM http://dockone.io/article/2513  
    > 在 Master 和 Slave 之间，一般都是由 Master 主动向每一个 Slave 发送Ping消息，如果在设定时间内（flag.slave_ping_timeout，默认15s）没有收到Slave 的回复，并且达到一定次数（flag.max_slave_ping_timeouts，默认次数为5），那么 Master 会操作以下几个步骤：  
    > 1. 将该 Slave 从 Master 中删除，此时该 Slave 的资源将不会再分配给Scheduler。  
    > 2. 遍历该 Slave 上运行的所有任务，向对应的 Framework 发送任务的 Task_Lost 状态更新，同时把这些任务从Master中删除。  
    > 3. 遍历该 Slave 上的所有 Executor，并删除。  
    > 4. 触发 Rescind Offer，把这个 Slave 上已经分配给 Scheduler 的 Offer 撤销。  
    > 5. 把这个 Slave 从 Master 的 Replicated log 中删除（Mesos Master 依赖 Replicated log 中的部分持久化集群配置信息进行 failer over / recovery）。 
    
    * 必须 Slave 进行重启，因为对执行器的相关操作只能通过 Mesos Slave，即 **Scheduler <=> Mesos Master <=> Mesos Slave <=> Executor**。如果 Slave 一直不进行重启，执行器会一直运行，除非有另外的机制，**通知**到执行器。

    But..................  
    笔者尝试如上流程，使用 `kill -9` 模拟 Mesos Slave 异常崩溃，等待 Mesos Master 发现 Mesos Slave 已经关闭，**重启 Mesos Slave**，结果执行器( Elastic-Job-Cloud-Executor )未关闭，调度器( Elastic-Job-Cloud-Scheduler )并未收到任务的 TASK_LOST。？？？什么情况？？？翻查如下文档：
    
    * [《Mesos 官方文档 —— high-availability-framework-guide》](http://mesos.apache.org/documentation/latest/high-availability-framework-guide/)搜索标题 "Dealing with Partitioned or Failed Agents"。
    * [《Mesos 官方文档 —— agent-recovery》](http://mesos.apache.org/documentation/latest/agent-recovery/)搜索关标题 "Agent Recovery"。

   因为 Elastic-Job-Cloud-Scheduler 注册到 Mesos Master 时，开启了 `checkpoint` 和 `PARTITION_AWARE`。开启 `checkpoint` 后，Mesos Slave 会将记录**检查点**信息， Mesos Slave 重启后，会读取检查点检查信息，**重新连接上( 不会关闭 )**运行在它上面的执行器( Elastic-Job-Cloud-Scheduler )。开启 `PARTITION_AWARE` 后，TASK_LOST 会被区分成 TASK_UNREACHABLE, TASK_DROPPED, TASK_GONE, TASK_GONE_BY_OPERATOR, and TASK_UNKNOWN。表现如下：
   
   * `kill -9` 模拟 Mesos Slave 异常崩溃，等待 Mesos Master 发现 Mesos Slave 已经关闭
   * 调度器( Elastic-Job-Cloud-Scheduler ) 接收直接由 Mesos Master 发送的该 Mesos Slave 上的每个任务 TASK_UNREACHABLE。
   * Mesos Slave 重启完成。
   * 执行器( Elastic-Job-Cloud-Executor ) 重新注册到重启好的 Mesos Slave ，并继续运行任务。
  
  如果 Elastic-Job-Cloud-Scheduler 注册到 Mesos Master 时，关闭了 `PARTITION_AWARE` 和 `checkpoint`，表现同 **TASK_LOST** 描述的过程。

  开启 `checkpoint` 和 `PARTITION_AWARE` 实现代码如下：
      
      ```Java
      // SchedulerService.java
      private SchedulerDriver getSchedulerDriver(final TaskScheduler taskScheduler, final JobEventBus jobEventBus, final FrameworkIDService frameworkIDService) {
            Protos.FrameworkInfo.Builder builder = Protos.FrameworkInfo.newBuilder();
            // PARTITION_AWARE
            builder.addCapabilitiesBuilder().setType(Protos.FrameworkInfo.Capability.Type.PARTITION_AWARE);
            Protos.FrameworkInfo frameworkInfo = builder.setUser(mesosConfig.getUser()).setName(frameworkName)
                .setHostname(mesosConfig.getHostname()).setFailoverTimeout(FRAMEWORK_FAILOVER_TIMEOUT_SECONDS)
                .setWebuiUrl(WEB_UI_PROTOCOL + env.getFrameworkHostPort())
                .setCheckpoint(true) // checkpoint
                .build();
            // ... 省略无关代码
       }
      ```

  **是不是开启了 `checkpoint`，Mesos Slave 重启不会关闭执行器？**
   
   答案当然是不是的。当 Mesos Slave 配置 `recover = cleanup` 或者 重启时间超过 `recovery_timeout` ( 默认，15 分钟 )时，重启完成后，Mesos Slave 关闭运行在它上面的执行器( Elastic-Job-Cloud-Executor )，调度器( Elastic-Job-Cloud-Scheduler ) 接收到的该 Mesos Slave 上的每个任务 TASK_FAILED。
   
   * 参考文档：[《Mesos 官方文档 —— agent-recovery》](http://mesos.apache.org/documentation/latest/agent-recovery/)搜索标题 "Agent Configuration"。

![](http://www.iocoder.cn/images/Elastic-Job/2018_01_10/01.png)

-------

调用 `FacadeService#recordFailoverTask(...)` 方法，记录失效转移队列，实现代码如下：

```Java
public void recordFailoverTask(final TaskContext taskContext) {
   Optional<CloudJobConfiguration> jobConfigOptional = jobConfigService.load(taskContext.getMetaInfo().getJobName());
   if (!jobConfigOptional.isPresent()) {
       return;
   }
   if (isDisable(jobConfigOptional.get())) {
       return;
   }
   CloudJobConfiguration jobConfig = jobConfigOptional.get();
   if (jobConfig.getTypeConfig().getCoreConfig().isFailover() // 开启失效转移
           || CloudJobExecutionType.DAEMON == jobConfig.getJobExecutionType()) { // 常驻作业
       failoverService.add(taskContext);
   }
}
```

* 对于**瞬时**作业，必须开启 `JobCoreConfiguration.failover = true`，才能失效转移，这个比较好理解。
* 对于**常驻**作业，暂时不支持失效转移。因为常驻作业是在执行器( Elastic-Job-Executor ) 进行调度执行，如果不添加到失效转移作业队列，重新提交到执行器( Elastic-Job-Executor )，后续就不能调度执行该作业了。
* 调用 `FailoverService#add(...)` 方法，将任务放入失效转移队列，实现代码如下：

```Java
// FailoverService.java
public void add(final TaskContext taskContext) {
   if (regCenter.getNumChildren(FailoverNode.ROOT) > env.getFrameworkConfiguration().getJobStateQueueSize()) {
       log.warn("Cannot add job, caused by read state queue size is larger than {}.", env.getFrameworkConfiguration().getJobStateQueueSize());
       return;
   }
   String failoverTaskNodePath = FailoverNode.getFailoverTaskNodePath(taskContext.getMetaInfo().toString());
   if (!regCenter.isExisted(failoverTaskNodePath) // 判断不在失效转移队列
           && !runningService.isTaskRunning(taskContext.getMetaInfo())) { // 判断不在运行中
       regCenter.persist(failoverTaskNodePath, taskContext.getId());
   }
}

// FailoverNode.java
final class FailoverNode {
    
    static final String ROOT = StateNode.ROOT + "/failover";
    
    private static final String FAILOVER_JOB = ROOT + "/%s"; // %s=${JOB_NAME}
    
    private static final String FAILOVER_TASK = FAILOVER_JOB + "/%s"; // %s=${TASK_META_INFO}
}
```

* FailoverService，失效转移队列服务。
* **失效转移队列**存储在注册中心( Zookeeper )的**持久**数据节点 `/${NAMESPACE}/state/failover/${JOB_NAME}/${TASK_META_INFO}`，存储值为任务编号。使用 zkClient 查看如下：

    ```shell
    [zk: localhost:2181(CONNECTED) 2] ls /elastic-job-cloud/state/failover/test_job_simple
[test_job_simple@-@0]
[zk: localhost:2181(CONNECTED) 3] get /elastic-job-cloud/state/failover/test_job_simple/test_job_simple@-@0
test_job_simple@-@0@-@READY@-@4da72be3-43d5-4f02-9d7e-45feb30b8fcb-S2@-@8f2a5bb5-2941-4ece-b192-0f936e60faa7
    ```
* 在运维平台，我们可以看到失效转移队列：
    
    ![](http://www.iocoder.cn/images/Elastic-Job/2018_01_10/02.png) 

# 3. 提交失效转移作业

在[《Elastic-Job-Cloud 源码分析 —— 作业调度（一）》「4.1 创建 Fenzo 任务请求」](http://www.iocoder.cn/Elastic-Job/cloud-job-scheduler-and-executor-first/?self)里，调用 `FacadeService#getEligibleJobContext()` 方法，获取有资格运行的作业时。`FacadeService#getEligibleJobContext()` 不仅调用 `ReadyService#getAllEligibleJobContexts(...)` 方法，从**待执行队列**中获取所有有资格执行的作业上下文，也调用 `FailoverService#getAllEligibleJobContexts()` 方法，从**失效转移队列**中获取所有有资格执行的作业上下文。实现代码如下：

```Java
// FailoverService.java
public Collection<JobContext> getAllEligibleJobContexts() {
   // 不存在 失效转移队列
   if (!regCenter.isExisted(FailoverNode.ROOT)) {
       return Collections.emptyList();
   }
   // 获取 失效转移队列 的作业们
   List<String> jobNames = regCenter.getChildrenKeys(FailoverNode.ROOT);
   Collection<JobContext> result = new ArrayList<>(jobNames.size());
   Set<HashCode> assignedTasks = new HashSet<>(jobNames.size() * 10, 1);
   for (String each : jobNames) {
       // 为空时，移除 失效转移队列 的作业
       List<String> taskIdList = regCenter.getChildrenKeys(FailoverNode.getFailoverJobNodePath(each));
       if (taskIdList.isEmpty()) {
           regCenter.remove(FailoverNode.getFailoverJobNodePath(each));
           continue;
       }
       // 排除 作业配置 不存在的作业
       Optional<CloudJobConfiguration> jobConfig = configService.load(each);
       if (!jobConfig.isPresent()) {
           regCenter.remove(FailoverNode.getFailoverJobNodePath(each));
           continue;
       }
       // 获得待执行的分片集合
       List<Integer> assignedShardingItems = getAssignedShardingItems(each, taskIdList, assignedTasks);
       //
       if (!assignedShardingItems.isEmpty() && jobConfig.isPresent()) {
           result.add(new JobContext(jobConfig.get(), assignedShardingItems, ExecutionType.FAILOVER));    
       }
   }
   return result;
}
    
private List<Integer> getAssignedShardingItems(final String jobName, final List<String> taskIdList, final Set<HashCode> assignedTasks) {
   List<Integer> result = new ArrayList<>(taskIdList.size());
   for (String each : taskIdList) {
       TaskContext.MetaInfo metaInfo = TaskContext.MetaInfo.from(each);
       if (assignedTasks.add(Hashing.md5().newHasher().putString(jobName, Charsets.UTF_8).putInt(metaInfo.getShardingItems().get(0)).hash()) // 排重
               && !runningService.isTaskRunning(metaInfo)) { // 排除正在运行中
           result.add(metaInfo.getShardingItems().get(0));
       }
   }
   return result;
}
```

-------

在[《Elastic-Job-Cloud 源码分析 —— 作业调度（一）》「4.4 创建 Mesos 任务信息」](http://www.iocoder.cn/Elastic-Job/cloud-job-scheduler-and-executor-first/?self)里，调用 `LaunchingTasks#getIntegrityViolationJobs()` 方法，获得作业分片不完整的作业集合。实现代码如下：

```Java
// LaunchingTasks.java
/**
* 获得作业分片不完整的作业集合
*
* @param vmAssignmentResults 主机分配任务结果集合
* @return 作业分片不完整的作业集合
*/
Collection<String> getIntegrityViolationJobs(final Collection<VMAssignmentResult> vmAssignmentResults) {
   Map<String, Integer> assignedJobShardingTotalCountMap = getAssignedJobShardingTotalCountMap(vmAssignmentResults);
   Collection<String> result = new HashSet<>(assignedJobShardingTotalCountMap.size(), 1);
   for (Map.Entry<String, Integer> entry : assignedJobShardingTotalCountMap.entrySet()) {
       JobContext jobContext = eligibleJobContextsMap.get(entry.getKey());
       if (ExecutionType.FAILOVER != jobContext.getType() // 不包括 FAILOVER 执行类型的作业
               && !entry.getValue().equals(jobContext.getJobConfig().getTypeConfig().getCoreConfig().getShardingTotalCount())) {
           log.warn("Job {} is not assigned at this time, because resources not enough to run all sharding instances.", entry.getKey());
           result.add(entry.getKey());
       }
   }
   return result;
}
```

* 一个作业可能存在部分分片需要失效转移，不需要考虑完整性。

-------

在[《Elastic-Job-Cloud 源码分析 —— 作业调度（一）》「4.7 从队列中删除已运行的作业」](http://www.iocoder.cn/Elastic-Job/cloud-job-scheduler-and-executor-first/?self)里，调用 `FailoverService#remove(...)` 方法，从失效转移队列中删除相关任务。实现代码如下：

```Java
public void remove(final Collection<TaskContext.MetaInfo> metaInfoList) {
   for (TaskContext.MetaInfo each : metaInfoList) {
       regCenter.remove(FailoverNode.getFailoverTaskNodePath(each.toString()));
   }
}
```

# 666. 彩蛋

原本以为会是一篇水更，后面研究 TASK_LOST，发现收获大大的，干货妥妥的。

![](http://www.iocoder.cn/images/Elastic-Job/2018_01_10/03.png)

道友，赶紧上车，分享一波朋友圈！



