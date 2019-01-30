title: Resilience4j 实现原理与源码解析系统 —— 精品合集
date: 2018-02-02
tags:
categories:
permalink: Resilience4j/good-collection

-------

摘要: 原创出处 http://www.iocoder.cn/Resilience4j/Resilience4j-collection/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1.【coding到灯火阑珊】Resilience4j 源码解析](http://www.iocoder.cn/Resilience4j/good-collection/)
- [666. 欢迎投稿](http://www.iocoder.cn/Resilience4j/good-collection/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2018_05_18.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

# 1.【coding到灯火阑珊】Resilience4j 源码解析

* 作者 ：coding到灯火阑珊
* 博客 ：https://blog.csdn.net/lonewolf79218/article/list
* 目录 ：
    * [《Resilience4j 源码解析-1 介绍及环境搭建》](http://www.iocoder.cn/Resilience4j/coding/build-debugging-environment)
    * [《Resilience4j 源码解析-2.1 CircuitBreaker模块》](http://www.iocoder.cn/Resilience4j/coding/CircuitBreaker-21)
    * [《Resilience4j 源码解析-2.2 CircuitBreaker模块之配置》](http://www.iocoder.cn/Resilience4j/coding/CircuitBreaker-Config)
    * [《Resilience4j 源码解析-2.3.1 CircuitBreaker模块之有限状态机》](http://www.iocoder.cn/Resilience4j/coding/CircuitBreaker-231)
    * [《Resilience4j 源码解析-2.3.2 CircuitBreaker模块之有限状态机》](http://www.iocoder.cn/Resilience4j/coding/CircuitBreaker-232)
    * [《Resilience4j 源码解析-2.4 CircuitBreaker模块之度量指标》](http://www.iocoder.cn/Resilience4j/coding/CircuitBreaker-Metrics)
    * [《Resilience4j 源码解析-2.5 CircuitBreaker模块之事件发布》](http://www.iocoder.cn/Resilience4j/coding/CircuitBreaker-Event)
    * [《Resilience4j 源码解析-2.6 CircuitBreaker模块之熔断》](http://www.iocoder.cn/Resilience4j/coding/CircuitBreaker-break)
    * [《Resilience4j 源码解析-2.7 CircuitBreaker模块之总结》](http://www.iocoder.cn/Resilience4j/coding/CircuitBreaker-summary)
    * [《Resilience4j 源码解析-3 RateLimiter模块介绍及限流算法》](http://www.iocoder.cn/Resilience4j/coding/RateLimiter)

# 666. 彩蛋

![](http://www.iocoder.cn/images/common/zsxq/01.png)

