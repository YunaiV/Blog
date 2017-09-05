title: RocketMQ 之 Namesrv 小结
date: 2017-04-05
tags:
categories: RocketMQ
permalink: RocketMQ/namesrv-intro

-------

![](http://www.yunai.me/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

## Namesrv组件

* KVConfigManager：KV配置管理
   * key-value配置管理，增删改查
* RouteInfoManager：路由信息管理
   * 注册Broker，提供Broker信息（名字、角色编号、地址、集群名）
   * 注册Topic，提供Topic信息（Topic名、读写权限、队列情况）

