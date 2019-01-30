title: JDK 源码解析 —— 网络(二)之 NIO(中)之 Selector
date: 2019-02-07
tags:
categories: JDK 源码
permalink: JDK/net-2-nio-2
author: 一叶知秋
from_url: http://t.cn/Eq9MbZs
wechat_url:

-------

摘要: 原创出处 http://t.cn/Eq9MbZs 「一叶知秋」欢迎转载，保留摘要，谢谢！

- [SelectionKey的引入](http://www.iocoder.cn/JDK/net-2-nio-2/)
  - [interestOps](http://www.iocoder.cn/JDK/net-2-nio-2/)
  - [readyOps](http://www.iocoder.cn/JDK/net-2-nio-2/)
- [Selector探究](http://www.iocoder.cn/JDK/net-2-nio-2/)
  - [Selector的open方法](http://www.iocoder.cn/JDK/net-2-nio-2/)
  - [PollArrayWrapper解读](http://www.iocoder.cn/JDK/net-2-nio-2/)
  - [SelectionKey在selector中的管理](http://www.iocoder.cn/JDK/net-2-nio-2/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------


# SelectionKey的引入
如我们在前面内容所讲，在学生确定之后，我们就要对其状态进行设定，然后再交由`Selector`进行管理，其状态的设定我们就通过`SelectionKey`来进行。

那这里我们先通过之前在`Channel`中并未仔细讲解的`SelectableChannel`下的`register`方法。我们前面有提到过， `SelectableChannel`将`channel`打造成可以通过`Selector`来进行多路复用。作为管理者，`channel`想要实现复用，就必须在管理者这里进行注册登记。所以，`SelectableChannel`下的`register`方法也就是我们值得二次关注的核心了，也是对接我们接下来内容的切入点，对于`register`方法的解读，请看我们之前的文章[BIO到NIO源码的一些事儿之NIO 上](https://juejin.im/post/5c2e23156fb9a049ff4e4009) 中**赋予Channel可被多路复用的能力**这一节的内容。

这里要记住的是`SelectableChannel`是对接`channel`特征(即`SelectionKey`)的关键所在，这有点类似于表设计，原本可以将特征什么的设定在一张表内，但为了操作更加具有针对性，即为了让代码功能更易于管理，就进行抽取并设计了第二张表，这个就有点像人体器官，整体上大家共同协作完成一件事，但器官内部自己专注于自己的主要特定功能，偶尔也具备其他器官的一些小功能。

由此，我们也就可以知道，`SelectionKey`表示一个`SelectableChannel`与`Selector`关联的标记，可以简单理解为一个`token`。就好比是我们做权限管理系统用户登录后前台会从后台拿到的一个`token`一样，用户可以凭借此`token`来访问操作相应的资源信息。
``` java
//java.nio.channels.spi.AbstractSelectableChannel#register
public final SelectionKey register(Selector sel, int ops, Object att)
    throws ClosedChannelException
{       ...
    synchronized (regLock) {
       ...
        synchronized (keyLock) {
           ...
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
结合上下两段源码，在每次`Selector`使用`register`方法注册`channel`时，都会创建并返回一个`SelectionKey`。
``` java
//sun.nio.ch.SelectorImpl#register
@Override
protected final SelectionKey register(AbstractSelectableChannel ch,
                                        int ops,
                                        Object attachment)
{
    if (!(ch instanceof SelChImpl))
        throw new IllegalSelectorException();
    SelectionKeyImpl k = new SelectionKeyImpl((SelChImpl)ch, this);
    k.attach(attachment);

    // register (if needed) before adding to key set
    implRegister(k);

    // add to the selector's key set, removing it immediately if the selector
    // is closed. The key is not in the channel's key set at this point but
    // it may be observed by a thread iterating over the selector's key set.
    keys.add(k);
    try {
        k.interestOps(ops);
    } catch (ClosedSelectorException e) {
        assert ch.keyFor(this) == null;
        keys.remove(k);
        k.cancel();
        throw e;
    }
    return k;
}
```
我们在[BIO到NIO源码的一些事儿之NIO 上](https://juejin.im/post/5c2e23156fb9a049ff4e4009) 中**赋予Channel可被多路复用的能力**这一节的内容知道，一旦注册到`Selector`上，`Channel`将一直保持注册直到其被解除注册。在解除注册的时候会解除`Selector`分配给`Channel`的所有资源。 
也就是`SelectionKey`在其调用`SelectionKey#channel`方法，或这个key所代表的`channel` 关闭，抑或此key所关联的`Selector`关闭之前，都是有效。我们在前面的文章分析中也知道，取消一个`SelectionKey`，不会立刻从`Selector`移除，它将被添加到`Selector`的`cancelledKeys`这个`Set`集合中，以便在下一次选择操作期间删除，我们可以通过`java.nio.channels.SelectionKey#isValid`判断一个`SelectionKey`是否有效。 

SelectionKey包含四个操作集，每个操作集用一个Int来表示，int值中的低四位的bit 用于表示`channel`支持的可选操作种类。
 ``` java
 
    /**
     * Operation-set bit for read operations.
     */
    public static final int OP_READ = 1 << 0;

    /**
     * Operation-set bit for write operations.
     */
    public static final int OP_WRITE = 1 << 2;

    /**
     * Operation-set bit for socket-connect operations.
     */
    public static final int OP_CONNECT = 1 << 3;

    /**
     * Operation-set bit for socket-accept operations.
     */
    public static final int OP_ACCEPT = 1 << 4;
 ```
## interestOps
通过`interestOps`来确定了`selector`在下一个选择操作的过程中将测试哪些操作类别的准备情况，操作事件是否是`channel`关注的。`interestOps` 在`SelectionKey`创建时，初始化为注册`Selector`时的ops值，这个值可通过`sun.nio.ch.SelectionKeyImpl#interestOps(int)`来改变，这点我们在`SelectorImpl#register`可以清楚的看到。
 ``` java
 //sun.nio.ch.SelectionKeyImpl
 public final class SelectionKeyImpl
    extends AbstractSelectionKey
{
    private static final VarHandle INTERESTOPS =
            ConstantBootstraps.fieldVarHandle(
                    MethodHandles.lookup(),
                    "interestOps",
                    VarHandle.class,
                    SelectionKeyImpl.class, int.class);

    private final SelChImpl channel;
    private final SelectorImpl selector;

    private volatile int interestOps;
    private volatile int readyOps;

    // registered events in kernel, used by some Selector implementations
    private int registeredEvents;

    // index of key in pollfd array, used by some Selector implementations
    private int index;

    SelectionKeyImpl(SelChImpl ch, SelectorImpl sel) {
        channel = ch;
        selector = sel;
    }
   ...
}
 ```
## readyOps
`readyOps`表示通过`Selector`检测到`channel`已经准备就绪的操作事件。在`SelectionKey`创建时（即上面源码所示），`readyOps`值为0，在`Selector`的`select`操作中可能会更新，但是需要注意的是我们不能直接调用来更新。

`SelectionKey`的`readyOps`表示一个`channel`已经为某些操作准备就绪，但不能保证在针对这个就绪事件类型的操作过程中不会发生阻塞，即该操作所在线程有可能会发生阻塞。在完成`select`操作后，大部分情况下会立即对`readyOps`更新，此时`readyOps`值最准确，如果外部的事件或在该`channel`有IO操作，`readyOps`可能不准确。所以，我们有看到其是`volatile`类型。

`SelectionKey`定义了所有的操作事件，但是具体`channel`支持的操作事件依赖于具体的`channel`，即具体问题具体分析。 
所有可选择的`channel`（即`SelectableChannel`的子类）都可以通过`SelectableChannel#validOps`方法，判断一个操作事件是否被`channel`所支持，即每个子类都会有对`validOps`的实现，返回一个数字，仅标识`channel`支持的哪些操作。尝试设置或测试一个不被`channel`所支持的操作设定，将会抛出相关的运行时异常。 
不同应用场景下，其所支持的`Ops`是不同的，摘取部分如下所示:
``` java
//java.nio.channels.SocketChannel#validOps
public final int validOps() {
    //即1|4|8  1101
    return (SelectionKey.OP_READ
            | SelectionKey.OP_WRITE
            | SelectionKey.OP_CONNECT);
}
//java.nio.channels.ServerSocketChannel#validOps
public final int validOps() {
    // 16
    return SelectionKey.OP_ACCEPT;
}
//java.nio.channels.DatagramChannel#validOps
public final int validOps() {
    // 1|4
    return (SelectionKey.OP_READ
            | SelectionKey.OP_WRITE);
}
```
如果需要经常关联一些我们程序中指定数据到`SelectionKey`，比如一个我们使用一个object表示上层的一种高级协议的状态，object用于通知实现协议处理器。所以，SelectionKey支持通过`attach`方法将一个对象附加到`SelectionKey`的`attachment`上。`attachment`可以通过`java.nio.channels.SelectionKey#attachment`方法进行访问。如果要取消该对象，则可以通过该种方式:` selectionKey.attach(null)`。

需要注意的是如果附加的对象不再使用，一定要人为清除，如果没有，假如此`SelectionKey`一直存在，由于此处属于强引用，那么垃圾回收器不会回收该对象，若不清除的话会成内存泄漏。

SelectionKey在由多线程并发使用时，是线程安全的。我们只需要知道，`Selector`的`select`操作会一直使用在调用该操作开始时当前的`interestOps`所设定的值。

# Selector探究

到现在为止，我们已经多多少少接触了`Selector`，其是一个什么样的角色，想必都很清楚了，那我们就在我们已经接触到的来进一步深入探究`Selector`的设计运行机制。
## Selector的open方法
从命名上就可以知道 `SelectableChannel`对象是依靠`Selector`来实现多路复用的。
我们可以通过调用`java.nio.channels.Selector#open`来创建一个`selector`对象:
``` java
//java.nio.channels.Selector#open
public static Selector open() throws IOException {
    return SelectorProvider.provider().openSelector();
}
```
关于这个`SelectorProvider.provider()`，其使用了根据所在系统的默认实现，我这里是windows系统，那么其默认实现为`sun.nio.ch.WindowsSelectorProvider`，这样，就可以调用基于相应系统的具体实现了。
``` java
//java.nio.channels.spi.SelectorProvider#provider
public static SelectorProvider provider() {
    synchronized (lock) {
        if (provider != null)
            return provider;
        return AccessController.doPrivileged(
            new PrivilegedAction<>() {
                public SelectorProvider run() {
                        if (loadProviderFromProperty())
                            return provider;
                        if (loadProviderAsService())
                            return provider;
                        provider = sun.nio.ch.DefaultSelectorProvider.create();
                        return provider;
                    }
                });
    }
}
//sun.nio.ch.DefaultSelectorProvider
public class DefaultSelectorProvider {

/**
 * Prevent instantiation.
 */
private DefaultSelectorProvider() { }

/**
 * Returns the default SelectorProvider.
 */
public static SelectorProvider create() {
    return new sun.nio.ch.WindowsSelectorProvider();
}

}
```
基于windows来讲，selector这里最终会使用`sun.nio.ch.WindowsSelectorImpl`来做一些核心的逻辑。
``` java
public class WindowsSelectorProvider extends SelectorProviderImpl {

    public AbstractSelector openSelector() throws IOException {
        return new WindowsSelectorImpl(this);
    }
}
```
这里，我们需要来看一下`WindowsSelectorImpl`的构造函数:
``` java
//sun.nio.ch.WindowsSelectorImpl#WindowsSelectorImpl
WindowsSelectorImpl(SelectorProvider sp) throws IOException {
    super(sp);
    pollWrapper = new PollArrayWrapper(INIT_CAP);
    wakeupPipe = Pipe.open();
    wakeupSourceFd = ((SelChImpl)wakeupPipe.source()).getFDVal();

    // Disable the Nagle algorithm so that the wakeup is more immediate
    SinkChannelImpl sink = (SinkChannelImpl)wakeupPipe.sink();
    (sink.sc).socket().setTcpNoDelay(true);
    wakeupSinkFd = ((SelChImpl)sink).getFDVal();

    pollWrapper.addWakeupSocket(wakeupSourceFd, 0);
}
```
我们由`Pipe.open()`就可知道`selector`会保持打开的状态，直到其调用它的`close`方法:
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
//sun.nio.ch.WindowsSelectorImpl#implClose
@Override
protected void implClose() throws IOException {
    assert !isOpen();
    assert Thread.holdsLock(this);

    // prevent further wakeup
    synchronized (interruptLock) {
        interruptTriggered = true;
    }

    wakeupPipe.sink().close();
    wakeupPipe.source().close();
    pollWrapper.free();

    // Make all remaining helper threads exit
    for (SelectThread t: threads)
            t.makeZombie();
    startLock.startThreads();
}

```
可以看到，前面的`wakeupPipe`在close方法中关闭掉了。这里的close方法中又涉及了`wakeupPipe.sink()`与`wakeupPipe.source()`的关闭与`pollWrapper.free()`的释放，此处也是我们本篇的难点所在，这里，我们来看看它们到底是什么样的存在。
首先，我们对`WindowsSelectorImpl(SelectorProvider sp)`这个构造函数做下梳理:
- 创建一个`PollArrayWrapper`对象（`pollWrapper`）；
- `Pipe.open()`打开一个管道；
- 拿到`wakeupSourceFd`和`wakeupSinkFd`两个文件描述符；
- 把pipe内Source端的文件描述符（`wakeupSourceFd`）放到`pollWrapper`里；
### Pipe.open()的解惑
这里我们会有疑惑，为什么要创建一个管道，它是用来做什么的。

我们来看`Pipe.open()`源码实现:
``` java
//java.nio.channels.Pipe#open
public static Pipe open() throws IOException {
    return SelectorProvider.provider().openPipe();
}
//sun.nio.ch.SelectorProviderImpl#openPipe
public Pipe openPipe() throws IOException {
    return new PipeImpl(this);
}
//sun.nio.ch.PipeImpl#PipeImpl
PipeImpl(final SelectorProvider sp) throws IOException {
    try {
        AccessController.doPrivileged(new Initializer(sp));
    } catch (PrivilegedActionException x) {
        throw (IOException)x.getCause();
    }
}
private class Initializer
implements PrivilegedExceptionAction<Void>
{

private final SelectorProvider sp;

private IOException ioe = null;

private Initializer(SelectorProvider sp) {
    this.sp = sp;
}

@Override
public Void run() throws IOException {
    LoopbackConnector connector = new LoopbackConnector();
    connector.run();
    if (ioe instanceof ClosedByInterruptException) {
        ioe = null;
        Thread connThread = new Thread(connector) {
            @Override
            public void interrupt() {}
        };
        connThread.start();
        for (;;) {
            try {
                connThread.join();
                break;
            } catch (InterruptedException ex) {}
        }
        Thread.currentThread().interrupt();
    }

    if (ioe != null)
        throw new IOException("Unable to establish loopback connection", ioe);

    return null;
}
```
从上述源码我们可以知道，创建了一个`PipeImpl`对象， 在`PipeImpl`的构造函数里会执行`AccessController.doPrivileged`，在它调用后紧接着会执行`Initializer`的`run`方法:
``` java
//sun.nio.ch.PipeImpl.Initializer.LoopbackConnector
private class LoopbackConnector implements Runnable {

    @Override
    public void run() {
        ServerSocketChannel ssc = null;
        SocketChannel sc1 = null;
        SocketChannel sc2 = null;

        try {
            // Create secret with a backing array.
            ByteBuffer secret = ByteBuffer.allocate(NUM_SECRET_BYTES);
            ByteBuffer bb = ByteBuffer.allocate(NUM_SECRET_BYTES);

            // Loopback address
            InetAddress lb = InetAddress.getLoopbackAddress();
            assert(lb.isLoopbackAddress());
            InetSocketAddress sa = null;
            for(;;) {
                // Bind ServerSocketChannel to a port on the loopback
                // address
                if (ssc == null || !ssc.isOpen()) {
                    ssc = ServerSocketChannel.open();
                    ssc.socket().bind(new InetSocketAddress(lb, 0));
                    sa = new InetSocketAddress(lb, ssc.socket().getLocalPort());
                }

                // Establish connection (assume connections are eagerly
                // accepted)
                sc1 = SocketChannel.open(sa);
                RANDOM_NUMBER_GENERATOR.nextBytes(secret.array());
                do {
                    sc1.write(secret);
                } while (secret.hasRemaining());
                secret.rewind();

                // Get a connection and verify it is legitimate
                sc2 = ssc.accept();
                do {
                    sc2.read(bb);
                } while (bb.hasRemaining());
                bb.rewind();

                if (bb.equals(secret))
                    break;

                sc2.close();
                sc1.close();
            }

            // Create source and sink channels
            source = new SourceChannelImpl(sp, sc1);
            sink = new SinkChannelImpl(sp, sc2);
        } catch (IOException e) {
            try {
                if (sc1 != null)
                    sc1.close();
                if (sc2 != null)
                    sc2.close();
            } catch (IOException e2) {}
            ioe = e;
        } finally {
            try {
                if (ssc != null)
                    ssc.close();
            } catch (IOException e2) {}
        }
    }
}
}
```
这里即为创建`pipe`的过程，`windows`下的实现是创建两个本地的`socketChannel`，然后连接（连接的过程通过写一个随机数据做两个socket的连接校验），两个`socketChannel`分别实现了管道`pipe`的`source`与`sink`端。
而我们依然不清楚这个`pipe`到底干什么用的，
假如大家熟悉系统调用的`C/C++`的话，就可以知道，一个阻塞在`select`上的线程有以下三种方式可以被唤醒：

1. 有数据可读/写，或出现异常。
2. 阻塞时间到，即`time out`。
3. 收到一个`non-block`的信号。可由`kill`或`pthread_kill`发出。

所以，`Selector.wakeup()`要唤醒阻塞的`select`，那么也只能通过这三种方法，其中：

- 第二种方法可以排除，因为`select`一旦阻塞，无法修改其`time out`时间。
- 而第三种看来只能在`Linux`上实现，`Windows`上没有这种信号通知的机制。

看来只有第一种方法了。假如我们多次调用`Selector.open()`，那么在`Windows`上会每调用一次，就会建立一对自己和自己的`loopback`的`TCP`连接；在Linux上的话，每调用一次，会开一对`pipe`（pipe在Linux下一般都成对打开），到这里，估计我们能够猜得出来——那就是如果想要唤醒`select`，只需要朝着自己的这个`loopback`连接发点数据过去，于是，就可以唤醒阻塞在`select`上的线程了。

我们对上面所述做下总结:在`Windows`下，`Java`虚拟机在`Selector.open()`时会自己和自己建立`loopback`的`TCP`连接；在`Linux`下，`Selector`会创建`pipe`。这主要是为了`Selector.wakeup()`可以方便唤醒阻塞在`select()`系统调用上的线程（通过向自己所建立的`TCP`链接和管道上随便写点什么就可以唤醒阻塞线程）。

## PollArrayWrapper解读
在`WindowsSelectorImpl`构造器最后，我们看到这一句代码:`pollWrapper.addWakeupSocket(wakeupSourceFd, 0);`，即把pipe内Source端的文件描述符（`wakeupSourceFd`）放到`pollWrapper`里。`pollWrapper`作为`PollArrayWrapper`的实例，它到底是什么，这一节，我们就来对其探索一番。
``` java
class PollArrayWrapper {

    private AllocatedNativeObject pollArray; // The fd array

    long pollArrayAddress; // pollArrayAddress

    @Native private static final short FD_OFFSET     = 0; // fd offset in pollfd
    @Native private static final short EVENT_OFFSET  = 4; // events offset in pollfd

    static short SIZE_POLLFD = 8; // sizeof pollfd struct

    private int size; // Size of the pollArray

    PollArrayWrapper(int newSize) {
        int allocationSize = newSize * SIZE_POLLFD;
        pollArray = new AllocatedNativeObject(allocationSize, true);
        pollArrayAddress = pollArray.address();
        this.size = newSize;
    }

    ...

    // Access methods for fd structures
    void putDescriptor(int i, int fd) {
        pollArray.putInt(SIZE_POLLFD * i + FD_OFFSET, fd);
    }

    void putEventOps(int i, int event) {
        pollArray.putShort(SIZE_POLLFD * i + EVENT_OFFSET, (short)event);
    }
    ...
   // Adds Windows wakeup socket at a given index.
    void addWakeupSocket(int fdVal, int index) {
        putDescriptor(index, fdVal);
        putEventOps(index, Net.POLLIN);
    }
}
```
这里将`wakeupSourceFd`的`POLLIN`事件标识为`pollArray`的`EventOps`的对应的值，这里使用的是unsafe直接操作的内存，也就是相对于这个`pollArray`所在内存地址的偏移量`SIZE_POLLFD * i + EVENT_OFFSET`这个位置上写入`Net.POLLIN`所代表的值，即参考下面本地方法相关源码所展示的值。`putDescriptor`同样是这种类似操作。当`sink端`有数据写入时，`source`对应的文件描述符`wakeupSourceFd`就会处于就绪状态。
``` java
//java.base/windows/native/libnio/ch/nio_util.h
    /* WSAPoll()/WSAPOLLFD and the corresponding constants are only defined   */
    /* in Windows Vista / Windows Server 2008 and later. If we are on an      */
    /* older release we just use the Solaris constants as this was previously */
    /* done in PollArrayWrapper.java.                                         */
    #define POLLIN       0x0001
    #define POLLOUT      0x0004
    #define POLLERR      0x0008
    #define POLLHUP      0x0010
    #define POLLNVAL     0x0020
    #define POLLCONN     0x0002
```
`AllocatedNativeObject`这个类的父类有大量的`unsafe`类的操作，这些都是直接基于内存级别的操作。从其父类的构造器中，我们能也清楚的看到`pollArray`是通过`unsafe.allocateMemory(size + ps)`分配的一块系统内存。
``` java
class AllocatedNativeObject                             // package-private
    extends NativeObject
{
    /**
     * Allocates a memory area of at least {@code size} bytes outside of the
     * Java heap and creates a native object for that area.
     */
    AllocatedNativeObject(int size, boolean pageAligned) {
        super(size, pageAligned);
    }

    /**
     * Frees the native memory area associated with this object.
     */
    synchronized void free() {
        if (allocationAddress != 0) {
            unsafe.freeMemory(allocationAddress);
            allocationAddress = 0;
        }
    }

}
//sun.nio.ch.NativeObject#NativeObject(int, boolean)
protected NativeObject(int size, boolean pageAligned) {
        if (!pageAligned) {
            this.allocationAddress = unsafe.allocateMemory(size);
            this.address = this.allocationAddress;
        } else {
            int ps = pageSize();
            long a = unsafe.allocateMemory(size + ps);
            this.allocationAddress = a;
            this.address = a + ps - (a & (ps - 1));
        }
    }
```
至此，我们算是完成了对`Selector.open()`的解读，其主要任务就是完成建立`Pipe`，并把`pipe` `source`端的`wakeupSourceFd`放入`pollArray`中，这个`pollArray`是`Selector`完成其角色任务的枢纽。本篇主要围绕Windows的实现来进行分析，即在windows下通过两个连接的`socketChannel`实现了`Pipe`，`linux`下则直接使用系统的`pipe`即可。

## SelectionKey在selector中的管理

### SelectionKey在selector中注册

所谓的注册，其实就是将一个对象放到注册地对象内的一个容器字段上，这个字段可以是数组，队列，也可以是一个set集合，也可以是一个list。这里，同样是这样，只不过，其需要有个返回值，那么把这个要放入集合的对象返回即可。
``` java
//sun.nio.ch.SelectorImpl#register
@Override
protected final SelectionKey register(AbstractSelectableChannel ch,
                                        int ops,
                                        Object attachment)
{
    if (!(ch instanceof SelChImpl))
        throw new IllegalSelectorException();
    SelectionKeyImpl k = new SelectionKeyImpl((SelChImpl)ch, this);
    k.attach(attachment);

    // register (if needed) before adding to key set
    implRegister(k);

    // add to the selector's key set, removing it immediately if the selector
    // is closed. The key is not in the channel's key set at this point but
    // it may be observed by a thread iterating over the selector's key set.
    keys.add(k);
    try {
        k.interestOps(ops);
    } catch (ClosedSelectorException e) {
        assert ch.keyFor(this) == null;
        keys.remove(k);
        k.cancel();
        throw e;
    }
    return k;
}
//sun.nio.ch.WindowsSelectorImpl#implRegister
@Override
protected void implRegister(SelectionKeyImpl ski) {
    ensureOpen();
    synchronized (updateLock) {
        newKeys.addLast(ski);
    }
}
```
这段代码我们之前已经有看过，这里我们再次温习下。
首先会新建一个`SelectionKeyImpl`对象，这个对象就是对`Channel`的包装，不仅如此，还顺带把当前这个`Selector`对象给收了进去，这样，我们也可以通过`SelectionKey`的对象来拿到其对应的`Selector`对象。

接着，基于`windows`平台实现的`implRegister`，先通过`ensureOpen()`来确保该`Selector`是打开的。接着将这个`SelectionKeyImpl`加入到`WindowsSelectorImpl`内针对于**新注册**SelectionKey进行管理的`newKeys`之中，`newKeys`是一个`ArrayDeque`对象。对于`ArrayDeque`有不懂的，可以参考[Java 容器源码分析之 Deque 与 ArrayDeque](https://github.com/muyinchen/woker/blob/master/JAVA8/jdk%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/Java%20%E5%AE%B9%E5%99%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8B%20Deque%20%E4%B8%8E%20ArrayDeque.md)这篇文章。

然后再将此这个`SelectionKeyImpl`加入到`sun.nio.ch.SelectorImpl#keys`中去，这个`Set<SelectionKey>`集合代表那些已经注册到当前这个`Selector`对象上的`SelectionKey`集合。我们来看`sun.nio.ch.SelectorImpl`的构造函数:
``` java
//sun.nio.ch.SelectorImpl#SelectorImpl
protected SelectorImpl(SelectorProvider sp) {
    super(sp);
    keys = ConcurrentHashMap.newKeySet();
    selectedKeys = new HashSet<>();
    publicKeys = Collections.unmodifiableSet(keys);
    publicSelectedKeys = Util.ungrowableSet(selectedKeys);
}
```
也就是说，这里的`publicKeys`就来源于`keys`，只是`publicKeys`属于只读的，我们想要知道当前`Selector`对象上所注册的`keys`，就可以调用`sun.nio.ch.SelectorImpl#keys`来得到:
``` java
//sun.nio.ch.SelectorImpl#keys
@Override
public final Set<SelectionKey> keys() {
    ensureOpen();
    return publicKeys;
}
```
再回到这个构造函数中，`selectedKeys`，顾名思义，其属于已选择Keys，即前一次操作期间，已经准备就绪的`Channel`所对应的`SelectionKey`。此集合为`keys`的子集。通过`selector.selectedKeys()`获取。
``` java
//sun.nio.ch.SelectorImpl#selectedKeys
@Override
public final Set<SelectionKey> selectedKeys() {
    ensureOpen();
    return publicSelectedKeys;
}
```
我们看到其返回的是`publicSelectedKeys`，针对这个字段里的元素操作可以做删除，但不能做增加。
在前面的内容中，我们有涉及到`SelectionKey`的取消，所以，我们在`java.nio.channels.spi.AbstractSelector`方法内，是有定义`cancelledKeys`的，也是一个`HashSet`对象。其代表已经被取消但尚未取消注册(deregister)的`SelectionKey`。此Set集合无法直接访问，同样，它也是keys()的子集。

对于新的`Selector`实例，上面几个集合均为空。由上面展示的源码可知，通过`channel.register`将`SelectionKey`添加`keys`中，此为key的来源。
如果某个`selectionKey.cancel()`被调用,那么此key将会被添加到`cancelledKeys`这个集合中，然后在下一次调用selector `select`方法期间，此时`canceldKeys`不为空，将会触发此`SelectionKey`的`deregister`操作(释放资源,并从`keys`中移除)。无论通过`channel.close()`还是通过`selectionKey.cancel()`，都会导致`SelectionKey`被加入到`cannceldKey`中.

每次选择操作(select)期间，都可以将key添加到`selectedKeys`中或者将从`cancelledKeys`中移除。

### Selector的select方法的解读

了解了上面的这些，我们来进入到`select`方法中，观察下它的细节。由`Selector`的api可知，`select`操作有两种形式，一种为
select(),selectNow(),select(long timeout);另一种为`select(Consumer<SelectionKey> action, long timeout)`，`select(Consumer<SelectionKey> action)`，`selectNow(Consumer<SelectionKey> action)`。后者为JDK11新加入的api，主要针对那些准备好进行I/O操作的channels在select过程中对相应的key进行的一个字的自定义的一个操作。
需要注意的是，有`Consumer<SelectionKey> action`参数的select操作是阻塞的，只有在选择了至少一个Channel的情况下，才会调用此`Selector`实例的`wakeup`方法来唤醒，同样，其所在线程被打断也可以。

``` java
//sun.nio.ch.SelectorImpl
@Override
public final int select(long timeout) throws IOException {
    if (timeout < 0)
        throw new IllegalArgumentException("Negative timeout");
    return lockAndDoSelect(null, (timeout == 0) ? -1 : timeout);
}

//sun.nio.ch.SelectorImpl
@Override
public final int select(Consumer<SelectionKey> action, long timeout)
    throws IOException
{
    Objects.requireNonNull(action);
    if (timeout < 0)
        throw new IllegalArgumentException("Negative timeout");
    return lockAndDoSelect(action, (timeout == 0) ? -1 : timeout);
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
                    return doSelect(action, timeout);
                }
            } finally {
                inSelect = false;
            }
        }
    }

```
我们可以观察，无论哪种，它们最后都落在了`lockAndDoSelect`这个方法上，最终会执行特定系统上的`doSelect(action, timeout)`实现。
这里我们以`sun.nio.ch.WindowsSelectorImpl#doSelect`为例来讲述其操作执行的步骤:

``` java
// sun.nio.ch.WindowsSelectorImpl#doSelect
@Override
protected int doSelect(Consumer<SelectionKey> action, long timeout)
    throws IOException
    {
        assert Thread.holdsLock(this);
        this.timeout = timeout; // set selector timeout
        processUpdateQueue();  // <1>
        processDeregisterQueue(); // <2>
        if (interruptTriggered) {
            resetWakeupSocket();
            return 0;
        }
        // Calculate number of helper threads needed for poll. If necessary
        // threads are created here and start waiting on startLock
        adjustThreadsCount();
        finishLock.reset(); // reset finishLock
        // Wakeup helper threads, waiting on startLock, so they start polling.
        // Redundant threads will exit here after wakeup.
        startLock.startThreads();
        // do polling in the main thread. Main thread is responsible for
        // first MAX_SELECTABLE_FDS entries in pollArray.
        try {
            begin();
            try {
                subSelector.poll();  // <3>
            } catch (IOException e) {
                finishLock.setException(e); // Save this exception
            }
            // Main thread is out of poll(). Wakeup others and wait for them
            if (threads.size() > 0)
                finishLock.waitForHelperThreads();
          } finally {
              end();
          }
        // Done with poll(). Set wakeupSocket to nonsignaled  for the next run.
        finishLock.checkForException();
        processDeregisterQueue();  // <4>
        int updated = updateSelectedKeys(action); // <5>
        // Done with poll(). Set wakeupSocket to nonsignaled  for the next run.
        resetWakeupSocket(); // <6>
        return updated;
    }
```
### processUpdateQueue解读

1. 首先通过相应操作系统实现类（此处是WindowsSelectorImpl）的具体实现我们可以知道，通过`<1>` 处的 `processUpdateQueue()`获得关于每个剩余`Channel`（有些Channel取消了）的在此刻的`interestOps`，这里包括新注册的和`updateKeys`，并对其进行`pollWrapper`的管理操作。

   > - 即对于新注册的`SelectionKeyImpl`，我们在相对于这个`pollArray`所在内存地址的偏移量`SIZE_POLLFD * totalChannels + FD_OFFSET`与`SIZE_POLLFD * totalChannels + EVENT_OFFSET`分别存入`SelectionKeyImpl`的文件描述符`fd`与其对应的`EventOps`（初始为0）。
   >
   > - 对`updateKeys`，因为是其之前已经在`pollArray`的某个相对位置上存储过，这里我们还需要对拿到的key的有效性进行判断，如果有效，只需要将正在操作的这个`SelectionKeyImpl`对象的`interestOps`写入到在`pollWrapper`中的存放它的`EventOps`位置上。

   > **注意**: 在对`newKeys`进行key的有效性判断之后，如果有效，会调用`growIfNeeded()`方法，这里首先会判断`channelArray.length == totalChannels`，此为一个`SelectionKeyImpl`的数组，初始容量大小为8。`channelArray`其实就是方便`Selector`管理在册`SelectionKeyImpl`数量的一个数组而已，通过判断它的数组长度大小，如果和`totalChannels`(初始值为1)相等，不仅仅是为了`channelArray`扩容，更重要的是为了辅助`pollWrapper`，让`pollWrapper`扩容才是这里的目的所在。
   > 而当`totalChannels % MAX_SELECTABLE_FDS == 0`时，则多开一个线程处理`selector`。`windows`上`select`系统调用有最大文件描述符限制，一次只能轮询`1024`个文件描述符，如果多于1024个，需要多线程进行轮询。通过`ski.setIndex(totalChannels)`选择键记录下在数组中的索引位置`SelectionKeyImpl`选择键的映射关系，以待后续使用。同时调用`pollWrapper.addWakeupSocket(wakeupSourceFd, totalChannels)`在相对于这个`pollArray`所在内存地址的偏移量`SIZE_POLLFD * totalChannels + FD_OFFSET`这个位置上写入`wakeupSourceFd`所代表的`fdVal`值。这样在新起的线程就可以通过`MAX_SELECTABLE_FDS`来确定这个用来监控的`wakeupSourceFd`。
``` java
   /**
    * sun.nio.ch.WindowsSelectorImpl#processUpdateQueue
    * Process new registrations and changes to the interest ops.
    */
private void processUpdateQueue() {
    assert Thread.holdsLock(this);

    synchronized (updateLock) {
        SelectionKeyImpl ski;

        // new registrations
        while ((ski = newKeys.pollFirst()) != null) {
            if (ski.isValid()) {
                growIfNeeded();
                channelArray[totalChannels] = ski;
                ski.setIndex(totalChannels);
                pollWrapper.putEntry(totalChannels, ski);
                totalChannels++;
                MapEntry previous = fdMap.put(ski);
                assert previous == null;
            }
        }

        // changes to interest ops
        while ((ski = updateKeys.pollFirst()) != null) {
            int events = ski.translateInterestOps();
            int fd = ski.getFDVal();
            if (ski.isValid() && fdMap.containsKey(fd)) {
                int index = ski.getIndex();
                assert index >= 0 && index < totalChannels;
                pollWrapper.putEventOps(index, events);
            }
        }
    }
}

//sun.nio.ch.PollArrayWrapper#putEntry
// Prepare another pollfd struct for use.
void putEntry(int index, SelectionKeyImpl ski) {
    putDescriptor(index, ski.getFDVal());
    putEventOps(index, 0);
}
//sun.nio.ch.WindowsSelectorImpl#growIfNeeded
private void growIfNeeded() {
    if (channelArray.length == totalChannels) {
        int newSize = totalChannels * 2; // Make a larger array
        SelectionKeyImpl temp[] = new SelectionKeyImpl[newSize];
        System.arraycopy(channelArray, 1, temp, 1, totalChannels - 1);
        channelArray = temp;
        pollWrapper.grow(newSize);
    }
    if (totalChannels % MAX_SELECTABLE_FDS == 0) { // more threads needed
        pollWrapper.addWakeupSocket(wakeupSourceFd, totalChannels);
        totalChannels++;
        threadsCount++;
    }
}
// Initial capacity of the poll array
private final int INIT_CAP = 8;
// Maximum number of sockets for select().
// Should be INIT_CAP times a power of 2
private static final int MAX_SELECTABLE_FDS = 1024;

// The list of SelectableChannels serviced by this Selector. Every mod
// MAX_SELECTABLE_FDS entry is bogus, to align this array with the poll
// array,  where the corresponding entry is occupied by the wakeupSocket
private SelectionKeyImpl[] channelArray = new SelectionKeyImpl[INIT_CAP];
// The number of valid entries in  poll array, including entries occupied
// by wakeup socket handle.
private int totalChannels = 1;

//sun.nio.ch.PollArrayWrapper#grow
// Grows the pollfd array to new size
void grow(int newSize) {
    PollArrayWrapper temp = new PollArrayWrapper(newSize);
    for (int i = 0; i < size; i++)
        replaceEntry(this, i, temp, i);
    pollArray.free();
    pollArray = temp.pollArray;
    this.size = temp.size;
    pollArrayAddress = pollArray.address();
}

// Maps file descriptors to their indices in  pollArray
private static final class FdMap extends HashMap<Integer, MapEntry> {
    static final long serialVersionUID = 0L;
    private MapEntry get(int desc) {
        return get(Integer.valueOf(desc));
    }
    private MapEntry put(SelectionKeyImpl ski) {
        return put(Integer.valueOf(ski.getFDVal()), new MapEntry(ski));
    }
    private MapEntry remove(SelectionKeyImpl ski) {
        Integer fd = Integer.valueOf(ski.getFDVal());
        MapEntry x = get(fd);
        if ((x != null) && (x.ski.channel() == ski.channel()))
            return remove(fd);
        return null;
    }
}

// class for fdMap entries
private static final class MapEntry {
    final SelectionKeyImpl ski;
    long updateCount = 0;
    MapEntry(SelectionKeyImpl ski) {
        this.ski = ski;
    }
}
private final FdMap fdMap = new FdMap();
```

#### processDeregisterQueue解读

2. 接着通过`上面WindowsSelectorImpl#doSelect展示源码中<2>` 处的 `processDeregisterQueue()`。
   - 对`cancelledKeys`进行清除，遍历`cancelledKeys`，并对每个`key`进行`deregister`操作，然后从`cancelledKeys`集合中删除，从`keys`集合与`selectedKeys`中删除，以此来释放引用，方便gc回收，
   - 其内调用`implDereg`方法，将会从`channelArray`中移除对应的`Channel`代表的`SelectionKeyImpl`，调整`totalChannels`和线程数，从`map`和`keys`中移除`SelectionKeyImpl`，移除`Channel`上的`SelectionKeyImpl`并关闭`Channel`。
   - 同时还发现该`processDeregisterQueue()`方法在调用`poll`方法前后都进行调用，这是确保能够正确处理在调用`poll`方法阻塞的这一段时间之内取消的键能被及时清理。
   - 最后，还会判断这个`cancelledKey`所代表的`channel`是否打开和解除注册，如果关闭并解除注册，则应该将相应的文件描述符对应占用的资源给关闭掉。
``` java
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
//sun.nio.ch.WindowsSelectorImpl#implDereg
@Override
protected void implDereg(SelectionKeyImpl ski) {
    assert !ski.isValid();
    assert Thread.holdsLock(this);

    if (fdMap.remove(ski) != null) {
        int i = ski.getIndex();
        assert (i >= 0);

        if (i != totalChannels - 1) {
            // Copy end one over it
            SelectionKeyImpl endChannel = channelArray[totalChannels-1];
            channelArray[i] = endChannel;
            endChannel.setIndex(i);
            pollWrapper.replaceEntry(pollWrapper, totalChannels-1, pollWrapper, i);
        }
        ski.setIndex(-1);

        channelArray[totalChannels - 1] = null;
        totalChannels--;
        if (totalChannels != 1 && totalChannels % MAX_SELECTABLE_FDS == 1) {
            totalChannels--;
            threadsCount--; // The last thread has become redundant.
        }
    }
}

//sun.nio.ch.SocketChannelImpl#kill
@Override
public void kill() throws IOException {
    synchronized (stateLock) {
        if (state == ST_KILLPENDING) {
            state = ST_KILLED;
            nd.close(fd);
        }
    }
}
//C:/Program Files/Java/jdk-11.0.1/lib/src.zip!/java.base/sun/nio/ch/SocketChannelImpl.java:1126
static {
    IOUtil.load();
    nd = new SocketDispatcher();
}
//sun.nio.ch.SocketDispatcher#close
void close(FileDescriptor fd) throws IOException {
    close0(fd);
}
```
#### adjustThreadsCount解读

3. 接着我们来看到`上面WindowsSelectorImpl#doSelect`展示源码中`adjustThreadsCount()`方法的调用。
    - 前面有提到如果`totalChannels % MAX_SELECTABLE_FDS == 0`，则多开一个线程处理`selector`。这里就是根据**分配的线程数量值**来增加或减少线程，其实就是针对操作系统的最大`select`操作的文件描述符限制对线程个数进行调整。
    - 我们来观察所建线程做了什么事情，即观察`SelectThread`的`run`方法实现。通过观察其源码可以看到它首先是`while (true)`，通过`startLock.waitForStart(this)`来控制该线程是否运行还是等待，运行状态的话，会进而调用`subSelector.poll(index)`（这个我们后面内容详细解读），
    - 当此线程`poll`结束，而且相对于当前主线程假如有多条`SelectThread`子线程的话，当前这条`SelectThread`线程第一个结束`poll`的话，就调用`finishLock.threadFinished()`来通知主线程。在刚新建这个线程并调用其`run`方法的时候，此时`lastRun = 0`，在第一次启动的时候`sun.nio.ch.WindowsSelectorImpl.StartLock#runsCounter`同样为0，所以会调用`startLock.wait()`进而进入等待状态。 

> **注意：**
> - `sun.nio.ch.WindowsSelectorImpl.StartLock`同样会判断当前其所检测的线程是否废弃，废弃的话就返回`true`，这样被检测线程也就能跳出其内run方法的`while`循环从而结束线程运行。
> - 在调整线程的时候（调用`adjustThreadsCount`方法）与`Selector`调用`close`方法会间接调用到`sun.nio.ch.WindowsSelectorImpl#implClose`，这两个方法都会涉及到`Selector`线程的释放，即调用`sun.nio.ch.WindowsSelectorImpl.SelectThread#makeZombie`。
> - `finishLock.threadFinished()`会调用`wakeup()`方法来通知主线程，这里，我们可以学到一个细节，如果线程正阻塞在`select`方法上，就可以调用`wakeup`方法会使阻塞的选择操作立即返回，通过`Windows`的相关实现，原理其实是向`pipe`的`sink`端写入了一个字节，`source`文件描述符就会处于就绪状态，`poll`方法会返回，从而导致`select`方法返回。而在其他solaris或者linux系统上其实采用系统调用`pipe`来完成管道的创建，相当于直接用了系统的管道。通过`wakeup()`相关实现还可以看出，调用`wakeup`会设置`interruptTriggered`的标志位，所以连续多次调用`wakeup`的效果等同于一次调用，不会引起无所谓的bug出现。

``` java
//sun.nio.ch.WindowsSelectorImpl#adjustThreadsCount
// After some channels registered/deregistered, the number of required
// helper threads may have changed. Adjust this number.
private void adjustThreadsCount() {
    if (threadsCount > threads.size()) {
        // More threads needed. Start more threads.
        for (int i = threads.size(); i < threadsCount; i++) {
            SelectThread newThread = new SelectThread(i);
            threads.add(newThread);
            newThread.setDaemon(true);
            newThread.start();
        }
    } else if (threadsCount < threads.size()) {
        // Some threads become redundant. Remove them from the threads List.
        for (int i = threads.size() - 1 ; i >= threadsCount; i--)
            threads.remove(i).makeZombie();
    }
}

//sun.nio.ch.WindowsSelectorImpl.SelectThread
// Represents a helper thread used for select.
private final class SelectThread extends Thread {
    private final int index; // index of this thread
    final SubSelector subSelector;
    private long lastRun = 0; // last run number
    private volatile boolean zombie;
    // Creates a new thread
    private SelectThread(int i) {
        super(null, null, "SelectorHelper", 0, false);
        this.index = i;
        this.subSelector = new SubSelector(i);
        //make sure we wait for next round of poll
        this.lastRun = startLock.runsCounter;
    }
    void makeZombie() {
        zombie = true;
    }
    boolean isZombie() {
        return zombie;
    }
    public void run() {
        while (true) { // poll loop
            // wait for the start of poll. If this thread has become
            // redundant, then exit.
            if (startLock.waitForStart(this))
                return;
            // call poll()
            try {
                subSelector.poll(index);
            } catch (IOException e) {
                // Save this exception and let other threads finish.
                finishLock.setException(e);
            }
            // notify main thread, that this thread has finished, and
            // wakeup others, if this thread is the first to finish.
            finishLock.threadFinished();
        }
    }
}

// sun.nio.ch.WindowsSelectorImpl.FinishLock#threadFinished
// Each helper thread invokes this function on finishLock, when
// the thread is done with poll().
private synchronized void threadFinished() {
    if (threadsToFinish == threads.size()) { // finished poll() first
        // if finished first, wakeup others
        wakeup();
    }
    threadsToFinish--;
    if (threadsToFinish == 0) // all helper threads finished poll().
        notify();             // notify the main thread
}

//sun.nio.ch.WindowsSelectorImpl#wakeup
@Override
public Selector wakeup() {
    synchronized (interruptLock) {
        if (!interruptTriggered) {
            setWakeupSocket();
            interruptTriggered = true;
        }
    }
    return this;
}
//sun.nio.ch.WindowsSelectorImpl#setWakeupSocket
// Sets Windows wakeup socket to a signaled state.
private void setWakeupSocket() {
    setWakeupSocket0(wakeupSinkFd);
}
private native void setWakeupSocket0(int wakeupSinkFd);

JNIEXPORT void JNICALL
Java_sun_nio_ch_WindowsSelectorImpl_setWakeupSocket0(JNIEnv *env, jclass this,
                                                jint scoutFd)
{
    /* Write one byte into the pipe */
    const char byte = 1;
    send(scoutFd, &byte, 1, 0);
}
```
#### subSelector的poll方法解读

4. `subSelector.poll()` 是select的核心，由`native`函数`poll0`实现，并把`pollWrapper.pollArrayAddress`作为参数传给`poll0`，`readFds`、`writeFds` 和`exceptFds`数组用来保存底层`select`的结果，数组的第一个位置都是存放发生事件的`socket`的总数，其余位置存放发生事件的`socket`句柄`fd`。
    我们通过下面的代码可知:
    这个`poll0()`会监听`pollWrapper`中的`FD`有没有数据进出，这里会造成`IO`阻塞，直到有数据读写事件发生。由于`pollWrapper`中保存的也有`ServerSocketChannel`的`FD`，所以只要`ClientSocket`发一份数据到`ServerSocket`,那么`poll0()`就会返回；又由于`pollWrapper`中保存的也有`pipe`的`write`端的`FD`，所以只要`pipe`的`write`端向`FD`发一份数据，也会造成`poll0()`返回；如果这两种情况都没有发生，那么`poll0()`就一直阻塞，也就是`selector.select()`会一直阻塞；如果有任何一种情况发生，那么`selector.select()`就会返回，所有在`SelectThread`的`run()`里要用`while (true) {}`，这样就可以保证在`selector`接收到数据并处理完后继续监听`poll()`;

> 可以看出，NIO依然是阻塞式的IO，那么它和BIO的区别究竟在哪呢。
> 其实它的区别在于阻塞的位置不同，`BIO`是阻塞在`read`方法(recvfrom)，而`NIO`阻塞在`select`方法。那么这样做有什么好处呢。如果单纯的改变阻塞的位置，自然是没有什么变化的，但`epoll等`的实现的巧妙之处就在于，它利用回调机制，让监听能够只需要知晓哪些`socket`上的数据已经准备好了，只需要处理这些线程上面的数据就行了。采用`BIO`，假设有`1000`个连接，需要开`1000`个线程，然后有`1000`个`read`的位置在阻塞(我们在讲解BIO部分已经通过Demo体现)，采用`NIO`编程，只需要**1**个线程，它利用`select`的轮询策略配合`epoll`的事件机制及红黑树数据结构，降低了其内部轮询的开销，同时极大的减小了线程上下文切换的开销。
>


``` java
//sun.nio.ch.WindowsSelectorImpl.SubSelector
private final class SubSelector {
        private final int pollArrayIndex; // starting index in pollArray to poll
        // These arrays will hold result of native select().
        // The first element of each array is the number of selected sockets.
        // Other elements are file descriptors of selected sockets.
        // 保存发生read的FD
        private final int[] readFds = new int [MAX_SELECTABLE_FDS + 1];
        // 保存发生write的FD
        private final int[] writeFds = new int [MAX_SELECTABLE_FDS + 1];
        //保存发生except的FD
        private final int[] exceptFds = new int [MAX_SELECTABLE_FDS + 1];

        private SubSelector() {
            this.pollArrayIndex = 0; // main thread
        }

        private SubSelector(int threadIndex) { // helper threads
            this.pollArrayIndex = (threadIndex + 1) * MAX_SELECTABLE_FDS;
        }

        private int poll() throws IOException{ // poll for the main thread
            return poll0(pollWrapper.pollArrayAddress,
                         Math.min(totalChannels, MAX_SELECTABLE_FDS),
                         readFds, writeFds, exceptFds, timeout);
        }

        private int poll(int index) throws IOException {
            // poll for helper threads
            return  poll0(pollWrapper.pollArrayAddress +
                     (pollArrayIndex * PollArrayWrapper.SIZE_POLLFD),
                     Math.min(MAX_SELECTABLE_FDS,
                             totalChannels - (index + 1) * MAX_SELECTABLE_FDS),
                     readFds, writeFds, exceptFds, timeout);
        }

        private native int poll0(long pollAddress, int numfds,
             int[] readFds, int[] writeFds, int[] exceptFds, long timeout);
             ...
}
```
#### updateSelectedKeys解读

5. 接下来将通过`上面WindowsSelectorImpl#doSelect展示源码中<5>` 处的 `updateSelectedKeys(action)`来处理每个`channel`的 **准备就绪**的信息。
  - 如果该通道的`key`尚未在`selectedKeys`中存在，则将其添加到该集合中。
  - 如果该通道的`key`已经存在`selectedKeys`中，即这个`channel`存在所支持的`ReadyOps`就绪操作中必须包含一个这种操作(由`(ski.nioReadyOps() & ski.nioInterestOps()) != 0`来确定)，此时修改其`ReadyOps`为当前所要进行的操作。而我们之前看到的`Consumer<SelectionKey>`这个动作也是在此处进行。而由下面源码可知，先前记录在`ReadyOps`中的任何就绪信息在调用此`action`之前被丢弃掉，直接进行设定。 
``` java
//sun.nio.ch.WindowsSelectorImpl#updateSelectedKeys
private int updateSelectedKeys(Consumer<SelectionKey> action) {
    updateCount++;
    int numKeysUpdated = 0;
    numKeysUpdated += subSelector.processSelectedKeys(updateCount, action);
    for (SelectThread t: threads) {
        numKeysUpdated += t.subSelector.processSelectedKeys(updateCount, action);
    }
    return numKeysUpdated;
}
//sun.nio.ch.SelectorImpl#processReadyEvents
protected final int processReadyEvents(int rOps,
                                        SelectionKeyImpl ski,
                                        Consumer<SelectionKey> action) {
    if (action != null) {
        ski.translateAndSetReadyOps(rOps);
        if ((ski.nioReadyOps() & ski.nioInterestOps()) != 0) {
            action.accept(ski);
            ensureOpen();
            return 1;
        }
    } else {
        assert Thread.holdsLock(publicSelectedKeys);
        if (selectedKeys.contains(ski)) {
            if (ski.translateAndUpdateReadyOps(rOps)) {
                return 1;
            }
        } else {
            ski.translateAndSetReadyOps(rOps);
            if ((ski.nioReadyOps() & ski.nioInterestOps()) != 0) {
                selectedKeys.add(ski);
                return 1;
            }
        }
    }
    return 0;
}
//sun.nio.ch.WindowsSelectorImpl.SubSelector#processSelectedKeys
private int processSelectedKeys(long updateCount, Consumer<SelectionKey> action) {
    int numKeysUpdated = 0;
    numKeysUpdated += processFDSet(updateCount, action, readFds,
                                    Net.POLLIN,
                                    false);
    numKeysUpdated += processFDSet(updateCount, action, writeFds,
                                    Net.POLLCONN |
                                    Net.POLLOUT,
                                    false);
    numKeysUpdated += processFDSet(updateCount, action, exceptFds,
                                    Net.POLLIN |
                                    Net.POLLCONN |
                                    Net.POLLOUT,
                                    true);
    return numKeysUpdated;
}

    /**
    * sun.nio.ch.WindowsSelectorImpl.SubSelector#processFDSet
    * updateCount is used to tell if a key has been counted as updated
    * in this select operation.
    *
    * me.updateCount <= updateCount
    */
private int processFDSet(long updateCount,
                            Consumer<SelectionKey> action,
                            int[] fds, int rOps,
                            boolean isExceptFds)
{
    int numKeysUpdated = 0;
    for (int i = 1; i <= fds[0]; i++) {
        int desc = fds[i];
        if (desc == wakeupSourceFd) {
            synchronized (interruptLock) {
                interruptTriggered = true;
            }
            continue;
        }
        MapEntry me = fdMap.get(desc);
        // If me is null, the key was deregistered in the previous
        // processDeregisterQueue.
        if (me == null)
            continue;
        SelectionKeyImpl sk = me.ski;

        // The descriptor may be in the exceptfds set because there is
        // OOB data queued to the socket. If there is OOB data then it
        // is discarded and the key is not added to the selected set.
        if (isExceptFds &&
            (sk.channel() instanceof SocketChannelImpl) &&
            discardUrgentData(desc))
        {
            continue;
        }
        //我们应该关注的
        int updated = processReadyEvents(rOps, sk, action);
        if (updated > 0 && me.updateCount != updateCount) {
            me.updateCount = updateCount;
            numKeysUpdated++;
        }
    }
    return numKeysUpdated;
}

```
至此，关于Selector的内容就暂时告一段落，在下一篇中，我会针对Java NIO Buffer进行相关解读。