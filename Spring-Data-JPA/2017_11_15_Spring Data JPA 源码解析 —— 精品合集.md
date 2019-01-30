title: Spring Data JPA 实现原理与源码解析系统 —— 精品合集
date: 2017-11-15
tags:
categories:
permalink: Spring-Data-JPA/good-collection

-------

摘要: 原创出处 http://www.iocoder.cn/Spring-Data-JPA/good-collection/ 「芋道源码」欢迎转载，保留摘要，谢谢！

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# 1. Spring For All 社区 Spring Data JPA 从入门到进阶系列教程

* 作者 ：Spring For All
* 博客 ：http://www.spring4all.com/article/500

** Spring Data JPA 入门系列 **

- [《Spring Data JPA系列：基本配置》](http://www.spring4all.com/article/459)
- [《Spring Data JPA系列:继承的方法》](http://www.spring4all.com/article/460)
- [《Spring Data JPA系列:创建查询（Query creation）》](http://www.spring4all.com/article/462)
- [《Spring Data JPA系列：预定义查询（NamedQueries）》](http://www.spring4all.com/article/463)
- [《Spring Data JPA系列：使用@Query注解（Using @Query）》](http://www.spring4all.com/article/464)
- [《Spring Data JPA系列：使用Sort进行排序（Using Sort）》](http://www.spring4all.com/article/465)
- [《Spring Data JPA系列：使用@Modifying修改（Modifying queries）》](http://www.spring4all.com/article/466)
- [《Spring Data JPA系列：应用查询提示（Applying query hints）》](http://www.spring4all.com/article/467)
- [《Spring Data JPA系列：分页（Pageable）》](http://www.spring4all.com/article/468)
- [《Spring Data JPA系列：投影（Projection）的用法》](http://www.spring4all.com/article/469)
- [《Spring Data JPA系列：数据更新（Update）》](http://www.spring4all.com/article/470)
- [《Spring Data JPA系列：数据查询（Specification）（一）》](http://www.spring4all.com/article/471)
- [《Spring Data JPA系列：数据查询（Specification）（二）》](http://www.spring4all.com/article/472)

** Spring Data JPA 实践系列 **

- [《Spring Data 项目领导喊你来实战》](http://www.spring4all.com/article/522)

** Spring Data JPA 进阶系列 **

- [《技术专题讨论第二期总结：如何对 JPA 或者 MyBatis 进行技术选型》](http://www.spring4all.com/article/391)
- [《Spring Data Jpa 让@Query复杂查询分页支持实体返回》](http://www.spring4all.com/article/290)
- [《JPA的 多表 复杂查询 详细篇》](http://www.spring4all.com/article/164)
- [《Spring Boot 两种多数据源配置：JdbcTemplate、Spring-data-jpa》](http://www.spring4all.com/article/253)

** Spring Data JPA 源码分析系列 **

- [《Repository方法名查询推导（Query Derivation From Method Names）的实现原理》](http://www.spring4all.com/article/519)
- [《Spring Data JPA 方法名查询推导的实现原理2后半部分》](http://www.spring4all.com/article/520)
- [《Spring Data JPA 源码阅读笔记：Repository方法名查询推导的实现原理 3 》](http://www.spring4all.com/article/521)

# 2. 其他精品文章

- [《[JavaEE - JPA] 1. 事务的基础概念》](https://blog.csdn.net/dm_vincent/article/details/52566964)
- [《[JavaEE - JPA] 2. EJB中的事务管理》](https://blog.csdn.net/dm_vincent/article/details/52579719)
- [《[JavaEE - JPA] 3. Spring Framework中的事务管理》](https://blog.csdn.net/dm_vincent/article/details/52615499)
- [《[JavaEE - JPA] 4. EntityManager相关核心概念》](https://blog.csdn.net/dm_vincent/article/details/52615532)
- [《[JavaEE - JPA] 5. ORM的核心注解 - 访问方式，表映射以及主键生成》](https://blog.csdn.net/dm_vincent/article/details/52695373)
- [《[JavaEE - JPA] 6. ORM的核心注解 - 基础类型以及嵌套类型》](https://blog.csdn.net/dm_vincent/article/details/52843078)
- [《[JavaEE - JPA] 7. ORM的核心注解 - 关系类型》](https://blog.csdn.net/dm_vincent/article/details/52877296)
- [《[JavaEE - JPA] 性能优化: 4种触发懒加载的方式》](https://blog.csdn.net/dm_vincent/article/details/53366934)
- [《[JavaEE - JPA] 性能优化: 如何定位性能问题》](https://blog.csdn.net/dm_vincent/article/details/53444490)

# 666. 欢迎投稿

![](http://www.iocoder.cn/images/common/zsxq/01.png)