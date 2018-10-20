title: 【占小狼】深入浅出 Netty write
date: 2018-01-07
tags:
categories: Netty
permalink: Netty/zhanxiaolang/write
author: 占小狼
from_url: https://www.jianshu.com/p/1ad424c53e80
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484701&idx=1&sn=8770ddd2786bb6dab4defece6b4d5aa3&chksm=fa497aaccd3ef3baba0a6dc4cae3f774ff83313c39267b66feb84d5c975329c6b32baea5bc09#rd

-------

摘要: 原创出处 https://www.jianshu.com/p/1ad424c53e80 「占小狼」欢迎转载，保留摘要，谢谢！


-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

上一章节中，分析了Netty如何处理read事件，本节分析Netty如何把数据写会客户端。

把数据返回客户端，需要经历三个步骤：
1、申请一块缓存buf，写入数据。
2、将buf保存到ChannelOutboundBuffer中。
3、将ChannelOutboundBuffer中的buff输出到socketChannel中。

```Java
  public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
      ReferenceCountUtil.release(msg);

      ByteBuf buf1 = ctx.alloc().buffer(4);
      buf1.writeInt(1);

      ByteBuf buf2 = ctx.alloc().buffer(4);
      buf2.writeInt(2);

      ByteBuf buf3 = ctx.alloc().buffer(4);
      buf3.writeInt(3);

      ctx.write(buf1);
      ctx.write(buf2);
      ctx.write(buf3);
      ctx.flush();
  }
```

为什么需要把buf保存到ChannelOutboundBuffer？

**ctx.write()**实现：

```Java
//AbstractChannelHandlerContext.java
public ChannelFuture write(Object msg) {
  return write(msg, newPromise());
}

private void write(Object msg, boolean flush, ChannelPromise promise) {
    AbstractChannelHandlerContext next = findContextOutbound();
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeWrite(msg, promise);
        if (flush) {
            next.invokeFlush();
        }
    } else {
        AbstractWriteTask task;
        if (flush) {
            task = WriteAndFlushTask.newInstance(next, msg, promise);
        }  else {
            task = WriteTask.newInstance(next, msg, promise);
        }
        safeExecute(executor, task, promise, msg);
    }
}
```

默认情况下，findContextOutbound()会找到pipeline的head节点，触发write方法。

```Java
//HeadContext.java
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    unsafe.write(msg, promise);
}

//AbstractUnsafe
public final void write(Object msg, ChannelPromise promise) {
    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        safeSetFailure(promise, CLOSED_CHANNEL_EXCEPTION);
        ReferenceCountUtil.release(msg);
        return;
    }

    int size;
    try {
        msg = filterOutboundMessage(msg);
        size = estimatorHandle().size(msg);
        if (size < 0) {
            size = 0;
        }
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        ReferenceCountUtil.release(msg);
        return;
    }

    outboundBuffer.addMessage(msg, size, promise);
}
```

outboundBuffer 随着Unsafe一起实例化，最终将msg通过outboundBuffer封装起来。

**ChannelOutboundBuffer**内部维护了一个Entry链表，并使用Entry封装msg。
1、unflushedEntry：指向链表头部
2、tailEntry：指向链表尾部
3、totalPendingSize：保存msg的字节数
4、unwritable：不可写标识

```Java
public void addMessage(Object msg, int size, ChannelPromise promise) {
    Entry entry = Entry.newInstance(msg, size, total(msg), promise);
    if (tailEntry == null) {
        flushedEntry = null;
        tailEntry = entry;
    } else {
        Entry tail = tailEntry;
        tail.next = entry;
        tailEntry = entry;
    }
    if (unflushedEntry == null) {
        unflushedEntry = entry;
    }

    // increment pending bytes after adding message to the unflushed arrays.
    // See https://github.com/netty/netty/issues/1619
    incrementPendingOutboundBytes(size, false);
}
```

通过Entry.newInstance返回Entry实例，Netty对Entry采用了缓存策略，使用完的Entry实例需要清空并回收，难道是因为Entry实例化比较耗时？

新的entry默认插入链表尾部，并让tailEntry指向它。

