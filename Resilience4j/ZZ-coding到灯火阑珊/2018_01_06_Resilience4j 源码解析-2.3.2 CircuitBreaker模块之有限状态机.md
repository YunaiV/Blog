title: Resilience4j 源码解析-2.3.2 CircuitBreaker模块之有限状态机
date: 2018-01-06
tag: 
categories: Resilience4j
permalink: Resilience4j/coding/CircuitBreaker-232
author: coding到灯火阑珊
from_url: https://mp.weixin.qq.com/s/n80_vS3eoW3H4-YEA5rFig
wechat_url: 

-------

摘要: 原创出处 https://mp.weixin.qq.com/s/n80_vS3eoW3H4-YEA5rFig 「coding到灯火阑珊」欢迎转载，保留摘要，谢谢！

- [1. 有限状态机的实现](http://www.iocoder.cn/Resilience4j/coding/CircuitBreaker-232/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# 1. 有限状态机的实现

状态建模完成，还需要建模一个状态机来驱动状态的变化，把状态机封装成类CircuitBreakerStateMachine。

如图：

![img](http://static.iocoder.cn/c953873aea6c0073006cad7813fdd17a)

CircuitBreakerStateMachine类实现了熔断器CircuitBreaker接口，除了实现状态转换机制，还实现了熔断机制和事件发布机制。

所以CircuitBreakerStateMachine类是整个熔断器模块的核心类，在这篇文章中我们只研究状态转换机制，在后续的文章中我们逐步讲解熔断机制和事件发布机制。

下面我们看一下CircuitBreaker接口和CircuitBreakerStateMachine实现类中与状态相关的属性和方法：

1，在CircuitBreaker接口中声明了状态的枚举类，主要是为了方便进行判等操作，同时也设置了每种状态是否允许发布事件

![img](http://static.iocoder.cn/ede70ea60941f1dbcfbfad37bf25624a)

2，CircuitBreakerStateMachine实现类中，用AtomicReference保证对CircuitBreakerState引用的原子性，在构造方法中初始化状态机为关闭状态。


```Java
// 用AtomicReference保证对CircuitBreakerState引用的原子性
private final AtomicReference<CircuitBreakerState> stateReference;
```

构造方法：

![img](http://static.iocoder.cn/b7234df57fab142fb20c1e96ae5f5ad3)

状态转换系列方法：

![img](http://static.iocoder.cn/9c9af37b9c3aad84e5e316614ba05b2b)


```Java
/**
 * 转换到不可用状态
*/
@Override
public void transitionToDisabledState() {
   stateTransition(DISABLED, currentState -> new DisabledState(this));
}

/**
 * 转换到强制打开状态
*/
@Override
public void transitionToForcedOpenState() {
   stateTransition(FORCED_OPEN, currentState -> new ForcedOpenState(this));
}

/**
 * 转换到关闭状态
*/
@Override
public void transitionToClosedState() {
   stateTransition(CLOSED, currentState -> new ClosedState(this,currentState.getMetrics()));
}

/**
 * 转换到打开状态
*/
@Override
public void transitionToOpenState() {
   stateTransition(OPEN, currentState -> new OpenState(this,currentState.getMetrics()));
}

/**
 * 转换到半开状态
*/
@Override
public void transitionToHalfOpenState() {
   stateTransition(HALF_OPEN, currentState -> new HalfOpenState(this));
}
```

这些状态转换方法是由各状态类实例来调用的，具体可以参考上一篇文章《Resilience4j源码解析-2.3.1 CircuitBreaker模块之有限状态机》

下一篇文章《Resilience4j源码解析-2.4 CircuitBreaker模块之度量指标》讲解熔断器触发状态转换的度量指标。
