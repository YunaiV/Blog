title: Jetty 源码剖析系列(6) - Connector 与 ServerSocket
date: 2018-01-07
tag: 
categories: Jetty
permalink: Jetty/EricCen/connector-yu-server-socket
author: Eric Cen
from_url: http://ericcenblog.com/2018/02/22/jettyyuan-ma-pou-xi-xi-lie-6-connectoryu-serversocket
wechat_url: 

-------

摘要: 原创出处 http://ericcenblog.com/2018/01/12/jettyyuan-ma-pou-xi-xi-lie-5-serveryu-handler 「Eric Cen」欢迎转载，保留摘要，谢谢！


-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

本文主要从源代码的角度来分析Jetty的Connector如何通过ServerSocket来绑定和监听网络地址和端口的过程。

Jetty的`Connector`实现类是`ServerConnector`，循例从它的`doStart()`方法开始(至于是如何到达这个方法，请移步本系列的前几篇)：
![img](http://ericcenblog.com/content/images/2018/02/Jetty1.png)
它的`doStart`方法先调用父类`AbstractNetworkConnector`的`doStart`方法:![img](http://ericcenblog.com/content/images/2018/02/Jetty2.png)
在父类的`doStart`方法里面会调用子类实现的`open`方法，此处是`ServerConnector`的`open`方法:![img](http://ericcenblog.com/content/images/2018/02/Jetty3.png)
到这里已经可以看到`Jetty`的`Connector`与`Java NIO`的关系了，将会调用`openAcceptChannel`方法来构建一个NIO的`ServerSocketChannel`：![img](http://ericcenblog.com/content/images/2018/02/Jetty4.png)
从上图的代码我们可以看到，`Jetty`默认用`Java NIO`来实现了它的网络框架，另外一点就是，如果你用`java`来实现一套网络框架，无非就是`BIO`和`NIO`两种，返朴归真。

先常规调用`ServerSocketChannel`的静态方法`open`方法来打开一个`ServerSocketChannel`：![img](http://ericcenblog.com/content/images/2018/02/Jetty5.png)因为Java的NIO是基于Selector来实现的，所以在这里会通过`SelectorProvider`类的静态方法`provider`来加载Selector的实现类：![img](http://ericcenblog.com/content/images/2018/02/Jetty6.png)它先尝试从`System Property`里加载`Selector`的实现类并实例化:![img](http://ericcenblog.com/content/images/2018/02/Jetty7.png)如果没有的话，就尝试通过`SPI`的方式来加载`Selector`的实现类并实例化：![img](http://ericcenblog.com/content/images/2018/02/Jetty8.png)如果还是没有的话，就用`Default`的`Selector`，这个是根据底层操作系统来决定，比如，在`Windows`系统里将会是`WindowsSelectorProvider`这个类：![img](http://ericcenblog.com/content/images/2018/02/Jetty9.png)![img](http://ericcenblog.com/content/images/2018/02/Jetty10.png)`WindowsSelectorProvider`继承于`SelectorProviderImpl`这个抽象类, 它的`openServerSocketChannel`方法将会返回一个新建出来的`ServerSocketChannelImpl`对象：![img](http://ericcenblog.com/content/images/2018/02/Jetty11.png)

到此我们再回过头看ServerConnector的openAcceptChannel方法，![img](http://ericcenblog.com/content/images/2018/02/Jetty12.png)上图通过`ServerSocketChannel.open()`方法拿到的就是一个`ServerSocketChannelImpl`实例对象, 我们接着往下看，

![img](http://ericcenblog.com/content/images/2018/02/Jetty13.png)如果你没在启动参数里加--host参数，那它就会去指定所谓的全零地址：![img](http://ericcenblog.com/content/images/2018/02/Jetty14.png)

那这个所谓的anyLocalAddress又是怎么来的呢？![img](http://ericcenblog.com/content/images/2018/02/Jetty15.png)这里impl是InetAddressImpl，它是在加载InetAddress类的时候，通过它的类静态块初始化的：![img](http://ericcenblog.com/content/images/2018/02/Jetty16.png)而InetAddressImplFactory会通过一个本地方法isIPv6Supported来判断底层操作系统是否支持IPv6，来加载InetAddressImpl的实现类：![img](http://ericcenblog.com/content/images/2018/02/Jetty17.png)![img](http://ericcenblog.com/content/images/2018/02/Jetty18.png)在我的windows系统里，是支持IPv6的，所以返回的InetAddressImpl应该是Inet6AddressImpl对象，再来看它的anyLocalAddress方法：![img](http://ericcenblog.com/content/images/2018/02/Jetty19.png)这里会通过preferIPv6Address这个静态便来来判断应用有没有指定prefer使用IPv6地址，这个静态变量同样在InetAddress类的静态代码块里面初始化了：![img](http://ericcenblog.com/content/images/2018/02/Jetty20.png)我们可以看到，如果应用没有指定了java.net.preferIPv6Addresses这个-D参数的话，就算InetAddressImpl实例对象是Inet6AddressImpl，所谓的anyLocalAddress还是会用Inet4AddressImpl的anyLocalAddress:![img](http://ericcenblog.com/content/images/2018/02/Jetty21.png)而Inet4AddressImpl的anyLocalAddress是大名鼎鼎的0.0.0.0：![img](http://ericcenblog.com/content/images/2018/02/Jetty22.png)

好了，扯远了，我们再回到ServerConnector的openAcceptChannel方法：![img](http://ericcenblog.com/content/images/2018/02/Jetty23.png)在这里它就会拿出ServerSocketChannelImpl里的ServerSocket来进行网络地址和端口的绑定和监听！ 我们先来看ServerSocketChannelImpl的socket()方法：![img](http://ericcenblog.com/content/images/2018/02/Jetty24.png)它会创建一个ServerSocket的适配对象：![img](http://ericcenblog.com/content/images/2018/02/Jetty25.png)ServerSocketAdaptor继承于ServerSocket。
我们接着来看ServerSocket的bind方法，它接收两个参数，一个是前面分析的InetSocketAddress，此处为IPv4的全零，0.0.0.0地址，一个是接收队列的size，此处为0(这个值在bind方法里面将会会重置)，下面进入bind方法：![img](http://ericcenblog.com/content/images/2018/02/Jetty26.png)我们会看到接收队列的size会被重置为50，也就是说TCP接收队列里面最多只能有50个请求。我们接着要关注的是下面两行代码：![img](http://ericcenblog.com/content/images/2018/02/Jetty27.png)上面两行代码就是实现了网络地址的绑定和端口的监听。 我们先来看ServerSocket的getImpl方法：![img](http://ericcenblog.com/content/images/2018/02/Jetty28.png)首次调用的话，它会去调用createImpl方法：![img](http://ericcenblog.com/content/images/2018/02/Jetty29.png)这里impl是一个SocketImpl，它是通过调用setImpl方法来创建：![img](http://ericcenblog.com/content/images/2018/02/Jetty30.png)这里的factory是一个SocketImplFactory的静态变量，默认为null，而一路我们也没有设置，所以它还是为null，所以impl将会被新建为一个SocksSocketImpl的实例对象。 SocksSocketImpl继承于PlainSocketImpl，我们再看PlainSocketImpl的默认构造函数：
![img](http://ericcenblog.com/content/images/2018/02/Jetty31.png)我们看到它会根据userDualStackImpl这个静态变量来实例化不同的AbstractPlainSocketImpl对象，而useDUalStackImpl这个静态变量也是在PlainSocketImpl的静态代码块里面初始化：![img](http://ericcenblog.com/content/images/2018/02/Jetty32.png)我们可以看到它是根据两个System Property值来确定的，一个是os.version,系统版本，我当前用的是JDK1.8的windows版， 另一个是java.net.preferIPv4Stack(这个跟我们上文看到的java.net.preferIPv6Addresses参数很相像，一个IPv4,一个IPv6)，就是说如果系统版本(此处为windows版本)大于6.0并且应用没有指定参数java.net.preferIPv4Stack的话，useDualStackImpl为true，也就是说将会使用IPv4和IPv6双栈！ 双栈的话 ，impl就会是DualStackPlainSocketImpl， 单栈的话impl就是TwoStacksPlainSocketImpl（其实这里我有点不明白，Dual和Two不都是有指两个的意思吗？望有识之士不吝赐教，私信告知。eric_cen@outlook.com）， 再回到之前的bind方法，实际上调用的是AbstractPlainSocketImpl的bind方法，![img](http://ericcenblog.com/content/images/2018/02/Jetty33.png)而最终双栈和单栈的区别又在于DualStackPlainSocketImpl和TwoStacksPlainSocketImpl的socketBind方法。 DualStackPlainSocketImpl.socketBind方法:
![img](http://ericcenblog.com/content/images/2018/02/Jetty34.png)最终调用的是本地方法bind0. TwoStacksPlainSocketImpl.socketBind方法：
![img](http://ericcenblog.com/content/images/2018/02/Jetty35.png)最终调用的是本地方法socketBind。

而监听端口的listen方法最终也是分开DualStackPlainSocketImpl和TwoStacksPlainSocketImpl各自实现。

至此，Jetty的Connector如何绑定网络地址和监听端口分析完毕。其实本文大部分都是在分析Java自带的ServerSocket工作细节。