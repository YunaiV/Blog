title: 【占小狼】Netty 源码分析之 ChannelPipeline
date: 2018-01-04
tags:
categories: Netty
permalink: Netty/zhanxiaolang/ChannelPipeline
author: 占小狼
from_url: https://www.jianshu.com/p/3876874306d5
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484542&idx=1&sn=ca29470859ad4ec78194cecd16749236&chksm=fa497bcfcd3ef2d9d76ef384f69978d65ea2658e9bbaaf8f8e922984b8bf21f3d0dc42704377#rd

-------

摘要: 原创出处 https://www.jianshu.com/p/3876874306d5 「占小狼」欢迎转载，保留摘要，谢谢！

- [DefaultChannelPipeline](http://www.iocoder.cn/Netty/zhanxiaolang/ChannelPipeline/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

本章节分析Netty中的ChannelPipeline模块。

每个channel内部都会持有一个ChannelPipeline对象pipeline.
pipeline默认实现DefaultChannelPipeline内部维护了一个DefaultChannelHandlerContext链表。

![img](http://upload-images.jianshu.io/upload_images/2184951-beacd91367f1f4eb.png)

ChannelPipeline

当channel完成register、active、read等操作时，会触发pipeline的相应方法。
1、当channel注册到selector时，触发pipeline的fireChannelRegistered方法。
2、当channel的socket绑定完成时，触发pipeline的fireChannelActive方法。
3、当有客户端请求时，触发pipeline的fireChannelRead方法。
4、当本次客户端请求，pipeline执行完fireChannelRead，触发pipeline的fireChannelReadComplete方法。

接下去看看pipeline是如何组织并运行handler对应的方法。

### DefaultChannelPipeline

其中DefaultChannelHandlerContext保存了当前handler的上下文，如channel、pipeline等信息，默认实现了head和tail。

```Java
class DefaultChannelPipeline implements ChannelPipeline {
    final Channel channel; // pipeline所属的channel
    //head和tail都是handler上下文
    final DefaultChannelHandlerContext head;
    final DefaultChannelHandlerContext tail;
    ...
    public DefaultChannelPipeline(AbstractChannel channel) {
        if (channel == null) {
            throw new NullPointerException("channel");
        }
        this.channel = channel;

        tail = new TailContext(this);
        head = new HeadContext(this);

        head.next = tail;
        tail.prev = head;
    }
}
```

1、TailContext实现了ChannelOutboundHandler接口。
2、HeadContext实现了ChannelInboundHandler接口。
3、head和tail形成了一个链表。

对于Inbound的操作，当channel注册到selector时，触发pipeline的fireChannelRegistered，从head开始遍历，找到实现了ChannelInboundHandler接口的handler，并执行其fireChannelRegistered方法。

```Java
@Override
public ChannelPipeline fireChannelRegistered() {
    head.fireChannelRegistered();
    return this;
}

@Override
public ChannelHandlerContext fireChannelRegistered() {
    final DefaultChannelHandlerContext next = findContextInbound();
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeChannelRegistered();
    } else {
        executor.execute(new Runnable() {
            @Override
            public void run() {
                next.invokeChannelRegistered();
            }
        });
    }
    return this;
}

private DefaultChannelHandlerContext findContextInbound() {
    DefaultChannelHandlerContext ctx = this;
    do {
        ctx = ctx.next;
    } while (!(ctx.handler() instanceof ChannelInboundHandler));
    return ctx;
}

private void invokeChannelRegistered() {
    try {
        ((ChannelInboundHandler) handler()).channelRegistered(this);
    } catch (Throwable t) {
        notifyHandlerException(t);
    }
}
```

假如我们通过pipeline的addLast方法添加一个inboundHandler实现。

```Java
public class ClientHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRegistered(ChannelHandlerContext ctx)
            throws Exception {
        super.channelRegistered(ctx);
        System.out.println(" ClientHandler  registered channel ");
    }
}
```

当channel注册完成时会触发pipeline的channelRegistered方法，从head开始遍历，找到ClientHandler，并执行channelRegistered方法。

对于Outbound的操作，则从tail向前遍历，找到实现ChannelOutboundHandler接口的handler，具体实现和Inbound一样。

服务启动过程中，ServerBootstrap在init方法中，会给ServerSocketChannel的pipeline添加ChannelInitializer对象，其中ChannelInitializer继承ChannelInboundHandlerAdapter，并实现了ChannelInboundHandler接口，所以当ServerSocketChannel注册到selector之后，会触发其channelRegistered方法。

```Java
public final void channelRegistered(ChannelHandlerContext ctx) throws Exception {
    initChannel((C) ctx.channel());
    ctx.pipeline().remove(this);
    ctx.fireChannelRegistered();
}

public void initChannel(Channel ch) throws Exception {
    ChannelPipeline pipeline = ch.pipeline();
    ChannelHandler handler = handler();
    if (handler != null) {
        pipeline.addLast(handler);
    }
    pipeline.addLast(new ServerBootstrapAcceptor(
            currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
}
```

在initChannel实现中，添加ServerBootstrapAcceptor实例到pipeline中。

ServerBootstrapAcceptor继承自ChannelInboundHandlerAdapter，负责把接收到的客户端socketChannel注册到childGroup中，由childGroup中的eventLoop负责数据处理。

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

# 666. 彩蛋

如果你对 Netty 并发感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)