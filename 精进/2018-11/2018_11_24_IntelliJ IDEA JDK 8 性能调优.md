title: IntelliJ IDEA JDK 8 性能调优
date: 2018-11-24
tags: IntelliJ IDEA
categories: 精进
permalink: Fight/IntelliJ-IDEA-JDK-8-performance-tuning
author: 小马哥
from_url: https://segmentfault.com/a/1190000010144588
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247485718&idx=2&sn=a1b0c9fdcaa78d2353a66ff4e23ebb89&chksm=fa4976a7cd3effb1c1a8a60eb3b49a025d753685dbe0feda97853e1e023f7bc3f5716bcad020&token=1908850988&lang=zh_CN#rd

-------

摘要: 原创出处 https://segmentfault.com/a/1190000010144588 「小马哥」欢迎转载，保留摘要，谢谢！

- [IntelliJ IDEA 问题描述](http://www.iocoder.cn/Fight/IntelliJ-IDEA-JDK-8-performance-tuning/)
  - [MacBook Pro 配置](http://www.iocoder.cn/Fight/IntelliJ-IDEA-JDK-8-performance-tuning/)
  - [IntelliJ IDEA 版本](http://www.iocoder.cn/Fight/IntelliJ-IDEA-JDK-8-performance-tuning/)
- [问题原因](http://www.iocoder.cn/Fight/IntelliJ-IDEA-JDK-8-performance-tuning/)
- [解决方法](http://www.iocoder.cn/Fight/IntelliJ-IDEA-JDK-8-performance-tuning/)
  - [修改`Info.plist`文件](http://www.iocoder.cn/Fight/IntelliJ-IDEA-JDK-8-performance-tuning/)
  - [修改用户`idea.vmoptions`文件](http://www.iocoder.cn/Fight/IntelliJ-IDEA-JDK-8-performance-tuning/)
- [调优后观察](http://www.iocoder.cn/Fight/IntelliJ-IDEA-JDK-8-performance-tuning/)
  - [风扇旋转情况](http://www.iocoder.cn/Fight/IntelliJ-IDEA-JDK-8-performance-tuning/)
  - [JVM 概要情况](http://www.iocoder.cn/Fight/IntelliJ-IDEA-JDK-8-performance-tuning/)
  - [JVM 参数情况](http://www.iocoder.cn/Fight/IntelliJ-IDEA-JDK-8-performance-tuning/)
  - [JVM 内存情况](http://www.iocoder.cn/Fight/IntelliJ-IDEA-JDK-8-performance-tuning/)
- [为什么要选择用户`idea.vmoptions`文件](http://www.iocoder.cn/Fight/IntelliJ-IDEA-JDK-8-performance-tuning/)
  - [IDEA 官方的说明](http://www.iocoder.cn/Fight/IntelliJ-IDEA-JDK-8-performance-tuning/)
  - [避免升级配置覆盖](http://www.iocoder.cn/Fight/IntelliJ-IDEA-JDK-8-performance-tuning/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# IntelliJ IDEA 问题描述

IntelliJ IDEA 在 多窗口、多项目协作开发时，MacBook Pro的散热风扇凶猛地转动，相关配置如下：

## MacBook Pro 配置

```
MacBook Pro (Retina, 15-inch, Mid 2015)
型号名称：    MacBook Pro
型号标识符：    MacBookPro11,4
处理器名称：    Intel Core i7
处理器速度：    2.2 GHz
处理器数目：    1
核总数：    4
L2 缓存（每个核）：    256 KB
L3 缓存：    6 MB
内存：    16 GB
Boot ROM 版本：    MBP114.0172.B16
SMC 版本（系统）：    2.29f24
```

## IntelliJ IDEA 版本

```
IntelliJ IDEA 2017.1.5
Build #IC-171.4694.70, built on July 4, 2017
JRE: 1.8.0_131-b11 x86_64
JVM: Java HotSpot(TM) 64-Bit Server VM by Oracle Corporation
Mac OS X 10.12.5
```

# 问题原因

默认的IDEA JVM参数配置较低，其中配置存放在 `/Applications/IntelliJ IDEA CE.app/Contents/bin/idea.vmoptions`文件中，该文件为IDEA 全局配置文件：

```
-Xms128m
-Xmx750m
-XX:ReservedCodeCacheSize=240m
```

# 解决方法

## 修改`Info.plist`文件

### 定位`Info.plist`文件

该文件存放在`/Applications/IntelliJ IDEA CE.app/Contents` 目录下：

```
total 32
16 -rw-r--r--    1 Mercy  admin  4210  7 11 18:43 Info.plist
 0 drwxr-xr-x@   3 Mercy  admin   102  7 11 16:21 MacOS
 0 drwxr-xr-x@   7 Mercy  admin   238  7  5 14:06 Resources
 0 drwxr-xr-x@   3 Mercy  admin   102  7  5 14:06 _CodeSignature
 0 drwxr-xr-x@  13 Mercy  admin   442  7 11 18:00 bin
 0 drwxr-xr-x@ 116 Mercy  admin  3944  7  5 14:06 lib
 0 drwxr-xr-x@  34 Mercy  admin  1156  4 25 15:49 license
 0 drwxr-xr-x@  33 Mercy  admin  1122  4 25 15:49 plugins
 0 drwxr-xr-x@   3 Mercy  admin   102  7  5 14:06 redist
```

### 修改`VMOptions`

用 vi 工具打开`Info.plist` ，其中存在一个 key 元素内容为`VMOptions`的设置，如下所示：

```
<key>VMOptions</key>
<string>-Dfile.encoding=UTF-8 -XX:+UseConcMarkSweepGC -XX:SoftRefLRUPolicyMSPerMB=50 -ea -Dsun.io.useCanonCaches=false -Djava.net.preferIPv4Stack=true -XX:+HeapDumpOnOutOfMemoryError -XX:-OmitStackTraceInFastThrow -Xverify:none -XX:ErrorFile=$USER_HOME/java_error_in_idea_%p.log -XX:HeapDumpPath=$USER_HOME/java_error_in_idea.hprof -Xbootclasspath/a:../lib/boot.jar</string>
```

其中JVM 参数 `-XX:+UseConcMarkSweepGC` 为 IDEA 默认配置GC 算法，将其移除，修改为：

```
<key>VMOptions</key>
<string>-Dfile.encoding=UTF-8 -XX:SoftRefLRUPolicyMSPerMB=50 -ea -Dsun.io.useCanonCaches=false -Djava.net.preferIPv4Stack=true -XX:+HeapDumpOnOutOfMemoryError -XX:-OmitStackTraceInFastThrow -Xverify:none -XX:ErrorFile=$USER_HOME/java_error_in_idea_%p.log -XX:HeapDumpPath=$USER_HOME/java_error_in_idea.hprof -Xbootclasspath/a:../lib/boot.jar</string>
```

## 修改用户`idea.vmoptions`文件

### 切换当前用户的IDEA 配置目录

通过命令行，cd到`~/Library/Preferences/IntelliJIdeaXX/`目录下，如本人的机器路径：

```
/Users/Mercy/Library/Preferences/IdeaIC2017.1
```

### 新建或更新用户`idea.vmoptions`文件

将新建或者待更新的`idea.vmoptions`文件，更新以下JVM 配置项

```
-server
-XX:+UseG1GC
-XX:+UseNUMA
-Xms512m
-Xmn512m
-Xmx8g
-XX:MaxMetaspaceSize=512m
-XX:ReservedCodeCacheSize=240m
```

# 调优后观察

## 风扇旋转情况

启动 IntelliJ IDEA 2017.1 后一小时有余，发现风扇狂转的问题基本上没有发生。

## JVM 概要情况

通过工具`JConsole` 连接 IDEA 进程，观察相关数据。

```
连接名称: pid: 9743                                        运行时间: 1 小时 39 分钟
虚拟机: Java HotSpot(TM) 64-Bit Server VM版本 25.131-b11   进程 CPU 时间: 6 分钟
```

## JVM 参数情况

```
VM 参数:-Dfile.encoding=UTF-8 -XX:SoftRefLRUPolicyMSPerMB=50 -ea -Dsun.io.useCanonCaches=false -Djava.net.preferIPv4Stack=true
-XX:+HeapDumpOnOutOfMemoryError -XX:-OmitStackTraceInFastThrow -Xverify:none
-XX:ErrorFile=/Users/Mercy/java_error_in_idea_%p.log -XX:HeapDumpPath=/Users/Mercy/java_error_in_idea.hprof -Xbootclasspath/a:../lib/boot.jar
-server -XX:+UseG1GC -XX:+UseNUMA -Xms512m -Xmx8g -XX:MaxMetaspaceSize=512m -XX:ReservedCodeCacheSize=240m
-Djb.vmOptionsFile=/Users/Mercy/Library/Preferences/IdeaIC2017.1/idea.vmoptions -Didea.java.redist=jdk-bundled
-Didea.home.path=/Applications/IntelliJ IDEA CE.app/Contents -Didea.executable=idea -Didea.platform.prefix=Idea -Didea.paths.selector=IdeaIC2017.1
```

其中用户`idea.vmoptions`文件中的配置信息已经追加到JVM 启动参数中：

```
-server -XX:+UseG1GC -XX:+UseNUMA -Xms512m -Xmx8g -XX:MaxMetaspaceSize=512m -XX:ReservedCodeCacheSize=240m
```

## JVM 内存情况

```
当前堆大小:   376,068 KB
最大堆大小: 8,388,608 KB
提交的内存: 524,288 KB
暂挂最终处理: 0对象
垃圾收集器: 名称 = 'G1 Young Generation', 收集 = 58, 总花费时间 = 1.583 秒
垃圾收集器: 名称 = 'G1 Old Generation', 收集 = 2, 总花费时间 = 1.930 秒
```

GC 算法已经由`CMS`切换成了`G1`算法！

# 为什么要选择用户`idea.vmoptions`文件

## IDEA 官方的说明

```
Since version 14.0.0, the file /Applications/IntelliJ Idea XX.app/Contents/bin/idea.vmoptions or /Applications/IntelliJ Idea CE XX.app/Contents/bin/idea.vmoptions should be copied to ~/Library/Preferences/IntelliJIdeaXX/idea.vmoptions or ~/Library/Preferences/IdeaICXX/idea.vmoptions.
```

## 避免升级配置覆盖

IntelliJ IDEA 版本升级时，除非用户自行控制，默认情况IDEA会将全局的`idea.vmoptions`文件覆盖，因此，选择用户的`idea.vmoptions`文件可避免升级配置覆盖。