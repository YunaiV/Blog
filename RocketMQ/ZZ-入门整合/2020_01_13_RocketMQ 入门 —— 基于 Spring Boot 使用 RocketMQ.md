title: RocketMQ 入门 —— 基于 Spring Boot 使用 RocketMQ
date: 2020-01-13
tags:
categories: RocketMQ
permalink: RocketMQ/start/spring-boot-example
author: zhisheng
from_url: https://mp.weixin.qq.com/s/lAUrwcwqzoUt571WhXrHaA

-------

摘要: 原创出处 https://mp.weixin.qq.com/s/lAUrwcwqzoUt571WhXrHaA 「辽天」欢迎转载，保留摘要，谢谢！

- [1. 前言](http://www.iocoder.cn/RocketMQ/start/spring-boot-example/)
- [2. Spring 中的消息框架](http://www.iocoder.cn/RocketMQ/start/spring-boot-example/)
  - [2.1  Spring Messaging](http://www.iocoder.cn/RocketMQ/start/spring-boot-example/)
  - [2.2 Spring Cloud Stream](http://www.iocoder.cn/RocketMQ/start/spring-boot-example/)
- [3. spring-boot-starter的实现](http://www.iocoder.cn/RocketMQ/start/spring-boot-example/)
  - [3.1 spring-boot-starter的实现步骤](http://www.iocoder.cn/RocketMQ/start/spring-boot-example/)
  - [3.2 消息发送端实现](http://www.iocoder.cn/RocketMQ/start/spring-boot-example/)
  - [3.3. 消息消费端实现](http://www.iocoder.cn/RocketMQ/start/spring-boot-example/)
- [4. 使用示例](http://www.iocoder.cn/RocketMQ/start/spring-boot-example/)
  - [4.1  RocketMQ服务端的准备](http://www.iocoder.cn/RocketMQ/start/spring-boot-example/)
  - [4.2. 编译rocketmq-spring-boot-starter](http://www.iocoder.cn/RocketMQ/start/spring-boot-example/)
  - [4.3. 编写客户端代码](http://www.iocoder.cn/RocketMQ/start/spring-boot-example/)
- [5. 参考文档](http://www.iocoder.cn/RocketMQ/start/spring-boot-example/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# 1. 前言

上世纪90年代末，随着Java EE(Enterprise Edition)的出现，特别是Enterprise Java Beans的使用需要复杂的描述符配置和死板复杂的代码实现，增加了广大开发者的学习曲线和开发成本，由此基于简单的XML配置和普通Java对象(Plain Old Java Objects)的Spring技术应运而生，依赖注入(Dependency Injection), 控制反转(Inversion of Control)和面向切面编程(AOP)的技术更加敏捷地解决了传统Java企业及版本的不足。

随着Spring的持续演进，基于注解(Annotation)的配置逐渐取代了XML文件配置， 2014年4月1日，Spring Boot 1.0.0正式发布，它基于“约定大于配置”（Convention over configuration)这一理念来快速地开发、测试、运行和部署Spring应用，并能通过简单地与各种启动器(如 spring-boot-web-starter)结合，让应用直接以命令行的方式运行，不需再部署到独立容器中。这种简便直接快速构建和开发应用的过程，可以使用约定的配置并且简化部署，受到越来越多的开发者的欢迎。

Apache RocketMQ是业界知名的分布式消息和流处理中间件，简单地理解，它由Broker服务器和客户端两部分组成：

> 其中客户端一个是消息发布者客户端(Producer)，它负责向Broker服务器发送消息；
>
> 另外一个是消息的消费者客户端(Consumer)，多个消费者可以组成一个消费组，来订阅和拉取消费Broker服务器上存储的消息。

为了利用Spring Boot的快速开发和让用户能够更灵活地使用RocketMQ消息客户端，Apache RocketMQ社区推出了spring-boot-starter实现。随着分布式事务消息功能在RocketMQ 4.3.0版本的发布，近期升级了相关的spring-boot代码，**通过注解方式支持分布式事务的回查和事务消息的发送。**

本文将对当前的设计实现做一个简单的介绍，读者可以通过本文了解将RocketMQ Client端集成为spring-boot-starter框架的开发细节，然后通过一个简单的示例来一步一步的讲解如何使用这个spring-boot-starter工具包来配置，发送和消费RocketMQ消息。

# 2. Spring 中的消息框架

顺便在这里讨论一下在Spring中关于消息的两个主要的框架，即Spring Messaging和Spring Cloud Stream。它们都能够与Spring Boot整合并提供了一些参考的实现。和所有的实现框架一样，消息框架的目的是实现轻量级的消息驱动的微服务，可以有效地简化开发人员对消息中间件的使用复杂度，让系统开发人员可以有更多的精力关注于核心业务逻辑的处理。

## 2.1  Spring Messaging

Spring Messaging是Spring Framework 4中添加的模块，是Spring与消息系统集成的一个扩展性的支持。它实现了从基于JmsTemplate的简单的使用JMS接口到异步接收消息的一整套完整的基础架构，Spring AMQP提供了该协议所要求的类似的功能集。 在与Spring Boot的集成后，它拥有了自动配置能力，能够在测试和运行时与相应的消息传递系统进行集成。



单纯对于客户端而言，Spring Messaging提供了一套抽象的API或者说是约定的标准，对消息发送端和消息接收端的模式进行规定，不同的消息中间件提供商可以在这个模式下提供自己的Spring实现：在消息发送端需要实现的是一个XXXTemplate形式的Java Bean，结合Spring Boot的自动化配置选项提供多个不同的发送消息方法；在消息的消费端是一个XXXMessageListener接口（实现方式通常会使用一个注解来声明一个消息驱动的POJO），提供回调方法来监听和消费消息，这个接口同样可以使用Spring Boot的自动化选项和一些定制化的属性。

如果有兴趣深入的了解Spring Messaging及针对不同的消息产品的使用，推荐阅读这个文件。参考Spring Messaging的既有实现，RocketMQ的spring-boot-starter中遵循了相关的设计模式并结合RocketMQ自身的功能特点提供了相应的API(如，顺序，异步和事务半消息等)。

## 2.2 Spring Cloud Stream

Spring Cloud Stream结合了Spring Integration的注解和功能，它的应用模型如下：

![该图片引自spring cloud stream](http://static.iocoder.cn/c7990b604259ccb133a24975afe563c5)

Spring Cloud Stream框架中提供一个独立的应用内核，它通过输入(@Input)和输出(@Output)通道与外部世界进行通信，消息源端(Source)通过输入通道发送消息，消费目标端(Sink)通过监听输出通道来获取消费的消息。这些通道通过专用的Binder实现与外部代理连接。开发人员的代码只需要针对应用内核提供的固定的接口和注解方式进行编程，而不需要关心运行时具体的Binder绑定的消息中间件。在运行时，Spring Cloud Stream能够自动探测并使用在classpath下找到的Binder。

这样开发人员可以轻松地在相同的代码中使用不同类型的中间件：仅仅需要在构建时包含进不同的Binder。在更加复杂的使用场景中，也可以在应用中打包多个Binder并让它自己选择Binder，甚至在运行时为不同的通道使用不同的Binder。

Binder抽象使得Spring Cloud Stream应用可以灵活的连接到中间件，加之Spring Cloud Stream使用利用了Spring Boot的灵活配置配置能力，这样的配置可以通过外部配置的属性和Spring Boo支持的任何形式来提供（包括应用启动参数、环境变量和application.yml或者application.properties文件），部署人员可以在运行时动态选择通道连接destination（例如，Kafka的topic或者RabbitMQ的exchange）。

Binder SPI的方式来让消息中间件产品使用可扩展的API来编写相应的Binder，并集成到Spring Cloud Steam环境，目前RocketMQ还没有提供相关的Binder，我们计划在下一步将完善这一功能，也希望社区里有这方面经验的同学积极尝试，贡献PR或建议。

# 3. spring-boot-starter的实现

在开始的时候我们已经知道，spring boot starter构造的启动器对于使用者是非常方便的，使用者只要在pom.xml引入starter的依赖定义，相应的编译，运行和部署功能就全部自动引入。因此常用的开源组件都会为Spring的用户提供一个spring-boot-starter封装给开发者，让开发者非常方便集成和使用，这里我们详细的介绍一下RocketMQ（客户端）的starter实现过程。

## 3.1 spring-boot-starter的实现步骤

对于一个spring-boot-starter实现需要包含如下几个部分：

**1. 在pom.xml的定义**

- 定义最终要生成的starter组件信息

```XML
<groupId>org.apache.rocketmq</groupId>
<artifactId>spring-boot-starter-rocketmq</artifactId>
<version>1.0.0-SNAPSHOT</version>
```

- 定义依赖包，

  它分为两个部分: A、Spring自身的依赖包； B、RocketMQ的依赖包

```XML

    <dependencies>
        <!-- spring-boot-start internal depdencies -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>


        <!-- rocketmq dependencies -->
        <dependency>
            <groupId>org.apache.rocketmq</groupId>
            <artifactId>rocketmq-client</artifactId>
            <version>${rocketmq-version}</version>
        </dependency>
    </dependencies>
        <dependencyManagement>
        <dependencies>
            <!-- spring-boot-start parent depdency definition -->
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-parent</artifactId>
                <version>${spring.boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
```

**2. 配置文件类**

定义应用属性配置文件类RocketMQProperties,这个Bean定义一组默认的属性值。用户在使用最终的starter时，可以根据这个类定义的属性来修改取值，当然不是直接修改这个类的配置，而是spring-boot应用中对应的配置文件：src/main/resources/application.properties.

**3. 定义自动加载类**

定义 src/resources/META-INF/spring.factories文件中的自动加载类， 其目的是让spring boot更具文中中所指定的自动化配置类来自动初始化相关的Bean，Component或Service，它的内容如下：

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\ org.apache.rocketmq.spring.starter.RocketMQAutoConfiguration
```

在RocketMQAutoConfiguration类的具体实现中，定义开放给用户直接使用的Bean对象. 包括：

- RocketMQProperties  加载应用属性配置文件的处理类；
- RocketMQTemplate    发送端用户发送消息的发送模板类；
- ListenerContainerConfiguration 容器Bean负责发现和注册消费端消费实现接口类，这个类要求：由@RocketMQMessageListener注解标注；实现RocketMQListener泛化接口。

**4. 最后具体的RocketMQ相关的封装**

在发送端（producer）和消费端(consumer)客户端分别进行封装，在当前的实现版本提供了对Spring Messaging接口的兼容方式。

## 3.2 消息发送端实现

**1. 普通发送端**

发送端的代码封装在RocketMQTemplate POJO中，下图是发送端的相关代码的调用关系图：

![img](http://static.iocoder.cn/94a503a2639ccc9bff66c2c8666e01e0)

为了与Spring Messaging的发送模板兼容，在RocketMQTemplate集成了AbstractMessageSendingTemplate抽象类，来支持相关的消息转换和发送方法，这些方法最终会代理给doSend()方法；doSend()以及RocoketMQ所特有的一些方法如异步，单向和顺序等方法直接添加到RoketMQTempalte中，这些方法直接代理调用到RocketMQ的Producer API来进行消息的发送。

**2. 事务消息发送端**

对于事务消息的处理，在消息发送端进行了部分的扩展，参考下图的调用关系类图：

![img](http://static.iocoder.cn/2939675c43177481fa169c61c0d1eba5)

RocketMQTemplate里加入了一个发送事务消息的方法sendMessageInTransaction(), 并且最终这个方法会代理到RocketMQ的TransactionProducer进行调用，在这个Producer上会注册其关联的TransactionListener实现类，以便在发送消息后能够对TransactionListener里的方法实现进行调用。

## 3.3. 消息消费端实现

![img](http://static.iocoder.cn/89550d06be89406f428e3ad5b6215886)

在消费端Spring-Boot应用启动后，会扫描所有包含@RocketMQMessageListener注解的类(这些类需要集成RocketMQListener接口，并实现onMessage()方法)，这个Listener会一对一的被放置到DefaultRocketMQListenerContainer容器对象中，容器对象会根据消费的方式(并发或顺序)，将RocketMQListener封装到具体的RocketMQ内部的并发或者顺序接口实现。在容器中创建RocketMQ Consumer对象，启动并监听定制的Topic消息，如果有消费消息，则回调到Listener的onMessage()方法。

# 4. 使用示例

上面的一章介绍了RocketMQ在spring-boot-starter方式的实现，这里通过一个最简单的消息发送和消费的例子来介绍如何使这个rocketmq-spring-boot-starter。

## 4.1  RocketMQ服务端的准备

**1. 启动NameServer和Broker**

要验证RocketMQ的Spring-Boot客户端，首先要确保RocketMQ服务正确的下载并启动。可以参考RocketMQ主站的快速开始来进行操作。确保启动NameServer和Broker已经正确启动。

**2. 创建实例中所需要的Topics**

在执行启动命令的目录下执行下面的命令行操作

```Bash
bash bin/mqadmin updateTopic -c DefaultCluster -t string-topic
```

## 4.2. 编译rocketmq-spring-boot-starter

目前的spring-boot-starter依赖还没有提交的Maven的中心库，用户使用前需要自行下载git源码，然后执行mvn clean install 安装到本地仓库。

```Bash
git clone https://github.com/apache/rocketmq-externals.git cd rocketmq-spring-boot-starter mvn clean install
```

## 4.3. 编写客户端代码

用户如果使用它，需要在消息的发布和消费客户端的maven配置文件pom.xml中添加如下的依赖：

```XML
<properties>
    <spring-boot-starter-rocketmq-version>1.0.0-SNAPSHOT</spring-boot-starter-rocketmq-version>
</properties>

<dependency>
   <groupId>org.apache.rocketmq</groupId>
   <artifactId>spring-boot-starter-rocketmq</artifactId>
   <version>${spring-boot-starter-rocketmq-version}</version>
</dependency>
```

属性spring-boot-starter-rocketmq-version的取值为：1.0.0-SNAPSHOT， 这与上一步骤中执行安装到本地仓库的版本一致。

**1. 消息发送端的代码**

发送端的配置文件application.properties

```Properties
# 定义name-server地址
spring.rocketmq.name-server=localhost:9876
# 定义发布者组名
spring.rocketmq.producer.group=my-group1
# 定义要发送的topic
spring.rocketmq.topic=string-topic
```

发送端的Java代码

```Java

import org.apache.rocketmq.spring.starter.core.RocketMQTemplate;
...

@SpringBootApplication
public class ProducerApplication implements CommandLineRunner {
    // 声明并引用RocketMQTemplate
    @Resource
    private RocketMQTemplate rocketMQTemplate;

    // 使用application.properties里定义的topic属性
    @Value("${spring.rocketmq.springTopic}")
    private String springTopic;

    public static void main(String[] args){
        SpringApplication.run(ProducerApplication.class, args);
    }

    public void run(String... args) throws Exception {
        // 以同步的方式发送字符串消息给指定的topic
        SendResult sendResult = rocketMQTemplate.syncSend(springTopic, "Hello, World!");
        // 打印发送结果信息
        System.out.printf("string-topic syncSend1 sendResult=%s %n", sendResult);
    }
}
```

**2. 消息消费端代码**

消费端的配置文件application.properties

```Properties
# 定义name-server地址
spring.rocketmq.name-server=localhost:9876
# 定义发布者组名
spring.rocketmq.consumer.group=my-customer-group1
# 定义要发送的topic
spring.rocketmq.topic=string-topic
```

消费端的Java代码

```Java
@SpringBootApplication
public class ConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConsumerApplication.class, args);
    }
}


// 声明消费消息的类，并在注解中指定，相关的消费信息
@Service
@RocketMQMessageListener(topic = "${spring.rocketmq.topic}", consumerGroup = "${spring.rocketmq.consumer.group}")
class StringConsumer implements RocketMQListener<String> {
    @Override
    public void onMessage(String message) {
        System.out.printf("------- StringConsumer received: %s %f", message);
    }
}
```

这里只是简单的介绍了使用spring-boot来编写最基本的消息发送和接收的代码，如果需要了解更多的调用方式，如: 异步发送，对象消息体，指定tag标签以及指定事务消息，请参看github的说明文档和详细的代码。我们后续还会对这些高级功能进行陆续的介绍。



# 5. 参考文档

1. Spring Boot features-Messaging
2. Enterprise Integration Pattern-组成简介
3. Spring Cloud Stream Reference Guide
4. https://dzone.com/articles/creating-custom-springboot-starter-for-twitter4j
