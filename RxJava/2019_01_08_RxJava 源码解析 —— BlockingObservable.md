title: RxJava 源码解析 —— BlockingObservable
date: 2019-01-08
tags:
categories: RxJava
permalink: RxJava/blocking-observable

-------

摘要: 原创出处 http://www.iocoder.cn/RxJava/blocking-observable/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 RxJava 1.2.X 版本**  

本系列写作目的，为了辅助 Hystrix 的理解，因此会较为零散与琐碎，望见谅见谅。

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2018_05_18.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

> [《ReactiveX/RxJava文档中文版 —— 阻塞操作》](https://mcxiaoke.gitbooks.io/rxdocs/content/operators/Blocking-Observable-Operators.html)  
> BlockingObservable 的方法不是将一个 Observable 变换为另一个，也不是过滤Observables，它们会打断 Observable 的调用链，会阻塞等待直到 Observable 发射了想要的数据，然后返回这个数据（而不是一个 Observable ）。

# 1. toBlocking

调用 `Observable#toBlocking()` 或 `BlockingObservable#from(Observable)` 方法，将 Observable 转换成 BlockingObservable 。代码如下：

```Java
// BlockingObservable.java
public final class BlockingObservable<T> {

    // ... 省略无关代码

    private final Observable<? extends T> o;
    
    private BlockingObservable(Observable<? extends T> o) {
        this.o = o;
    }

    public static <T> BlockingObservable<T> from(final Observable<? extends T> o) {
        return new BlockingObservable<T>(o);
    }

}

// Observable.java
public class Observable<T> {

    // ... 省略无关代码

    public final BlockingObservable<T> toBlocking() {
        return BlockingObservable.from(this);
    }

}
```
* 从代码上我们可以看到，BlockingObservable 并未将 Observable 转换成新的，而是简单的包了一层。

# 2. toFuture

> [《ReactiveX/RxJava文档中文版 —— TO》](https://mcxiaoke.gitbooks.io/rxdocs/content/operators/To.html#tofuture)  
> ![](http://www.iocoder.cn/images/Hystrix/2018_10_08/04.png)
> `#toFuture()` 操作符也是只能用于 BlockingObservable 。这个操作符将Observable 转换为一个返回**单个数据项**的 Future 。
> 
> * 如果原始 Observable 发射多个数据项，Future会收到一个IllegalArgumentException；
> * 如果原始 Observable 没有发射任何数据，Future会收到一个NoSuchElementException。
>
> 如果你想将发射多个数据项的 Observable 转换为 Future ，可以这样用：`myObservable.toList().toBlocking().toFuture()` 。

点击[链接](https://github.com/ReactiveX/RxJava/blob/396b6104e419b80002c45faf76ac38f00d2ff64a/src/main/java/rx/internal/operators/BlockingOperatorToFuture.java) 查看 `#toFuture()` 的代码实现：

* 通过向传入 Observable 订阅 Subscriber ，打断 Observable 的调用链，会阻塞等待直到 Observable 发射了想要的数据。
    * `#onNext()` 方法，设置执行的返回值( `value` )。
    * `#onCompleted()` 方法，CountDownLatch (`finished`) 减一。
    * `#onError()` 方法，设置执行时发生的异常( `error` )，并 CountDownLatch (`finished`) 减一。
* 返回的 Future ，通过 CountDownLatch ( `error` ) 判断是否执行完成；通过 `value` ， `error` 获得执行的结果。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

