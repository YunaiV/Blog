title: 【占小狼】深入浅出 Netty 内存管理 PoolChunkList
date: 2018-01-10
tags:
categories: Netty
permalink: Netty/zhanxiaolang/PoolChunkList
author: 占小狼
from_url: https://www.jianshu.com/p/a1debfe4ff02
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484874&idx=2&sn=b4c98142edaadff1935884399c378b58&chksm=fa497a7bcd3ef36dd9eafe97dc1598ee81184fb4359bd2fa3fe9eabde28778df88381c462fd6#rd

-------

摘要: 原创出处 https://www.jianshu.com/p/a1debfe4ff02 「占小狼」欢迎转载，保留摘要，谢谢！

- [PoolChunkList](http://www.iocoder.cn/Netty/zhanxiaolang/PoolChunkList/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

前面两篇分别分析了PoolChunk和PoolSubpage的实现，本文主要分析管理PoolChunk生命周期的PoolChunkList。
[1、深入浅出Netty内存管理 PoolChunk](https://www.jianshu.com/p/c4bd37a3555b)
[2、深入浅出Netty内存管理 PoolSubpage](https://www.jianshu.com/p/d91060311437)

### PoolChunkList

PoolChunkList负责管理多个chunk的生命周期，在此基础上对内存分配进行进一步的优化。

```Java
final class PoolChunkList<T> implements PoolChunkListMetric {

    private final PoolChunkList<T> nextList;
    private final int minUsage;
    private final int maxUsage;

    private PoolChunk<T> head;
    private PoolChunkList<T> prevList;
    ...
}
```

从代码实现可以看出，每个PoolChunkList实例维护了一个PoolChunk链表，自身也形成一个链表，为何要这么实现？

![img](http://upload-images.jianshu.io/upload_images/2184951-c5a5bf6c03d86ce0.png)

Paste_Image.png

随着chunk中page的不断分配和释放，会导致很多碎片内存段，大大增加了之后分配一段连续内存的失败率，针对这种情况，可以把内存使用率较大的chunk放到PoolChunkList链表更后面，具体实现如下：

```Java
boolean allocate(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
    if (head == null) {
        return false;
    }

    for (PoolChunk<T> cur = head;;) {
        long handle = cur.allocate(normCapacity);
        if (handle < 0) {
            cur = cur.next;
            if (cur == null) {
                return false;
            }
        } else {
            cur.initBuf(buf, handle, reqCapacity);
            if (cur.usage() >= maxUsage) {   // (1)
                remove(cur);
                nextList.add(cur);
            }
            return true;
        }
    }
}
```

假设poolChunkList中已经存在多个chunk。当分配完内存后，如果当前chunk的使用量超过maxUsage，则把该chunk从当前链表中删除，添加到下一个链表中。

但是，随便chunk中内存的释放，其内存使用率也会随着下降，当下降到minUsage时，该chunk会移动到前一个列表中，实现如下：

```Java
boolean free(PoolChunk<T> chunk, long handle) {
    chunk.free(handle);
    if (chunk.usage() < minUsage) {
        remove(chunk);
        if (prevList == null) {
            assert chunk.usage() == 0;
            return false;
        } else {
            prevList.add(chunk);
            return true;
        }
    }
    return true;
}
```

从poolChunkList的实现可以看出，每个chunkList的都有一个上下限：minUsage和maxUsage，两个相邻的chunkList，前一个的maxUsage和后一个的minUsage必须有一段交叉值进行缓冲，否则会出现某个chunk的usage处于临界值，而导致不停的在两个chunk间移动。

所以chunk的生命周期不会固定在某个chunkList中，随着内存的分配和释放，根据当前的内存使用率，在chunkList链表中前后移动。

# 666. 彩蛋

如果你对 Netty 并发感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)