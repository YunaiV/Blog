title: SkyWalking 源码分析 —— 调试环境搭建
date: 2020-07-01
tags:
categories: SkyWalking
permalink: SkyWalking/build-debugging-environment
wechat_url:
toutiao_url:

---

摘要: 原创出处 http://www.iocoder.cn/SkyWalking/build-debugging-environment/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 SkyWalking 3.2.6 正式版**

- [1. 依赖工具](http://www.iocoder.cn/SkyWalking/build-debugging-environment/)
- [2. 源码拉取](http://www.iocoder.cn/SkyWalking/build-debugging-environment/)
- [3. 启动 SkyWalking Collector](http://www.iocoder.cn/SkyWalking/build-debugging-environment/)
- [4. 启动 SkyWalking Agent](http://www.iocoder.cn/SkyWalking/build-debugging-environment/)
- [5. 启动 SkyWalking Web UI](http://www.iocoder.cn/SkyWalking/build-debugging-environment/)
- [6. 彩蛋](http://www.iocoder.cn/SkyWalking/build-debugging-environment/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2018_05_18.jpg)

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
* JDK
* IntelliJ IDEA

# 2. 源码拉取

从官方仓库 [https://github.com/OpenSkywalking/skywalking](https://github.com/OpenSkywalking/skywalking) `Fork` 出属于自己的仓库。为什么要 `Fork` ？既然开始阅读、调试源码，我们可能会写一些注释，有了自己的仓库，可以进行自由的提交。😈

使用 `IntelliJ IDEA` 从 `Fork` 出来的仓库拉取代码。拉取完成后，`Maven` 会下载依赖包，可能会花费一些时间，耐心等待下。

本文基于 `master` 分支。

# 3. 启动 SkyWalking Collector

参考 [《官方文档 —— How to build》](https://github.com/OpenSkywalking/skywalking/wiki/How-to-build)

1. 在 IntelliJ IDEA Terminal 中，执行 `mvn compile -Dmaven.test.skip=true` 进行编译。
2. 设置 gRPC 的**自动生成**的代码目录，为**源码**目录 ：

    * /apm-network/target/generated-sources/protobuf/ 下的 `grpc-java` 和 `java` 目录
    * /apm-collector-remote/collector-remote-grpc-provider/target/generated-sources/protobuf/ 下的 `grpc-java` 和 `java` 目录
    * ![](http://www.iocoder.cn/images/SkyWalking/2020_07_01/01.png)

    > 从 3.2 开始，网络通讯协议引入 GRPC ，所以增加上述的步骤

3. 运行 `org.skywalking.apm.collector.bootCollector.BootStartUp` 的 `#main(args)` 方法，启动 Collector 。
4. 访问 `http://127.0.0.1:10800/agent/jetty` 地址，返回 `["localhost:12800/"]` ，说明启动**成功**。

# 4. 启动 SkyWalking Agent

感谢 [吴晟](https://github.com/wu-sheng) 指导如何搭建 Agent 调试环境。

1. 在 IntelliJ IDEA Terminal 中，执行 `mvn compile -Dmaven.test.skip=true` 进行编译。在 /packages/skywalking-agent 目录下，我们可以看到编译出来的 Agent ：![](http://www.iocoder.cn/images/SkyWalking/2020_07_01/02.png)
2. 使用 Spring Boot 创建一个简单的 Web 项目。类似如下 ：![](http://www.iocoder.cn/images/SkyWalking/2020_07_01/03.png)

    > 友情提示 ：**这里一定要注意下**。创建的 Web 项目，使用  IntelliJ IDEA 的**菜单** File / New / Module 或 File / New / Module from Existing Sources ，**保证 Web 项目和 skywalking 项目平级**。这样，才可以使用 IntelliJ IDEA 调试  Agent 。

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

# 5. 启动 SkyWalking Web UI

考虑到调试过程中，我们要看下是否收集到追踪日志，可以安装 SkyWalking Web UI 进行查看。

参考 [《官方文档 —— Sky Walking Web UI》](https://github.com/OpenSkywalking/skywalking-ui#quickstart-zh) 安装。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

SkyWalking 环境搭建完成，胖友们可以起飞，源码读起来。

这会是个系列文章，笔者会慢慢更新。

![](http://www.iocoder.cn/images/SkyWalking/2020_07_01/08.png)

如下是笔者对代码量和用途的简单整理，完善度比较低，可能有一丢丢的帮助 ：

![](http://www.iocoder.cn/images/SkyWalking/2020_07_01/09.png)


