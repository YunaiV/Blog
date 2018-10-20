title: Sentinel 为 Dubbo 服务保驾护航
date: 2018-01-02
tag: 
categories: Sentinel
permalink: Sentinel/all/sentinel-introduction-for-dubbo
author: 未知
from_url: http://dubbo.incubator.apache.org/zh-cn/blog/sentinel-introduction-for-dubbo.html
wechat_url: 

-------

摘要: 原创出处 http://dubbo.incubator.apache.org/zh-cn/blog/sentinel-introduction-for-dubbo.html 「未知」欢迎转载，保留摘要，谢谢！

- [快速接入 Sentinel](http://www.iocoder.cn/Sentinel/all/sentinel-introduction-for-dubbo/)
- [限流必备 - 监控管理](http://www.iocoder.cn/Sentinel/all/sentinel-introduction-for-dubbo/)
- [Sentinel 基于 Dubbo 的最佳实践](http://www.iocoder.cn/Sentinel/all/sentinel-introduction-for-dubbo/)
  - [Service Provider](http://www.iocoder.cn/Sentinel/all/sentinel-introduction-for-dubbo/)
  - [Service Consumer](http://www.iocoder.cn/Sentinel/all/sentinel-introduction-for-dubbo/)
  - [Fallback](http://www.iocoder.cn/Sentinel/all/sentinel-introduction-for-dubbo/)
- [Sentinel 与 Hystrix 的比较](http://www.iocoder.cn/Sentinel/all/sentinel-introduction-for-dubbo/)
- [总结](http://www.iocoder.cn/Sentinel/all/sentinel-introduction-for-dubbo/)
- [666. 彩蛋](http://www.iocoder.cn/Sentinel/all/sentinel-introduction-for-dubbo/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

在复杂的生产环境下可能部署着成千上万的 Dubbo 服务实例，流量持续不断地进入，服务之间进行相互调用。但是分布式系统中可能会因流量激增、系统负载过高、网络延迟等一系列问题，导致某些服务不可用，如果不进行相应的控制可能导致级联故障，影响服务的可用性，因此如何对流量进行合理的控制，成为保障服务稳定性的关键。

[Sentinel](https://github.com/alibaba/Sentinel) 是阿里中间件团队开源的，面向分布式服务架构的轻量级流量控制产品，主要以流量为切入点，从**流量控制**、**熔断降级**、**系统负载保护**等多个维度来帮助用户保护服务的稳定性。本文将基于 Dubbo，看看 Sentinel 是如何进行流量控制的，并且提供 Dubbo 整合 Sentinel 的最佳实践。

# 快速接入 Sentinel

Sentinel 意为**哨兵**，这个命名形象的诠释了 Sentinel 在分布式系统中的工作角色和重要性。以 Sentinel 在 Dubbo 生态系统中的作用为例，Dubbo 的核心模块包括注册中心、服务提供方、服务消费方（服务调用方）和监控四个模块。Sentinel 通过对服务提供方和服务消费方的限流来进一步提升服务的可用性。接下来我们看看 Sentinel 对服务提供方和服务消费方限流的技术实现方式。

![Dubbo Arch](http://dubbo.incubator.apache.org/img/architecture.png)

Sentinel 提供了与 Dubbo 适配的模块 – [Sentinel Dubbo Adapter](https://github.com/dubbo/dubbo-sentinel-support)，包括针对服务提供方的过滤器和服务消费方的过滤器（Filter）。使用时我们只需引入以下模块（以 Maven 为例）：

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-dubbo-adapter</artifactId>
    <version>x.y.z</version>
</dependency>
```

引入此依赖后，Dubbo 的服务接口和方法（包括调用端和服务端）就会成为 Sentinel 中的资源，在配置了规则后就可以自动享受到 Sentinel 的防护能力。同时提供了灵活的配置选项，例如若不希望开启 Sentinel Dubbo Adapter 中的某个 Filter，可以手动关闭对应的 Filter。

接入 Sentinel Dubbo Adapter 后，即使未配置规则，Sentinel 也会对相应的 Dubbo 服务的调用信息进行统计。那么我们怎么知道 Sentinel 接入成功了呢？这时候就要请出一大利器 —— Sentinel 控制台了。

# 限流必备 - 监控管理

流量具有很强的实时性，之所以需要限流，是因为我们无法对流量的到来作出精确的预判，不然的话我们完全可以通过弹性的计算资源来处理，所以这时候为了保证限流的准确性，限流框架的监控功能就非常重要了。

Sentinel 的控制台（Dashboard）是流量控制、熔断降级规则统一配置和管理的入口，同时它为用户提供了多个维度的监控功能。在 Sentinel 控制台上，我们可以配置规则并实时查看流量控制效果。

接入 Sentinel 控制台的步骤如下（**缺一不可**）：

1. 按照 [Sentinel 控制台文档](https://github.com/alibaba/Sentinel/wiki/%E6%8E%A7%E5%88%B6%E5%8F%B0) 启动控制台
2. 应用引入 `sentinel-transport-simple-http` 依赖，以便控制台可以拉取对应应用的相关信息
3. 给应用添加相关的启动参数，启动应用。需要配置的参数有：
   - `-Dcsp.sentinel.api.port`：客户端的 port，用于上报相关信息（默认为 8719）
   - `-Dcsp.sentinel.dashboard.server`：控制台的地址
   - `-Dproject.name`：应用名称，会在控制台中显示

注意某些环境下本地运行 Dubbo 服务还需要加上 `-Djava.net.preferIPv4Stack=true` 参数。比如中 Service Provider 的启动参数可以配成：

```bash
-Djava.net.preferIPv4Stack=true -Dcsp.sentinel.api.port=8720 -Dcsp.sentinel.dashboard.server=localhost:8080 -Dproject.name=dubbo-provider-demo
```

这样在启动应用后就能在控制台找到对应的应用了。以下是常用功能：

- **单台设备监控**：当在机器列表中看到您的机器，就代表着已经成功接入控制台，可以查看单台设备的设备名称、IP地址、端口号、健康状态和心跳时间等信息。

![Discovery](https://github.com/alibaba/Sentinel/wiki/image/machinediscover.png)

- **链路监控**：簇点链路实时的去拉取指定客户端资源的运行情况，它提供了两种展示模式，一种用书状结构展示资源的调用链路；另外一种则不区分调用链路展示资源的运行情况。通过链路监控，可以查看到每个资源的流控和降级的历史状态。

| 树状链路                                                     | 平铺链路                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![resourceTree](https://github.com/alibaba/Sentinel/wiki/image/resourceTree.png) | ![cluster](https://github.com/alibaba/Sentinel/wiki/image/sentine_dashboard.gif) |

- **聚合监控**：同一个服务下的所有机器的簇点信息会被汇总，实现实时监控，精确度达秒级。

![秒级实时监控](http://dubbo.incubator.apache.org/img/blog/sentinel-dashboard-metrics.png)

- **规则配置**：可以查看已有的限流、降级和系统保护规则，并实时地进行配置。

![规则配置](http://dubbo.incubator.apache.org/img/blog/sentinel-dashboard-view-rules.png)

# Sentinel 基于 Dubbo 的最佳实践

> 具体 Demo 代码请见 [sentinel-demo-dubbo](https://github.com/alibaba/Sentinel/tree/master/sentinel-demo/sentinel-demo-dubbo)。

## Service Provider

> 对服务提供方的流量控制可分为**服务提供方的自我保护能力**和**服务提供方对服务消费方的请求分配能力**两个维度。

Service Provider 用于向外界提供服务，处理各个消费者的调用请求。为了保护 Provider 不被激增的流量拖垮影响稳定性，可以给 Provider 配置 **QPS 模式**的限流，这样当每秒的请求量超过设定的阈值时会自动拒绝多的请求。限流粒度可以是 *服务接口* 和 *服务方法* 两种粒度。若希望整个服务接口的 QPS 不超过一定数值，则可以为对应服务接口资源（resourceName 为**接口全限定名**）配置 QPS 阈值；若希望服务的某个方法的 QPS 不超过一定数值，则可以为对应服务方法资源（resourceName 为**接口全限定名:方法签名**）配置 QPS 阈值。有关配置详情请参考 [流量控制 | Sentinel](https://github.com/alibaba/Sentinel/wiki/%E6%B5%81%E9%87%8F%E6%8E%A7%E5%88%B6)。

我们看一下这种模式的限流产生的效果。假设我们已经定义了某个服务接口 `com.alibaba.csp.sentinel.demo.dubbo.FooService`，其中有一个方法 `sayHello(java.lang.String)`，Provider 端该方法设定 QPS 阈值为 10。在 Consumer 端在 1s 之内连续发起 15 次调用，可以通过日志文件看到 Provider 端被限流。拦截日志统一记录在 `~/logs/csp/sentinel-block.log` 中：

```
2018-07-24 17:13:43|1|com.alibaba.csp.sentinel.demo.dubbo.FooService:sayHello(java.lang.String),FlowException,default,|5,0
```

在 Provider 对应的 metrics 日志中也有记录：

```
1532423623000|2018-07-24 17:13:43|com.alibaba.csp.sentinel.demo.dubbo.FooService|15|0|15|0|3
1532423623000|2018-07-24 17:13:43|com.alibaba.csp.sentinel.demo.dubbo.FooService:sayHello(java.lang.String)|10|5|10|0|0
```

根据**调用方**的需求来分配服务提供方的处理能力也是常见的限流方式。比如有两个服务 A 和 B 都向 Service Provider 发起调用请求，我们希望只对来自服务 B 的请求进行限流，则可以设置限流规则的 `limitApp` 为服务 B 的名称。Sentinel Dubbo Adapter 会自动解析 Dubbo 消费者（调用方）的 application name 作为调用方名称（`origin`），在进行资源保护的时候都会带上调用方名称。若限流规则未配置调用方（`default`），则该限流规则对所有调用方生效。若限流规则配置了调用方则限流规则将仅对指定调用方生效。

> 注：Dubbo 默认通信不携带对端 application name 信息，因此需要开发者在调用端手动将 application name 置入 attachment 中，provider 端进行相应的解析。Sentinel Dubbo Adapter 实现了一个 Filter 用于自动从 consumer 端向 provider 端透传 application name。若调用端未引入 Sentinel Dubbo Adapter，又希望根据调用端限流，可以在调用端手动将 application name 置入 attachment 中，key 为 `dubboApplication`。

在限流日志中会也会记录调用方的名称，如下面的日志中的 `demo-consumer` 即为调用方名称：

```
2018-07-25 16:26:48|1|com.alibaba.csp.sentinel.demo.dubbo.FooService:sayHello(java.lang.String),FlowException,default,demo-consumer|5,0
```

## Service Consumer

> 对服务提供方的流量控制可分为**控制并发线程数**和**服务降级**两个维度。

### 并发线程数限流

Service Consumer 作为客户端去调用远程服务。每一个服务都可能会依赖几个下游服务，若某个服务 A 依赖的下游服务 B 出现了不稳定的情况，服务 A 请求 服务 B 的响应时间变长，从而服务 A 调用服务 B 的线程就会产生堆积，最终可能耗尽服务 A 的线程数。我们通过用并发线程数来控制对下游服务 B 的访问，来保证下游服务不可靠的时候，不会拖垮服务自身。基于这种场景，推荐给 Consumer 配置**线程数模式**的限流，来保证自身不被不稳定服务所影响。采用基于线程数的限流模式后，我们不需要再显式地去进行线程池隔离，Sentinel 会控制资源的线程数，超出的请求直接拒绝，直到堆积的线程处理完成，可以达到**信号量隔离**的效果。

我们看一下这种模式的效果。假设当前服务 A 依赖两个远程服务方法 `sayHello(java.lang.String)` 和 `doAnother()`。前者远程调用的响应时间 为 1s-1.5s 之间，后者 RT 非常小（30 ms 左右）。服务 A 端设两个远程方法 thread count 为 5。然后每隔 50 ms 左右向线程池投入两个任务，作为消费者分别远程调用对应方法，持续 10 次。可以看到 `sayHello` 方法被限流 5 次，因为后面调用的时候前面的远程调用还未返回（RT 高）；而 `doAnother()` 调用则不受影响。线程数目超出时快速失败能够有效地防止自己被慢调用所影响。

### 服务降级

当服务依赖于多个下游服务，而某个下游服务调用非常慢时，会严重影响当前服务的调用。这里我们可以利用 Sentinel 熔断降级的功能，为调用端配置基于平均 RT 的[降级规则](https://github.com/alibaba/Sentinel/wiki/%E7%86%94%E6%96%AD%E9%99%8D%E7%BA%A7)。这样当调用链路中某个服务调用的平均 RT 升高，在一定的次数内超过配置的 RT 阈值，Sentinel 就会对此调用资源进行降级操作，接下来的调用都会立刻拒绝，直到过了一段设定的时间后才恢复，从而保护服务不被调用端短板所影响。同时可以配合 fallback 功能使用，在被降级的时候提供相应的处理逻辑。

## Fallback

从 0.1.1 版本开始，Sentinel Dubbo Adapter 还支持配置全局的 fallback 函数，可以在 Dubbo 服务被限流/降级/负载保护的时候进行相应的 fallback 处理。用户只需要实现自定义的 [`DubboFallback`](https://github.com/alibaba/Sentinel/blob/master/sentinel-adapter/sentinel-dubbo-adapter/src/main/java/com/alibaba/csp/sentinel/adapter/dubbo/fallback/DubboFallback.java) 接口，并通过 `DubboFallbackRegistry` 注册即可。默认情况会直接将 `BlockException` 包装后抛出。同时，我们还可以配合 [Dubbo 的 fallback 机制](http://dubbo.apache.org/#!/docs/user/demos/local-mock.md?lang=zh-cn) 来为降级的服务提供替代的实现。

# Sentinel 与 Hystrix 的比较

目前业界常用的熔断降级/隔离的库是 Netflix 的 [Hystrix](https://github.com/Netflix/Hystrix)，那么 Sentinel 与 Hystrix 有什么异同呢？Hystrix 的关注点在于以 *隔离* 和 *熔断* 为主的容错机制，而 Sentinel 的侧重点在于多样化的流量控制、熔断降级、系统负载保护、实时监控和控制台，可以看到解决的问题还是有比较大的不同的。

Hystrix 采用命令模式封装资源调用逻辑，并且资源的定义与隔离规则是强依赖的，即在创建 HystrixCommand 的时候就要指定隔离规则（因其执行模型依赖于隔离模式）。Sentinel 的设计更为简单，不关注资源是如何执行的，资源的定义与规则的配置相分离。用户可以先定义好资源，然后在需要的时候配置规则即可。Sentinel 的原则非常简单：根据对应资源配置的规则来为资源执行相应的限流/降级/负载保护策略，若规则未配置则仅进行统计。从 0.1.1 版本开始，Sentinel 还引入了[注解支持](https://github.com/alibaba/Sentinel/wiki/%E6%B3%A8%E8%A7%A3%E6%94%AF%E6%8C%81)，可以更方便地定义资源。

隔离是 Hystrix 的核心功能。Hystrix 通过线程池或信号量的方式来对依赖（即 Sentinel 中对应的资源）进行隔离，其中最常用的是资源隔离。Hystrix 线程池隔离的好处是比较彻底，但是不足之处在于要开很多线程池，在应用本身线程数目比较多的时候上下文切换的 overhead 会非常大；Hystrix 的信号量隔离模式可以限制调用的并发数而不显式创建线程，这样的方式比较轻量级，但缺点是无法对慢调用自动进行降级，只能等待客户端自己超时，因此仍然可能会出现级联阻塞的情况。Sentinel 可以通过并发线程数模式的流量控制来提供信号量隔离的功能。并且结合基于响应时间的熔断降级模式，可以在不稳定资源的平均响应时间比较高的时候自动降级，防止过多的慢调用占满并发数，影响整个系统。

Hystrix 熔断降级功能采用熔断器模式，在某个服务失败比率高时自动进行熔断。Sentinel 的熔断降级功能更为通用，支持平均响应时间与失败比率两个指标。Sentinel 还提供各种调用链路关系和流量控制效果支持，同时还可以根据系统负载去实时地调整流量来保护系统，应用场景更为丰富。同时，Sentinel 还提供了实时的监控 API 和控制台，可以方便用户快速了解目前系统的状态，对服务的稳定性了如指掌。

更详细的对比请参见 [Sentinel 与 Hystrix 的对比](https://github.com/alibaba/Sentinel/wiki/Sentinel-%E4%B8%8E-Hystrix-%E7%9A%84%E5%AF%B9%E6%AF%94)。

# 总结

以上介绍的只是 Sentinel 的一个最简单的场景 —— 限流。Sentinel 还可以处理更复杂的各种情况，比如超时熔断、冷启动、请求匀速等。可以参考 [Sentinel 文档](https://github.com/alibaba/Sentinel/wiki/%E4%B8%BB%E9%A1%B5)，更多的场景等待你去挖掘！

# 666. 彩蛋

如果你对 Sentinel 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)