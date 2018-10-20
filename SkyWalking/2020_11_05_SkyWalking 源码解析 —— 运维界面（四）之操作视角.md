title: SkyWalking 源码分析 —— 运维界面（四）之操作视角
date: 2020-11-05
tags:
categories: SkyWalking
permalink: SkyWalking/ui-4-operation

-------

摘要: 原创出处 http://www.iocoder.cn/SkyWalking/ui-4-operation/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 SkyWalking 3.2.6 正式版**

- [1. 概述](http://www.iocoder.cn/SkyWalking/ui-4-operation/)
- [2. AllInstanceLastTimeGetHandler](http://www.iocoder.cn/SkyWalking/ui-4-operation/)
- [3. ApplicationsGetHandler](http://www.iocoder.cn/SkyWalking/ui-4-operation/)
- [4. EntryServiceGetHandler](http://www.iocoder.cn/SkyWalking/ui-4-operation/)
- [5. ServiceTreeGetByIdHandler](http://www.iocoder.cn/SkyWalking/ui-4-operation/)
- [6. 彩蛋](http://www.iocoder.cn/SkyWalking/ui-4-operation/)

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

本文主要分享**运维界面的第四部分，操作视角**。

> SkyWalking WEBUI ：https://github.com/apache/incubator-skywalking-ui

在我们打开 SkyWalking WEBUI 的 `Service Tree` ( `service/serviceTree.html` ) 页时，如下图：

![](http://www.iocoder.cn/images/SkyWalking/2020_11_05/01.png)

* 以操作为维度进行展示。
* 黄色部分，时间进度条，调用 [「2. AllInstanceLastTimeGetHandler」](#) 接口，获得应用实例最后心跳时间。大多情况下，我们进入该界面，看的是从最后心跳时间开始的操作情况。
* 红色部分，应用筛选器，调用 [「3. ApplicationsGetHandler」](#) 接口，获得应用列表。
* 紫色部分，入口操作( EntryService )分页列表，调用 [「4. EntryServiceGetHandler」](#) 接口，获得数据。
* 蓝色部分，【点击单个操作】，获得指定操作的关联操作调用统计**树列表**，调用 [「5. ServiceTreeGetByIdHandler」](#) 接口，获得数据。

> 基情提示：运维界面相关 HTTP 接口，逻辑简单易懂，笔者写的会比较简略一些。

# 2. AllInstanceLastTimeGetHandler

同 [《SkyWalking 源码分析 —— 运维界面（一）之应用视角》「2. AllInstanceLastTimeGetHandler」](http://www.iocoder.cn/SkyWalking/ui-1-application/?self) 相同。

# 3. ApplicationsGetHandler

同 [《SkyWalking 源码分析 —— 运维界面（二）之应用实例视角》「3. ApplicationsGetHandler」](http://www.iocoder.cn/SkyWalking/ui-2-instance/?self) 相同。

# 4. EntryServiceGetHandler

[`org.skywalking.apm.collector.ui.jetty.handler.servicetree.EntryServiceGetHandler`](https://github.com/YunaiV/skywalking/blob/3b31539e2e77baf00fafbc60ac9c30802e6c922a/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/servicetree/EntryServiceGetHandler.java) ，实现 JettyHandler 接口，获得入口操作( EntryService )分页列表的逻辑处理器。

* [`#pathSpec()`](https://github.com/YunaiV/skywalking/blob/3b31539e2e77baf00fafbc60ac9c30802e6c922a/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/servicetree/EntryServiceGetHandler.java#L40) ，路径定义，`"/service/entry"` 。
* 响应示例：![](http://www.iocoder.cn/images/SkyWalking/2020_11_05/02.png)
* [`#doGet()`](https://github.com/YunaiV/skywalking/blob/3b31539e2e77baf00fafbc60ac9c30802e6c922a/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/servicetree/EntryServiceGetHandler.java#L50) 方法，代码如下：
    * 第 66 至 71 行：解析 `applicationId` 参数。
    * 第 73 至 85 行：解析 `startTime` 和 `endTime` 参数。
    * 第 88 至 99 行：解析 `from` 和 `size` 分页参数。
    * 第 73 行：调用 [`ServiceTreeService#loadEntryService(...)`](https://github.com/YunaiV/skywalking/blob/3b31539e2e77baf00fafbc60ac9c30802e6c922a/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/service/ServiceTreeService.java#L57) 方法，获得入口操作( EntryService )分页列表。代码如下：
        * 第 44 行：调用 [`ServiceEntryEsUIDAO#load(...)`](https://github.com/YunaiV/skywalking/blob/3b31539e2e77baf00fafbc60ac9c30802e6c922a/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/ServiceEntryEsUIDAO.java#L49) 方法，查询 ServiceEntry 分页 JSON 数组。
        * 第 63 至 69 行：设置应用编码。

# 5. ServiceTreeGetByIdHandler

[`org.skywalking.apm.collector.ui.jetty.handler.servicetree.ServiceTreeGetByIdHandler`](https://github.com/YunaiV/skywalking/blob/7e453f0e8237685b7b46ddd390afce3b76b45123/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/servicetree/ServiceTreeGetByIdHandler.java) ，实现 JettyHandler 接口，获得指定操作的关联操作调用统计树列表的逻辑处理器。

* [`#pathSpec()`](https://github.com/YunaiV/skywalking/blob/7e453f0e8237685b7b46ddd390afce3b76b45123/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/servicetree/ServiceTreeGetByIdHandler.java#L40) ，路径定义，`"/service/tree/entryServiceId"` 。
* 响应示例：![](http://www.iocoder.cn/images/SkyWalking/2020_11_05/03.png)
* [`#doGet()`](https://github.com/YunaiV/skywalking/blob/68b704ef2395067fdb135262089c5c3d316efee7/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/instancehealth/InstanceHealthGetHandler.java#L52) 方法，代码如下：
    * 第 61 至 66 行：解析 `entryServiceId` 参数。
    * 第 60 至 74 行：解析 `startTime` 和 `endTime` 参数。
    * 第 94 行：调用 [`ServiceTreeService#loadServiceTree(entryServiceId, startTime, endTime)`](https://github.com/YunaiV/skywalking/blob/7e453f0e8237685b7b46ddd390afce3b76b45123/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/service/ServiceTreeService.java#L74) 方法，获得指定操作的关联操作调用统计树列表，涉及 **ServiceReference** 数据表。代码比较简单易懂( 笔者太懒了 )，胖友自己阅读理解。![](http://www.iocoder.cn/images/SkyWalking/2020_11_05/04.png)

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

水更第四发！

胖友，分享一波朋友圈可好？


