title: 一起来学 SpringBoot 2.x | 第七篇：整合 Mybatis
date: 2018-01-08
tag: 
categories: Spring Boot
permalink: Spring-Boot/battcn/v2-orm-mybatis/
author: 唐亚峰
from_url: http://blog.battcn.com/2018/05/09/springboot/v2-orm-mybatis/
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484976&idx=1&sn=518c86dbe9fa4f289ce43c56856797da&chksm=fa497981cd3ef097d6d0d8da5a2864ba9a877c4735e825e05626ec71834514640f39eef3f553&token=1460889491&lang=zh_CN#rd

-------

摘要: 原创出处 http://blog.battcn.com/2018/05/09/springboot/v2-orm-mybatis/ 「唐亚峰」欢迎转载，保留摘要，谢谢！

- [ORM对比图](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-mybatis//)
- [导入依赖](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-mybatis//)
- [连接数据库](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-mybatis//)
- [注意注意](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-mybatis//)
- [具体编码](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-mybatis//)
  - [表结构](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-mybatis//)
  - [实体类](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-mybatis//)
  - [持久层](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-mybatis//)
  - [测试](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-mybatis//)
- [总结](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-mybatis//)
- [说点什么](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-mybatis//)

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

`MyBatis` 是一款优秀的持久层框架，它**支持定制化 SQL、存储过程以及高级映射，几乎避免了所有的 JDBC 代码和手动设置参数以及获取结果集，使用简单的 XML 或注解来配置和映射原生信息，将接口和 Java 的 POJOs(Plain Old Java Objects,普通的 Java对象)映射成数据库中的记录，在国内可谓是占据了半壁江山……**

# ORM对比图

以下针对**Spring JDBC、Spring Data Jpa、Mybatis**三款框架做了个粗略的对比。**一般应用的性能瓶颈并不是在于ORM，所以这三个框架技术选型应该考虑项目的场景、团队的技能掌握情况、开发周期(开发效率)…**

| 框架对比 | Spring JDBC | Spring Data Jpa | Mybatis |
| -------- | ----------- | --------------- | ------- |
| 性能     | 性能最好    | 性能最差        | 居中    |
| 代码量   | 多          | 少              | 多      |
| 学习成本 | 低          | 高              | 居中    |
| 推荐指数 | ❤❤❤         | ❤❤❤❤❤           | ❤❤❤❤❤   |

> **个人观点**

抛开学习成本而言，对于业务简单的中小型项目中使用`Spring Data Jpa` 开发无异于是最快速的。但是鉴于国内市场环境而言，掌握`Mybatis`无异于是佳的选择，低学习成本和动态SQL解耦的特点使得更容易被人们所接受。对于业务复杂且对性能要求较高的项目来说`Mybatis`往往能更好的胜任，可以自己进行SQL优化，同时更让我喜欢的是[Mybatis分页插件](https://gitee.com/free/Mybatis_PageHelper)与[通用Mapper(单表CURD无需自己手写)](https://gitee.com/free/Mapper)有了这两款插件的支持，还有什么理由拒绝`Mybatis`呢

# 导入依赖

在 `pom.xml` 中添加 `Mybatis` 的依赖包`mybatis-spring-boot-starter`，该包拥有自动装配的特点

```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>1.3.2</version>
</dependency>
<!-- MYSQL包 -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
<!-- 默认就内嵌了Tomcat 容器，如需要更换容器也极其简单-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!-- 测试包,当我们使用 mvn package 的时候该包并不会被打入,因为它的生命周期只在 test 之内-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

# 连接数据库

与**SpringDataJpa、Spring JDBC**一样，需要在`application.properties`中添加数据源的配置，同时也需要添加对`mybatis`的配置

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/chapter6?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull&allowMultiQueries=true&useSSL=false
spring.datasource.password=root
spring.datasource.username=root
# 注意注意
mybatis.mapper-locations=classpath:com/battcn/mapper/*.xml
#mybatis.mapper-locations=classpath:mapper/*.xml        #这种方式需要自己在resources目录下创建mapper目录然后存放xml
mybatis.type-aliases-package=com.battcn.entity
# 驼峰命名规范 如：数据库字段是  order_id 那么 实体字段就要写成 orderId
mybatis.configuration.map-underscore-to-camel-case=true
```

**mybatis.configuration.map-underscore-to-camel-case是一个非常好的配置项，合理的命名规范可以让我们省略很多不必要的麻烦，比如xx-mapper.xml中的resultMap的映射可以省略掉了**

> 注意事项

由于 **mybatis.mapper-locations=classpath:com/battcn/mapper/\*.xml**配置的在`java package`中，而`Spring Boot`默认只打入`java package -> *.java`，所以我们需要给`pom.xml`文件添加如下内容

```xml
<build>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
        </resource>
        <resource>
            <directory>src/main/java</directory>
            <includes>
                <include>**/*.xml</include>
            </includes>
            <filtering>true</filtering>
        </resource>
    </resources>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

# 具体编码

完成基本配置后，接下来进行具体的编码操作。

## 表结构

创建一张 `t_user` 的表

```SQL
CREATE TABLE `t_user` (
  `id` int(8) NOT NULL AUTO_INCREMENT COMMENT '主键自增',
  `username` varchar(50) NOT NULL COMMENT '用户名',
  `password` varchar(50) NOT NULL COMMENT '密码',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='用户表';
```

## 实体类

```jAVA
package com.battcn.entity;

import java.io.Serializable;

/**
 * @author Levin
 * @since 2018/5/9 0007
 */
public class User implements Serializable {

    private static final long serialVersionUID = 8655851615465363473L;

    private Long id;
    private String username;
    private String password;
    // TODO  省略get set
}
```

## 持久层

这里提供了两种方式操作接口，第一种带`@Select`注解的是`Mybatis3.x`提供的新特性，同理它还有**@Update、@Delete、@Insert等等一系列注解**，第二种就是传统方式了，写个接口映射，然后在XML中写上我们的SQL语句…

> UserMapper

```java
package com.battcn.mapper;

import com.battcn.entity.User;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;
import org.apache.ibatis.annotations.Select;

import java.util.List;

/**
 * t_user 操作：演示两种方式
 * <p>第一种是基于mybatis3.x版本后提供的注解方式<p/>
 * <p>第二种是早期写法，将SQL写在 XML 中<p/>
 *
 * @author Levin
 * @since 2018/5/7 0007
 */
@Mapper
public interface UserMapper {

    /**
     * 根据用户名查询用户结果集
     *
     * @param username 用户名
     * @return 查询结果
     */
    @Select("SELECT * FROM t_user WHERE username = #{username}")
    List<User> findByUsername(@Param("username") String username);


    /**
     * 保存用户信息
     *
     * @param user 用户信息
     * @return 成功 1 失败 0
     */
    int insert(User user);
}
```

> `UserMapper` 映射文件

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.battcn.mapper.UserMapper">

  <insert id="insert" parameterType="com.battcn.entity.User">
    INSERT INTO `t_user`(`username`,`password`) VALUES (#{username},#{password})
  </insert>

</mapper>
```

## 测试

完成数据访问层接口后，最后编写一个`junit`测试类来检验代码的正确性。

```java
package com.battcn;

import com.battcn.entity.User;
import com.battcn.mapper.UserMapper;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import java.util.List;

/**
 * @author Levin
 */
@RunWith(SpringRunner.class)
@SpringBootTest
public class Chapter6ApplicationTests {

    private static final Logger log = LoggerFactory.getLogger(Chapter6ApplicationTests.class);

    @Autowired
    private UserMapper userMapper;

    @Test
    public void test1() throws Exception {
        final int row1 = userMapper.insert(new User("u1", "p1"));
        log.info("[添加结果] - [{}]", row1);
        final int row2 = userMapper.insert(new User("u2", "p2"));
        log.info("[添加结果] - [{}]", row2);
        final int row3 = userMapper.insert(new User("u1", "p3"));
        log.info("[添加结果] - [{}]", row3);
        final List<User> u1 = userMapper.findByUsername("u1");
        log.info("[根据用户名查询] - [{}]", u1);
    }
}
```

# 总结

更多`Mybatis`的骚操作，请参考[官方文档](http://www.mybatis.org/mybatis-3/zh/index.html)

目前很多大佬都写过关于 **SpringBoot** 的教程了，如有雷同，请多多包涵，本教程基于最新的 `spring-boot-starter-parent：2.0.1.RELEASE`编写，包括新版本的特性都会一起介绍…

# 说点什么

全文代码：<https://github.com/battcn/spring-boot2-learning/tree/master/chapter6>

# 666. 彩蛋

如果你对 SpringBoot 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)