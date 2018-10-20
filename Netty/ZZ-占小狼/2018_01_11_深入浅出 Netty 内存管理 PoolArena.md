title: 【占小狼】深入浅出 Netty 内存管理 PoolArena
date: 2018-01-11
tags:
categories: Netty
permalink: Netty/zhanxiaolang/PoolArena
author: 占小狼
from_url: https://www.jianshu.com/p/4856bd30dd56
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484909&idx=2&sn=54c9d7b5d3e49783a0504074bd79e763&chksm=fa497a5ccd3ef34a860d733c587b2dd7a8b382d5b10d7b10f7d985dcb0a319af4c2bc65ac1a1&token=1286521154&lang=zh_CN#rd

-------

摘要: 原创出处 https://www.jianshu.com/p/4856bd30dd56 「占小狼」欢迎转载，保留摘要，谢谢！

- [PoolArena](http://www.iocoder.cn/Netty/zhanxiaolang/PoolArena/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

前面分别分析了PoolChunk、PoolSubpage和PoolChunkList，本文主要分析PoolArena。
[1、深入浅出Netty内存管理 PoolChunk](https://www.jianshu.com/p/c4bd37a3555b)
[2、深入浅出Netty内存管理 PoolSubpage](https://www.jianshu.com/p/d91060311437)
[3、深入浅出Netty内存管理 PoolChunkList](https://www.jianshu.com/p/a1debfe4ff02)

### PoolArena

应用层的内存分配主要通过如下实现，但最终还是委托给PoolArena实现。

```Java
PooledByteBufAllocator.DEFAULT.directBuffer(128);
```

由于netty通常应用于高并发系统，不可避免的有多线程进行同时内存分配，可能会极大的影响内存分配的效率，为了缓解线程竞争，可以通过创建多个poolArena细化锁的粒度，提高并发执行的效率。

先看看poolArena的内部结构：

![img](http://upload-images.jianshu.io/upload_images/2184951-c4d3a846c6051aed.png)

poolArena

所有内存分配的size都会经过normalizeCapacity进行处理，当size>=512时，size成倍增长512->1024->2048->4096->8192，而size<512则是从16开始，每次加16字节。

poolArena提供了两种方式进行内存分配：

1. PoolSubpage用于分配小于8k的内存；

- tinySubpagePools：用于分配小于512字节的内存，默认长度为32，因为内存分配最小为16，每次增加16，直到512，区间[16，512)一共有32个不同值；
- smallSubpagePools：用于分配大于等于512字节的内存，默认长度为4；
- tinySubpagePools和smallSubpagePools中的元素都是默认subpage。

1. poolChunkList用于分配大于8k的内存；

- qInit：存储内存利用率0-25%的chunk
- q000：存储内存利用率1-50%的chunk
- q025：存储内存利用率25-75%的chunk
- q050：存储内存利用率50-100%的chunk
- q075：存储内存利用率75-100%的chunk
- q100：存储内存利用率100%的chunk

![img](http://upload-images.jianshu.io/upload_images/2184951-d854b1697ada013d.png)

poolChunkList

1. qInit前置节点为自己，且minUsage=Integer.MIN_VALUE，意味着一个初分配的chunk，在最开始的内存分配过程中(内存使用率<25%)，即使完全释放也不会被回收，会始终保留在内存中。
2. q000没有前置节点，当一个chunk进入到q000列表，如果其内存被完全释放，则不再保留在内存中，其分配的内存被完全回收。

接下去看看poolArena如何实现内存的分配，实现如下：

```Java
private void allocate(PoolThreadCache cache, PooledByteBuf<T> buf, final int reqCapacity) {
    final int normCapacity = normalizeCapacity(reqCapacity);
    if (isTinyOrSmall(normCapacity)) { // capacity < pageSize
        int tableIdx;
        PoolSubpage<T>[] table;
        boolean tiny = isTiny(normCapacity);
        if (tiny) { // < 512
            if (cache.allocateTiny(this, buf, reqCapacity, normCapacity)) {
                // was able to allocate out of the cache so move on
                return;
            }
            tableIdx = tinyIdx(normCapacity);
            table = tinySubpagePools;
        } else {
            if (cache.allocateSmall(this, buf, reqCapacity, normCapacity)) {
                // was able to allocate out of the cache so move on
                return;
            }
            tableIdx = smallIdx(normCapacity);
            table = smallSubpagePools;
        }

        final PoolSubpage<T> head = table[tableIdx];

        /**
         * Synchronize on the head. This is needed as {@link PoolChunk#allocateSubpage(int)} and
         * {@link PoolChunk#free(long)} may modify the doubly linked list as well.
         */
        synchronized (head) {
            final PoolSubpage<T> s = head.next;
            if (s != head) {
                assert s.doNotDestroy && s.elemSize == normCapacity;
                long handle = s.allocate();
                assert handle >= 0;
                s.chunk.initBufWithSubpage(buf, handle, reqCapacity);

                if (tiny) {
                    allocationsTiny.increment();
                } else {
                    allocationsSmall.increment();
                }
                return;
            }
        }
        allocateNormal(buf, reqCapacity, normCapacity);
        return;
    }
    if (normCapacity <= chunkSize) {
        if (cache.allocateNormal(this, buf, reqCapacity, normCapacity)) {
            // was able to allocate out of the cache so move on
            return;
        }
        allocateNormal(buf, reqCapacity, normCapacity);
    } else {
        // Huge allocations are never served via the cache so just call allocateHuge
        allocateHuge(buf, reqCapacity);
    }
}
```

1、默认先尝试从poolThreadCache中分配内存，PoolThreadCache利用ThreadLocal的特性，消除了多线程竞争，提高内存分配效率；首次分配时，poolThreadCache中并没有可用内存进行分配，当上一次分配的内存使用完并释放时，会将其加入到poolThreadCache中，提供该线程下次申请时使用。
2、如果是分配小内存，则尝试从tinySubpagePools或smallSubpagePools中分配内存，如果没有合适subpage，则采用方法allocateNormal分配内存。
3、如果分配一个page以上的内存，直接采用方法allocateNormal分配内存。

allocateNormal实现如下：

```Java
private synchronized void allocateNormal(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
    ++allocationsNormal;
    if (q050.allocate(buf, reqCapacity, normCapacity)
     || q025.allocate(buf, reqCapacity, normCapacity)
     || q000.allocate(buf, reqCapacity, normCapacity)
     || qInit.allocate(buf, reqCapacity, normCapacity)
     || q075.allocate(buf, reqCapacity, normCapacity)
     || q100.allocate(buf, reqCapacity, normCapacity)) {
        return;
    }

    // Add a new chunk.
    PoolChunk<T> c = newChunk(pageSize, maxOrder, pageShifts, chunkSize);
    long handle = c.allocate(normCapacity);
    assert handle > 0;
    c.initBuf(buf, handle, reqCapacity);
    qInit.add(c);
}
```

第一次进行内存分配时，chunkList没有chunk可以分配内存，需通过方法newChunk新建一个chunk进行内存分配，并添加到qInit列表中。如果分配如512字节的小内存，除了创建chunk，还有创建subpage，PoolSubpage在初始化之后，会添加到smallSubpagePools中，其实并不是直接插入到数组，而是添加到head的next节点。下次再有分配512字节的需求时，直接从smallSubpagePools获取对应的subpage进行分配。

![img](http://upload-images.jianshu.io/upload_images/2184951-0a64f27f9ec8aca6.png)

smallSubpagePools

分配内存时，为什么不从内存使用率较低的q000开始？在chunkList中，我们知道一个chunk随着内存的释放，会往当前chunklist的前一个节点移动。

q000存在的目的是什么？
q000是用来保存内存利用率在1%-50%的chunk，那么这里为什么不包括0%的chunk？
直接弄清楚这些，才好理解为什么不从q000开始分配。q000中的chunk，当内存利用率为0时，就从链表中删除，直接释放物理内存，避免越来越多的chunk导致内存被占满。

想象一个场景，当应用在实际运行过程中，碰到访问高峰，这时需要分配的内存是平时的好几倍，当然也需要创建好几倍的chunk，如果先从q0000开始，这些在高峰期创建的chunk被回收的概率会大大降低，延缓了内存的回收进度，造成内存使用的浪费。

那么为什么选择从q050开始？
1、q050保存的是内存利用率50%~100%的chunk，这应该是个折中的选择！这样大部分情况下，chunk的利用率都会保持在一个较高水平，提高整个应用的内存利用率；
2、qinit的chunk利用率低，但不会被回收；
3、q075和q100由于内存利用率太高，导致内存分配的成功率大大降低，因此放到最后；

# 666. 彩蛋

如果你对 Netty 并发感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)