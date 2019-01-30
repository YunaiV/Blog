title: Java 线程的 wait 和 notify 的神坑
date: 2019-01-31
tags:
categories: 精进
permalink: Fight/Java-threads-for-wait-and-notify-for-sinkholes
author: 忘净空
from_url: https://www.jianshu.com/p/91d95bb5a4bd
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247486113&idx=2&sn=47a495e0dc67fdb6c28c5779a557301c&chksm=fa497510cd3efc06c68237fdb9a687ae5c5978e220a9499eabaf47f6ea62e9028402c0baea19&token=1524868883&lang=zh_CN#rd

-------

摘要: 原创出处 https://www.jianshu.com/p/91d95bb5a4bd 「忘净空」欢迎转载，保留摘要，谢谢！

- [问题一：通知丢失](http://www.iocoder.cn/Fight/Java-threads-for-wait-and-notify-for-sinkholes/)
  - [问题分析](http://www.iocoder.cn/Fight/Java-threads-for-wait-and-notify-for-sinkholes/)
- [问题二：假唤醒](http://www.iocoder.cn/Fight/Java-threads-for-wait-and-notify-for-sinkholes/)
- [等待/通知的典型范式](http://www.iocoder.cn/Fight/Java-threads-for-wait-and-notify-for-sinkholes/)
  - [等待方遵循原则](http://www.iocoder.cn/Fight/Java-threads-for-wait-and-notify-for-sinkholes/)
  - [通知方遵循原则](http://www.iocoder.cn/Fight/Java-threads-for-wait-and-notify-for-sinkholes/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

也许我们只知道wait和notify是实现线程通信的，同时要使用synchronized包住，其实在开发中知道这个是远远不够的。接下来看看两个常见的问题。

# 问题一：通知丢失

创建2个线程，一个线程负责计算，一个线程负责获取计算结果。

```Java
public class Calculator extends Thread {
    int total;

    @Override
    public void run() {
        synchronized (this){
            for(int i = 0; i < 101; i++){
                total += i;
            }
            this.notify();
        }

    }
}

public class ReaderResult extends Thread {
    Calculator c;
    public ReaderResult(Calculator c) {
        this.c = c;
    }

    @Override
    public void run() {
        synchronized (c) {
            try {
                System.out.println(Thread.currentThread() + "等待计算结...");
                c.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread() + "计算结果为:" + c.total);
        }
    }


    public static void main(String[] args) {
        Calculator calculator = new Calculator();
        //先启动获取计算结果线程
        new ReaderResult(calculator).start();
        calculator.start();

    }
}

我们会获得预期的结果：
Thread[Thread-1,5,main]等待计算结...
Thread[Thread-1,5,main]计算结果为:5050

但是我们修改为先启动计算线程呢？
calculator.start();
new ReaderResult(calculator).start();

这是获取结算结果线程一直等待：
Thread[Thread-1,5,main]等待计算结...
```

## 问题分析

打印出线程堆栈：

```Java
"Thread-1" prio=5 tid=0x00007f983b87e000 nid=0x4d03 in Object.wait() [0x0000000118988000]
   java.lang.Thread.State: WAITING (on object monitor)
    at java.lang.Object.wait(Native Method)
    - waiting on <0x00000007d56fb4d0> (a com.concurrent.waitnotify.Calculator)
    at java.lang.Object.wait(Object.java:503)
    at com.concurrent.waitnotify.ReaderResult.run(ReaderResult.java:18)
    - locked <0x00000007d56fb4d0> (a com.concurrent.waitnotify.Calculator)
```

可以看出ReaderResult在Calculator上等待。发生这个现象就是常说的通知丢失，在获取通知前，通知提前到达，我们先计算结果，计算完后再通知，但是这个时候获取结果没有在等待通知，等到获取结果的线程想获取结果时，这个通知已经通知过了，所以就发生丢失，那我们该如何避免?可以设置变量表示是否被通知过，修改代码如下：

```Java
public class Calculator extends Thread {
    int total;
    boolean isSignalled = false;

    @Override
    public void run() {
        synchronized (this) {
            isSignalled = true;//已经通知过
                for (int i = 0; i < 101; i++) {
                    total += i;
                }
                this.notify();
            }
    }
}

public class ReaderResult extends Thread {

    Calculator c;

    public ReaderResult(Calculator c) {
        this.c = c;
    }

    @Override
    public void run() {
        synchronized (c) {
            if (!c.isSignalled) {//判断是否被通知过
                try {
                    System.out.println(Thread.currentThread() + "等待计算结...");
                    c.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread() + "计算结果为:" + c.total);
            }

        }
    }

    public static void main(String[] args) {
        Calculator calculator = new Calculator();
        new ReaderResult(calculator).start();
        calculator.start();
    }
}
```

# 问题二：假唤醒

两个线程去删除数组的元素，当没有元素的时候等待，另一个线程添加一个元素，添加完后通知删除数据的线程。

```Java
public class EarlyNotify{
    private List list;

    public EarlyNotify() {
        list = Collections.synchronizedList(new LinkedList());
    }

    public String removeItem() throws InterruptedException {

        synchronized ( list ) {
            if ( list.isEmpty() ) {  //问题在这
                list.wait();
            }

            //删除元素
            String item = (String) list.remove(0);
            return item;
        }
    }

    public void addItem(String item) {
        synchronized ( list ) {
            //添加元素
            list.add(item);
            //添加后，通知所有线程
            list.notifyAll();
        }
    }

    private static void print(String msg) {
        String name = Thread.currentThread().getName();
        System.out.println(name + ": " + msg);
    }

    public static void main(String[] args) {
        final EarlyNotify en = new EarlyNotify();

        Runnable runA = new Runnable() {
            public void run() {
                try {
                    String item = en.removeItem();

                } catch ( InterruptedException ix ) {
                    print("interrupted!");
                } catch ( Exception x ) {
                    print("threw an Exception!!!\n" + x);
                }
            }
        };

        Runnable runB = new Runnable() {
            public void run() {
                en.addItem("Hello!");
            }
        };

        try {
            //启动第一个删除元素的线程
            Thread threadA1 = new Thread(runA, "threadA1");
            threadA1.start();

            Thread.sleep(500);

            //启动第二个删除元素的线程
            Thread threadA2 = new Thread(runA, "threadA2");
            threadA2.start();

            Thread.sleep(500);
            //启动增加元素的线程
            Thread threadB = new Thread(runB, "threadB");
            threadB.start();

            Thread.sleep(1000); // wait 10 seconds

            threadA1.interrupt();
            threadA2.interrupt();
        } catch ( InterruptedException x ) {}
    }
}

结果：
threadA1: threw an Exception!!!
java.lang.IndexOutOfBoundsException: Index: 0, Size: 0
```

这里发生了假唤醒，当添加完一个元素然后唤醒两个线程去删除，这个只有一个元素，所以会抛出数组越界，这时我们需要唤醒的时候在判断一次是否还有元素。
 修改代码：

```Java
  public String removeItem() throws InterruptedException {

        synchronized ( list ) {
            while ( list.isEmpty() ) {  //问题在这
                list.wait();
            }

            //删除元素
            String item = (String) list.remove(0);
            return item;
        }
    }
```

# 等待/通知的典型范式

从上面的问题我们可归纳出等待/通知的典型范式。该范式分为两部分，分别针对等待方（消费者）和通知方（生产者）。

## 等待方遵循原则如下：

1. 获取对象的锁
2. 如果条件不满足，那么调用对象的wait()方法，被通知后仍要检查条件
3. 条件满足则执行对应的逻辑

对应伪代码如下：

```Java
synchronized(对象){
    while(条件不满足){
        对象.wait();
    }
    对应的处理逻辑
}
```

## 通知方遵循原则如下：

1. 获得对象的锁
2. 改变条件
3. 通知所以等待在对象上的线程

对应伪代码如下：

```Java
synchronized(对象){
    改变条件
    对象.notifyAll();
}
```