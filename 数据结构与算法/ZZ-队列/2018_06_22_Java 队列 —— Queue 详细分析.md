title: Java 队列 —— Queue 详细分析
date: 2018-06-22
tags:
categories: 数据结构与算法
permalink: Data-Structures-and-Algorithms/How-does-front-end-API-interaction-ensure-data-security
author: 低调人生
from_url: https://www.cnblogs.com/lemon-flm/p/7877898.html
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484647&idx=2&sn=6db84dfe271090a75145cac9845d6a0c&chksm=fa497b56cd3ef240a58c3fe83707e77af06273631e932508a61b12108d46b8a4715e7a4e2037#rd

-------

摘要: 原创出处 https://www.cnblogs.com/lemon-flm/p/7877898.html 「低调人生」欢迎转载，保留摘要，谢谢！

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

**Queue： 基本上，一个队列就是一个先入先出（FIFO）的数据结构**

**Queue接口与List、Set同一级别，都是继承了Collection接口。LinkedList实现了Deque接 口。**

**Queue的实现**

**1、没有实现的阻塞接口的LinkedList： 实现了java.util.Queue接口和java.util.AbstractQueue接口**
　　内置的不阻塞队列： PriorityQueue 和 ConcurrentLinkedQueue
　　PriorityQueue 和 ConcurrentLinkedQueue 类在 Collection Framework 中加入两个具体集合实现。
　　PriorityQueue 类实质上维护了一个有序列表。加入到 Queue 中的元素根据它们的天然排序（通过其 java.util.Comparable 实现）或者根据传递给构造函数的 java.util.Comparator 实现来定位。
　　ConcurrentLinkedQueue 是基于链接节点的、线程安全的队列。并发访问不需要同步。因为它在队列的尾部添加元素并从头部删除它们，所以只要不需要知道队列的大 小，　　　　    　　ConcurrentLinkedQueue 对公共集合的共享访问就可以工作得很好。收集关于队列大小的信息会很慢，需要遍历队列。

**2)实现阻塞接口的：**
　　java.util.concurrent 中加入了 BlockingQueue 接口和五个阻塞队列类。它实质上就是一种带有一点扭曲的 FIFO 数据结构。不是立即从队列中添加或者删除元素，线程执行操作阻塞，直到有空间或者元素可用。
五个队列所提供的各有不同：
　　* ArrayBlockingQueue ：一个由数组支持的有界队列。
　　* LinkedBlockingQueue ：一个由链接节点支持的可选有界队列。
　　* PriorityBlockingQueue ：一个由优先级堆支持的无界优先级队列。
　　* DelayQueue ：一个由优先级堆支持的、基于时间的调度队列。
　　* SynchronousQueue ：一个利用 BlockingQueue 接口的简单聚集（rendezvous）机制。

