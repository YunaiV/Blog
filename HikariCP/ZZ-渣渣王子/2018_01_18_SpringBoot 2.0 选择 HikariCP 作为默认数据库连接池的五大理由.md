title: 【追光者系列】SpringBoot 2.0 选择 HikariCP 作为默认数据库连接池的五大理由
date: 2018-01-18
tags:
categories: HikariCP
permalink: HikariCP/zhazhawangzi/springboot-2-default
author: 渣渣王子
from_url: https://mp.weixin.qq.com/s/gXw14pQStgr78NqC0K6Ufg
wechat_url:

-------

摘要: 原创出处 https://mp.weixin.qq.com/s/gXw14pQStgr78NqC0K6Ufg 「渣渣王子」欢迎转载，保留摘要，谢谢！

- [Springboot2默认数据库连接池选择了HikariCP](http://www.iocoder.cn/HikariCP/zhazhawangzi/springboot-2-default/)
- [为何选择HikariCP](http://www.iocoder.cn/HikariCP/zhazhawangzi/springboot-2-default/)
  - [理由一、代码量](http://www.iocoder.cn/HikariCP/zhazhawangzi/springboot-2-default/)
  - [理由二、口碑](http://www.iocoder.cn/HikariCP/zhazhawangzi/springboot-2-default/)
  - [理由三、速度](http://www.iocoder.cn/HikariCP/zhazhawangzi/springboot-2-default/)
  - [理由四、稳定性](http://www.iocoder.cn/HikariCP/zhazhawangzi/springboot-2-default/)
  - [理由五、可靠性](http://www.iocoder.cn/HikariCP/zhazhawangzi/springboot-2-default/)
- [HikariCP为什么这么快](http://www.iocoder.cn/HikariCP/zhazhawangzi/springboot-2-default/)
  - [优化并精简字节码](http://www.iocoder.cn/HikariCP/zhazhawangzi/springboot-2-default/)
  - [ConcurrentBag：更好的并发集合类实现](http://www.iocoder.cn/HikariCP/zhazhawangzi/springboot-2-default/)
  - [使用FastList替代ArrayList](http://www.iocoder.cn/HikariCP/zhazhawangzi/springboot-2-default/)
- [HikariCP与Druid相比哪个更好？](http://www.iocoder.cn/HikariCP/zhazhawangzi/springboot-2-default/)
- [Springboot2快速上手](http://www.iocoder.cn/HikariCP/zhazhawangzi/springboot-2-default/)
- [jdbc_config   datasource](http://www.iocoder.cn/HikariCP/zhazhawangzi/springboot-2-default/)
- [Hikari will use the above plus the following to setup connection pooling](http://www.iocoder.cn/HikariCP/zhazhawangzi/springboot-2-default/)
- [参考资料](http://www.iocoder.cn/HikariCP/zhazhawangzi/springboot-2-default/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# Springboot2默认数据库连接池选择了HikariCP

默认的数据库连接池由Tomcat换成HikariCP. 如果在一个Tomcat应用中用spring.datasource.type来强制使用Hikari连接池, 则可以去掉这个override.

# 为何选择HikariCP

HiKariCP是数据库连接池的一个后起之秀，号称性能最好，可以完美地PK掉其他连接池，是一个高性能的JDBC连接池，基于BoneCP做了不少的改进和优化。其作者还有另外一个开源作品——高性能的JSON解析器HikariJSON。

它，超快，快到连Spring Boot 2都宣布支持了。

代码体积更是少的可怜，130kb。

https://github.com/brettwooldridge/HikariJSON

为何要使用HiKariCP？这要先从BoneCP说起：
什么？不是有C3P0/DBCP这些成熟的数据库连接池吗？一直用的好好的，为什么又搞出一个BoneCP来？因为，传说中BoneCP在快速这个特点上做到了极致，官方数据是C3P0等的25倍左右。不相信？其实我也不怎么信。可是，有图有真相啊（图片来自BoneCP官网：http://jolbox.com/benchmarks.html）：

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnWWL8hroXqE2yOj7UlRfG38srUicDYzduBkCj01DvV0ZVuJe68ZLXaCw1ldVaWadqoKoIAuTiaJutVw/640)

从上述结果可以看出HikariCP的性能远高于c3p0、tomcat等连接池，以致后来BoneCP作者都放弃了维护，在Github项目主页推荐大家使用HikariCP。另外，Spring Boot将在2.0版本中把HikariCP作为其默认的JDBC连接池。

PS:需要指出的是，上图中的数据是HikariCP作者对各个连接池调用DataSource.getConnection()、Connection.close()、Connection.prepareStatement()、Statement.execute()、Statement.close()方法的性能测试结果。

而且，网上对于BoneCP是好评如潮啊，推荐的文章一搜一大堆。

然而，上Maven Repository网站（http://mvnrepository.com/artifact/com.jolbox/bonecp）查找有没有最新版本的时候，你会发现最新的是2013年10月份的（这么久没新版本出来了？）。于是，再去BoneCP的Githut（https://github.com/wwadge/bonecp）上看看最近有没有提交代码。却发现，BoneCP的作者对于这个项目貌似已经心灰意冷，说是要让步给HikariCP了（有图有真相）：

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnWWL8hroXqE2yOj7UlRfG38In6jTBnuN4PVFDwk5xGCdSV2qAzYDXvQicmn1s8eYjv2iaFWic4PErIzQ/640)

……什么？又来一个CP？……什么是Hikari？
Hikari来自日文，是“光”（阳光的光，不是光秃秃的光）的意思。作者估计是为了借助这个词来暗示这个CP速度飞快。不知作者是不是日本人，不过日本也有很多优秀的码农，听说比特币据说日本人搞出来的。。。

这个产品的口号是“快速、简单、可靠”。实际情况跟这个口号真的匹配吗？又是有图有真相（Benchmarks又来了）：

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnWWL8hroXqE2yOj7UlRfG38jx7cPbBNcJpEurGbm2kzlh2AWZkte38ShRP2KWBq7zoxgWGR7NibwZA/640)

这个图，也间接地、再一次地证明了boneCP比c3p0强大很多，当然，跟“光”比起来，又弱了不少啊。

那么，这么好的是怎么做到的呢？官网详细地说明了HikariCP所做的一些优化，总结如下：

- **字节码精简** ：优化代码，直到编译后的字节码最少，这样，CPU缓存可以加载更多的程序代码；
- **优化代理和拦截器**：减少代码，例如HikariCP的Statement proxy只有100行代码，只有BoneCP的十分之一；
- **自定义数组类型（FastStatementList）代替ArrayList**：避免每次get()调用都要进行range check，避免调用remove()时的从头到尾的扫描；
- **自定义集合类型（ConcurrentBag**：提高并发读写的效率；
- **其他针对BoneCP缺陷的优化**，比如对于耗时超过一个CPU时间片的方法调用的研究（但没说具体怎么优化）。

很多优化的对比都是针对BoneCP的……哈哈。
（参考文章：https://github.com/brettwooldridge/HikariCP/wiki/Down-the-Rabbit-Hole）

## 理由一、代码量

几个连接池的代码量对比（代码量越少，一般意味着执行效率越高、发生bug的可能性越低）：

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnWWL8hroXqE2yOj7UlRfG38wY6qGo0ChQg8d4ld25rbSWUToaKZQotAaEjMOV1W3O82Ur6F586ZCA/640)

##

## 理由二、口碑

可是，“黄婆卖瓜，自催自擂”这个俗语日本人也是懂得，于是，用户的好评如潮也是有图有真相：

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnWWL8hroXqE2yOj7UlRfG38NC0V0D9L3FicJSNCVN4JYRDzvuGFttEYbqlzUmrFWIUhymphZcSmiauQ/640)

##

## 理由三、速度

还有第三方关于速度的测试：

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnWWL8hroXqE2yOj7UlRfG38AjFbnibfncicDDML68me3Yt0meFCts7u5sia2zGYObYmyUU8JwksJSlSQ/640)

## 理由四、稳定性

也许你会说，速度高，如果不稳定也是硬伤啊。于是，关于稳定性的图也来了：

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnWWL8hroXqE2yOj7UlRfG38yZguChcF9NCVA5JPNNL0hcIArQia0gxyzPkpzYf8Tc3x2oqeZPUvF4A/640)

## 理由五、可靠性

另外，关于可靠性方面，也是有实验和数据支持的。对于数据库连接中断的情况，通过测试getConnection()，各种CP的不相同处理方法如下：
（所有CP都配置了跟connectionTimeout类似的参数为5秒钟）

- **HikariCP**：等待5秒钟后，如果连接还是没有恢复，则抛出一个SQLExceptions 异常；后续的getConnection()也是一样处理；
- **C3P0**：完全没有反应，没有提示，也不会在“CheckoutTimeout”配置的时长超时后有任何通知给调用者；然后等待2分钟后终于醒来了，返回一个error；
- **Tomcat**：返回一个connection，然后……调用者如果利用这个无效的connection执行SQL语句……结果可想而知；大约55秒之后终于醒来了，这时候的getConnection()终于可以返回一个error，但没有等待参数配置的5秒钟，而是立即返回error；
- **BoneCP**：跟Tomcat的处理方法一样；也是大约55秒之后才醒来，有了正常的反应，并且终于会等待5秒钟之后返回error了；

可见，HikariCP的处理方式是最合理的。根据这个测试结果，对于各个CP处理数据库中断的情况，评分如下：

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnWWL8hroXqE2yOj7UlRfG38ibD9mAJPMPjP3YXTI5lhVdyuydEqN7j0eYreBe5DAHvlDZxNWHOSh6g/640)

参考文章：https://github.com/brettwooldridge/HikariCP/wiki/Bad-Behavior:-Handling-Database-Down

# HikariCP为什么这么快

JDBC连接池的实现并不复杂，主要是对JDBC中几个核心对象Connection、Statement、PreparedStatement、CallableStatement以及ResultSet的封装与动态代理。接下来从几个方面来看看HikariCP为什么这么快：

## 优化并精简字节码

HikariCP利用了一个第三方的Java字节码修改类库Javassist来生成委托实现动态代理。动态代理的实现在ProxyFactory类，源码如下：

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnWWL8hroXqE2yOj7UlRfG38kWGDQ08x84TTnRNABJOxEDkz8WRiaBvqtwKvqPstDpAZUyuzIXn9hCg/640)

发现这些代理方法中只有一行直接抛异常的代码，注释写着“Body is replaced (injected) by JavassistProxyFactory”，其实方法body中的代码是在编译时调用JavassistProxyFactory才生成的，主要代码见下图：

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnWWL8hroXqE2yOj7UlRfG381BiaHlZmkIicukwww3GtC1GLoPh8c0VdEQCC5NyK36k1MydKkB3Gm6XQ/640)

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnWWL8hroXqE2yOj7UlRfG38pW4KDL3I3Qp4xZ0ibglWxc8iaufLLTwZkd6OOdTooUEAVKRKhzOVvVGQ/640)

之所以使用Javassist生成动态代理，是因为其速度更快，相比于JDK Proxy生成的字节码更少，精简了很多不必要的字节码。

## ConcurrentBag：更好的并发集合类实现

ConcurrentBag的实现借鉴于C#中的同名类，是一个专门为连接池设计的lock-less集合，实现了比LinkedBlockingQueue、LinkedTransferQueue更好的并发性能。ConcurrentBag内部同时使用了ThreadLocal和CopyOnWriteArrayList来存储元素，其中CopyOnWriteArrayList是线程共享的。ConcurrentBag采用了queue-stealing的机制获取元素：首先尝试从ThreadLocal中获取属于当前线程的元素来避免锁竞争，如果没有可用元素则再次从共享的CopyOnWriteArrayList中获取。此外，ThreadLocal和CopyOnWriteArrayList在ConcurrentBag中都是成员变量，线程间不共享，避免了伪共享(false sharing)的发生。

## 使用FastList替代ArrayList

FastList是一个List接口的精简实现，只实现了接口中必要的几个方法。JDK ArrayList每次调用get()方法时都会进行rangeCheck检查索引是否越界，FastList的实现中去除了这一检查，只要保证索引合法那么rangeCheck就成为了不必要的计算开销(当然开销极小)。此外，HikariCP使用List来保存打开的Statement，当Statement关闭或Connection关闭时需要将对应的Statement从List中移除。通常情况下，同一个Connection创建了多个Statement时，后打开的Statement会先关闭。ArrayList的remove(Object)方法是从头开始遍历数组，而FastList是从数组的尾部开始遍历，因此更为高效。

# HikariCP与Druid相比哪个更好？

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnWWL8hroXqE2yOj7UlRfG38Cm9wkQF4SCUToNDHD9h9n5ar7iapMlrpvqA7gAaHBoFrY2ibrajvCG3A/640)

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnWWL8hroXqE2yOj7UlRfG38GsKfwfJF2CgS3EquStH45INiazghzpPgoQPegQm5lTIPaicJ9gy6mZJg/640)

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnWWL8hroXqE2yOj7UlRfG38JkwiaokkqM5eUCsncswWoudvO1LaduWhvKIumCVvaTB73XgkSjnq2EA/640)

有些用户给了druid这样的评论：

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnWWL8hroXqE2yOj7UlRfG38f1hssV2bz6gvgxWFFicAhvt4XyXETWpCGRIKfqrRp6SBqicGUeO6dYrQ/640)

不评论，一个追求性能，一个偏向监控，直接看之前有人给HikariCP提的关于跟Druid对比分析的issue吧。HikariCP作者对Druid做了测试并给出了测试结果数据，Druid作者温少也对此作了评论。Issue链接：

https://github.com/brettwooldridge/HikariCP/issues/232

笔者个人的观点是，hikariCP可以提供监控功能的，比如metrics，可以参见笔者的这篇文章 [【追光者系列】HikariCP连接池监控指标实战](http://mp.weixin.qq.com/s?__biz=MzUzNTY4NTYxMA==&mid=2247483699&idx=1&sn=5efd1f9d872688eba0029c71b3668662&chksm=fa80f1b6cdf778a0bbd12ce5e97507d697058d7bc070082a8ca828c9910c1379ead43c7b9a05&scene=21#wechat_redirect)。
另外，监控方面，skywalking、pinpoint、mycat这些agent也是可以做到的，以后service mesh普及了更加可以监控了，比如sharding-jdbc也可以做监控，datamesh，sidecar也可以做监控的。

# Springboot2快速上手

说得这么好，用起来会不会很麻烦啊，会不会有很多参数要配置才能有这样的效果啊？答案是：不会。

springboot 2.0 默认连接池就是Hikari了，所以引用parents后不用专门加依赖

配置一下就好

```
# jdbc_config   datasource
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/datebook?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&useSSL=false&zeroDateTimeBehavior=convertToNull
spring.datasource.username=root
spring.datasource.password=root
# Hikari will use the above plus the following to setup connection pooling
spring.datasource.type=com.zaxxer.hikari.HikariDataSource
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.maximum-pool-size=15
spring.datasource.hikari.auto-commit=true
spring.datasource.hikari.idle-timeout=30000
spring.datasource.hikari.pool-name=DatebookHikariCP
spring.datasource.hikari.max-lifetime=1800000
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.connection-test-query=SELECT 1
```

直接启动即可 如图

![img](http://static.iocoder.cn/mp/mmbiz_png/a5BAX19eYnWWL8hroXqE2yOj7UlRfG38PPtaiab8CKdwIf7A5Yjb3ibuctglvddBEveWsvvBaFvoN0a0LzM6ibrvw/640)

# 参考资料

https://blog.csdn.net/clementad/article/details/46928621

# 666. 彩蛋

如果你对 HikariCP 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)