title: RxJava 源码解析 —— Scheduler
date: 2019-01-15
tags:
categories: RxJava
permalink: RxJava/scheduler

-------

摘要: 原创出处 http://www.iocoder.cn/RxJava/scheduler/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 RxJava 1.2.X 版本**  

本系列写作目的，为了辅助 Hystrix 的理解，因此会较为零散与琐碎，望见谅见谅。

- [1. Scheduler](http://www.iocoder.cn/RxJava/scheduler/)
- [2. Worker](http://www.iocoder.cn/RxJava/scheduler/)
- [3. 默认调度器实现](http://www.iocoder.cn/RxJava/scheduler/)
- [4. 操作符与调度器](http://www.iocoder.cn/RxJava/scheduler/)
- [5. 使用示例](http://www.iocoder.cn/RxJava/scheduler/)
- [666. 彩蛋](http://www.iocoder.cn/RxJava/scheduler/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2018_05_18.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

# 1. Scheduler

`rx.Scheduler` ，**抽象类**，一个可以调度工作单元( `rx.Scheduler.Worker` )**们**的对象。

> FROM [《ReactiveX文档中文翻译 —— Scheduler》](https://mcxiaoke.gitbooks.io/rxdocs/content/Scheduler.html)  
> 如果你想给 Observable 操作符链添加**多线程**功能，你可以指定操作符( 或者特定的Observable )在特定的调度器( Scheduler )上执行。

> 某些 ReactiveX 的 Observable 操作符有一些变体，它们可以接受一个 Scheduler 参数。这个参数指定操作符将它们的部分或全部任务放在一个特定的调度器上执行。

> **使用 ObserveOn 和 SubscribeOn 操作符**，你可以让 Observable 在一个特定的调度器上执行。  
> 
> * ObserveOn 指示一个 Observable 在一个特定的调度器上调用观察者的 onNext , onError 和 onCompleted 方法。
> * SubscribeOn 更进一步，它指示 Observable 将全部的处理过程( 包括发射数据和通知 )放在特定的调度器上执行。

* [`Observable#subscribeOn(Scheduler)`](https://github.com/ReactiveX/RxJava/blob/5b2394c9ee91f298661fff5e043744c84b425808/src/main/java/rx/Observable.java#L10404) ，在 [《RxJava 源码解析 —— Observable#subscribeOn(Scheduler)》](http://www.iocoder.cn/RxJava/observable-subscribe-on-scheduler/) 详细解析。

为什么是**抽象类**，而不是**接口**呢？官方说明如下 ：

```Java
/*
 * Why is this an abstract class instead of an interface?
 *
 *  : Java doesn't support extension methods and there are many overload methods needing default
 *    implementations.
 *
 *  : Virtual extension methods aren't available until Java8 which RxJava will not set as a minimum target for
 *    a long time.
 *
 *  : If only an interface were used Scheduler implementations would then need to extend from an
 *    AbstractScheduler pair that gives all of the functionality unless they intend on copy/pasting the
 *    functionality.
 *
 *  : Without virtual extension methods even additive changes are breaking and thus severely impede library
 *    maintenance.
 */
```

* 【第一、二点】Java 在 8.0 版本之前，接口不支持默认实现方法，而 Scheduler 需要多个方法提供默认实现。RxJava 考虑到兼容性，将长期使用低版本的 Java 。
* 【第三、四点】如果将 Scheduler 定义为接口，那么需要添加一个 AbstractScheduler 抽象类，实现接口的默认方法实现。

Scheduler 提供方法如下 ：

* [`#createWorker()`](https://github.com/ReactiveX/RxJava/blob/5b2394c9ee91f298661fff5e043744c84b425808/src/main/java/rx/Scheduler.java#L54) **抽象**方法 ：创建 Worker 。
* [`#now()`](https://github.com/ReactiveX/RxJava/blob/5b2394c9ee91f298661fff5e043744c84b425808/src/main/java/rx/Scheduler.java#L129) **默认**方法 ：返回当前时间。
* ~~[`#when(...)`](https://github.com/ReactiveX/RxJava/blob/5b2394c9ee91f298661fff5e043744c84b425808/src/main/java/rx/Scheduler.java#L208) **默认**方法 ：跳过，Hystrix 暂未使用。~~

# 2. Worker

`rx.Scheduler.Worker` ，工作单元对象**抽象类**，执行 Scheduler 调度的操作( `rx.functions.Action0` )。

Worker 提供方法如下 ：

* [`#schedule(Action0)`](https://github.com/ReactiveX/RxJava/blob/5b2394c9ee91f298661fff5e043744c84b425808/src/main/java/rx/Scheduler.java#L70) **抽象**方法 ：**立即**调度操作。
* [`#schedulePeriodically(Action0, long, long, TimeUnit)`](https://github.com/ReactiveX/RxJava/blob/5b2394c9ee91f298661fff5e043744c84b425808/src/main/java/rx/Scheduler.java#L87) **抽象**方法：**延迟**调度操作。
* ~~[`#schedulePeriodically(Action0, long, long, TimeUnit)`](https://github.com/ReactiveX/RxJava/blob/5b2394c9ee91f298661fff5e043744c84b425808/src/main/java/rx/Scheduler.java#L109) **默认**方法 ：**周期性**操作。跳过，Hystrix 暂未使用。~~
* [`#now()`](https://github.com/ReactiveX/RxJava/blob/5b2394c9ee91f298661fff5e043744c84b425808/src/main/java/rx/Scheduler.java#L119) **默认**方法 ：返回当前时间。

Worker 实现 `rx.Subscription` **接口**，但是并未实现对应的方法，需要子类实现，用于 ：

* `#unsubscribe()` ：**原意**取消订阅，**实意**取消操作。
* `#isUnsubscribed()` ：**原意**订阅是否取消，**实意**操作是否取消。

# 3. 默认调度器实现

在 `rx.internal.schedulers` 包下，提供了多种默认调度器的实现。

![](http://www.iocoder.cn/images/RxJava/2019_01_15/01.png)

[`rx.schedulers.Schedulers`](https://github.com/ReactiveX/RxJava/blob/5b2394c9ee91f298661fff5e043744c84b425808/src/main/java/rx/schedulers/Schedulers.java) ，默认调度器单例工厂，创建上图调度器工厂并进行管理。

> 参考 [《ReactiveX文档中文翻译 —— Scheduler》「调度器的种类」](https://mcxiaoke.gitbooks.io/rxdocs/content/Scheduler.html#调度器的种类)

| 单例 | 类 | 说明 |
| --- | --- | --- |
| `Schedulers#io()` | CachedThreadScheduler | 用于 IO 密集型任务，如异步阻塞 IO 操作，这个调度器的线程池会根据需要增长 |
| `Schedulers#computation()` | EventLoopsScheduler | 用于普通的计算任务，默认线程数等于处理器的数量 |
| `Schedulers#from(Executor)` | ExecutorScheduler | 使用指定的 Executor 作为调度器 |
| `Schedulers#immediate()` | ImmediateScheduler | 在当前线程立即开始执行任务 |
| `Schedulers#newThread()` | NewThreadScheduler | 为每个任务创建一个新线程 |
| `Schedulers#trampoline()` | TrampolineScheduler | 当其它排队的任务完成后，在当前线程排队开始执行 |

在 Hystrix 里，继承 Scheduler **抽象类**，实现了**自定义**的 Scheduler 。

因此，跳过默认调度器的源码解析。

# 4. 操作符与调度器

点击 [《ReactiveX文档中文翻译 —— Scheduler》「默认调度器」](https://mcxiaoke.gitbooks.io/rxdocs/content/Scheduler.html#默认调度器) 查看。

# 5. 使用示例

点击 [《ReactiveX文档中文翻译 —— Scheduler》「使用调度器」](https://mcxiaoke.gitbooks.io/rxdocs/content/Scheduler.html#使用调度器) 查看。

可能你会觉得示例有丢丢“奇怪”，在 [《RxJava 源码解析 —— Observable#subscribeOn(Scheduler)》](http://www.iocoder.cn/RxJava/observable-subscribe-on-scheduler/) 你将获得答案。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

本文偏介绍性，大量内容引用 [《ReactiveX文档中文翻译 —— Scheduler》](https://mcxiaoke.gitbooks.io/rxdocs/content/Scheduler.html) 。

后续根据需要，可能解析默认调度器的源码实现。

