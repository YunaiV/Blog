title: SkyWalking 6.x 源码分析 —— 调试环境搭建
date: 2022-07-01
tags:
categories: SkyWalking
permalink: SkyWalking/6/build-debugging-environment

---

摘要: 原创出处 http://www.iocoder.cn/SkyWalking/6/build-debugging-environment/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 依赖工具](http://www.iocoder.cn/SkyWalking/6/build-debugging-environment/)
- [2. 源码拉取](http://www.iocoder.cn/SkyWalking/6/build-debugging-environment/)
- [3. 编译 SkyWalking](http://www.iocoder.cn/SkyWalking/6/build-debugging-environment/)
- [4. 启动 SkyWalking OAP Server](http://www.iocoder.cn/SkyWalking/6/build-debugging-environment/)
- [5. 启动 SkyWalking UI](http://www.iocoder.cn/SkyWalking/6/build-debugging-environment/)
- [6. 启动 SkyWalking Agent](http://www.iocoder.cn/SkyWalking/6/build-debugging-environment/)
- [666. 彩蛋](http://www.iocoder.cn/SkyWalking/6/build-debugging-environment/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# 1. 依赖工具

* Maven
* Git
* JDK 8

    > 艿艿：注意，**JDK 的版本一定要是 8** ，不然执行 `maven package` 命令时，会发生报错。 

* IntelliJ IDEA
* NPM

    > 艿艿：关于它的安装，胖友自己查询下噢。记得安装 [nvm](http://bubkoo.com/2017/01/08/quick-tip-multiple-versions-node-nvm/) 。

# 2. 源码拉取

从官方仓库 <https://github.com/apache/incubator-skywalking> `Fork` 出属于自己的仓库。为什么要 `Fork` ？既然开始阅读、调试源码，我们可能会写一些注释，有了自己的仓库，可以进行自由的提交。😈

使用 `IntelliJ IDEA` 从 `Fork` 出来的仓库拉取代码。拉取完成后，`Maven` 会下载依赖包，可能会花费一些时间，耐心等待下。

本文基于 `master` 分支，使用 SkyWalking `6.0.0-beta-SNAPSHOT` 版本。

# 3. 编译 SkyWalking

参考 [《官方文档 —— How to build》](https://github.com/apache/incubator-skywalking/blob/master/docs/en/guides/How-to-build.md)

1. 打开 IntelliJ IDEA Terminal 中，执行输入命令：
    1. `git submodule init` ，初始化子模块。
    2. `git submodule update` ，更新子模块。
    3. `mvn package -Dmaven.test.skip=true` ，进行编译。这个编译的过程中，依赖 [npm](https://www.npmjs.com/) 环境。一般情况下，我们不需要编译 SkyWalking UI 项目，所以我们可以把 `apm-webapp/pom.xml` 的如下部分，进行注释掉。

2. 设置 gRPC 的**自动生成**的代码目录，为**源码**目录 ：

    * 将 `apm-protocol/apm-network/target/generated-sources/protobuf` 目录下面`grpc-java` 和 `java` 目录**右键**设置为 `Generated Rources Root` 。![](http://www.iocoder.cn/images/SkyWalking/2022_07_01/01.png)
    * 将 `oap-server/server-core/target/generated-sources/protobuf` 目录下面`grpc-java` 和 `java` 目录**右键**设置为 `Generated Rources Root` 。
    * 将 `oap-server/server-receiver-plugin/skywalking-istio-telemetry-receiver-plugin/target/generated-sources/protobuf` 目录下面`grpc-java` 和 `java` 目录**右键**设置为 `Generated Rources Root` 。

# 4. 启动 SkyWalking OAP Server

1. 安装 ElasticSearch 6.x 因为 SkyWalking 的 Tracing 的数据存储在它上面。具体的安全过程，胖友看看 [《ElasticSearch 6.x 学习笔记：1.下载安装与配置》](org.apache.skywalking.oap.server.starter) 。

2. 运行 `oap-server` 的 `server-starter` 的  `org.apache.skywalking.oap.server.starter.OAPServerStartUp` 的 `#main(args)` 方法，启动 SkyWalking OAP Server 。

3. 启动成功日志。

    ```Java
    2018-11-19 17:12:26,331 - org.eclipse.jetty.server.Server - 71 [main] INFO  [] - Started @5389ms
    ```

# 5. 启动 SkyWalking UI

1. 运行 `apm-webapp` 的  `org.apache.skywalking.apm.webapp.ApplicationStartUp` 的 `#main(args)` 方法，启动 SkyWalking UI 。
2. 浏览器打开 `http://127.0.0.1:8080` ，输入账号密码 `admin / admin` 进行登录。![SkyWalking UI 首页](http://www.iocoder.cn/images/SkyWalking/2022_07_01/03.png)

# 6. 启动 SkyWalking Agent

1. 在 `skywalking-agent` 目录下，我们可以看到编译出来的 `skywalking-agent.jar` ：![skywalking-agent.jar 包](http://www.iocoder.cn/images/SkyWalking/2022_07_01/02.png)
2. 使用 Spring Boot 创建一个简单的 Web 项目，注意端口不要使用 8080 ，因为 SkyWalking UI 使用了 8080 端口。类似如下 ：![示例](http://www.iocoder.cn/images/SkyWalking/2020_07_01/03.png)

    > 友情提示 ：**这里一定要注意下**。创建的 Web 项目，使用  IntelliJ IDEA 的**菜单** File / New / Module 或 File / New / Module from Existing Sources ，**保证 Web 项目和 SkyWalking 项目平级**。这样，才可以使用 IntelliJ IDEA 调试  Agent 。

    * ![](http://www.iocoder.cn/images/SkyWalking/2020_07_01/04.png)
    * ![](http://www.iocoder.cn/images/SkyWalking/2020_07_01/05.png)

3. 在 `org.skywalking.apm.agent.SkyWalkingAgent` 的 `#premain(...)` 方法，打上调试断点。
4. 运行 Web 项目的 Application 的 `#main(args)` 方法，并增加 JVM 启动参数，`-javaagent:/path/to/skywalking-agent/skywalking-agent.jar`。`/path/to` **参数值**为上面我们编译出来的 /packages/skywalking-agent 目录的绝对路径。如下图 ：![](http://www.iocoder.cn/images/SkyWalking/2020_07_01/06.png)
5. 如果在【**第三步**】的调试断点停住，说明 Agent 启动**成功**。

-------

考虑到可能我们会在 Agent 上增加代码注释，这样每次不得不重新编译 Agent 。可以配置如下图，自动编译 Agent ：![](http://www.iocoder.cn/images/SkyWalking/2020_07_01/07.png)

* `-T 1C clean  package -Dmaven.test.skip=true -Dmaven.compile.fork=true` 。

-------

另外，使用 IntelliJ IDEA Remote 远程调试，也是可以的。如下图 ：![](http://www.iocoder.cn/images/SkyWalking/2020_07_01/10.png)

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

SkyWalking 环境搭建完成，胖友们可以起飞，源码读起来。

如下是笔者在 SkyWalking **3.x** 对代码量和用途的简单整理，完善度比较低，可能有一丢丢的帮助：

![](http://www.iocoder.cn/images/SkyWalking/2020_07_01/09.png)

因为准备重新读 SkyWalking **6.x** 的代码，所以又简单的整理了下，这次的完成度更低，哈哈哈哈：

![](http://www.iocoder.cn/images/SkyWalking/2022_07_01/04.png)

-------

另外，超级推荐看看胖友在录制的 SkyWalking 的视频，快来点击 [传送门](http://www.iocoder.cn/SkyWalking/videos/) 。