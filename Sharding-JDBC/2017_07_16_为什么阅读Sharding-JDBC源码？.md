title: Sharding-JDBC 源码分析 —— 为什么阅读 Sharding-JDBC 源码？
date: 2017-07-16
tags:
categories: Sharding-JDBC
permalink: Sharding-JDBC/why-read-Sharding-JDBC-source-code

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------


## 为什么阅读 Sharding-JDBC 源码？

1. 看完大部分的 MyCAT 源码，有惊喜的地方，也有失望的地方，因而想看看 Sharding-JDBC 进行下对比。尽管，Sharding-JDBC 是 Client 端级别，MyCAT 是 Server 级别。
2. Sharding-JDBC 经历过当当本身业务的考验，从可靠性上来说会更让人有信赖感。
3. 文档更加完善，开发体系更加健全。
4. Sharding-JDBC 1.5.0.M3 发布。
5. **最大努力送达型**事务支持，想要进一步了解分布式事务的解决方案。Last But Very Importment。

## 使用公司

1. 京东 ( FROM 民间 )
1. 唯品会 ( FROM 民间 )
1. 转转 ( FROM https://github.com/dangdangdotcom/sharding-jdbc/issues/234 )
1. 足记 ( FROM https://github.com/dangdangdotcom/sharding-jdbc/issues/234 )
1. **不定义更新** ( FROM https://github.com/dangdangdotcom/sharding-jdbc/issues/234 )

## 步骤

* FROM MyCAT

    **从 MyCAT 阅读计划复制，用于对比。**

* [x] ~~NIO~~
* [x] ~~分布式事务~~
* [x] ~~MyCAT 主从~~
* [x] ~~支持prepare预编译指令~~
* [x] 自增序列
* [x] 单库任意 Join
* [x] ~~跨库2表 Join~~
* [x] ~~跨库多表 Join~~
* [x] SQL 解析
* [ ] 读写分离
* [x] ~~MySQL 主从~~
* [x] ~~自动故障切换~~
* [x] ~~Galera Cluster 集群~~
* [x] ~~MHA 集群~~
* [x] ~~Percona 集群~~
* [x] ~~服务降级~~
* [x] ~~多租户~~
* [x] 路由
* [x] ~~MyCAT 集群~~
* [x] ~~注解~~
* [x] ~~缓存~~
* [x] ~~监控~~
* [x] ~~Mongodb~~
* [ ] 内存管理
* [ ] 数据聚合
* [ ] 数据排序
* [x] 分表
* [x] 分库
* [x] ~~全局表~~
* [x] ~~E/R关系~~
* [x] ~~服务降级~~
* [x] ~~SQL 注入攻击拦截~~
* [x] ~~MySQL 协议~~
* [x] ~~PostgreSQL 协议~~
* [ ] 存储过程

* FROM Sharding-JDBC

    **从 官网 介绍获取。**
    
* [ ] 分布式事务 ：最大努力送达型事务
* [ ] 分布式事务 ：TCC型事务(TBD)


