title: 【占小狼】深入浅出 Netty read
date: 2018-01-06
tags:
categories: Netty
permalink: Netty/zhanxiaolang/read
author: 占小狼
from_url: https://www.jianshu.com/p/6b48196b5043
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484643&idx=1&sn=88b9c980992ae39a3581d729856ddeb4&chksm=fa497b52cd3ef2441b58531156f95f3aa6bb5ea3f6c677afe87902f21c78c6569810564650ab#rd

-------

摘要: 原创出处 https://www.jianshu.com/p/6b48196b5043 「占小狼」欢迎转载，保留摘要，谢谢！


-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

boss线程主要负责监听并处理accept事件，将socketChannel注册到work线程的selector，由worker线程来监听并处理read事件，本节主要分析Netty如何处理read事件。

![img](http://upload-images.jianshu.io/upload_images/2184951-3afb05bc34f28479.png)

accept->read

当work线程的selector检测到OP_READ事件发生时，触发read操作。

```Java
//NioEventLoop
if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
    unsafe.read();
    if (!ch.isOpen()) {
        // Connection already closed - no need to handle write.
        return;
    }
}
```

该read方法定义在类NioByteUnsafe中。

```Java
//AbstractNioByteChannel.NioByteUnsafe
public final void read() {
    final ChannelConfig config = config();
    if (!config.isAutoRead() && !isReadPending()) {
        // ChannelConfig.setAutoRead(false) was called in the meantime
        removeReadOp();
        return;
    }

    final ChannelPipeline pipeline = pipeline();
    final ByteBufAllocator allocator = config.getAllocator();
    final int maxMessagesPerRead = config.getMaxMessagesPerRead();
    RecvByteBufAllocator.Handle allocHandle = this.allocHandle;
    if (allocHandle == null) {
        this.allocHandle = allocHandle = config.getRecvByteBufAllocator().newHandle();
    }

    ByteBuf byteBuf = null;
    int messages = 0;
    boolean close = false;
    try {
        int totalReadAmount = 0;
        boolean readPendingReset = false;
        do {
            byteBuf = allocHandle.allocate(allocator);
            int writable = byteBuf.writableBytes();
            int localReadAmount = doReadBytes(byteBuf);
            if (localReadAmount <= 0) {
                // not was read release the buffer
                byteBuf.release();
                byteBuf = null;
                close = localReadAmount < 0;
                break;
            }
            if (!readPendingReset) {
                readPendingReset = true;
                setReadPending(false);
            }
            pipeline.fireChannelRead(byteBuf);
            byteBuf = null;

            if (totalReadAmount >= Integer.MAX_VALUE - localReadAmount) {
                // Avoid overflow.
                totalReadAmount = Integer.MAX_VALUE;
                break;
            }

            totalReadAmount += localReadAmount;

            // stop reading
            if (!config.isAutoRead()) {
                break;
            }

            if (localReadAmount < writable) {
                // Read less than what the buffer can hold,
                // which might mean we drained the recv buffer completely.
                break;
            }
        } while (++ messages < maxMessagesPerRead);

        pipeline.fireChannelReadComplete();
        allocHandle.record(totalReadAmount);

        if (close) {
            closeOnRead(pipeline);
            close = false;
        }
    } catch (Throwable t) {
        handleReadException(pipeline, byteBuf, t, close);
    } finally {
        // Check if there is a readPending which was not processed yet.
        // This could be for two reasons:
        // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
        // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
        //
        // See https://github.com/netty/netty/issues/2254
        if (!config.isAutoRead() && !isReadPending()) {
            removeReadOp();
        }
    }
}
```

1、allocHandle负责自适应调整当前缓存分配的大小，以防止缓存分配过多或过少，先看看AdaptiveRecvByteBufAllocator内部实现：

```Java
public class AdaptiveRecvByteBufAllocator implements RecvByteBufAllocator {
    static final int DEFAULT_MINIMUM = 64;
    static final int DEFAULT_INITIAL = 1024;
    static final int DEFAULT_MAXIMUM = 65536;
    private static final int INDEX_INCREMENT = 4;
    private static final int INDEX_DECREMENT = 1;
    private static final int[] SIZE_TABLE;
}
```

**SIZE_TABLE**：按照从小到大的顺序预先存储可以分配的缓存大小。
从16开始，每次累加16，直到496，接着从512开始，每次增大一倍，直到溢出。
**DEFAULT_MINIMUM**：最小缓存（64），在SIZE_TABLE中对应的下标为3。
**DEFAULT_MAXIMUM **：最大缓存（65536），在SIZE_TABLE中对应的下标为38。
**DEFAULT_INITIAL **：初始化缓存大小，第一次分配缓存时，由于没有上一次实际收到的字节数做参考，需要给一个默认初始值。
**INDEX_INCREMENT**：上次预估缓存偏小，下次index的递增值。
**INDEX_DECREMENT **：上次预估缓存偏大，下次index的递减值。

2、allocHandle.allocate(allocator) 申请一块指定大小的内存。

```Java
//AdaptiveRecvByteBufAllocator.HandleImpl
public ByteBuf allocate(ByteBufAllocator alloc) {
    return alloc.ioBuffer(nextReceiveBufferSize);
}
```

通过ByteBufAllocator的ioBuffer方法申请缓存。

```Java
//AbstractByteBufAllocator
public ByteBuf ioBuffer(int initialCapacity) {
    if (PlatformDependent.hasUnsafe()) {
        return directBuffer(initialCapacity);
    }
    return heapBuffer(initialCapacity);
}
```

根据平台是否支持unsafe，选择使用直接物理内存还是堆上内存。

direct buffer方案：

```Java
//AbstractByteBufAllocator
public ByteBuf directBuffer(int initialCapacity) {
    return directBuffer(initialCapacity, Integer.MAX_VALUE);
}

public ByteBuf directBuffer(int initialCapacity, int maxCapacity) {
    if (initialCapacity == 0 && maxCapacity == 0) {
        return emptyBuf;
    }
    validate(initialCapacity, maxCapacity);
    return newDirectBuffer(initialCapacity, maxCapacity);
}

//UnpooledByteBufAllocator
protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
    ByteBuf buf;
    if (PlatformDependent.hasUnsafe()) {
        buf = new UnpooledUnsafeDirectByteBuf(this, initialCapacity, maxCapacity);
    } else {
        buf = new UnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
    }
    return toLeakAwareBuffer(buf);
}
```

UnpooledUnsafeDirectByteBuf是如何实现缓存管理的？对Nio的ByteBuffer进行了封装，通过ByteBuffer的allocateDirect方法实现缓存的申请。

```Java
protected UnpooledUnsafeDirectByteBuf(ByteBufAllocator alloc, ByteBuffer initialBuffer, int maxCapacity) {
    //判断逻辑已经忽略
   this.alloc = alloc;
   setByteBuffer(allocateDirect(initialCapacity));
}

protected ByteBuffer allocateDirect(int initialCapacity) {
    return ByteBuffer.allocateDirect(initialCapacity);
}

private void setByteBuffer(ByteBuffer buffer) {
    ByteBuffer oldBuffer = this.buffer;
    if (oldBuffer != null) {
        if (doNotFree) {
            doNotFree = false;
        } else {
            freeDirect(oldBuffer);
        }
    }
    this.buffer = buffer;
    memoryAddress = PlatformDependent.directBufferAddress(buffer);
    tmpNioBuf = null;
    capacity = buffer.remaining();
}
```

memoryAddress = PlatformDependent.directBufferAddress(buffer) 获取buffer的address字段值，指向缓存地址。
capacity = buffer.remaining() 获取缓存容量。

方法toLeakAwareBuffer(buf)对申请的buf又进行了一次包装：

```Java
protected static ByteBuf toLeakAwareBuffer(ByteBuf buf) {
    ResourceLeak leak;
    switch (ResourceLeakDetector.getLevel()) {
        case SIMPLE:
            leak = AbstractByteBuf.leakDetector.open(buf);
            if (leak != null) {
                buf = new SimpleLeakAwareByteBuf(buf, leak);
            }
            break;
        case ADVANCED:
        case PARANOID:
            leak = AbstractByteBuf.leakDetector.open(buf);
            if (leak != null) {
                buf = new AdvancedLeakAwareByteBuf(buf, leak);
            }
            break;
    }
    return buf;
}
```

Netty中使用引用计数机制来管理资源，ByteBuf实现了ReferenceCounted接口，当实例化一个ByteBuf时，引用计数为1， 代码中需要保持一个该对象的引用时需要调用retain方法将计数增1，对象使用完时调用release将计数减1。当引用计数变为0时，对象将释放所持有的底层资源或将资源返回资源池。

3、方法doReadBytes(byteBuf) 将socketChannel数据写入缓存。

```Java
//NioSocketChannel
@Override
protected int doReadBytes(ByteBuf byteBuf) throws Exception {
    return byteBuf.writeBytes(javaChannel(), byteBuf.writableBytes());
}

//WrappedByteBuf
@Override
public int writeBytes(ScatteringByteChannel in, int length) throws IOException {
    return buf.writeBytes(in, length);
}

//AbsractByteBuf
@Override
public int writeBytes(ScatteringByteChannel in, int length) throws IOException {
    ensureAccessible();
    ensureWritable(length);
    int writtenBytes = setBytes(writerIndex, in, length);
    if (writtenBytes > 0) {
        writerIndex += writtenBytes;
    }
    return writtenBytes;
}

//UnpooledUnsafeDirectByteBuf
@Override
public int setBytes(int index, ScatteringByteChannel in, int length) throws IOException {
    ensureAccessible();
    ByteBuffer tmpBuf = internalNioBuffer();
    tmpBuf.clear().position(index).limit(index + length);
    try {
        return in.read(tmpBuf);
    } catch (ClosedChannelException ignored) {
        return -1;
    }
}

private ByteBuffer internalNioBuffer() {
    ByteBuffer tmpNioBuf = this.tmpNioBuf;
    if (tmpNioBuf == null) {
        this.tmpNioBuf = tmpNioBuf = buffer.duplicate();
    }
    return tmpNioBuf;
}
```

最终底层采用ByteBuffer实现read操作，这里有一块逻辑不清楚，为什么要用tmpNioBuf？

int localReadAmount = doReadBytes(byteBuf);
1、如果返回0，则表示没有读取到数据，则退出循环。
2、如果返回-1，表示对端已经关闭连接，则退出循环。
3、否则，表示读取到了数据，数据读入缓存后，触发pipeline的ChannelRead事件，byteBuf作为参数进行后续处理，这时自定义Inbound类型的handler就可以进行业务处理了。

```Java
static class DiscardServerHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf in = (ByteBuf) msg;
        try {
            while (in.isReadable()) { // (1)
                System.out.print((char) in.readByte());
                System.out.flush();
            }
        } finally {
            ReferenceCountUtil.release(msg); // (2)
        }
    }
}
```

其中参数msg，就是对应的byteBuf，当请求的数据量比较大时，会多次触发channelRead事件，默认最多触发16次，可以通过maxMessagesPerRead字段进行配置。
如果客户端传输的数据过大，可能会分成好几次传输，因为TCP一次传输内容大小有上限，所以同一个selectKey会触发多次read事件，剩余的数据会在下一轮select操作继续读取。

在实际应用中，应该把所有请求数据都缓存起来再进行业务处理。
所有数据都处理完，触发pipeline的ChannelReadComplete事件，并且allocHandle记录这次read的字节数，进行下次处理时缓存大小的调整。

到此为止，整个NioSocketChannel的read事件已经处理完成。

# 666. 彩蛋

如果你对 Netty 并发感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)