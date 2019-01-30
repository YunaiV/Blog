title: Eureka 源码解析 —— 基于令牌桶算法的 RateLimiter
date: 2018-08-14
tags:
categories: Eureka
permalink: Eureka/rate-limiter
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484900&idx=2&sn=f947661b35c325f0f6aa4f23332aed49&chksm=fa497a55cd3ef343dcaedcbda0613b98876367cd24176f638fc1074411427ff5563180d81075&token=1286521154&lang=zh_CN#rd

---

摘要: 原创出处 http://www.iocoder.cn/Eureka/rate-limiter/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Eureka 1.8.X 版本**

- [1. 概述](http://www.iocoder.cn/Eureka/rate-limiter/)
- [2. RateLimiter](http://www.iocoder.cn/Eureka/rate-limiter/)
  - [2.1 refillToken](http://www.iocoder.cn/Eureka/rate-limiter/)
  - [2.2 consumeToken](http://www.iocoder.cn/Eureka/rate-limiter/)
- [3. RateLimitingFilter](http://www.iocoder.cn/Eureka/rate-limiter/)
- [4. InstanceInfoReplicator](http://www.iocoder.cn/Eureka/rate-limiter/)
- [666. 彩蛋](http://www.iocoder.cn/Eureka/rate-limiter/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2018_05_18.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

---

# 1. 概述

本文主要分享 **RateLimiter 的代码实现和 RateLimiter 在 Eureka 中的应用**。

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



# 2. RateLimiter

`com.netflix.discovery.util.RateLimiter` ，基于**Token Bucket Algorithm ( 令牌桶算法 )**的速率限制器。

> FROM [《接口限流实践》](http://www.cnblogs.com/LBSer/p/4083131.html)  
> 令牌桶算法的原理是系统会以一个恒定的速度往桶里放入令牌，而如果请求需要被处理，则需要先从桶里获取一个令牌，当桶里没有令牌可取时，则拒绝服务。    
> ![](http://www.iocoder.cn/images/Eureka/2018_08_14/01.png)

RateLimiter 目前支持**分钟级**和**秒级**两种速率限制。构造方法如下：

```Java
public class RateLimiter {

    /**
     * 速率单位转换成毫秒
     */
    private final long rateToMsConversion;
    
    public RateLimiter(TimeUnit averageRateUnit) {
        switch (averageRateUnit) {
            case SECONDS: // 秒级
                rateToMsConversion = 1000;
                break;
            case MINUTES: // 分钟级
                rateToMsConversion = 60 * 1000;
                break;
            default:
                throw new IllegalArgumentException("TimeUnit of " + averageRateUnit + " is not supported");
        }
    }
}
```

* `averageRateUnit` 参数，速率**单位**。构造方法里将 `averageRateUnit` 转换成 `rateToMsConversion` 。

调用 `#acquire(...)` 方法，获取令牌，并返回**是否获取成功**

```Java
// RateLimiter.java
/**
* 获取令牌( Token )
*
* @param burstSize 令牌桶上限
* @param averageRate 令牌再装平均速率
* @return 是否获取成功
*/
public boolean acquire(int burstSize, long averageRate) {
   return acquire(burstSize, averageRate, System.currentTimeMillis());
}

public boolean acquire(int burstSize, long averageRate, long currentTimeMillis) {
   if (burstSize <= 0 || averageRate <= 0) { // Instead of throwing exception, we just let all the traffic go
       return true;
   }

   // 填充 令牌
   refillToken(burstSize, averageRate, currentTimeMillis);
   // 消费 令牌
   return consumeToken(burstSize);
}
```

* `burstSize` 参数 ：令牌桶上限。
* `averageRate` 参数 ：令牌填充**平均**速率。
* 我们举个 🌰 来理解这两个参数 + 构造方法里的一个参数：
    * `averageRateUnit = SECONDS`
    * `averageRate = 2000`
    * `burstSize = 10`
    * 每**秒**可获取 `2000` 个令牌。例如，每秒允许请求 `2000` 次。
    * 每**毫秒**可填充 `2000 / 1000 = 2` 个**消耗**的令牌。
    * 每**毫秒**可获取 `10` 个令牌。例如，每毫秒允许请求上限为 `10` 次，并且请求**消耗**掉的令牌，需要逐步填充。这里要注意下，虽然每毫秒允许请求上限为 `10` 次，这是在没有任何令牌被**消耗**的情况下，实际每秒允许请求依然是 `2000` 次。
    * **这就是基于令牌桶算法的限流的特点：让流量平稳，而不是瞬间流量。1000 QPS 相对平均的分摊在这一秒内，而不是第 1 ms 999 请求，后面 999 ms 0 请求**。
* 从代码上看，`#acquire(...)` 分成两部分，我们分别解析，整体如下图：

    ![](http://www.iocoder.cn/images/Eureka/2018_08_14/02.png)

## 2.1 refillToken

调用 `#refillToken(...)` 方法，填充**已消耗**的令牌。可能很多同学开始和我想的一样，一个后台每毫秒执行填充。**为什么不适合这样呢？**一方面，实际项目里每个接口都会有相应的 RateLimiter ，导致**太多**执行频率**极高**的后台任务；另一方面，获取令牌时才计算，多次令牌填充可以合并成一次，减少冗余和无效的计算。

代码如下：

```Java
  1: /**
  2:  * 速率单位转换成毫秒
  3:  */
  4: private final long rateToMsConversion;
  5: 
  6: /**
  7:  * 消耗令牌数
  8:  */
  9: private final AtomicInteger consumedTokens = new AtomicInteger();
 10: /**
 11:  * 最后填充令牌的时间
 12:  */
 13: private final AtomicLong lastRefillTime = new AtomicLong(0);
 14: 
 15: private void refillToken(int burstSize, long averageRate, long currentTimeMillis) {
 16:     // 获得 最后填充令牌的时间
 17:     long refillTime = lastRefillTime.get();
 18:     // 获得 过去多少毫秒
 19:     long timeDelta = currentTimeMillis - refillTime;
 20: 
 21:     // 计算 可填充最大令牌数量
 22:     long newTokens = timeDelta * averageRate / rateToMsConversion;
 23:     if (newTokens > 0) {
 24:         // 计算 新的填充令牌的时间
 25:         long newRefillTime = refillTime == 0
 26:                 ? currentTimeMillis
 27:                 : refillTime + newTokens * rateToMsConversion / averageRate;
 28:         // CAS 保证有且仅有一个线程进入填充
 29:         if (lastRefillTime.compareAndSet(refillTime, newRefillTime)) {
 30:             while (true) { // 死循环，直到成功
 31:                 // 计算 填充令牌后的已消耗令牌数量
 32:                 int currentLevel = consumedTokens.get();
 33:                 int adjustedLevel = Math.min(currentLevel, burstSize); // In case burstSize decreased
 34:                 int newLevel = (int) Math.max(0, adjustedLevel - newTokens);
 35:                 // CAS 避免和正在消费令牌的线程冲突
 36:                 if (consumedTokens.compareAndSet(currentLevel, newLevel)) {
 37:                     return;
 38:                 }
 39:             }
 40:         }
 41:     }
 42: }
```

* 第 17 行 ：获取最后填充令牌的时间( `refillTime` ) 。每次填充令牌，会设置 `currentTimeMillis` 到 `refillTime` 。
* 第 19 行 ：获得距离最后填充令牌的时间差( `timeDelta` )，用于计算需要填充的令牌数。
* 第 22 行 ：计算**可填充的**最大令牌数量( `newTokens` )。`newTokens` 可能超过 `burstSize` ，所以下面会有逻辑调整 `newTokens` 。
* 第 25 至 27 行 ：计算**新的**填充令牌的时间。**为什么不能用 `currentTimeMillis` 呢**？例如，`averageRate = 500 && averageRateUnit = SECONDS` 时， 每 2 毫秒才填充一个令牌，如果设置 `currentTimeMillis` ，**会导致不足以填充一个令牌的时长被吞了**。
* 第 29 行 ：通过 **CAS** 保证有且**仅有一个**线程进入填充逻辑。
* 第 30 行 ：**死循环直到成功**。
* 第 32 至 34 行 ：计算**新的**填充令牌后的**已消耗**的令牌数量。
    * 第 33 行 ：`burstSize` 可能调小，例如，系统接入分布式配置中心，可以远程调整该数值。如果此时 `burstSize` 更小，以它作为**已消耗**的令牌数量。
* 第 36 行 ：通过 **CAS** 保证避免覆盖设置正在消费令牌的线程。

## 2.2 consumeToken

用 `#refillToken(...)` 方法，填充**消耗( 获取 )**的令牌。

代码如下 ：

```Java
  1: private boolean consumeToken(int burstSize) {
  2:     while (true) { // 死循环，直到没有令牌，或者获取令牌成功
  3:         // 没有令牌
  4:         int currentLevel = consumedTokens.get();
  5:         if (currentLevel >= burstSize) {
  6:             return false;
  7:         }
  8:         // CAS 避免和正在消费令牌或者填充令牌的线程冲突
  9:         if (consumedTokens.compareAndSet(currentLevel, currentLevel + 1)) {
 10:             return true;
 11:         }
 12:     }
 13: }
```

* 第 2 行 ：**死循环直到没有令牌或者竞争获取令牌成功**。
* 第 4 至 7 行 ：没有令牌。
* 第 9 至 11 行 ：通过 **CAS** 避免和正在消费令牌或者填充令牌的线程冲突。

# 3. RateLimitingFilter

`com.netflix.eureka.RateLimitingFilter` ，Eureka-Server 限流过滤器。使用 RateLimiting ，保证 Eureka-Server 稳定性。

`#doFilter(...)` 方法，代码如下：

```Java
  1: @Override
  2: public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
  3:     // 获得 Target
  4:     Target target = getTarget(request);
  5: 
  6:     // Other Target ，不做限流
  7:     if (target == Target.Other) {
  8:         chain.doFilter(request, response);
  9:         return;
 10:     }
 11: 
 12:     HttpServletRequest httpRequest = (HttpServletRequest) request;
 13:     // 判断是否被限流
 14:     if (isRateLimited(httpRequest, target)) {
 15:         // TODO[0012]：监控相关，跳过
 16:         incrementStats(target);
 17:         // 如果开启限流，返回 503 状态码
 18:         if (serverConfig.isRateLimiterEnabled()) {
 19:             ((HttpServletResponse) response).setStatus(HttpServletResponse.SC_SERVICE_UNAVAILABLE);
 20:             return;
 21:         }
 22:     }
 23:     chain.doFilter(request, response);
 24: }
```

* 第 4 行 ：调用 `#getTarget()` 方法，获取 Target。RateLimitingFilter 只对符合正在表达式 `^.*/apps(/[^/]*)?$` 的接口做限流，其中不包含 Eureka-Server 集群批量同步接口。
    * 点击 [链接](https://github.com/YunaiV/eureka/blob/7f868f9ca715a8862c0c10cac04e238bbf371db0/eureka-core/src/main/java/com/netflix/eureka/RateLimitingFilter.java#L98) 查看 Target 枚举类代码。
    * 点击 [链接](https://github.com/YunaiV/eureka/blob/7f868f9ca715a8862c0c10cac04e238bbf371db0/eureka-core/src/main/java/com/netflix/eureka/RateLimitingFilter.java#L150) 查看 `#getTarget(...)` 方法代码。  
* 第 14 行 ：调用 `#isRateLimited(...)` 方法，判断是否被限流。代码如下：

    ```Java
      1: private boolean isRateLimited(HttpServletRequest request, Target target) {
      2:     // 判断是否特权应用
      3:     if (isPrivileged(request)) {
      4:         logger.debug("Privileged {} request", target);
      5:         return false;
      6:     }
      7:     // 判断是否被超载( 限流 )
      8:     if (isOverloaded(target)) {
      9:         logger.debug("Overloaded {} request; discarding it", target);
     10:         return true;
     11:     }
     12:     logger.debug("{} request admitted", target);
     13:     return false;
     14: }
    ```
    * 第 3 至 6 行 ：调用 `#isPrivileged()` 方法，判断是否为特权应用，对特权应用不开启限流逻辑。代码如下：

        ```Java
        private boolean isPrivileged(HttpServletRequest request) {
            // 是否对标准客户端开启限流
            if (serverConfig.isRateLimiterThrottleStandardClients()) {
                return false;
            }
            // 以请求头( "DiscoveryIdentity-Name" ) 判断是否在标准客户端名集合内
            Set<String> privilegedClients = serverConfig.getRateLimiterPrivilegedClients();
            String clientName = request.getHeader(AbstractEurekaIdentity.AUTH_NAME_HEADER_KEY);
            return privilegedClients.contains(clientName) || DEFAULT_PRIVILEGED_CLIENTS.contains(clientName);
        }
        ```
        * x

    * 第 8 至 11 行 ：调用 `#isOverloaded(...)` 方法，判断是否超载( 限流 )。代码如下：

        ```Java
        /**
        * Includes both full and delta fetches.
        */
        private static final RateLimiter registryFetchRateLimiter = new RateLimiter(TimeUnit.SECONDS);
        
        /**
        * Only full registry fetches.
        */
        private static final RateLimiter registryFullFetchRateLimiter = new RateLimiter(TimeUnit.SECONDS);
        
        private boolean isOverloaded(Target target) {
            int maxInWindow = serverConfig.getRateLimiterBurstSize(); // 10
            int fetchWindowSize = serverConfig.getRateLimiterRegistryFetchAverageRate(); // 500
            boolean overloaded = !registryFetchRateLimiter.acquire(maxInWindow, fetchWindowSize);
            if (target == Target.FullFetch) {
                int fullFetchWindowSize = serverConfig.getRateLimiterFullFetchAverageRate(); // 100
                    overloaded |= !registryFullFetchRateLimiter.acquire(maxInWindow, fullFetchWindowSize);
            }
            return overloaded;
        }
        ```
        * x

* 第 18 至 21 行 ：若 `eureka.rateLimiter.enabled = true`( 默认值 ：`false` ，可配 )，返回 503 状态码。

# 4. InstanceInfoReplicator

`com.netflix.discovery.InstanceInfoReplicator` ，Eureka-Client 应用实例复制器。在 [《Eureka 源码解析 —— 应用实例注册发现（一）之注册》「2.1 应用实例信息复制器」](http://www.iocoder.cn/Eureka/instance-registry-register/?self) 有详细解析。

应用实例状态发生变化时，调用 `#onDemandUpdate()` 方法，向 Eureka-Server 发起注册，同步应用实例信息。InstanceInfoReplicator 使用 RateLimiter ，避免状态**频繁**发生变化，向 Eureka-Server **频繁**同步。代码如下：

```Java
class InstanceInfoReplicator implements Runnable {

    /**
     * RateLimiter
     */
    private final RateLimiter rateLimiter;
    /**
     * 令牌桶上限，默认：2
     */
    private final int burstSize;
    /**
     * 令牌再装平均速率，默认：60 * 2 / 30 = 4
     */
    private final int allowedRatePerMinute;
    
    InstanceInfoReplicator(DiscoveryClient discoveryClient, InstanceInfo instanceInfo, int replicationIntervalSeconds, int burstSize) {
        // ... 省略其他代码
        
        this.rateLimiter = new RateLimiter(TimeUnit.MINUTES);
        this.replicationIntervalSeconds = replicationIntervalSeconds;
        this.burstSize = burstSize;

        this.allowedRatePerMinute = 60 * this.burstSize / this.replicationIntervalSeconds;
        logger.info("InstanceInfoReplicator onDemand update allowed rate per min is {}", allowedRatePerMinute);
    }

    public boolean onDemandUpdate() {
        if (rateLimiter.acquire(burstSize, allowedRatePerMinute)) { // 限流
            scheduler.submit(new Runnable() {
                @Override
                public void run() {
                    logger.debug("Executing on-demand update of local InstanceInfo");
                    // 取消任务
                    Future latestPeriodic = scheduledPeriodicRef.get();
                    if (latestPeriodic != null && !latestPeriodic.isDone()) {
                        logger.debug("Canceling the latest scheduled update, it will be rescheduled at the end of on demand update");
                        latestPeriodic.cancel(false);
                    }
                    // 再次调用
                    InstanceInfoReplicator.this.run();
                }
            });
            return true;
        } else {
            logger.warn("Ignoring onDemand update due to rate limiter");
            return false;
        }
    }

}
```

* 在 `#onDemandUpdate()` 方法，调用 `RateLimiter#acquire(...)` 方法，获取令牌。
    * 若获取成功，向 Eureka-Server 发起注册，同步应用实例信息。
    * 若获取失败，**不**向 Eureka-Server 发起注册，同步应用实例信息。**这样会不会有问题**？答案是**不会**。
        * InstanceInfoReplicator 会**固定周期**检查本地应用实例是否有没向 Eureka-Server ，若未同步，则发起同步。在 [《Eureka 源码解析 —— 应用实例注册发现（一）之注册》「2.1 应用实例信息复制器」](http://www.iocoder.cn/Eureka/instance-registry-register/?self) 有详细解析。
        * Eureka-Client 向 Eureka-Server 心跳时，Eureka-Server 会对比应用实例的 `lastDirtyTimestamp` ，若 Eureka-Client 的更大，则 Eureka-Server 返回 404 状态码。Eureka-Client 接收到 404 状态码后，发起注册同步。在 [Eureka 源码解析 —— 应用实例注册发现（二）之续租》「2.2 HeartbeatThread」](http://www.iocoder.cn/Eureka/instance-registry-renew/?self) 有详细解析。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

后面找时间研究下 Google Guava RateLimiter 的源码实现，从功能上更加强大，感兴趣的胖友可以瞅瞅呀。

胖友，分享我的公众号( **芋道源码** ) 给你的胖友可好？

