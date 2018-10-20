title: Sharding-JDBC 源码分析 —— SQL 解析（七）之DDL
date: 2017-08-03
tags:
categories: Sharding-JDBC
permalink: Sharding-JDBC/sql-parse-7
keywords: Sharding-JDBC,ShardingJDBC,Sharding-JDBC 源码,SQL解析, SQL 解析

-------

摘要: 原创出处 http://www.iocoder.cn/Sharding-JDBC/sql-parse-7/ 「芋道源码」欢迎转载，保留摘要，谢谢！

抱歉，站坑文。近期看情况更新。

-------

![](https://www.iocoder.cn/images/common/wechat_mp_2018_05_18.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

