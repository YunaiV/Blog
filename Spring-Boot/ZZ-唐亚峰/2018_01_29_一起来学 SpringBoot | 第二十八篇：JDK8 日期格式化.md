title: 一起来学 SpringBoot 2.x | 第二十八篇：JDK8 日期格式化
date: 2018-01-29
tag: 
categories: Spring Boot
permalink: Spring-Boot/battcn/v2-localdatetime/
author: 唐亚峰
from_url: http://blog.battcn.com/2018/10/01/springboot/v2-localdatetime/
wechat_url: 

-------

摘要: 原创出处 http://blog.battcn.com/2018/10/01/springboot/v2-localdatetime/ 「唐亚峰」欢迎转载，保留摘要，谢谢！

- [为什么要用新的日期类型](http://www.iocoder.cn/Spring-Boot/battcn/v2-localdatetime//)
- [为什么要格式化](http://www.iocoder.cn/Spring-Boot/battcn/v2-localdatetime//)
- [导入依赖](http://www.iocoder.cn/Spring-Boot/battcn/v2-localdatetime//)
- [配置文件](http://www.iocoder.cn/Spring-Boot/battcn/v2-localdatetime//)
- [方案一（强烈推荐）](http://www.iocoder.cn/Spring-Boot/battcn/v2-localdatetime//)
- [方案二（强烈推荐）](http://www.iocoder.cn/Spring-Boot/battcn/v2-localdatetime//)
- [方案三（可选）](http://www.iocoder.cn/Spring-Boot/battcn/v2-localdatetime//)
- [控制层](http://www.iocoder.cn/Spring-Boot/battcn/v2-localdatetime//)
- [主函数](http://www.iocoder.cn/Spring-Boot/battcn/v2-localdatetime//)
- [测试](http://www.iocoder.cn/Spring-Boot/battcn/v2-localdatetime//)
- [总结](http://www.iocoder.cn/Spring-Boot/battcn/v2-localdatetime//)
- [666. 彩蛋](http://www.iocoder.cn/Spring-Boot/battcn/v2-localdatetime//)

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

# 为什么要用新的日期类型

**在 JDK8 中，一个新的重要特性就是引入了全新的时间和日期API，它被收录在 java.time 包中。借助新的时间和日期API可以以更简洁的方法处理时间和日期。**

**在 JDK8 之前，所有关于时间和日期的API存在以下几个缺陷，也正是这些缺陷，出现了很多第三方的日期处理框架，例如 Joda-Time，date4j 等开源项目。但是，Java 需要一套标准的用于处理时间和日期的框架，于是乎在 JDK8 中引入了新的日期API。遵循 JSR-310 规范的实现，而 Joda-Time 框架的作者正是 JSR-310 的规范的倡导者，所以用过 Joda-Time 的对新日期API也不会陌生。**

> 缺陷

- **之前的 java.util.Date 和 java.util.Calendar 类易用性差，不支持时区，且非线程安全的；**
- **日期格式化类 java.text.DateFormat 是一个抽象类，使用时需要先实例化一个 SimpleDateFormat 对象来处理日期格式化，同时DateFormat 也不是线程安全的，这意味着如果你在多线程程序中调用同一个 DateFormat 对象，会得到意想不到的结果。**
- **对日期的计算方式繁琐，而且容易出错，因为月份是从0开始的，从 Calendar 中获取的月份需要加一才能表示当前月份。**

# 为什么要格式化

说了这么多，和 `Spring Boot` 有什么关系呢？不要急，待我娓娓道来！

- **格式化前：{“payTime”:”2018-09-30T09:51:56.77”}**
- **格式化后：{“payTime”:”2018-09-30 09:51:56”}**

**都知道我们国人习惯 yyyy-MM-dd HH:mm:ss 这种格式的日期，但奈何框架是歪国大佬们写的，他们的日期格式与我们相差甚远，好在Spring Boot 提供了 spring.jackson.date-format，奈何它只能格式化 java.util.Date。那么解决办法是什么呢？**

# 导入依赖

首先一个 `WEB` 项目，必不可少的依赖就是 `spring-boot-starter-web` 了，一路学习下来的小伙伴们肯定都熟记于心了

```XML
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

# 配置文件

**由于 spring.jackson.date-format 对新的日期类型不在有效果，所以这里0配置文件了**

# 方案一（强烈推荐）

只需要定义一个配置类，在里面定义两个 `Bean` 即可完成全局日期格式化处理，这种方式也是本人现在使用的，同时还兼顾了 `Date` 和 `LocalDateTime` 并存

```Java
package com.battcn.config;

import com.fasterxml.jackson.datatype.jsr310.ser.LocalDateTimeSerializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.autoconfigure.jackson.Jackson2ObjectMapperBuilderCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

/**
 * @author Levin
 * @since 2018/9/30 0030
 */
@Configuration
public class LocalDateTimeSerializerConfig {

    @Value("${spring.jackson.date-format:yyyy-MM-dd HH:mm:ss}")
    private String pattern;

    @Bean
    public LocalDateTimeSerializer localDateTimeDeserializer() {
        return new LocalDateTimeSerializer(DateTimeFormatter.ofPattern(pattern));
    }

    @Bean
    public Jackson2ObjectMapperBuilderCustomizer jackson2ObjectMapperBuilderCustomizer() {
        return builder -> builder.serializerByType(LocalDateTime.class, localDateTimeDeserializer());
    }
}
```

# 方案二（强烈推荐）

有时候，我们对日期格式要做特殊的处理，全局的格式化方式无法满足我们需求是，使用该方案是非常好的选择，通过 `@JsonFormat` 注解我们可以更为精准的为日期字段格式化，它也是优先级最高的

```
public class Order {

    @JsonFormat(pattern = "yyyy-MM-dd")
    private LocalDateTime payTime;
    // 省略 GET SET ...
}
```

# 方案三（可选）

其实和第一种类似，只不过第一种的写法更加优雅简洁，如果有多种类型需要做统一格式化处理，这种方案也不是不可以考虑

```Java
package com.battcn.config;

import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.core.JsonParser;
import com.fasterxml.jackson.databind.*;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import com.fasterxml.jackson.datatype.jsr310.deser.LocalDateDeserializer;
import com.fasterxml.jackson.datatype.jsr310.ser.LocalDateSerializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

import java.io.IOException;
import java.time.LocalDate;
import java.time.LocalDateTime;

import static java.time.format.DateTimeFormatter.ofPattern;

/**
 * @author Levin
 * @since 2018/9/30 0030
 */
@Configuration
public class LocalDateTimeSerializerConfig1 {

    @Value("${spring.jackson.date-format:yyyy-MM-dd HH:mm:ss}")
    private String pattern;

    @Bean
    @Primary
    public ObjectMapper serializingObjectMapper() {
        ObjectMapper objectMapper = new ObjectMapper();
        JavaTimeModule javaTimeModule = new JavaTimeModule();
        javaTimeModule.addSerializer(LocalDateTime.class, new LocalDateTimeSerializer());
        javaTimeModule.addDeserializer(LocalDateTime.class, new LocalDateTimeDeserializer());
        objectMapper.registerModule(javaTimeModule);
        return objectMapper;
    }

    public class LocalDateTimeSerializer extends JsonSerializer<LocalDateTime> {
        @Override
        public void serialize(LocalDateTime value, JsonGenerator gen, SerializerProvider serializers) throws IOException {
            gen.writeString(value.format(ofPattern(pattern)));
        }
    }

    public class LocalDateTimeDeserializer extends JsonDeserializer<LocalDateTime> {
        @Override
        public LocalDateTime deserialize(JsonParser p, DeserializationContext deserializationContext) throws IOException {
            return LocalDateTime.parse(p.getValueAsString(), ofPattern(pattern));
        }
    }
}
```

# 控制层

定义了一个很简单的响应

```Java
package com.battcn.controller;

import com.battcn.pojo.Order;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.time.LocalDateTime;

/**
 * @author Levin
 * @since 2018/8/2 0002
 */
@RestController
@RequestMapping("/orders")
public class OrderController {

    @GetMapping
    public Order query() {
        Order order = new Order();
        order.setPayTime(LocalDateTime.now());
        return order;
    }
}
```

# 主函数

```Java
package com.battcn;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * @author Levin
 */
@SpringBootApplication
public class Chapter28Application {

    public static void main(String[] args) {
        SpringApplication.run(Chapter28Application.class, args);
    }
}
```

# 测试

完成准备事项后，启动`Chapter28Application`，访问 <http://localhost:8080/orders> 即可看到格式化后的日期啦…

# 总结

目前很多大佬都写过关于 **SpringBoot** 的教程了，如有雷同，请多多包涵，本教程基于最新的 `spring-boot-starter-parent：2.0.3.RELEASE`编写，包括新版本的特性都会一起介绍…

全文代码：https://github.com/battcn/spring-boot2-learning/tree/master/chapter28


# 666. 彩蛋

如果你对 SpringBoot 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)