![img](http://upload-images.jianshu.io/upload_images/2184951-53e95abefcc0504f.png)

Paste_Image.png

```Java
private void incrementPendingOutboundBytes(long size, boolean invokeLater) {
    if (size == 0) {
        return;
    }
    long newWriteBufferSize = TOTAL_PENDING_SIZE_UPDATER.addAndGet(this, size);
    if (newWriteBufferSize >= channel.config().getWriteBufferHighWaterMark()) {
        setUnwritable(invokeLater);
    }
}
```

方法incrementPendingOutboundBytes主要采用CAS更新totalPendingSize字段，并判断当前totalPendingSize是否超过阈值writeBufferHighWaterMark，默认是65536。如果totalPendingSize >= 65536，则采用CAS更新unwritable为1，并触发ChannelWritabilityChanged事件。

到此为止，全部的buf数据已经保存在outboundBuffer中。

**ctx.flush()**实现：

```Java
public ChannelHandlerContext flush() {
    final AbstractChannelHandlerContext next = findContextOutbound();
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeFlush();
    } else {
        Runnable task = next.invokeFlushTask;
        if (task == null) {
            next.invokeFlushTask = task = new Runnable() {
                @Override
                public void run() {
                    next.invokeFlush();
                }
            };
        }
        safeExecute(executor, task, channel().voidPromise(), null);
    }

    return this;
}
```

默认情况下，findContextOutbound()会找到pipeline的head节点，触发flush方法。

```Java
//HeadContext.java
public void flush(ChannelHandlerContext ctx) throws Exception {
    unsafe.flush();
}

//AbstractUnsafe
public final void flush() {
    assertEventLoop();
    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        return;
    }
    outboundBuffer.addFlush();
    flush0();
}
```

方法addFlush主要对write过程添加的msg进行flush标识，其实我不清楚，这个标识过程有什么意义。

直接看flush0方法：

```Java
protected final void flush0() {
    // Flush immediately only when there's no pending flush.
    // If there's a pending flush operation, event loop will call forceFlush() later,
    // and thus there's no need to call it now.
    if (isFlushPending()) {
        return;
    }
    super.flush0();
}

private boolean isFlushPending() {
    SelectionKey selectionKey = selectionKey();
    return selectionKey.isValid() && (selectionKey.interestOps() & SelectionKey.OP_WRITE) != 0;
}
```

1、如果当前selectionKey 是写事件，说明有线程执行flush过程，则直接返回。
2、否则直接执行flush操作。

```Java
protected void flush0() {
    if (inFlush0) {
        // Avoid re-entrance
        return;
    }

    final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null || outboundBuffer.isEmpty()) {
        return;
    }

    inFlush0 = true;

    // Mark all pending write requests as failure if the channel is inactive.
    if (!isActive()) {
        try {
            if (isOpen()) {
                outboundBuffer.failFlushed(NOT_YET_CONNECTED_EXCEPTION, true);
            } else {
                // Do not trigger channelWritabilityChanged because the channel is closed already.
                outboundBuffer.failFlushed(CLOSED_CHANNEL_EXCEPTION, false);
            }
        } finally {
            inFlush0 = false;
        }
        return;
    }

    try {
        doWrite(outboundBuffer);
    } catch (Throwable t) {
        if (t instanceof IOException && config().isAutoClose()) {
            /**
             * Just call {@link #close(ChannelPromise, Throwable, boolean)} here which will take care of
             * failing all flushed messages and also ensure the actual close of the underlying transport
             * will happen before the promises are notified.
             *
             * This is needed as otherwise {@link #isActive()} , {@link #isOpen()} and {@link #isWritable()}
             * may still return {@code true} even if the channel should be closed as result of the exception.
             */
            close(voidPromise(), t, false);
        } else {
            outboundBuffer.failFlushed(t, true);
        }
    } finally {
        inFlush0 = false;
    }
}

public boolean isActive() {
    SocketChannel ch = javaChannel();
    return ch.isOpen() && ch.isConnected();
}
```

1、如果当前socketChannel已经关闭，或断开连接，则执行失败操作。
2、否则执行doWrite把数据写入到socketChannel。

```Java
protected void doWrite(ChannelOutboundBuffer in) throws Exception {
    for (;;) {
        int size = in.size();
        if (size == 0) {
            // All written so clear OP_WRITE
            clearOpWrite();
            break;
        }
        long writtenBytes = 0;
        boolean done = false;
        boolean setOpWrite = false;

        // Ensure the pending writes are made of ByteBufs only.
        ByteBuffer[] nioBuffers = in.nioBuffers();
        int nioBufferCnt = in.nioBufferCount();
        long expectedWrittenBytes = in.nioBufferSize();
        SocketChannel ch = javaChannel();

        // Always us nioBuffers() to workaround data-corruption.
        // See https://github.com/netty/netty/issues/2761
        switch (nioBufferCnt) {
            case 0:
                // We have something else beside ByteBuffers to write so fallback to normal writes.
                super.doWrite(in);
                return;
            case 1:
                // Only one ByteBuf so use non-gathering write
                ByteBuffer nioBuffer = nioBuffers[0];
                for (int i = config().getWriteSpinCount() - 1; i >= 0; i --) {
                    final int localWrittenBytes = ch.write(nioBuffer);
                    if (localWrittenBytes == 0) {
                        setOpWrite = true;
                        break;
                    }
                    expectedWrittenBytes -= localWrittenBytes;
                    writtenBytes += localWrittenBytes;
                    if (expectedWrittenBytes == 0) {
                        done = true;
                        break;
                    }
                }
                break;
            default:
                for (int i = config().getWriteSpinCount() - 1; i >= 0; i --) {
                    final long localWrittenBytes = ch.write(nioBuffers, 0, nioBufferCnt);
                    if (localWrittenBytes == 0) {
                        setOpWrite = true;
                        break;
                    }
                    expectedWrittenBytes -= localWrittenBytes;
                    writtenBytes += localWrittenBytes;
                    if (expectedWrittenBytes == 0) {
                        done = true;
                        break;
                    }
                }
                break;
        }

        // Release the fully written buffers, and update the indexes of the partially written buffer.
        in.removeBytes(writtenBytes);

        if (!done) {
            // Did not write all buffers completely.
            incompleteWrite(setOpWrite);
            break;
        }
    }
}
```

1、size方法返回outboundBuffer有多少Entry实例。
2、in.nioBuffers()负责把Entry中保存的ByteBuf类型的msg，重新返回Nio的ByteBuffer实例，并返回ByteBuffer数组nioBuffers，其实msg和ByteBuffer实例指向的是同一块内存，因为在UnpooledDirectByteBuf实现类中，已经维护了ByteBuffer的实例。
3、socketChannel.write()方法把nioBuffers的数据写到socket中，这是Nio中的实现。

到此为止，nioBuffers的数据都flush到socket，客户端可以准备接收了。

# 666. 彩蛋

如果你对 Netty 并发感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)