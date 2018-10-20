title: 【占小狼】Netty 源码分析之 accept 过程
date: 2018-01-05
tags:
categories: Netty
permalink: Netty/zhanxiaolang/accept
author: 占小狼
from_url: https://www.jianshu.com/p/ffc6fd82e32b
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484596&idx=1&sn=720f0382f6c6dd99f71f319a285b7450&chksm=fa497b05cd3ef21380900d22a286d6d1473c0775730e53fe03d60bace3c5d7584e17973713e3#rd

-------

摘要: 原创出处 https://www.jianshu.com/p/ffc6fd82e32b 「占小狼」欢迎转载，保留摘要，谢谢！


-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

本章节分析服务端如何accept客户端的connect请求。

在[Netty源码分析之NioEventLoop](https://www.jianshu.com/p/9acf36f7e025)章节中，已经分析了NioEventLoop的工作机制，当有客户端connect请求，selector可以返回其对应的SelectionKey，方法processSelectedKeys进行后续的处理。

![img](http://upload-images.jianshu.io/upload_images/2184951-d9cb43b57e0dfd11.png)

```Java
private void processSelectedKeys() {
    if (selectedKeys != null) {
        processSelectedKeysOptimized(selectedKeys.flip());
    } else {
        processSelectedKeysPlain(selector.selectedKeys());
    }
}
```

默认采用优化过的SelectedSelectionKeySet保存有事件发生的selectedKey。
1、SelectedSelectionKeySet内部使用两个大小为1024的SelectionKey数组keysA和keysB保存selectedKey。
2、把SelectedSelectionKeySet实例映射到selector的原生selectedKeys和publicSelectedKeys。

```Java
private void processSelectedKeysOptimized(SelectionKey[] selectedKeys) {
    for (int i = 0;; i ++) {
        final SelectionKey k = selectedKeys[i];
        if (k == null) {
            break;
        }
        // null out entry in the array to allow to have it GC'ed once the Channel close
        // See https://github.com/netty/netty/issues/2363
        selectedKeys[i] = null;

        final Object a = k.attachment();

        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            @SuppressWarnings("unchecked")
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task);
        }

        if (needsToSelectAgain) {
            // null out entries in the array to allow to have it GC'ed once the Channel close
            // See https://github.com/netty/netty/issues/2363
            for (;;) {
                i++;
                if (selectedKeys[i] == null) {
                    break;
                }
                selectedKeys[i] = null;
            }

            selectAgain();
            // Need to flip the optimized selectedKeys to get the right reference to the array
            // and reset the index to -1 which will then set to 0 on the for loop
            // to start over again.
            //
            // See https://github.com/netty/netty/issues/1523
            selectedKeys = this.selectedKeys.flip();
            i = -1;
        }
    }
}
```

因为selector的I/O多路复用机制，一次可以返回多个selectedKey，所以要用for循环处理全部selectionKey。

假设这时有请求进来，selectedKeys中就存在一个selectionKey，这块逻辑不清楚的可以回头看看[深入浅出Nio Socket](https://www.jianshu.com/p/0d497fe5484a)。
1、通过k.attachment()可以获取ServerSocketChannel注册时绑定上去的附件，其实这个附件就是ServerSocketChannel自身。
2、如果selectedKey的附件是AbstractNioChannel类型的，执行processSelectedKey(k, (AbstractNioChannel) a)方法进行下一步操作。

```Java
private static void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final NioUnsafe unsafe = ch.unsafe();
    if (!k.isValid()) {
        // close the channel if the key is not valid anymore
        unsafe.close(unsafe.voidPromise());
        return;
    }

    try {
        int readyOps = k.readyOps();
        // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
        // to a spin loop
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            unsafe.read();
            if (!ch.isOpen()) {
                // Connection already closed - no need to handle write.
                return;
            }
        }
        if ((readyOps & SelectionKey.OP_WRITE) != 0) {
            // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
            ch.unsafe().forceFlush();
        }
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
            // See https://github.com/netty/netty/issues/924
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);

            unsafe.finishConnect();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}
```

1、获取ServerSocketChannel的unsafe对象。
2、当前selectionKey发生的事件是SelectionKey.OP_ACCEPT，执行unsafe的read方法。

该read方法定义在NioMessageUnsafe类中：

```Java
private final List<Object> readBuf = new ArrayList<Object>();

@Override
public void read() {
    assert eventLoop().inEventLoop();
    final ChannelConfig config = config();
    if (!config.isAutoRead() && !isReadPending()) {
        // ChannelConfig.setAutoRead(false) was called in the meantime
        removeReadOp();
        return;
    }

    final int maxMessagesPerRead = config.getMaxMessagesPerRead();
    final ChannelPipeline pipeline = pipeline();
    boolean closed = false;
    Throwable exception = null;
    try {
        try {
            for (;;) {
                int localRead = doReadMessages(readBuf);
                if (localRead == 0) {
                    break;
                }
                if (localRead < 0) {
                    closed = true;
                    break;
                }

                // stop reading and remove op
                if (!config.isAutoRead()) {
                    break;
                }

                if (readBuf.size() >= maxMessagesPerRead) {
                    break;
                }
            }
        } catch (Throwable t) {
            exception = t;
        }
        setReadPending(false);
        int size = readBuf.size();
        for (int i = 0; i < size; i ++) {
            pipeline.fireChannelRead(readBuf.get(i));
        }

        readBuf.clear();
        pipeline.fireChannelReadComplete();

        if (exception != null) {
            if (exception instanceof IOException && !(exception instanceof PortUnreachableException)) {
                // ServerChannel should not be closed even on IOException because it can often continue
                // accepting incoming connections. (e.g. too many open files)
                closed = !(AbstractNioMessageChannel.this instanceof ServerChannel);
            }

            pipeline.fireExceptionCaught(exception);
        }

        if (closed) {
            if (isOpen()) {
                close(voidPromise());
            }
        }
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

1、readBuf 用来保存客户端NioSocketChannel，默认一次不超过16个。
2、方法doReadMessages进行处理ServerSocketChannel的accept操作。

```Java
protected int doReadMessages(List<Object> buf) throws Exception {
    SocketChannel ch = javaChannel().accept();
    try {
        if (ch != null) {
            buf.add(new NioSocketChannel(this, ch));
            return 1;
        }
    } catch (Throwable t) {
        logger.warn("Failed to create a new channel from an accepted socket.", t);
        try {
            ch.close();
        } catch (Throwable t2) {
            logger.warn("Failed to close a socket.", t2);
        }
    }
    return 0;
}
```

1、javaChannel()返回NioServerSocketChannel对应的ServerSocketChannel。
2、ServerSocketChannel.accept返回客户端的socketChannel 。
3、把 NioServerSocketChannel 和 socketChannel 封装成 NioSocketChannel，并缓存到readBuf。
4、遍历redBuf中的NioSocketChannel，触发各自pipeline的ChannelRead事件，从pipeline的head开始遍历，最终执行ServerBootstrapAcceptor的channelRead方法。

```Java
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    final Channel child = (Channel) msg;
    child.pipeline().addLast(childHandler);
    for (Entry<ChannelOption<?>, Object> e: childOptions) {
        try {
            if (!child.config().setOption((ChannelOption<Object>) e.getKey(), e.getValue())) {
                logger.warn("Unknown channel option: " + e);
            }
        } catch (Throwable t) {
            logger.warn("Failed to set a channel option: " + child, t);
        }
    }
    for (Entry<AttributeKey<?>, Object> e: childAttrs) {
        child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
    }
    try {
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}
```

1、child.pipeline().addLast(childHandler)添加childHandler到NioSocketChannel的pipeline。
其中childHandler是通过ServerBootstrap的childHandler方法进行配置的，和NioServerSocketChannel类似，NioSocketChannel在注册到selector后会触发其pipeline的fireChannelRegistered方法，并执行initChannel方法，为NioSocketChannel的pipeline添加更多自定义的handler，进行业务处理。
2、childGroup.register(child)将NioSocketChannel注册到work的eventLoop中，这个过程和NioServerSocketChannel注册到boss的eventLoop的过程一样，最终由work线程对应的selector进行read事件的监听。

当readBuf中缓存的NioSocketChannel都处理完成后，清空readBuf，并触发ChannelReadComplete。

到此为止，一次accept流程已经执行完。

# 666. 彩蛋

如果你对 Netty 并发感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)