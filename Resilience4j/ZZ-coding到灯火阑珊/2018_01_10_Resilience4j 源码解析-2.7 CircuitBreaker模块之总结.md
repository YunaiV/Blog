title: Resilience4j 源码解析-2.7 CircuitBreaker模块之总结
date: 2018-01-10
tag: 
categories: Resilience4j
permalink: Resilience4j/coding/CircuitBreaker-summary
author: coding到灯火阑珊
from_url: https://mp.weixin.qq.com/s/pwBS5o9ouVcFTYjAt3diew
wechat_url: 

-------

摘要: 原创出处 https://mp.weixin.qq.com/s/pwBS5o9ouVcFTYjAt3diew 「coding到灯火阑珊」欢迎转载，保留摘要，谢谢！

- [1. 熔断器模块总结](http://www.iocoder.cn/Resilience4j/coding/CircuitBreaker-summary/)
  - [1.1 熔断器初始化时的状态：](http://www.iocoder.cn/Resilience4j/coding/CircuitBreaker-summary/)
  - [1.2 熔断器状态转换](http://www.iocoder.cn/Resilience4j/coding/CircuitBreaker-summary/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# 1. 熔断器模块总结

通过前面几篇文章分析了熔断器模块的6个主要组成部分，这篇文章我们来分析这6个部分是如何协同工作的。

## 1.1 熔断器初始化时的状态：

一般创建熔断器的代码：

![img](http://static.iocoder.cn/1d0ce616772e0fb6d22b129e09053720)

这2行代码产生了一系列的调用关系，如图：

![img](http://static.iocoder.cn/9ee34f926b232e9da9a3982e21469bf3)

这时熔断器的状态是关闭状态(ClosedState)，度量指标实例及事件处理器实例都已经准备好。我们这时可以用CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();这行代码实时的获取度量指标值。

## 1.2 熔断器状态转换

在这里，我们只分析熔断器从关闭状态转换到打开状态，其他状态之间的转换过程是一样的。

ClosedState ==> OpenState 状态的转换过程，如图：

![img](http://static.iocoder.cn/66bf5626ec3aed8ba25da93f98329c10)

熔断器通过decorateSupplier方法装饰请求调用的方法，如：

CircuitBreaker.decorateSupplier(circuitBreaker,helloWorldService::returnHelloWorld);

会发生如图所示的调用过程：

1）熔断器判断当前状态是否允许请求调用，关闭状态始终返回true。

2）执行请求调用方法

3）请求调用的结果成功或失败，熔断器最终会调用度量指标CircuitBreakerMetrics的onSuccess或onError方法返回请求调用失败率到ClosedState。ClosedState会在它的onSuccess或onError方法中判断请求失败率是否达到了设置的阈值，如果达到了阈值则调用状态机CircuitBreakerStateMachine的transitionToOpenState方法生成OpenState对象，同时把关闭状态的度量指标对象传给打开状态。然后熔断器把当前持有的状态更改为打开状态，完成了状态转换。

Resilience4j的熔断器模块的源码分析告一段落，接下来的系列文章会分析Resilience4j的限流器模块。
