title: 面试问烂的 Spring AOP 原理
date: 2018-11-09
tags:
categories: 精进
permalink: Fight/Interview-poorly-asked-Spring-AOP-principles
author: 莫那一鲁道
from_url: https://www.jianshu.com/p/e18fd44964eb
wechat_url:

-------

摘要: 原创出处 https://www.jianshu.com/p/e18fd44964eb 「莫那一鲁道」欢迎转载，保留摘要，谢谢！

- [Spring MVC 过程](http://www.iocoder.cn/Fight/Interview-poorly-asked-SpringMVC-process/)
  - [1. 设置属性](http://www.iocoder.cn/Fight/Interview-poorly-asked-SpringMVC-process/)
  - [2. 根据 Request 请求的 URL 得到对应的 handler 执行链，其实就是拦截器和 Controller 代理对象。](http://www.iocoder.cn/Fight/Interview-poorly-asked-SpringMVC-process/)
  - [3. 得到 handler 的适配器](http://www.iocoder.cn/Fight/Interview-poorly-asked-SpringMVC-process/)
  - [4. 循环执行 handler 的 pre 拦截器](http://www.iocoder.cn/Fight/Interview-poorly-asked-SpringMVC-process/)
  - [5. 执行真正的 handler，并返回  ModelAndView(Handler 是个代理对象，可能会执行 AOP )](http://www.iocoder.cn/Fight/Interview-poorly-asked-SpringMVC-process/)
  - [6. 循环执行 handler 的 post 拦截器](http://www.iocoder.cn/Fight/Interview-poorly-asked-SpringMVC-process/)
  - [7. 根据 ModelAndView 信息得到 View 实例](http://www.iocoder.cn/Fight/Interview-poorly-asked-SpringMVC-process/)
  - [8. 渲染 View 返回](http://www.iocoder.cn/Fight/Interview-poorly-asked-SpringMVC-process/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

Spring AOP ，应该是国内面试必问题，网上有很多答案，其实背背就可以。但今天笔者带大家一起深入浅出源码，看看他的原理。以期让印象更加深刻，面试的时候游刃有余。

# Spring AOP 原理

简单说说 AOP 的设计：

1. 每个 Bean 都会被 JDK 或者 Cglib 代理。取决于是否有接口。
2. 每个 Bean 会有多个“方法拦截器”。注意：拦截器分为两层，外层由 Spring 内核控制流程，内层拦截器是用户设置，也就是 AOP。
3. 当代理方法被调用时，先经过外层拦截器，外层拦截器根据方法的各种信息判断该方法应该执行哪些“内层拦截器”。内层拦截器的设计就是职责连的设计。

是不是贼简单。事实上，楼主之前已经写过一个简单的例子，地址：[使用 Cglib 实现多重代理](http://thinkinjava.cn/2018/10/%E4%BD%BF%E7%94%A8-Cglib-%E5%AE%9E%E7%8E%B0%E5%A4%9A%E9%87%8D%E4%BB%A3%E7%90%86/)

看完之后更简单。

可以将 AOP 分成 2 个部分来扯，哦，不，来分析。。。
 第一：代理的创建；
 第二：代理的调用。

注意：我们尽量少贴代码，尽量用文字叙述，因为面试的时候，也是文字叙述，不可能让你把代码翻出来的。。。所以，这里需要保持一定的简洁，想知道细节，看 interface 21 源码，想知道的更细，看 Spring Framework 最新的 master 分支代码。

代码位置：`com.interface21.aop` 包下。

开始分析（扯）：

1. 代理的创建（按步骤）：

- 首先，需要创建代理工厂，代理工厂需要 3 个重要的信息：拦截器数组，目标对象接口数组，目标对象。
- 创建代理工厂时，默认会在拦截器数组尾部再增加一个默认拦截器 —— 用于最终的调用目标方法。
- 当调用 getProxy 方法的时候，会根据接口数量大余 0 条件返回一个代理对象（JDK or  Cglib）。
- 注意：创建代理对象时，同时会创建一个外层拦截器，这个拦截器就是 Spring 内核的拦截器。用于控制整个 AOP 的流程。

2. 代理的调用

- 当对代理对象进行调用时，就会触发外层拦截器。
- 外层拦截器根据代理配置信息，创建内层拦截器链。创建的过程中，会根据表达式判断当前拦截是否匹配这个拦截器。而这个拦截器链设计模式就是职责链模式。
- 当整个链条执行到最后时，就会触发创建代理时那个尾部的默认拦截器，从而调用目标方法。最后返回。

题外话：Spring 的事务也就是个拦截器。

来张不是很标准的 UML 图：



![img](https:////upload-images.jianshu.io/upload_images/4236553-52cc7fe83aba8a2d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/jpeg)



关于调用过程，来张流程图：



![img](https:////upload-images.jianshu.io/upload_images/4236553-115860c16ceed67b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/jpeg)



大概就是这样子，具体更多的细节，请看源码，如果还不是很明白的话，请咨询本人，本人不确定这个图是否画的很浅显易懂 —— 最起码萌新看得懂才能称之为浅显易懂。