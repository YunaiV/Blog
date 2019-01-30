title: Nacos 源码分析 —— Raft 如何选举
date: 2019-01-21
tag:
categories: Nacos
permalink: Nacos/huangyunbin/How-Raft-is-elected
author: 黄云斌
from_url: https://www.jianshu.com/p/5a2d965174ae
wechat_url: 

-------

摘要: 原创出处 https://www.jianshu.com/p/5a2d965174ae 「黄云斌」欢迎转载，保留摘要，谢谢！

- [发起请求：](http://www.iocoder.cn/Nacos/huangyunbin/How-Raft-is-elected/)
- [其他节点收到选举请求](http://www.iocoder.cn/Nacos/huangyunbin/How-Raft-is-elected/)
- [如果对方的term比自己小，voteFor为自己，然后返回结果。意思是我自己更适合做leader，这一票我投给自己。](http://www.iocoder.cn/Nacos/huangyunbin/How-Raft-is-elected/)
- [如果对方的term比自己大，设置voteFor为对方，然后返回结果，意思是就按你说的做，这一票就投给你了。](http://www.iocoder.cn/Nacos/huangyunbin/How-Raft-is-elected/)
- [把所有的节点投票信息放到TreeBag，这个可以看成是个按value排序的有序map。排第一的就是得票最多的节点](http://www.iocoder.cn/Nacos/huangyunbin/How-Raft-is-elected/)
- [假如一个节点选举自己成功，他会认为自己是leader，就会定时发送心跳给其他的节点，这个时候其他节点的leader还是旧的，收到心跳会报错的。](http://www.iocoder.cn/Nacos/huangyunbin/How-Raft-is-elected/)
- [所以其他节点都经历一次选举：](http://www.iocoder.cn/Nacos/huangyunbin/How-Raft-is-elected/)
- [因为已经选举成功过，所以local.voteFor都有值，为上一次选举成功的节点，所以其他节点选举的结果都会统一了。](http://www.iocoder.cn/Nacos/huangyunbin/How-Raft-is-elected/)
- [但是这里有个关键逻辑就是term的比较，这个是决定了所有的逻辑的。](http://www.iocoder.cn/Nacos/huangyunbin/How-Raft-is-elected/)
- [假如节点2开始选举，它的term是最高的，选举自己是可以成功的。](http://www.iocoder.cn/Nacos/huangyunbin/How-Raft-is-elected/)
- [假如节点2和节点3同时选举呢，节点2得到自己和节点4的票，节点3得到自己和节点5的票。这个时候两边都不能成功。所以等待下一轮，因为下一次开始的时间是随机的，所以同时的概率很小。谁先，谁就是新的leader了。](http://www.iocoder.cn/Nacos/huangyunbin/How-Raft-is-elected/)
- [假如所有的节点的term相同，其实是选举不出leader的，因为都只有自己一票。这个是怎么解决的呢？](http://www.iocoder.cn/Nacos/huangyunbin/How-Raft-is-elected/)
- [收到投票请求的时候，如果对方的term比自己的大，为什么要放弃这一轮的发起选举](http://www.iocoder.cn/Nacos/huangyunbin/How-Raft-is-elected/)
- [为什么每次发布新的内容，term都会加100呢](http://www.iocoder.cn/Nacos/huangyunbin/How-Raft-is-elected/)
- [问题1](http://www.iocoder.cn/Nacos/huangyunbin/How-Raft-is-elected/)
- [问题2](http://www.iocoder.cn/Nacos/huangyunbin/How-Raft-is-elected/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

在上一篇文章说到了如果follower长时间收不到leader的心跳，就会发起leader选举。具体的过程是怎么样的呢？

# 发起请求：



![img](https:////upload-images.jianshu.io/upload_images/7835103-dd15d97d9659148c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/jpeg)


public static final String API_VOTE = UtilsAndCommons.NACOS_NAMING_CONTEXT + "/raft/vote";

# 其他节点收到选举请求



![img](https:////upload-images.jianshu.io/upload_images/7835103-c4ca4c8c8109aa25.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/jpeg)


收到请求的处理过程是：

# 如果对方的term比自己小，voteFor为自己，然后返回结果。意思是我自己更适合做leader，这一票我投给自己。

# 如果对方的term比自己大，设置voteFor为对方，然后返回结果，意思是就按你说的做，这一票就投给你了。



![img](https:////upload-images.jianshu.io/upload_images/7835103-ba6afd1deb1d2be5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/jpeg)



发起投票的节点收集到回应之后就开始处理了：



![img](https:////upload-images.jianshu.io/upload_images/7835103-e9d1cd35e8dfedb8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/jpeg)





![img](https:////upload-images.jianshu.io/upload_images/7835103-e94467e6d5884566.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/jpeg)



# 把所有的节点投票信息放到TreeBag，这个可以看成是个按value排序的有序map。排第一的就是得票最多的节点

public int majorityCount() {
 return peers.size() / 2 + 1;
 }
 超过半数，表示选举新的leader成功。

我们发现这的leader成功，并不会通知其他节点修改leader。最后是怎么变成一致的呢？

# 假如一个节点选举自己成功，他会认为自己是leader，就会定时发送心跳给其他的节点，这个时候其他节点的leader还是旧的，收到心跳会报错的。



![img](https:////upload-images.jianshu.io/upload_images/7835103-9c2d3e7157ae3bac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/jpeg)



# 所以其他节点都经历一次选举：



![img](https:////upload-images.jianshu.io/upload_images/7835103-36c7263c0516c548.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/jpeg)



# 因为已经选举成功过，所以local.voteFor都有值，为上一次选举成功的节点，所以其他节点选举的结果都会统一了。

看起来是不是很简单啊。。。

# 但是这里有个关键逻辑就是term的比较，这个是决定了所有的逻辑的。

每次发布新的内容的时候，term都会增加。而且follower的term也会增加，最终会同步为leader的term。



![img](https:////upload-images.jianshu.io/upload_images/7835103-54dcb90478468e70.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/jpeg)



------

### 举个例子

假如5个节点
 节点1的term为4  为leader
 节点2的term为4
 节点3的term为4
 节点4的term为3
 节点5的term为3
 节点1这个leader挂的情况下，

# 假如节点2开始选举，它的term是最高的，选举自己是可以成功的。

同样，节点3也是可以选举成功的。这个就看节点2，3谁先开始选举了，谁先，谁就是新的leader。

# 假如节点2和节点3同时选举呢，节点2得到自己和节点4的票，节点3得到自己和节点5的票。这个时候两边都不能成功。所以等待下一轮，因为下一次开始的时间是随机的，所以同时的概率很小。谁先，谁就是新的leader了。

------

# 假如所有的节点的term相同，其实是选举不出leader的，因为都只有自己一票。这个是怎么解决的呢？

每次发起投票的时候都会给自己的term加1 ，是这里制造term的差异的：



![img](https:////upload-images.jianshu.io/upload_images/7835103-32b3416734492ee3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/jpeg)



------

# 收到投票请求的时候，如果对方的term比自己的大，为什么要放弃这一轮的发起选举

```Java
        local.resetLeaderDue();
```



![img](https:////upload-images.jianshu.io/upload_images/7835103-15ca7c065383fb2a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/jpeg)



这个是为了减少选举冲突。对方比自己的term大1，自己不放弃这一轮选举的话，自己发起选举，term会加1，其实term就一样大了，可能的结果就是两个都选举不成功。

------

# 为什么每次发布新的内容，term都会加100呢

local.term.addAndGet(PUBLISH_TERM_INCREASE_COUNT);
 public static final int PUBLISH_TERM_INCREASE_COUNT = 100;
 上面也看到了，每次发起投票都term都会加1，如果发布内容也是加一的话，内容落后的节点第二次发起投票的时候就是加2了，term居然高过内容最新的节点。这个时候就不对了。
 100其实就是允许重新发起投票的次数，这个数字越大越安全，100这个数字已经足够大了，100轮投票都产生不了leader，这个概率可以忽略不计了。

------

# 问题1

假如一个节点只是和leader不通，和其他节点都是通的。刚开始的时候，他的term其实是最新的，所以它是可以成功选自己为leader的。
 这个时候看起来就会有两个leader，其他节点认为旧的leader是ok的，所以不会重新投票选举。但是其他节点会受到这两个leader的心跳，只是对于第二个心跳会报错而已。。这种情况确实有点蛋疼，不过理论上很少发生这种情况的。

------

# 问题2

选举发生冲突，都失败的时候，等待下一轮选举的时间是15~20秒，感觉这个时间等得太久了。
 而且随机的区间就是0-5000 ms，这个命中比较接近的数字还不小，搞不好下一轮还冲突。那就一共等30多秒了。。
 public void resetLeaderDue() {
 leaderDueMs = GlobalExecutor.LEADER_TIMEOUT_MS + RandomUtils.nextLong(0, GlobalExecutor.RAMDOM_MS);
 }

public static final long LEADER_TIMEOUT_MS = TimeUnit.SECONDS.toMillis(15L);

```Java
public static final long RAMDOM_MS = TimeUnit.SECONDS.toMillis(5L);
```