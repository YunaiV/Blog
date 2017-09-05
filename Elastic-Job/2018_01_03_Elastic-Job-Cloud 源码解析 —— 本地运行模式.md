title: Elastic-Job-Cloud 源码分析 —— 本地运行模式
date: 2018-01-03
tags:
categories: Elastic-Job-Cloud
permalink: Elastic-Job/cloud-local-executor

-------

**本文基于 Elastic-Job V2.1.5 版本分享**

- [1. 概述](#)
- [2. 配置](#)
- [3. 运行](#)
- [666. 彩蛋](#)

-------

# 1. 概述

本文主要分享 **Elastic-Job-Cloud 本地运行模式**，对应[《官方文档 —— 本地运行模式》](http://elasticjob.io/docs/elastic-job-cloud/02-guide/local-executor/)。

**有什么用呢**？引用官方解答：

> 在开发 Elastic-Job-Cloud 作业时，开发人员可以脱离 Mesos 环境，在本地运行和调试作业。可以利用本地运行模式充分的调试业务功能以及单元测试，完成之后再部署至 Mesos 集群。  
> 本地运行作业无需安装 Mesos 环境。

😈 是不是很赞 + 1024？！

本文涉及到主体类的类图如下( [打开大图](http://www.yunai.me/images/Elastic-Job/2018_01_03/01.png) )：

![](http://www.yunai.me/images/Elastic-Job/2018_01_03/01.png)

> 你行好事会因为得到赞赏而愉悦  
> 同理，开源项目贡献者会因为 Star 而更加有动力  
> 为 Elastic-Job 点赞！[传送门](https://github.com/dangdangdotcom/elastic-job/stargazers)

# 2. 配置

LocalCloudJobConfiguration，本地云作业配置，在[《Elastic-Job-Cloud 源码分析 —— 作业配置》「3.2 本地云作业配置」](http://www.yunai.me/Elastic-Job/cloud-local-executor/?self)有详细解析。

创建本地云作业配置示例代码如下（来自官方）：

```Java
LocalCloudJobConfiguration config = new LocalCloudJobConfiguration(
    new SimpleJobConfiguration(
    // 配置作业类型和作业基本信息
    JobCoreConfiguration.newBuilder("FooJob", "*/2 * * * * ?", 3) 
        .shardingItemParameters("0=Beijing,1=Shanghai,2=Guangzhou")
        .jobParameter("dbName=dangdang").build(), "com.dangdang.foo.FooJob"),
        // 配置当前运行的作业是第几个分片 
        1,  
        // 配置Spring相关参数。如果不配置，代表不使用 Spring 配置。
        "testSimpleJob" , "applicationContext.xml"); 
```

# 3. 运行

LocalTaskExecutor，本地作业执行器。

创建本地作业执行器示例代码如下（来自官方）：

``` Java
new LocalTaskExecutor(localJobConfig).execute();
```

可以看到，调用 `LocalTaskExecutor#execute()` 方法，执行作业逻辑，实现代码如下：

```Java
// LocalTaskExecutor.java
public void execute() {
   AbstractElasticJobExecutor jobExecutor;
   CloudJobFacade jobFacade = new CloudJobFacade(getShardingContexts(), getJobConfigurationContext(), new JobEventBus());
   // 创建执行器
   switch (localCloudJobConfiguration.getTypeConfig().getJobType()) {
       case SIMPLE:
           jobExecutor = new SimpleJobExecutor(getJobInstance(SimpleJob.class), jobFacade);
           break;
       case DATAFLOW:
           jobExecutor = new DataflowJobExecutor(getJobInstance(DataflowJob.class), jobFacade);
           break;
       case SCRIPT:
           jobExecutor = new ScriptJobExecutor(jobFacade);
           break;
       default:
           throw new UnsupportedOperationException(localCloudJobConfiguration.getTypeConfig().getJobType().name());
   }
   // 执行作业
   jobExecutor.execute();
}
```

* 调用 `#getShardingContexts()` 方法，创建分片上下文集合( ShardingContexts )，实现代码如下：

    ```Java
    private ShardingContexts getShardingContexts() {
       JobCoreConfiguration coreConfig = localCloudJobConfiguration.getTypeConfig().getCoreConfig();
       Map<Integer, String> shardingItemMap = new HashMap<>(1, 1);
       shardingItemMap.put(localCloudJobConfiguration.getShardingItem(),
               new ShardingItemParameters(coreConfig.getShardingItemParameters()).getMap().get(localCloudJobConfiguration.getShardingItem()));
       return new ShardingContexts(
               // taskId 👇
               Joiner.on("@-@").join(localCloudJobConfiguration.getJobName(), localCloudJobConfiguration.getShardingItem(), "READY", "foo_slave_id", "foo_uuid"),
               localCloudJobConfiguration.getJobName(), coreConfig.getShardingTotalCount(), coreConfig.getJobParameter(), shardingItemMap);
    }
    ```

* 调用 `#getJobConfigurationContext()` 方法，创建内部的作业配置上下文( JobConfigurationContext )，实现代码如下：

    ```Java
    private JobConfigurationContext getJobConfigurationContext() {
       Map<String, String> jobConfigurationMap = new HashMap<>();
       jobConfigurationMap.put("jobClass", localCloudJobConfiguration.getTypeConfig().getJobClass());
       jobConfigurationMap.put("jobType", localCloudJobConfiguration.getTypeConfig().getJobType().name());
       jobConfigurationMap.put("jobName", localCloudJobConfiguration.getJobName());
       jobConfigurationMap.put("beanName", localCloudJobConfiguration.getBeanName());
       jobConfigurationMap.put("applicationContext", localCloudJobConfiguration.getApplicationContext());
       if (JobType.DATAFLOW == localCloudJobConfiguration.getTypeConfig().getJobType()) { // 数据流作业
           jobConfigurationMap.put("streamingProcess", Boolean.toString(((DataflowJobConfiguration) localCloudJobConfiguration.getTypeConfig()).isStreamingProcess()));
       } else if (JobType.SCRIPT == localCloudJobConfiguration.getTypeConfig().getJobType()) { // 脚本作业
           jobConfigurationMap.put("scriptCommandLine", ((ScriptJobConfiguration) localCloudJobConfiguration.getTypeConfig()).getScriptCommandLine());
       }
       return new JobConfigurationContext(jobConfigurationMap);
    }
    ```

* 调用 `#getJobInstance(...)` 方法， 获得分布式作业( ElasticJob )实现实例，实现代码如下：

    ```Java
    private <T extends ElasticJob> T getJobInstance(final Class<T> clazz) {
       Object result;
       if (Strings.isNullOrEmpty(localCloudJobConfiguration.getApplicationContext())) { // 直接创建 ElasticJob
           String jobClass = localCloudJobConfiguration.getTypeConfig().getJobClass();
           try {
               result = Class.forName(jobClass).newInstance();
           } catch (final ReflectiveOperationException ex) {
               throw new JobSystemException("Elastic-Job: Class '%s' initialize failure, the error message is '%s'.", jobClass, ex.getMessage());
           }
       } else { // Spring 环境获得 ElasticJob
           result = new ClassPathXmlApplicationContext(localCloudJobConfiguration.getApplicationContext()).getBean(localCloudJobConfiguration.getBeanName());
       }
       return clazz.cast(result);
    }
    ```

* 调用 `AbstractElasticJobExecutor#execute()` 方法，执行作业逻辑。 Elastic-Job-Lite 和 Elastic-Job-Cloud 作业执行基本一致，在[《Elastic-Job-Lite 源码分析 —— 作业执行》](http://www.yunai.me/Elastic-Job/job-execute/?self)有详细解析。

# 666. 彩蛋

芋道君：可能有点水更，和大家实际开发太相关，想想还是更新下。  
旁白君：哎哟哟，哎哟喂。

![](http://www.yunai.me/images/Elastic-Job/2018_01_03/02.png)

道友，赶紧上车，分享一波朋友圈！


