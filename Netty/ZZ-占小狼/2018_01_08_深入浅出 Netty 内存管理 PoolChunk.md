title: 【占小狼】深入浅出 Netty 内存管理 PoolChunk
date: 2018-01-08
tags:
categories: Netty
permalink: Netty/zhanxiaolang/PoolChunk
author: 占小狼
from_url: https://www.jianshu.com/p/c4bd37a3555b
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484783&idx=1&sn=fdf54738dcba95563a01273c4bb6514b&chksm=fa497adecd3ef3c8333b4d29e49809e2a7e4129989a51f85bae78345574ffd7ddb7337bbe336#rd

-------

摘要: 原创出处 https://www.jianshu.com/p/c4bd37a3555b 「占小狼」欢迎转载，保留摘要，谢谢！

- [PoolChunk](http://www.iocoder.cn/Netty/zhanxiaolang/PoolChunk/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

多年之前，从C内存的手动管理上升到java的自动GC，是历史的巨大进步。然而多年之后，netty的内存实现又曲线的回到了手动管理模式，正印证了马克思哲学观：社会总是在螺旋式前进的，没有永远的最好。的确，就内存管理而言，GC给程序员带来的价值是不言而喻的，不仅大大的降低了程序员的负担，而且也极大的减少了内存管理带来的Crash困扰，不过也有很多情况，可能手动的内存管理更为合适。

接下去准备几个篇幅对Netty的内存管理进行深入分析。

### PoolChunk

为了能够简单的操作内存，必须保证每次分配到的内存时连续的。Netty中底层的内存分配和回收管理主要由PoolChunk实现，其内部维护一棵平衡二叉树memoryMap，所有子节点管理的内存也属于其父节点。

![img](http://upload-images.jianshu.io/upload_images/2184951-a74c7b675cfa0aa3.png)

memoryMap

poolChunk默认由2048个page组成，一个page默认大小为8k，图中节点的值为在数组memoryMap的下标。
1、如果需要分配大小8k的内存，则只需要在第11层，找到第一个可用节点即可。
2、如果需要分配大小16k的内存，则只需要在第10层，找到第一个可用节点即可。
3、如果节点1024存在一个已经被分配的子节点2048，则该节点不能被分配，如需要分配大小16k的内存，这个时候节点2048已被分配，节点2049未被分配，就不能直接分配节点1024，因为该节点目前只剩下8k内存。

poolChunk内部会保证每次分配内存大小为8K*(2n)，为了分配一个大小为chunkSize/(2k)的节点，需要在深度为k的层从左开始匹配节点，那么如何快速的分配到指定内存？

memoryMap初始化：

```Java
memoryMap = new byte[maxSubpageAllocs << 1];
depthMap = new byte[memoryMap.length];
int memoryMapIndex = 1;
for (int d = 0; d <= maxOrder; ++ d) { // move down the tree one level at a time
    int depth = 1 << d;
    for (int p = 0; p < depth; ++ p) {
        // in each level traverse left to right and set value to the depth of subtree
        memoryMap[memoryMapIndex] = (byte) d;
        depthMap[memoryMapIndex] = (byte) d;
        memoryMapIndex ++;
    }
}
```

memoryMap数组中每个位置保存的是该节点所在的层数，有什么作用？对于节点512，其层数是9，则：
1、如果memoryMap[512] = 9，则表示其本身到下面所有的子节点都可以被分配；
2、如果memoryMap[512] = 10， 则表示节点512下有子节点已经分配过，则该节点不能直接被分配，而其子节点中的第10层还存在未分配的节点;
3、如果memoryMap[512] = 12 (即总层数 + 1）, 可分配的深度已经大于总层数,  则表示该节点下的所有子节点都已经被分配。

下面看看如何向PoolChunk申请一段内存：

```Java
long allocate(int normCapacity) {
    if ((normCapacity & subpageOverflowMask) != 0) { // >= pageSize
        return allocateRun(normCapacity);
    } else {
        return allocateSubpage(normCapacity);
    }
}
```

1、当需要分配的内存大于pageSize时，使用allocateRun实现内存分配。
2、否则使用方法allocateSubpage分配内存，在allocateSubpage实现中，会把一个page分割成多段，进行内存分配。

这里先看看allocateRun是如何实现的：

```Java
private long allocateRun(int normCapacity) {
    int d = maxOrder - (log2(normCapacity) - pageShifts);
    int id = allocateNode(d);
    if (id < 0) {
        return id;
    }
    freeBytes -= runLength(id);
    return id;
}
```

1、normCapacity是处理过的值，如申请大小为1000的内存，实际申请的内存大小为1024。
2、d = maxOrder - (log2(normCapacity) - pageShifts) 可以确定需要在二叉树的d层开始节点匹配。
其中pageShifts默认值为13，为何是13？因为只有当申请内存大小大于2^13（8192）时才会使用方法allocateRun分配内存。
3、方法allocateNode实现在二叉树中进行节点匹配，具体实现如下：

```Java
private int allocateNode(int d) {
    int id = 1;
    int initial = - (1 << d);
    //value(id)=memoryMap[id]
    byte val = value(id);
    if (val > d) { // unusable
        return -1;
    }
    while (val < d || (id & initial) == 0) { // id & initial == 1 << d for all ids at depth d, for < d it is 0
        id <<= 1;
        val = value(id);
        if (val > d) {
            id ^= 1;
            val = value(id);
        }
    }
    byte value = value(id);
    assert value == d && (id & initial) == 1 << d : String.format("val = %d, id & initial = %d, d = %d",
            value, id & initial, d);
    setValue(id, unusable); // mark as unusable
    updateParentsAlloc(id);
    return id;
}
```

1、从根节点开始遍历，如果当前节点的 `val<d` ，则通过 `id <<=1` 匹配下一层；
2、如果val > d，则表示存在子节点被分配的情况，而且剩余节点的内存大小不够，此时需要在兄弟节点上继续查找；
3、分配成功的节点需要标记为不可用，防止被再次分配，在memoryMap对应位置更新为12；
4、分配节点完成后，其父节点的状态也需要更新，并可能引起更上一层父节点的更新，实现如下：

```Java
private void updateParentsAlloc(int id) {
    while (id > 1) {
        int parentId = id >>> 1;
        byte val1 = value(id);
        byte val2 = value(id ^ 1);
        byte val = val1 < val2 ? val1 : val2;
        setValue(parentId, val);
        id = parentId;
    }
}
```

比如节点2048被分配出去，更新过程如下：

![img](http://upload-images.jianshu.io/upload_images/2184951-c0bd288e2201fa94.png)

memoryMap节点更新

到目前为止，基于poolChunk的节点分配已经完成。

# 666. 彩蛋

如果你对 Netty 并发感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)