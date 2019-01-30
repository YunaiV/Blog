title: SkyWalking 源码分析 —— Agent 插件（二）之 Dubbo
date: 2020-11-25
tags:
categories: SkyWalking
permalink: SkyWalking/agent-plugin-dubbo

-------

摘要: 原创出处 http://www.iocoder.cn/SkyWalking/agent-plugin-dubbo/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 SkyWalking 3.2.6 正式版**

- [1. 概述](http://www.iocoder.cn/SkyWalking/agent-plugin-dubbo/)
- [2. DubboInstrumentation](http://www.iocoder.cn/SkyWalking/agent-plugin-dubbo/)
  - [2.1 DubboInterceptor](http://www.iocoder.cn/SkyWalking/agent-plugin-dubbo/)
- [666. 彩蛋](http://www.iocoder.cn/SkyWalking/agent-plugin-dubbo/)

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

本文主要分享 **SkyWalking Agent Dubbo 插件**。涉及到的代码不多，如下图：

![](http://www.iocoder.cn/images/SkyWalking/2020_11_25/01.png)

# 2. DubboInstrumentation

在 [`skywalking-plugin.def`](https://github.com/YunaiV/skywalking/blob/0128349b40592b8ae329443c52f43577cc9fa16b/apm-sniffer/apm-sdk-plugin/dubbo-plugin/src/main/resources/skywalking-plugin.def) 里，定义了插件，如下图：

![](http://www.iocoder.cn/images/SkyWalking/2020_11_25/03.png)

-------

[`org.skywalking.apm.plugin.tomcat78x.define.TomcatInstrumentation`](https://github.com/YunaiV/skywalking/blob/0128349b40592b8ae329443c52f43577cc9fa16b/apm-sniffer/apm-sdk-plugin/tomcat-7.x-8.x-plugin/src/main/java/org/skywalking/apm/plugin/tomcat78x/define/TomcatInstrumentation.java) ，实现 ClassInstanceMethodsEnhancePluginDefine 抽象类，定义了方法切面，代码如下：

![](http://www.iocoder.cn/images/SkyWalking/2020_11_25/02.png)

## 2.1 DubboInterceptor

[`org.skywalking.apm.plugin.dubbo.DubboInterceptor`](https://github.com/YunaiV/skywalking/blob/0128349b40592b8ae329443c52f43577cc9fa16b/apm-sniffer/apm-sdk-plugin/dubbo-plugin/src/main/java/org/skywalking/apm/plugin/dubbo/DubboInterceptor.java) ，实现 InstanceMethodsAroundInterceptor 接口，DubboInstrumentation 的拦截器。**同时适用于 Dubbo 服务提供者与消费者**。代码如下：

* [`#beforeMethod(...)`](https://github.com/YunaiV/skywalking/blob/e4bfdfad2540adbc85b4437359b9a5183f05f403/apm-sniffer/apm-sdk-plugin/dubbo-plugin/src/main/java/org/skywalking/apm/plugin/dubbo/DubboInterceptor.java) 方法，若是服务提供者，创建 EntrySpan 对象；若是服务消费者，创建 ExitSpan 对象。代码如下：
    * ----- 服务消费者 -----
    * 第 71 行：创建 ContextCarrier 对象。
    * 第 74 行：调用 `ContextManager#createExitSpan(...)` 创建 ExitSpan 对象，并将**链路追踪上下文注入到 ContextCarrier 对象**，用于跨进程的链路追踪。其中，调用 [`#generateOperationName(URL, Invocation)`](https://github.com/YunaiV/skywalking/blob/e4bfdfad2540adbc85b4437359b9a5183f05f403/apm-sniffer/apm-sdk-plugin/dubbo-plugin/src/main/java/org/skywalking/apm/plugin/dubbo/DubboInterceptor.java#L146) 方法，生成操作名，例如：`"org.skywalking.apm.plugin.test.Test.test(String)"` 。
    * 第 79 至 83 行：设置 ContextCarrier 对象到 RPCContext ，从而将 ContextCarrier 对象**隐式传参**。
    * ----- 服务提供者者 -----
    * 第 87 至 92 行：解析 ContextCarrier 对象，用于跨进程的链路追踪。在 [《SkyWalking 源码分析 —— Agent 收集 Trace 数据》「 3.2.3 ContextCarrier 」](http://www.iocoder.cn/SkyWalking/agent-collect-trace/?self) 有详细解析。
    * 第 95 行：调用 `ContextManager#createLocalSpan(operationName, contextCarrier)` 方法，创建 EntrySpan 对象。
    * ----- ALL -----
    * 第 99 至 72 行：设置 EntrySpan 对象的 `url` 标签键值对。其中，调用 [`#generateRequestURL(URL, Invocation)`](https://github.com/YunaiV/skywalking/blob/e4bfdfad2540adbc85b4437359b9a5183f05f403/apm-sniffer/apm-sdk-plugin/dubbo-plugin/src/main/java/org/skywalking/apm/plugin/dubbo/DubboInterceptor.java#L169-L176) 方法，生成链接。例如，`"dubbo://127.0.0.1:20880/org.skywalking.apm.plugin.test.Test.test(String)"`。
    * 第 102 行：设置 EntrySpan 对象的组件类型。
    * 第 105 行：设置 EntrySpan 对象的分层。
* [`#afterMethod(...)`](https://github.com/YunaiV/skywalking/blob/e4bfdfad2540adbc85b4437359b9a5183f05f403/apm-sniffer/apm-sdk-plugin/dubbo-plugin/src/main/java/org/skywalking/apm/plugin/dubbo/DubboInterceptor.java#L109) 方法，完成 EntrySpan 对象。
    * 第 112 至 115 行：当返回结果包含异常时，调用 [`#dealException(Throwable)`](https://github.com/YunaiV/skywalking/blob/e4bfdfad2540adbc85b4437359b9a5183f05f403/apm-sniffer/apm-sdk-plugin/dubbo-plugin/src/main/java/org/skywalking/apm/plugin/dubbo/DubboInterceptor.java#L132) 方法，处理异常。
    * 调用 `ContextManager#stopSpan()` 方法，完成 EntrySpan 对象。
* [`#handleMethodException(...)`](https://github.com/YunaiV/skywalking/blob/e4bfdfad2540adbc85b4437359b9a5183f05f403/apm-sniffer/apm-sdk-plugin/dubbo-plugin/src/main/java/org/skywalking/apm/plugin/dubbo/DubboInterceptor.java#L123) 方法，调用 [`#dealException(Throwable)`](https://github.com/YunaiV/skywalking/blob/e4bfdfad2540adbc85b4437359b9a5183f05f403/apm-sniffer/apm-sdk-plugin/dubbo-plugin/src/main/java/org/skywalking/apm/plugin/dubbo/DubboInterceptor.java#L132) 方法，处理异常。


# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

一大波插件来袭，继续！

胖友，分享一波朋友圈。


