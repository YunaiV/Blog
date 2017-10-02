title: Eureka 源码解析 —— Eureka-Server 启动（二）之 ServerConfig
date: 2018-05-07
tags:
categories: Eureka
permalink: Eureka/eureka-server-init-second

---

摘要: 原创出处 http://www.iocoder.cn/Eureka/eureka-server-init-second/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Eureka 1.8.X 版本** 

TODO

---

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

---

# 1. 概述

本文接[《Eureka 源码解析 —— Eureka-Server 启动（一）之 EurekaServerConfig》](http://www.iocoder.cn/Eureka/eureka-server-init-first/?self)，主要分享 **Eureka-Server 启动的过程**的第二部分 —— **EurekaBootStrap**。

考虑到整个初始化的过程中涉及的代码特别多，拆分成两两篇文章：

* [ServerConfig](http://www.iocoder.cn/Eureka/eureka-server-init-first/?self)
* 【本文】EurekaBootStrap

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。

# 2. 

