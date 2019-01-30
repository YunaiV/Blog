title: Log4j1、Logback 以及 Log4j2 性能测试对
date: 2019-01-07
tags:
categories: 精进
permalink: Fight/Log4j1-and-Logback-and-Log4j2-performance-test
author: ksfzhaohui
from_url: https://my.oschina.net/OutOfMemory/blog/789267
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247486008&idx=1&sn=845c2d802ab7c8bac1007a9157f0ff8e&chksm=fa497589cd3efc9f3cb86960c9f0344f4e82f19981eb6e5168e54236602878ea65fb2d65b109&token=933039983&lang=zh_CN#rd

-------

摘要: 原创出处 https://my.oschina.net/OutOfMemory/blog/789267 「ksfzhaohui」欢迎转载，保留摘要，谢谢！

- [环境](http://www.iocoder.cn/Fight/Log4j1-and-Logback-and-Log4j2-performance-test/)
- [准备](http://www.iocoder.cn/Fight/Log4j1-and-Logback-and-Log4j2-performance-test/)
- [测试](http://www.iocoder.cn/Fight/Log4j1-and-Logback-and-Log4j2-performance-test/)
- [结果](http://www.iocoder.cn/Fight/Log4j1-and-Logback-and-Log4j2-performance-test/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# 环境

jdk：1.7.0_79
cpu：i5-4570@3.20GHz 4核
eclipse：3.7
操作系统：win7

# 准备

**1.log4j:1.7.21**

```XML
<dependency>
        <groupId>org.slf4j</groupId>
    <artifactId>slf4j-log4j12</artifactId>
    <version>1.7.21</version>
</dependency>
```

**log4j.xml**

```XML
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE log4j:configuration SYSTEM "log4j.dtd">
<log4j:configuration xmlns:log4j='http://jakarta.apache.org/log4j/'>

    <appender name="myConsole" class="org.apache.log4j.ConsoleAppender">
        <layout class="org.apache.log4j.PatternLayout">
            <param name="ConversionPattern" value="[%d{dd HH:mm:ss,SSS\} %-5p] [%t] %c{2\} - %m%n" />
        </layout>

        <!--过滤器设置输出的级别 -->
        <filter class="org.apache.log4j.varia.LevelRangeFilter">
            <param name="levelMin" value="debug" />
            <param name="levelMax" value="warn" />
            <param name="AcceptOnMatch" value="true" />
        </filter>
    </appender>

    <appender name="myFile" class="org.apache.log4j.DailyRollingFileAppender">
        <param name="File" value="log4jTest.log" />
        <param name="Append" value="true" />
        <param name="DatePattern" value="'.'yyyy-MM-dd'.log'" />
        <layout class="org.apache.log4j.PatternLayout">
            <param name="ConversionPattern" value="[%t] - %m%n" />
        </layout>
    </appender>

    <appender name="async_file" class="org.apache.log4j.AsyncAppender">
        <param name="BufferSize" value="32" />
        <appender-ref ref="myFile" />
    </appender>

    <logger name="org.logTest" additivity="false">
        <level value="info" />
        <appender-ref ref="async_file" /> <!-- 同步:FILE 异步:async_file -->
    </logger>

</log4j:configuration>
```

**2.logback:1.1.7**

```XML
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>1.1.7</version>
</dependency>
```

**logback.xml**

```XML
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <!-- encoder 默认配置为PatternLayoutEncoder -->
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n
            </pattern>
        </encoder>
    </appender>

    <appender name="FILE" class="ch.qos.logback.core.FileAppender">
        <file>testFile.log</file>
        <append>true</append>
        <encoder>
            <pattern>[%t] - %m%n
            </pattern>
        </encoder>
    </appender>

    <!-- 异步输出 -->
    <appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
        <discardingThreshold>0</discardingThreshold>
        <appender-ref ref="FILE" />
    </appender>

    <logger name="org.logTest" level="INFO"
        additivity="false">
        <appender-ref ref="ASYNC" />  <!-- 同步:FILE 异步:ASYNC-->
    </logger>

    <root level="ERROR">
        <appender-ref ref="STDOUT" />
    </root>
</configuration>
```

**3.log4j2:2.6.2**

```XML
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <version>2.6.2</version>
</dependency>
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-slf4j-impl</artifactId>
    <version>2.6.2</version>
</dependency>
<dependency>
    <groupId>com.lmax</groupId>
    <artifactId>disruptor</artifactId>
    <version>3.3.4</version>
</dependency>
```

**log4j2.xml**

```XML
<?xml version="1.0" encoding="UTF-8"?>
<!--设置log4j2的自身log级别为warn -->
<configuration status="warn">

    <appenders>
         <console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="[%d{HH:mm:ss:SSS}] [%p] - %l - %m%n" />
        </console>

        <RollingFile name="RollingFileInfo" fileName="info.log"
            filePattern="${sys:user.home}/logs/hpaasvc/$${date:yyyy-MM}/info-%d{yyyy-MM-dd}-%i.log">
            <Filters>
                <ThresholdFilter level="INFO" />
                <ThresholdFilter level="WARN" onMatch="DENY"
                    onMismatch="NEUTRAL" />
            </Filters>
            <PatternLayout pattern="[%t] - %m%n" />
            <Policies>
                <TimeBasedTriggeringPolicy />
                <SizeBasedTriggeringPolicy size="100 MB" />
            </Policies>
        </RollingFile>

        <RandomAccessFile name="RandomAccessFile" fileName="asyncWithLocation.log"
            immediateFlush="false" append="true">
            <PatternLayout>
                <Pattern>[%t] - %m%n</Pattern>
            </PatternLayout>
        </RandomAccessFile>

    </appenders>

    <loggers>
        <!-- <AsyncLogger name="asynLogger" level="trace"
            includeLocation="true">
            <AppenderRef ref="RandomAccessFile" />
        </AsyncLogger> -->
        <Root level="info" includeLocation="true">
            <AppenderRef ref="RollingFileInfo" />
        </Root>
    </loggers>

</configuration>
```

# 测试

准备50条线程同时记录1000000条数据，然后统计时间，详细代码如下：

```Java
import java.util.concurrent.CountDownLatch;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class App {
    private static Logger log = LoggerFactory.getLogger(App.class);

    public static void main(String[] args) throws InterruptedException {
        int messageSize = 1000000;
        int threadSize = 50;
        final int everySize = messageSize / threadSize;

        final CountDownLatch cdl = new CountDownLatch(threadSize);
        long startTime = System.currentTimeMillis();
        for (int ts = 0; ts < threadSize; ts++) {
            new Thread(new Runnable() {

                @Override
                public void run() {
                    for (int es = 0; es < everySize; es++) {
                        log.info("======info");
                    }
                    cdl.countDown();
                }
            }).start();
        }

        cdl.await();
        long endTime = System.currentTimeMillis();
        System.out.println("log4j1:messageSize = " + messageSize
                + ",threadSize = " + threadSize + ",costTime = "
                + (endTime - startTime) + "ms");
    }
}
```

log4j1和logback的同步和异步分别修改为对应的appender就行了
log4j2的异步方式提供了2中模式：
1.全局开启
设置Log4jContextSelector系统属性为：
org.apache.logging.log4j.core.async.AsyncLoggerContextSelector

```Java
System.setProperty("Log4jContextSelector", "org.apache.logging.log4j.core.async.AsyncLoggerContextSelector");
```

2.混合同步异步模式
不需要设置Log4jContextSelector，但是需要使用AsyncLogger标签

更多详细参考官方文档：<http://logging.apache.org/log4j/2.x/manual/async.html#AllAsync>

# 结果

分别测试完以后统计成表格如下：

![img](http://static.iocoder.cn/292d1cdd843f81437199abf6cbb0c64b)

log4j2的异步模式表现了绝对的性能优势，优势主要得益于Disruptor框架的使用

```
LMAX Disruptor technology. Asynchronous Loggers internally use the Disruptor, a lock-free inter-thread communication library, instead of queues, resulting in higher throughput and lower latency.
```

一个无锁的线程间通信库代替了原来的队列

更多Disruptor ：

<http://developer.51cto.com/art/201306/399370.htm>
<http://ifeve.com/disruptor/>
