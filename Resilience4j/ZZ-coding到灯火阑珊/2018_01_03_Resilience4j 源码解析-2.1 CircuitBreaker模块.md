title: Resilience4j 源码解析-2.1 CircuitBreaker模块
date: 2018-01-03
tag: 
categories: Resilience4j
permalink: Resilience4j/coding/CircuitBreaker-21
author: coding到灯火阑珊
from_url: https://mp.weixin.qq.com/s/fbD9HDOmmHd9SHn7VnxpIA
wechat_url: 

-------

摘要: 原创出处 https://mp.weixin.qq.com/s/fbD9HDOmmHd9SHn7VnxpIA 「coding到灯火阑珊」欢迎转载，保留摘要，谢谢！

- [1. CircuitBreaker模块(熔断器)模块总体介绍](http://www.iocoder.cn/Resilience4j/coding/CircuitBreaker-21/)
- [2. CircuitBreaker模块各部分详解](http://www.iocoder.cn/Resilience4j/coding/CircuitBreaker-21/)
  - [2.1 CircuitBreakerRegistry（熔断器容器）](http://www.iocoder.cn/Resilience4j/coding/CircuitBreaker-21/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# 1. CircuitBreaker模块(熔断器)模块总体介绍

Resilience4j的CircuitBreaker主要由6个部分组成：管理熔断器实例的注册容器、熔断器的相关配置、熔断器的各种状态、触发熔断器状态变化的指标、熔断器行为变化产生的事件以及熔断器本身。

![img](http://static.iocoder.cn/696a813141bb899c58edd73179454754)

它们之间的基本调用关系如下图：

![img](http://static.iocoder.cn/f6c6cc1edad8ac12efab5f75c17d9e31)

# 2. CircuitBreaker模块各部分详解

源码位置如下图：

![img](http://static.iocoder.cn/187cda60918407cd86704b4cd0cda34e)

## 2.1 CircuitBreakerRegistry（熔断器容器）

![img](http://static.iocoder.cn/8b1de7bb1a176beed693dcaf40ffc75f)

CircuitBreakerRegistry接口的实现类InMemoryCircuitBreakerRegistry在内部使用ConcurrentHashMap容器来存储CircuitBreaker实例，保证了并发安全及原子性。

通过CircuitBreakerRegistry可以创建新的CircuitBreaker实例及检索创建的CircuitBreaker实例。



需主要关注的源码如下：

```Java
public interface CircuitBreakerRegistry {

   // 返回所有的CircuitBreaker实例
   Seq<CircuitBreaker> getAllCircuitBreakers();

   // 根据名称返回CircuitBreaker实例，
   // 如果不存在则根据默认配置，创建CircuitBreaker实例并返回
   CircuitBreaker circuitBreaker(String name);

   // 根据名称返回CircuitBreaker实例，
   // 如果不存在则 根据传入的配置实例，创建CircuitBreaker实例并返回
   CircuitBreaker circuitBreaker(String name, CircuitBreakerConfig circuitBreakerConfig);

   // 同上
   CircuitBreaker circuitBreaker(String name, Supplier<CircuitBreakerConfig> circuitBreakerConfigSupplier);



   // 根据自定义的配置，创建CircuitBreakerRegistry实例
   // 线程安全的单例
   static CircuitBreakerRegistry of(CircuitBreakerConfig circuitBreakerConfig){
​       return new InMemoryCircuitBreakerRegistry(circuitBreakerConfig);
   }



   // 使用默认配置，创建CircuitBreakerRegistry实例
   // 线程安全的单例
   static CircuitBreakerRegistry ofDefaults(){
​       return new InMemoryCircuitBreakerRegistry();
   }

}
```

注意：在CircuitBreakerRegistry接口中使用static方法实现了线程安全的单例模式。


```Java
final class InMemoryCircuitBreakerRegistry implements CircuitBreakerRegistry {

   private final CircuitBreakerConfig defaultCircuitBreakerConfig;

   private final ConcurrentMap<String, CircuitBreaker> circuitBreakers;

   public InMemoryCircuitBreakerRegistry() {
​       // 创建默认配置实例
​       this.defaultCircuitBreakerConfig = CircuitBreakerConfig.ofDefaults();

​       // 创建并发安全的容器放置CircuitBreaker实例
​       this.circuitBreakers = new ConcurrentHashMap<>();
   }

  ......



   // 调用CircuitBreaker.of(...)方法创建CircuitBreaker实例
   @Override
   public CircuitBreaker circuitBreaker(String name) {
​       return circuitBreakers.computeIfAbsent(Objects.requireNonNull(name, "Name must not be null"), (k) -> CircuitBreaker.of(name,
​               defaultCircuitBreakerConfig));
   }

......

}
```

在InMemoryCircuitBreakerRegistry实现类中，主要做了2件事：创建了ConcurrentHashMap实例及调用CricuitBreaker.of(...)方法来创建CircuitBreaker实例。

下一篇文章《Resilience4j源码解析-2.2 CircuitBreaker模块》讲解熔断器配置部分。
