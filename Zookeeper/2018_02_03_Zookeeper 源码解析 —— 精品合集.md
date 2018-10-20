title: Zookeeper 实现原理与源码解析系统 —— 精品合集
date: 2018-02-02
tags:
categories:
permalink: Zookeeper/good-collection

-------

摘要: 原创出处 http://www.iocoder.cn/Zookeeper/good-collection/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1.【leesf】Zookeeper 源码解析](http://www.iocoder.cn/Zookeeper/good-collection/)
- [666. 欢迎投稿](http://www.iocoder.cn/Zookeeper/good-collection/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2018_05_18.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

# 1.【leesf】Zookeeper 源码解析

* 作者 ：leesf
* 博客 ：http://www.cnblogs.com/leesf456/p/6518040.html
* 目录 ：

* [【Zookeeper】源码分析之序列化](http://www.cnblogs.com/leesf456/p/6278853.html)
* [【Zookeeper】源码分析之持久化（一）之FileTxnLog](http://www.cnblogs.com/leesf456/p/6279956.html)
* [【Zookeeper】源码分析之持久化（二）之FileSnap](http://www.cnblogs.com/leesf456/p/6285014.html)
* [【Zookeeper】源码分析之持久化（三）之FileTxnSnapLog](http://www.cnblogs.com/leesf456/p/6285703.html)
* [【Zookeeper】源码分析之Watcher机制（一）](http://www.cnblogs.com/leesf456/p/6286827.html)
* [【Zookeeper】源码分析之Watcher机制（二）之WatchManager](http://www.cnblogs.com/leesf456/p/6288709.html)
* [【Zookeeper】源码分析之Watcher机制（三）之ZooKeeper](http://www.cnblogs.com/leesf456/p/6291004.html)
* [【Zookeeper】源码分析之请求处理链（一）](http://www.cnblogs.com/leesf456/p/6410793.html)
* [【Zookeeper】源码分析之请求处理链（二）之PrepRequestProcessor](http://www.cnblogs.com/leesf456/p/6412843.html)
* [【Zookeeper】源码分析之请求处理链（三）之SyncRequestProcessor](http://www.cnblogs.com/leesf456/p/6438411.html)
* [【Zookeeper】源码分析之请求处理链（四）之FinalRequestProcessor](http://www.cnblogs.com/leesf456/p/6472496.html)
* [【Zookeeper】源码分析之网络通信（一）](http://www.cnblogs.com/leesf456/p/6477815.html)
* [【Zookeeper】源码分析之网络通信（二）之NIOServerCnxn](http://www.cnblogs.com/leesf456/p/6484780.html)
* [【Zookeeper】源码分析之网络通信（三）之NettyServerCnxn](http://www.cnblogs.com/leesf456/p/6486454.html)
* [【Zookeeper】源码分析之Leader选举（一）](http://www.cnblogs.com/leesf456/p/6494290.html)
* [【Zookeeper】源码分析之Leader选举（二）之FastLeaderElection](http://www.cnblogs.com/leesf456/p/6508185.html)
* [【Zookeeper】源码分析之服务器（一）](http://www.cnblogs.com/leesf456/p/6514897.html)
* [【Zookeeper】源码分析之服务器（二）之ZooKeeperServer](http://www.cnblogs.com/leesf456/p/6515105.html)
* [【Zookeeper】源码分析之服务器（三）之LeaderZooKeeperServer](http://www.cnblogs.com/leesf456/p/6516805.html)
* [【Zookeeper】源码分析之服务器（四）之FollowerZooKeeperServer](http://www.cnblogs.com/leesf456/p/6517058.html)
* [【Zookeeper】源码分析之服务器（五）之ObserverZooKeeperServer](http://www.cnblogs.com/leesf456/p/6517945.html)

# 666. 彩蛋

![](http://www.iocoder.cn/images/common/zsxq/01.png)

