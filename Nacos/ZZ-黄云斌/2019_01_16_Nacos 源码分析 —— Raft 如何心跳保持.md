title: Nacos 源码分析 —— Raft 如何心跳保持
date: 2019-01-16
tag:
categories: Nacos
permalink: Nacos/huangyunbin/312ca3c9e6756c1ee1c2128db2f8229e
author: 黄云斌
from_url: https://www.jianshu.com/p/b0cdaa64688e
wechat_url: 

-------

摘要: 原创出处 https://www.jianshu.com/p/b0cdaa64688e 「黄云斌」欢迎转载，保留摘要，谢谢！

  - [raft的一个基本逻辑是leader隔一段时间给所有的follower发心跳。如果follower长时间没收到心跳，就认为leader已经挂了，就发起投票选举新的leader。](http://www.iocoder.cn/Nacos/huangyunbin/312ca3c9e6756c1ee1c2128db2f8229e/)
- [HeartBeat 就是leader的心跳定时任务](http://www.iocoder.cn/Nacos/huangyunbin/312ca3c9e6756c1ee1c2128db2f8229e/)
- [MasterElection 就是follower长时间没收到心跳就选举的定时任务](http://www.iocoder.cn/Nacos/huangyunbin/312ca3c9e6756c1ee1c2128db2f8229e/)
  - [HeartBeat的sendBeat就是具体发送心跳信息了](http://www.iocoder.cn/Nacos/huangyunbin/312ca3c9e6756c1ee1c2128db2f8229e/)
- [follower收到心跳请求的时候](http://www.iocoder.cn/Nacos/huangyunbin/312ca3c9e6756c1ee1c2128db2f8229e/)
  - [receivedBeat 方法会执行 resetLeaderDue();](http://www.iocoder.cn/Nacos/huangyunbin/312ca3c9e6756c1ee1c2128db2f8229e/)
  - [follower就是根据这个变量判断是否要重新选leader的。](http://www.iocoder.cn/Nacos/huangyunbin/312ca3c9e6756c1ee1c2128db2f8229e/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

## raft的一个基本逻辑是leader隔一段时间给所有的follower发心跳。如果follower长时间没收到心跳，就认为leader已经挂了，就发起投票选举新的leader。

在RaftCore的init方法里面

![](http://upload-images.jianshu.io/upload_images/7835103-6d834612b1e1e85b.png)

# HeartBeat 就是leader的心跳定时任务

# MasterElection 就是follower长时间没收到心跳就选举的定时任务

-------

## HeartBeat的sendBeat就是具体发送心跳信息了

![](http://upload-images.jianshu.io/upload_images/7835103-f6e57e02d85d1915.png)

-------

# follower收到心跳请求的时候

![](http://upload-images.jianshu.io/upload_images/7835103-bac145002e3815e2.png)

## receivedBeat 方法会执行 resetLeaderDue();

![](http://upload-images.jianshu.io/upload_images/7835103-32229cd18325135b.png)

## follower就是根据这个变量判断是否要重新选leader的。

![](http://upload-images.jianshu.io/upload_images/7835103-ca5e1adc87e8fe7c.png)

#### 这样就全部串起来了