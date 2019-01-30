title: 【死磕 Java 并发】—– Java 内存模型之从 JMM 角度分析 DCL
date: 2018-05-21
tags:
categories: JUC
permalink: JUC/sike/jmm-3-dcl
author: 小明哥
from_url: http://cmsblogs.com/?p=2161
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247483793&idx=1&sn=40ffc5ed384b6d7beacf87a44e30c8c3&chksm=fa497e20cd3ef736ae174e9a9698e5d4ff04dbfe68dc56c636584dcbd4952242ecc4476b8956#rd

-------

摘要: 原创出处 http://cmsblogs.com/?p=2161 「小明哥」欢迎转载，保留摘要，谢谢！

作为「小明哥」的忠实读者，「老艿艿」略作修改，记录在理解过程中，参考的资料。

- [1. 问题分析](http://www.iocoder.cn/JUC/sike/jmm-3-dcl/)
- [2. 解决方案](http://www.iocoder.cn/JUC/sike/jmm-3-dcl/)
  - [2.1 基于 volatile 解决方案](http://www.iocoder.cn/JUC/sike/jmm-3-dcl/)
  - [2.2 基于类初始化的解决方案](http://www.iocoder.cn/JUC/sike/jmm-3-dcl/)
- [3. 总结](http://www.iocoder.cn/JUC/sike/jmm-3-dcl/)
- [参考资料](http://www.iocoder.cn/JUC/sike/jmm-3-dcl/)
- [666. 彩蛋](http://www.iocoder.cn/JUC/sike/jmm-3-dcl/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

DCL ，即 Double Check Lock ，中文称为“**双重检查锁定**”。

其实 DCL 很多人在单例模式中用过，LZ 面试人的时候也要他们写过，但是有很多人都会写错。他们为什么会写错呢？其错误根源在哪里？有什么解决方案？下面就随 LZ 一起来分析。

# 1. 问题分析

我们先看单例模式里面的**懒汉式**：

```Java
public class Singleton {
    
    private static Singleton singleton;

    private Singleton(){}

    public static Singleton getInstance(){
        if (singleton == null) {
            singleton = new Singleton();
        }

        return singleton;
    }
    
}
```

我们都知道这种写法是**错误的**，因为**它无法保证线程的安全性**。优化如下：

```Java
public class Singleton {

    private static Singleton singleton;

    private Singleton(){}

    public static synchronized Singleton getInstance() {
        if (singleton == null) {
            singleton = new Singleton();
        }

        return singleton;
    }
    
}
```

优化非常简单，就是在 `#getInstance()` 方法上面做了**同步**，但是 `synchronized` 就会导致这个方法比较低效，导致程序性能下降，那么怎么解决呢？聪明的人们想到了双重检查  DCL：

```Java
public class Singleton {

    private static Singleton singleton;

    private Singleton() {}

    public static Singleton getInstance(){
        if(singleton == null){                              // 1
            synchronized (Singleton.class){                 // 2
                if(singleton == null){                      // 3
                    singleton = new Singleton();            // 4
                }
            }
        }
        return singleton;
    }
}
```

就如上面所示，这个代码看起来很完美，理由如下：

* 如果检查第一个 `singleton` 不为 null ，则不需要执行下面的加锁动作，极大提高了程序的性能。
* 如果第一个 `singleton` 为 null ，即使有多个线程同一时间判断，但是由于 `synchronized` 的存在，只会有一个线程能够创建对象。
* 当第一个获取锁的线程创建完成后 `singleton` 对象后，其他的在第二次判断 `singleton` 一定不会为 null ，则直接返回已经创建好的 `singleton` 对象。

通过上面的分析，DCL 看起确实是非常完美，但是可以明确地告诉你，这个**错误的**。上面的逻辑确实是没有问题，分析也对，但是就是有问题，那么问题出在哪里呢？在回答这个问题之前，我们先来复习一下创建对象过程，实例化一个对象要分为三个步骤：

```Java
memory = allocate();   //1：分配内存空间
ctorInstance(memory);  //2：初始化对象
instance = memory;     //3：将内存空间的地址赋值给对应的引用
```

但是由于**重排序**的原因，步骤 2、3 可能会发生重排序，其过程如下：

```Java
memory = allocate();   // 1：分配内存空间
instance = memory;     // 3：将内存空间的地址赋值给对应的引用
                                    // 😈 注意，此时对象还没有被初始化！
ctorInstance(memory);  // 2：初始化对象
```

如果 2、3 发生了重排序，就会导致第二个判断会出错，`singleton != null`，但是它其实仅仅只是一个地址而已，此时对象还没有被初始化，所以 `return` 的 `singleton` 对象是一个没有被初始化的对象，如下：

![DCL00001_2](https://gitee.com/chenssy/blog-home/raw/master/image/sijava/201812086001.png)

按照上面图例所示，线程 B 访问的是一个没有被初始化的 `singleton` 对象。

通过上面的阐述，我们可以判断 DCL 的错误根源在于**步骤 4**：

```Java
singleton = new Singleton();
```

知道问题根源所在，那么怎么解决呢？有两个解决办法：

1. 不允许初始化阶段步骤 2、3 发生重排序。
2. 允许初始化阶段步骤 2、3 发生重排序，但是不允许其他线程“看到”这个重排序。

# 2. 解决方案

解决方案依据上面两个解决办法即可。

## 2.1 基于 volatile 解决方案

对于上面的DCL其实只需要做一点点修改即可：将变量singleton生命为volatile即可：

```Java
public class Singleton {

    // 通过volatile关键字来确保安全
    private volatile static Singleton singleton;

    private Singleton(){}

    public static Singleton getInstance(){
        if(singleton == null){
            synchronized (Singleton.class){
                if(singleton == null){
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```

当 `singleton` 声明为 `volatile`后，步骤 2、3 就不会被重排序了，也就可以解决上面那问题了。

## 2.2 基于类初始化的解决方案

该解决方案的根本就在于：**利用 ClassLoder 的机制，保证初始化 instance 时只有一个线程。JVM 在类初始化阶段会获取一个锁，这个锁可以同步多个线程对同一个类的初始化**。

```Java
public class Singleton {

    private static class SingletonHolder{
        public static Singleton singleton = new Singleton();
    }

    public static Singleton getInstance(){
        return SingletonHolder.singleton;
    }
}
```

这种解决方案的实质是：运行步骤 2 和步骤 3 重排序，但是不允许其他线程看见。

> Java 语言规定，对于每一个类或者接口 C ，都有一个唯一的初始化锁 LC 与之相对应。从C 到 LC 的映射，由 JVM 的具体实现去自由实现。JVM 在类初始化阶段期间会获取这个初始化锁，并且每一个线程至少获取一次锁来确保这个类已经被初始化过了。

![singleton](https://gitee.com/chenssy/blog-home/raw/master/image/sijava/201812086002.png)

> 老艿艿：因为**基于类初始化的解决方案**，涉及到类加载机制，本文就不拓展开来，感兴趣的胖友，可以看看 [《双重检查锁定与延迟初始化》](http://www.infoq.com/cn/articles/double-checked-locking-with-delay-initialization#%E5%9F%BA%E4%BA%8E%E7%B1%BB%E5%88%9D%E5%A7%8B%E5%8C%96%E7%9A%84%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88) 的 [「基于类初始化的解决方案」](#) 小节。

# 3. 总结

> 延迟初始化降低了初始化类或创建实例的开销，但增加了访问被延迟初始化的字段的开销。**在大多数时候，正常的初始化要优于延迟初始化**。
> 
> * 如果确实需要对**实例字段**使用线程安全的延迟初始化，请使用上面介绍的基于 `volatile` 的延迟初始化的方案。
> * 如果确实需要对**静态字段**使用线程安全的延迟初始化，请使用上面介绍的基于类初始化的方案。

# 参考资料

1. 方腾飞：《Java并发编程的艺术》
2. 程晓明：[《双重检查锁定与延迟初始化》](http://www.infoq.com/cn/articles/double-checked-locking-with-delay-initialization)

# 666. 彩蛋

如果你对 Java 并发感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

