title: 一起来学 SpringBoot 2.x | 第十九篇：轻松搞定数据验证（一）
date: 2018-01-20
tag: 
categories: Spring Boot
permalink: Spring-Boot/battcn/v2-other-validate1/
author: 唐亚峰
from_url: http://blog.battcn.com/2018/06/05/springboot/v2-other-validate1/
wechat_url: 

-------

摘要: 原创出处 http://blog.battcn.com/2018/06/05/springboot/v2-other-validate1/ 「唐亚峰」欢迎转载，保留摘要，谢谢！

- [为什么要轻松搞定？](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-validate1//)
- [本章目标](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-validate1//)
- [具体代码](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-validate1//)
  - [导入依赖](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-validate1//)
  - [JSR-303 注释介绍](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-validate1//)
  - [实体类](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-validate1//)
  - [控制层](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-validate1//)
  - [主函数](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-validate1//)
  - [测试](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-validate1//)
- [总结](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-validate1//)
- [说点什么](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-validate1//)

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

对于任何一个应用而言，客户端做的数据有效性验证都不是安全有效的，**而数据验证又是一个企业级项目架构上最为基础的功能模块**，这时候就要求我们在服务端接收到数据的时候也对数据的有效性进行验证。为什么这么说呢？往往我们在编写程序的时候都会感觉后台的验证无关紧要，毕竟客户端已经做过验证了，后端没必要在浪费资源对数据进行验证了，但恰恰是这种思维最为容易被别人钻空子。毕竟只要有点开发经验的都知道，我们完全可以模拟 `HTTP` 请求到后台地址，模拟请求过程中发送一些涉及系统安全的数据到后台，后果可想而知….

# 为什么要轻松搞定？

相信通过上面的阅读，大家对数据验证的重要性有了一定的了解，那么为什么我这里要说 **轻松搞定呢？**

下面这段代码很多人一定见到过，就是对参数进行有效性校验，但仔细观察的话就会发现；**随着参数的增加，格式的变化，校验数据有效性的代码愈发的繁琐杂乱，一点都不轻松**

```java
public String test1(String name) {
    if (name == null) {
        throw new NullPointerException("name 不能为空");
    }
    if (name.length() < 2 || name.length() > 10) {
        throw new RuntimeException("name 长度必须在 2 - 10 之间");
    }
    return "success";
}
```

# 本章目标

通过 `Spring Boot` 完成参数后台数据校验，**轻松搞定数据有效性验证，留出更多的时间来和小姐姐聊天…**

# 具体代码

看看是如何轻松搞定的….

## 导入依赖

在 `pom.xml` 中添加上 `spring-boot-starter-web` 的依赖即可

```xml
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
</dependencies>
```

## JSR-303 注释介绍

这里只列举了 `javax.validation` 包下的注解，同理在 `spring-boot-starter-web` 包中也存在 `hibernate-validator` 验证包，里面包含了一些 `javax.validation` 没有的注解，有兴趣的可以看看

| 注解                        | 说明                                                         |
| --------------------------- | ------------------------------------------------------------ |
| `@NotNull`                  | **限制必须不为null**                                         |
| `@NotEmpty`                 | **验证注解的元素值不为 null 且不为空（字符串长度不为0、集合大小不为0）** |
| `@NotBlank`                 | **验证注解的元素值不为空（不为null、去除首位空格后长度为0），不同于@NotEmpty，@NotBlank只应用于字符串且在比较时会去除字符串的空格** |
| `@Pattern(value)`           | **限制必须符合指定的正则表达式**                             |
| `@Size(max,min)`            | **限制字符长度必须在 min 到 max 之间（也可以用在集合上）**   |
| `@Email`                    | **验证注解的元素值是Email，也可以通过正则表达式和flag指定自定义的email格式** |
| `@Max(value)`               | **限制必须为一个不大于指定值的数字**                         |
| `@Min(value)`               | **限制必须为一个不小于指定值的数字**                         |
| `@DecimalMax(value)`        | **限制必须为一个不大于指定值的数字**                         |
| `@DecimalMin(value)`        | **限制必须为一个不小于指定值的数字**                         |
| `@Null`                     | **限制只能为null（很少用）**                                 |
| `@AssertFalse`              | **限制必须为false （很少用）**                               |
| `@AssertTrue`               | **限制必须为true （很少用）**                                |
| `@Past`                     | **限制必须是一个过去的日期**                                 |
| `@Future`                   | **限制必须是一个将来的日期**                                 |
| `@Digits(integer,fraction)` | **限制必须为一个小数，且整数部分的位数不能超过 integer，小数部分的位数不能超过 fraction （很少用）** |

## 实体类

为了体现 `validation` 的强大，分别演示普通参数属性验证与对象的验证

```java
package com.battcn.pojo;

import org.hibernate.validator.constraints.Length;

import javax.validation.constraints.DecimalMin;
import javax.validation.constraints.NotBlank;
import javax.validation.constraints.NotNull;
import java.math.BigDecimal;

/**
 * @author Levin
 * @since 2018/6/5 0005
 */
public class Book {

    private Integer id;
    @NotBlank(message = "name 不允许为空")
    @Length(min = 2, max = 10, message = "name 长度必须在 {min} - {max} 之间")
    private String name;
    @NotNull(message = "price 不允许为空")
    @DecimalMin(value = "0.1", message = "价格不能低于 {value}")
    private BigDecimal price;

    // 省略 GET SET ...
}
```

## 控制层

与前面的代码相比，新的代码中仅仅多了几个注解而已。**（此处只是为了图方便写在了 Controller 层，同理你可以将它作用在 Service 层）**

> 注解介绍

- **@Validated：** 开启数据有效性校验，添加在类上即为验证方法，添加在方法参数中即为验证参数对象。（添加在方法上无效）
- **@NotBlank：** 被注释的字符串不允许为空（`value.trim() > 0 ? true : false`）
- **@Length：** 被注释的字符串的大小必须在指定的范围内
- **@NotNull：** 被注释的字段不允许为空(`value != null ? true : false`)
- **@DecimalMin：** 被注释的字段必须大于或等于指定的数值

```java
package com.battcn.controller;

import com.battcn.pojo.Book;
import org.hibernate.validator.constraints.Length;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.validation.constraints.NotBlank;

/**
 * 参数校验
 *
 * @author Levin
 * @since 2018/6/04 0031
 */
@Validated
@RestController
public class ValidateController {

    @GetMapping("/test2")
    public String test2(@NotBlank(message = "name 不能为空") @Length(min = 2, max = 10, message = "name 长度必须在 {min} - {max} 之间") String name) {
        return "success";
    }

    @GetMapping("/test3")
    public String test3(@Validated Book book) {
        return "success";
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
public class Chapter18Application {

    public static void main(String[] args) {
        SpringApplication.run(Chapter18Application.class, args);
    }
}
```

## 测试

完成准备事项后，启动 `Chapter18Application` 自行测试即可，测试手段相信大伙都不陌生了，如 `浏览器`、`postman`、`junit`、`swagger`，此处基于 `postman`，如果你觉得自带的异常信息不够友好，那么配上[一起来学SpringBoot | 第十八篇：轻松搞定全局异常](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-exception/) 可以轻松搞定…

> test2接口（name参数没传）

[![test2接口](https://image.battcn.com/article/images/20180604/springboot/v2-other-validate/1.png)](https://image.battcn.com/article/images/20180604/springboot/v2-other-validate/1.png)

> test3接口（price参数值过低）

[![test3接口](https://image.battcn.com/article/images/20180604/springboot/v2-other-validate/2.png)](https://image.battcn.com/article/images/20180604/springboot/v2-other-validate/2.png)

# 总结

目前很多大佬都写过关于 **SpringBoot** 的教程了，如有雷同，请多多包涵，本教程基于最新的 `spring-boot-starter-parent：2.0.2.RELEASE`编写，包括新版本的特性都会一起介绍…

# 说点什么

全文代码：<https://github.com/battcn/spring-boot2-learning/tree/master/chapter18>

# 666. 彩蛋

如果你对 SpringBoot 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)