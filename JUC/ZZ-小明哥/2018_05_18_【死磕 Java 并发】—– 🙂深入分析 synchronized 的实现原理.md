title: 【死磕 Java 并发】—– 深入分析 synchronized 的实现原理
date: 2018-05-18
tags:
categories: JUC
permalink: JUC/sike/synchronized
author: 小明哥
from_url: http://cmsblogs.com/?p=2071
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247483775&idx=1&sn=e3c249e55dc25f323d3922d215e17999&chksm=fa497ececd3ef7d82a9ce86d6ca47353acd45d7d1cb296823267108a06fbdaf71773f576a644#rd

-------

摘要: 原创出处 http://cmsblogs.com/?p=2071 「小明哥」欢迎转载，保留摘要，谢谢！

作为「小明哥」的忠实读者，「老艿艿」略作修改，记录在理解过程中，参考的资料。

- [1. 实现原理](http://www.iocoder.cn/JUC/sike/synchronized/)
- [2. Java 对象头、Monitor](http://www.iocoder.cn/JUC/sike/synchronized/)
  - [2.1 Java对象头](http://www.iocoder.cn/JUC/sike/synchronized/)
  - [2.2 Monitor](http://www.iocoder.cn/JUC/sike/synchronized/)
- [3. 锁优化](http://www.iocoder.cn/JUC/sike/synchronized/)
  - [3.1 自旋锁](http://www.iocoder.cn/JUC/sike/synchronized/)
    - [3.1.1 适应自旋锁](http://www.iocoder.cn/JUC/sike/synchronized/)
  - [3.2 锁消除](http://www.iocoder.cn/JUC/sike/synchronized/)
  - [3.3 锁粗化](http://www.iocoder.cn/JUC/sike/synchronized/)
  - [3.4 锁的升级](http://www.iocoder.cn/JUC/sike/synchronized/)
    - [3.4.1 重量级锁](http://www.iocoder.cn/JUC/sike/synchronized/)
    - [3.4.2 轻量级锁](http://www.iocoder.cn/JUC/sike/synchronized/)
    - [3.4.3 偏向锁](http://www.iocoder.cn/JUC/sike/synchronized/)
    - [3.4.4 对比和转换](http://www.iocoder.cn/JUC/sike/synchronized/)
- [参考资料](http://www.iocoder.cn/JUC/sike/synchronized/)
- [666. 彩蛋](http://www.iocoder.cn/JUC/sike/synchronized/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

记得刚刚开始学习 Java 的时候，一遇到多线程情况就是 `synchronized` ，相对于当时的我们来说 `synchronized` 是这么的神奇而又强大，那个时候我们赋予它一个名字“**同步**”，也成为了我们解决多线程情况的百试不爽的良药。

但是，随着我们学习的进行我们知道 `synchronized` 是一个**重量级锁**，相对于 Lock ，它会显得那么笨重，以至于我们认为它不是那么的高效而慢慢摒弃它。

诚然，随着 Javs SE 1.6 对 `synchronized` 进行的各种优化后，`synchronized` 并不会显得那么重了。下面跟随 LZ 一起来探索 `synchronized` 的实现机制、Java 是如何对它进行了优化、锁优化机制、锁的存储结构和升级过程。

# 1. 实现原理

> `synchronized` 可以保证方法或者代码块在运行时，同一时刻只有一个方法可以进入到临界区，同时它还可以保证共享变量的内存可见性。

Java 中每一个对象都可以作为锁，这是 `synchronized` 实现同步的基础：

1. 普通同步方法，锁是当前实例对象
2. 静态同步方法，锁是当前类的 class 对象
3. 同步方法块，锁是括号里面的对象

当一个线程访问同步代码块时，它首先是需要得到锁才能执行同步代码，当退出或者抛出异常时必须要释放锁，那么它是如何来实现这个机制的呢？我们先看一段简单的代码：

```Java
public class SynchronizedTest {

    public synchronized void test1() {

    }

    public void test2() {
        synchronized(this) {

        }
    }
    
}
```

利用 [Javap](http://www.importnew.com/18398.html) 工具查看生成的 class 文件信息来分析 `synchronized` 的实现

![Synchronize-1](https://gitee.com/chenssy/blog-home/raw/master/image/sijava/201812081001.png)

从上面可以看出：1）同步代码块是使用 `monitorenter` 和 `monitorexit` 指令实现的；2）同步方法（在这看不出来需要看JVM底层实现）依靠的是方法修饰符上的`ACC_SYNCHRONIZED` 实现。

* **同步代码块**：`monitorenter` 指令插入到同步代码块的开始位置，`monitorexit` 指令插入到同步代码块的结束位置，JVM 需要保证每一个 `monitorenter` 都有一个 `monitorexit` 与之相对应。任何对象都有一个 Monitor 与之相关联，当且一个 Monitor 被持有之后，他将处于锁定状态。线程执行到 `monitorenter` 指令时，将会尝试获取对象所对应的 Monitor 所有权，即尝试获取对象的锁。

* **同步方法**：`synchronized` 方法则会被翻译成普通的方法调用和返回指令如：`invokevirtual`、`areturn` 指令，在 VM 字节码层面并没有任何特别的指令来实现被`synchronized` 修饰的方法，而是在 Class 文件的方法表中将该方法的 `access_flags` 字段中的 `synchronized` 标志位置设置为 1，表示该方法是同步方法，并使用**调用该方法的对象**或**该方法所属的 Class 在 JVM 的内部对象表示 Klass** 作为锁对象。（摘自：[《JVM内部细节之一：synchronized关键字及实现细节（轻量级锁Lightweight Locking）》](<http://www.cnblogs.com/javaminer/p/3889023.html>)）

    > * `accessFlags` 是类和方法的访问标志，总共 16 bits 。具体每 1 bit 的表示，可参见 [AccessFlag.java](https://github.com/zachaxy/JVM/blob/d42329556f38a012233ea34cca42af127a85dbf1/Java/src/runtimedata/heap/AccessFlag.java#L16) 。
    >
    > * 关于 Klass ，感兴趣的胖友，可以看看 [《【理解HotSpot虚拟机】对象在 JVM 中的表示：OOP-Klass 模型》](https://blog.csdn.net/linxdcn/article/details/73287490) 。
    >
    > * 关于**同步代码块**和**同步方法**的更加**详细**的实现原理，可参见 [《Java 8 并发篇 - 冷静分析 Synchronized（下）》](https://juejin.im/post/5abc9de851882555770c8c72#heading-35) [「4.3 同步代码块同步原理」](#) 部分，写的非常棒。

下面我们来继续分析，但是在深入之前我们需要了解两个重要的概念：Java对象头，Monitor。

# 2. Java 对象头、Monitor

Java 对象头和 Monitor 是实现 `synchronized` 的基础！下面就这两个概念来做详细介绍。

## 2.1 Java对象头

`synchronized` 用的锁是存在Java对象头里的。**那么什么是 Java 对象头呢**？Hotspot 虚拟机的对象头主要包括两部分数据：Mark Word（标记字段）、Klass Pointer（类型指针）。其中：

* Klass Point 是是对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。
* Mark Word 用于存储对象自身的运行时数据，它是实现轻量级锁和偏向锁的关键，所以下面将重点阐述 Mark Word 。

    > 关于 Klass Point  和 Mark Word 的说明，感兴趣的胖友，可以看看 [《【理解HotSpot虚拟机】对象在 JVM 中的表示：OOP-Klass 模型》](https://blog.csdn.net/linxdcn/article/details/73287490)  的 [「实例说明」](#)。

Mark Word 用于存储对象自身的运行时数据，如哈希码（HashCode）、GC 分代年龄、锁状态标志、线程持有的锁、偏向线程 ID、偏向时间戳等等。Java 对象头一般占有两个机器码（在 32 位虚拟机中，1 个机器码等于 4 字节，也就是 32 bits）。但是如果对象是数组类型，则需要三个机器码，因为 JVM 虚拟机可以通过 Java 对象的元数据信息确定 Java 对象的大小，无法从数组的元数据来确认数组的大小，所以用一块来记录数组长度。

下图是 Java 对象头的存储结构（32位虚拟机）：

![存储结构](https://gitee.com/chenssy/blog-home/raw/master/image/sijava/201812081002.png)

对象头信息是与对象自身定义的数据无关的额外存储成本，但是考虑到虚拟机的空间效率，Mark Word 被设计成一个**非固定**的数据结构以便在极小的空间内存存储尽量多的数据，它会根据对象的状态复用自己的存储空间，也就是说，Mark Word 会随着程序的运行发生变化，变化状态如下：

* 32 位虚拟机：

    ![32 位虚拟机](https://gitee.com/chenssy/blog-home/raw/master/image/sijava/201812081003.png)
    
    * 每一行，是一种情况。

* 64 位虚拟机：

    ![](https://user-gold-cdn.xitu.io/2018/3/29/16270c8c27da265d?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

    * 对于 32 位无锁状态，有 25 bits 没有使用。


简单介绍了 Java 对象头，我们下面再看 Monitor。

## 2.2 Monitor

**什么是 Monitor** ？我们可以把它理解为一个同步工具，也可以描述为一种同步机制，它通常被描述为一个对象。

> FROM [《Java 8 并发篇 - 冷静分析 Synchronized（下）》](https://juejin.im/post/5abc9de851882555770c8c72)
> 
> * **互斥**： 一个 Monitor 锁在同一时刻只能被一个线程占用，其他线程无法占用。
> * **信号机制( signal )**： 占用 Monitor 锁失败的线程会暂时放弃竞争并等待某个谓词成真（条件变量），但该条件成立后，当前线程会通过释放锁通知正在等待这个条件变量的其他线程，让其可以重新竞争锁。

与一切皆对象一样，所有的 Java 对象是天生的 Monitor ，每一个 Java 对象都有成为Monitor 的潜质，因为在 Java 的设计中 ，每一个 Java 对象自打娘胎里出来就带了一把看不见的锁，它叫做**内部锁**或者 **Monitor 锁**。

> FROM 《Java并发编程的艺术》的 [「2.2 synchronized 的实现原理与引用」](#) 章节。
> 
> Monitor Record 是线程**私有**的数据结构，每一个线程都有一个可用 Monitor Record 列表，同时还有一个全局的可用列表。  
> 每一个被锁住的对象都会和一个 Monitor Record 关联（对象头的 MarkWord 中的 LockWord 指向 Monitor 的起始地址），Monitor Record 中有一个 Owner 字段，存放拥有该锁的线程的唯一标识，表示该锁被这个线程占用。其结构如下：

> ![Monitor Record](https://gitee.com/chenssy/blog-home/raw/master/image/sijava/201812081004.png)

> * **Owner**：1）初始时为 NULL 表示当前没有任何线程拥有该 Monitor Record；2）当线程成功拥有该锁后保存线程唯一标识；3）当锁被释放时又设置为 NULL 。
> * **EntryQ**：关联一个系统互斥锁（ semaphore ），阻塞所有试图锁住 Monitor Record失败的线程 。
> * **RcThis**：表示 blocked 或 waiting 在该 Monitor Record 上的所有线程的个数。
> * **Nest**：用来实现重入锁的计数。
> * **HashCode**：保存从对象头拷贝过来的 HashCode 值（可能还包含 GC age ）。
> * **Candidate**：用来避免不必要的阻塞或等待线程唤醒。因为每一次只有一个线程能够成功拥有锁，如果每次前一个释放锁的线程唤醒所有正在阻塞或等待的线程，会引起不必要的上下文切换（从阻塞到就绪然后因为竞争锁失败又被阻塞）从而导致性能严重下降。Candidate 只有两种可能的值 ：1）0 表示没有需要唤醒的线程；2）1 表示要唤醒一个继任线程来竞争锁。

我们知道 `synchronized` 是重量级锁，效率不怎么滴，同时这个观念也一直存在我们脑海里，不过在 JDK 1.6 中对 `synchronize` 的实现进行了各种优化，使得它显得不是那么重了，那么 JVM 采用了那些优化手段呢？

# 3. 锁优化

> FROM [《JVM 内部细节之一：synchronized 关键字及实现细节（轻量级锁Lightweight Locking）》](https://www.cnblogs.com/javaminer/p/3889023.html)
> 
> 简单来说，在 JVM 中 `monitorenter` 和 `monitorexit` 字节码依赖于底层的操作系统的Mutex Lock 来实现的，但是由于使用 Mutex Lock 需要将当前线程挂起并从用户态切换到内核态来执行，这种切换的代价是非常昂贵的。然而，在现实中的大部分情况下，同步方法是运行在单线程环境（**无锁竞争环境**），如果每次都调用 Mutex Lock 那么将严重的影响程序的性能。

因此，JDK 1.6 对锁的实现引入了大量的优化，如自旋锁、适应性自旋锁、锁消除、锁粗化、偏向锁、轻量级锁等技术来减少锁操作的开销。

## 3.1 自旋锁

**由来**

线程的阻塞和唤醒，**需要 CPU 从用户态转为核心态**。频繁的阻塞和唤醒对 CPU 来说是一件负担很重的工作，势必会给系统的并发性能带来很大的压力。同时，我们发现在许多应用上面，**对象锁的锁状态只会持续很短一段时间**。为了这一段很短的时间，频繁地阻塞和唤醒线程是非常不值得的。所以引入自旋锁。

**定义**

所谓自旋锁，就是让该线程等待一段时间，不会被立即挂起，看持有锁的线程是否会很快释放锁。

怎么等待呢？**执行一段无意义的循环即可**（自旋）。

自旋等待不能替代阻塞，先不说对处理器数量的要求（多核，貌似现在没有单核的处理器了），虽然它可以避免线程切换带来的开销，但是它占用了处理器的时间。如果持有锁的线程很快就释放了锁，那么自旋的效率就非常好，反之，自旋的线程就会白白消耗掉处理的资源，它不会做任何有意义的工作，典型的占着茅坑不拉屎，这样反而会带来性能上的浪费。

所以说，自旋等待的时间（自旋的**次数**）必须要有一个限度，如果自旋超过了定义的时间仍然没有获取到锁，则应该被挂起。

自旋锁在 JDK 1.4.2 中引入，默认关闭，但是可以使用 `-XX:+UseSpinning` 开开启。  
在 JDK1.6 中默认开启。同时自旋的默认次数为 10 次，可以通过参数 `-XX:PreBlockSpin` 来调整。

如果通过参数 `-XX:PreBlockSpin` 来调整自旋锁的自旋次数，会带来诸多不便。假如我将参数调整为 10 ，但是系统很多线程都是等你刚刚退出的时候，就释放了锁（假如你多自旋一两次就可以获取锁），你是不是很尴尬。于是 JDK 1.6 引入自适应的自旋锁，让虚拟机会变得越来越聪明。

### 3.1.1 适应自旋锁

JDK 1.6 引入了更加聪明的自旋锁，即自适应自旋锁。

所谓自适应就意味着自旋的次数**不再是固定**的，**它是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定**。它怎么做呢？

* 线程如果自旋成功了，那么下次自旋的次数会更加多，因为虚拟机认为既然上次成功了，那么此次自旋也很有可能会再次成功，那么它就会允许自旋等待持续的次数更多。
* 反之，如果对于某个锁，很少有自旋能够成功的，那么在以后要或者这个锁的时候自旋的次数会减少甚至省略掉自旋过程，以免浪费处理器资源。

有了自适应自旋锁，随着程序运行和性能监控信息的不断完善，虚拟机对程序锁的状况预测会越来越准确，虚拟机会变得越来越聪明。

## 3.2 锁消除

**由来**

为了保证数据的完整性，我们在进行操作时需要对这部分操作进行同步控制。但是，在有些情况下，JVM检测到不可能存在共享数据竞争，这是JVM会对这些同步锁进行锁消除。如果不存在竞争，为什么还需要加锁呢？所以锁消除可以节省毫无意义的请求锁的时间。

**定义**

锁消除的依据是**逃逸分析的数据支持**。变量是否逃逸，对于虚拟机来说需要使用数据流分析来确定，但是对于我们程序员来说这还不清楚么？我们会在明明知道不存在数据竞争的代码块前加上同步吗？但是有时候程序并不是我们所想的那样？我们虽然没有显示使用锁，但是我们在使用一些 JDK 的内置 API 时，如 StringBuffer、Vector、HashTable 等，这个时候会存在**隐性的加锁操作**。比如 StringBuffer 的 `#append(..)`方法，Vector 的 `add(...)` 方法：

```Java
public void vectorTest(){
    Vector<String> vector = new Vector<String>();
    for (int i = 0 ; i < 10 ; i++){
    	vector.add(i + "");
    }
    System.out.println(vector);
}
```

在运行这段代码时，JVM 可以明显检测到变量 `vector` 没有逃逸出方法 `#vectorTest()` 之外，所以 JVM 可以大胆地将 `vector` 内部的加锁操作消除。

## 3.3 锁粗化

**由来**

我们知道在使用同步锁的时候，需要让同步块的作用范围尽可能小：仅在共享数据的实际作用域中才进行同步。这样做的目的，是为了使需要同步的操作数量尽可能缩小，如果存在锁竞争，那么等待锁的线程也能尽快拿到锁。

在大多数的情况下，上述观点是正确的，LZ 也一直坚持着这个观点。但是如果一系列的连续加锁解锁操作，可能会导致不必要的性能损耗，所以引入**锁粗话**的概念。

**定义**

锁粗话概念比较好理解，就是将**多个连续的加锁、解锁操作连接在一起，扩展成一个范围更大的锁**。

如上面实例：`vector` 每次 add 的时候都需要加锁操作，JVM 检测到对同一个对象（`vector`）连续加锁、解锁操作，会合并一个更大范围的加锁、解锁操作，即加锁解锁操作会移到 `for` 循环之外。

## 3.4 锁的升级

锁主要存在四种状态，依次是：无锁状态、偏向锁状态、轻量级锁状态、重量级锁状态。它们会随着竞争的激烈而逐渐升级。**注意，锁可以升级不可降级，这种策略是为了提高获得锁和释放锁的效率**。

### 3.4.1 重量级锁

重量级锁通过对象内部的监视器（Monitor）实现。

其中，Monitor 的**本质**是，依赖于底层操作系统的 [Mutex Lock](http://dreamrunner.org/blog/2014/06/29/qian-tan-mutex-lock/) 实现。操作系统实现线程之间的切换，需要从用户态到内核态的切换，切换成本非常高。

### 3.4.2 轻量级锁

引入轻量级锁的主要目的，是在没有多线程竞争的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗。

当关闭偏向锁功能或者多个线程竞争偏向锁，导致**偏向锁升级为轻量级锁**，则会尝试获取轻量级锁，其步骤如下：

**获取锁**

1. 判断当前对象是否处于无锁状态？**若是**，则 JVM 首先将在当前线程的栈帧中，建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的 Mark Word的 拷贝（官方把这份拷贝加了一个 Displaced 前缀，即 Displaced Mark Word）；**否则**，执行步骤（3）；
2. JVM 利用 CAS 操作尝试将对象的 Mark Word 更新为指向 Lock Record 的指正。如果**成功**，表示竞争到锁，则将锁标志位变成 `00`（表示此对象处于轻量级锁状态），执行同步操作；如果**失败**，则执行步骤（3）；
3. 判断当前对象的 Mark Word 是否指向当前线程的栈帧？如果**是**，则表示当前线程已经持有当前对象的锁，则直接执行同步代码块；**否则**，只能说明该锁对象已经被其他线程抢占了，当前线程便尝试使用**自旋**来获取锁。若自旋后没有获得锁，此时轻量级锁会升级为重量级锁，锁标志位变成 `10`，当前线程会被阻塞。

**释放锁**

轻量级锁的释放也是通过 CAS 操作来进行的，主要步骤如下：

1. 取出在获取轻量级锁保存在 Displaced Mark Word 中 数据。
2. 使用 CAS 操作将取出的数据替换当前对象的 Mark Word 中。如果**成功**，则说明释放锁成功；**否则**，执行（3）。
3. 如果 CAS 操作替换失败，说明有其他线程尝试获取该锁，则需要在释放锁的同时需要唤醒被挂起的线程。

    > 老艿艿：这块的描述不太准确，我的理解是，无论（**2**）是否释放成功，都会唤醒被挂起的线程，重新争夺锁，访问同步代码块。

下图是争夺锁导致的**锁膨胀**的流程图：

![争夺锁导致的锁膨胀](https://gitee.com/chenssy/blog-home/raw/master/image/sijava/201812081005.png)

* 其中，绿框的 `0` 指的是无偏向锁，`01` 指的是无锁状态。

-------

**注意事项**

对于轻量级锁，其性能提升的依据是：“**对于绝大部分的锁，在整个生命周期内都是不会存在竞争的**”。如果打破这个依据则除了互斥的开销外，还有额外的 CAS 操作，因此在有多线程竞争的情况下，轻量级锁比重量级锁更慢。

### 3.4.3 偏向锁

引入偏向锁主要**目的**是：为了在无多线程竞争的情况下，尽量减少不必要的轻量级锁执行路径。

上面提到了轻量级锁的加锁解锁操作，是需要依赖多次 CAS 原子指令的。那么偏向锁是如何来减少不必要的 CAS 操作呢？我们可以查看 Mark Word 的数据结构就明白了。

> 老艿艿：在上文，我们可以看到**偏向锁**时，Mark Word 的数据结构为：线程 ID、Epoch( 偏向锁的时间戳 )、对象分带年龄、是否是偏向锁( `1` )、锁标识位( `01` )。

只需要检查是否为偏向锁、锁标识为以及 ThreadID 即可，处理流程如下：

**获取偏向锁**

1. 检测 Mark Word是 否为可偏向状态，即是否为偏向锁的标识位为 `1` ，锁标识位为 `01` 。
2. 若为可偏向状态，则测试线程 ID 是否为当前线程 ID ？如果**是**，则执行步骤（5）；**否则**，执行步骤（3）。
3. 如果线程 ID 不为当前线程 ID ，则通过 CAS 操作竞争锁。竞争**成功**，则将 Mark Word 的线程 ID 替换为**当前**线程 ID ，则执行步骤（5）；**否则**，执行线程（4）。
4. 通过 CAS 竞争锁失败，证明当前存在多线程竞争情况，当到达全局安全点，获得偏向锁的线程被挂起，**偏向锁升级为轻量级锁**，然后被阻塞在安全点的线程继续往下执行同步代码块。
5. 执行同步代码块

**撤销偏向锁**

**偏向锁的释放采用了一种只有竞争才会释放锁的机制**，线程是**不会主动**去释放偏向锁，需要等待其他线程来竞争。

偏向锁的撤销需要等待**全局安全点**（这个时间点是上没有正在执行的代码）。其步骤如下：

1. 暂停拥有偏向锁的线程，判断锁对象是否还处于被锁定状态。
2. 撤销偏向锁，恢复到无锁状态（ `01` ）或者轻量级锁的状态。

最后唤醒暂停的线程。

> 老艿艿：关于**偏向锁的撤销**，网上的文章大同小异，/(ㄒoㄒ)/~~ 看的是一脸懵逼。
> 
> 后面撸下源码，如果撸的动。
> 
> 如下是 [《Java 8 并发篇 - 冷静分析 Synchronized（下）》](https://juejin.im/post/5abc9de851882555770c8c72#heading-35) 对这块的描述：
> 
> 首先会**暂停拥有偏向锁的线程并检查该线程是否存活**：  
> 
> 1. 如果线程**非活动状态**，则将**对象头设置为无锁状态**（其他线程会重新获取该偏向锁）。
> 2. 如果线程是**活动状态**，拥有偏向锁的栈会被执行，遍历偏向对象的锁记录，并**将对栈中的锁记录和对象头的 MarkWord 进行重置**：  
> 
>     - 要么**重新偏向于其他线程**(即将偏向锁交给其他线程，相当于当前线程"被"释放了锁)
>     - 要么**恢复到无锁**或者**标记锁对象不适合作为偏向锁**(此时锁会被升级为轻量级锁)
> 
> **最后唤醒暂停的线程，被阻塞在安全点的线程继续往下执行同步代码块**


下图是偏向锁的获取和释放流程：

![偏向锁的获取和释放流程](https://gitee.com/chenssy/blog-home/raw/master/image/sijava/201812081006.png)

**关闭偏向锁**

偏向锁在 JDK 1.6 以上，默认开启。开启后程序启动几秒后才会被激活，可使用 JVM 参数 `-XX：BiasedLockingStartupDelay = 0` 来关闭延迟。

如果确定锁**通常处于竞争状态**，则可通过JVM参数 `-XX:-UseBiasedLocking=false` 关闭偏向锁，那么默认会进入轻量级锁。

**注意事项**

> 如下是 [《Java 8 并发篇 - 冷静分析 Synchronized（下）》](https://juejin.im/post/5abc9de851882555770c8c72) 对这块的描述：
> 
> * 优势：偏向锁只需要在置换 ThreadID 的时候依赖一次 CAS 原子指令，其余时刻不需要 CAS 指令(相比其他锁)。
> * 隐患：由于一旦出现多线程竞争的情况就必须撤销偏向锁，所以偏向锁的撤销操作的性能损耗必须小于节省下来的 CAS 原子指令的性能消耗（这个通常只能通过大量压测才可知）。
> * 对比：轻量级锁是为了在线程交替执行同步块时提高性能，而偏向锁则是在只有一个线程执行同步块时进一步提高性能

### 3.4.4 对比和转换

如下是引用自 [《Java并发编程的艺术》](#) 的**对比图**：

![偏向锁 vs 轻量级锁 vs 重量级锁](https://user-gold-cdn.xitu.io/2018/4/8/162a318a71ce992d?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

如下是三种锁之间的**转换图**：

![偏向锁 => 轻量级锁 => 重量级锁](http://static.iocoder.cn/cnblogs/9EB59781-D801-4922-90CA-C6D34944BB0C.png)

如果觉得解释不够清晰的胖友，推荐阅读**占小狼**的 [《JVM 源码分析之 synchronized 实现》](https://www.jianshu.com/p/c5058b6fe8e5) 。

# 参考资料

1. 周志明：《深入理解Java虚拟机》
    2. 方腾飞：《Java并发编程的艺术》的 [「2.2 synchronized 的实现原理与引用」](#) 章节。
3.  [《Java 8 并发篇 - 冷静分析 Synchronized（下）》](https://juejin.im/post/5abc9de851882555770c8c72) 

# 666. 彩蛋

整理本小节，简单脑图如下：![脑图](http://www.iocoder.cn/images/JUC/synchroized-01.png)

如果你对 Java 并发感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

