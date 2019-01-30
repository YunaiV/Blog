title: 八种 WebSocket 框架的性能比较
date: 2018-11-22
tags:
categories: 精进
permalink: Fight/Performance-comparison-of-eight-WebSocket-frameworks
author: colobu
from_url: https://colobu.com/2015/07/14/performance-comparison-of-7-websocket-frameworks/
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247485766&idx=3&sn=89283e6b32d61cfecb65686c28556eac&chksm=fa4976f7cd3effe15e4345733277f6cf62552894b49c08abf42ca5d0be0c0241dc9b84176424&token=582518212&lang=zh_CN#rd

-------

摘要: 原创出处 https://colobu.com/2015/07/14/performance-comparison-of-7-websocket-frameworks/ 「colobu」欢迎转载，保留摘要，谢谢！

- [1. 测试环境](http://www.iocoder.cn/Fight/Performance-comparison-of-seven-WebSocket-frameworks/)
- [2. 测试结果](http://www.iocoder.cn/Fight/Performance-comparison-of-seven-WebSocket-frameworks/)
  - [2.1 Netty](http://www.iocoder.cn/Fight/Performance-comparison-of-seven-WebSocket-frameworks/)
  - [2.2 Vert.x](http://www.iocoder.cn/Fight/Performance-comparison-of-seven-WebSocket-frameworks/)
  - [2.3 Undertow](http://www.iocoder.cn/Fight/Performance-comparison-of-seven-WebSocket-frameworks/)
  - [2.4 Jetty](http://www.iocoder.cn/Fight/Performance-comparison-of-seven-WebSocket-frameworks/)
  - [2.5 Grizzly](http://www.iocoder.cn/Fight/Performance-comparison-of-seven-WebSocket-frameworks/)
  - [2.6 Spray](http://www.iocoder.cn/Fight/Performance-comparison-of-seven-WebSocket-frameworks/)
  - [2.7 Node.js](http://www.iocoder.cn/Fight/Performance-comparison-of-seven-WebSocket-frameworks/)
  - [2.8 Go](http://www.iocoder.cn/Fight/Performance-comparison-of-seven-WebSocket-frameworks/)
- [3. 测试结果分析](http://www.iocoder.cn/Fight/Performance-comparison-of-seven-WebSocket-frameworks/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

前一篇文章[使用四种框架分别实现百万websocket常连接的服务器](https://colobu.com/2015/05/22/implement-C1000K-servers-by-spray-netty-undertow-and-node-js)介绍了四种websocket框架的测试方法和基本数据。 最近我又使用几个框架实现了websocket push服务器的原型，并专门对这七种实现做了测试。 本文记录了测试结果和一些对结果的分析。
这七种框架是：

- [Netty](http://netty.io/)
- [Undertow](http://undertow.io/)
- [Jetty](http://www.eclipse.org/jetty/)
- [Vert.x](http://http//vertx.io)
- [Grizzly](https://grizzly.java.net/)
- [spray-websocket](https://github.com/wandoulabs/spray-websocket)
- [nodejs-websocket/Node.js](https://github.com/sitegui/nodejs-websocket)

最近用Golang实现了第八种，Go表现还不错。

- [Go](https://golang.org/)

# 1. 测试环境

使用三台C3.4xlarge AWS服务器做测试。 一台作为服务器，两台作为客户端机器， 每台客户端机器启动10个client,一共20个client
C3.4xlarge的配置如下：

| 型号       | vCPU | 内存 (GiB) | SSD 存储 (GB) |
| ---------- | ---- | ---------- | ------------- |
| c3.large   | 2    | 3.75       | 2 x 16        |
| c3.xlarge  | 4    | 7.5        | 2 x 40        |
| c3.2xlarge | 8    | 15         | 2 x 80        |
| c3.4xlarge | 16   | 30         | 2 x 160       |
| c3.8xlarge | 32   | 60         | 2 x 320       |

服务器和客户端机器按照上一篇文章做了基本的优化。

以下是测试的配置数据：

- 20 clients
- setup rate设为500 * 20 requests/second = 10000 request /second
- 每个client负责建立50000个websocket 连接
- 等1,000,000个websocket建好好，发送一个消息(时间戳)给所有的客户端，客户端根据时间戳计算latency
- 如果服务器setup rate建立很慢，主动停止测试
- 监控三个阶段的性能指标： setup时， setup完成后应用发呆(idle)时，发送消息时

# 2. 测试结果

## 2.1 Netty

**Setup时**

- cpu idle: 90%
- minor gc: Few
- full gc: No

**Setup完成， 应用Idle时**

- cpu idle: 100%
- memory usage: 1.68G
- server free memory: 16.3G

**发送消息时**

- cpu idle: 75%

- minor gc: few

- full gc: No

- Message latency (one client)

  ```
         count = 50000
           min = 0
           max = 18301
          mean = 2446.09
        stddev = 3082.11
        median = 1214.00
          75% <= 3625.00
          95% <= 8855.00
          98% <= 12069.00
          99% <= 13274.00
        99.9% <= 18301.00
  ```

## 2.2 Vert.x

**Setup时**

- cpu idle: 95%
- minor gc: Few
- full gc: No

**Setup完成， 应用Idle时**

- cpu idle: 100%
- memory usage: 6.37G
- server free memory: 16.3G

**发送消息时**

- cpu idle: 47% ~ 76%

- minor gc: few

- full gc: few

- Message latency (one client)

  ```
         count = 50000
           min = 49
           max = 18949
          mean = 10427.00
        stddev = 5182.72
        median = 10856.00
          75% <= 14934.00
          95% <= 17949.00
          98% <= 18458.00
          99% <= 18658.00
        99.9% <= 18949.00
  ```

## 2.3 Undertow

**Setup时**

- cpu idle: 90%
- minor gc: Few
- full gc: No

**Setup完成， 应用Idle时**

- cpu idle: 100%
- memory usage: 4.02G
- server free memory: 14.2G

**发送消息时**

- cpu idle: 65%

- minor gc: few

- full gc: No

- Message latency

  ```
         count = 50000
           min = 1
           max = 11948
          mean = 1366.86
        stddev = 2007.77
        median = 412.00
          75% <= 2021.00
          95% <= 5838.00
          98% <= 7222.00
          99% <= 8051.00
        99.9% <= 11948.00
  ```

## 2.4 Jetty

**Setup时**

- cpu idle: 2%
- minor gc: Many
- full gc: No
- memory usage: 5G
- server free memory: 17.2G

*当建立360,000左右的websocket时， setup非常的慢， gc频繁，无法继续正常建立websocket， 主动终止测试。*

## 2.5 Grizzly

**Setup时**

- cpu idle: 20%
- minor gc: Some
- full gc: Some
- memory usage: 11.5G
- server free memory: 12.3G

*当建立500,000左右的websocket时， setup非常的慢， gc频繁，无法继续正常建立websocket， 主动终止测试。*

## 2.6 Spray

**Setup时**

- cpu idle: 80%
- minor gc: Many
- full gc: No

*当建立500,000左右的websocket时， setup非常的慢， gc频繁，无法继续正常建立websocket， 主动终止测试。*

## 2.7 Node.js

**Setup时**

- cpu idle: 94%

**Setup完成， 应用Idle时**

- cpu idle: 100%
- memory usage: 5.0G
- server free memory: 16.3G

**发送消息时**

- cpu idle: 94%

- Message latency (one client)

- Message latency

  ```
         count = 50000
           min = 0
           max = 18
          mean = 1.27
        stddev = 3.08
        median = 1.00
          75% <= 1.00
          95% <= 1.00
          98% <= 1.00
          99% <= 1.00
        99.9% <= 15.00
  ```

## 2.8 Go

**Setup时**

- cpu idle: 94%

**Setup完成， 应用Idle时**

- cpu idle: 100%
- memory usage: 15G
- server free memory: 6G

**发送消息时**

- cpu idle: 94%

- Message latency (one client)

- Message latency

  ```
         count = 50000
           min = 0
           max = 35
          mean = 1.89
        stddev = 1.83
        median = 1.00
          75% <= 1.00
          95% <= 2.00
          98% <= 2.00
          99% <= 4.00
        99.9% <= 34.00
  ```

# 3. 测试结果分析

- Netty, Go, Node.js, Undertow, Vert.x都能正常建立百万连接。 Jetty, Grizzly 和 Spray未能完成百万连接
- Netty表现最好。内存占用非常的少， CPU使用率也不高。 尤其内存占用，远远小于其它框架
- Jetty, Grizzly和Spray会产生大量的中间对象，导致垃圾回收频繁。Jetty表现最差
- Node.js表现非常好。 尤其是测试中使用单实例单线程，建立速度非常快，消息的latency也很好。 内存占用也不错
- Undertow表现也不错，内存占用比Netty高一些，其它差不多
- 这里还未测到Spray另一个不好的地方。 在大量连接的情况小，即使没有消息发送，Spray也会占用40% CPU 时间