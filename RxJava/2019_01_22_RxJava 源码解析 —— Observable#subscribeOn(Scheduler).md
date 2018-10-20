title: RxJava 源码解析 —— Observable#subscribeOn(Scheduler)
date: 2019-01-22
tags:
categories: RxJava
permalink: RxJava/observable-subscribe-on-scheduler

-------

摘要: 原创出处 http://www.iocoder.cn/RxJava/observable-subscribe-on-scheduler/ 「芋道源码」欢迎转载，保留摘要，谢谢！

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

`Observable#subscribeOn(Scheduler)` 方法，用途如下 ：

> FROM [《ReactiveX文档中文翻译 —— SubscribeOn》](https://mcxiaoke.gitbooks.io/rxdocs/content/operators/SubscribeOn.html)  
> 指定 Observable **自身**在哪个调度器上执行  
> ![](http://www.iocoder.cn/images/RxJava/2019_01_22/01.png)  
> 很多 ReactiveX 实现都使用调度器 `"Scheduler"` 来管理多线程环境中Observable 的转场。你可以使用 SubscribeOn 操作符指定 Observable 在一个特定的调度器上运转。

从概念上可能比较模糊，或者我们换一种说法 ：

> FROM [《给 Android 开发者的 RxJava 详解》「 3. 线程控制 —— Scheduler (一) 」](http://gank.io/post/560e15be2dca930e00da1083#toc_14)
> `#subscribeOn()` ：指定 `#subscribe()` 所发生的线程，即 `Observable.OnSubscribe` 被激活时所处的线程。**或者叫做事件产生的线程**。

来来来，一起瞅瞅源码，更加清理的理解。`Observable#subscribeOn(Scheduler)` 方法，代码如下 ：

```Java
// Observable.java

final OnSubscribe<T> onSubscribe;

protected Observable(OnSubscribe<T> f) {
    this.onSubscribe = f;
}

public final Observable<T> subscribeOn(Scheduler scheduler) {
    if (this instanceof ScalarSynchronousObservable) {
        return ((ScalarSynchronousObservable<T>)this).scalarScheduleOn(scheduler);
   }
    return create(new OperatorSubscribeOn<T>(this, scheduler));
}

public static <T> Observable<T> create(OnSubscribe<T> f) {
    return new Observable<T>(RxJavaHooks.onCreate(f));
}
```
* ScalarSynchronousObservable ，跳过，不在本文范围内。
* 创建 OperatorSubscribeOn 对象，将 Observable ( `this` ) 和 Scheduler ( `scheduler` ) 传入。

-------

OperatorSubscribeOn 类，代码如下 ：

```Java
  1: public final class OperatorSubscribeOn<T> implements OnSubscribe<T> {
  2: 
  3:     final Scheduler scheduler;
  4:     final Observable<T> source;
  5: 
  6:     public OperatorSubscribeOn(Observable<T> source, Scheduler scheduler) {
  7:         this.scheduler = scheduler;
  8:         this.source = source;
  9:     }
 10: 
 11:     @Override
 12:     public void call(final Subscriber<? super T> subscriber) {
 13:         final Worker inner = scheduler.createWorker();
 14:         subscriber.add(inner);
 15: 
 16:         inner.schedule(new Action0() {
 17:             @Override
 18:             public void call() {
 19:                 final Thread t = Thread.currentThread();
 20: 
 21:                 Subscriber<T> s = new Subscriber<T>(subscriber) {
 22:                     @Override
 23:                     public void onNext(T t) {
 24:                         subscriber.onNext(t);
 25:                     }
 26: 
 27:                     @Override
 28:                     public void onError(Throwable e) {
 29:                         try {
 30:                             subscriber.onError(e);
 31:                         } finally {
 32:                             inner.unsubscribe();
 33:                         }
 34:                     }
 35: 
 36:                     @Override
 37:                     public void onCompleted() {
 38:                         try {
 39:                             subscriber.onCompleted();
 40:                         } finally {
 41:                             inner.unsubscribe();
 42:                         }
 43:                     }
 44: 
 45:                     @Override
 46:                     public void setProducer(final Producer p) {
 47:                         subscriber.setProducer(new Producer() {
 48:                             @Override
 49:                             public void request(final long n) {
 50:                                 if (t == Thread.currentThread()) {
 51:                                     p.request(n);
 52:                                 } else {
 53:                                     inner.schedule(new Action0() {
 54:                                         @Override
 55:                                         public void call() {
 56:                                             p.request(n);
 57:                                         }
 58:                                     });
 59:                                 }
 60:                             }
 61:                         });
 62:                     }
 63:                 };
 64: 
 65:                 source.unsafeSubscribe(s);
 66:             }
 67:         });
 68:     }
 69: }
```

* `scheduler`，`source` 属性就不用说了，上文我们已经看到。
* 可能有同学对 `OnSubscribe#call(Subscriber)` 方法的调用链路不太熟悉，我们手撸一个实例，并且打个断点感受下 ：

    ```Java
    public class RxDemo11 {
    
        public static void main(String[] args) throws InterruptedException {
            Observable.just("1", "2")
                    .subscribeOn(Schedulers.newThread()) // Scheduler 开启新线程
                    .subscribe(s -> System.out.println(s)); // Subscriber 打印
            Thread.sleep(Long.MAX_VALUE); // Scheduler 异步，Sleep 等待
        }
    
    }
    ```
    * 在**第 13 行**处打断点，方法的调用链路如下图 ：![](http://www.iocoder.cn/images/RxJava/2019_01_22/02.png)  

* 第 13 行 ：使用 Scheduler 创建 Worker 。在 [《RxJava 源码解析 —— Scheduler》](http://www.iocoder.cn/RxJava/scheduler/?self) 有详细解析。 
* 第 14 行 ：将 Worker 添加到 `subscriber.subscriptions` 里。Worker 类实现了 `rx.Subscription` **接口**。
* 第 16 至 66 行 ：使用 Worker 执行操作。例如 Scheduler 为 NewThreadScheduler 时，此处的 Worker 对应 NewThreadWorker ，执行操作时使用**新线程**，而不是当前线程。
    * 第 19 行 ：获取执行操作的当前线程，用于第 50 行的判断。
    * 第 21 至 63 行 ：创建**新的** Subscriber 。比较关键的是 `#setProducer()` 方法，判断 `#request()` 时，线程是否是 `t` ( Worker 的线程 )，如果不是，重新使用 Worker 执行 `#request()` 方法。通过这样的方式，达到上文所说的 _"指定 `#subscribe()` 所发生的线程，即 `Observable.OnSubscribe` 被激活时所处的线程。**或者叫做事件产生的线程**。"_。
* 第 65 行 ：调用 `Observable#unsafeSubscribe(...)` 方法，**继续订阅逻辑**。
* 另外，想要触发**第 53 行**的情况，示例代码如下：

    ```Java
    public class RxDemo10 {
    
        public static void main(String[] args) throws InterruptedException {
            Observable.defer(() -> Observable.just("1", "2")
                        .subscribeOn(Schedulers.newThread()) // Scheduler
                    )
                    .subscribeOn(Schedulers.newThread()) // Scheduler
                    .subscribe(s -> System.out.println(s));
            Thread.sleep(Long.MAX_VALUE); // Scheduler 异步，Sleep 等待
        }
    
    }
    ```

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)


