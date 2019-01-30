title: 一起来学 SpringBoot 2.x | 第十篇：使用 Spring Cache 集成 Redis
date: 2018-01-11
tag: 
categories: Spring Boot
permalink: Spring-Boot/battcn/v2-cache-redis/
author: 唐亚峰
from_url: http://blog.battcn.com/2018/05/13/springboot/v2-cache-redis/
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247485513&idx=2&sn=1b6d52519d6c15d18f2cac75a7e54309&chksm=fa4977f8cd3efeee7ccb94567404b68493c43473d20122abdfad06edeebcc821096300ca9867&token=696637778&lang=zh_CN#rd

-------

摘要: 原创出处 http://blog.battcn.com/2018/05/13/springboot/v2-cache-redis/ 「唐亚峰」欢迎转载，保留摘要，谢谢！

- [特点](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redis//)
- [使用前后](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redis//)
- [添加依赖](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redis//)
- [属性配置](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redis//)
- [具体编码](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redis//)
  - [实体类](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redis//)
  - [定义接口](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redis//)
  - [主函数](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redis//)
  - [测试](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redis//)
  - [根据条件操作缓存](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redis//)
  - [注解介绍](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redis//)
- [总结](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redis//)
- [说点什么](http://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redis//)

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

`Spring 3.1` 引入了激动人心的**基于注释（annotation）的缓存（cache）技术**，它本质上不是一个具体的缓存实现方案（例如 `EHCache` 或者 `Redis`），而是一个对缓存使用的抽象，通过在既有代码中添加少量它定义的各种 `annotation`，即能够达到缓存方法的返回对象的效果。

# 1. 特点

具备相当的好的灵活性，不仅能够使用 **SpEL（Spring Expression Language）**来定义缓存的 key 和各种 condition，还提供开箱即用的缓存临时存储方案，也支持和主流的专业缓存例如 EHCache、Redis、Guava 的集成。

- **基于 annotation 即可使得现有代码支持缓存**
- **开箱即用 Out-Of-The-Box，不用安装和部署额外第三方组件即可使用缓存**
- **支持 Spring Express Language，能使用对象的任何属性或者方法来定义缓存的 key 和 condition**
- **支持 AspectJ，并通过其实现任何方法的缓存支持**
- **支持自定义 key 和自定义缓存管理者，具有相当的灵活性和扩展性**

# 2. 使用前后

下面针对`Spring Cache`使用前后给出了伪代码部分，具体中也许比这要更加复杂，但是`Spring Cache`都可以很好的应对

> 使用前

我们需要硬编码，如果切换`Cache Client`还需要修改代码，耦合度高，不易于维护

```java
public String get(String key) {
    String value = userMapper.selectById(key);
    if (value != null) {
        cache.put(key,value);
    }
    return value;
}
```

> 使用后

基于`Spring Cache`注解，缓存由开发者自己配置，但不用参与到具体编码

```java
@Cacheable(value = "user", key = "#key")
public String get(String key) {
    return userMapper.selectById(key);
}
```

# 3. 添加依赖

在 `pom.xml` 中添加 `spring-boot-starter-data-redis`的依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

# 4. 属性配置

在 `application.properties` 文件中配置如下内容，由于`Spring Boot2.x` 的改动，连接池相关配置需要通过`spring.redis.lettuce.pool`或者 `spring.redis.jedis.pool` 进行配置了。使用了`Spring Cache`后，能指定`spring.cache.type`就手动指定一下，虽然它会自动去适配已有`Cache`的依赖，但先后顺序会对`Redis`使用有影响**（JCache -> EhCache -> Redis -> Guava）**

```properties
spring.redis.host=localhost
spring.redis.password=battcn
# 一般来说是不用配置的，Spring Cache 会根据依赖的包自行装配
spring.cache.type=redis
# 连接超时时间（毫秒）
spring.redis.timeout=10000
# Redis默认情况下有16个分片，这里配置具体使用的分片
spring.redis.database=0
# 连接池最大连接数（使用负值表示没有限制） 默认 8
spring.redis.lettuce.pool.max-active=8
# 连接池最大阻塞等待时间（使用负值表示没有限制） 默认 -1
spring.redis.lettuce.pool.max-wait=-1
# 连接池中的最大空闲连接 默认 8
spring.redis.lettuce.pool.max-idle=8
# 连接池中的最小空闲连接 默认 0
spring.redis.lettuce.pool.min-idle=0
```

# 5. 具体编码

## 5.1 实体类

创建一个`User`类，目的是为了模拟对象存储

```java
package com.battcn.entity;

import java.io.Serializable;

/**
 * @author Levin
 * @since 2018/5/11 0007
 */
public class User implements Serializable {

    private static final long serialVersionUID = 8655851615465363473L;
    private Long id;
    private String username;
    private String password;
    // TODO  省略get set
}
```

## 5.2 定义接口

```java
package com.battcn.service;

import com.battcn.entity.User;

/**
 * @author Levin
 * @since 2018/5/11 0011
 */
public interface UserService {

    /**
     * 删除
     *
     * @param user 用户对象
     * @return 操作结果
     */
    User saveOrUpdate(User user);

    /**
     * 添加
     *
     * @param id key值
     * @return 返回结果
     */
    User get(Long id);

    /**
     * 删除
     *
     * @param id key值
     */
    void delete(Long id);
}
```

### 5.2.1 实现类

为了方便演示数据库操作，直接定义了一个`Map<Long, User> DATABASES`，这里的核心就是**@Cacheable、@CachePut、@CacheEvict** 三个注解

```java
package com.battcn.service.impl;

import com.battcn.entity.User;
import com.battcn.service.UserService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

import java.util.HashMap;
import java.util.Map;

/**
 * @author Levin
 * @since 2018/5/11 0011
 */
@Service
public class UserServiceImpl implements UserService {

    private static final Map<Long, User> DATABASES = new HashMap<>();

    static {
        DATABASES.put(1L, new User(1L, "u1", "p1"));
        DATABASES.put(2L, new User(2L, "u2", "p2"));
        DATABASES.put(3L, new User(3L, "u3", "p3"));
    }


    private static final Logger log = LoggerFactory.getLogger(UserServiceImpl.class);

    @Cacheable(value = "user", key = "#id")
    @Override
    public User get(Long id) {
        // TODO 我们就假设它是从数据库读取出来的
        log.info("进入 get 方法");
        return DATABASES.get(id);
    }

    @CachePut(value = "user", key = "#user.id")
    @Override
    public User saveOrUpdate(User user) {
        DATABASES.put(user.getId(), user);
        log.info("进入 saveOrUpdate 方法");
        return user;
    }

    @CacheEvict(value = "user", key = "#id")
    @Override
    public void delete(Long id) {
        DATABASES.remove(id);
        log.info("进入 delete 方法");
    }
}
```

## 5.3 主函数

**@EnableCaching** 必须要加，否则`spring-data-cache`相关注解不会生效…

```java
package com.battcn;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;

/**
 * @author Levin
 */
@SpringBootApplication
@EnableCaching
public class Chapter9Application {

    public static void main(String[] args) {
        SpringApplication.run(Chapter9Application.class, args);
    }

}
```

## 5.4 测试

完成准备事项后，编写一个`junit`测试类来检验代码的正确性，有很多人质疑过`Redis`线程安全性，故下面也提供了响应的测试案例，如有疑问欢迎指正

```java
package com.battcn;

import com.battcn.entity.User;
import com.battcn.service.UserService;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

/**
 * @author Levin
 * @since 2018/5/10 0010
 */
@RunWith(SpringRunner.class)
@SpringBootTest
public class Chapter9ApplicationTest {

    private static final Logger log = LoggerFactory.getLogger(Chapter9ApplicationTest.class);


    @Autowired
    private UserService userService;


    @Test
    public void get() {
        final User user = userService.saveOrUpdate(new User(5L, "u5", "p5"));
        log.info("[saveOrUpdate] - [{}]", user);
        final User user1 = userService.get(5L);
        log.info("[get] - [{}]", user1);
        userService.delete(5L);
    }
}
```

启动测试类，结果和我们期望的一致，可以看到增删改查中，**查询是没有日志输出的，因为它直接从缓存中获取的数据，而添加、修改、删除都是会进入方法内执行具体的业务代码，然后通过切面去删除掉Redis中的缓存数据。其中 # 号代表这是一个 SpEL 表达式，此表达式可以遍历方法的参数对象，具体语法可以参考 Spring 的相关文档手册。**

```shell
2018-05-14 09:20:55.303  INFO 21176 --- [           main] com.battcn.service.impl.UserServiceImpl  : 进入 saveOrUpdate 方法
2018-05-14 09:20:55.582  INFO 21176 --- [           main] io.lettuce.core.EpollProvider            : Starting without optional epoll library
2018-05-14 09:20:55.584  INFO 21176 --- [           main] io.lettuce.core.KqueueProvider           : Starting without optional kqueue library
2018-05-14 09:20:56.316  INFO 21176 --- [           main] com.battcn.Chapter9ApplicationTest       : [saveOrUpdate] - [User{id=5, username='u5', password='p5'}]
2018-05-14 09:20:56.320  INFO 21176 --- [           main] com.battcn.Chapter9ApplicationTest       : [get] - [User{id=5, username='u5', password='p5'}]
2018-05-14 09:20:56.322  INFO 21176 --- [           main] com.battcn.service.impl.UserServiceImpl  : 进入 delete 方法
```

> 其它类型

下列的就是`Redis`其它类型所对应的操作方式

- **opsForValue：** 对应 String（字符串）
- **opsForZSet：** 对应 ZSet（有序集合）
- **opsForHash：** 对应 Hash（哈希）
- **opsForList：** 对应 List（列表）
- **opsForSet：** 对应 Set（集合）
- **opsForGeo：** 对应 GEO（地理位置）

## 5.5 根据条件操作缓存

**根据条件操作缓存内容并不影响数据库操作，条件表达式返回一个布尔值，true/false，当条件为true，则进行缓存操作，否则直接调用方法执行的返回结果。**

- **长度：** `@CachePut(value = "user", key = "#user.id",condition = "#user.username.length() < 10")` 只缓存用户名长度少于10的数据
- **大小：** `@Cacheable(value = "user", key = "#id",condition = "#id < 10")` 只缓存ID小于10的数据
- **组合：** `@Cacheable(value="user",key="#user.username.concat(##user.password)")`
- **提前操作：** `@CacheEvict(value="user",allEntries=true,beforeInvocation=true)` 加上`beforeInvocation=true`后，不管内部是否报错，缓存都将被清除，默认情况为`false`

## 5.6 注解介绍

> @Cacheable(根据方法的请求参数对其结果进行缓存)

- **key：** 缓存的 key，可以为空，如果指定要按照 SpEL 表达式编写，如果不指定，则缺省按照方法的所有参数进行组合（如：`@Cacheable(value="user",key="#userName")`）
- **value：** 缓存的名称，在 Spring 配置文件中定义，必须指定至少一个（如：`@Cacheable(value="user")` 或者
  `@Cacheable(value={"user1","use2"})`）
- **condition：** 缓存的条件，可以为空，使用 SpEL 编写，返回 true 或者 false，只有为 true 才进行缓存（如：`@Cacheable(value = "user", key = "#id",condition = "#id < 10")`）

> @CachePut(根据方法的请求参数对其结果进行缓存，和 `@Cacheable` 不同的是，它每次都会触发真实方法的调用)

- **key：** 同上
- **value：** 同上
- **condition：** 同上

> @CachEvict(根据条件对缓存进行清空)

- **key：** 同上
- **value：** 同上
- **condition：** 同上
- **allEntries：** 是否清空所有缓存内容，缺省为 false，如果指定为 true，则方法调用后将立即清空所有缓存（如：`@CacheEvict(value = "user", key = "#id", allEntries = true)`）
- **beforeInvocation：** 是否在方法执行前就清空，缺省为 false，如果指定为 true，则在方法还没有执行的时候就清空缓存，缺省情况下，如果方法执行抛出异常，则不会清空缓存（如：`@CacheEvict(value = "user", key = "#id", beforeInvocation = true)`）

# 6. 总结

**spring-cache文档：** <https://docs.spring.io/spring/docs/5.0.5.RELEASE/spring-framework-reference/integration.html#cache-introduction>
**spring-data-redis文档：** <https://docs.spring.io/spring-data/redis/docs/2.0.1.RELEASE/reference/html/#new-in-2.0.0>
**Redis 文档：** <https://redis.io/documentation>
**Redis 中文文档：** <http://www.redis.cn/commands.html>

目前很多大佬都写过关于 **SpringBoot** 的教程了，如有雷同，请多多包涵，本教程基于最新的 `spring-boot-starter-parent：2.0.1.RELEASE`编写，包括新版本的特性都会一起介绍…

# 7. 说点什么

全文代码：<https://github.com/battcn/spring-boot2-learning/tree/master/chapter9>

# 666. 彩蛋

如果你对 SpringBoot 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)