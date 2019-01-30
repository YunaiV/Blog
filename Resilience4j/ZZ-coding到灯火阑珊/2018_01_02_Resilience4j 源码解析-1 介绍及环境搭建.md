title: Resilience4j 源码解析-1 介绍及环境搭建
date: 2018-01-02
tag: 
categories: Resilience4j
permalink: Resilience4j/coding/build-debugging-environment
author: coding到灯火阑珊
from_url: https://mp.weixin.qq.com/s/Frvb20GcMtnjPr0q8wrRRg
wechat_url: 

-------

摘要: 原创出处 https://mp.weixin.qq.com/s/Frvb20GcMtnjPr0q8wrRRg 「coding到灯火阑珊」欢迎转载，保留摘要，谢谢！

- [1. 简介](http://www.iocoder.cn/Resilience4j/coding/build-/)
- [2. 环境搭建](http://www.iocoder.cn/Resilience4j/coding/build-/)
  - [2.1 安装 Gradle 及设置](http://www.iocoder.cn/Resilience4j/coding/build-/)
  - [2.2 代码调试](http://www.iocoder.cn/Resilience4j/coding/build-/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# 1. 简介

Resilience4j是受Hystrix启发而做的新一代轻量级熔断器，基于Java8的函数式编程开发。resilience4j只依赖一个Vavr包(函数式库)，不需要再引入其他包，所以相对于Hystrix是轻量级的。同时resilience4j的模块化程度更好，很容易的扩展附加模块。

目前Resilience4j包括熔断(CircuitBreaker)、限流(RateLimiter)、隔离(BulkHead)、重试(Retry)、缓存(Cache)、执行时间限制(TimeLimiter)等模块及一些附加模块(Add-on modules)。

# 2. 环境搭建

## 2.1 安装 Gradle 及设置

由于Resilience4j采用gradle进行编译及打包，所以我们需要下载及安装最新版本的gradle，同时在IDE中进行环境设置。Resilience4j在github上的地址：https://github.com/resilience4j/resilience4j.git clone下来及导入到IntelliJ IDEA中，设置resilience4j的gradle环境，如图：

![img](http://static.iocoder.cn/6f29faabd877dc6fbf27a089d8585562)

Service directory path是放置类包的仓库。勾选Use auto-import选项，grade会对resilience4j源码进行依赖导入及编译。


## 2.2 代码调试

编译好的Resilience4j代码，如下图：

![img](http://static.iocoder.cn/88c68074f34d3c05be0d2df85de162d8)

如果我们要研究circuitbreaker的源码，就可以进入到resilience4j-circuitbreaker模块中，找到单元测试类CircuitBreakerTest类，设置断点，进行代码跟踪调试了。

测试类路径在：

![img](http://static.iocoder.cn/da4b1add389a26789a394a8b412f75b1)

其中的一个测试方法如下图：

![img](http://static.iocoder.cn/7ea8e5ae75883e5fbb84701cfa630d31)