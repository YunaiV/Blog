title: Resilience4j 源码解析-2.2 CircuitBreaker模块之配置
date: 2018-01-04
tag: 
categories: Resilience4j
permalink: Resilience4j/coding/CircuitBreaker-Config
author: coding到灯火阑珊
from_url: https://mp.weixin.qq.com/s/chaHuJGqoRluMzRL1A3mkg
wechat_url: 

-------

摘要: 原创出处 https://mp.weixin.qq.com/s/chaHuJGqoRluMzRL1A3mkg 「coding到灯火阑珊」欢迎转载，保留摘要，谢谢！

- [1. CircuitBreakerConfig（熔断器配置）](http://www.iocoder.cn/Resilience4j/coding/CircuitBreaker-Config/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# 1. CircuitBreakerConfig（熔断器配置）

CircuitBreakerConfig类中封装了与熔断器相关的配置属性，属性包括：请求调用失败的阈值、熔断器在打开状态时的持续时间、熔断器在半开状态下的Ring Buffer大小、熔断器在关闭状态下的Ring Buffer大小、是否记录请求调用失败的断言。Ring Buffer的原理在后面研究CircuitBreakerMetrics部分时再详细讲解，现在只需要了解，它是一个存储每次请求调用成功与否的环形缓存区。

![img](http://static.iocoder.cn/5e42ffda6824c4707ab0a0cb77698e58)


另外CircuitBreakerConfig类通过Builder模式构造CircuitBreakerConfig实例及流式的设置配置属性值。

需主要关注的源码如下：


```Java
// 请求调用失败的阈值，百分比。默认是50%
DEFAULT_MAX_FAILURE_THRESHOLD = 50; // Percentage

// 熔断器在打开状态时的持续时间。默认是60秒
DEFAULT_WAIT_DURATION_IN_OPEN_STATE = 60; // Seconds

// 熔断器在半开状态下的ring buffer大小。默认10
DEFAULT_RING_BUFFER_SIZE_IN_HALF_OPEN_STATE = 10;

// 熔断器在关闭状态下的ring buffer大小。默认100
DEFAULT_RING_BUFFER_SIZE_IN_CLOSED_STATE = 100;

// 是否记录请求调用失败的断言，默认所有异常都记录。
DEFAULT_RECORD_FAILURE_PREDICATE = (throwable) -> true;
```


------

```Java
/**
 * 构造者模式
*/
public static class Builder {

**......**

// 请求调用失败，存储异常记录的集合
private Class<? extends Throwable>[] recordExceptions = new Class[0];

// 请求调用失败，忽略异常记录的集合
private Class<? extends Throwable>[] ignoreExceptions = new Class[0];

**......**

/**
 * 异常记录集合与异常忽略集合取交集，如果有值则为false，否则为true。
*/
private void buildErrorRecordingPredicate() {
   this.errorRecordingPredicate =
​           getRecordingPredicate()
​                   .and(buildIgnoreExceptionsPredicate()
​                           .orElse(DEFAULT_RECORD_FAILURE_PREDICATE));
}

**......**

}
```


下一篇文章《Resilience4j源码解析-2.3 CircuitBreaker模块之有限状态机》讲解熔断器的核心理念-有限状态机，及状态转换。
