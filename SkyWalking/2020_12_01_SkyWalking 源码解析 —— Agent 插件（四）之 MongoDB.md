title: SkyWalking 源码分析 —— Agent 插件（四）之 MongoDB
date: 2020-12-01
tags:
categories: SkyWalking
permalink: SkyWalking/agent-plugin-mongodb

-------

摘要: 原创出处 http://www.iocoder.cn/SkyWalking/agent-plugin-mongodb/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 SkyWalking 3.2.6 正式版**

- [1. 概述](http://www.iocoder.cn/SkyWalking/agent-plugin-mongodb/)
- [2. MongoDBInstrumentation](http://www.iocoder.cn/SkyWalking/agent-plugin-mongodb/)
  - [2.1 MongoDBMethodInterceptor](http://www.iocoder.cn/SkyWalking/agent-plugin-mongodb/)
- [666. 彩蛋](http://www.iocoder.cn/SkyWalking/agent-plugin-mongodb/)

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

本文主要分享 **SkyWalking Agent MongoDB 插件**。涉及到的代码不多，如下图：

![](http://www.iocoder.cn/images/SkyWalking/2020_12_01/02.png)

考虑到大多数团队目前使用的 MongoDB 3.X 版本，本文解析 `mongodb-3.x-plugin` 插件。

# 2. MongoDBInstrumentation

在 [`skywalking-plugin.def`](https://github.com/apache/incubator-skywalking/blob/ea37839950c58b77114fbfebffc13f051e1caeb7/apm-sniffer/apm-sdk-plugin/mongodb-3.x-plugin/src/main/resources/skywalking-plugin.def) 里，定义了插件，如下图：

![](http://www.iocoder.cn/images/SkyWalking/2020_12_01/03.png)

-------

[`org.apache.skywalking.apm.plugin.mongodb.v3.define.MongoDBInstrumentation`](https://github.com/apache/incubator-skywalking/blob/ea37839950c58b77114fbfebffc13f051e1caeb7/apm-sniffer/apm-sdk-plugin/mongodb-3.x-plugin/src/main/java/org/apache/skywalking/apm/plugin/mongodb/v3/define/MongoDBInstrumentation.java) ，实现 ClassInstanceMethodsEnhancePluginDefine 抽象类，定义了方法切面，代码如下：

![](http://www.iocoder.cn/images/SkyWalking/2020_12_01/01.png)

## 2.1 MongoDBMethodInterceptor

[`org.apache.skywalking.apm.plugin.mongodb.v3.MongoDBMethodInterceptor`](https://github.com/YunaiV/skywalking/blob/0128349b40592b8ae329443c52f43577cc9fa16b/apm-sniffer/apm-sdk-plugin/dubbo-plugin/src/main/java/org/skywalking/apm/plugin/dubbo/DubboInterceptor.java) ，实现 InstanceMethodsAroundInterceptor 和 InstanceConstructorInterceptor 接口，MongoDBInstrumentation 的拦截器。代码如下：

* `DB_TYPE` 静态属性，数据库类型。
* `MONGO_DB_OP_PREFIX` 静态属性，操作前缀。
* `FILTER_LENGTH_LIMIT` 静态属性，操作语句长度限制，**256** 。
    * `EMPTY` 静态属性，未知操作的参数。
    * 当开启 `Config.Plugin.MongoDB.TRACE_PARAM = true` 时，记录 MongoDB 操作语句。可以通过 `agent.config` 配置，如下图所示：![](http://www.iocoder.cn/images/SkyWalking/2020_12_01/04.png)
* [`#onConstruct(objInst, allArguments)`](https://github.com/YunaiV/skywalking/blob/2f629ab1c9b96a77ecf6cca1e1dc20def0d20f1f/apm-sniffer/apm-sdk-plugin/mongodb-3.x-plugin/src/main/java/org/skywalking/apm/plugin/mongodb/v3/MongoDBMethodInterceptor.java#L206) 方法，拼接拼接集群地址，设置集群地址到 Mongo 对象的私有变量( SkyWalking 自动生成 )。
* [`#beforeMethod(...)`](https://github.com/YunaiV/skywalking/blob/2f629ab1c9b96a77ecf6cca1e1dc20def0d20f1f/apm-sniffer/apm-sdk-plugin/mongodb-3.x-plugin/src/main/java/org/skywalking/apm/plugin/mongodb/v3/MongoDBMethodInterceptor.java#L163) 方法，创建 ExitSpan 对象。代码如下：
    * 第 171 行：调用 `ContextManager#createExitSpan(...)` 创建 ExitSpan 对象。其中，操作名使用 `MONGO_DB_OP_PREFIX` + `Method#getName()` 。
    * 第 174 行：设置 EntrySpan 对象的组件类型。
    * 第 177 行：设置 EntrySpan 对象的 `db.type` 标签键值对。
    * 第 180 行：设置 EntrySpan 对象的分层。
    * 第 183 至 185 行：当 `Config.Plugin.MongoDB.TRACE_PARAM = true` 开启时，设置**操作语句**到 EntrySpan 对象的 `db.statement` 标签键值对。其中，操作语句通过 [`#getTraceParam(Object)`](https://github.com/YunaiV/skywalking/blob/2f629ab1c9b96a77ecf6cca1e1dc20def0d20f1f/apm-sniffer/apm-sdk-plugin/mongodb-3.x-plugin/src/main/java/org/skywalking/apm/plugin/mongodb/v3/MongoDBMethodInterceptor.java#L79) 方法生成。
        * [`#getFilter(List<? extends WriteRequest>)`](https://github.com/YunaiV/skywalking/blob/2f629ab1c9b96a77ecf6cca1e1dc20def0d20f1f/apm-sniffer/apm-sdk-plugin/mongodb-3.x-plugin/src/main/java/org/skywalking/apm/plugin/mongodb/v3/MongoDBMethodInterceptor.java#L136)
        * [`#limitFilter(String)`](https://github.com/YunaiV/skywalking/blob/2f629ab1c9b96a77ecf6cca1e1dc20def0d20f1f/apm-sniffer/apm-sdk-plugin/mongodb-3.x-plugin/src/main/java/org/skywalking/apm/plugin/mongodb/v3/MongoDBMethodInterceptor.java#L154)
* [`#afterMethod(...)`](https://github.com/YunaiV/skywalking/blob/2f629ab1c9b96a77ecf6cca1e1dc20def0d20f1f/apm-sniffer/apm-sdk-plugin/mongodb-3.x-plugin/src/main/java/org/skywalking/apm/plugin/mongodb/v3/MongoDBMethodInterceptor.java#L188) 方法，调用 `ContextManager#stopSpan()` 方法，完成 ExitSpan 对象。
* [`#handleMethodException(...)`](https://github.com/YunaiV/skywalking/blob/2f629ab1c9b96a77ecf6cca1e1dc20def0d20f1f/apm-sniffer/apm-sdk-plugin/mongodb-3.x-plugin/src/main/java/org/skywalking/apm/plugin/mongodb/v3/MongoDBMethodInterceptor.java#L195) 方法，处理异常。代码如下：
    * 第 199 行：标记 ExitSpan 发生异常。
    * 第 202 行：设置 EntrySpan 的日志。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

算是本系列的小完结？偷懒下，后面在补充一些其他的。

呼呼，撒花！

胖友，分享一波朋友圈可好~


