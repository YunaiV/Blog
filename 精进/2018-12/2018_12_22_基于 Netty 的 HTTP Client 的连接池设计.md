title: 基于 Netty 的 HTTP Client 的连接池设计
date: 2018-12-22
tags:
categories: 精进
permalink: Fight/Connection-pool-design-of-HTTP-Client-based-on-Netty
author: hetaohappy
from_url: https://blog.csdn.net/hetaohappy/article/details/51867059
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247485891&idx=1&sn=868fd2b55d7e4eee64d4ffed5de1ab99&chksm=fa497672cd3eff649a1bb7d48136a8063b93759b7eeb535ff6a97b95140d771b4be5df094a42&token=396846451&lang=zh_CN#rd

-------

摘要: 原创出处 https://blog.csdn.net/hetaohappy/article/details/51867059 「hetaohappy」欢迎转载，保留摘要，谢谢！

- [1. 复用类型的选型](http://www.iocoder.cn/Fight/Connection-pool-design-of-HTTP-Client-based-on-Netty/)
  - [1.1 channel 复用](http://www.iocoder.cn/Fight/Connection-pool-design-of-HTTP-Client-based-on-Netty/)
  - [1.2 channel 独享](http://www.iocoder.cn/Fight/Connection-pool-design-of-HTTP-Client-based-on-Netty/)
  - [1.3 选择](http://www.iocoder.cn/Fight/Connection-pool-design-of-HTTP-Client-based-on-Netty/)
- [2. 组件选型](http://www.iocoder.cn/Fight/Connection-pool-design-of-HTTP-Client-based-on-Netty/)
- [3. pool的设计](http://www.iocoder.cn/Fight/Connection-pool-design-of-HTTP-Client-based-on-Netty/)
  - [3.1 模型](http://www.iocoder.cn/Fight/Connection-pool-design-of-HTTP-Client-based-on-Netty/)
  - [3.2 主要功能点](http://www.iocoder.cn/Fight/Connection-pool-design-of-HTTP-Client-based-on-Netty/)
  - [3.3 获取连接](http://www.iocoder.cn/Fight/Connection-pool-design-of-HTTP-Client-based-on-Netty/)
  - [3.4 资源池](http://www.iocoder.cn/Fight/Connection-pool-design-of-HTTP-Client-based-on-Netty/)
  - [3.5 归还连接](http://www.iocoder.cn/Fight/Connection-pool-design-of-HTTP-Client-based-on-Netty/)
  - [3.6 超时控制](http://www.iocoder.cn/Fight/Connection-pool-design-of-HTTP-Client-based-on-Netty/)
  - [3.7 性能优化](http://www.iocoder.cn/Fight/Connection-pool-design-of-HTTP-Client-based-on-Netty/)
  - [3.8 配置参数](http://www.iocoder.cn/Fight/Connection-pool-design-of-HTTP-Client-based-on-Netty/)
- [4. 面临的问题](http://www.iocoder.cn/Fight/Connection-pool-design-of-HTTP-Client-based-on-Netty/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

使用netty作为http的客户端，pool又该如何进行设计。本文将会进行详细的描述。

# 1. 复用类型的选型

## 1.1 channel 复用

多个请求可以共用一个channel

**模型如下**：

![模型](http://static.iocoder.cn/7dbced9a61ca20c18fcfb432904b94f7)

 **特点**：

* 1：callback队列为回调队列。 不同的callback通过一个全局的id进行标识。发送的时候会把该id发到服务端，服务端在回复的时候必须把该id再返回到客户端。
* 2：获取连接只需要随机获取一个channel即可，将callback添加到队列里面。
* 3： 获取连接时消除了锁的竞争，性能高效。
* 4：结构简单。

**示例**：

* osp(唯品会的SOA框架) client pool实现(thrift协议) 
* spray 的 akka client pool

**约束**：

需要服务端配合支持channel复用。需要有一个全局唯一的id用于识别请求。 通常id先发给服务端，服务端还要把id会给客户端。

## 1.2 channel 独享

 每个请求独立使用一个channel。

 **模型如下**：

![模型](http://static.iocoder.cn/fdc50755b6e3dbc885b25a40e69a29b7)

 **特点**：

* 1：同一个channel同时只给一个request使用。
* 2：连接池的设计较为复杂。

**示例**：

* 1：数据库连接池[druid，c3p0，dbcp，hikaricp，caelus(唯品会内部连接池)]
* 2：netty的http pool ; apache的httpclient pool, httpasyncclient pool ; nginx的pool。

## 1.3 选择

​由于http1.1协议原生不支持channel复用(http2是支持的)，如果需要支持，则需要在header里面加入一个唯一id，所有的应用服务器均需要进行改动。为了和nginx的连接池保持一致，**确定使用channel的独享方式**。

# 2. 组件选型



| 组件         | 优点                | 缺点               |
| ------------ | ------------------- | ------------------ |
| common-pool  | 功能完整            | 不支持异步连接     |
| rxnetty pool | 功能完整，支持netty | 使用的为rxjava机制 |
| netty pool   | netty原生实现       | 功能较为简单       |

选择netty pool作为连接池的实现。4.0.33版本有该功能，可能老版本没有pool的功能。

# 3. pool的设计

## 3.1 模型

​![模型](http://static.iocoder.cn/522d27e4f73eb00559b6614d6bd7aa1d)

通过ip,port路由到对应的pool，每个pool之间完全独立。

## 3.2 主要功能点

![功能点](http://static.iocoder.cn/1b6407b6876b85c22f5389e50fa62f18)

## 3.3 获取连接

* ​1：通过控制最大连接数，来避免无限的创建连接。
* 2：当超过最大连接数时，则需要等待。由于整个流程是全异步的，需要将当前信息进行任务封装注册回调。
* 3：需要设置等待连接的个数及超时时间，避免把内存给撑爆。
* 4：需要对获取的连接进行有效性检查。一般只需校验channel.isactive()即可。如果检验失败，需要重新获取有效连接。

## 3.4 资源池

* 1：使用无锁的ConcurrentLinkedDeque 双向队列来存放所有idle的连接(jdk1.7才有该类)。
* 2：该队列通过cas的操作来避免同步。 由于拿到连接后业务执行的速度较慢，所以这里的cas冲突应该很小。

## 3.5 归还连接

![归还连接](http://static.iocoder.cn/6fd3334e2176c67e47c9013d284d86ed)

归还连接主要包含两部分：正常release和异常的forceClose

* 1：在netty中，如果收到FIN(服务端发送的正常close请求)，则会通知到netty的channelInactive接口，需要在该接口处进行forceClose.
* 2：收到RST(服务端非正常的关闭)，则会通知到exceptionCaught接口，需要在该接口处进行foreclose。关于RST的问题可参考：<http://blog.csdn.net/hetaohappy/article/details/51851880>
* 3：在收到正常数据后(channel的channelRead接口)，需要判断header里面是否有Connection:close，如果有，则进行forceClose，否则进行release
* 4：如果空闲超时，则关闭连接，来避免连接一直被无效的占用。只需要增加IdleStateHandler ,判断连接空闲超时，则fire一个event事件。只需要注册对该事件的监听，进行foreclose即可。
* 5：占有超时：连接在规定的时间内未还，则进行forceClose。

6：发送请求时，发现channel已经被close掉或者其他io异常，则进行forceClose。

7：forceclose接口里面，需要通过一个状态位来控制是否操作 acquiredChannelCount(已获取连接数)。由于调用forceclose，连接可能在资源池中，如果操作该字段，会导致该字段统计不准确。

## 3.6 超时控制

**获取连接timeout**

在规定的时间内没有获取到连接，则抛异常。

* 1：一般实现是通过ReentrantLock来设置lock的超时时间或者直接通过unsafe.park设置超时时间。该种机制会对当前线程进行block。
* 2：由于netty是纯异步机制，如果进行block，会严重影响性能。所以这里是将当前信息进行task封装，然后schedule一个定时任务。如果在设定时间内该task没有被消费，则会抛出timeout的异常。

**建立连接timeout**

* 1：在BIO中，通过设置socket的connect(SocketAddress endpoint, int timeout) 时间即可。Tips：该值不要和setSoTimeout(int timeout)混淆，sotimeout是设置调用read的超时时间。
* 2：在NIO中，需要业务自己控制连接的超时时间。 一般是通过schedule一个定时任务来控制超时时间。(在netty中即使用的该机制)

**连接空闲timeout**

* 1： 通过设置一个handler(IdleStateHandler ),在新建连接的时候schedule一个任务(时间为空闲超时时间)，在调用read或者write方法的时候，进行时间的更新。如果任务运行的时候发现空闲超时，则进行event的触发。
* 2：业务handler捕获该event，进行连接空闲超时的处理。

**连接被占有timeout** 

避免连接泄露

* 1：在获取连接的时候 schedule一个任务(时间为连接被占用的timeout)，在归还的时候会cancel该任务。如果该任务被运行，说明在规定的时间没有归还，则进行timeout的处理。

## 3.7 性能优化

* 1：资源池无锁化：ConcurrentLinkedDeque  (前面已有介绍)

* 2：acquiredChannelCount(已获取连接数)的无锁化(该字段用来控制是否达到了最大连接数，正常情况为获取连接后加1，归还连接后减1)。

连接池均会通过acquiredChannelCount来控制当前已经获取的连接个数。该参数会面临着多线程的竞争，需要进行同步或者cas的设计。如何设计让acquiredChannelCount完全不用考虑多线程竞争？

看能不能从akka的设计中找点思路： akka消除竞争的方式就是让一个actor同一时刻只能在一个线程中运行，这样actor里面所有的全局参数就不需要考虑多线程竞争，一个actor里面所有的任务都是串行执行的，完全消除竞争。

那么能不能串行操作acquiredChannelCount呢? 答案是可以的，并且在netty中实现非常简单，只需要实现如下代码即可：

```Java
if (executor.inEventLoop()) {
    acquiredChannelCount++;
 } else{
    executor.execute(newOneTimeTask() {
       
       @Override
       public void run() {
             acquiredChannelCount++;
       }
   }
}
```

 其中executor 就是一个固定的线程。判定当前执行的线程是否是executor这个线程，如果是则直接执行。如果不是，则放到executor线程的队列里达到串行操作的目的(类似于actor的mailbox) (netty的设计及抽象能力确实非常高)

## 3.8 配置参数

- **http_pool_aquire_timeout** ：获取连接超时时间：默认为5000ms
- **http_pool_maxConnections**：连接大小：默认为1000
- **http_connection_timeout** ：建立连接的超时时间：默认为5000ms
- **http_pool_idle_timeout**：连接空闲多久关闭：默认为：30分钟
- **http_pool_maxPending**：连接池不够用，最大允许有多少个pendingRequest：默认为无限大
- **http_pool_maxHolding**：拿连接的使用时间。在规定时间未还，则强制close掉。默认为5000ms。

# 4. 面临的问题

* 1：所有的操作都是纯异步，导致callback嵌套的特别深(netty通过promise机制，来方便callback的使用)，如果控制不好，很容易出问题。
* 2：连接被require后，一定要保证归还，由于异步特性，很容易在某些异常下将连接漏还(笔者遇到在高并发下由于代码bug导致漏还的情况)
* 3：如何避免在拿到连接后，同时web服务器(http的keepalive机制)将该连接给close掉了，导致执行的失败。有两种解决方案可以参考。
    * 3.1：可以参考common-pool的设计思想，在后端开启一个线程定时对所有连接进行心跳检测。问题：
        * 如何确定该线程定时的时间。后端web服务器对连接的超时时间可能不一致，该定时时间一定要小于web服务器的连接超时时间。
        * 心跳执行的接口问题。需要所有的web服务器均需要实现固定的接口进行心跳检测，可行性比较差。
    * 3.2：重试机制：
        * 捕获执行失败的异常，如果是特定的异常，则forceClose当前的连接，重新拿一个连接进行访问。如果超过重试次数，则抛出异常。
