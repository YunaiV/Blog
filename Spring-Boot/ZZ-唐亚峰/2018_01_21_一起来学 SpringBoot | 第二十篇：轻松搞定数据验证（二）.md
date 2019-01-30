title: 一起来学 SpringBoot 2.x | 第二十篇：轻松搞定数据验证（二）
date: 2018-01-21
tag: 
categories: Spring Boot
permalink: Spring-Boot/battcn/v2-other-validate2/
author: 唐亚峰
from_url: http://blog.battcn.com/2018/06/06/springboot/v2-other-validate2/
wechat_url: 

-------

摘要: 原创出处 http://blog.battcn.com/2018/06/06/springboot/v2-other-validate2/ 「唐亚峰」欢迎转载，保留摘要，谢谢！

- [为何要自定义](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-validate2//)
- [本章目标](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-validate2//)
- [具体代码](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-validate2//)
  - [导入依赖](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-validate2//)
  - [自定义注解](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-validate2//)
  - [具体验证](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-validate2//)
  - [控制层](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-validate2//)
  - [主函数](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-validate2//)
  - [测试](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-validate2//)
- [总结](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-validate2//)
- [说点什么](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-validate2//)

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

[一起来学SpringBoot | 第十九篇：轻松搞定数据验证（一）](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-validate1/) 中介绍了**数据有效性校验的重要性，也简单介绍了如何用轻松的方式搞定数据有效性校验**，但是当系统自带的注解无法满足我们的要求时候应该咋办呢？这就是本章将给各位介绍的**自定义 Validator 注解**

# 为何要自定义

`javax.validation` 包与 `hibernate-validator` 包中存在的注解几乎可以满足大部分的要求，又拥有基于正则表达式的`@Pattern`，为什么还需要自己去定义呢？

> 原因如下

- 正则效率不高
- 正则可读性不好
- 正则门槛较高，很多开发者并不会编写正则表达式

# 本章目标

熟悉 `ConstraintValidator` 接口并且编写自己的数据验证注解

# 具体代码

非常简单…

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

## 自定义注解

这里定义了一个 `@DateTime` 注解，在该注解上标注了 `@Constraint` 注解，它的作用就是指定一个具体的校验器类

> 关键字段（强制性）

- **message：** 验证失败提示的消息内容
- **groups：** 为约束指定验证组（非常不错的一个功能，下一章介绍）
- **payload：** 不太清楚（欢迎留言交流）

```java
package com.battcn.annotation;


import com.battcn.validator.DateTimeValidator;

import javax.validation.Constraint;
import javax.validation.Payload;
import java.lang.annotation.Retention;
import java.lang.annotation.Target;

import static java.lang.annotation.ElementType.FIELD;
import static java.lang.annotation.ElementType.PARAMETER;
import static java.lang.annotation.RetentionPolicy.RUNTIME;

/**
 * @author Levin
 * @since 2018/6/6 0006
 */
@Target({FIELD, PARAMETER})
@Retention(RUNTIME)
@Constraint(validatedBy = DateTimeValidator.class)
public @interface DateTime {

    String message() default "格式错误";

    String format() default "yyyy-MM-dd";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

## 具体验证

定义校验器类 `DateTimeValidator` 实现 `ConstraintValidator` 接口，实现接口后需要实现它里面的 `initialize：` 与 `isValid：` 方法。

> 方法介绍

- **initialize：** 主要用于初始化，它可以获得当前注解的所有属性
- **isValid：** 进行约束验证的主体方法，其中 `value` 就是验证参数的具体实例，`context` 代表约束执行的上下文环境。

这里的验证方式虽然简单，但职责明确；**为空验证可以使用 @NotBlank、@NotNull、@NotEmpty 等注解来进行控制，而不是在一个注解中做各种各样的规则判断，应该职责分离**

```java
package com.battcn.validator;

import com.battcn.annotation.DateTime;

import javax.validation.ConstraintValidator;
import javax.validation.ConstraintValidatorContext;
import java.text.ParseException;
import java.text.SimpleDateFormat;

/**
 * 日期格式验证
 *
 * @author Levin
 * @version 1.0.0
 * @since 2018-06-06
 */
public class DateTimeValidator implements ConstraintValidator<DateTime, String> {

    private DateTime dateTime;

    @Override
    public void initialize(DateTime dateTime) {
        this.dateTime = dateTime;
    }

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        // 如果 value 为空则不进行格式验证，为空验证可以使用 @NotBlank @NotNull @NotEmpty 等注解来进行控制，职责分离
        if (value == null) {
            return true;
        }
        String format = dateTime.format();
        if (value.length() != format.length()) {
            return false;
        }
        SimpleDateFormat simpleDateFormat = new SimpleDateFormat(format);
        try {
            simpleDateFormat.parse(value);
        } catch (ParseException e) {
            return false;
        }
        return true;
    }
}
```

## 控制层

```java
package com.battcn.controller;

import com.battcn.annotation.DateTime;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * 参数校验
 *
 * @author Levin
 * @since 2018/6/04 0031
 */
@Validated
@RestController
public class ValidateController {

    @GetMapping("/test")
    public String test(@DateTime(message = "您输入的格式错误，正确的格式为：{format}", format = "yyyy-MM-dd HH:mm") String date) {
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
public class Chapter19Application {

    public static void main(String[] args) {
        SpringApplication.run(Chapter19Application.class, args);
    }
}
```

## 测试

完成准备事项后，启动 `Chapter19Application` 自行测试即可，测试手段相信大伙都不陌生了，如 `浏览器`、`postman`、`junit`、`swagger`，此处基于 `postman`，如果你觉得自带的异常信息不够友好，那么配上[一起来学SpringBoot | 第十八篇：轻松搞定全局异常](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-exception/) 可以轻松搞定…

> 错误格式

[![错误格式](https://image.battcn.com/article/images/20180606/springboot/v2-other-validate/3.png)](https://image.battcn.com/article/images/20180606/springboot/v2-other-validate/3.png)

> 正确格式

[![正确格式](https://image.battcn.com/article/images/20180606/springboot/v2-other-validate/4.png)](https://image.battcn.com/article/images/20180606/springboot/v2-other-validate/4.png)

# 总结

目前很多大佬都写过关于 **SpringBoot** 的教程了，如有雷同，请多多包涵，本教程基于最新的 `spring-boot-starter-parent：2.0.2.RELEASE`编写，包括新版本的特性都会一起介绍…

# 说点什么

全文代码：<https://github.com/battcn/spring-boot2-learning/tree/master/chapter19>

# 666. 彩蛋

如果你对 SpringBoot 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)