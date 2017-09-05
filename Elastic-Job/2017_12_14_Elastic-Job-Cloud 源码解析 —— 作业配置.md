title: Elastic-Job-Cloud 源码分析 —— 作业配置
date: 2017-12-14
tags:
categories: Elastic-Job-Cloud
permalink: Elastic-Job/cloud-job-config

-------

**本文基于 Elastic-Job V2.1.5 版本分享**

- [1. 概述](#)
- [2. 云作业App](#)
  - [2.1 云作业App配置类](#)
  - [2.2 操作云作业App配置](#)
- [3. 云作业](#3)
  - [3.1 云作业配置](#)
    - [3.1.1 操作云作业配置](#)
  - [3.2 本地云作业配置](#)
  - [3.3 云作业配置总结](#)
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

本文主要分享 **Elastic-Job-Cloud 作业配置**。

如果你阅读过以下文章，有助于对本文的理解：

* [《官方文档 —— RESTFUL API》](http://elasticjob.io/docs/elastic-job-cloud/02-guide/cloud-restful-api/)
* [《Elastic-Job-Lite 源码分析 —— 作业配置》](http://www.yunai.me/Elastic-Job/job-config/?self)
* [《由浅入深 | 如何优雅地写一个Mesos Framework》](https://segmentfault.com/a/1190000007723430)

😈 另外，笔者假设你已经对 **[《Elastic-Job-Lite 源码分析系列》](http://www.yunai.me/categories/Elastic-Job/?self)** 有一定的了解。

本文涉及到主体类的类图如下( [打开大图](http://www.yunai.me/images/Elastic-Job/2017_12_14/01.png) )：

![](http://www.yunai.me/images/Elastic-Job/2017_12_14/01.png)

* **黄色**的类在 `elastic-job-common-core` 项目里，为 Elastic-Job-Lite、Elastic-Job-Cloud **公用**作业配置类。
* **紫色**的类在 `elastic-job-cloud` 项目里，为 Elastic-Job-Cloud 作业配置类。
* **红色**的类在 `elastic-job-lite` 项目里，为 Elastic-Job-Lite 作业配置类。

> 你行好事会因为得到赞赏而愉悦  
> 同理，开源项目贡献者会因为 Star 而更加有动力  
> 为 Elastic-Job 点赞！[传送门](https://github.com/dangdangdotcom/elastic-job/stargazers)

# 2. 云作业App

首先，理解下 **云作业App** 的定义：

> FROM http://dangdangdotcom.github.io/elastic-job/elastic-job-cloud/02-guide/cloud-concepts/      
>     
> 作业APP指作业打包部署后的应用，描述了作业启动需要用到的CPU、内存、启动脚本及应用下载路径等基本信息，每个APP可以包含一个或多个作业。

简单来说，一个云作业App可以理解成由多个作业打在一起的 `jar`。

## 2.1 云作业App配置类

CloudAppConfiguration，云作业App配置。实现代码如下：

```Java
public final class CloudAppConfiguration {

    /**
     * 应用名
     */
    private final String appName;
    /**
     * 应用包地址
     */
    private final String appURL;
    /**
     * 应用启动脚本
     */
    private final String bootstrapScript;
    /**
     * cpu 数量
     */
    private double cpuCount = 1;
    /**
     * 内存 大小
     */
    private double memoryMB = 128;
    /**
     * 每次执行作业时是否从缓存中读取应用。禁用则每次执行任务均从应用仓库下载应用至本地
     */
    private boolean appCacheEnable = true;
    /**
     * 常驻作业事件采样率统计条数，默认不采样全部记录。
     * 为避免数据量过大，可对频繁调度的常驻作业配置采样率，即作业每执行N次，才会记录作业执行及追踪相关数据
     */
    private int eventTraceSamplingCount;
}
```

* 在 Elastic-Job-Lite 里，打包作业，部署到服务器里启动。而在 Elastic-Job-Cloud 里，打包作业上传至可下载的地址。作业被调度时，Mesos 会从 `appURL` 下载应用包，使用 `bootstrapScript` 启动应用进行执行。实际情况会更加复杂一丢丢，在[《Elastic-Job-Cloud 源码解析 —— 作业调度（一）》](http://www.yunai.me/Elastic-Job/cloud-job-scheduler-and-executor/?self)详细解析。
* `cpuCount`, `memoryMB` 配置**云作业App自身占用的资源情况**。其包含的每个作业占用的资源情况，使用作业对应的云作业配置( CloudJobConfiguration ) ，下文也会看到。
* `appCacheEnable`：每次执行作业时是否从缓存中读取应用。禁用则每次执行任务均从应用仓库下载应用至本地。
* `eventTraceSamplingCount`：常驻作业事件采样率统计条数，默认采样全部记录。为避免数据量过大，可对频繁调度的常驻作业配置采样率，即作业每执行N次，才会记录作业执行及追踪相关数据。

## 2.2 操作云作业App配置

云作业App配置有多种操作：

1. 添加 / 更新 / 删除
2. 开启 / 禁用

有两种方式进行操作，以**添加**举例子：

* 调用 HTTP 接口：

    ```shell
    curl -l -H "Content-type: application/json" -X POST -d '{"appName":"foo_app","appURL":"http://app_host:8080/yourJobs.gz","cpuCount":0.1,"memoryMB":64.0,"bootstrapScript":"bin/start.sh","appCacheEnable":true,"eventTraceSamplingCount":0}' http://elastic_job_cloud_host:8899/api/app
    ```

* 运维平台

    ![](http://www.yunai.me/images/Elastic-Job/2017_12_14/02.png)

运维平台是对调用 HTTP 接口的UI封装，实现代码如下：

```Java
// CloudAppRestfulApi
@Path("/app")
public final class CloudAppRestfulApi {
    /**
     * 注册应用配置.
     * 
     * @param appConfig 应用配置
     */
    @POST
    @Consumes(MediaType.APPLICATION_JSON)
    public void register(final CloudAppConfiguration appConfig) {
        Optional<CloudAppConfiguration> appConfigFromZk = appConfigService.load(appConfig.getAppName());
        if (appConfigFromZk.isPresent()) {
            throw new AppConfigurationException("app '%s' already existed.", appConfig.getAppName());
        }
        appConfigService.add(appConfig);
    }
}

// CloudAppConfigurationService.java
/**
* 添加云作业APP配置.
*
* @param appConfig 云作业App配置对象
*/
public void add(final CloudAppConfiguration appConfig) {
   regCenter.persist(CloudAppConfigurationNode.getRootNodePath(appConfig.getAppName()), CloudAppConfigurationGsonFactory.toJson(appConfig));
}

// CloudAppConfigurationNode.JAVA
public final class CloudAppConfigurationNode {
    
    public static final String ROOT =  "/config/app";
    
    private static final String APP_CONFIG =  ROOT + "/%s"; // %s = ${APP_NAME}
}
```

* CloudAppRestfulApi，云作业应用的REST API，实现了云作业App配置的多种操作的 HTTP 接口。
* CloudAppConfigurationService，云作业App配置服务，实现了云作业应用的存储功能。
* 调用 `AppConfigService#add(...)` 方法，存储 CloudAppConfiguration 到注册中心( Zookeeper )的**持久**数据节点 `${NAMESPACE}/config/app/${APP_NAME}`，JSON 格式化对象。使用 zkClient 查看如下：

```SHELL
[zk: localhost:2181(CONNECTED) 1] get /elastic-job-cloud/config/app/exampleApp
{"appName":"exampleApp","appURL":"http://785j8w.com1.z0.glb.clouddn.com/elastic-job-example-cloud-2.1.5.tar.gz","bootstrapScript":"bin/start.sh","cpuCount":1.0,"memoryMB":128.0,"appCacheEnable":true,"eventTraceSamplingCount":0}
```

# 3. 云作业

一个云作业应用可以包含一个或多个云作业。云作业有**两种**作业配置：云作业配置、本地云作业配置。下面来分别分享它们。

## 3.1 云作业配置

CloudJobConfiguration，云作业配置。实现代码如下：

```Java
public final class CloudJobConfiguration implements JobRootConfiguration {

    /**
     * 作业应用名称 {@link com.dangdang.ddframe.job.cloud.scheduler.config.app.CloudAppConfiguration}
     */
    private final String appName;
    /**
     * 作业类型配置
     */
    private final JobTypeConfiguration typeConfig;
    /**
     * 单片作业所需要的CPU数量，最小值为0.001
     */
    private final double cpuCount;
    /**
     * 单片作业所需要的内存MB，最小值为1
     */
    private final double memoryMB;
    /**
     * 作业执行类型
     */
    private final CloudJobExecutionType jobExecutionType;
    /**
     * Spring容器中配置的bean名称
     */
    private String beanName;
    /**
     * Spring方式配置Spring配置文件相对路径以及名称，如：META-INF\applicationContext.xml
     */
    private String applicationContext; 
}
```

* JobTypeConfiguration，作业类型配置，在 `elastic-job-common-core` 项目里，为 Elastic-Job-Lite、Elastic-Job-Cloud **公用**作业配置类。在[《Elastic-Job-Lite 源码分析 —— 作业配置》的「2.2.1 作业类型配置」](http://www.yunai.me/Elastic-Job/job-config/?self)有详细解析。
* `cpuCount`, `memoryMB` 配置**单片作业占用的资源情况**。这里一定要注意是单片作业，例如一个作业有三个分片( `shardingTotalCount = 3` )，则占用资源为 3 * `cpuCount` + 3 * `memoryMB`。
* 作业执行类型( CloudJobExecutionType )有两种：常驻作业( DAEMON )，瞬时作业( TRANSIENT )。在[《Elastic-Job-Cloud 源码解析 —— 作业调度（一）》](http://www.yunai.me/Elastic-Job/cloud-job-scheduler-and-executor/?self)详细解析。Elastic-Job-Cloud 独有，非常有趣。👍👍👍
* `beanName`, `applicationContext` 实现 Spring 启动方式作业。在[《Elastic-Job-Cloud 源码解析 —— 作业调度（一）》](http://www.yunai.me/Elastic-Job/cloud-job-scheduler-and-executor/?self)有详细解析。

### 3.1.1 操作云作业配置

云作业配置有多种操作：

1. 添加 / 更新 / 删除
2. 开启 / 禁用

有两种方式进行操作，以**添加**举例子：

* 调用 HTTP 接口：

    ```shell
    // Java启动方式作业注册
    curl -l -H "Content-type: application/json" -X POST -d '{"jobName":"foo_job","appName":"foo_app","jobClass":"yourJobClass","jobType":"SIMPLE","jobExecutionType":"TRANSIENT","cron":"0/5 * * * * ?","shardingTotalCount":5,"cpuCount":0.1,"memoryMB":64.0,"appURL":"http://app_host:8080/foo-job.tar.gz","failover":true,"misfire":true,"bootstrapScript":"bin/start.sh"}' http://elastic_job_cloud_host:8899/api/job/register
    
    // Spring启动方式作业注册
    curl -l -H "Content-type: application/json" -X POST -d '{"jobName":"foo_job","jobClass":"yourJobClass","beanName":"yourBeanName","applicationContext":"applicationContext.xml","jobType":"SIMPLE","jobExecutionType":"TRANSIENT","cron":"0/5 * * * * ?","shardingTotalCount":5,"cpuCount":0.1,"memoryMB":64.0,"appURL":"http://file_host:8080/foo-job.tar.gz","failover":false,"misfire":true,"bootstrapScript":"bin/start.sh"}' http://elastic_job_cloud_masterhost:8899/api/job/register
    ```

* 运维平台

    ![](http://www.yunai.me/images/Elastic-Job/2017_12_14/03.png)

运维平台是对调用 HTTP 接口的UI封装，实现代码如下：

```Java
// CloudJobRestfulApi.java
public final class CloudJobRestfulApi {
    /**
     * 注册作业.
     * 
     * @param jobConfig 作业配置
     */
    @POST
    @Path("/register")
    @Consumes(MediaType.APPLICATION_JSON)
    public void register(final CloudJobConfiguration jobConfig) {
        producerManager.register(jobConfig);
    }
}

// ProducerManager.java
public final class ProducerManager {
    /**
     * 注册作业.
     * 
     * @param jobConfig 作业配置
     */
    public void register(final CloudJobConfiguration jobConfig) {
        if (disableJobService.isDisabled(jobConfig.getJobName())) {
            throw new JobConfigurationException("Job '%s' has been disable.", jobConfig.getJobName());
        }
        Optional<CloudAppConfiguration> appConfigFromZk = appConfigService.load(jobConfig.getAppName());
        if (!appConfigFromZk.isPresent()) {
            throw new AppConfigurationException("Register app '%s' firstly.", jobConfig.getAppName());
        }
        Optional<CloudJobConfiguration> jobConfigFromZk = configService.load(jobConfig.getJobName());
        if (jobConfigFromZk.isPresent()) {
            throw new JobConfigurationException("Job '%s' already existed.", jobConfig.getJobName());
        }
        // 添加云作业配置
        configService.add(jobConfig);
        // 调度作业
        schedule(jobConfig);
    }
}

// CloudJobConfigurationService.java
/**
* 添加云作业配置.
* 
* @param jobConfig 云作业配置对象
*/
public void add(final CloudJobConfiguration jobConfig) {
   regCenter.persist(CloudJobConfigurationNode.getRootNodePath(jobConfig.getJobName()), CloudJobConfigurationGsonFactory.toJson(jobConfig));
}

// CloudJobConfigurationNode.java
public final class CloudJobConfigurationNode {
    
    public static final String ROOT =  "/config/job";
    
    private static final String JOB_CONFIG =  ROOT + "/%s";  // %s = ${JOB_NAME}
}
```

* CloudJobRestfulApi，作业云Job的REST API，实现了作业云Job配置的多种操作、查询运行中 / 待运行 / 失效转移作业列表等 HTTP 接口。
* ProducerManager，发布任务作业调度管理器。这是一个很重要的类，在[《Elastic-Job-Cloud 源码解析 —— 作业调度（一）》](http://www.yunai.me/Elastic-Job/cloud-job-scheduler-and-executor/?self)详细解析。
* CloudJobConfigurationService，作业配置服务，实现了作业配置的存储功能。
* 调用 `CloudJobConfigurationService#add(...)` 方法，存储 CloudJobConfiguration 到注册中心( Zookeeper )的**持久**数据节点 `${NAMESPACE}/config/job/${JOB_NAME}`，JSON 格式化对象。使用 zkClient 查看如下：

    ```SHELL
    [zk: localhost:2181(CONNECTED) 3] get /elastic-job-cloud/config/job/test_job_simple
{"jobName":"test_job_simple","jobClass":"com.dangdang.ddframe.job.example.job.simple.JavaSimpleJob","jobType":"SIMPLE","cron":"0/10 * * * * ?","shardingTotalCount":1,"shardingItemParameters":"","jobParameter":"","failover":false,"misfire":false,"description":"","jobProperties":{"job_exception_handler":"com.dangdang.ddframe.job.executor.handler.impl.DefaultJobExceptionHandler","executor_service_handler":"com.dangdang.ddframe.job.executor.handler.impl.DefaultExecutorServiceHandler"},"appName":"exampleApp","cpuCount":0.1,"memoryMB":64.0,"jobExecutionType":"TRANSIENT"}
    ```
* 调用 `#schedule(...)` 方法，调度作业。这是个很有趣的方法，在[《Elastic-Job-Cloud 源码解析 —— 作业调度（一）》](http://www.yunai.me/Elastic-Job/cloud-job-scheduler-and-executor/?self)详细解析。


## 3.2 本地云作业配置

LocalCloudJobConfiguration，本地云作业配置。实现代码如下：

```
public final class LocalCloudJobConfiguration implements JobRootConfiguration {
    
    private final JobTypeConfiguration typeConfig;

    /**
     * 分片作业序号
     */
    private final int shardingItem;
    
    private String beanName;
    
    private String applicationContext;
}
```

* `shardingItem`，分片作业序号，用于本地调试指定分片作业项。

到底有什么用呢？

> 在开发Elastic-Job-Cloud作业时，开发人员可以脱离Mesos环境，在本地运行和调试作业。可以利用本地运行模式充分的调试业务功能以及单元测试，完成之后再部署至Mesos集群。  
> 本地运行作业无需安装Mesos环境。

在[《官方文档 —— 本地运行模式》](http://elasticjob.io/docs/elastic-job-cloud/02-guide/local-executor/)有详细解析。

## 3.3 云作业配置总结

* CloudJobConfiguration：生产运行使用。
* LocalCloudJobConfiguration：本地开发调试。

# 666. 彩蛋

芋道君：本文主要为[《Elastic-Job-Cloud 源码解析 —— 作业调度（一）》](http://www.yunai.me/Elastic-Job/cloud-job-scheduler-and-executor/?self)做铺垫，这会是一篇长文。读懂 Elastic-Job-Cloud 作业调度后，整个人脑洞又开的不行不行的！  
旁白君：支持+1024。

![](http://www.yunai.me/images/Elastic-Job/2017_12_14/04.png)

另外，推荐资料如下，对理解 Elastic-Job-Cloud 很有帮助。

* [《基于Mesos的当当作业云Elastic Job Cloud》](http://www.infoq.com/cn/news/2016/09/Mesos-Elastic-Job-Cloud)
* [《如何从0到1搭建弹性作业云Elastic-Job-Cloud》](http://www.infoq.com/cn/presentations/how-to-build-elastic-job-cloud)


道友，赶紧上车，分享一波朋友圈！

