title: RocketMQ 入门 —— 基于 Spring 使用 RocketMQ 以及监控
date: 2020-01-10
tags:
categories: RocketMQ
permalink: RocketMQ/start/spring-example-and-monitor
author: zhisheng
from_url: http://www.54tianzhisheng.cn/2018/02/07/SpringBoot-RocketMQ/

-------

摘要: 原创出处 http://www.54tianzhisheng.cn/2018/02/07/SpringBoot-RocketMQ/ 「zhisheng」欢迎转载，保留摘要，谢谢！

- [前提](http://www.iocoder.cn/RocketMQ/start/spring-example-and-monitor/)
- [创建项目](http://www.iocoder.cn/RocketMQ/start/spring-example-and-monitor/)
- [RocketMQ](http://www.iocoder.cn/RocketMQ/start/spring-example-and-monitor/)
- [监控](http://www.iocoder.cn/RocketMQ/start/spring-example-and-monitor/)
- [错误解决方法](http://www.iocoder.cn/RocketMQ/start/spring-example-and-monitor/)
- [总结](http://www.iocoder.cn/RocketMQ/start/spring-example-and-monitor/)
- [参考文章](http://www.iocoder.cn/RocketMQ/start/spring-example-and-monitor/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# 前提

通过前面两篇文章可以简单的了解 RocketMQ 和 安装 RocketMQ ，今天就将 SpringBoot 和 RocketMQ 整合起来使用。

# 创建项目

在 IDEA 创建一个 SpringBoot 项目，项目结构如下：

![rocketmq01](https://ws4.sinaimg.cn/large/006tNc79gy1fo8vq6ufg3j30pe0yoacc.jpg)

## pom 文件

引入 RocketMQ 的一些相关依赖，最后的 pom 文件如下：

```XML
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.zhisheng</groupId>
	<artifactId>rocketmq</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>rocketmq</name>
	<description>Demo project for Spring Boot RocketMQ</description>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.9.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>

		<dependency>
			<groupId>org.apache.rocketmq</groupId>
			<artifactId>rocketmq-common</artifactId>
			<version>4.2.0</version>
		</dependency>

		<dependency>
			<groupId>org.apache.rocketmq</groupId>
			<artifactId>rocketmq-client</artifactId>
			<version>4.2.0</version>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
</project>
```

## 配置文件

application.properties 中如下：

```Properties
# 消费者的组名
apache.rocketmq.consumer.PushConsumer=PushConsumer
# 生产者的组名
apache.rocketmq.producer.producerGroup=Producer
# NameServer地址
apache.rocketmq.namesrvAddr=localhost:9876
```

## 生产者

```Java
package com.zhisheng.rocketmq.client;

import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.common.message.Message;
import org.apache.rocketmq.remoting.common.RemotingHelper;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import org.springframework.util.StopWatch;

import javax.annotation.PostConstruct;

/**
 * Created by zhisheng_tian on 2018/2/6
 */
@Component
public class RocketMQClient {
    /**
     * 生产者的组名
     */
    @Value("${apache.rocketmq.producer.producerGroup}")
    private String producerGroup;

    /**
     * NameServer 地址
     */
    @Value("${apache.rocketmq.namesrvAddr}")
    private String namesrvAddr;

    @PostConstruct
    public void defaultMQProducer() {
        //生产者的组名
        DefaultMQProducer producer = new DefaultMQProducer(producerGroup);
        //指定NameServer地址，多个地址以 ; 隔开
        producer.setNamesrvAddr(namesrvAddr);

        try {
            /**
             * Producer对象在使用之前必须要调用start初始化，初始化一次即可
             * 注意：切记不可以在每次发送消息时，都调用start方法
             */
            producer.start();

          	 //创建一个消息实例，包含 topic、tag 和 消息体
             //如下：topic 为 "TopicTest"，tag 为 "push"
            Message message = new Message("TopicTest", "push", "发送消息----zhisheng-----".getBytes(RemotingHelper.DEFAULT_CHARSET));

            StopWatch stop = new StopWatch();
            stop.start();

            for (int i = 0; i < 10000; i++) {
                SendResult result = producer.send(message);
                System.out.println("发送响应：MsgId:" + result.getMsgId() + "，发送状态:" + result.getSendStatus());
            }
            stop.stop();
            System.out.println("----------------发送一万条消息耗时：" + stop.getTotalTimeMillis());
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            producer.shutdown();
        }
    }
}
```

## 消费者

```Java
package com.zhisheng.rocketmq.server;

import org.apache.rocketmq.client.consumer.DefaultMQPushConsumer;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyStatus;
import org.apache.rocketmq.client.consumer.listener.MessageListenerConcurrently;
import org.apache.rocketmq.common.consumer.ConsumeFromWhere;
import org.apache.rocketmq.common.message.MessageExt;
import org.apache.rocketmq.remoting.common.RemotingHelper;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;

/**
 * Created by zhisheng_tian on 2018/2/6
 */
@Component
public class RocketMQServer {
    /**
     * 消费者的组名
     */
    @Value("${apache.rocketmq.consumer.PushConsumer}")
    private String consumerGroup;

    /**
     * NameServer 地址
     */
    @Value("${apache.rocketmq.namesrvAddr}")
    private String namesrvAddr;

    @PostConstruct
    public void defaultMQPushConsumer() {
        //消费者的组名
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer(consumerGroup);

        //指定NameServer地址，多个地址以 ; 隔开
        consumer.setNamesrvAddr(namesrvAddr);
        try {
            //订阅PushTopic下Tag为push的消息
            consumer.subscribe("TopicTest", "push");

            //设置Consumer第一次启动是从队列头部开始消费还是队列尾部开始消费
            //如果非第一次启动，那么按照上次消费的位置继续消费
            consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
            consumer.registerMessageListener((MessageListenerConcurrently) (list, context) -> {
                try {
                    for (MessageExt messageExt : list) {

                        System.out.println("messageExt: " + messageExt);//输出消息内容

                        String messageBody = new String(messageExt.getBody(), RemotingHelper.DEFAULT_CHARSET);

                        System.out.println("消费响应：msgId : " + messageExt.getMsgId() + ",  msgBody : " + messageBody);//输出消息内容
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                    return ConsumeConcurrentlyStatus.RECONSUME_LATER; //稍后再试
                }
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS; //消费成功
            });
            consumer.start();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

## 启动类

```Java
package com.zhisheng.rocketmq;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class RocketmqApplication {

	public static void main(String[] args) {
		SpringApplication.run(RocketmqApplication.class, args);
	}
}
```

# RocketMQ

代码已经都写好了，接下来我们需要将与 RocketMQ 有关的启动起来。

## 启动 Name Server

在前面文章中已经写过怎么启动，<http://www.54tianzhisheng.cn/2018/02/06/RocketMQ-install/#%E5%90%AF%E5%8A%A8-NameServer>

进入到目录 ：

```Bash
cd distribution/target/apache-rocketmq
```

启动：

```Bash
nohup sh bin/mqnamesrv &

tail -f ~/logs/rocketmqlogs/namesrv.log //通过日志查看是否启动成功
```

## 启动 Broker

```Bash
nohup sh bin/mqbroker -n localhost:9876 &

tail -f ~/logs/rocketmqlogs/broker.log	//通过日志查看是否启动成功
```

然后运行启动类，运行效果如下：

![2018-02-07_22-21-14](https://ws4.sinaimg.cn/large/006tNc79gy1fo8vqiq0jvj31kw0x1x6m.jpg)

# 监控

RocketMQ有一个对其扩展的开源项目 ocketmq-console ，如今也提交给了 Apache ，地址在：[https://github.com/apache/rocketmq-externals/tree/master/rocketmq-console](http://www.54tianzhisheng.cn/2018/02/07/SpringBoot-RocketMQ/) ，官方也给出了其支持的功能的中文文档：[https://github.com/apache/rocketmq-externals/blob/master/rocketmq-console/doc/1_0_0/UserGuide_CN.md](http://www.54tianzhisheng.cn/2018/02/07/SpringBoot-RocketMQ/) ， 那么该如何安装？

## Docker 安装

1、获取 Docker 镜像

```Bash
docker pull styletang/rocketmq-console-ng
```

2、运行，注意将你自己的 NameServer 地址替换下面的 127.0.0.1

```Bash
docker run -e "JAVA_OPTS=-Drocketmq.namesrv.addr=127.0.0.1:9876 -Dcom.rocketmq.sendMessageWithVIPChannel=false" -p 8080:8080 -t styletang/rocketmq-console-ng
```

## 非 Docker 安装

我们 git clone 一份代码到本地：

```
git clone https://github.com/apache/rocketmq-externals.git

cd rocketmq-externals/rocketmq-console/
```

需要 jdk 1.7 以上。 执行以下命令：

```
mvn spring-boot:run
```

或者

```
mvn clean package -Dmaven.test.skip=true

java -jar target/rocketmq-console-ng-1.0.0.jar
```

**注意：**

1、如果你下载依赖缓慢，你可以重新设置 maven 的 mirror 为阿里云的镜像

```
<mirrors>
    <mirror>
          <id>alimaven</id>
          <name>aliyun maven</name>
          <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
          <mirrorOf>central</mirrorOf>        
    </mirror>
</mirrors>
```

2、如果你使用的 RocketMQ 版本小于 3.5.8，如果您使用 rocketmq < 3.5.8，请在启动 rocketmq-console-ng 时添加 `-Dcom.rocketmq.sendMessageWithVIPChannel = false`（或者您可以在 ops 页面中更改它）

3、更改 resource / application.properties 中的 rocketmq.config.namesrvAddr（或者可以在ops页面中更改它）

# 错误解决方法

1、Docker 启动项目报错

```Bash
org.apache.rocketmq.remoting.exception.RemotingConnectException: connect to <null> failed
```

![2018-02-07_23-28-51](https://ws4.sinaimg.cn/large/006tNc79gy1fo8vqhpxcrj31kw0qc4ln.jpg)

将 Docker 启动命令改成如下以后：

```Bash
docker run -e "JAVA_OPTS=-Drocketmq.config.namesrvAddr=127.0.0.1:9876 -Drocketmq.config.isVIPChannel=false" -p 8080:8080 -t styletang/rocketmq-console-ng
```

报错信息改变了，新的报错信息如下：

```
ERROR op=global_exception_handler_print_error

org.apache.rocketmq.console.exception.ServiceException: This date have't data!
```

![2018-02-08_00-23-15](https://ws1.sinaimg.cn/large/006tNc79gy1fo8vql4zvij31kw0vz1kx.jpg)

看到网上有人也遇到这个问题，他们都通过自己的方式解决了，但是方法我都试了，不适合我。不得不说，阿里，你能再用心点吗？既然把 RocketMQ 捐给 Apache 了，这些文档啥的都必须更新啊，不要还滞后着呢，不然少不了被吐槽！

搞了很久这种方法没成功，暂时放弃！mmp

2、非 Docker 安装，只好把源码编译打包了。

1) 注意需要修改如下图中的配置：

![2018-02-08_10-26-03](https://ws3.sinaimg.cn/large/006tNc79gy1fo8vq7xv71j31kw0nktmi.jpg)

```Properteis
rocketmq.config.namesrvAddr=localhost:9876		//注意替换你自己的ip

#如果你 rocketmq 版本小于 3.5.8 才需设置 `rocketmq.config.isVIPChannel` 为 false，默认是 true, 这个可以在源码中可以看到的
rocketmq.config.isVIPChannel=
```

2) 执行以下命令：

```
mvn clean package -Dmaven.test.skip=true
```

编译成功：

![2018-02-08_10-41-35](https://ws3.sinaimg.cn/large/006tNc79gy1fo8vqk3r9ej31kw0tsh2z.jpg)

可以看到已经打好了 jar 包：

运行：

```
java -jar rocketmq-console-ng-1.0.0.jar
```

成功，不报错了，开心😄，访问 <http://localhost:8080/>

![2018-02-08_11-06-26](https://ws3.sinaimg.cn/large/006tNc79gy1fo8vqdzp1kj31kw0vyqap.jpg)

![2018-02-08_11-08-01](https://ws2.sinaimg.cn/large/006tNc79gy1fo8vqlthfqj31e20liwgz.jpg)

![2018-02-08_11-09-16](https://ws4.sinaimg.cn/large/006tNc79gy1fo8vqgsrl8j31kw09pjur.jpg)

![2018-02-08_11-09-31](https://ws1.sinaimg.cn/large/006tNc79gy1fo8vqaazsyj31kw0p8qce.jpg)

![2018-02-08_11-09-47](https://ws2.sinaimg.cn/large/006tNc79gy1fo8vqbosxvj31kw0vowp0.jpg)

![2018-02-08_11-10-03](https://ws3.sinaimg.cn/large/006tNc79gy1fo8vqet48yj31kw0bsmzk.jpg)

![2018-02-08_11-10-22](https://ws3.sinaimg.cn/large/006tNc79gy1fo8vq8q3v9j31kw0dfn0m.jpg)

整个监控大概就是这些了。

然后我运行之前的 SpringBoot 整合项目，查看监控信息如下：

![2018-02-08_11-22-11](https://ws3.sinaimg.cn/large/006tNc79gy1fo8vqcyj00j31kw0up46m.jpg)

# 总结

整篇文章讲述了 SpringBoot 与 RocketMQ 整合和 RocketMQ 监控平台的搭建。

# 参考文章

1、[http://www.ymq.io/2018/02/02/spring-boot-rocketmq-example/#%E6%96%B0%E5%8A%A0%E9%A1%B9%E7%9B%AE](http://www.54tianzhisheng.cn/2018/02/07/SpringBoot-RocketMQ/)

2、GitHub 官方 README