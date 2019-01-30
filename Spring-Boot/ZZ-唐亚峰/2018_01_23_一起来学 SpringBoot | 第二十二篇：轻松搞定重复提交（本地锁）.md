title: 一起来学 SpringBoot 2.x | 第二十二篇：轻松搞定重复提交（本地锁）
date: 2018-01-23
tag: 
categories: Spring Boot
permalink: Spring-Boot/battcn/v2-cache-locallock/
author: 唐亚峰
from_url: http://blog.battcn.com/2018/06/12/springboot/v2-cache-locallock/
wechat_url: 

-------

摘要: 原创出处 http://blog.battcn.com/2018/06/12/springboot/v2-cache-locallock/ 「唐亚峰」欢迎转载，保留摘要，谢谢！

- [重复提交](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-locallock//)
- [本章目标](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-locallock//)
- [具体代码](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-locallock//)
  - [导入依赖](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-locallock//)
  - [Lock 注解](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-locallock//)
  - [Lock 拦截器（AOP）](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-locallock//)
  - [控制层](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-locallock//)
  - [主函数](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-locallock//)
  - [测试](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-locallock//)
- [总结](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-locallock//)
- [说点什么](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-locallock//)

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

在平时开发中，如果网速比较慢的情况下，用户提交表单后，发现服务器半天都没有响应，那么用户可能会以为是自己没有提交表单，就会再点击提交按钮重复提交表单，我们在开发中必须防止表单重复提交….

# 重复提交

字面意思就是提交了很多次，这种情况一般都是前端给你挖的坑….

前段时间在开发中遇到一个这样的问题；前端小哥哥调用接口的时候存在 **循环调用** 的问题，正常情况下发送一个请求添加一条数据，结果变成了同一时刻并发的发送了 N 个请求，服务端瞬间懵逼的插入了 N 条一模一样的数据，前端小哥哥也不知道问题在哪里（`恩...坑就这样挖好了，反正不填坑，气死你`） 这时候咋办呢；后端干呗，反正脏活累活，背锅的事情也没少干了，多一件也不多….

# 本章目标

利用 `自定义注解`、`Spring Aop`、`Guava Cache` 实现表单防重复提交（`不适用于分布式哦，后面会讲分布式方式...`）

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
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
    <dependency>
        <groupId>com.google.guava</groupId>
        <artifactId>guava</artifactId>
        <version>21.0</version>
    </dependency>
</dependencies>
```

## Lock 注解

创建一个 `LocalLock` 注解，简单点就一个 `key` 可以了，由于暂时未用到 `redis` 所以 `expire` 是摆设….

```java
package com.battcn.annotation;

import java.lang.annotation.*;

/**
 * 锁的注解
 *
 * @author Levin
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
public @interface LocalLock {

    /**
     * @author fly
     */
    String key() default "";

    /**
     * 过期时间 TODO 由于用的 guava 暂时就忽略这属性吧 集成 redis 需要用到
     *
     * @author fly
     */
    int expire() default 5;
}
```

## Lock 拦截器（AOP）

首先通过 `CacheBuilder.newBuilder()` 构建出缓存对象，设置好过期时间；其目的就是为了防止因程序崩溃锁得不到释放（当然如果单机这种方式程序都炸了，锁早没了；但这不妨碍我们写好点）

在具体的 `interceptor()` 方法上采用的是 `Around（环绕增强）` ，所有带 `LocalLock` 注解的都将被切面处理；

如果想更为灵活，key 的生成规则可以定义成接口形式（**可以参考：org.springframework.cache.interceptor.KeyGenerator**），这里就偷个懒了；

```java
package com.battcn.interceptor;

import com.battcn.annotation.LocalLock;
import com.google.common.cache.Cache;
import com.google.common.cache.CacheBuilder;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.context.annotation.Configuration;
import org.springframework.util.StringUtils;

import java.lang.reflect.Method;
import java.util.concurrent.TimeUnit;

/**
 * 本章先基于 本地缓存来做，后续讲解 redis 方案
 *
 * @author Levin
 * @since 2018/6/12 0012
 */
@Aspect
@Configuration
public class LockMethodInterceptor {

    private static final Cache<String, Object> CACHES = CacheBuilder.newBuilder()
            // 最大缓存 100 个
            .maximumSize(1000)
            // 设置写缓存后 5 秒钟过期
            .expireAfterWrite(5, TimeUnit.SECONDS)
            .build();

    @Around("execution(public * *(..)) && @annotation(com.battcn.annotation.LocalLock)")
    public Object interceptor(ProceedingJoinPoint pjp) {
        MethodSignature signature = (MethodSignature) pjp.getSignature();
        Method method = signature.getMethod();
        LocalLock localLock = method.getAnnotation(LocalLock.class);
        String key = getKey(localLock.key(), pjp.getArgs());
        if (!StringUtils.isEmpty(key)) {
            if (CACHES.getIfPresent(key) != null) {
                throw new RuntimeException("请勿重复请求");
            }
            // 如果是第一次请求,就将 key 当前对象压入缓存中
            CACHES.put(key, key);
        }
        try {
            return pjp.proceed();
        } catch (Throwable throwable) {
            throw new RuntimeException("服务器异常");
        } finally {
            // TODO 为了演示效果,这里就不调用 CACHES.invalidate(key); 代码了
        }
    }

    /**
     * key 的生成策略,如果想灵活可以写成接口与实现类的方式（TODO 后续讲解）
     *
     * @param keyExpress 表达式
     * @param args       参数
     * @return 生成的key
     */
    private String getKey(String keyExpress, Object[] args) {
        for (int i = 0; i < args.length; i++) {
            keyExpress = keyExpress.replace("arg[" + i + "]", args[i].toString());
        }
        return keyExpress;
    }
}
```

## 控制层

在接口上添加 `@LocalLock(key = "book:arg[0]")`；意味着会将 `arg[0]` 替换成第一个参数的值，生成后的新 key 将被缓存起来；

```java
package com.battcn.controller;

import com.battcn.annotation.LocalLock;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

/**
 * BookController
 *
 * @author Levin
 * @since 2018/6/06 0031
 */
@RestController
@RequestMapping("/books")
public class BookController {

    @LocalLock(key = "book:arg[0]")
    @GetMapping
    public String query(@RequestParam String token) {
        return "success - " + token;
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
public class Chapter21Application {

    public static void main(String[] args) {

        SpringApplication.run(Chapter21Application.class, args);

    }
}
```

## 测试

完成准备事项后，启动 `Chapter21Application` 自行测试即可，测试手段相信大伙都不陌生了，如 `浏览器`、`postman`、`junit`、`swagger`，此处基于 `postman`，如果你觉得自带的异常信息不够友好，那么配上[一起来学SpringBoot | 第十八篇：轻松搞定全局异常](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-exception/) 可以轻松搞定…

> 第一次请求

[![正确响应](https://image.battcn.com/article/images/20180612/springboot/v2-cache-locallock/1.png)](https://image.battcn.com/article/images/20180612/springboot/v2-cache-locallock/1.png)

> 第二次请求

[![错误响应](https://image.battcn.com/article/images/20180612/springboot/v2-cache-locallock/2.png)](https://image.battcn.com/article/images/20180612/springboot/v2-cache-locallock/2.png)

# 总结

目前很多大佬都写过关于 **SpringBoot** 的教程了，如有雷同，请多多包涵，本教程基于最新的 `spring-boot-starter-parent：2.0.2.RELEASE`编写，包括新版本的特性都会一起介绍…

# 说点什么

全文代码：<https://github.com/battcn/spring-boot2-learning/tree/master/chapter21>

# 666. 彩蛋

如果你对 SpringBoot 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)