title: Resilience4j 源码解析-2.6 CircuitBreaker模块之熔断
date: 2018-01-09
tag: 
categories: Resilience4j
permalink: Resilience4j/coding/CircuitBreaker-break
author: coding到灯火阑珊
from_url: https://mp.weixin.qq.com/s/PYkO5AAqfkiARqvpPu5o5g
wechat_url: 

-------

摘要: 原创出处 https://mp.weixin.qq.com/s/PYkO5AAqfkiARqvpPu5o5g 「coding到灯火阑珊」欢迎转载，保留摘要，谢谢！

- [6. CircuitBreaker（熔断器）](http://www.iocoder.cn/Resilience4j/coding/CircuitBreaker-break/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# 6. CircuitBreaker（熔断器）

通过前面几篇的讲解，现在终于可以来看一看熔断功能是如何实现的了。与熔断器相关的有CircuitBreaker接口和CircuitBreakerStateMachine实现类。

![img](http://static.iocoder.cn/97b4a3e69b310d68b2b18e85448bea30)

在CircuitBreaker中：

1）声明了与状态相关的枚举类State和与状态转换相关的枚举类StateTransition：

![img](http://static.iocoder.cn/e1c0421d9afce700e1da1d5f3c402549)

![img](http://static.iocoder.cn/877a15c5d18cfffc87cd71ea1e08161d)

有限状态机的5种状态及状态的转换在文章《Resilience4j源码解析-2.3.1 CircuitBreaker模块之有限状态机》中有讲解。

2）声明了度量指标接口Metrics：

![img](http://static.iocoder.cn/ae6f97c5cc717d110bf5a629419248ac)

度量指标及存储方式在文章《Resilience4j源码解析-2.4 CircuitBreaker模块之度量指标》中有讲解。

3）声明了事件处理器接口EventPublisher接口：

![img](http://static.iocoder.cn/c6f44364c3b676929471fd9a9f5f6372)

事件发布及处理机制在文章《Resilience4j源码解析-2.5 CircuitBreaker模块之事件发布》中有讲解。

4）熔断功能

在CircuitBreaker接口中以线程安全的单例模式生成了CircuitBreakerStateMachine的实例，有三种实现方式：

![img](http://static.iocoder.cn/fb6edee47b50524f8029934b2189224c)

熔断方法

三个default方法：

![img](http://static.iocoder.cn/81c4d1d7c65b6841a96fbf2996e8119c)

其他是static方法：

![img](http://static.iocoder.cn/ba844ffa4ea847e94f2c4f188a02145a)

这些装饰方法逻辑基本一样，我们来看看其中常用的decorateSupplier(...)方法。

![img](http://static.iocoder.cn/53b2797fc66a92fc52bc5777b2c40345)

CircuitBreakerUtils.isCallPermitted(circuitBreaker)最终调用了CircuitBreakerStateMachine实现类中的isCallPermitted()方法，circuitBreaker.onSuccess(...)和circuitBreaker.onError(...)也分别调用了CircuitBreakerStateMachine实现类中的onSuccess(...)和onError(...)方法，

注释如下：

![img](http://static.iocoder.cn/d69354b4baa86463a04ef5917c2802d4)

至此，Resilience4j熔断器模块CircuitBreaker的六个组成部分全部分析完了，

下一篇文章《Resilience4j源码解析-2.7 CircuitBreaker模块之总结》主要分析六个组成部分是如何有效的协同工作来达到熔断的目的。
