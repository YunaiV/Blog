title: SkyWalking 源码分析 —— Agent 插件（一）之 Tomcat
date: 2020-11-20
tags:
categories: SkyWalking
permalink: SkyWalking/agent-plugin-tomcat

-------

摘要: 原创出处 http://www.iocoder.cn/SkyWalking/agent-plugin-tomcat/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 SkyWalking 3.2.6 正式版**

- [1. 概述](http://www.iocoder.cn/SkyWalking/agent-plugin-tomcat/)
- [2. TomcatInstrumentation](http://www.iocoder.cn/SkyWalking/agent-plugin-tomcat/)
  - [2.1 TomcatInvokeInterceptor](http://www.iocoder.cn/SkyWalking/agent-plugin-tomcat/)
  - [2.2 TomcatExceptionInterceptor](http://www.iocoder.cn/SkyWalking/agent-plugin-tomcat/)
- [666. 彩蛋](http://www.iocoder.cn/SkyWalking/agent-plugin-tomcat/)

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

本文主要分享 **SkyWalking Agent Tomcat 插件**。涉及到的代码不多，如下图：

![](http://www.iocoder.cn/images/SkyWalking/2020_11_20/01.png)

# 2. TomcatInstrumentation

在 [`skywalking-plugin.def`](https://github.com/YunaiV/skywalking/blob/0128349b40592b8ae329443c52f43577cc9fa16b/apm-sniffer/apm-sdk-plugin/tomcat-7.x-8.x-plugin/src/main/resources/skywalking-plugin.def) 里，定义了插件，如下图：

![](http://www.iocoder.cn/images/SkyWalking/2020_11_20/03.png)

-------

[`org.skywalking.apm.plugin.tomcat78x.define.TomcatInstrumentation`](https://github.com/YunaiV/skywalking/blob/0128349b40592b8ae329443c52f43577cc9fa16b/apm-sniffer/apm-sdk-plugin/tomcat-7.x-8.x-plugin/src/main/java/org/skywalking/apm/plugin/tomcat78x/define/TomcatInstrumentation.java) ，实现 ClassInstanceMethodsEnhancePluginDefine 抽象类，定义了方法切面，代码如下：

![](http://www.iocoder.cn/images/SkyWalking/2020_11_20/02.png)

## 2.1 TomcatInvokeInterceptor

[`org.skywalking.apm.plugin.tomcat78x.TomcatInvokeInterceptor`](https://github.com/YunaiV/skywalking/blob/0128349b40592b8ae329443c52f43577cc9fa16b/apm-sniffer/apm-sdk-plugin/tomcat-7.x-8.x-plugin/src/main/java/org/skywalking/apm/plugin/tomcat78x/TomcatInvokeInterceptor.java) ，实现 InstanceMethodsAroundInterceptor 接口，TomcatInstrumentation 的拦截器。代码如下：

* [`#beforeMethod(...)`](https://github.com/YunaiV/skywalking/blob/0128349b40592b8ae329443c52f43577cc9fa16b/apm-sniffer/apm-sdk-plugin/tomcat-7.x-8.x-plugin/src/main/java/org/skywalking/apm/plugin/tomcat78x/TomcatInvokeInterceptor.java#L5) 方法，创建 EntrySpan 对象。代码如下：
    * 第 59 至 65 行：解析 ContextCarrier 对象，用于跨进程的链路追踪。在 [《SkyWalking 源码分析 —— Agent 收集 Trace 数据》「 3.2.3 ContextCarrier 」](http://www.iocoder.cn/SkyWalking/agent-collect-trace/?self) 有详细解析。
    * 第 68 行：调用 `ContextManager#createEntrySpan(operationName, contextCarrier)` 方法，创建 EntrySpan 对象。
    * 第 71 至 72 行：设置 EntrySpan 对象的 `url` / `http.method` 标签键值对。
    * 第 75 行：设置 EntrySpan 对象的组件类型。
    * 第 78 行：设置 EntrySpan 对象的分层。
* [`#afterMethod(...)`](https://github.com/YunaiV/skywalking/blob/0128349b40592b8ae329443c52f43577cc9fa16b/apm-sniffer/apm-sdk-plugin/tomcat-7.x-8.x-plugin/src/main/java/org/skywalking/apm/plugin/tomcat78x/TomcatInvokeInterceptor.java#L81) 方法，完成 EntrySpan 对象。
    * 第 89 至 92 行：当返回状态码大于等于 400 时，标记 EntrySpan 发生异常，并设置 `status_code` 标签键值对。
    * 第 95 行：调用 `ContextManager#stopSpan()` 方法，完成 EntrySpan 对象。
* [`#handleMethodException(...)`](https://github.com/YunaiV/skywalking/blob/0128349b40592b8ae329443c52f43577cc9fa16b/apm-sniffer/apm-sdk-plugin/tomcat-7.x-8.x-plugin/src/main/java/org/skywalking/apm/plugin/tomcat78x/TomcatInvokeInterceptor.java#L99) 方法，处理异常。**注意**，该方法实际并且调用，在 Tomcat 的[`StandardWrapperValve#invoke(request, response)`](https://github.com/Oreste-Luci/apache-tomcat-8.0.26-src/blob/master/java/org/apache/catalina/core/StandardWrapperValve.java#L94) 方法里，发生异常时，会提交异常给 [`StandardWrapperValve#exception(request, response, exception)`](https://github.com/Oreste-Luci/apache-tomcat-8.0.26-src/blob/master/java/org/apache/catalina/core/StandardWrapperValve.java#L507) 处理，所以会被 [「 2.2 TomcatExceptionInterceptor 」](#) 拦截。

## 2.2 TomcatExceptionInterceptor

[`org.skywalking.apm.plugin.tomcat78x.TomcatExceptionInterceptor`](https://github.com/YunaiV/skywalking/blob/0128349b40592b8ae329443c52f43577cc9fa16b/apm-sniffer/apm-sdk-plugin/tomcat-7.x-8.x-plugin/src/main/java/org/skywalking/apm/plugin/tomcat78x/TomcatExceptionInterceptor.java) ，实现 InstanceMethodsAroundInterceptor 接口，TomcatInstrumentation 的拦截器。代码如下：

* [`#beforeMethod(...)`](https://github.com/YunaiV/skywalking/blob/0128349b40592b8ae329443c52f43577cc9fa16b/apm-sniffer/apm-sdk-plugin/tomcat-7.x-8.x-plugin/src/main/java/org/skywalking/apm/plugin/tomcat78x/TomcatExceptionInterceptor.java#L31) 方法，处理异常。代码如下：
    * 第 35 行：调用 `AbstractSpan#errorOccurred()` 方法，标记 EntrySpan 对象发生异常。
    * 第 35 行：调用 `AbstractSpan#log(Throwable)` 方法，记录异常日志到 EntrySpan 对象。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

一大波插件来袭！

胖友，分享一波朋友圈。


