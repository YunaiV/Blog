title: SkyWalking 源码分析 —— 运维界面（一）之应用视角
date: 2020-10-25
tags:
categories: SkyWalking
permalink: SkyWalking/ui-1-application

-------

摘要: 原创出处 http://www.iocoder.cn/SkyWalking/ui-1-application/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 SkyWalking 3.2.6 正式版**

- [1. 概述](http://www.iocoder.cn/SkyWalking/ui-1-application/)
- [2. AllInstanceLastTimeGetHandler](http://www.iocoder.cn/SkyWalking/ui-1-application/)
- [3. TraceDagGetHandler](http://www.iocoder.cn/SkyWalking/ui-1-application/)
- [666. 彩蛋](http://www.iocoder.cn/SkyWalking/ui-1-application/)

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

本文主要分享**运维界面的第一部分，应用视角**。

> SkyWalking WEBUI ：https://github.com/apache/incubator-skywalking-ui

在我们打开 SkyWalking WEBUI 的首页时，如下图：

![](http://www.iocoder.cn/images/SkyWalking/2020_10_25/01.png)

* 以应用为维度进行展示。
* 紫色部分，时间进度条，调用 [「2. AllInstanceLastTimeGetHandler」](#) 接口，获得应用实例最后心跳时间。大多情况下，我们进入该界面，看的是从最后心跳时间开始的应用调用情况。
* 红色部分，应用调用拓扑图，初始化以 [ 实例最后心跳时间前一小时 , 实例最后心跳时间 ] 调用 [「3. TraceDagGetHandler」](#) 接口，获得数据，而后每 2 秒刷新一次，30 次刷新后，即过了 1 分钟后，数据范围向前走一分钟，为 [ 实例最后心跳时间前一小时 + 一分钟 , 实例最后心跳时间 + 一分钟 ]。
* 通过进度条的【**播放/暂停按钮**】可以切换到暂停模式，切换时间范围，查看时间范围为一小时内的应用调用拓扑图。注意，一定要切换到【暂时模式】才可调整时间范围，在【播放模式】下，每次**自动**请求都会重设时间范围。

> 基情提示：运维界面相关 HTTP 接口，逻辑简单易懂，笔者写的会比较简略一些。

# 2. AllInstanceLastTimeGetHandler

[`org.skywalking.apm.collector.ui.jetty.handler.time.AllInstanceLastTimeGetHandler`](https://github.com/YunaiV/skywalking/blob/9f2dab1c61b49610eca0fc2634ee7af918ba7d1f/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/time/AllInstanceLastTimeGetHandler.java) ，实现 JettyHandler 接口，获得应用实例最后心跳时间处理器。代码如下：

* [`#pathSpec()`](https://github.com/YunaiV/skywalking/blob/9f2dab1c61b49610eca0fc2634ee7af918ba7d1f/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/time/AllInstanceLastTimeGetHandler.java#L43) ，路径定义，`"/time/allInstance"` 。
* 响应示例：![](http://www.iocoder.cn/images/SkyWalking/2020_10_25/02.png)
* [`#doGet()`](https://github.com/YunaiV/skywalking/blob/9f2dab1c61b49610eca0fc2634ee7af918ba7d1f/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/time/AllInstanceLastTimeGetHandler.java#L53) 方法，代码如下：
    * 第 55 至 59 行：调用 [`TimeSynchronousService#allInstanceLastTime()`](https://github.com/YunaiV/skywalking/blob/9f2dab1c61b49610eca0fc2634ee7af918ba7d1f/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/service/TimeSynchronousService.java#L40) 方法，获得应用实例最后心跳时间。
        * [`InstanceEsUIDAO#lastHeartBeatTime()`](https://github.com/YunaiV/skywalking/blob/fe20d4fff8e2ebf4ad44c9e7ac455f69146c0b9c/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/InstanceEsUIDAO.java#L61)
    * 第 61 至 65 行：减 5 秒，因为应用心跳是最频繁的，防止其他信息还没上传。
    * 第 68 至 69 行：返回数据。

# 3. TraceDagGetHandler

[`org.skywalking.apm.collector.ui.jetty.handler.TraceDagGetHandler`](https://github.com/YunaiV/skywalking/blob/9f2dab1c61b49610eca0fc2634ee7af918ba7d1f/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/time/AllInstanceLastTimeGetHandler.java) ，实现 JettyHandler 接口，获得应用拓扑图数据逻辑处理器。

* [`#pathSpec()`](https://github.com/YunaiV/skywalking/blob/f32d6d88343ec18a3b32127bf9c4152e5dc9d4d1/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/TraceDagGetHandler.java#L38) ，路径定义，`"traceDag"` 。
* 响应示例：![](http://www.iocoder.cn/images/SkyWalking/2020_10_25/03.png)
* [`#doGet()`](https://github.com/YunaiV/skywalking/blob/9f2dab1c61b49610eca0fc2634ee7af918ba7d1f/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/time/AllInstanceLastTimeGetHandler.java#L53) 方法，代码如下：
    * 第 73 行：调用 [`TraceDagService#load(startTime, endTime)`](https://github.com/YunaiV/skywalking/blob/f32d6d88343ec18a3b32127bf9c4152e5dc9d4d1/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/TraceDagGetHandler.java#L73) 方法，获得应用拓扑图数据。代码如下：
        * 第 53 行：调用 [`NodeComponentEsUIDAO#load(startTime, endTime)`](https://github.com/YunaiV/skywalking/blob/f32d6d88343ec18a3b32127bf9c4152e5dc9d4d1/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/NodeComponentEsUIDAO.java#L59) 方法，获得 NodeComponent JSON 数组。
        * 第 56 行：调用 [`NodeMappingEsUIDAO#load(startTime, endTime)`](https://github.com/YunaiV/skywalking/blob/f32d6d88343ec18a3b32127bf9c4152e5dc9d4d1/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/NodeMappingEsUIDAO.java#L59) 方法，获得 NodeMapping JSON 数组。
        * 第 59 行：调用 [`NodeReferenceEsUIDAO#load(startTime, endTime)`](https://github.com/YunaiV/skywalking/blob/f32d6d88343ec18a3b32127bf9c4152e5dc9d4d1/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/NodeReferenceEsUIDAO.java#L68) 方法，获得 NodeReference JSON 数组。
        * 第 62 行：调用 [`TraceDagDataBuilder#build(nodeCompArray, nodesMappingArray, resSumArray)`](https://github.com/YunaiV/skywalking/blob/f32d6d88343ec18a3b32127bf9c4152e5dc9d4d1/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/TraceDagGetHandler.java#L48) 方法，使用获得的 NodeComponent 、NodeMapping 、NodeReference 数据，构建应用拓扑图。逻辑较为繁琐，笔者已经添加注释，胖友调试一下，很容易明白滴。![](http://www.iocoder.cn/images/SkyWalking/2020_10_25/04.png)

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

水更第一发！

胖友，分享一波朋友圈可好？


