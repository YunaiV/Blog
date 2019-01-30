title: Nacos 源码分析 —— Raft 如何发布内容
date: 2019-01-26
tag:
categories: Nacos
permalink: Nacos/huangyunbin/how-will-Raft-deliver-content
author: 黄云斌
from_url: https://www.jianshu.com/p/3e40376a34db
wechat_url: 

-------

摘要: 原创出处 https://www.jianshu.com/p/3e40376a34db 「黄云斌」欢迎转载，保留摘要，谢谢！

- [在上一篇介绍如何选举后，发布内容就相对简单很多了。](http://www.iocoder.cn/Nacos/huangyunbin/how-will-Raft-deliver-content/)
- [onPublish可以当做是一次心跳了，更新选举检查时间，然后一个重点就是term增加100了。](http://www.iocoder.cn/Nacos/huangyunbin/how-will-Raft-deliver-content/)
- [当然还是就是更新内容了，先写文件，再更新内存缓存。](http://www.iocoder.cn/Nacos/huangyunbin/how-will-Raft-deliver-content/)
- [内存的缓存其实就是一个ConcurrentHashMap](http://www.iocoder.cn/Nacos/huangyunbin/how-will-Raft-deliver-content/)
- [之前也说到这个term很重要，那么自然是要持久化到文件了。](http://www.iocoder.cn/Nacos/huangyunbin/how-will-Raft-deliver-content/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# 在上一篇介绍如何选举后，发布内容就相对简单很多了。

发布内容的入口

![](http:///upload-images.jianshu.io/upload_images/7835103-5dd6dfd8d9c00a1a.png)

signalPublish 的很简单

#### 如果自己不是leader就转发给leader

![](http:///upload-images.jianshu.io/upload_images/7835103-65eff3bf82dea3b5.png)

#### 如果自己是leader，就向所有节点发送onPublish请求，注意这里是所有的节点，包括自己。

![](http:///upload-images.jianshu.io/upload_images/7835103-af73bcd909d94638.png)

所以其实还是看onPublish的逻辑了

![](http:///upload-images.jianshu.io/upload_images/7835103-1a7194182a0d85a7.png)

# onPublish可以当做是一次心跳了，更新选举检查时间，然后一个重点就是term增加100了。

# 当然还是就是更新内容了，先写文件，再更新内存缓存。

![](http:///upload-images.jianshu.io/upload_images/7835103-fa1cf20652d8e848.png)

可以看到写文件的时候，一个key就是一个文件，文件的内容就是value这个json

![](http:///upload-images.jianshu.io/upload_images/7835103-f930306ea68e8cff.png)

# 内存的缓存其实就是一个ConcurrentHashMap

```Java
private static ConcurrentMap<String, Datum> datums = new ConcurrentHashMap<String, Datum>();
```

RaftCore.datums.put(datum.key, datum);

# 之前也说到这个term很重要，那么自然是要持久化到文件了。

![](http:///upload-images.jianshu.io/upload_images/7835103-f9add0898f5c3db9.png)
          

