title: 一起来学 SpringBoot 2.x | 第五篇：使用 JdbcTemplate 访问数据库
date: 2018-01-06
tag: 
categories: Spring Boot
permalink: Spring-Boot/battcn/v2-orm-jdbc/
author: 唐亚峰
from_url: http://blog.battcn.com/2018/05/07/springboot/v2-orm-jdbc/
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484785&idx=1&sn=2f81d867008a0f5fdb312d2d2509e943&chksm=fa497ac0cd3ef3d6321ecd627b646972d6ee130e58037a531876c4e0f995619798f0366b2e4c#rd

-------

摘要: 原创出处 http://blog.battcn.com/2018/05/07/springboot/v2-orm-jdbc/ 「唐亚峰」欢迎转载，保留摘要，谢谢！

- [导入依赖](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-jdbc//)
- [连接数据库](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-jdbc//)
- [具体编码](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-jdbc//)
  - [表结构](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-jdbc//)
  - [实体类](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-jdbc//)
  - [restful 风格接口](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-jdbc//)
  - [测试](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-jdbc//)
- [总结](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-jdbc//)
- [说点什么](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-jdbc//)

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

`Spring Framework`对数据库的操作在`JDBC`上面做了深层次的封装，通过`依赖注入`功能，可以将 `DataSource` 注册到`JdbcTemplate`之中，使我们可以轻易的完成对象关系映射，并有助于规避常见的错误，在`SpringBoot`中我们可以很轻松的使用它。

> 特点

- 速度快，对比其它的ORM框架而言，JDBC的方式无异于是最快的
- 配置简单，`Spring`自家出品，几乎没有额外配置
- 学习成本低，毕竟`JDBC`是基础知识，`JdbcTemplate`更像是一个`DBUtils`

# 导入依赖

在 `pom.xml` 中添加对 `JdbcTemplate` 的依赖

```xml
<!-- Spring JDBC 的依赖包，使用 spring-boot-starter-jdbc 或 spring-boot-starter-data-jpa 将会自动获得HikariCP依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
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
```

# 连接数据库

在`application.properties`中添加如下配置。值得注意的是，SpringBoot默认会自动配置`DataSource`，它将优先采用`HikariCP`连接池，如果没有该依赖的情况则选取`tomcat-jdbc`，如果前两者都不可用最后选取`Commons DBCP2`。**通过spring.datasource.type属性可以指定其它种类的连接池**

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/chapter4?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull&allowMultiQueries=true&useSSL=false
spring.datasource.password=root
spring.datasource.username=root
#spring.datasource.type
#更多细微的配置可以通过下列前缀进行调整
#spring.datasource.hikari
#spring.datasource.tomcat
#spring.datasource.dbcp2
```

启动项目，通过日志，可以看到默认情况下注入的是`HikariDataSource`

```shell
2018-05-07 10:33:54.021  INFO 9640 --- [           main] o.s.j.e.a.AnnotationMBeanExporter        : Bean with name 'dataSource' has been autodetected for JMX exposure
2018-05-07 10:33:54.026  INFO 9640 --- [           main] o.s.j.e.a.AnnotationMBeanExporter        : Located MBean 'dataSource': registering with JMX server as MBean [com.zaxxer.hikari:name=dataSource,type=HikariDataSource]
2018-05-07 10:33:54.071  INFO 9640 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2018-05-07 10:33:54.075  INFO 9640 --- [           main] com.battcn.Chapter4Application           : Started Chapter4Application in 3.402 seconds (JVM running for 3.93)
```

# 具体编码

完成基本配置后，接下来进行具体的编码操作。**为了减少代码量，就不写UserDao、UserService之类的接口了，将直接在Controller中使用JdbcTemplate进行访问数据库操作，这点是不规范的，各位别学我…**

## 表结构

创建一张 `t_user` 的表

```sql
CREATE TABLE `t_user` (
  `id` int(8) NOT NULL AUTO_INCREMENT COMMENT '主键自增',
  `username` varchar(50) NOT NULL COMMENT '用户名',
  `password` varchar(50) NOT NULL COMMENT '密码',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='用户表';
```

## 实体类

```Java
package com.battcn.entity;

/**
 * @author Levin
 * @since 2018/5/7 0007
 */
public class User {

    private Long id;
    private String username;
    private String password;
    // TODO  省略get set
}
```

## restful 风格接口

```java
package com.battcn.controller;

import com.battcn.entity.User;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.BeanPropertyRowMapper;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.web.bind.annotation.*;

import java.util.List;

/**
 * @author Levin
 * @since 2018/4/23 0023
 */
@RestController
@RequestMapping("/users")
public class SpringJdbcController {

    private final JdbcTemplate jdbcTemplate;

    @Autowired
    public SpringJdbcController(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @GetMapping
    public List<User> queryUsers() {
        // 查询所有用户
        String sql = "select * from t_user";
        return jdbcTemplate.query(sql, new Object[]{}, new BeanPropertyRowMapper<>(User.class));
    }

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        // 根据主键ID查询
        String sql = "select * from t_user where id = ?";
        return jdbcTemplate.queryForObject(sql, new Object[]{id}, new BeanPropertyRowMapper<>(User.class));
    }

    @DeleteMapping("/{id}")
    public int delUser(@PathVariable Long id) {
        // 根据主键ID删除用户信息
        String sql = "DELETE FROM t_user WHERE id = ?";
        return jdbcTemplate.update(sql, id);
    }

    @PostMapping
    public int addUser(@RequestBody User user) {
        // 添加用户
        String sql = "insert into t_user(username, password) values(?, ?)";
        return jdbcTemplate.update(sql, user.getUsername(), user.getPassword());
    }


    @PutMapping("/{id}")
    public int editUser(@PathVariable Long id, @RequestBody User user) {
        // 根据主键ID修改用户信息
        String sql = "UPDATE t_user SET username = ? ,password = ? WHERE id = ?";
        return jdbcTemplate.update(sql, user.getUsername(), user.getPassword(), id);
    }
}
```

## 测试

由于上面的接口是 restful 风格的接口，添加和修改无法通过浏览器完成，所以需要我们自己编写`junit`或者使用`postman`之类的工具。

创建单元测试`Chapter4ApplicationTests`，通过`TestRestTemplate`模拟**GET、POST、PUT、DELETE等请求操作**

```java
package com.battcn;

import com.battcn.entity.User;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.boot.web.server.LocalServerPort;
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.http.HttpMethod;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.junit4.SpringRunner;

import java.util.List;

/**
 * @author Levin
 */
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Chapter4Application.class, webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class Chapter4ApplicationTests {

    private static final Logger log = LoggerFactory.getLogger(Chapter4ApplicationTests.class);
    @Autowired
    private TestRestTemplate template;
    @LocalServerPort
    private int port;

    @Test
    public void test1() throws Exception {
        template.postForEntity("http://localhost:" + port + "/users", new User("user1", "pass1"), Integer.class);
        log.info("[添加用户成功]\n");
        // TODO 如果是返回的集合,要用 exchange 而不是 getForEntity ，后者需要自己强转类型
        ResponseEntity<List<User>> response2 = template.exchange("http://localhost:" + port + "/users", HttpMethod.GET, null, new ParameterizedTypeReference<List<User>>() {
        });
        final List<User> body = response2.getBody();
        log.info("[查询所有] - [{}]\n", body);
        Long userId = body.get(0).getId();
        ResponseEntity<User> response3 = template.getForEntity("http://localhost:" + port + "/users/{id}", User.class, userId);
        log.info("[主键查询] - [{}]\n", response3.getBody());
        template.put("http://localhost:" + port + "/users/{id}", new User("user11", "pass11"), userId);
        log.info("[修改用户成功]\n");
        template.delete("http://localhost:" + port + "/users/{id}", userId);
        log.info("[删除用户成功]");
    }
}
```

# 总结

本章介绍了`JdbcTemplate`常用的几种操作，详细请参考[JdbcTemplate API文档](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/jdbc/core/JdbcTemplate.html)

目前很多大佬都写过关于 **SpringBoot** 的教程了，如有雷同，请多多包涵，本教程基于最新的 `spring-boot-starter-parent：2.0.1.RELEASE`编写，包括新版本的特性都会一起介绍…

# 说点什么

全文代码：<https://github.com/battcn/spring-boot2-learning/tree/master/chapter4>

# 666. 彩蛋

如果你对 SpringBoot 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)