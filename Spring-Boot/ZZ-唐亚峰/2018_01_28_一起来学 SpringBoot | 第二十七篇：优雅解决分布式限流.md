title: 一起来学 SpringBoot 2.x | 第二十七篇：优雅解决分布式限流
date: 2018-01-28
tag: 
categories: Spring Boot
permalink: Spring-Boot/battcn/v2-cache-redislimter/
author: 唐亚峰
from_url: http://blog.battcn.com/2018/08/08/springboot/v2-cache-redislimter/
wechat_url: 

-------

摘要: 原创出处 http://blog.battcn.com/2018/08/08/springboot/v2-cache-redislimter/ 「唐亚峰」欢迎转载，保留摘要，谢谢！

- [分布式限流](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redislimter//)
- [本章目标](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redislimter//)
- [具体代码](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redislimter//)
  - [导入依赖](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redislimter//)
  - [属性配置](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redislimter//)
  - [Limit 注解](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redislimter//)
  - [RedisTemplate](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redislimter//)
  - [Limit 拦截器（AOP）](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redislimter//)
  - [控制层](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redislimter//)
  - [主函数](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redislimter//)
  - [测试](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redislimter//)
- [总结](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redislimter//)
- [666. 彩蛋](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redislimter//)

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

在前面的两篇文章中，介绍了一些限流的类型和策略，本篇从 `Spring Boot`、`Redis` 应用层面来实现分布式的限流….


# 分布式限流

**单机版中我们了解到 AtomicInteger、RateLimiter、Semaphore 这几种解决方案，但它们也仅仅是单机的解决手段，在集群环境下就透心凉了，后面又讲述了 Nginx 的限流手段，可它又属于网关层面的策略之一，并不能解决所有问题。例如供短信接口，你无法保证消费方是否会做好限流控制，所以自己在应用层实现限流还是很有必要的。**

# 本章目标

利用 `自定义注解`、`Spring Aop`、`Redis Cache` 实现分布式限流….

# 具体代码

很简单…

## 导入依赖

在 `pom.xml` 中添加上 `starter-web`、`starter-aop`、`starter-data-redis` 的依赖即可，习惯了使用 `commons-lang3` 和 `guava` 中的一些工具包…

```XML
<dependencies>
    <!-- 默认就内嵌了Tomcat 容器，如需要更换容器也极其简单-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>com.google.guava</groupId>
        <artifactId>guava</artifactId>
        <version>21.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-lang3</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
    </dependency>
</dependencies>
```

## 属性配置

在 `application.properites` 资源文件中添加 `redis` 相关的配置项

```Properties
spring.redis.host=localhost
spring.redis.port=6379
spring.redis.password=battcn
```

## Limit 注解

创建一个 `Limit` 注解，不多说注释都给各位写齐全了….

```Java
package com.battcn.limiter.annotation;


import com.battcn.limiter.LimitType;

import java.lang.annotation.*;

/**
 * 限流
 *
 * @author Levin
 * @since 2018-02-05
 */
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Limit {

    /**
     * 资源的名字
     *
     * @return String
     */
    String name() default "";

    /**
     * 资源的key
     *
     * @return String
     */
    String key() default "";

    /**
     * Key的prefix
     *
     * @return String
     */
    String prefix() default "";

    /**
     * 给定的时间段
     * 单位秒
     *
     * @return int
     */
    int period();

    /**
     * 最多的访问限制次数
     *
     * @return int
     */
    int count();

    /**
     * 类型
     *
     * @return LimitType
     */
    LimitType limitType() default LimitType.CUSTOMER;
}

public enum LimitType {
    /**
     * 自定义key
     */
    CUSTOMER,
    /**
     * 根据请求者IP
     */
    IP;
}
```

## RedisTemplate

默认情况下 `spring-boot-data-redis` 为我们提供了`StringRedisTemplate` 但是满足不了其它类型的转换，所以还是得自己去定义其它类型的模板….

```Java
package com.battcn.limiter;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.io.Serializable;

/**
 * @author Levin
 * @since 2018/8/2 0002
 */
@Configuration
public class RedisLimiterHelper {

    @Bean
    public RedisTemplate<String, Serializable> limitRedisTemplate(LettuceConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Serializable> template = new RedisTemplate<>();
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }
}
```

## Limit 拦截器（AOP）

**熟悉 Redis 的朋友都知道它是线程安全的，我们利用它的特性可以实现分布式锁、分布式限流等组件**，在[一起来学SpringBoot | 第二十三篇：轻松搞定重复提交（分布式锁）](http://blog.battcn.com/2018/06/13/springboot/v2-cache-redislock/)中讲述了分布式锁的实现，限流相比它稍微复杂一点，官方虽然没有提供相应的API，但却提供了支持 Lua 脚本的功能，我们可以通过编写 Lua 脚本实现自己的API，同时他是满足原子性的….

下面核心就是调用 `execute` 方法传入我们的 Lua 脚本内容，然后通过返回值判断是否超出我们预期的范围，超出则给出错误提示。

```Java
package com.battcn.limiter;


import com.battcn.limiter.annotation.Limit;
import com.google.common.collect.ImmutableList;
import org.apache.commons.lang3.StringUtils;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.data.redis.core.script.RedisScript;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import javax.servlet.http.HttpServletRequest;
import java.io.Serializable;
import java.lang.reflect.Method;

/**
 * @author Levin
 * @since 2018/2/5 0005
 */
@Aspect
@Configuration
public class LimitInterceptor {

    private static final Logger logger = LoggerFactory.getLogger(LimitInterceptor.class);

    private final RedisTemplate<String, Serializable> limitRedisTemplate;

    @Autowired
    public LimitInterceptor(RedisTemplate<String, Serializable> limitRedisTemplate) {
        this.limitRedisTemplate = limitRedisTemplate;
    }


    @Around("execution(public * *(..)) && @annotation(com.battcn.limiter.annotation.Limit)")
    public Object interceptor(ProceedingJoinPoint pjp) {
        MethodSignature signature = (MethodSignature) pjp.getSignature();
        Method method = signature.getMethod();
        Limit limitAnnotation = method.getAnnotation(Limit.class);
        LimitType limitType = limitAnnotation.limitType();
        String name = limitAnnotation.name();
        String key;
        int limitPeriod = limitAnnotation.period();
        int limitCount = limitAnnotation.count();
        switch (limitType) {
            case IP:
                key = getIpAddress();
                break;
            case CUSTOMER:
                // TODO 如果此处想根据表达式或者一些规则生成 请看 一起来学Spring Boot | 第二十三篇：轻松搞定重复提交（分布式锁）
                key = limitAnnotation.key();
                break;
            default:
                key = StringUtils.upperCase(method.getName());
        }
        ImmutableList<String> keys = ImmutableList.of(StringUtils.join(limitAnnotation.prefix(), key));
        try {
            String luaScript = buildLuaScript();
            RedisScript<Number> redisScript = new DefaultRedisScript<>(luaScript, Number.class);
            Number count = limitRedisTemplate.execute(redisScript, keys, limitCount, limitPeriod);
            logger.info("Access try count is {} for name={} and key = {}", count, name, key);
            if (count != null && count.intValue() <= limitCount) {
                return pjp.proceed();
            } else {
                throw new RuntimeException("You have been dragged into the blacklist");
            }
        } catch (Throwable e) {
            if (e instanceof RuntimeException) {
                throw new RuntimeException(e.getLocalizedMessage());
            }
            throw new RuntimeException("server exception");
        }
    }

    /**
     * 限流 脚本
     *
     * @return lua脚本
     */
    public String buildLuaScript() {
        StringBuilder lua = new StringBuilder();
        lua.append("local c");
        lua.append("\nc = redis.call('get',KEYS[1])");
        // 调用不超过最大值，则直接返回
        lua.append("\nif c and tonumber(c) > tonumber(ARGV[1]) then");
        lua.append("\nreturn c;");
        lua.append("\nend");
        // 执行计算器自加
        lua.append("\nc = redis.call('incr',KEYS[1])");
        lua.append("\nif tonumber(c) == 1 then");
        // 从第一次调用开始限流，设置对应键值的过期
        lua.append("\nredis.call('expire',KEYS[1],ARGV[2])");
        lua.append("\nend");
        lua.append("\nreturn c;");
        return lua.toString();
    }

    private static final String UNKNOWN = "unknown";

    public String getIpAddress() {
        HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getRequest();
        String ip = request.getHeader("x-forwarded-for");
        if (ip == null || ip.length() == 0 || UNKNOWN.equalsIgnoreCase(ip)) {
            ip = request.getHeader("Proxy-Client-IP");
        }
        if (ip == null || ip.length() == 0 || UNKNOWN.equalsIgnoreCase(ip)) {
            ip = request.getHeader("WL-Proxy-Client-IP");
        }
        if (ip == null || ip.length() == 0 || UNKNOWN.equalsIgnoreCase(ip)) {
            ip = request.getRemoteAddr();
        }
        return ip;
    }
}
```

## 控制层

在接口上添加 `@Limit()` 注解，如下代码会在 Redis 中生成过期时间为 100s 的 key = test 的记录，特意定义了一个 `AtomicInteger` 用作测试…

```Java
package com.battcn.controller;

import com.battcn.limiter.annotation.Limit;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.concurrent.atomic.AtomicInteger;

/**
 * @author Levin
 * @since 2018/8/2 0002
 */
@RestController
public class LimiterController {

    private static final AtomicInteger ATOMIC_INTEGER = new AtomicInteger();

    @Limit(key = "test", period = 100, count = 10)
    @GetMapping("/test")
    public int testLimiter() {
        // 意味著 100S 内最多允許訪問10次
        return ATOMIC_INTEGER.incrementAndGet();
    }
}
```

## 主函数

就一个普通的不能在普通的主函数类了

```Java
package com.battcn;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * @author Levin
 */
@SpringBootApplication
public class Chapter27Application {

    public static void main(String[] args) {
        SpringApplication.run(Chapter27Application.class, args);
    }
}
```

## 测试

完成准备事项后，启动 `Chapter27Application` 自行测试即可，测试手段相信大伙都不陌生了，如 `浏览器`、`postman`、`junit`、`swagger`，此处基于 `postman`，如果你觉得自带的异常信息不够友好，那么配上[一起来学SpringBoot | 第十八篇：轻松搞定全局异常](http://blog.battcn.com/2018/06/01/springboot/v2-other-exception/) 可以轻松搞定…

> 未达设定的阀值时

[![正确响应](https://image.battcn.com/article/images/20180802/springboot/v2-cache-limiter/1.png)](https://image.battcn.com/article/images/20180802/springboot/v2-cache-limiter/1.png)正确响应

> 达到设置的阀值时

[![错误响应](https://image.battcn.com/article/images/20180802/springboot/v2-cache-limiter/2.png)](https://image.battcn.com/article/images/20180802/springboot/v2-cache-limiter/2.png)错误响应

# 总结

目前很多大佬都写过关于 **Spring Boot** 的教程了，如有雷同，请多多包涵，本教程基于最新的 `spring-boot-starter-parent：2.0.3.RELEASE`编写…

本篇文章核心的 Lua 脚本截取自军哥的 [Aquarius](https://github.com/Nepxion/Aquarius) 开源项目，有兴趣的朋友可以 fork star ，该项目干货满满…

全文代码：https://github.com/battcn/spring-boot2-learning/tree/master/chapter27

# 666. 彩蛋

如果你对 SpringBoot 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)