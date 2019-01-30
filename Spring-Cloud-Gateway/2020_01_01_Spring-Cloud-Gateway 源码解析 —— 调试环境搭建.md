title: Spring-Cloud-Gateway 源码解析 —— 调试环境搭建
date: 2020-01-01
tags:
categories: Spring-Cloud-Gateway
permalink: Spring-Cloud-Gateway/build-debugging-environment

-------

**本文主要基于 Spring-Cloud-Gateway 2.0.X M4**  

摘要: 原创出处 http://www.iocoder.cn/Spring-Cloud-Gateway/build-debugging-environment/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 依赖工具](http://www.iocoder.cn/Spring-Cloud-Gateway/build-debugging-environment/)
- [2. 源码拉取](http://www.iocoder.cn/Spring-Cloud-Gateway/build-debugging-environment/)
- [3. 运行示例](http://www.iocoder.cn/Spring-Cloud-Gateway/build-debugging-environment/)
- [4. 彩蛋](http://www.iocoder.cn/Spring-Cloud-Gateway/build-debugging-environment/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2018_05_18.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

---

# 1. 依赖工具

* Maven ( `>= 3.3.3` )
* JDK
* IntelliJ IDEA

-------

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



# 2. 源码拉取

从官方仓库 [https://github.com/spring-cloud/spring-cloud-gateway.git](https://github.com/spring-cloud/spring-cloud-gateway.git) `Fork` 出属于自己的仓库。为什么要 `Fork` ？既然开始阅读、调试源码，我们可能会写一些注释，有了自己的仓库，可以进行自由的提交。😈

使用 `IntelliJ IDEA` 从 `Fork` 出来的仓库拉取代码。

**如果** `master` 分支处于 `1.x` 版本，请使用 `IntelliJ IDEA` 切换到 `2.0.X` 分支。

在项目路径下，在**命令行**执行 `mvn package -Dmaven.test.skip=true` 命令，`Maven` 会下载依赖包，可能会花费一些时间，耐心等待下。其间可能会出现因为网络原因( 我相信你懂的 )，可能会出现失败的情况，淡定，重新执行上述命令直到成功。此刻，你就是一个 `while(true)` 的小强。

执行完命令后，在 `IntelliJ IDEA` 的 `Maven Projects` 视图看到**部分依赖库处于报错状态**，将 `Profiles` 的 `spring` 勾选上，如下图所示：

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_01_01/01.png)

* 再在耐心等待 `Maven` 下载依赖库。
* 这步卡了很久，感谢 [知秋【基佬】](https://muyinchen.github.io) 的帮助。

# 3. 运行示例

`spring-cloud-gateway-sample` 子项目，官方示例。运行 GatewaySampleApplication 的 `#main()` 方法，启动示例。

运行成功日志如下 ：

```
2017-11-24 15:57:23.913  INFO 54587 --- [           main] o.s.b.web.embedded.netty.NettyWebServer  : Netty started on port(s): 8080
2017-11-24 15:57:23.915 DEBUG 54587 --- [ctor-http-nio-1] r.ipc.netty.http.server.HttpServer       : [id: 0xec536a1f, L:/0:0:0:0:0:0:0:0:8080] ACTIVE
2017-11-24 15:57:23.917  INFO 54587 --- [           main] o.s.c.g.sample.GatewaySampleApplication  : Started GatewaySampleApplication in 17.744 seconds (JVM running for 28.245)
```

使用浏览器访问 `http://127.0.0.1:8080/image/webp` 地址，我们会看到一张 **"狼"** 图。恭喜你，调试环境已经搭建完成。为什么会返回一张图片呢，答案在 `GatewaySampleApplication#customRouteLocator()` 方法的路由配置。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

胖友，分享一波朋友圈可好！

对了，这是一个系列文，所以，千万不要错过。

在送一波真彩蛋 ：

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_01_01/02.png)

