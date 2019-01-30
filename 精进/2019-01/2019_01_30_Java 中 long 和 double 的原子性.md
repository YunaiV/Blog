title: Java 中 long 和 double 的原子性
date: 2019-01-30
tags:
categories: 精进
permalink: Fight/Atomicity-of-long-and-double-in-Java
author: LouisWong
from_url: https://my.oschina.net/u/1753415/blog/724242
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247486080&idx=2&sn=1fc60b6bf77308e17ab8cb731f33f99f&chksm=fa497531cd3efc27fd2479e4ffd6ba32d5900eae0c3aa565662a0d9a38667b403040aec811fc&token=1524868883&lang=zh_CN#rd

-------

摘要: 原创出处 https://my.oschina.net/u/1753415/blog/724242 「LouisWong」欢迎转载，保留摘要，谢谢！

- [JVM中对long的操作是不是原子操作？](http://www.iocoder.cn/Fight/Atomicity-of-long-and-double-in-Java/)
- [为什么对long的操作不是原子的？](http://www.iocoder.cn/Fight/Atomicity-of-long-and-double-in-Java/)
- [在硬件，操作系统，JVM都是64位的情况下呢？](http://www.iocoder.cn/Fight/Atomicity-of-long-and-double-in-Java/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

> java中基本类型中，long和double的长度都是8个字节，32位（4字节）处理器对其读写操作无法一次完成，那么，JVM，long和double是原子性的吗？

# JVM中对long的操作是不是原子操作？

首先，通过一段程序对long的原子性进行判断。测试程序如下：

```Java
public class LongAtomTest implements Runnable {

    private static long field = 0;

    private volatile long value;

    public long getValue() {
        return value;
    }

    public void setValue(long value) {
        this.value = value;
    }

    public LongAtomTest(long value) {
        this.setValue(value);
    }

    @Override
    public void run() {
        int i = 0;
        while (i < 100000) {
            LongAtomTest.field = this.getValue();
            i++;
            long temp = LongAtomTest.field;
            if (temp != 1L && temp != -1L) {
                System.out.println("出现错误结果" + temp);
                System.exit(0);
            }
        }
        System.out.println("运行正确");
    }

    public static void main(String[] args) throws InterruptedException {
        // 获取并打印当前JVM是32位还是64位的
        String arch = System.getProperty("sun.arch.data.model");
        System.out.println(arch+"-bit");
        LongAtomTest t1 = new LongAtomTest(1);
        LongAtomTest t2 = new LongAtomTest(-1);
        Thread T1 = new Thread(t1);
        Thread T2 = new Thread(t2);
        T1.start();
        T2.start();
        T1.join();
        T2.join();
    }

}
```

可以看到，程序中有两条线程t1,t2； t1,t2各自不停的给long类型的静态变量field赋值为1，-1； t1,t2每次赋值后，会读取field的值，若field值既不是1又不是-1，就将field的值打印出来

如果对long的写入和读取操作是原子性的，那么，field的值只可能是1或者-1

运行结果如下

```
32-bit
出现错误结果-4294967295
运行正确
```

可以看出，当线程t1,t2同时对long进行写的时候，long出现了既不是t1写入的值，又不是t2写入的值。可以推测，jvm中对long的操作并非原子操作。

# 为什么对long的操作不是原子的？

JVM内存模型中定义了8中原子操作：

> 1. lock:将一个变量标识为被一个线程独占状态
> 2. unclock:将一个变量从独占状态释放出来，释放后的变量才可以被其他线程锁定
> 3. read:将一个变量的值从主内存传输到工作内存中，以便随后的load操作
> 4. load:把read操作从主内存中得到的变量值放入工作内存的变量的副本中
> 5. use:把工作内存中的一个变量的值传给执行引擎，每当虚拟机遇到一个使用到变量的指令时都会使用该指令
> 6. assign:把一个从执行引擎接收到的值赋给工作内存中的变量，每当虚拟机遇到一个给变量赋值的指令时，都要使用该操作
> 7. store:把工作内存中的一个变量的值传递给主内存，以便随后的write操作
> 8. write:把store操作从工作内存中得到的变量的值写到主内存中的变量

其中，与赋值，取值相关的包括 read,load,use,assign,store,write

按照这个规定，long的读写都是原子操作，与我们的实践结果相反，为什会导致这种问题呢？

对于32位操作系统来说，单次次操作能处理的最长长度为32bit，而long类型8字节64bit，所以对long的读写都要两条指令才能完成（即每次读写64bit中的32bit）。如果JVM要保证long和double读写的原子性，势必要做额外的处理。 那么，JVM有对这一情况进行额外处理吗？ 针对这一问题可以参考Java语言规范文档：[jls-17 Non-Atomic Treatment of double and long](http://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html#jls-17.7)

> *For the purposes of the Java programming language memory model, a single write to a non-volatile long or double value is treated as two separate writes: one to each 32-bit half. This can result in a situation where a thread sees the first 32 bits of a 64-bit value from one write, and the second 32 bits from another write.*

> *Writes and reads of volatile long and double values are always atomic.*

> *Writes to and reads of references are always atomic, regardless of whether they are implemented as 32-bit or 64-bit values.*

> *Some implementations may find it convenient to divide a single write action on a 64-bit long or double value into two write actions on adjacent 32-bit values. For efficiency's sake, this behavior is implementation-specific; an implementation of the Java Virtual Machine is free to perform writes to long and double values atomically or in two parts.*

> *Implementations of the Java Virtual Machine are encouraged to avoid splitting 64-bit values where possible. Programmers are encouraged to declare shared 64-bit values as volatile or synchronize their programs correctly to avoid possible complications.*

从规定中我们可以知道

1. 对于64位的long和double，如果没有被volatile修饰，那么对其操作可以不是原子的。在操作的时候，可以分成两步，每次对32位操作。
2. 如果使用volatile修饰long和double，那么其读写都是原子操作
3. 对于64位的引用地址的读写，都是原子操作
4. 在实现JVM时，可以自由选择是否把读写long和double作为原子操作
5. 推荐JVM实现为原子操作

从程序得到的结果来看，32位的HotSpot没有把long和double的读写实现为原子操作。 在读写的时候，分成两次操作，每次读写32位。因为采用了这种策略，所以64位的long和double的读与写都不是原子操作。

# 在硬件，操作系统，JVM都是64位的情况下呢？

对于64bit的环境来说，单次操作可以操作64bit的数据，即可以以一次性读写long或double的整个64bit。因此我们可以猜测，在64位的环境下，long和double的读写有可能是原子操作。 在换了64位的JVM之后，多次运行，结果都是正确的

```
64-bit
运行正确
运行正确
```

结果表明，在64bit的虚拟机下，long的处理是原子性的。