![img](https://images2017.cnblogs.com/blog/1182892/201711/1182892-20171122100317930-842768608.png)

下表显示了jdk1.5中的阻塞队列的操作：



　　**add**        增加一个元索                     如果队列已满，则抛出一个IIIegaISlabEepeplian异常
　　**remove**   移除并返回队列头部的元素    如果队列为空，则抛出一个NoSuchElementException异常
　　**element**  返回队列头部的元素             如果队列为空，则抛出一个NoSuchElementException异常
　　**offer**       添加一个元素并返回true       如果队列已满，则返回false
　　**poll**         移除并返问队列头部的元素    如果队列为空，则返回null
　　**peek**       返回队列头部的元素             如果队列为空，则返回null
　　**put**         添加一个元素                      如果队列满，则阻塞
　　**take**        移除并返回队列头部的元素     如果队列为空，则阻塞



**remove、element、offer 、poll、peek 其实是属于Queue接口。**



阻塞队列的操作可以根据它们的响应方式分为以下三类：aad、removee和element操作在你试图为一个已满的队列增加元素或从空队列取得元素时 抛出异常。当然，在多线程程序中，队列在任何时间都可能变成满的或空的，所以你可能想使用offer、poll、peek方法。这些方法在无法完成任务时 只是给出一个出错示而不会抛出异常。



注意：poll和peek方法出错进返回null。因此，向队列中插入null值是不合法的



最后，我们有阻塞操作put和take。put方法在队列满时阻塞，take方法在队列空时阻塞。





**LinkedBlockingQueue**的容量是没有上限的（说的不准确，在不指定时容量为Integer.MAX_VALUE，不要然的话在put时怎么会受阻呢），但是也可以选择指定其最大容量，它是基于链表的队列，此队列按 FIFO（先进先出）排序元素。

**ArrayBlockingQueue**在构造时需要指定容量， 并可以选择是否需要公平性，如果公平参数被设置true，等待时间最长的线程会优先得到处理（其实就是通过将ReentrantLock设置为true来 达到这种公平性的：即等待时间最长的线程会先操作）。通常，公平性会使你在性能上付出代价，只有在的确非常需要的时候再使用它。它是基于数组的阻塞循环队 列，此队列按 FIFO（先进先出）原则对元素进行排序。

**PriorityBlockingQueue**是一个带优先级的 队列，而不是先进先出队列。元素按优先级顺序被移除，该队列也没有上限（看了一下源码，PriorityBlockingQueue是对 PriorityQueue的再次包装，是基于堆数据结构的，而PriorityQueue是没有容量限制的，与ArrayList一样，所以在优先阻塞 队列上put时是不会受阻的。虽然此队列逻辑上是无界的，但是由于资源被耗尽，所以试图执行添加操作可能会导致 OutOfMemoryError），但是如果队列为空，那么取元素的操作take就会阻塞，所以它的检索操作take是受阻的。另外，往入该队列中的元 素要具有比较能力。

**DelayQueue**（基于PriorityQueue来实现的）是一个存放Delayed 元素的无界阻塞队列，只有在延迟期满时才能从中提取元素。该队列的头部是延迟期满后保存时间最长的 Delayed 元素。如果延迟都还没有期满，则队列没有头部，并且poll将返回null。当一个元素的 getDelay(TimeUnit.NANOSECONDS) 方法返回一个小于或等于零的值时，则出现期满，poll就以移除这个元素了。此队列不允许使用 null 元素。



![img](https://images2017.cnblogs.com/blog/1182892/201711/1182892-20171122101159430-391005054.jpg)

 **一个例子：**



```java
package com.yao;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
public class BlockingQueueTest {
 /**
 定义装苹果的篮子
  */
 public static class Basket{
  // 篮子，能够容纳3个苹果
  BlockingQueue<String> basket = new ArrayBlockingQueue<String>(3);

  // 生产苹果，放入篮子
  public void produce() throws InterruptedException{
   // put方法放入一个苹果，若basket满了，等到basket有位置
   basket.put("An apple");
  }
  // 消费苹果，从篮子中取走
  public String consume() throws InterruptedException{
   // get方法取出一个苹果，若basket为空，等到basket有苹果为止
   String apple = basket.take();
   return apple;
  }

  public int getAppleNumber(){
   return basket.size();
  }

 }
 //　测试方法
 public static void testBasket() {
  // 建立一个装苹果的篮子
  final Basket basket = new Basket();
  // 定义苹果生产者
  class Producer implements Runnable {
   public void run() {
    try {
     while (true) {
      // 生产苹果
      System.out.println("生产者准备生产苹果："
        + System.currentTimeMillis());
      basket.produce();
      System.out.println("生产者生产苹果完毕："
        + System.currentTimeMillis());
      System.out.println("生产完后有苹果："+basket.getAppleNumber()+"个");
      // 休眠300ms
      Thread.sleep(300);
     }
    } catch (InterruptedException ex) {
    }
   }
  }
  // 定义苹果消费者
  class Consumer implements Runnable {
   public void run() {
    try {
     while (true) {
      // 消费苹果
      System.out.println("消费者准备消费苹果："
        + System.currentTimeMillis());
      basket.consume();
      System.out.println("消费者消费苹果完毕："
        + System.currentTimeMillis());
      System.out.println("消费完后有苹果："+basket.getAppleNumber()+"个");
      // 休眠1000ms
      Thread.sleep(1000);
     }
    } catch (InterruptedException ex) {
    }
   }
  }

  ExecutorService service = Executors.newCachedThreadPool();
  Producer producer = new Producer();
  Consumer consumer = new Consumer();
  service.submit(producer);
  service.submit(consumer);
  // 程序运行10s后，所有任务停止
  try {
   Thread.sleep(10000);
  } catch (InterruptedException e) {
  }
  service.shutdownNow();
 }
 public static void main(String[] args) {
  BlockingQueueTest.testBasket();
 }
}
```

# 666. 彩蛋

如果你对 Dubbo 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)