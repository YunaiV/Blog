title: LinkedList、ConcurrentLinkedQueue、LinkedBlockingQueue 对比分析
date: 2018-06-29
tags:
categories: 数据结构与算法
permalink: Data-Structures-and-Algorithms/Comparison-and-analysis-of-LinkedList-and-ConcurrentLinkedQueue-and-LinkedBlockingQueue
author: mantu
from_url: https://www.cnblogs.com/mantu/p/5802393.html
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484683&idx=2&sn=640e7e3e4d16a0d9046dc1e16cc117ea&chksm=fa497abacd3ef3ac44f1376672250de0a1d13941daeac217cb9fced3d5089d8f4b4d3edee1ab#rd

-------

摘要: 原创出处 https://www.cnblogs.com/mantu/p/5802393.html 「mantu」欢迎转载，保留摘要，谢谢！

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

​      写这篇文章源于我经历过的一次生产事故，在某家公司的时候，有个服务会收集业务系统的日志，此服务的开发人员在给业务系统的sdk中就因为使用了LinkedList，又没有做并发控制，就造成了此服务经常不能正常收集到业务系统的日志(丢日志以及日志上报的线程停止运行)。看一下add()方法的源码，我们就可以知道原因了：

​    demo 　Lesson2LinkedListThreads 展示了在多线程且没有做并发控制的环境下，size的值远远大于了队列的实际值，100个线程，每个添加1000个元素，最后实际只加进去2030个元素：

​         List的变量size值为：88371
         第2031个元素取出为null

​    解决方案，使用锁或者使用ConcurrentLinkedQueue、LinkedBlockingQueue等支持添加元素为原子操作的队列。

​    上一节我们已经分析过LinkedBlockingQueue的put等方法的源码，是使用ReentrantLock来实现的添加元素原子操作。我们再简单看一下高并发queue的add和offer(）方法，方法中使用了CAS来实现的无锁的原子操作：


```Java
public boolean add(E e) {
　　　　   return offer(e);
　　   }

public boolean offer(E e) {
        checkNotNull(e);
        final Node<E> newNode = new Node<E>(e);

        for (Node<E> t = tail, p = t;;) {
            Node<E> q = p.next;
            if (q == null) {
                // p is last node
                if (p.casNext(null, newNode)) {
                    // Successful CAS is the linearization point
                    // for e to become an element of this queue,
                    // and for newNode to become "live".
                    if (p != t) // hop two nodes at a time
                        casTail(t, newNode);  // Failure is OK.
                    return true;
                }
                // Lost CAS race to another thread; re-read next
            }
            else if (p == q)
                // We have fallen off list.  If tail is unchanged, it
                // will also be off-list, in which case we need to
                // jump to head, from which all live nodes are always
                // reachable.  Else the new tail is a better bet.
                p = (t != (t = tail)) ? t : head;
            else
                // Check for tail updates after two hops.
                p = (p != t && t != (t = tail)) ? t : q;
        }
    }
```

　　接下来，我们再利用高并发queue对上面的demo进行改造，大家只要改变demo中的内容，讲下面两行的注释内容颠倒，即可发现没有丢失任何的元素：

​      public static LinkedList list = new LinkedList();
      //public static ConcurrentLinkedQueue list = new ConcurrentLinkedQueue();

​     再看一下高性能queue的poll()方法，才觉得NB，取元素的方法也用CAS实现了原子操作，因此在实际使用的过程中，当我们在不那么在意元素处理顺序的情况下，队列元素的消费者，完全可以是多个，不会丢任何数据：



```java
    public E poll() {
        restartFromHead:
        for (;;) {
            for (Node<E> h = head, p = h, q;;) {
                E item = p.item;

                if (item != null && p.casItem(item, null)) {
                    // Successful CAS is the linearization point
                    // for item to be removed from this queue.
                    if (p != h) // hop two nodes at a time
                        updateHead(h, ((q = p.next) != null) ? q : p);
                    return item;
                }
                else if ((q = p.next) == null) {
                    updateHead(h, p);
                    return null;
                }
                else if (p == q)
                    continue restartFromHead;
                else
                    p = q;
            }
        }
    }
```



​    关于ConcurrentLinkedQueue和LinkedBlockingQueue：

​    1.LinkedBlockingQueue是使用锁机制，ConcurrentLinkedQueue是使用CAS算法，虽然LinkedBlockingQueue的底层获取锁也是使用的CAS算法

​    2.关于取元素，ConcurrentLinkedQueue不支持阻塞去取元素，LinkedBlockingQueue支持阻塞的take()方法，如若大家需要ConcurrentLinkedQueue的消费者产生阻塞效果，需要自行实现

​    3.关于插入元素的性能，从字面上和代码简单的分析来看ConcurrentLinkedQueue肯定是最快的，但是这个也要看具体的测试场景，我做了两个简单的demo做测试，测试的结果如下，两个的性能差不多，但在实际的使用过程中，尤其在多cpu的服务器上，有锁和无锁的差距便体现出来了，ConcurrentLinkedQueue会比LinkedBlockingQueue快很多：

demo Lesson2ConcurrentLinkedQueuePerform:在使用ConcurrentLinkedQueue的情况下100个线程循环增加的元素数为：33828193

demo Lesson2LinkedBlockingQueuePerform:在使用LinkedBlockingQueue的情况下100个线程循环增加的元素数为：33827382

# 666. 彩蛋

如果你对 Dubbo 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)