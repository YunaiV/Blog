title: RxJava 源码解析 —— Observable#lift(Operator)
date: 2019-01-29
tags:
categories: RxJava
permalink: RxJava/observable-lift-operator

-------

摘要: 原创出处 http://www.iocoder.cn/RxJava/observable-subscribe-on-scheduler/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 RxJava 1.2.X 版本**  

本系列写作目的，为了辅助 Hystrix 的理解，因此会较为零散与琐碎，望见谅见谅。

- [1. 概述](http://www.iocoder.cn/RxJava/observable-lift-operator/)
- [2. 示例](http://www.iocoder.cn/RxJava/observable-lift-operator/)
- [3. 原理](http://www.iocoder.cn/RxJava/observable-lift-operator/)
- [4. 源码](http://www.iocoder.cn/RxJava/observable-lift-operator/)
  - [4.1 Operator](http://www.iocoder.cn/RxJava/observable-lift-operator/)
  - [4.2 Observable#lift(Operator)](http://www.iocoder.cn/RxJava/observable-lift-operator/)
  - [4.3 OnSubscribeLift](http://www.iocoder.cn/RxJava/observable-lift-operator/)
- [666. 彩蛋](http://www.iocoder.cn/RxJava/observable-lift-operator/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2018_05_18.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

# 1. 概述

`Observable#lift(Operator)` 方法，RxJava 中**所有**操作符的**基础**，不清晰的理解，后面会是懵逼脸状。

怎么解释这个方法呢？笔者暂时没有想好。搜了下互联网上相关的文章，也没找到合适的寿命。But ，这不影响我们对该方法的理解。

# 2. 示例

下面我们先一起来看**第一段示例** ：

```Java
public static void main(String[] args) {
   Observable.just("1", "2")
           .subscribe(o -> System.out.println("subscriber ：" + o));
}
```
* 输出如下 ：

    ```
    subscriber ：1
    subscriber ：2
    ```

**第二段示例** ：

```Java
public static void main(String[] args) {
   Observable.just("1", "2")
           .lift(new Observable.Operator<String, String>() {
               @Override
               public Subscriber<? super String> call(Subscriber<? super String> subscriber) {
                   return new Subscriber<String>() {
                       @Override
                       public void onCompleted() {

                       }

                       @Override
                       public void onError(Throwable e) {

                       }

                       @Override
                       public void onNext(String s) {
                           System.out.println("operator ：" + s);
                           subscriber.onNext(s);
                       }
                   };
               }
           })
           .subscribe(o -> System.out.println("subscriber ：" + o));
}
```

* 输出如下 ：

    ```Java
    operator ：1
    subscriber ：1
    operator ：2
    subscriber ：2
    ```
    
* `Observable#lift(Operator)` 方法，返回一个**新的** Subscriber ，对**传递进来**的 Subscriber 形成**代理**，对 `#onNext()` / `#onCompleted()` / `#onError()` 方法进行拦截。**这就是 `Observable#lift(Operator)` 的实现原理**。如果我们去掉 `subscriber.onNext(s)` 部分代码，则输出如下 ：

    ```Java
    operator ：1
    operator ：2
    ```

# 3. 原理

OK ，下面在用一张图来理解下 `Observable#lift(Operator)` 的**原理** ：

![](http://www.iocoder.cn/images/RxJava/2019_01_29/01.png)  


# 4. 源码

开始我们的源码旅程。

## 4.1 Operator

`rx.Observable.Operator` ，代码如下 ：

```Java
public interface Func1<T, R> extends Function {
    R call(T t);
}

public interface Operator<R, T> extends Func1<Subscriber<? super R>, Subscriber<? super T>> {
    // cover for generics insanity
}
```

* T `t` ：传递来的 Subscriber ，上图**粉色**的 Subscriber 。
* R ：返回一个**新**的 Subscriber ，上图**绿色**的 Subscriber 。


## 4.2 Observable#lift(Operator)

`Observable#lift(Operator)` 方法，代码如下 ：

```Java
public final <R> Observable<R> lift(final Operator<? extends R, ? super T> operator) {
    return create(new OnSubscribeLift<T, R>(onSubscribe, operator));
}

public static <T> Observable<T> create(OnSubscribe<T> f) {
    return new Observable<T>(RxJavaHooks.onCreate(f));
}
```

* 使用 `operator` ( Operator ) + `onSubscribe` ( OnSubscribe )，生成**新**的 `onSubscribe` ( OnSubscribe【OnSubscribeLift】 ) ，从而生成**新**的 Observable 。
* OnSubscribeLift 在 [「4.3 OnSubscribeLift」](#) 详细解析。

## 4.3 OnSubscribeLift

OnSubscribeLift ，代码如下 ：

```Java
  1: public final class OnSubscribeLift<T, R> implements OnSubscribe<R> {
  2: 
  3:     final OnSubscribe<T> parent;
  4: 
  5:     final Operator<? extends R, ? super T> operator;
  6: 
  7:     public OnSubscribeLift(OnSubscribe<T> parent, Operator<? extends R, ? super T> operator) {
  8:         this.parent = parent;
  9:         this.operator = operator;
 10:     }
 11: 
 12:     @Override
 13:     public void call(Subscriber<? super R> o) {
 14:         try {
 15:             Subscriber<? super T> st = RxJavaHooks.onObservableLift(operator).call(o);
 16:             try {
 17:                 // new Subscriber created and being subscribed with so 'onStart' it
 18:                 st.onStart();
 19:                 parent.call(st);
 20:             } catch (Throwable e) {
 21:                 // localized capture of errors rather than it skipping all operators
 22:                 // and ending up in the try/catch of the subscribe method which then
 23:                 // prevents onErrorResumeNext and other similar approaches to error handling
 24:                 Exceptions.throwIfFatal(e);
 25:                 st.onError(e);
 26:             }
 27:         } catch (Throwable e) {
 28:             Exceptions.throwIfFatal(e);
 29:             // if the lift function failed all we can do is pass the error to the final Subscriber
 30:             // as we don't have the operator available to us
 31:             o.onError(e);
 32:         }
 33:     }
 34: }
```

* 可能有胖友对 `Observable#subscribe(Subscriber)` 方法的**调用链**不是很熟悉，在**第 15 行**打**断点**，调用连如下图 ：![](http://www.iocoder.cn/images/RxJava/2019_01_29/02.png)  
* 第 15 行 ：调用 `Operator#call()` 方法，创建**新**的 Subscriber 。
* 第 18 行 ：调用 `Subscriber#onStart()` 方法，因为**新**的 Subscriber 被订阅。
* 第 19 行 ：调用 `OnSubscribe#call(Subscriber)` 方法，**继续订阅**。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

是不是非常清晰！聪慧如你！

胖友，分享个朋友圈可好？


