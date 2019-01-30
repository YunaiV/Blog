title: Jetty 源码剖析系列(3) - Connector 如何接收处理网络请求
date: 2018-01-04
tag: 
categories: Jetty
permalink: Jetty/EricCen/connectorru-he-jie-shou-wang-luo-qing-qiu
author: Eric Cen
from_url: http://ericcenblog.com/2017/11/02/jettyyuan-ma-pou-xi-xi-lie-3-connectorru-he-jie-shou-wang-luo-qing-qiu/
wechat_url: 

-------

摘要: 原创出处 http://ericcenblog.com/2017/11/02/jettyyuan-ma-pou-xi-xi-lie-3-connectorru-he-jie-shou-wang-luo-qing-qiu/ 「Eric Cen」欢迎转载，保留摘要，谢谢！


-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

在本系列的[第一篇](http://www.iocoder.cn/Jetty/EricCen/intro)提到，Jetty由Connector组件负责接收网络请求，如下图：![img](http://ericcenblog.com/content/images/2017/11/----_20171102200408.png)

`ServerConnector`是Jetty `Connector`的实现类，我们直接看它的`doStart`方法：![img](http://ericcenblog.com/content/images/2017/11/----_20171102200408-1.png)它先是调用了父类`AbstractNetworkConnector`的`doStart`方法：![img](http://ericcenblog.com/content/images/2017/11/----_20171102200408-2.png)这个父类的`doStart`方法先是调用了`open`方法，注意这里实际调用的是`ServerConnector`实现的`open`方法：![img](http://ericcenblog.com/content/images/2017/11/----_20171102200408-3.png)它先是调用`openAcceptChannel`方法来创建一个NIO的`ServerSocketChannel`：![img](http://ericcenblog.com/content/images/2017/11/----_20171102200408-4.png)上面的就是一个经典的NIO `ServerSocketChannel`创建过程：先是`ServerSocketChannel.open()`,然后再把`ServerSocketChannel`的`ServerSocket`绑定到相应的`InetSocketAddress`，熟悉的配方，熟悉的味道：）
再回到`ServerConnector`的`open`方法，它拿到这个`ServerSocketChannel`后，会调用它的`configureBlocking`方法，把它设置为阻塞的，这里其实是为了设置它的ServerSocket在调用accept方法的时是阻塞模式(即调用accept方法就会进入线程阻塞直到有网络连接进来)。
我们再回到`AbstractNetworkConnector`的`doStart`方法,当它执行完`open`方法打开`ServerSocketChannel`后,接着会调用它的父类`AbstractConnector`的`doStart`方法:![img](http://ericcenblog.com/content/images/2017/11/----_20171102200408-5.png)这个方法很重要，到这里就开始了本文真正要关注的地方：`Connector`是如何接收处理网络请求的。我们来看其中的这段代码：![img](http://ericcenblog.com/content/images/2017/11/----_20171102200408-6.png)我们先看一下`_acceptors`（`AbstractConnector`的构造方法里初始化的）是什么：![img](http://ericcenblog.com/content/images/2017/11/----_20171102200408-7.png)我们可以看到它其实是个acceptor（我翻译为网络请求接收者)线程数组，这个数组的长度定得有点讲究，如果你没有指定acceptor的数目，acceptors这个值传进来的时候会默认为-1，那么就会将主机的CPU数量除以8取整跟4比较取最小的再跟1比较取最大的，如果acceptors的数目比JVM CPU的数目还多的话，它就会通过打一句log来提醒你Acceptor的数目不应大于JVM CPU的数目。

TO BE CONTINUE.....