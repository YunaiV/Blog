title: Nacos 源码分析 —— Raft 如何保证内容一致
date: 2019-01-31
tag:
categories: Nacos
permalink: Nacos/huangyunbin/How-does-Raft-keep-things-consistent
author: 黄云斌
from_url: https://www.jianshu.com/p/b30245d1713e
wechat_url: 

-------

摘要: 原创出处 https://www.jianshu.com/p/b30245d1713e 「黄云斌」欢迎转载，保留摘要，谢谢！

- [其实是通过心跳来保证的](http://www.iocoder.cn/Nacos/huangyunbin/How-does-Raft-keep-things-consistent/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------


上文说到的raft内容发布，那其实是对于某一个key来说的，而且还不保证发布的成功。那要如何保证数据的最终一致呢。

# 其实是通过心跳来保证的

#### leader发送心跳的时候会带上所有的key和key对应的时间戳，follower发现key不存在或者时间戳比较小，就发送请求给leader拿这些key的数据，最终达到数据一致。

发送心跳带上所有的key

![](http://upload-images.jianshu.io/upload_images/7835103-c70ab5e5577e3d94.png)

##### 这些数据会有点大，所以做了gzip压缩

![](http://upload-images.jianshu.io/upload_images/7835103-ee0ca399898884c6.png)

##### follower收到心跳的时候，判断key的存在和时间戳，和leader对不上的，发起请求拿最新的

![](http://upload-images.jianshu.io/upload_images/7835103-a078f62a3cf45b54.png)

##### 不一致的key可能比较多，所以批量去拿，每次拿50个key。

数据差距可能比较大，一直循环拿的话对leader的压力会比较大，所以拿一次之后slepp 200ms。

##### HttpClient异步去获取这些内容，异步回来可能是不同的线程，会有多线程的问题，所以要加锁。

当然这里HttpClient是同步请求，也会是要加锁的，因为受到心跳请求是不同的线程，如果心跳处理比较慢，也变成多线程处理了。

![](http://upload-images.jianshu.io/upload_images/7835103-3dcf4fe8c8f7adab.png)

##### 数据更新后会发消息，通知有变更

```Java
notifier.addTask(datum, Notifier.ApplyAction.CHANGE);
```

-------

#### 上面的流程只是处理了key的新增和更新，那key的删除怎么同步呢？

本地的所有的key放到一个map，处理过的key标记一下，如果到最后还有key没标记，说明是本地有，但是leader没有。这种key就是leader已经删除的key了。

![](http://upload-images.jianshu.io/upload_images/7835103-383ab8d1c877a98c.png)

-------

#### 问题1 心跳发送所有的key及其时间戳，如果有几十万个key呢，而且是500ms一次的心跳，压力真的很大啊。

不知道阿里的人哪里来的信心用这种方案，如果我提出这种方案，估计早就被老板砍死了。

#### 问题二  用时间戳来作为判断一个key的内容是否一致真的问题很大啊。时间戳只能精确到1ms，1ms发布多次的话就判断不了了。

而且时间同步之后，时间甚至可能回退，通过时间戳大小的判断短时间内就会失效，难道是要所有的机器禁止时间同步吗。

就算是禁止了时间同步，机器之间的时间差是客观存在的，假如leader的变化在很快的时间内完成，新leader可能会落后旧leader一段时间的，这时间差也是可能出问题的啊。

          

