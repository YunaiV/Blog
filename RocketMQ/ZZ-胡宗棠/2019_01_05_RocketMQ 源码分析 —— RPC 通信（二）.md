title: RocketMQ 源码分析 —— RPC 通信（二）
date: 2019-01-05
tags:
categories: RocketMQ
permalink: RocketMQ/huzhongtang/rpc-2
author: 胡宗棠
from_url: https://mp.weixin.qq.com/s/iJww26xFSwEytoz8NjpFRw
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484740&idx=2&sn=5b2ef8833fa2b5c0074da5009ff8905c&chksm=fa497af5cd3ef3e3d24dd33779ef1a9a134599776281e22aeb851be46430ae91687cfb559d3f#rd

-------

摘要: 原创出处 https://mp.weixin.qq.com/s/iJww26xFSwEytoz8NjpFRw 「胡宗棠」欢迎转载，保留摘要，谢谢！

- [一、为何要使用Netty作为高性能的通信库？](http://www.iocoder.cn/RocketMQ/huzhongtang/rpc-2/)
- [二、RocketMQ中RPC通信的Netty多线程模型](http://www.iocoder.cn/RocketMQ/huzhongtang/rpc-2/)
  - [2.1、Netty的Reactor多线程模型设计概念与简述](http://www.iocoder.cn/RocketMQ/huzhongtang/rpc-2/)
  - [2.2、RocketMQ中RPC通信的1+N+M1+M2的Reactor多线程设计与实现](http://www.iocoder.cn/RocketMQ/huzhongtang/rpc-2/)
- [三、总结](http://www.iocoder.cn/RocketMQ/huzhongtang/rpc-2/)
- [666. 彩蛋](http://www.iocoder.cn/RocketMQ/huzhongtang/rpc-2/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

**文章摘要：如何设计RPC通信层模型是任何一款性能强劲的MQ所要重点考虑的问题**

在（一）篇中主要介绍了RocketMQ的协议格式，消息编解码，通信方式(同步/异步/单向)、消息发送/接收以及异步回调的主要通信流程。而本篇将主要对RocketMQ消息队列RPC通信部分的Netty多线程模型进行重点介绍。

# 一、为何要使用Netty作为高性能的通信库？

在看RocketMQ的RPC通信部分时候，可能有不少同学有这样子的疑问，RocketMQ为何要选择Netty而不直接使用JDK的NIO进行网络编程呢？这里有必要先来简要介绍下Netty。
Netty是一个封装了JDK的NIO库的高性能网络通信开源框架。它提供异步的、事件驱动的网络应用程序框架和工具，用以快速开发高性能、高可靠性的网络服务器和客户端程序。
下面主要列举了下一般系统的RPC通信模块会选择Netty作为底层通信库的理由（作者认为RocketMQ的RPC同样也是基于此选择了Netty）：

（1）Netty的编程API使用简单，开发门槛低，无需编程者去关注和了解太多的NIO编程模型和概念；

（2）对于编程者来说，可根据业务的要求进行定制化地开发，通过Netty的ChannelHandler对通信框架进行灵活的定制化扩展；

（3）Netty框架本身支持拆包/解包，异常检测等机制，让编程者可以从JAVA NIO的繁琐细节中解脱，而只需要关注业务处理逻辑；

（4）Netty解决了（准确地说应该是采用了另一种方式完美规避了）JDK NIO的Bug（Epoll bug，会导致Selector空轮询，最终导致CPU 100%）；

（5）Netty框架内部对线程，selector做了一些细节的优化，精心设计的reactor多线程模型，可以实现非常高效地并发处理；

（6）Netty已经在多个开源项目（Hadoop的RPC框架avro使用Netty作为通信框架）中都得到了充分验证，健壮性/可靠性比较好。

# 二、RocketMQ中RPC通信的Netty多线程模型

RocketMQ的RPC通信部分采用了**"1+N+M1+M2"**的Reactor多线程模式，对网络通信部分进行了一定的扩展与优化，这一节主要让我们来看下这一部分的具体设计与实现内容。

## 2.1、Netty的Reactor多线程模型设计概念与简述

这里有必要先来简要介绍下Netty的Reactor多线程模型。Reactor多线程模型的设计思想是分而治之+事件驱动。

**（1）分而治之**
一般来说，一个网络请求连接的完整处理过程可以分为接受（accept）、数据读取（read）、解码/编码（decode/encode）、业务处理（process）、发送响应（send）这几步骤。Reactor模型将每个步骤都映射成为一个任务，服务端线程执行的最小逻辑单元不再是一次完整的网络请求，而是这个任务，且采用以非阻塞方式执行。

**（2）事件驱动**
每个任务对应特定网络事件。当任务准备就绪时，Reactor收到对应的网络事件通知，并将任务分发给绑定了对应网络事件的Handler执行。

## 2.2、RocketMQ中RPC通信的1+N+M1+M2的Reactor多线程设计与实现

**（1）RocketMQ中RPC通信的Reactor多线程设计与流程**
RocketMQ的RPC通信采用Netty组件作为底层通信库，同样也遵循了Reactor多线程模型，同时又在这之上做了一些扩展和优化。下面先给出一张RocketMQ的RPC通信层的Netty多线程模型框架图，让大家对RocketMQ的RPC通信中的多线程分离设计有一个大致的了解。
![RocketMQ的RPC通信层—1+N+M1+M2模型.png](https://upload-images.jianshu.io/upload_images/4325076-04942d70483f28d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
从上面的框图中可以大致了解RocketMQ中NettyRemotingServer的Reactor 多线程模型。一个 **Reactor 主线程（eventLoopGroupBoss，即为上面的1）**负责监听 TCP网络连接请求，建立好连接后丢给**Reactor 线程池（eventLoopGroupSelector，即为上面的“N”，源码中默认设置为3）**，它负责将建立好连接的socket 注册到 selector上去（RocketMQ的源码中会自动根据OS的类型选择NIO和Epoll，也可以通过参数配置），然后监听真正的网络数据。拿到网络数据后，再丢给**Worker线程池（defaultEventExecutorGroup，即为上面的“M1”，源码中默认设置为8）**。
为了更为高效的处理RPC的网络请求，这里的Worker线程池是专门用于处理Netty网络通信相关的（包括编码/解码、空闲链接管理、网络连接管理以及网络请求处理）。而处理业务操作放在业务线程池中执行（这个内容在“RocketMQ的RPC通信（一）篇”中也有提到），根据 RomotingCommand 的业务请求码code去processorTable这个本地缓存变量中找到对应的 processor，然后封装成task任务后，提交给对应的**业务processor处理线程池来执行（sendMessageExecutor，以发送消息为例，即为上面的 “M2”）**。
下面以表格的方式列举了下上面所述的“1+N+M1+M2”Reactor多线程模型

| 线程数 | 线程名 |          线程具体说明          |
| :----: | :----: | :----------------------------: |
|        |   1    |          NettyBoss_%d          |
|        |   N    | NettyServerEPOLLSelector_%d_%d |
|        |   M1   |   NettyServerCodecThread_%d    |
|        |   M2   |   RemotingExecutorThread_%d    |

**（2）RocketMQ中RPC通信的Reactor多线程的代码具体实现**
说完了Reactor多线程整体的设计与流程，大家应该就对RocketMQ的RPC通信的Netty部分有了一个比较全面的理解了，那接下来就从源码上来看下一些细节部分（**在看该部分代码时候需要读者对JAVA NIO和Netty的相关概念与技术点有所了解**）。
在NettyRemotingServer的实例初始化时，会初始化各个相关的变量包括serverBootstrap、nettyServerConfig参数、channelEventListener监听器并同时初始化eventLoopGroupBoss和eventLoopGroupSelector两个Netty的EventLoopGroup线程池（**这里需要注意的是，如果是Linux平台，并且开启了native epoll，就用EpollEventLoopGroup，这个也就是用JNI，调的c写的epoll；否则，就用Java NIO的NioEventLoopGroup。**），具体代码如下：

```java
public NettyRemotingServer(final NettyServerConfig nettyServerConfig,
        final ChannelEventListener channelEventListener) {
        super(nettyServerConfig.getServerOnewaySemaphoreValue(), nettyServerConfig.getServerAsyncSemaphoreValue());
        this.serverBootstrap = new ServerBootstrap();
        this.nettyServerConfig = nettyServerConfig;
        this.channelEventListener = channelEventListener;
      //省略部分代码
      //初始化时候nThreads设置为1,说明RemotingServer端的Disptacher链接管理和分发请求的线程为1,用于接收客户端的TCP连接
        this.eventLoopGroupBoss = new NioEventLoopGroup(1, new ThreadFactory() {
            private AtomicInteger threadIndex = new AtomicInteger(0);

            @Override
            public Thread newThread(Runnable r) {
                return new Thread(r, String.format("NettyBoss_%d", this.threadIndex.incrementAndGet()));
            }
        });

        /**
         * 根据配置设置NIO还是Epoll来作为Selector线程池
         * 如果是Linux平台，并且开启了native epoll，就用EpollEventLoopGroup，这个也就是用JNI，调的c写的epoll；否则，就用Java NIO的NioEventLoopGroup。
         *
         */
        if (useEpoll()) {
            this.eventLoopGroupSelector = new EpollEventLoopGroup(nettyServerConfig.getServerSelectorThreads(), new ThreadFactory() {
                private AtomicInteger threadIndex = new AtomicInteger(0);
                private int threadTotal = nettyServerConfig.getServerSelectorThreads();

                @Override
                public Thread newThread(Runnable r) {
                    return new Thread(r, String.format("NettyServerEPOLLSelector_%d_%d", threadTotal, this.threadIndex.incrementAndGet()));
                }
            });
        } else {
            this.eventLoopGroupSelector = new NioEventLoopGroup(nettyServerConfig.getServerSelectorThreads(), new ThreadFactory() {
                private AtomicInteger threadIndex = new AtomicInteger(0);
                private int threadTotal = nettyServerConfig.getServerSelectorThreads();

                @Override
                public Thread newThread(Runnable r) {
                    return new Thread(r, String.format("NettyServerNIOSelector_%d_%d", threadTotal, this.threadIndex.incrementAndGet()));
                }
            });
        }
        //省略部分代码
```

在NettyRemotingServer实例初始化完成后，就会将其启动。Server端在启动阶段会将之前实例化好的1个acceptor线程（eventLoopGroupBoss），N个IO线程（eventLoopGroupSelector），M1个worker 线程（defaultEventExecutorGroup）绑定上去。前面部分也已经介绍过各个线程池的作用了。
这里需要说明的是，Worker线程拿到网络数据后，就交给Netty的ChannelPipeline（其采用责任链设计模式），从Head到Tail的一个个Handler执行下去，这些 Handler是在创建NettyRemotingServer实例时候指定的。NettyEncoder和NettyDecoder 负责网络传输数据和 RemotingCommand 之间的编解码。NettyServerHandler 拿到解码得到的 RemotingCommand 后，根据 RemotingCommand.type 来判断是 request 还是 response来进行相应处理，根据业务请求码封装成不同的task任务后，提交给对应的业务processor处理线程池处理。

```java
 @Override
    public void start() {
        //默认的处理线程池组,使用默认的处理线程池组用于处理后面的多个Netty Handler的逻辑操作

        this.defaultEventExecutorGroup = new DefaultEventExecutorGroup(
                nettyServerConfig.getServerWorkerThreads(),
                new ThreadFactory() {

                    private AtomicInteger threadIndex = new AtomicInteger(0);

                    @Override
                    public Thread newThread(Runnable r) {
                        return new Thread(r, "NettyServerCodecThread_" + this.threadIndex.incrementAndGet());
                    }
                });
        /**
         * 首先来看下 RocketMQ NettyServer 的 Reactor 线程模型，
         * 一个 Reactor 主线程负责监听 TCP 连接请求;
         * 建立好连接后丢给 Reactor 线程池，它负责将建立好连接的 socket 注册到 selector
         * 上去（这里有两种方式，NIO和Epoll，可配置），然后监听真正的网络数据;
         * 拿到网络数据后，再丢给 Worker 线程池;
         *
         */
        //RocketMQ-> Java NIO的1+N+M模型：1个acceptor线程，N个IO线程，M1个worker 线程。
        ServerBootstrap childHandler =
                this.serverBootstrap.group(this.eventLoopGroupBoss, this.eventLoopGroupSelector)
                        .channel(useEpoll() ? EpollServerSocketChannel.class : NioServerSocketChannel.class)
                        .option(ChannelOption.SO_BACKLOG, 1024)
                        //服务端处理客户端连接请求是顺序处理的，所以同一时间只能处理一个客户端连接，多个客户端来的时候，服务端将不能处理的客户端连接请求放在队列中等待处理，backlog参数指定了队列的大小
                        .option(ChannelOption.SO_REUSEADDR, true)//这个参数表示允许重复使用本地地址和端口
                        .option(ChannelOption.SO_KEEPALIVE, false)//当设置该选项以后，如果在两小时内没有数据的通信时,TCP会自动发送一个活动探测数据报文。
                        .childOption(ChannelOption.TCP_NODELAY, true)//该参数的作用就是禁止使用Nagle算法，使用于小数据即时传输
                        .childOption(ChannelOption.SO_SNDBUF, nettyServerConfig.getServerSocketSndBufSize())//这两个参数用于操作接收缓冲区和发送缓冲区
                        .childOption(ChannelOption.SO_RCVBUF, nettyServerConfig.getServerSocketRcvBufSize())
                        .localAddress(new InetSocketAddress(this.nettyServerConfig.getListenPort()))
                        .childHandler(new ChannelInitializer<SocketChannel>() {
                            @Override
                            public void initChannel(SocketChannel ch) throws Exception {

                                ch.pipeline()
                                        .addLast(defaultEventExecutorGroup, HANDSHAKE_HANDLER_NAME,
                                                new HandshakeHandler(TlsSystemConfig.tlsMode))
                                        .addLast(defaultEventExecutorGroup,
                                                new NettyEncoder(),//rocketmq解码器,他们分别覆盖了父类的encode和decode方法
                                                new NettyDecoder(),//rocketmq编码器
                                                new IdleStateHandler(0, 0, nettyServerConfig.getServerChannelMaxIdleTimeSeconds()),//Netty自带的心跳管理器
                                                new NettyConnectManageHandler(),//连接管理器，他负责捕获新连接、连接断开、异常等事件，然后统一调度到NettyEventExecuter处理器处理。
                                                new NettyServerHandler()//当一个消息经过前面的解码等步骤后，然后调度到channelRead0方法，然后根据消息类型进行分发
                                        );
                            }
                        });

        if (nettyServerConfig.isServerPooledByteBufAllocatorEnable()) {
            childHandler.childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
        }

        try {
            ChannelFuture sync = this.serverBootstrap.bind().sync();
            InetSocketAddress addr = (InetSocketAddress) sync.channel().localAddress();
            this.port = addr.getPort();
        } catch (InterruptedException e1) {
            throw new RuntimeException("this.serverBootstrap.bind().sync() InterruptedException", e1);
        }

        if (this.channelEventListener != null) {
            this.nettyEventExecutor.start();
        }

        //定时扫描responseTable,获取返回结果,并且处理超时
        this.timer.scheduleAtFixedRate(new TimerTask() {

            @Override
            public void run() {
                try {
                    NettyRemotingServer.this.scanResponseTable();
                } catch (Throwable e) {
                    log.error("scanResponseTable exception", e);
                }
            }
        }, 1000 * 3, 1000);
    }
```

从上面的描述中可以概括得出RocketMQ的RPC通信部分的Reactor线程池模型框图。
![RocketMQ的RPC通信层—Reactor线程池.png](https://upload-images.jianshu.io/upload_images/4325076-08a0ff9259f6354b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
整体可以看出RocketMQ的RPC通信借助Netty的多线程模型，其服务端监听线程和IO线程分离，同时将RPC通信层的业务逻辑与处理具体业务的线程进一步相分离。时间可控的简单业务都直接放在RPC通信部分来完成，复杂和时间不可控的业务提交至后端业务线程池中处理，这样提高了通信效率和MQ整体的性能。（ps：其中抽象出NioEventLoop来表示一个不断循环执行处理任务的线程，每个NioEventLoop有一个selector，用于监听绑定在其上的socket链路。）

# 三、总结

仔细阅读RocketMQ的过程中收获了很多关于网络通信设计技术和知识点。对于刚接触开源版的RocketMQ的童鞋来说，想要自己掌握RPC通信部分的各个技术知识点，还需要不断地使用本地环境进行debug调试和阅读源码反复思考。限于笔者的才疏学浅，对本文内容可能还有理解不到位的地方，如有阐述不合理之处还望留言一起探讨。后续还会陆续发布RocketMQ其他模块（Client、Broker和NameServer等）的相关技术文章，敬请关注。
在此顺便为自己打个Call，有兴趣的朋友可以关注下我的个人公众号：“匠心独运的博客”，对于Java并发、Spring、数据库和消息队列的一些细节、问题的文章将会在这个公众号上发布，欢迎交流与讨论。

# 666. 彩蛋

如果你对 Dubbo 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)