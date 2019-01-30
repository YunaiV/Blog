title: Resilience4j 源码解析-2.5 CircuitBreaker模块之事件发布
date: 2018-01-08
tag: 
categories: Resilience4j
permalink: Resilience4j/coding/CircuitBreaker-Event
author: coding到灯火阑珊
from_url: https://mp.weixin.qq.com/s/_0WtEojq8U8Tn_CANM3kMQ
wechat_url: 

-------

摘要: 原创出处 https://mp.weixin.qq.com/s/_0WtEojq8U8Tn_CANM3kMQ 「coding到灯火阑珊」欢迎转载，保留摘要，谢谢！

- [1. CircuitBreakerEvent（熔断器事件）](http://www.iocoder.cn/Resilience4j/coding/CircuitBreaker-Event/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# 1. CircuitBreakerEvent（熔断器事件）



1），事件机制框架

Resilience4j的事件机制是采用观察者模式设计的。Resilience4j事件的框架代码在：

![img](http://static.iocoder.cn/d26689d774b2c61139d230218078009f)

关系如图：

![img](http://static.iocoder.cn/de5063391355079e5ccd6e8da384c9f8)

EventPublisher是事件发布者接口(被观察者)，EventConsumer是事件消费者接口(观察者)。

在EventPublisher中只有一个方法onEvent(EventConsumer<T> onEventConsumer)用于设置处理T事件的消费者。

EventConsumer被注解为@FunctionalInterface，所以它有一个唯一的函数式方法consumeEvent(T event)，用来处理T类型的事件。

EventProcessor是EventPublisher的通用实现类，它主要实现了事件消费者的注册及调用相应的函数式方法进行事件处理。

EventProcessor源码如下：

![img](http://static.iocoder.cn/cdf46c1b6f11ece7d22f0efe5cef4fb8)

![img](http://static.iocoder.cn/42f59a3ea7c18e105fec3d0df2f10362)

2），CircuitBreaker的事件类型

CircuitBreakerEvent有六种具体的事件类型，如图：

![img](http://static.iocoder.cn/a8733988af4c156fdbb28af4431df378)

- CircuitBreakerOnErrorEvent: 请求调用失败时发布的事件
- CircuitBreakerOnSuccessEvent：请求调用成功时发布的事件
- CircuitBreakerOnStateTransitionEvent：熔断器状态转换时发布的事件
- CircuitBreakerOnResetEvent：熔断器重置时发布的事件
- CircuitBreakerOnIgnoredErrorEvent：当忽略的调用异常发生时发布的事件
- CircuitBreakerOnCallNotPermittedEvent：当请求不被允许调用时发布的事件


在CircuitBreakerEvent接口中声明了与具体事件相对应的枚举类Type，用来表示事件类型。

![img](http://static.iocoder.cn/0d436b526f6c17174d9f7d8b598212be)

2），CircuitBreaker的事件处理器

CircuitBreaker的事件处理器CircuitBreakerEventProcessor即是事件的发布者，同时也是事件的处理者。关系如图：

![img](http://static.iocoder.cn/b9fafbb5daa6b5eb2844ae26cd178485)

在CircuitBreaker.EventPublisher接口中声明了六个方法，用于向EventProcessor中注册处理六种事件的EventConsumer。

![img](http://static.iocoder.cn/a6b89ad2fb14384c41c2e0ccdc8edb0d)

CircuitBreakerStateMachine.CircuitBreakerEventProcessor实现类：

![img](http://static.iocoder.cn/86788e1eccccb544ba5162eec03577c8)

下一篇文章《Resilience4j源码解析-2.6 CircuitBreaker模块之熔断》讲解Resilience4j中的事件机制。
