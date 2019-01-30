title: 一起来学 SpringBoot 2.x | 第十三篇：RabbitMQ 延迟队列
date: 2018-01-14
tag: 
categories: Spring Boot
permalink: Spring-Boot/battcn/v2-queue-rabbitmq-delay/
author: 唐亚峰
from_url: http://blog.battcn.com/2018/05/23/springboot/v2-queue-rabbitmq-delay/
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247485829&idx=2&sn=59fd11d1b19d201fa958e8c9e11d1d2e&chksm=fa497634cd3eff22505453f31a210fd88dc11c3994e3582b955ca30e5059f3fda35f8ae0da65&token=55862109&lang=zh_CN#rd

-------

摘要: 原创出处 http://blog.battcn.com/2018/05/23/springboot/v2-queue-rabbitmq-delay/ 「唐亚峰」欢迎转载，保留摘要，谢谢！

- [延迟队列](http://www.iocoder.cn/Spring-Boot/battcn/v2-queue-rabbitmq-delay//)
- [RabbitMQ 实现思路](http://www.iocoder.cn/Spring-Boot/battcn/v2-queue-rabbitmq-delay//)
- [导入依赖](http://www.iocoder.cn/Spring-Boot/battcn/v2-queue-rabbitmq-delay//)
- [属性配置](http://www.iocoder.cn/Spring-Boot/battcn/v2-queue-rabbitmq-delay//)
- [手动ACK 不开启自动ACK模式,目的是防止报错后未正确处理消息丢失 默认 为 none](http://www.iocoder.cn/Spring-Boot/battcn/v2-queue-rabbitmq-delay//)
- [具体编码](http://www.iocoder.cn/Spring-Boot/battcn/v2-queue-rabbitmq-delay//)
  - [定义队列](http://www.iocoder.cn/Spring-Boot/battcn/v2-queue-rabbitmq-delay//)
  - [实体类](http://www.iocoder.cn/Spring-Boot/battcn/v2-queue-rabbitmq-delay//)
  - [控制器](http://www.iocoder.cn/Spring-Boot/battcn/v2-queue-rabbitmq-delay//)
  - [消息消费者](http://www.iocoder.cn/Spring-Boot/battcn/v2-queue-rabbitmq-delay//)
  - [主函数](http://www.iocoder.cn/Spring-Boot/battcn/v2-queue-rabbitmq-delay//)
  - [测试](http://www.iocoder.cn/Spring-Boot/battcn/v2-queue-rabbitmq-delay//)
- [总结](http://www.iocoder.cn/Spring-Boot/battcn/v2-queue-rabbitmq-delay//)
- [说点什么](http://www.iocoder.cn/Spring-Boot/battcn/v2-queue-rabbitmq-delay//)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

> `SpringBoot` 是为了简化 `Spring` 应用的创建、运行、调试、部署等一系列问题而诞生的产物，**自动装配的特性让我们可以更好的关注业务本身而不是外部的XML配置，我们只需遵循规范，引入相关的依赖就可以轻易的搭建出一个 WEB 工程**

[初探RabbitMQ消息队列](http://www.iocoder.cn/Spring-Boot/battcn/v2-queue-rabbitmq/)中介绍了`RabbitMQ`的简单用法，顺带提及了下延迟队列的作用。**所谓延时消息就是指当消息被发送以后，并不想让消费者立即拿到消息，而是等待指定时间后，消费者才拿到这个消息进行消费。**

# 延迟队列

> 延迟队列能做什么？

- **订单业务：** 在电商/点餐中，都有下单后 30 分钟内没有付款，就自动取消订单。
- **短信通知：** 下单成功后 60s 之后给用户发送短信通知。
- **失败重试：** 业务操作失败后，间隔一定的时间进行失败重试。

这类业务的特点就是：非实时的，需要延迟处理，需要进行失败重试。一种比较笨的方式是采用定时任务，轮训数据库，方法简单好用，但性能底下，在高并发情况下容易弄死数据库，间隔时间不好设置，时间过大，影响精度，过小影响性能，而且做不到按超时的时间顺序处理。另一种就是用**Java中的DelayQueue 位于java.util.concurrent包下，本质是由PriorityQueue和BlockingQueue实现的阻塞优先级队列。**，这玩意最大的问题就是**不支持分布式与持久化**

# RabbitMQ 实现思路

`RabbitMQ队列`本身是没有直接实现支持延迟队列的功能，但可以通过它的[Time-To-Live Extensions](https://www.rabbitmq.com/ttl.html) 与 [Dead Letter Exchange](http://www.rabbitmq.com/dlx.html) 的特性模拟出延迟队列的功能。

> Time-To-Live Extensions

**RabbitMQ支持为队列或者消息设置TTL（time to live 存活时间）**。TTL表明了一条消息可在队列中存活的最大时间。当某条消息被设置了TTL或者当某条消息进入了设置了TTL的队列时，这条消息会在TTL时间后**死亡成为Dead Letter**。如果既配置了消息的TTL，又配置了队列的TTL，那么较小的那个值会被取用。

> Dead Letter Exchange

**死信交换机**，上文中提到设置了 TTL 的消息或队列最终会成为`Dead Letter`。如果为队列设置了`Dead Letter Exchange（DLX）`，那么这些`Dead Letter`就会被重新发送到`Dead Letter Exchange`中，然后通过`Dead Letter Exchange`路由到其他队列，即可实现延迟队列的功能。

[![流程图](https://image.battcn.com/article/images/20180523/springboot/v2-queue-rabbitmq-delay/1.png)](https://image.battcn.com/article/images/20180523/springboot/v2-queue-rabbitmq-delay/1.png)

# 导入依赖

在 `pom.xml` 中添加 `spring-boot-starter-amqp`的依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>1.2.46</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

# 属性配置

在 `application.properties` 文件中配置`rabbitmq`相关内容，值得注意的是这里配置了手动ACK的开关

```properties
spring.rabbitmq.username=battcn
spring.rabbitmq.password=battcn
spring.rabbitmq.host=192.168.0.133
spring.rabbitmq.port=5672
spring.rabbitmq.virtual-host=/
# 手动ACK 不开启自动ACK模式,目的是防止报错后未正确处理消息丢失 默认 为 none
spring.rabbitmq.listener.simple.acknowledge-mode=manual
```

# 具体编码

## 定义队列

如果手动创建过或者`RabbitMQ`中已经存在该队列那么也可以省略下述代码…

```JAVA
package com.battcn.config;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.core.*;
import org.springframework.amqp.rabbit.connection.CachingConnectionFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.HashMap;
import java.util.Map;

/**
 * RabbitMQ配置
 *
 * @author Levin
 * @since 2018/4/11 0011
 */
@Configuration
public class RabbitConfig {

    private static final Logger log = LoggerFactory.getLogger(RabbitConfig.class);

    @Bean
    public RabbitTemplate rabbitTemplate(CachingConnectionFactory connectionFactory) {
        connectionFactory.setPublisherConfirms(true);
        connectionFactory.setPublisherReturns(true);
        RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
        rabbitTemplate.setMandatory(true);
        rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> log.info("消息发送成功:correlationData({}),ack({}),cause({})", correlationData, ack, cause));
        rabbitTemplate.setReturnCallback((message, replyCode, replyText, exchange, routingKey) -> log.info("消息丢失:exchange({}),route({}),replyCode({}),replyText({}),message:{}", exchange, routingKey, replyCode, replyText, message));
        return rabbitTemplate;
    }

    /**
     * 延迟队列 TTL 名称
     */
    private static final String REGISTER_DELAY_QUEUE = "dev.book.register.delay.queue";
    /**
     * DLX，dead letter发送到的 exchange
     * TODO 此处的 exchange 很重要,具体消息就是发送到该交换机的
     */
    public static final String REGISTER_DELAY_EXCHANGE = "dev.book.register.delay.exchange";
    /**
     * routing key 名称
     * TODO 此处的 routingKey 很重要要,具体消息发送在该 routingKey 的
     */
    public static final String DELAY_ROUTING_KEY = "";


    public static final String REGISTER_QUEUE_NAME = "dev.book.register.queue";
    public static final String REGISTER_EXCHANGE_NAME = "dev.book.register.exchange";
    public static final String ROUTING_KEY = "all";

    /**
     * 延迟队列配置
     * <p>
     * 1、params.put("x-message-ttl", 5 * 1000);
     * TODO 第一种方式是直接设置 Queue 延迟时间 但如果直接给队列设置过期时间,这种做法不是很灵活,（当然二者是兼容的,默认是时间小的优先）
     * 2、rabbitTemplate.convertAndSend(book, message -> {
     * message.getMessageProperties().setExpiration(2 * 1000 + "");
     * return message;
     * });
     * TODO 第二种就是每次发送消息动态设置延迟时间,这样我们可以灵活控制
     **/
    @Bean
    public Queue delayProcessQueue() {
        Map<String, Object> params = new HashMap<>();
        // x-dead-letter-exchange 声明了队列里的死信转发到的DLX名称，
        params.put("x-dead-letter-exchange", REGISTER_EXCHANGE_NAME);
        // x-dead-letter-routing-key 声明了这些死信在转发时携带的 routing-key 名称。
        params.put("x-dead-letter-routing-key", ROUTING_KEY);
        return new Queue(REGISTER_DELAY_QUEUE, true, false, false, params);
    }

    /**
     * 需要将一个队列绑定到交换机上，要求该消息与一个特定的路由键完全匹配。
     * 这是一个完整的匹配。如果一个队列绑定到该交换机上要求路由键 “dog”，则只有被标记为“dog”的消息才被转发，不会转发dog.puppy，也不会转发dog.guard，只会转发dog。
     * TODO 它不像 TopicExchange 那样可以使用通配符适配多个
     *
     * @return DirectExchange
     */
    @Bean
    public DirectExchange delayExchange() {
        return new DirectExchange(REGISTER_DELAY_EXCHANGE);
    }

    @Bean
    public Binding dlxBinding() {
        return BindingBuilder.bind(delayProcessQueue()).to(delayExchange()).with(DELAY_ROUTING_KEY);
    }


    @Bean
    public Queue registerBookQueue() {
        return new Queue(REGISTER_QUEUE_NAME, true);
    }

    /**
     * 将路由键和某模式进行匹配。此时队列需要绑定要一个模式上。
     * 符号“#”匹配一个或多个词，符号“*”匹配不多不少一个词。因此“audit.#”能够匹配到“audit.irs.corporate”，但是“audit.*” 只会匹配到“audit.irs”。
     **/
    @Bean
    public TopicExchange registerBookTopicExchange() {
        return new TopicExchange(REGISTER_EXCHANGE_NAME);
    }

    @Bean
    public Binding registerBookBinding() {
        // TODO 如果要让延迟队列之间有关联,这里的 routingKey 和 绑定的交换机很关键
        return BindingBuilder.bind(registerBookQueue()).to(registerBookTopicExchange()).with(ROUTING_KEY);
    }

}
```

## 实体类

创建一个`Book`类

```JAVA
public class Book implements java.io.Serializable {

    private static final long serialVersionUID = -2164058270260403154L;

    private String id;
    private String name;
	// 省略get set ...
}
```

## 控制器

编写一个`Controller`类，用于消息发送工作，同时为了看到测试效果，添加日志输出，将发送消息的时间记录下来..

```java
package com.battcn.controller;

import com.battcn.config.RabbitConfig;
import com.battcn.entity.Book;
import com.battcn.handler.BookHandler;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.support.converter.AbstractJavaTypeMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.time.LocalDateTime;

/**
 * @author Levin
 * @since 2018/4/2 0002
 */
@RestController
@RequestMapping(value = "/books")
public class BookController {

    private static final Logger log = LoggerFactory.getLogger(BookController.class);

    private final RabbitTemplate rabbitTemplate;

    @Autowired
    public BookController(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    /**
     * this.rabbitTemplate.convertAndSend(RabbitConfig.REGISTER_DELAY_EXCHANGE, RabbitConfig.DELAY_ROUTING_KEY, book); 对应 {@link BookHandler#listenerDelayQueue}
     */
    @GetMapping
    public void defaultMessage() {
        Book book = new Book();
        book.setId("1");
        book.setName("一起来学Spring Boot");
        // 添加延时队列
        this.rabbitTemplate.convertAndSend(RabbitConfig.REGISTER_DELAY_EXCHANGE, RabbitConfig.DELAY_ROUTING_KEY, book, message -> {
            // TODO 第一句是可要可不要,根据自己需要自行处理
            message.getMessageProperties().setHeader(AbstractJavaTypeMapper.DEFAULT_CONTENT_CLASSID_FIELD_NAME, Book.class.getName());
            // TODO 如果配置了 params.put("x-message-ttl", 5 * 1000); 那么这一句也可以省略,具体根据业务需要是声明 Queue 的时候就指定好延迟时间还是在发送自己控制时间
            message.getMessageProperties().setExpiration(5 * 1000 + "");
            return message;
        });
        log.info("[发送时间] - [{}]", LocalDateTime.now());
    }
}
```

## 消息消费者

默认情况下 `spring-boot-data-amqp` 是自动`ACK`机制，就意味着 MQ 会在消息消费完毕后自动帮我们去ACK，这样依赖就存在这样一个问题：**如果报错了，消息不会丢失，会无限循环消费，很容易就吧磁盘空间耗完，虽然可以配置消费的次数但这种做法也有失优雅。目前比较推荐的就是我们手动ACK然后将消费错误的消息转移到其它的消息队列中，做补偿处理。** 由于我们需要手动控制`ACK`，因此下面监听完消息后需要调用`basicAck`通知`rabbitmq`消息已被正确消费，可以将远程队列中的消息删除

```java
package com.battcn.handler;

import com.battcn.config.RabbitConfig;
import com.battcn.entity.Book;
import com.rabbitmq.client.Channel;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.time.LocalDateTime;

/**
 * BOOK_QUEUE 消费者
 *
 * @author Levin
 * @since 2018/4/11 0011
 */
@Component
public class BookHandler {

    private static final Logger log = LoggerFactory.getLogger(BookHandler.class);

    @RabbitListener(queues = {RabbitConfig.REGISTER_QUEUE_NAME})
    public void listenerDelayQueue(Book book, Message message, Channel channel) {
        log.info("[listenerDelayQueue 监听的消息] - [消费时间] - [{}] - [{}]", LocalDateTime.now(), book.toString());
        try {
            // TODO 通知 MQ 消息已被成功消费,可以ACK了
            channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
        } catch (IOException e) {
            // TODO 如果报错了,那么我们可以进行容错处理,比如转移当前消息进入其它队列
        }
    }
}
```

## 主函数

```java
package com.battcn;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * @author Levin
 */
@SpringBootApplication
public class Chapter12Application {

    public static void main(String[] args) {
        SpringApplication.run(Chapter12Application.class, args);
    }
}
```

## 测试

完成准备事项后，启动`Chapter12Application` 访问 <http://localhost:8080/books> 将会看到如下内容，就代表一切正常….

```shell
2018-05-23 19:56:36.248  INFO 29048 --- [nio-8080-exec-1] com.battcn.controller.BookController     : [发送时间] - [2018-05-23T19:56:36.248]
2018-05-23 19:56:41.256  INFO 29048 --- [cTaskExecutor-1] com.battcn.handler.BookHandler           : [listenerDelayQueue 监听的消息] - [消费时间] - [2018-05-23T19:56:41.256] - [Book{id='1', name='一起来学Spring Boot'}]
```

# 总结

目前很多大佬都写过关于 **SpringBoot** 的教程了，如有雷同，请多多包涵，本教程基于最新的 `spring-boot-starter-parent：2.0.2.RELEASE`编写，包括新版本的特性都会一起介绍…

# 说点什么

全文代码：<https://github.com/battcn/spring-boot2-learning/tree/master/chapter12>

# 666. 彩蛋

如果你对 SpringBoot 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)