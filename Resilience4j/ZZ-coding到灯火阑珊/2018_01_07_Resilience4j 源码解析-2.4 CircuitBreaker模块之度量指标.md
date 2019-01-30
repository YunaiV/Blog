title: Resilience4j 源码解析-2.4 CircuitBreaker模块之度量指标
date: 2018-01-07
tag: 
categories: Resilience4j
permalink: Resilience4j/coding/CircuitBreaker-Metrics
author: coding到灯火阑珊
from_url: https://mp.weixin.qq.com/s/INEBGTSM5zbDIe_zWBoU7w
wechat_url: 

-------

摘要: 原创出处 https://mp.weixin.qq.com/s/INEBGTSM5zbDIe_zWBoU7w 「coding到灯火阑珊」欢迎转载，保留摘要，谢谢！

- [1. CircuitBreakerMetrics（熔断器度量指标）](http://www.iocoder.cn/Resilience4j/coding/CircuitBreaker-Metrics/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------


# 1. CircuitBreakerMetrics（熔断器度量指标）

上一篇文章我们分析了熔断器状态机的状态转换机制，这一篇文章我们分析一下熔断器度量指标，比如：包括触发状态转换的请求调用失败率是如何计算的。


相关联的类，如图：

![img](http://static.iocoder.cn/4cbb049c3da5e550919fda91359b46f8)

1)，CircuitBreaker.Metrics是一个度量指标接口，定义了一系列获取度量指标的方法，包括获取失败率的百分比、获取请求调用总数量，获取请求调用失败的数量、获取不允许请求调用通过的数量、获取设置的最大调用总数和获取请求调用成功的数量。

源码如图：

![img](http://static.iocoder.cn/56c9c98f1bdee03fce3df1c57f489b7a)

2，RingBitSet

在RingBitSet内部有一个按位存储的Ring Bit Bufffer(环形缓存区)数据结构BitSetMod，原理与java.util.BitSet一样，可以先去研究一下java.util.BitSet的机制。Ring Bit Buffer用于在熔断器关闭状态下，一次成功的调用在其bit位上存储0值，一次失败的调用在其bit位上存储1值。

![img](http://static.iocoder.cn/7c6b4cb58b2e52faff5c62aa2e989e98)

计算请求失败率之前，Ring Bit Buffer的每一bit位都必须被存储过。例如，如果Ring Bit Buffer的大小设置为10，如果前9次的请求调用都失败也不会计算请求调用失败率。

3，CircuitBreakerMetrics类是CircuitBreaker.Metrics接口的实现类，内部持有RingBitSet类用来存储每次请求调用的结果，以便用于计算请求调用失败率，进而触发熔断器状态机的状态转换。

主要源码如下：

```Java
// 环形缓冲区大小
private final int ringBufferSize;

// Ring Bit Buffer
private final RingBitSet ringBitSet;

// 不允许请求调用通过的数量，采用并发更加高效的LongAdder类型
private final LongAdder numberOfNotPermittedCalls;
```

![img](http://static.iocoder.cn/0ebabd5f562543d5c837c296ff90adfd)

onError()和onSuccess()方法是由状态机的状态类中的onError(Throwable throwable)和onSuccess()方法调用的，用来检查是否达到了设定的请求调用失败率，进而调用状态机的状态转换方法。

以初始关闭状态转换到打开状态为例，如图：

![img](http://static.iocoder.cn/eaea301e1aa3d5979a5d9c783b982d00)

状态机如何驱动状态转换的请看上一篇文章《Resilience4j源码解析-2.3.2 CircuitBreaker模块之有限状态机》。

下一篇文章《Resilience4j源码解析-2.5 CircuitBreaker模块之事件发布》讲解Resilience4j中的事件机制。
