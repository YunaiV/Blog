title: Spring MVC 实现原理与源码解析系统 —— 精品合集
date: 2017-12-05
tags:
categories: 
permalink: Spring-MVC/good-collection

-------

摘要: 原创出处 http://www.iocoder.cn/Spring-MVC/good-collection/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 【芋艿】精尽 Spring MVC 原理与源码专栏](http://www.iocoder.cn/Spring-MVC/good-collection/)
- [2. 【zhisheng】Spring MVC 源码解析](http://www.iocoder.cn/Spring-MVC/good-collection/)
- [3. 【相见欢】Spring MVC 源码解析](http://www.iocoder.cn/Spring-MVC/good-collection/)
- [4. 【carl.zhao】Spring MVC 源码解析](http://www.iocoder.cn/Spring-MVC/good-collection/)
- [666. 欢迎投稿](http://www.iocoder.cn/Spring-MVC/good-collection/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2018_05_18.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

# 1. 【芋艿】精尽 Spring MVC 原理与源码专栏

* 作者：芋艿
* **只更新在笔者的知识星球，欢迎加入一起讨论 Spring MVC 源码与实现**。  ![](http://www.iocoder.cn/images/common/zsxq/01.png)
    * 目前已经有 **1000+** 位球友加入...
    * 进度：已经完成 **24+** 篇，预计总共 25+ 篇，完成度 **96%** 。
* 对应 Spring MVC 版本号：**5.1.1.BUILD-SNAPSHOT**

Spring MVC 整体代码量有 5w+ 行，通过本专栏，可以快速的研读核心部分的代码，节省你的时间。

## 1.1 核心

DispatcherServlet 的流程处理如下图：![DispatcherServlet 的流程处理](http://www.iocoder.cn/images/Spring/2017-12-05/01.png)

* 但是随着前后端分离，后端大多提供 Restful API ，里面的 ViewResolver 和 View 的流程，已经不再被需要。为什么呢？源码中，我们将得到解答。

UML 序列图如下：

![UML 序列图](http://www.iocoder.cn/images/Spring/2017-12-05/02.png)

* 虽然整体流程不复杂，但是通过阅读 Spring MVC 的源码，我们会发现，Spring MVC 优雅的提供了各种拓展点，例如 HttpMessageConvert、ExceptionHandler 等等。

## 1.2 文章目录

* 惊喜
    * [《精尽 Spring MVC 面试题》](https://t.zsxq.com/NFuv3jq)
    * [《精尽 Spring MVC 学习指南》](https://t.zsxq.com/NFuv3jq)
* 调试
    * [《精尽 Spring MVC 源码分析 —— 调试环境搭建》](https://t.zsxq.com/NFuv3jq) 
* 容器的初始化
    * [《精尽 Spring MVC 源码解析 —— 容器的初始化（一）之 Root WebApplicationContext 容器》](https://t.zsxq.com/NFuv3jq) 
    * [《精尽 Spring MVC 源码解析 —— 容器的初始化（二）之 Servlet WebApplicationContext 容器》](https://t.zsxq.com/NFuv3jq) 
    * [《精尽 Spring MVC 源码解析 —— 容器的初始化（三）之 Servlet 3.0 集成》](https://t.zsxq.com/NFuv3jq) 
    * [《精尽 Spring MVC 源码解析 —— 容器的初始化（四）之 Spring Boot 集成》](https://t.zsxq.com/NFuv3jq) 
* 整体一览
    * [《精尽 Spring MVC 源码分析 —— 组件一览》](https://t.zsxq.com/NFuv3jq)
    * [《精尽 Spring MVC 源码解析 —— 请求处理一览》](https://t.zsxq.com/NFuv3jq) 
* 组件详解 
    * [《精尽 Spring MVC 源码解析 —— HandlerMapping 组件（一）之 AbstractHandlerMapping》](https://t.zsxq.com/NFuv3jq)
    * [《精尽 Spring MVC 源码解析 —— HandlerMapping 组件（二）之 HandlerInterceptor》](https://t.zsxq.com/NFuv3jq)
    * [《精尽 Spring MVC 源码解析 —— HandlerMapping 组件（三）之 AbstractHandlerMethodMapping》](https://t.zsxq.com/NFuv3jq)
    * [《精尽 Spring MVC 源码解析 —— HandlerMapping 组件（四）之 AbstractUrlHandlerMapping》](https://t.zsxq.com/NFuv3jq)
    * [《精尽 Spring MVC 源码解析 —— HandlerAdapter 组件（一）之 HandlerAdapter》](https://t.zsxq.com/NFuv3jq)
    * [《精尽 Spring MVC 源码解析 —— HandlerAdapter 组件（二）之 ServletInvocableHandlerMethod》](https://t.zsxq.com/NFuv3jq)
    * [《精尽 Spring MVC 源码解析 —— HandlerAdapter 组件（三）之 HandlerMethodArgumentResolver》](https://t.zsxq.com/NFuv3jq)
    * [《精尽 Spring MVC 源码解析 —— HandlerAdapter 组件（四）之 HandlerMethodReturnValueHandler》](https://t.zsxq.com/NFuv3jq)
    * [《精尽 Spring MVC 源码解析 —— HandlerAdapter 组件（五）之 HttpMessageConverter》](https://t.zsxq.com/NFuv3jq)
    * [《精尽 Spring MVC 源码解析 —— HandlerExceptionResolver》](https://t.zsxq.com/NFuv3jq)
    * [《精尽 Spring MVC 源码解析 —— RequestToViewNameTranslator》](https://t.zsxq.com/NFuv3jq)
    * [《精尽 Spring MVC 源码解析 —— LocaleResolver》](https://t.zsxq.com/NFuv3jq)
    * [《精尽 Spring MVC 源码解析 —— ThemeResolver》](https://t.zsxq.com/NFuv3jq)
    * [《精尽 Spring MVC 源码解析 —— ViewResolver》](https://t.zsxq.com/NFuv3jq)
    * [《精尽 Spring MVC 源码解析 —— MultipartResolver》](https://t.zsxq.com/NFuv3jq)
    * [《精尽 Spring MVC 源码解析 —— FlashMapManager》](https://t.zsxq.com/NFuv3jq)

# 2. 【zhisheng】Spring MVC 源码解析

* 作者 ：zhisheng
* 博客 ：http://www.54tianzhisheng.cn
* 目录 ：
    * [《看透 Spring MVC 源代码分析与实践 —— 网站基础知识》](http://www.iocoder.cn/Spring-MVC/zhisheng/Spring-MVC01/)
    * [《看透 Spring MVC 源代码分析与实践 —— 俯视 Spring MVC》](http://www.iocoder.cn/Spring-MVC/zhisheng/Spring-MVC02/)
    * [《看透 Spring MVC 源代码分析与实践 —— Spring MVC 组件分析》](http://www.iocoder.cn/Spring-MVC/zhisheng/Spring-MVC03/)

# 3. 【相见欢】Spring MVC 源码解析

* 作者 ：相见欢
* 博客 ：https://my.oschina.net/lichhao/blog?catalog=285356
* 目录 ：
    * [《Spring MVC 源码剖析（一） —— 从抽象和接口说起》](http://www.iocoder.cn/Spring-MVC/lichhao/Start-with-abstractions-and-interfaces)
    * [《Spring MVC 源码剖析（二） —— DispatcherServlet 的前世今生》](http://www.iocoder.cn/Spring-MVC/lichhao/DispatcherServlet-1)
    * [《Spring MVC 源码剖析（三） —— DispatcherServlet 的初始化流程》](http://www.iocoder.cn/Spring-MVC/lichhao/DispatcherServlet-2)
    * [《Spring MVC 源码剖析（四） —— DispatcherServlet 请求转发的实现》](http://www.iocoder.cn/Spring-MVC/lichhao/DispatcherServlet-3)
    * [《Spring MVC 源码剖析（五） —— 消息转换器 HttpMessageConverter》](http://www.iocoder.cn/Spring-MVC/lichhao/HttpMessageConverter)

# 4. 【carl.zhao】Spring MVC 源码解析

* 作者 ：carl.zhao
* 博客 ：http://blog.csdn.net/u012410733/
* 目录 ：
    * [《【carl.zhao】Spring MVC @RequestMapping》](http://www.iocoder.cn/Spring-MVC/carlzhao/RequestMapping)
    * [《【carl.zhao】Spring MVC DispatcherServlet》](http://www.iocoder.cn/Spring-MVC/carlzhao/DispatcherServlet)
    * [《【carl.zhao】Spring MVC DataBinder》](http://www.iocoder.cn/Spring-MVC/carlzhao/DataBinder)
    * [《【carl.zhao】Spring MVC 与 Servlet》](http://www.iocoder.cn/Spring-MVC/carlzhao/Servlet)
    * [《【carl.zhao】Spring MVC 之调用复用》](http://www.iocoder.cn/Spring-MVC/carlzhao/Invoke)

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

