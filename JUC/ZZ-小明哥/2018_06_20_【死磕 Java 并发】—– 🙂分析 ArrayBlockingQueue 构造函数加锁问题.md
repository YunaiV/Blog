title: 【死磕 Java 并发】—– 分析 ArrayBlockingQueue 构造函数加锁问题
date: 2018-06-20
tag:
categories: JUC
permalink: JUC/sike/ArrayBlockingQueue-construction-lock
author: 小明哥
from_url: http://cmsblogs.com/?p=2458
wechat_url:

-------

摘要: 原创出处 http://cmsblogs.com/?p=2458 「小明哥」欢迎转载，保留摘要，谢谢！

作为「小明哥」的忠实读者，「老艿艿」略作修改，记录在理解过程中，参考的资料。

- [1. 问题](http://www.iocoder.cn/JUC/sike/ArrayBlockingQueue-construction-lock/)
- [2. 解释一](http://www.iocoder.cn/JUC/sike/ArrayBlockingQueue-construction-lock/)
- [3. 解释二](http://www.iocoder.cn/JUC/sike/ArrayBlockingQueue-construction-lock/)
- [666. 彩蛋](http://www.iocoder.cn/JUC/sike/ArrayBlockingQueue-construction-lock/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

# 1. 问题

昨天有位小伙伴问我一个 ArrayBlockingQueue 中的一个构造函数为何需要加锁，其实这个问题我还真没有注意过。主要是在看 ArrayBlockingQueue 源码时，觉得它很简单，不就是通过加锁的方式来操作一个数组 items 么，有什么难的，所以就没有关注这个问题，所以它一问我懵逼了。回家细想了下，所以产生这篇博客。

# 2. 解释一

我们先看构造方法：

```Java
public ArrayBlockingQueue(int capacity, boolean fair, Collection<? extends E> c) {
    this(capacity, fair);

    final ReentrantLock lock = this.lock;
    lock.lock(); // Lock only for visibility, not mutual exclusion    //----(1)
    try {
        int i = 0;
        try {
            for (E e : c) {
                checkNotNull(e);
                items[i++] = e;
            }
        } catch (ArrayIndexOutOfBoundsException ex) {
            throw new IllegalArgumentException();
        }
        count = i;
        putIndex = (i == capacity) ? 0 : i;
    } finally {
        lock.unlock();
    }
}
```

第五行代码获取互斥锁，解释为**锁的目的不是为了互斥，而是为了保证可见性**。保证可见性？保证哪个可见性？我们知道 ArrayBlockingQueue 操作的其实就是一个 `items` 数组，这个数组是不具备线程安全的，所以保证可见性就是保证 `items` 的可见性。如果不加锁为什么就没法保证 `items` 的可见性呢？这其实是指令重排序的问题。

什么是指令重排序？编译器或运行时环境为了优化程序性能而采取的对指令进行重新排序执行的一种手段。也就是说程序运行的顺序与我们所想的顺序是不一致的。虽然它遵循 as-if-serial 语义，但是还是无法保证多线程环境下的数据安全。更多请参考博客 [《【死磕 Java 并发】—– Java 内存模型之重排序》](http://www.iocoder.cn/JUC/sike/jmm-1) 。

为什么说指令重排序会影响 `items` 的可见性呢？创建一个对象要分为三个步骤：

1. 分配内存空间
2. 初始化对象
3. 将内存空间的地址赋值给对应的引用

但是由于指令重排序的问题，步骤 2 和步骤 3 是可能发生重排序的，如下：

1. 分配内存空间
2. 将内存空间的地址赋值给对应的引用
3. 初始化对象

**这个过程就会对上面产生影响**。假如我们两个线程：

* 线程 A，负责 ArrayBlockingQueue 的实例化工作
* 线程 B，负责入队、出队操作。

线程 A 优先执行。当它执行上述构造方法的第 2 行代码，也就是 `this(capacity, fair);` ，则会调用如下构造方法，代码如下：

```Java
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    this.items = new Object[capacity];
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
}
```

这个时候 `items` 是已经完成了初始化工作的，也就是说我们可以对其进行操作了。如果在线程 A 实例化对象过程中，步骤 2 和步骤 3 重排序了，那么对于线程 B 而言，ArrayBlockingQueue 是已经完成初始化工作了也就是可以使用了。其实线程 A 可能还正在执行构造函数中的某一个行代码。两个线程在不加锁的情况对一个不具备线程安全的数组同时操作，很有可能会引发线程安全问题。

# 3. 解释二

还有一种解释：缓存一致性。为了解决 CPU 处理速度以及读写主存速度不一致的问题，引入了 CPU 高速缓存。虽然解决了速度问题，但是也带来了缓存一致性的问题。在不加锁的前提下，线程 A 在构造函数中 `items` 进行操作，线程 B 通过入队、出队的方式对 `items` 进行操作，这个过程对 `items` 的操作结果有可能只存在各自线程的缓存中，并没有写入主存，这样肯定会造成数据不一致的情况。

# 666. 彩蛋

> 小明哥：以上仅为个人理解，如果不正确之处，望指出！
> 
> 老艿艿：支持第二种。因为 `items` 是 `final` 修饰，禁止重排序。

如果你对 Java 并发感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

