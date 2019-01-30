title: 当 HTTP 连接池遇上 KeepAlive 时
date: 2018-12-23
tags:
categories: 精进
permalink: Fight/When-the-HTTP-connection-pool-encounters-KeepAlive
author: hetaohappy
from_url: https://blog.csdn.net/hetaohappy/article/details/51851880
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247485891&idx=2&sn=82dd4786e38e23a10fa9210745c99130&chksm=fa497672cd3eff64fb06873ba909140f6f839f9565e2815eff5476102e91b60f46b67ea5acab&token=396846451&lang=zh_CN#rd

-------

摘要: 原创出处 https://blog.csdn.net/hetaohappy/article/details/51851880 「hetaohappy」欢迎转载，保留摘要，谢谢！

- [1. 连接种类](http://www.iocoder.cn/Fight/When-the-HTTP-connection-pool-encounters-KeepAlive/)
- [2 keepalive机制](http://www.iocoder.cn/Fight/When-the-HTTP-connection-pool-encounters-KeepAlive/)
  - [2.1 keepalived](http://www.iocoder.cn/Fight/When-the-HTTP-connection-pool-encounters-KeepAlive/)
  - [2.2 tcp的keepalive](http://www.iocoder.cn/Fight/When-the-HTTP-connection-pool-encounters-KeepAlive/)
  - [2.3 http的 keepalive](http://www.iocoder.cn/Fight/When-the-HTTP-connection-pool-encounters-KeepAlive/)
- [3. tomcat8对keepalive的实现](http://www.iocoder.cn/Fight/When-the-HTTP-connection-pool-encounters-KeepAlive/)
  - [3.1  http 1.0实现](http://www.iocoder.cn/Fight/When-the-HTTP-connection-pool-encounters-KeepAlive/)
  - [3.2 http 1.1实现](http://www.iocoder.cn/Fight/When-the-HTTP-connection-pool-encounters-KeepAlive/)
- [4. 连接池对keepalive的处理](http://www.iocoder.cn/Fight/When-the-HTTP-connection-pool-encounters-KeepAlive/)
- [5. Connection reset by peer异常](http://www.iocoder.cn/Fight/When-the-HTTP-connection-pool-encounters-KeepAlive/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

最近在使用netty作为http客户端通过pool连接tomcat的时候，出现了很多Connection reset by peer 的IOException的异常。便对问题的根源做了细致的调研。

# 1. 连接种类

一般连接主要分为长连接，短连接和http的keepalive连接。

**1.1 长连接**：建立完连接后，该连接不再进行释放。

* 优点
    * 性能较高，不需要重复建立tcp连接或者关闭tcp连接
    * 基本上不会出现CLOSE_WAIT和TIME_WAIT的问题
* 缺点：一般需要一个连接池来维护长连接（一般有数据库连接池，http的连接池等) 复杂度较高

**1.2 短连接**：每次请求均需要tcp三次握手建立连接，业务执行，tcp四次挥手关闭连接。

* 优点：实现简单。
* 缺点：
    * 性能较差。大部分都是tcp层面上的交互(新建和关闭tcp连接)
    * 系统会出现大量的tcp的状态是：TIME_WAIT  如果没有设置SO_RESUSEADDR ，很容易出现端口被占满的情况。(在关闭完连接时，tcp状态是TIME_WAIT，只有等2个MSL后，才会进行close掉)

**1.3 http的keepalive**：用于http协议。在http 1.1中，为了解决长连接提出的。

* 优点：用于维护长连接，提升性能
* 缺点：需要在header中进行控制，需要交互控制，相对复杂。

# 2 keepalive机制

提到keepalive， 容易对下面三种机制混淆：

* keepalived
* tcp 的 keepalive
* http 的 keepalive

## 2.1 keepalived

 用途：高可用，一般是和lvs一起使用。
 
 具体可参考：<http://outofmemory.cn/wiki/keepalived-configuration>

## 2.2 tcp的keepalive

用途：socket连接的保活。在新建socket的时候，可以设置SO_KEEPALIVE 进行打开。

keepalive主要有三个参数：

* **tcp_keepalive_time**： 一个连接需要TCP开始发送keepalive探测数据包之前的空闲时间。以秒为单位
* **tcp_keepalive_probes**： 发送TCP keepalive探测数据包的最大数量，默认是9.如果发送9个keepalive探测包后对端仍然没有响应，便发送RST关闭掉连接。
* **tcp_keepalive_intvl**： 发送两个TCP keepalive探测数据包的间隔时间，默认是75秒

## 2.3 http的 keepalive

用途：http的长连接，在http 1.0中使用的为短连接：每一次请求均需要新建tcp连接，http协议数据的发送接收，关闭tcp连接。 该种机制性能很 低，在http 1.1协议中引入了keepalive机制来保持tcp连接。

在http1.0中，全部是短连接，如果想建立长连接，需要在header里面加上keepalive，这样web服务器看到这个字段，不会立马关闭连接。而是将tcp连接维持一段时间。如果需要关闭，则在header中写 keepalive close，来告诉 客户端需要关闭该连接。

在http1.1中，默认会实现keepalive，如果使用的是http1.1协议，header是不需要加上keepalive的。

# 3. tomcat8对keepalive的实现

## 3.1 http 1.0实现

tomcat8中，如果发送的是http1.0的协议。 tomcat8返回的均是1.1的协议。并且不管请求的header有没有Connection:keepalive ，均会在返回的header中加上connection：close 。下面是访问tomcat8的截图：

* GET请求是http 1.0，但是返回的是1.1的协议：

    ![img](http://static.iocoder.cn/1da718624c950e1edf73ea08fdf1c272)

* 返回的header里面有Connection:close

    ![img](http://static.iocoder.cn/12c5b1fc1ed9ac1b175f5a980c7a5f2e)

## 3.2 http 1.1实现

tomcat8主要有两个参数来控制keepalive的机制。keepAliveTimeout 和maxKeepAliveRequests

* **keepAliveTimeout**： 默认和soTimeout 值保持一致，该值为20000ms，也就是在这么长时间内没有通信，tomcat会关闭掉该连接。设置为-1 则代表不会关闭该连接。
* **maxKeepAliveRequests** ：默认为100，也就是在keepAliveTimeout时间内，如果使用次数超过100，则会关闭掉该连接。设置为-1，则代表不会关闭连接。在关闭后，会在返回的header上面加上Connection:close 。

如果需要tomcat保持长连接：可配置 maxKeepAliveRequests = "-1" keepAliveTimeout=-"-1" ，则tomcat8不会关闭掉该连接。

# 4. 连接池对keepalive的处理

主要需要处理两个地方：

* 1：**maxKeepAliveRequests**  连接达到默认的设置次数。则会在header上面加Connection:close。
    * 在接收web服务器返回的数据时，需要检查一下header里面是否有Connection:close，如果close，则需要将该连接从连接池里物理关闭掉。否则容易出现connection reset by peer的异常。

* 2：**keepAliveTimeout** 超过该时间没有流量，则会关闭掉连接。
    * tomcat在连接空闲超过该时间后，会主动关闭掉连接。会向客户端发送FIN命令。

​如果是IO(同步socket)：则在获取连接的时候需要检查一下该socket的连接状态。 因为tcp在底层已经关闭了该连接。 如果不检查的话，则会SocketCloseException的错误。

​如果是NIO(异步channel) ：则在selector的时候，read数据的时候，会返回-1，然后将该连接从连接池给物理关闭掉。

# 5. Connection reset by peer异常

**异常场景**：

* 1： 当我们往一个对端已经close的通道写数据的时候，对方的tcp会收到这个报文，并且反馈一个reset报文，当收到reset报文的时候，继续做select读数据的时候就会抛出Connect reset by peer的异常。该异常为jdk抛出的异常。在native代码里面抛出。
* 2：尝试和未开放的服务器端口建立tcp连接时，服务器tcp将会直接向客户端发送reset报文
* 3：ack报文丢失，并且超出一定的重传次数或时间后，会主动向对端发送reset报文释放该TCP连接

**连接池出现该异常分析**：

* 1：由于客户端在收到Connection:close的header时候并没有物理关闭该连接，而是将该连接返回到了连接池中。
* 2：下一个请求拿到该连接发送数据，由于tomcat的该socket通道已经关闭，tomcat接收到该连接时，便会回复一个RST。
* 3：客户端在读取数据(RST的时候，内部会调用(JDK)SocketChannel.read的时候抛出 java.io.IOException(Connection reset by peer)
