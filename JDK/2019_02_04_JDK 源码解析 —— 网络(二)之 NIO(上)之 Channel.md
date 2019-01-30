title: JDK 源码解析 —— 网络(二)之 NIO(上)之 Channel
date: 2019-02-04
tags:
categories: JDK 源码
permalink: JDK/net-2-nio-1
author: 一叶知秋
from_url: http://t.cn/Eq9VCbj
wechat_url:

-------

摘要: 原创出处 http://t.cn/Eq9VCbj 「一叶知秋」欢迎转载，保留摘要，谢谢！

- [场景代入](http://www.iocoder.cn/JDK/net-2-nio-1/)
- [Channel解读](http://www.iocoder.cn/JDK/net-2-nio-1/)
  - [赋予Channel可异步可中断的能力](http://www.iocoder.cn/JDK/net-2-nio-1/)
  - [赋予Channel可被多路复用的能力](http://www.iocoder.cn/JDK/net-2-nio-1/)
  - [赋予Channel支持网络socket的能力](http://www.iocoder.cn/JDK/net-2-nio-1/)
  - [NIO包下SocketChannel解读](http://www.iocoder.cn/JDK/net-2-nio-1/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# 场景代入
接上一篇 [《JDK 源码解析 —— 网络(一)之 BIO》](http://www.iocoder.cn/JDK/net-1-bio)，我们来接触NIO的一些事儿。

在上一篇中，我们可以看到，我们要做到异步非阻塞，我们自己进行的是创建线程池同时对部分代码做timeout的修改来对接客户端，但是弊端也很清晰，我们转换下思维，这里举个场景例子，A班同学要和B班同学一起一对一完成任务，每对人拿到的任务是不一样的，消耗的时间有长有短，任务因为有奖励所以同学们会抢，传统模式下，A班同学和B班同学不经管理话，即便只是一个心跳检测的任务都得一起，在这种情况下，客户端根本不会有数据要发送，只是想告诉服务器自己还活着，这种情况下，假如B班再来一个同学做对接的话，就很有问题了，B班的每一个同学都可以看成服务器端的一个线程。所以，我们需要一个管理者，于是`Selector`就出现了，作为管理者，这里，我们往往需要管理同学们的状态，是否在等待任务，是否在接收信息，是否在输出信息等等，`Selector`更侧重于动作，针对于这些状态标签来做事情就可以了，那这些状态标签其实也是需要管理的，于是`SelectionKey`也就应运而生。接着我们需要对这些同学进行包装增强，使之携带这样的标签。同样，对于同学我们应该进一步解放双手的，比如给其配台电脑，这样，同学是不是可以做更多的事情了，那这个电脑在此处就是Buffer的存在了。
于是在NIO中最主要是有三种角色的，`Buffer`缓冲区，`Channel`通道，`Selector`选择器，我们都涉及到了，接下来，我们对其源码一步步分析解读。

# Channel解读
## 赋予Channel可异步可中断的能力
有上可知，同学其实都是代表着一个个的`Socket`的存在，那么这里`Channel`就是对其进行的增强包装，也就是`Channel`的具体实现里应该有`Socket`这个字段才行，然后具体实现类里面也是紧紧围绕着`Socket`具备的功能来做文章的。那么，我们首先来看`java.nio.channels.Channel`接口的设定:
``` java
public interface Channel extends Closeable {

    /**
     * Tells whether or not this channel is open.
     *
     * @return {@code true} if, and only if, this channel is open
     */
    public boolean isOpen();

    /**
     * Closes this channel.
     *
     * <p> After a channel is closed, any further attempt to invoke I/O
     * operations upon it will cause a {@link ClosedChannelException} to be
     * thrown.
     *
     * <p> If this channel is already closed then invoking this method has no
     * effect.
     *
     * <p> This method may be invoked at any time.  If some other thread has
     * already invoked it, however, then another invocation will block until
     * the first invocation is complete, after which it will return without
     * effect. </p>
     *
     * @throws  IOException  If an I/O error occurs
     */
    public void close() throws IOException;

}
```
此处就是很直接的设定，判断Channel是否是open状态，关闭Channel的动作，我们在接下来会讲到`ClosedChannelException`是如何具体在代码中发生的。
有时候，一个Channel可能会被异步关闭和中断，这也是我们所需求的。那么要实现这个效果我们须得设定一个可以进行此操作效果的接口。达到的具体的效果应该是如果线程在实现这个接口的的Channel中进行IO操作的时候，另一个线程可以调用该Channel的close方法。导致的结果就是，进行IO操作的那个阻塞线程会收到一个`AsynchronousCloseException`异常。

同样，我们应该考虑到另一种情况，如果线程在实现这个接口的的Channel中进行IO操作的时候，另一个线程可能会调用被阻塞线程的`interrupt`方法(`Thread#interrupt()`)，从而导致Channel关闭，那么这个阻塞的线程应该要收到`ClosedByInterruptException`异常，同时将中断状态设定到该阻塞线程之上。

这时候，如果中断状态已经在该线程设定完毕，此时在其之上的有Channel又调用了IO阻塞操作，那么，这个Channel会被关闭，同时，该线程会立即受到一个`ClosedByInterruptException`异常，它的interrupt状态仍然保持不变。
这个接口定义如下:
``` java
public interface InterruptibleChannel
    extends Channel
{

    /**
     * Closes this channel.
     *
     * <p> Any thread currently blocked in an I/O operation upon this channel
     * will receive an {@link AsynchronousCloseException}.
     *
     * <p> This method otherwise behaves exactly as specified by the {@link
     * Channel#close Channel} interface.  </p>
     *
     * @throws  IOException  If an I/O error occurs
     */
    public void close() throws IOException;

}
```
其针对上面所提到逻辑的具体实现是在`java.nio.channels.spi.AbstractInterruptibleChannel`进行的，关于这个类的解析，我们来参考这篇文章[InterruptibleChannel 与可中断 IO](https://github.com/muyinchen/woker/blob/master/NIO/%E8%A1%A5%E5%85%85%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB%EF%BC%9AInterruptibleChannel%20%E4%B8%8E%E5%8F%AF%E4%B8%AD%E6%96%AD%20IO.md)

## 赋予Channel可被多路复用的能力

我们在前面有说到，`Channel`可以被`Selector`进行使用，而`Selector`是根据`Channel`的状态来分配任务的，那么`Channel`应该提供一个注册到`Selector`上的方法，来和`Selector`进行绑定。也就是说`Channel`的实例要调用`register(Selector,int,Object)`。注意，因为`Selector`是要根据状态值进行管理的，所以此方法会返回一个`SelectionKey`对象来表示这个`channel`在`selector`上的状态。关于`SelectionKey`，它是包含很多东西的，这里暂不提。
``` java
//java.nio.channels.spi.AbstractSelectableChannel#register
public final SelectionKey register(Selector sel, int ops, Object att)
        throws ClosedChannelException
    {
        if ((ops & ~validOps()) != 0)
            throw new IllegalArgumentException();
        if (!isOpen())
            throw new ClosedChannelException();
        synchronized (regLock) {
            if (isBlocking())
                throw new IllegalBlockingModeException();
            synchronized (keyLock) {
                // re-check if channel has been closed
                if (!isOpen())
                    throw new ClosedChannelException();
                SelectionKey k = findKey(sel);
                if (k != null) {
                    k.attach(att);
                    k.interestOps(ops);
                } else {
                    // New registration
                    k = ((AbstractSelector)sel).register(this, ops, att);
                    addKey(k);
                }
                return k;
            }
        }
    }
//java.nio.channels.spi.AbstractSelectableChannel#addKey
    private void addKey(SelectionKey k) {
        assert Thread.holdsLock(keyLock);
        int i = 0;
        if ((keys != null) && (keyCount < keys.length)) {
            // Find empty element of key array
            for (i = 0; i < keys.length; i++)
                if (keys[i] == null)
                    break;
        } else if (keys == null) {
            keys = new SelectionKey[2];
        } else {
            // Grow key array
            int n = keys.length * 2;
            SelectionKey[] ks =  new SelectionKey[n];
            for (i = 0; i < keys.length; i++)
                ks[i] = keys[i];
            keys = ks;
            i = keyCount;
        }
        keys[i] = k;
        keyCount++;
    }
```
一旦注册到`Selector`上，Channel将一直保持注册直到其被解除注册。在解除注册的时候会解除Selector分配给Channel的所有资源。
也就是Channel并没有直接提供解除注册的方法，那我们换一个思路，我们将Selector上代表其注册的Key取消不就可以了。这里可以通过调用`SelectionKey#cancel()`方法来显式的取消key。然后在`Selector`下一次选择操作期间进行对Channel的取消注册。
``` java
//java.nio.channels.spi.AbstractSelectionKey#cancel
    /**
     * Cancels this key.
     *
     * <p> If this key has not yet been cancelled then it is added to its
     * selector's cancelled-key set while synchronized on that set.  </p>
     */
    public final void cancel() {
        // Synchronizing "this" to prevent this key from getting canceled
        // multiple times by different threads, which might cause race
        // condition between selector's select() and channel's close().
        synchronized (this) {
            if (valid) {
                valid = false;
                //还是调用Selector的cancel方法
                ((AbstractSelector)selector()).cancel(this);
            }
        }
    }


//java.nio.channels.spi.AbstractSelector#cancel
    void cancel(SelectionKey k) {
        synchronized (cancelledKeys) {
            cancelledKeys.add(k);
        }
    }


//在下一次select操作的时候来解除那些要求cancel的key，即解除Channel注册
//sun.nio.ch.SelectorImpl#select(long)
    @Override
    public final int select(long timeout) throws IOException {
        if (timeout < 0)
            throw new IllegalArgumentException("Negative timeout");
            //重点关注此方法
        return lockAndDoSelect(null, (timeout == 0) ? -1 : timeout);
    }
//sun.nio.ch.SelectorImpl#lockAndDoSelect
    private int lockAndDoSelect(Consumer<SelectionKey> action, long timeout)
        throws IOException
    {
        synchronized (this) {
            ensureOpen();
            if (inSelect)
                throw new IllegalStateException("select in progress");
            inSelect = true;
            try {
                synchronized (publicSelectedKeys) {
                    //重点关注此方法
                    return doSelect(action, timeout);
                }
            } finally {
                inSelect = false;
            }
        }
    }
//sun.nio.ch.WindowsSelectorImpl#doSelect
    protected int doSelect(Consumer<SelectionKey> action, long timeout)
        throws IOException
    {
        assert Thread.holdsLock(this);
        this.timeout = timeout; // set selector timeout
        processUpdateQueue();
        //重点关注此方法
        processDeregisterQueue();
        if (interruptTriggered) {
            resetWakeupSocket();
            return 0;
        }
        ...
    }

     /**
     * sun.nio.ch.SelectorImpl#processDeregisterQueue
     * Invoked by selection operations to process the cancelled-key set
     */
    protected final void processDeregisterQueue() throws IOException {
        assert Thread.holdsLock(this);
        assert Thread.holdsLock(publicSelectedKeys);

        Set<SelectionKey> cks = cancelledKeys();
        synchronized (cks) {
            if (!cks.isEmpty()) {
                Iterator<SelectionKey> i = cks.iterator();
                while (i.hasNext()) {
                    SelectionKeyImpl ski = (SelectionKeyImpl)i.next();
                    i.remove();

                    // remove the key from the selector
                    implDereg(ski);

                    selectedKeys.remove(ski);
                    keys.remove(ski);

                    // remove from channel's key set
                    deregister(ski);

                    SelectableChannel ch = ski.channel();
                    if (!ch.isOpen() && !ch.isRegistered())
                        ((SelChImpl)ch).kill();
                }
            }
        }
    }
```
这里，当Channel关闭时，无论是通过调用`Channel#close`还是通过打断线程的方式来对Channel进行关闭，其都会隐式的取消关于这个Channel的所有的keys，其内部也是调用了`k.cancel()`。
``` java
//java.nio.channels.spi.AbstractInterruptibleChannel#close
    /**
     * Closes this channel.
     *
     * <p> If the channel has already been closed then this method returns
     * immediately.  Otherwise it marks the channel as closed and then invokes
     * the {@link #implCloseChannel implCloseChannel} method in order to
     * complete the close operation.  </p>
     *
     * @throws  IOException
     *          If an I/O error occurs
     */
    public final void close() throws IOException {
        synchronized (closeLock) {
            if (closed)
                return;
            closed = true;
            implCloseChannel();
        }
    }
//java.nio.channels.spi.AbstractSelectableChannel#implCloseChannel
     protected final void implCloseChannel() throws IOException {
        implCloseSelectableChannel();

        // clone keys to avoid calling cancel when holding keyLock
        SelectionKey[] copyOfKeys = null;
        synchronized (keyLock) {
            if (keys != null) {
                copyOfKeys = keys.clone();
            }
        }

        if (copyOfKeys != null) {
            for (SelectionKey k : copyOfKeys) {
                if (k != null) {
                    k.cancel();   // invalidate and adds key to cancelledKey set
                }
            }
        }
    }
```

如果`Selector`自身关闭掉，那么Channel也会被解除注册，同时代表Channel注册的key也将变得无效：
``` java
//java.nio.channels.spi.AbstractSelector#close
public final void close() throws IOException {
        boolean open = selectorOpen.getAndSet(false);
        if (!open)
            return;
        implCloseSelector();
    }
//sun.nio.ch.SelectorImpl#implCloseSelector
@Override
public final void implCloseSelector() throws IOException {
    wakeup();
    synchronized (this) {
        implClose();
        synchronized (publicSelectedKeys) {
            // Deregister channels
            Iterator<SelectionKey> i = keys.iterator();
            while (i.hasNext()) {
                SelectionKeyImpl ski = (SelectionKeyImpl)i.next();
                deregister(ski);
                SelectableChannel selch = ski.channel();
                if (!selch.isOpen() && !selch.isRegistered())
                    ((SelChImpl)selch).kill();
                selectedKeys.remove(ski);
                i.remove();
            }
            assert selectedKeys.isEmpty() && keys.isEmpty();
        }
    }
}
```
一个channel最多可以最多只能在特定的selector注册一次。我们可以通过调用`java.nio.channels.SelectableChannel#isRegistered`的方法来确定是否向一个或多个Selector注册了channel。
``` java
//java.nio.channels.spi.AbstractSelectableChannel#isRegistered
 // -- Registration --

    public final boolean isRegistered() {
        synchronized (keyLock) {
            //我们在之前往Selector上注册的时候调用了addKey方法，即每次往//一个Selector注册一次，keyCount就要自增一次。
            return keyCount != 0;
        }
    }
```
至此，继承了SelectableChannel这个类之后，这个channel就可以安全的由多个并发线程来使用。
这里，要注意的是，继承了`AbstractSelectableChannel`这个类之后，新创建的channel始终处于阻塞模式。然而与`Selector`的多路复用有关的操作必须基于非阻塞模式，所以在注册到`Selector`之前，必须将`channel`置于非阻塞模式，并且在取消注册之前，`channel`可能不会返回到阻塞模式。
这里，我们涉及了Channel的阻塞模式与非阻塞模式。在阻塞模式下，在`Channel`上调用的每个I/O操作都将阻塞，直到完成为止。 在非阻塞模式下，I/O操作永远不会阻塞，并且可以传输比请求的字节更少的字节，或者根本不传输任何字节。 我们可以通过调用channel的isBlocking方法来确定其是否为阻塞模式。
``` java
//java.nio.channels.spi.AbstractSelectableChannel#register
 public final SelectionKey register(Selector sel, int ops, Object att)
        throws ClosedChannelException
    {
        if ((ops & ~validOps()) != 0)
            throw new IllegalArgumentException();
        if (!isOpen())
            throw new ClosedChannelException();
        synchronized (regLock) {
     //此处会做判断，假如是阻塞模式，则会返回true，然后就会抛出异常
            if (isBlocking())
                throw new IllegalBlockingModeException();
            synchronized (keyLock) {
                // re-check if channel has been closed
                if (!isOpen())
                    throw new ClosedChannelException();
                SelectionKey k = findKey(sel);
                if (k != null) {
                    k.attach(att);
                    k.interestOps(ops);
                } else {
                    // New registration
                    k = ((AbstractSelector)sel).register(this, ops, att);
                    addKey(k);
                }
                return k;
            }
        }
    }
```
所以，我们在使用的时候可以基于以下的例子作为参考:
``` java
public NIOServerSelectorThread(int port)
	{
		try {
			//打开ServerSocketChannel，用于监听客户端的连接，他是所有客户端连接的父管道
			serverSocketChannel = ServerSocketChannel.open();
			//将管道设置为非阻塞模式
			serverSocketChannel.configureBlocking(false);
			//利用ServerSocketChannel创建一个服务端Socket对象，即ServerSocket
			serverSocket = serverSocketChannel.socket();
			//为服务端Socket绑定监听端口
			serverSocket.bind(new InetSocketAddress(port));
			//创建多路复用器
			selector = Selector.open();
			//将ServerSocketChannel注册到Selector多路复用器上，并且监听ACCEPT事件
			serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
			System.out.println("The server is start in port: "+port);
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
```

## 赋予Channel支持网络socket的能力

我们最初的目的就是为了增强Socket，基于这个基本需求，没有条件创造条件，于是为了让Channel拥有网络socket的能力，这里定义了一个`java.nio.channels.NetworkChannel`接口。花不多说，我们来看这个接口的定义:
``` java
public interface NetworkChannel extends Channel
{
    NetworkChannel bind(SocketAddress local) throws IOException;

    SocketAddress getLocalAddress() throws IOException;

    <T> NetworkChannel setOption(SocketOption<T> name, T value) throws IOException;

    <T> T getOption(SocketOption<T> name) throws IOException;

    Set<SocketOption<?>> supportedOptions();
}
```
通过`bind(SocketAddress)` 方法将`socket`绑定到本地 `SocketAddress`上，通过getLocalAddress()方法返回`socket`绑定的地址，
通过 `setOption(SocketOption,Object)`和`getOption(SocketOption)`方法设置和查询`socket`支持的配置选项。
### bind
接下来我们来看 `java.nio.channels.ServerSocketChannel`抽象类及其实现类`sun.nio.ch.ServerSocketChannelImpl`对之实现的细节。
首先我们来看其对于bind的实现:
``` java
//sun.nio.ch.ServerSocketChannelImpl#bind
@Override
public ServerSocketChannel bind(SocketAddress local, int backlog) throws IOException {
    synchronized (stateLock) {
        ensureOpen();
        //通过localAddress判断是否已经调用过bind
        if (localAddress != null)
            throw new AlreadyBoundException();
        //InetSocketAddress(0)表示绑定到本机的所有地址，由操作系统选择合适的端口
        InetSocketAddress isa = (local == null)
                                ? new InetSocketAddress(0)
                                : Net.checkAddress(local);
        SecurityManager sm = System.getSecurityManager();
        if (sm != null)
            sm.checkListen(isa.getPort());
        NetHooks.beforeTcpBind(fd, isa.getAddress(), isa.getPort());
        Net.bind(fd, isa.getAddress(), isa.getPort());
        //开启监听，s如果参数backlog小于1，默认接受50个连接
        Net.listen(fd, backlog < 1 ? 50 : backlog);
        localAddress = Net.localAddress(fd);
    }
    return this;
}
```
下面我们来看看Net中的bind和listen方法是如何实现的。
#### Net.bind

``` java
//sun.nio.ch.Net#bind(java.io.FileDescriptor, java.net.InetAddress, int)
public static void bind(FileDescriptor fd, InetAddress addr, int port)
        throws IOException
    {
        bind(UNSPEC, fd, addr, port);
    }

static void bind(ProtocolFamily family, FileDescriptor fd,
                    InetAddress addr, int port) throws IOException
{
    //如果传入的协议域不是IPV4而且支持IPV6,则使用ipv6
    boolean preferIPv6 = isIPv6Available() &&
        (family != StandardProtocolFamily.INET);
    bind0(fd, preferIPv6, exclusiveBind, addr, port);
}

private static native void bind0(FileDescriptor fd, boolean preferIPv6,
                                    boolean useExclBind, InetAddress addr,
                                    int port)
    throws IOException;
```
bind0为native方法实现:

``` java
JNIEXPORT void JNICALL
Java_sun_nio_ch_Net_bind0(JNIEnv *env, jclass clazz, jobject fdo, jboolean preferIPv6,
                          jboolean useExclBind, jobject iao, int port)
{
    SOCKETADDRESS sa;
    int sa_len = 0;
    int rv = 0;
    //将java的InetAddress转换为c的struct sockaddr
    if (NET_InetAddressToSockaddr(env, iao, port, &sa, &sa_len,
                                  preferIPv6) != 0) {
        return;//转换失败，方法返回
    }
    //调用bind方法:int bind(int sockfd, struct sockaddr* addr, socklen_t addrlen)
    rv = NET_Bind(fdval(env, fdo), &sa, sa_len);
    if (rv != 0) {
        handleSocketError(env, errno);
    }
}

```
socket是用户程序与内核交互信息的枢纽，它自身没有网络协议地址和端口号等信息,在进行网络通信的时候，必须把一个socket与一个地址相关联。
很多时候内核会我们自动绑定一个地址，然而有时用户可能需要自己来完成这个绑定的过程，以满足实际应用的需要；
最典型的情况是一个服务器进程需要绑定一个众所周知的地址或端口以等待客户来连接。
对于客户端，很多时候并不需要调用bind方法，而是由内核自动绑定；


**这里要注意**，绑定归绑定，在有连接过来的时候会创建一个新的Socket，然后服务端操作这个新的Socket即可。这里就可以关注**accept**方法了。由`sun.nio.ch.ServerSocketChannelImpl#bind`最后，我们知道其通过`Net.listen(fd, backlog < 1 ? 50 : backlog)`开启监听，如果参数backlog小于1，默认接受50个连接。由此，我们来关注下`Net.listen`方法细节。

#### Net.listen
``` java
//sun.nio.ch.Net#listen
static native void listen(FileDescriptor fd, int backlog) throws IOException;
```
可以知道，`Net.listen`是`native`方法，源码如下：
``` java
JNIEXPORT void JNICALL
Java_sun_nio_ch_Net_listen(JNIEnv *env, jclass cl, jobject fdo, jint backlog)
{
    if (listen(fdval(env, fdo), backlog) < 0)
        handleSocketError(env, errno);
}
```
可以看到底层是调用`listen`实现的，`listen`函数在一般在调用`bind`之后到调用`accept`之前调用，它的函数原型是：
`int listen(int sockfd, int backlog)`返回值：0表示成功， -1表示失败

我们再来关注下bind操作中的其他细节，最开始时的`ensureOpen()`方法判断:
``` java
//sun.nio.ch.ServerSocketChannelImpl#ensureOpen
// @throws ClosedChannelException if channel is closed
private void ensureOpen() throws ClosedChannelException {
    if (!isOpen())
        throw new ClosedChannelException();
}
//java.nio.channels.spi.AbstractInterruptibleChannel#isOpen
public final boolean isOpen() {
        return !closed;
    }
```
如果`socket`关闭，则抛出`ClosedChannelException` 。

我们再来看下`Net#checkAddress`：
``` java
//sun.nio.ch.Net#checkAddress(java.net.SocketAddress)
public static InetSocketAddress checkAddress(SocketAddress sa) {
    if (sa == null)//地址为空
        throw new NullPointerException();
        //非InetSocketAddress类型地址
    if (!(sa instanceof InetSocketAddress))
        throw new UnsupportedAddressTypeException(); // ## needs arg
    InetSocketAddress isa = (InetSocketAddress)sa;
    //地址不可识别
    if (isa.isUnresolved())
        throw new UnresolvedAddressException(); // ## needs arg
    InetAddress addr = isa.getAddress();
        //非ip4和ip6地址
    if (!(addr instanceof Inet4Address || addr instanceof Inet6Address))
        throw new IllegalArgumentException("Invalid address type");
    return isa;
}
```
从上面可以看出，bind首先检查`ServerSocket`是否关闭，是否绑定地址， 如果既没有绑定也没关闭，则检查绑定的`socketaddress`是否正确或合法； 然后通过Net工具类的`bind`和`listen`，完成实际的`ServerSocket`地址绑定和开启监听，如果绑定是开启的参数小于`1`，则默认接受`50`个连接。

对照我们之前在第一篇中接触的BIO，我们来看些`accept()`方法的实现:
``` java
//sun.nio.ch.ServerSocketChannelImpl#accept()
@Override
public SocketChannel accept() throws IOException {
    acceptLock.lock();
    try {
        int n = 0;
        FileDescriptor newfd = new FileDescriptor();
        InetSocketAddress[] isaa = new InetSocketAddress[1];

        boolean blocking = isBlocking();
        try {
            begin(blocking);
            do {
                n = accept(this.fd, newfd, isaa);
            } while (n == IOStatus.INTERRUPTED && isOpen());
        } finally {
            end(blocking, n > 0);
            assert IOStatus.check(n);
        }

        if (n < 1)
            return null;
        //针对接受连接的处理通道socketchannelimpl，默认为阻塞模式
        // newly accepted socket is initially in blocking mode
        IOUtil.configureBlocking(newfd, true);

        InetSocketAddress isa = isaa[0];
        //构建SocketChannelImpl，这个具体在SocketChannelImpl再说
        SocketChannel sc = new SocketChannelImpl(provider(), newfd, isa);

        // check permitted to accept connections from the remote address
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            try {
                //检查地址和port权限
                sm.checkAccept(isa.getAddress().getHostAddress(), isa.getPort());
            } catch (SecurityException x) {
                sc.close();
                throw x;
            }
        }
         //返回socketchannelimpl
        return sc;

    } finally {
        acceptLock.unlock();
    }
}
```
对于`accept(this.fd, newfd, isaa)`，调用accept接收socket中已建立的连接，我们之前有在BIO中了解过，函数最终会调用:int accept(int sockfd,struct sockaddr *addr, socklen_t *addrlen);

- 如果fd监听socket的队列中没有等待的连接，socket也没有被标记为Non-blocking，accept()会阻塞直到连接出现；
- 如果socket被标记为Non-blocking，队列中也没有等待的连接，accept()返回错误EAGAIN或EWOULDBLOCK

这里`begin(blocking);` 与 `end(blocking, n > 0);`的合作模式我们在[InterruptibleChannel 与可中断 IO](https://github.com/muyinchen/woker/blob/master/NIO/%E8%A1%A5%E5%85%85%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB%EF%BC%9AInterruptibleChannel%20%E4%B8%8E%E5%8F%AF%E4%B8%AD%E6%96%AD%20IO.md)这一篇文章中已经涉及过，这里再次提一下，让大家看到其应用，此处专注的是等待连接这个过程，期间可以出现异常打断，这个过程正常结束的话，就会正常往下执行逻辑，不要搞的好像这个Channel要结束了一样，`end(blocking, n > 0)`的第二个参数completed也只是在判断这个等待过程是否结束而已，不要功能范围扩大化。
### supportedOptions
我们再来看下`NetworkChannel`的其他方法实现，首先来看`supportedOptions`：
``` java
//sun.nio.ch.ServerSocketChannelImpl#supportedOptions
@Override
public final Set<SocketOption<?>> supportedOptions() {
    return DefaultOptionsHolder.defaultOptions;
}
//sun.nio.ch.ServerSocketChannelImpl.DefaultOptionsHolder
private static class DefaultOptionsHolder {
    static final Set<SocketOption<?>> defaultOptions = defaultOptions();

    private static Set<SocketOption<?>> defaultOptions() {
        HashSet<SocketOption<?>> set = new HashSet<>();
        set.add(StandardSocketOptions.SO_RCVBUF);
        set.add(StandardSocketOptions.SO_REUSEADDR);
        if (Net.isReusePortAvailable()) {
            set.add(StandardSocketOptions.SO_REUSEPORT);
        }
        set.add(StandardSocketOptions.IP_TOS);
        set.addAll(ExtendedSocketOptions.options(SOCK_STREAM));
        //返回不可修改的HashSet
        return Collections.unmodifiableSet(set);
    }
}
```
对上述配置中的一些配置我们大致来瞅眼:
``` java
//java.net.StandardSocketOptions
//socket接受缓存大小
public static final SocketOption<Integer> SO_RCVBUF =
        new StdSocketOption<Integer>("SO_RCVBUF", Integer.class);
//是否可重用地址
public static final SocketOption<Boolean> SO_REUSEADDR =
        new StdSocketOption<Boolean>("SO_REUSEADDR", Boolean.class);
//是否可重用port
public static final SocketOption<Boolean> SO_REUSEPORT =
        new StdSocketOption<Boolean>("SO_REUSEPORT", Boolean.class);
//Internet协议（IP）标头（header）中的服务类型（ToS）。
public static final SocketOption<Integer> IP_TOS =
        new StdSocketOption<Integer>("IP_TOS", Integer.class);
```
### setOption实现
知道了上面的支持配置，我们来看下`setOption`实现细节:
``` java
//sun.nio.ch.ServerSocketChannelImpl#setOption
@Override
public <T> ServerSocketChannel setOption(SocketOption<T> name, T value)
    throws IOException
{
    Objects.requireNonNull(name);
    if (!supportedOptions().contains(name))
        throw new UnsupportedOperationException("'" + name + "' not supported");
    synchronized (stateLock) {
        ensureOpen();

        if (name == StandardSocketOptions.IP_TOS) {
            ProtocolFamily family = Net.isIPv6Available() ?
                StandardProtocolFamily.INET6 : StandardProtocolFamily.INET;
            Net.setSocketOption(fd, family, name, value);
            return this;
        }

        if (name == StandardSocketOptions.SO_REUSEADDR && Net.useExclusiveBind()) {
            // SO_REUSEADDR emulated when using exclusive bind
            isReuseAddress = (Boolean)value;
        } else {
            // no options that require special handling
            Net.setSocketOption(fd, Net.UNSPEC, name, value);
        }
        return this;
    }
}
```
这里，大家就能看到`supportedOptions().contains(name)`的作用了，首先会进行支持配置的判断，然后进行正常的设置逻辑。里面对于Socket配置设定主要执行了`Net.setSocketOption`，这里，就只对其代码做中文注释就好，整个逻辑过程没有太复杂的。

``` java
static void setSocketOption(FileDescriptor fd, ProtocolFamily family,
                            SocketOption<?> name, Object value)
    throws IOException
{
    if (value == null)
        throw new IllegalArgumentException("Invalid option value");

    // only simple values supported by this method
    Class<?> type = name.type();

    if (extendedOptions.isOptionSupported(name)) {
        extendedOptions.setOption(fd, name, value);
        return;
    }
    //非整形和布尔型，则抛出断言错误
    if (type != Integer.class && type != Boolean.class)
        throw new AssertionError("Should not reach here");

    // special handling
    if (name == StandardSocketOptions.SO_RCVBUF ||
        name == StandardSocketOptions.SO_SNDBUF)
    {
        //判断接受和发送缓冲区大小
        int i = ((Integer)value).intValue();
        if (i < 0)
            throw new IllegalArgumentException("Invalid send/receive buffer size");
    }
        //缓冲区有数据，延迟关闭socket的的时间
    if (name == StandardSocketOptions.SO_LINGER) {
        int i = ((Integer)value).intValue();
        if (i < 0)
            value = Integer.valueOf(-1);
        if (i > 65535)
            value = Integer.valueOf(65535);
    }
    //UDP单播
    if (name == StandardSocketOptions.IP_TOS) {
        int i = ((Integer)value).intValue();
        if (i < 0 || i > 255)
            throw new IllegalArgumentException("Invalid IP_TOS value");
    }
    //UDP多播
    if (name == StandardSocketOptions.IP_MULTICAST_TTL) {
        int i = ((Integer)value).intValue();
        if (i < 0 || i > 255)
            throw new IllegalArgumentException("Invalid TTL/hop value");
    }

    // map option name to platform level/name
    OptionKey key = SocketOptionRegistry.findOption(name, family);
    if (key == null)
        throw new AssertionError("Option not found");

    int arg;
    //转换配置参数值
    if (type == Integer.class) {
        arg = ((Integer)value).intValue();
    } else {
        boolean b = ((Boolean)value).booleanValue();
        arg = (b) ? 1 : 0;
    }

    boolean mayNeedConversion = (family == UNSPEC);
    boolean isIPv6 = (family == StandardProtocolFamily.INET6);
    //设置文件描述符的值及其他
    setIntOption0(fd, mayNeedConversion, key.level(), key.name(), arg, isIPv6);
}
```
### getOption
接下来，我们来看`getOption`实现，源码如下:
``` java
//sun.nio.ch.ServerSocketChannelImpl#getOption
@Override
@SuppressWarnings("unchecked")
public <T> T getOption(SocketOption<T> name)
    throws IOException
{
    Objects.requireNonNull(name);
    //非通道支持选项，则抛出UnsupportedOperationException
    if (!supportedOptions().contains(name))
        throw new UnsupportedOperationException("'" + name + "' not supported");

    synchronized (stateLock) {
        ensureOpen();
        if (name == StandardSocketOptions.SO_REUSEADDR && Net.useExclusiveBind()) {
            // SO_REUSEADDR emulated when using exclusive bind
            return (T)Boolean.valueOf(isReuseAddress);
        }
        //假如获取的不是上面的配置，则委托给Net来处理
        // no options that require special handling
        return (T) Net.getSocketOption(fd, Net.UNSPEC, name);
    }
}
//sun.nio.ch.Net#getSocketOption
static Object getSocketOption(FileDescriptor fd, ProtocolFamily family,
                                SocketOption<?> name)
    throws IOException
{
    Class<?> type = name.type();

    if (extendedOptions.isOptionSupported(name)) {
        return extendedOptions.getOption(fd, name);
    }
    //只支持整形和布尔型，否则抛出断言错误
    // only simple values supported by this method
    if (type != Integer.class && type != Boolean.class)
        throw new AssertionError("Should not reach here");

    // map option name to platform level/name
    OptionKey key = SocketOptionRegistry.findOption(name, family);
    if (key == null)
        throw new AssertionError("Option not found");

    boolean mayNeedConversion = (family == UNSPEC);
    //获取文件描述的选项配置
    int value = getIntOption0(fd, mayNeedConversion, key.level(), key.name());

    if (type == Integer.class) {
        return Integer.valueOf(value);
    } else {
        //我们要看到前面支持配置处的源码其支持的类型要么是Boolean，要么是Integer
        //所以，返回值为Boolean.FALSE 或 Boolean.TRUE也就不足为奇了
        return (value == 0) ? Boolean.FALSE : Boolean.TRUE;
    }
}
```
### ServerSocketChannel与ServerSocket在bind处的异同
在**Net.bind**一节中，我们最后说了一个注意点，每个连接过来的时候都会创建一个Socket来供此连接进行操作，这个在accept方法中可以看到，其在得到连接之后，就 `new SocketChannelImpl(provider(), newfd, isa)`这个对象。那这里，就引出一个话题，我们在使用bind方法的时候，是不是也应该绑定到一个Socket之上呢，那之前bio是怎么做呢，我们先来回顾一下。
我们之前在调用`java.net.ServerSocket#ServerSocket(int, int, java.net.InetAddress)`方法的时候，里面有一个`setImpl()`:
``` java
//java.net.ServerSocket
 public ServerSocket(int port, int backlog, InetAddress bindAddr) throws IOException {
        setImpl();
        if (port < 0 || port > 0xFFFF)
            throw new IllegalArgumentException(
                       "Port value out of range: " + port);
        if (backlog < 1)
          backlog = 50;
        try {
            bind(new InetSocketAddress(bindAddr, port), backlog);
        } catch(SecurityException e) {
            close();
            throw e;
        } catch(IOException e) {
            close();
            throw e;
        }
    }
//java.net.ServerSocket#setImpl
private void setImpl() {
        if (factory != null) {
            impl = factory.createSocketImpl();
            checkOldImpl();
        } else {
            // No need to do a checkOldImpl() here, we know it's an up to date
            // SocketImpl!
            impl = new SocksSocketImpl();
        }
        if (impl != null)
            impl.setServerSocket(this);
    }
```
但是，我们此处的重点在`bind(new InetSocketAddress(bindAddr, port), backlog);`，这里的代码如下:
``` java
//java.net.ServerSocket
public void bind(SocketAddress endpoint, int backlog) throws IOException {
        if (isClosed())
            throw new SocketException("Socket is closed");
        if (!oldImpl && isBound())
            throw new SocketException("Already bound");
        if (endpoint == null)
            endpoint = new InetSocketAddress(0);
        if (!(endpoint instanceof InetSocketAddress))
            throw new IllegalArgumentException("Unsupported address type");
        InetSocketAddress epoint = (InetSocketAddress) endpoint;
        if (epoint.isUnresolved())
            throw new SocketException("Unresolved address");
        if (backlog < 1)
          backlog = 50;
        try {
            SecurityManager security = System.getSecurityManager();
            if (security != null)
                security.checkListen(epoint.getPort());
                //重点！！
            getImpl().bind(epoint.getAddress(), epoint.getPort());
            getImpl().listen(backlog);
            bound = true;
        } catch(SecurityException e) {
            bound = false;
            throw e;
        } catch(IOException e) {
            bound = false;
            throw e;
        }
    }
```
我们有看到 `getImpl()`我标示了重点，这里面做了什么，我们走进去:
``` java
//java.net.ServerSocket#getImpl
SocketImpl getImpl() throws SocketException {
    if (!created)
        createImpl();
    return impl;
}
```
在整个过程中`created`还是对象刚创建时的初始值，为false，那么，铁定会进入`createImpl()`方法中:
``` java
//java.net.ServerSocket#createImpl
void createImpl() throws SocketException {
    if (impl == null)
        setImpl();
    try {
        impl.create(true);
        created = true;
    } catch (IOException e) {
        throw new SocketException(e.getMessage());
    }
}
```
而此处，因为前面`impl`已经赋值，所以，会走`impl.create(true)`，进而将`created`设定为`true`。而此刻，终于到我想讲的重点了:
``` java
//java.net.AbstractPlainSocketImpl#create
protected synchronized void create(boolean stream) throws IOException {
    this.stream = stream;
    if (!stream) {
        ResourceManager.beforeUdpCreate();
        // only create the fd after we know we will be able to create the socket
        fd = new FileDescriptor();
        try {
            socketCreate(false);
            SocketCleanable.register(fd);
        } catch (IOException ioe) {
            ResourceManager.afterUdpClose();
            fd = null;
            throw ioe;
        }
    } else {
        fd = new FileDescriptor();
        socketCreate(true);
        SocketCleanable.register(fd);
    }
    if (socket != null)
        socket.setCreated();
    if (serverSocket != null)
        serverSocket.setCreated();
}

```
可以看到，`socketCreate(true);`，它的实现如下:
``` java
@Override
void socketCreate(boolean stream) throws IOException {
    if (fd == null)
        throw new SocketException("Socket closed");

    int newfd = socket0(stream);

    fdAccess.set(fd, newfd);
}
```
通过本地方法`socket0(stream)`得到了一个文件描述符，由此，Socket创建了出来，然后进行相应的绑定。
我们再把眼光放回到`sun.nio.ch.ServerSocketChannelImpl#accept()`中，这里new的`SocketChannelImpl`对象是得到连接之后做的事情，那对于服务器来讲，绑定时候用的Socket呢，这里，我们在使用`ServerSocketChannel`的时候，往往要使用JDK给我们提供的对我统一的方法`open`，也是为了降低我们使用的复杂度，这里是`java.nio.channels.ServerSocketChannel#open`:
``` java
//java.nio.channels.ServerSocketChannel#open
public static ServerSocketChannel open() throws IOException {
    return SelectorProvider.provider().openServerSocketChannel();
}
//sun.nio.ch.SelectorProviderImpl#openServerSocketChannel
public ServerSocketChannel openServerSocketChannel() throws IOException {
    return new ServerSocketChannelImpl(this);
}
//sun.nio.ch.ServerSocketChannelImpl#ServerSocketChannelImpl(SelectorProvider)
ServerSocketChannelImpl(SelectorProvider sp) throws IOException {
    super(sp);
    this.fd =  Net.serverSocket(true);
    this.fdVal = IOUtil.fdVal(fd);
}
//sun.nio.ch.Net#serverSocket
static FileDescriptor serverSocket(boolean stream) {
    return IOUtil.newFD(socket0(isIPv6Available(), stream, true, fastLoopback));
}
```
可以看到，只要new了一个ServerSocketChannelImpl对象，就相当于拿到了一个`socket`然后bind也就有着落了。但是，我们要注意下细节`ServerSocketChannel#open`得到的是`ServerSocketChannel`类型。我们accept到一个客户端来的连接后，应该在客户端与服务器之间创建一个Socket通道来供两者通信操作的，所以，`sun.nio.ch.ServerSocketChannelImpl#accept()`中所做的是`SocketChannel sc = new SocketChannelImpl(provider(), newfd, isa);`，得到的是`SocketChannel`类型的对象，这样，就可以将Socket的读写数据的方法定义在这个类里面。
### 由ServerSocketChannel的socket方法延伸的
关于`ServerSocketChannel`，我们还有方法需要接触一下，如socket():
``` java
//sun.nio.ch.ServerSocketChannelImpl#socket
@Override
public ServerSocket socket() {
    synchronized (stateLock) {
        if (socket == null)
            socket = ServerSocketAdaptor.create(this);
        return socket;
    }
}
```
我们看到了`ServerSocketAdaptor`，我们通过此类的注释可知，这是一个和`ServerSocket`调用一样，但是底层是用`ServerSocketChannelImpl`来实现的一个类，其适配是的目的是适配我们使用`ServerSocket`的方式，所以该`ServerSocketAdaptor`继承`ServerSocket`并按顺序重写了它的方法，所以，我们在写这块儿代码的时候也就有了新的选择。

[InterruptibleChannel 与可中断 IO](https://github.com/muyinchen/woker/blob/master/NIO/%E8%A1%A5%E5%85%85%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB%EF%BC%9AInterruptibleChannel%20%E4%B8%8E%E5%8F%AF%E4%B8%AD%E6%96%AD%20IO.md)这一篇文章中已经涉及过`java.nio.channels.spi.AbstractInterruptibleChannel#close`的实现，这里，我们再来回顾下其中的某些细节，顺带引出我们新的话题:
``` java
//java.nio.channels.spi.AbstractInterruptibleChannel#close
public final void close() throws IOException {
    synchronized (closeLock) {
        if (closed)
            return;
        closed = true;
        implCloseChannel();
    }
}
//java.nio.channels.spi.AbstractSelectableChannel#implCloseChannel
protected final void implCloseChannel() throws IOException {
        implCloseSelectableChannel();

        // clone keys to avoid calling cancel when holding keyLock
        SelectionKey[] copyOfKeys = null;
        synchronized (keyLock) {
            if (keys != null) {
                copyOfKeys = keys.clone();
            }
        }

        if (copyOfKeys != null) {
            for (SelectionKey k : copyOfKeys) {
                if (k != null) {
                    k.cancel();   // invalidate and adds key to cancelledKey set
                }
            }
        }
    }
//sun.nio.ch.ServerSocketChannelImpl#implCloseSelectableChannel
@Override
protected void implCloseSelectableChannel() throws IOException {
    assert !isOpen();

    boolean interrupted = false;
    boolean blocking;

    // set state to ST_CLOSING
    synchronized (stateLock) {
        assert state < ST_CLOSING;
        state = ST_CLOSING;
        blocking = isBlocking();
    }

    // wait for any outstanding accept to complete
    if (blocking) {
        synchronized (stateLock) {
            assert state == ST_CLOSING;
            long th = thread;
            if (th != 0) {
                //本地线程不为null，则本地Socket预先关闭
                //并通知线程通知关闭
                nd.preClose(fd);
                NativeThread.signal(th);

                // wait for accept operation to end
                while (thread != 0) {
                    try {
                        stateLock.wait();
                    } catch (InterruptedException e) {
                        interrupted = true;
                    }
                }
            }
        }
    } else {
        // non-blocking mode: wait for accept to complete
        acceptLock.lock();
        acceptLock.unlock();
    }

    // set state to ST_KILLPENDING
    synchronized (stateLock) {
        assert state == ST_CLOSING;
        state = ST_KILLPENDING;
    }

    // close socket if not registered with Selector
    //如果未在Selector上注册，直接kill掉
    //即关闭文件描述
    if (!isRegistered())
        kill();

    // restore interrupt status
    //印证了我们上一篇中在异步打断中若是通过线程的中断方法中断线程的话
    //最后要设定该线程状态是interrupt
    if (interrupted)
        Thread.currentThread().interrupt();
}

@Override
public void kill() throws IOException {
    synchronized (stateLock) {
        if (state == ST_KILLPENDING) {
            state = ST_KILLED;
            nd.close(fd);
        }
    }
}
```
#### channel的close()应用
也是因为`close()`并没有在[InterruptibleChannel 与可中断 IO](https://github.com/muyinchen/woker/blob/master/NIO/%E8%A1%A5%E5%85%85%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB%EF%BC%9AInterruptibleChannel%20%E4%B8%8E%E5%8F%AF%E4%B8%AD%E6%96%AD%20IO.md)这一篇文章中进行具体的讲解应用，这里其应用的更多是在`SocketChannel`这里，其更多的涉及到客户端与服务端建立连接交换数据，所以断开连接后，将不用的Channel关闭是很正常的。
这里，在`sun.nio.ch.ServerSocketChannelImpl#accept()`中的源码中:
``` java
@Override
public SocketChannel accept() throws IOException {
        ...
        // newly accepted socket is initially in blocking mode
        IOUtil.configureBlocking(newfd, true);

        InetSocketAddress isa = isaa[0];
        SocketChannel sc = new SocketChannelImpl(provider(), newfd, isa);

        // check permitted to accept connections from the remote address
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            try {
                sm.checkAccept(isa.getAddress().getHostAddress(), isa.getPort());
            } catch (SecurityException x) {
                sc.close();
                throw x;
            }
        }
        return sc;

    } finally {
        acceptLock.unlock();
    }
}
```
这里通过对所接收的连接的远程地址做合法性判断，假如验证出现异常，则关闭上面创建的`SocketChannel`。
还有一个关于close()的实际用法，在客户端建立连接的时候，如果连接出异常，同样是要关闭所创建的Socket:
``` java
//java.nio.channels.SocketChannel#open(java.net.SocketAddress)
public static SocketChannel open(SocketAddress remote)
        throws IOException
    {
        SocketChannel sc = open();
        try {
            sc.connect(remote);
        } catch (Throwable x) {
            try {
                sc.close();
            } catch (Throwable suppressed) {
                x.addSuppressed(suppressed);
            }
            throw x;
        }
        assert sc.isConnected();
        return sc;
    }
```

接着，我们在`implCloseSelectableChannel`中会发现`nd.preClose(fd);`与`nd.close(fd);`，这个在`SocketChannelImpl`与`ServerSocketChannelImpl`两者对于`implCloseSelectableChannel`实现中都可以看到，这个nd是什么，这里，我们拿`ServerSocketChannelImpl`来讲，在这个类的最后面有一段静态代码块(`SocketChannelImpl`同理)，也就是在这个类加载的时候就会执行:
``` java
//C:/Program Files/Java/jdk-11.0.1/lib/src.zip!/java.base/sun/nio/ch/ServerSocketChannelImpl.java:550
static {
     //加载nio，net资源库
        IOUtil.load();
        initIDs();
        nd = new SocketDispatcher();
    }
```
也就是说，在`ServerSocketChannelImpl`这个类字节码加载的时候，就会创建`SocketDispatcher`对象。通过`SocketDispatcher`允许在不同的平台调用不同的本地方法进行读写操作,然后基于这个类，我们就可以在`sun.nio.ch.SocketChannelImpl`做Socket的I/O操作。
``` java
//sun.nio.ch.SocketDispatcher
class SocketDispatcher extends NativeDispatcher
{

    static {
        IOUtil.load();
    }
    //读操作
    int read(FileDescriptor fd, long address, int len) throws IOException {
        return read0(fd, address, len);
    }

    long readv(FileDescriptor fd, long address, int len) throws IOException {
        return readv0(fd, address, len);
    }
    //写操作
    int write(FileDescriptor fd, long address, int len) throws IOException {
        return write0(fd, address, len);
    }

    long writev(FileDescriptor fd, long address, int len) throws IOException {
        return writev0(fd, address, len);
    }
    //预关闭文件描述符
    void preClose(FileDescriptor fd) throws IOException {
        preClose0(fd);
    }
    //关闭文件描述
    void close(FileDescriptor fd) throws IOException {
        close0(fd);
    }

    //-- Native methods
    static native int read0(FileDescriptor fd, long address, int len)
        throws IOException;

    static native long readv0(FileDescriptor fd, long address, int len)
        throws IOException;

    static native int write0(FileDescriptor fd, long address, int len)
        throws IOException;

    static native long writev0(FileDescriptor fd, long address, int len)
        throws IOException;

    static native void preClose0(FileDescriptor fd) throws IOException;

    static native void close0(FileDescriptor fd) throws IOException;
}
```
### FileDescriptor
我们有看到`FileDescriptor`在前面代码中有大量的出现，这里，我们对它来专门介绍。通过FileDescriptor 这个类的实例来充当底层机器特定结构的不透明处理，表示打开文件，打开socket或其他字节源或接收器。
文件描述符的主要用途是创建一个 FileInputStream或 FileOutputStream来包含它。
注意: 应用程序不应创建自己的文件描述符。
我们来看其部分源码:
``` java
public final class FileDescriptor {

    private int fd;

    private long handle;

    private Closeable parent;
    private List<Closeable> otherParents;
    private boolean closed;

    /**
     * true, if file is opened for appending.
     */
    private boolean append;

    static {
        initIDs();
    }
    /**
     * 在未明确关闭FileDescriptor的情况下进行清理.
     */
    private PhantomCleanable<FileDescriptor> cleanup;

    /**
     * 构造一个无效的FileDescriptor对象，fd或handle会在之后进行设定
     */
    public FileDescriptor() {
        fd = -1;
        handle = -1;
    }

    /**
     * Used for standard input, output, and error only.
     * For Windows the corresponding handle is initialized.
     * For Unix the append mode is cached.
     * 仅用于标准输入，输出和错误。
     * 对于Windows，初始化相应的句柄。
     * 对于Unix，缓存附加模式。
     * @param fd the raw fd number (0, 1, 2)
     */
    private FileDescriptor(int fd) {
        this.fd = fd;
        this.handle = getHandle(fd);
        this.append = getAppend(fd);
    }
    ...
}
```
我们平时所用的标准输入，输出，错误流的句柄可以如下，通常，我们不会直接使用它们，而是使用`java.lang.System.in`，`java.lang.System#out`，`java.lang.System#err`:
``` java
public static final FileDescriptor in = new FileDescriptor(0);
public static final FileDescriptor out = new FileDescriptor(1);
public static final FileDescriptor err = new FileDescriptor(2);
```
测试该文件描述符是否有效可以使用如下方法:
``` java
//java.io.FileDescriptor#valid
public boolean valid() {
        return (handle != -1) || (fd != -1);
    }
```
返回值为true的话，那么这个文件描述符对象所代表的`socket` `文件操作`或其他活动的网络连接都是有效的，反之，false则是无效。
更多内容，读者可以自行深入源码，此处就不过多解释了。为了让大家可以更好的理解上述内容，我们会在后面的部分还要进一步涉及一下。

## NIO包下SocketChannel解读
在前面，我们已经接触了`SocketChannel`，这里，来接触下细节。

同样，我们也可以通过调用此类的`open`方法来创建`socket channel`。这里需要注意：

- 无法为任意预先存在的`socket`创建`channel`。
- 新创建的`socket channel`已打开但尚未连接。
- 尝试在未连接的`channel`上调用`I/O`操作将导致抛出`NotYetConnectedException`。
- 可以通过调用`connect`方法连接`socket channel`;
- 一旦连接后，`socket channel`会保持连接状态，直到它关闭。
- 是否有连接`socket channel`可以通过确定调用其`isConnected`方法。

`socket channel`支持 非阻塞连接：

- 可以先创建`socket channel`，然后可以通过 `connect` 方法建立到远程`socket`的连接。
- 通过调用`finishConnect`方法来结束连接。
- 判断是否正在进行连接操作可以通过调用`isConnectionPending`方法来确定。

`socket channel`支持异步关闭，类似于`Channel`类中的异步关闭操作。
- 如果`socket`的输入端被一个线程关闭而另一个线程在此`socket channel`上因在进行读操作而被阻塞，那么被阻塞线程中的读操作将不读取任何字节并将返回 `-1` 。
- 如果`socket`的输出端被一个线程关闭而另一个线程在`socket channel`上因在进行写操作而被阻塞，则被阻塞的线程将收到`AsynchronousCloseException`。

接下来，我们来看其具体实现方法。
### ServerSocketChannel与SocketChannel的open()
``` java
//java.nio.channels.SocketChannel#open()
public static SocketChannel open() throws IOException {
    return SelectorProvider.provider().openSocketChannel();
}
//java.nio.channels.SocketChannel#open(java.net.SocketAddress)
//这个方法省的我们再次调用connect了
public static SocketChannel open(SocketAddress remote)
    throws IOException
{
    //默认是堵塞的，这个在AbstractSelectableChannel处讨论过了
    SocketChannel sc = open();
    try {
        sc.connect(remote);
    } catch (Throwable x) {
        try {
            sc.close();
        } catch (Throwable suppressed) {
            x.addSuppressed(suppressed);
        }
        throw x;
    }
    assert sc.isConnected();
    return sc;
}
//sun.nio.ch.SelectorProviderImpl#openSocketChannel
public SocketChannel openSocketChannel() throws IOException {
    return new SocketChannelImpl(this);
}
//sun.nio.ch.SocketChannelImpl#SocketChannelImpl(java.nio.channels.spi.SelectorProvider)
SocketChannelImpl(SelectorProvider sp) throws IOException {
    super(sp);
     //调用socket函数，true表示TCP
    this.fd = Net.socket(true);
    this.fdVal = IOUtil.fdVal(fd);
}
//sun.nio.ch.Net#socket(boolean)
static FileDescriptor socket(boolean stream) throws IOException {
    return socket(UNSPEC, stream);
}
//sun.nio.ch.Net#socket(java.net.ProtocolFamily, boolean)
static FileDescriptor socket(ProtocolFamily family, boolean stream)
    throws IOException {
    boolean preferIPv6 = isIPv6Available() &&
        (family != StandardProtocolFamily.INET);
    return IOUtil.newFD(socket0(preferIPv6, stream, false, fastLoopback));
}
//sun.nio.ch.IOUtil#newFD
public static FileDescriptor newFD(int i) {
    FileDescriptor fd = new FileDescriptor();
    setfdVal(fd, i);
    return fd;
}
static native void setfdVal(FileDescriptor fd, int value);
```
关于`Net.socket(true)`，我们前面已经提到过了，这里，通过其底层源码来再次调教下 (此处不想看可以跳过):
``` java
JNIEXPORT jint JNICALL
Java_sun_nio_ch_Net_socket0(JNIEnv *env, jclass cl, jboolean preferIPv6,
                            jboolean stream, jboolean reuse, jboolean ignored)
{
    int fd;
    //字节流还是数据报,TCP对应SOCK_STREAM,UDP对应SOCK_DGRAM,此处传入的stream=true;
    int type = (stream ? SOCK_STREAM : SOCK_DGRAM);
    //判断是IPV6还是IPV4
    int domain = (ipv6_available() && preferIPv6) ? AF_INET6 : AF_INET;

    //调用Linux的socket函数,domain为代表协议;
    //type为套接字类型，protocol设置为0来表示使用默认的传输协议
    fd = socket(domain, type, 0);
    //出错
    if (fd < 0) {
        return handleSocketError(env, errno);
    }

    /* Disable IPV6_V6ONLY to ensure dual-socket support */
    if (domain == AF_INET6) {
        int arg = 0;
        //arg=1设置ipv6的socket只接收ipv6地址的报文,arg=0表示也可接受ipv4的请求
        if (setsockopt(fd, IPPROTO_IPV6, IPV6_V6ONLY, (char*)&arg,
                       sizeof(int)) < 0) {
            JNU_ThrowByNameWithLastError(env,
                                         JNU_JAVANETPKG "SocketException",
                                         "Unable to set IPV6_V6ONLY");
            close(fd);
            return -1;
        }
    }

    //SO_REUSEADDR有四种用途:
    //1.当有一个有相同本地地址和端口的socket1处于TIME_WAIT状态时，而你启动的程序的socket2要占用该地址和端口，你的程序就要用到该选项。
    //2.SO_REUSEADDR允许同一port上启动同一服务器的多个实例(多个进程)。但每个实例绑定的IP地址是不能相同的。
    //3.SO_REUSEADDR允许单个进程绑定相同的端口到多个socket上，但每个socket绑定的ip地址不同。
   //4.SO_REUSEADDR允许完全相同的地址和端口的重复绑定。但这只用于UDP的多播，不用于TCP;
    if (reuse) {
        int arg = 1;
        if (setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, (char*)&arg,
                       sizeof(arg)) < 0) {
            JNU_ThrowByNameWithLastError(env,
                                         JNU_JAVANETPKG "SocketException",
                                         "Unable to set SO_REUSEADDR");
            close(fd);
            return -1;
        }
    }

#if defined(__linux__)
    if (type == SOCK_DGRAM) {
        int arg = 0;
        int level = (domain == AF_INET6) ? IPPROTO_IPV6 : IPPROTO_IP;
        if ((setsockopt(fd, level, IP_MULTICAST_ALL, (char*)&arg, sizeof(arg)) < 0) &&
            (errno != ENOPROTOOPT)) {
            JNU_ThrowByNameWithLastError(env,
                                         JNU_JAVANETPKG "SocketException",
                                         "Unable to set IP_MULTICAST_ALL");
            close(fd);
            return -1;
        }
    }

     //IPV6_MULTICAST_HOPS用于控制多播的范围，
     // 1表示只在本地网络转发,
     //更多介绍请参考(http://www.ctt.sbras.ru/cgi-bin/www/unix_help/unix-man?ip6+4);
    /* By default, Linux uses the route default */
    if (domain == AF_INET6 && type == SOCK_DGRAM) {
        int arg = 1;
        if (setsockopt(fd, IPPROTO_IPV6, IPV6_MULTICAST_HOPS, &arg,
                       sizeof(arg)) < 0) {
            JNU_ThrowByNameWithLastError(env,
                                         JNU_JAVANETPKG "SocketException",
                                         "Unable to set IPV6_MULTICAST_HOPS");
            close(fd);
            return -1;
        }
    }
#endif
    return fd;
}

```
Linux 3.9之后加入了`SO_REUSEPORT`配置，这个配置很强大，多个`socket`(不管是处于监听还是非监听，不管是TCP还是UDP)只要在绑定之前设置了`SO_REUSEPORT`属性，那么就可以绑定到完全相同的地址和端口。
为了阻止"port 劫持"(`Port hijacking`)有一个特别的限制：所有希望共享源地址和端口的socket都必须拥有相同的有效用户id(`effective user ID`)。这样一个用户就不能从另一个用户那里"偷取"端口。另外，内核在处理`SO_REUSEPORT socket`的时候使用了其它系统上没有用到的"特殊技巧"：

- 对于UDP socket，内核尝试平均的转发数据报;
- 对于TCP监听socket，内核尝试将新的客户连接请求(由accept返回)平均的交给共享同一地址和端口的socket(服务器监听socket)。

例如：一个简单的服务器程序的多个实例可以使用`SO_REUSEPORT socket`，这样就实现一个简单的负载均衡，因为内核已经把请求的分配都做了。

在前面的代码中可以看到，在这个`socket`创建成功之后，调用`IOUtil.newFD`创建了文件描述符
。这里，我只是想知道这个Socket是可以输入呢，还是可以读呢，还是有错呢，参考`FileDescriptor`这一节最后那几个标准状态的设定，其实这里也是一样，因为我们要往Socket中写和读，其标准状态无非就这三种:输入，输出，出错。而这个Socket是绑定在`SocketChannel`上的，那就把`FileDescriptor`也绑定到上面即可，这样我们就可以获取到它的状态了。由于FileDescriptor没有提供外部设置fd的方法，setfdVal是通过本地方法实现的:
``` java
JNIEXPORT void JNICALL
Java_sun_nio_ch_IOUtil_setfdVal(JNIEnv *env, jclass clazz, jobject fdo, jint val)
{
    (*env)->SetIntField(env, fdo, fd_fdID, val);
}
```
假如各位有对Linux下的shell编程或者命令有了解的话，我们知道，shell对报错进行重定向要使用2>，也就是将错误信息由2号所指向的通道写出，这里0和1 同样指向一个通道。此处同样也代表了状态，这样就可以对代表Socket的状态进行操作了，也就是改变`SelectionKey`的`interest ops`，即首先对`SelectionKey`按输入输出类型进行分类，然后我们的读写状态的操作也就有着落了。此处我们打个戳，在下一篇中会对其进行细节讲解。

我们回归到`SocketChannel`的`open`方法中。我们可以看到，`SelectorProvider.provider().openSocketChannel()`返回的是`SocketChannelImpl`对象实例。在`SocketChannelImpl(SelectorProvider sp)`中我们并未看到其对`this.state`进行值操作，也就是其默认为0，即`ST_UNCONNECTED`(未连接状态)，同时Socket默认是堵塞的。
所以，一般情况下，当采用异步方式时，使用不带参数的open方法比较常见，这样，我们会随之调用`configureBlocking`来设置非堵塞。

### SocketChannel的connect解读
由前面可知，我们调用`connect`方法连接到远程服务器,其源码如下：
``` java
//sun.nio.ch.SocketChannelImpl#connect
@Override
public boolean connect(SocketAddress sa) throws IOException {
    InetSocketAddress isa = Net.checkAddress(sa);
    SecurityManager sm = System.getSecurityManager();
    if (sm != null)
        sm.checkConnect(isa.getAddress().getHostAddress(), isa.getPort());

    InetAddress ia = isa.getAddress();
    if (ia.isAnyLocalAddress())
        ia = InetAddress.getLocalHost();

    try {
        readLock.lock();
        try {
            writeLock.lock();
            try {
                int n = 0;
                boolean blocking = isBlocking();
                try {
                    //支持线程中断，通过设置当前线程的Interruptible blocker属性实现
                    beginConnect(blocking, isa);
                    do {
                    //调用connect函数实现，如果采用堵塞模式，会一直等待，直到成功或出//现异常
                        n = Net.connect(fd, ia, isa.getPort());
                    } while (n == IOStatus.INTERRUPTED && isOpen());
                } finally {
                    endConnect(blocking, (n > 0));
                }
                assert IOStatus.check(n);
                //连接成功
                return n > 0;
            } finally {
                writeLock.unlock();
            }
        } finally {
            readLock.unlock();
        }
    } catch (IOException ioe) {
        // connect failed, close the channel
        close();
        throw SocketExceptions.of(ioe, isa);
    }
}
```
关于`beginConnect`与`endConnect`，是针对`AbstractInterruptibleChannel`中`begin()`与`end`方法的一种增强。这里我们需要知道的是，假如是非阻塞Channel的话，我们无须去关心连接过程的打断。顾名思义，只有阻塞等待才需要去考虑打断这一场景的出现。剩下的细节我已经在代码中进行了完整的注释，读者可自行查看。
``` java
//sun.nio.ch.SocketChannelImpl#beginConnect
private void beginConnect(boolean blocking, InetSocketAddress isa)
    throws IOException
{   //只有阻塞的时候才会进入begin
    if (blocking) {
        // set hook for Thread.interrupt
        //支持线程中断，通过设置当前线程的Interruptible blocker属性实现
        begin();
    }
    synchronized (stateLock) {
        //默认为open, 除非调用了close方法
        ensureOpen();
        //检查连接状态
        int state = this.state;
        if (state == ST_CONNECTED)
            throw new AlreadyConnectedException();
        if (state == ST_CONNECTIONPENDING)
            throw new ConnectionPendingException();
        //断言当前的状态是否是未连接状态，如果是，赋值表示正在连接中
        assert state == ST_UNCONNECTED;
        //表示正在连接中
        this.state = ST_CONNECTIONPENDING;
        //只有未绑定本地地址也就是说未调用bind方法才执行,
        //该方法在ServerSocketChannel中也见过
        if (localAddress == null)
            NetHooks.beforeTcpConnect(fd, isa.getAddress(), isa.getPort());
        remoteAddress = isa;

        if (blocking) {
            // record thread so it can be signalled if needed
            readerThread = NativeThread.current();
        }
    }
}
```
在连接过程中，我们需要注意的就是几个连接的状态:`ST_UNCONNECTED`、`ST_CONNECTED` 、`ST_CONNECTIONPENDING`、`ST_CLOSING`、`ST_KILLPENDING`、`ST_KILLED`，也是因为其是一个公共状态，可能会有多个线程对其进行连接操作的。所以，`state`被定义为一个`volatile`变量，这个变量在改变的时候需要有`stateLock`这个对象来作为`synchronized`锁对象来控制同步操作的。
``` java
//sun.nio.ch.SocketChannelImpl#endConnect
private void endConnect(boolean blocking, boolean completed)
    throws IOException
{
    endRead(blocking, completed);
    //当上面代码中n>0，说明连接成功，更新状态为ST_CONNECTED
    if (completed) {
        synchronized (stateLock) {
            if (state == ST_CONNECTIONPENDING) {
                localAddress = Net.localAddress(fd);
                state = ST_CONNECTED;
            }
        }
    }
}
//sun.nio.ch.SocketChannelImpl#endRead
private void endRead(boolean blocking, boolean completed)
    throws AsynchronousCloseException
{   //当阻塞状态下的话，才进入
    if (blocking) {
        synchronized (stateLock) {
            readerThread = 0;
            // notify any thread waiting in implCloseSelectableChannel
            if (state == ST_CLOSING) {
                stateLock.notifyAll();
            }
        }
        //和begin成对出现，当线程中断时，抛出ClosedByInterruptException
        // remove hook for Thread.interrupt
        end(completed);
    }
}
```
我们来关注`connect`中的`Net.connect(fd, ia, isa.getPort())`方法:
``` java
//sun.nio.ch.Net#connect
static int connect(FileDescriptor fd, InetAddress remote, int remotePort)
    throws IOException
{
    return connect(UNSPEC, fd, remote, remotePort);
}
//sun.nio.ch.Net#connect
static int connect(ProtocolFamily family, FileDescriptor fd, InetAddress remote, int remotePort)
    throws IOException
{
    boolean preferIPv6 = isIPv6Available() &&
        (family != StandardProtocolFamily.INET);
    return connect0(preferIPv6, fd, remote, remotePort);
}
```
该方法最终会调用native方法，具体注释如下:
``` java
JNIEXPORT jint JNICALL
Java_sun_nio_ch_Net_connect0(JNIEnv *env, jclass clazz, jboolean preferIPv6,
                             jobject fdo, jobject iao, jint port)
{
    SOCKETADDRESS sa;
    int sa_len = 0;
    int rv;
    //地址转换为struct sockaddr格式
    if (NET_InetAddressToSockaddr(env, iao, port, &sa, &sa_len, preferIPv6) != 0) {
        return IOS_THROWN;
    }
    //传入fd和sockaddr,与远程服务器建立连接，一般就是TCP三次握手
   //如果设置了configureBlocking(false),不会堵塞，否则会堵塞一直到超时或出现异常
    rv = connect(fdval(env, fdo), &sa.sa, sa_len);
    //0表示连接成功，失败时通过errno获取具体原因
    if (rv != 0) {
        //非堵塞，连接还未建立(-2)
        if (errno == EINPROGRESS) {
            return IOS_UNAVAILABLE;
        } else if (errno == EINTR) {
            //中断(-3)
            return IOS_INTERRUPTED;
        }
        return handleSocketError(env, errno);
    }
    //连接建立,一般TCP连接连接都需要时间，因此除非是本地网络，
    //一般情况下非堵塞模式返回IOS_UNAVAILABLE比较多；
    return 1;
}
```
从上面可以通过注释看到，如果是非堵塞，而且连接也并未立马建立成功，其返回的是-2，也就是连接未建立成功，由之前`beginConnect`部分源码可知，此时状态为`ST_CONNECTIONPENDING`，那么，非阻塞条件下，什么时候会变为`ST_CONNECTED`?有什么方法可以查询状态或者等待连接完成？
那就让我们来关注下`sun.nio.ch.SocketChannelImpl#finishConnect`
### SocketChannelImpl中finishConnect解读

首先，我们回顾下，前面我们涉及了`sun.nio.ch.ServerSocketAdaptor`的用法，方便我们只有Socket编程习惯人群使用，这里，我们也就可以看到基本的核心实现逻辑，那么有`ServerSocketAdaptor`就有`SocketAdaptor`，这里，在BIO的Socket编程中最后也是调用了` connect(address)`操作:
``` java
//java.net.Socket#Socket
private Socket(SocketAddress address, SocketAddress localAddr,
                boolean stream) throws IOException {
    setImpl();

    // backward compatibility
    if (address == null)
        throw new NullPointerException();

    try {
        createImpl(stream);
        if (localAddr != null)
            bind(localAddr);
        connect(address);
    } catch (IOException | IllegalArgumentException | SecurityException e) {
        try {
            close();
        } catch (IOException ce) {
            e.addSuppressed(ce);
        }
        throw e;
    }
}
```
这里，我们可以调用`java.nio.channels.SocketChannel#open()`，然后调用所得到的`SocketChannel`对象的`socket()`方法，就可以得到`sun.nio.ch.SocketAdaptor`对象实例了。我们来查看`SocketAdaptor`的connect实现:
``` java
//sun.nio.ch.SocketAdaptor#connect
public void connect(SocketAddress remote) throws IOException {
    connect(remote, 0);
}

public void connect(SocketAddress remote, int timeout) throws IOException {
    if (remote == null)
        throw new IllegalArgumentException("connect: The address can't be null");
    if (timeout < 0)
        throw new IllegalArgumentException("connect: timeout can't be negative");

    synchronized (sc.blockingLock()) {
        if (!sc.isBlocking())
            throw new IllegalBlockingModeException();

        try {
            //未设定超时则会一直在此等待直到连接或者出现异常
            // no timeout
            if (timeout == 0) {
                sc.connect(remote);
                return;
            }
            //有超时设定，则会将Socket给设定为非阻塞
            // timed connect
            sc.configureBlocking(false);
            try {
                if (sc.connect(remote))
                    return;
            } finally {
                try {
                    sc.configureBlocking(true);
                } catch (ClosedChannelException e) { }
            }

            long timeoutNanos = NANOSECONDS.convert(timeout, MILLISECONDS);
            long to = timeout;
            for (;;) {
                //通过计算超时时间，在允许的时间范围内无限循环来进行连接，
                //如果超时，则关闭这个Socket
                long startTime = System.nanoTime();
                if (sc.pollConnected(to)) {
                    boolean connected = sc.finishConnect();
                    //看下文解释
                    assert connected;
                    break;
                }
                timeoutNanos -= System.nanoTime() - startTime;
                if (timeoutNanos <= 0) {
                    try {
                        sc.close();
                    } catch (IOException x) { }
                    throw new SocketTimeoutException();
                }
                to = MILLISECONDS.convert(timeoutNanos, NANOSECONDS);
            }

        } catch (Exception x) {
            Net.translateException(x, true);
        }
    }

}
```
这里先解释下一个小注意点:在Java中，`assert`关键字是从`JAVA SE 1.4` 引入的，为了避免和**老版本的Java代码**中使用了`assert`关键字导致错误，**Java在执行的时候默认是不启动断言检查的（这个时候，所有的断言语句都 将忽略！），如果要开启断言检查，则需要用开关-enableassertions或-ea来开启。**
通过上面的源码注释，相信大伙已经知道大致的流程了，那关于`sun.nio.ch.SocketChannelImpl#finishConnect`到底做了什么，此处，我们来探索一番:
``` java
//sun.nio.ch.SocketChannelImpl#finishConnect
@Override
public boolean finishConnect() throws IOException {
    try {
        readLock.lock();
        try {
            writeLock.lock();
            try {
                // no-op if already connected
                if (isConnected())
                    return true;

                boolean blocking = isBlocking();
                boolean connected = false;
                try {
                    beginFinishConnect(blocking);
                    int n = 0;
                    if (blocking) {
                        do {
                            //阻塞情况下，第二个参数传入true
                            n = checkConnect(fd, true);
                        } while ((n == 0 || n == IOStatus.INTERRUPTED) && isOpen());
                    } else {
                        //非阻塞情况下，第二个参数传入false
                        n = checkConnect(fd, false);
                    }
                    connected = (n > 0);
                } finally {
                    endFinishConnect(blocking, connected);
                }
                assert (blocking && connected) ^ !blocking;
                return connected;
            } finally {
                writeLock.unlock();
            }
        } finally {
            readLock.unlock();
        }
    } catch (IOException ioe) {
        // connect failed, close the channel
        close();
        throw SocketExceptions.of(ioe, remoteAddress);
    }
}
//sun.nio.ch.SocketChannelImpl#checkConnect
private static native int checkConnect(FileDescriptor fd, boolean block)
    throws IOException;
```
关于`beginFinishConnect`与`endFinishConnect`和我们之前分析的`sun.nio.ch.SocketChannelImpl#beginConnect`与`sun.nio.ch.SocketChannelImpl#endConnect`过程差不多，不懂读者可回看。剩下的，就是我们关注的主要核心逻辑`checkConnect(fd, true)`，它也是一个本地方法，涉及到的源码如下:
``` java
JNIEXPORT jint JNICALL
Java_sun_nio_ch_SocketChannelImpl_checkConnect(JNIEnv *env, jobject this,
                                               jobject fdo, jboolean block)
{
    int error = 0;
    socklen_t n = sizeof(int);
    //获取FileDescriptor中的fd
    jint fd = fdval(env, fdo);
    int result = 0;
    struct pollfd poller;
    //文件描述符
    poller.fd = fd;
    //请求的事件为写事件
    poller.events = POLLOUT;
    //返回的事件
    poller.revents = 0;

    //-1表示阻塞,0表示立即返回，不阻塞进程
    result = poll(&poller, 1, block ? -1 : 0);
    //小于0表示调用失败
    if (result < 0) {
        if (errno == EINTR) {
            return IOS_INTERRUPTED;
        } else {
            JNU_ThrowIOExceptionWithLastError(env, "poll failed");
            return IOS_THROWN;
        }
    }
    //非堵塞时，0表示没有准备好的连接
    if (!block && (result == 0))
        return IOS_UNAVAILABLE;
    //准备好写或出现错误的socket数量>0
    if (result > 0) {
        errno = 0;
        result = getsockopt(fd, SOL_SOCKET, SO_ERROR, &error, &n);
        //出错
        if (result < 0) {
            return handleSocketError(env, errno);
        //发生错误，处理错误
        } else if (error) {
            return handleSocketError(env, error);
        } else if ((poller.revents & POLLHUP) != 0) {
            return handleSocketError(env, ENOTCONN);
        }
        //socket已经准备好，可写，即连接已经建立好
        // connected
        return 1;
    }
    return 0;
}
```
具体的过程如源码注释所示，其中是否阻塞我们在本地方法源码中和之前`sun.nio.ch.SocketChannelImpl#finishConnect`的行为产生对应。另外，从上面的源码看到，底层是通过`poll`查询`socket`的状态，从而判断连接是否建立成功；由于在非堵塞模式下，`finishConnect`方法会立即返回，根据此处`sun.nio.ch.SocketAdaptor#connect`的处理，其使用循环的方式判断连接是否建立，在我们的nio编程中，这个是不建议的，属于半成品，而是建议注册到`Selector`，通过`ops=OP_CONNECT`获取连接完成的`SelectionKey`,然后调用`finishConnect`完成连接的建立；
那么`finishConnect`是否可以不调用呢？答案是否定的，因为只有`finishConnect`中才会将状态更新为`ST_CONNECTED`，而在调用`read`和`write`时都会对状态进行判断。

这里，我们算是引出了我们即将要涉及的`Selector`和`SelectionKey`，我们会在下一篇中进行详细讲解。