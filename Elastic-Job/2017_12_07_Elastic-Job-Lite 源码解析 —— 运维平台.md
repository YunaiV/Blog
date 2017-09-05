title: Elastic-Job-Lite 源码分析 —— 运维平台
date: 2017-12-07
tags:
categories: Elastic-Job-Lite
permalink: Elastic-Job/job-console

-------

**本文基于 Elastic-Job V2.1.5 版本分享**

- [1. 概述](#)
- [2. Maven模块 elastic-job-common-restful](#)
- [3. Maven模块 elastic-job-console](#)
- [4. Maven模块 elastic-job-lite-lifecycle](#)
- [5. 其它](#)
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

本文主要分享 **Elastic-Job-Lite 运维平台**。内容对应[《官方文档 —— 运维平台》](http://dangdangdotcom.github.io/elastic-job/elastic-job-lite/02-guide/web-console/)。

运维平台实现上比较易懂，就不特别**啰嗦**的解析，简略说下每个类的用途和 UI 上的关联。

> 你行好事会因为得到赞赏而愉悦  
> 同理，开源项目贡献者会因为 Star 而更加有动力  
> 为 Elastic-Job 点赞！[传送门](https://github.com/dangdangdotcom/elastic-job/stargazers)


# 2. Maven模块 elastic-job-common-restful

1. RestfulServer 内嵌服务器，基于 Jetty 实现
2. GSONProvider 后端接口 JSON 格式化 
3. RestfulExceptionMapper 异常映射
4. WwwAuthFilter 授权认证 Filter

# 3. Maven模块 elastic-job-console

## 3.1 domain 包

* RegistryCenterConfigurations / RegistryCenterConfiguration ：注册中心配置实体相关。
* EventTraceDataSourceConfigurations / EventTraceDataSourceConfiguration / EventTraceDataSource / EventTraceDataSourceFactory ：事件事件追踪数据源配置实体相关。
    
## 3.2 filter 包

* GlobalConfigurationFilter ：全局配置过滤器，加载当前会话( HttpSession ) 选择的 RegistryCenterConfiguration / EventTraceDataSource 。

## 3.3 repository 包

使用 **XML文件** 存储 EventTraceDataSource / RegistryCenterConfiguration 配置实体。

## 3.4 restful 包

* `config` / RegistryCenterRestfulApi ：注册中心配置( RegistryCenterConfiguration )的RESTful API
   ![](http://www.yunai.me/images/Elastic-Job/2017_12_07/01.png)

* `config` / EventTraceDataSourceRestfulApi ：事件追踪数据源配置( EventTraceDataSource )的RESTful API
   ![](http://www.yunai.me/images/Elastic-Job/2017_12_07/02.png)
    
* `config` / LiteJobConfigRestfulApi ：作业配置( LiteJobConfiguration )的RESTful API
   ![](http://www.yunai.me/images/Elastic-Job/2017_12_07/03.png)
    
* EventTraceHistoryRestfulApi ：事件追踪历史记录( `JOB_EXECUTION_LOG` / `JOB_STATUS_TRACE_LOG` )的RESTful API
   ![](http://www.yunai.me/images/Elastic-Job/2017_12_07/06.png)
   ![](http://www.yunai.me/images/Elastic-Job/2017_12_07/07.png)
    
* ServerOperationRestfulApi ：服务器维度操作的RESTful API。
   ![](http://www.yunai.me/images/Elastic-Job/2017_12_07/05.png)
    
* JobOperationRestfulApi ：作业维度操作的RESTful API。
   ![](http://www.yunai.me/images/Elastic-Job/2017_12_07/04.png)

## 3.5 service 包

* RegistryCenterConfigurationService ：注册中心( RegistryCenterConfiguration )配置服务。
* EventTraceDataSourceConfigurationService ：事件追踪数据源配置( EventTraceDataSource )服务。
* JobAPIService ：和作业相关的 API 集合服务。这些 API 在 Maven模块 `elastic-job-lite-lifecycle` 实现。
   * JobSettingsAPI：作业配置的API。
   * JobOperateAPI ：操作作业的API。
   * ShardingOperateAPI ：操作分片的API。
   * JobStatisticsAPI ：JobStatisticsAPI。
   * ServerStatisticsAPI ：作业服务器状态展示的API。
   * ShardingStatisticsAPI ：作业分片状态展示的API。

# 4. Maven模块 elastic-job-lite-lifecycle

在 JobAPIService 已经基本提到，这里不重复叙述。

# 5. 其它

1. 前后端分离，后端使用 JSON 为前端提供数据接口。
2. 后端 API 使用 Restful 设计规范。 
3. 国际化使用 `jquery.i18n.js` 实现。
4. 界面使用 Bootstrap AdminLTE 模板实现。

# 666. 彩蛋

旁白君：这写的... 略飘逸（随意）  
芋道君：哈哈哈，我要开始 Elastic-Job-Cloud 啦啦啦啦。

![](http://www.yunai.me/images/Elastic-Job/2017_12_07/08.png)

道友，赶紧上车，分享一波朋友圈！

