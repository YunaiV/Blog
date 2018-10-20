title: 一起来学 SpringBoot 2.x | 第八篇：通用 Mapper 与分页插件的集成
date: 2018-01-09
tag: 
categories: Spring Boot
permalink: Spring-Boot/battcn/v2-orm-mybatis-plugin/
author: 唐亚峰
from_url: http://blog.battcn.com/2018/05/10/springboot/v2-orm-mybatis-plugin/
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247485014&idx=2&sn=ea47f214f476d21e00ba72be9ebf8360&chksm=fa4979e7cd3ef0f110451db3d3b8526464de25331480e138c5579a597ff7bb531a797c9aca94&token=380459847&lang=zh_CN#rd

-------

摘要: 原创出处 http://blog.battcn.com/2018/05/10/springboot/v2-orm-mybatis-plugin/ 「唐亚峰」欢迎转载，保留摘要，谢谢！

- [插件介绍](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-mybatis-plugin//)
- [导入依赖](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-mybatis-plugin//)
- [属性配置](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-mybatis-plugin//)
- [具体编码](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-mybatis-plugin//)
  - [表结构](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-mybatis-plugin//)
  - [实体类](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-mybatis-plugin//)
  - [持久层](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-mybatis-plugin//)
  - [测试](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-mybatis-plugin//)
- [总结](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-mybatis-plugin//)
- [说点什么](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-mybatis-plugin//)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

> SpringBoot` 是为了简化 `Spring` 应用的创建、运行、调试、部署等一系列问题而诞生的产物，**自动装配的特性让我们可以更好的关注业务本身而不是外部的XML配置，我们只需遵循规范，引入相关的依赖就可以轻易的搭建出一个 WEB 工程**

在[一起来学SpringBoot | 第七篇：整合Mybatis](http://www.iocoder.cn/Spring-Boot/battcn/v2-orm-mybatis/)一文中，我们介绍了`Mybatis`这款优秀的框架，顺便提及了民间大神开发的两款插件**（通用Mapper、PageHelper）**，从此告别简单CURD代码的编写….

# 插件介绍

以下两款插件作者均是同一个人，如果你想深入了解`Mybatis`以及插件开发可以购买作者的书籍

[![MyBatis 从入门到精通](https://github.com/mybatis-book/book/raw/master/book.png)](https://github.com/mybatis-book/book/raw/master/book.png)

- **京东：** <https://item.jd.com/12103309.html>
- **当当：** <http://product.dangdang.com/25098208.html>

> 分页插件

- **GIT地址：** <https://github.com/pagehelper/Mybatis-PageHelper>

在没有分页插件之前，写一个分页需要两条SQL语句，一条查询一条统计，然后才能计算出页码，这样的代码冗余而又枯燥，更重要的一点是**数据库迁移**，众所周知不同的数据库分页写法是不同的，而`Mybatis`不同于`Hibernate`的是它只提供动态SQL和结果集映射。值得庆幸的是，它虽然没有为分页提供良好的解决方案，但却提供了`Interceptor`以供开发者自己扩展，这也是这款分页插件的由来….

> 通用Mapper

- **GIT地址：** <https://gitee.com/free/Mapper>

**通用 Mapper 是一个可以实现任意 MyBatis 通用方法的框架**，项目提供了常规的增删改查操作以及 `Example` 相关的单表操作。通用 Mapper 是为了解决 MyBatis 使用中 90% 的基本操作，使用它可以很方便的进行开发，可以节省开发人员大量的时间。

# 导入依赖

在 `pom.xml` 中添加通用Mapper与分页插件的依赖包

```xml
<!-- 通用Mapper插件
 文档地址：https://gitee.com/free/Mapper/wikis/Home -->
<dependency>
    <groupId>tk.mybatis</groupId>
    <artifactId>mapper-spring-boot-starter</artifactId>
    <version>2.0.2</version>
</dependency>
<!-- 分页插件
 文档地址：https://github.com/pagehelper/Mybatis-PageHelper/blob/master/wikis/zh/HowToUse.md -->
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper-spring-boot-starter</artifactId>
    <version>1.2.5</version>
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

# 属性配置

在 `application.properties` 文件中分别添加上**数据库、Mybatis、通用Mapper、PageHelper**的属性配置，这里只提供了常见场景的配置，更全的配置可以参考上文所述的**文文档(#^.^#)**

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/chapter7?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull&allowMultiQueries=true&useSSL=false
spring.datasource.password=root
spring.datasource.username=root
# 如果想看到mybatis日志需要做如下配置
logging.level.com.battcn=DEBUG
######### Mybatis 自身配置 ##########
mybatis.mapper-locations=classpath:com/battcn/mapper/*.xml
mybatis.type-aliases-package=com.battcn.entity
# 驼峰命名规范 如：数据库字段是  order_id 那么 实体字段就要写成 orderId
mybatis.configuration.map-underscore-to-camel-case=true
######### 通用Mapper ##########
# 主键自增回写方法,默认值MYSQL,详细说明请看文档
mapper.identity=MYSQL
mapper.mappers=tk.mybatis.mapper.common.BaseMapper
# 设置 insert 和 update 中，是否判断字符串类型!=''
mapper.not-empty=true
# 枚举按简单类型处理
mapper.enum-as-simple-type=true
######### 分页插件 ##########
pagehelper.helper-dialect=mysql
pagehelper.params=count=countSql
pagehelper.reasonable=false
pagehelper.support-methods-arguments=true
```

> 通用Mapper

- **mapper.enum-as-simple-type：** 枚举按简单类型处理，如果有枚举字段则需要加上该配置才会做映射
- **mapper.not-empty：** 设置以后，会去判断 insert 和 update 中符串类型!=’’

> 分页插件

- **pagehelper.reasonable：** 分页合理化参数，默认值为false。当该参数设置为 true 时，pageNum<=0 时会查询第一页， pageNum>pages（超过总数时），会查询最后一页。默认false 时，直接根据参数进行查询。
- **support-methods-arguments：** 支持通过 Mapper 接口参数来传递分页参数，默认值false，分页插件会从查询方法的参数值中，自动根据上面 params 配置的字段中取值，查找到合适的值时就会自动分页。

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

```sql
CREATE TABLE `t_user` (
  `id` int(8) NOT NULL AUTO_INCREMENT COMMENT '主键自增',
  `username` varchar(50) NOT NULL COMMENT '用户名',
  `password` varchar(50) NOT NULL COMMENT '密码',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='用户表';
```

## 实体类

`通用Mapper`采用了JPA规范包中的注解，这种的设计避免了重复造轮子，更是让`Spring Data Jpa`的应用可以轻松切换到`Mybatis`

```jAVA
package com.battcn.entity;

import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.persistence.Table;
import java.io.Serializable;

/**
 * @author Levin
 * @since 2018/5/10 0007
 */
@Table(name = "t_user")
public class User implements Serializable {

    private static final long serialVersionUID = 8655851615465363473L;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String username;
    private String password;
    // TODO  省略get set
}
```

## 持久层

为了更好的让熟悉它，此处模拟了一个自定义的SQL，可以发现使用 `通用Mapper` 后并不会破坏原有代码结构

> UserMapper

继承 `BaseMapper<T>` 就可以了，这点是不是有点类似 `JpaRepository`，同时也可以根据自己需要扩展出更适合自己项目的`BaseMapper`，它的灵活也是众多开发者喜爱的因素之一

```
package com.battcn.mapper;

import com.battcn.entity.User;
import org.apache.ibatis.annotations.Mapper;
import tk.mybatis.mapper.common.BaseMapper;

/**
 * t_user 操作，继承 BaseMapper<T> 就可以了，是不是有点类似 JpaRepository
 *
 * @author Levin
 * @since 2018/5/10 0007
 */
@Mapper
public interface UserMapper extends BaseMapper<User> {

    /**
     * 根据用户名统计（TODO 假设它是一个很复杂的SQL）
     *
     * @param username 用户名
     * @return 统计结果
     */
    int countByUsername(String username);
}
```

> `UserMapper` 映射文件

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.battcn.mapper.UserMapper">

  <select id="countByUsername" resultType="java.lang.Integer">
      SELECT count(1) FROM t_user WHERE username = #{username}
  </select>
</mapper>
```

## 测试

完成数据访问层接口后，编写一个`junit`测试类来检验代码的正确性。

```java
package com.battcn;

import com.battcn.entity.User;
import com.battcn.mapper.UserMapper;
import com.github.pagehelper.PageHelper;
import com.github.pagehelper.PageInfo;
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
public class Chapter7ApplicationTests {

    private static final Logger log = LoggerFactory.getLogger(Chapter7ApplicationTests.class);

    @Autowired
    private UserMapper userMapper;

    @Test
    public void test1() throws Exception {
        final User user1 = new User("u1", "p1");
        final User user2 = new User("u1", "p2");
        final User user3 = new User("u3", "p3");
        userMapper.insertSelective(user1);
        log.info("[user1回写主键] - [{}]", user1.getId());
        userMapper.insertSelective(user2);
        log.info("[user2回写主键] - [{}]", user2.getId());
        userMapper.insertSelective(user3);
        log.info("[user3回写主键] - [{}]", user3.getId());
        final int count = userMapper.countByUsername("u1");
        log.info("[调用自己写的SQL] - [{}]", count);

        // TODO 模拟分页
        userMapper.insertSelective(new User("u1", "p1"));
        userMapper.insertSelective(new User("u1", "p1"));
        userMapper.insertSelective(new User("u1", "p1"));
        userMapper.insertSelective(new User("u1", "p1"));
        userMapper.insertSelective(new User("u1", "p1"));
        userMapper.insertSelective(new User("u1", "p1"));
        userMapper.insertSelective(new User("u1", "p1"));
        userMapper.insertSelective(new User("u1", "p1"));
        userMapper.insertSelective(new User("u1", "p1"));
        userMapper.insertSelective(new User("u1", "p1"));
        // TODO 分页 + 排序 this.userMapper.selectAll() 这一句就是我们需要写的查询，有了这两款插件无缝切换各种数据库
        final PageInfo<Object> pageInfo = PageHelper.startPage(1, 10).setOrderBy("id desc").doSelectPageInfo(() -> this.userMapper.selectAll());
        log.info("[lambda写法] - [分页信息] - [{}]", pageInfo.toString());

        PageHelper.startPage(1, 10).setOrderBy("id desc");
        final PageInfo<User> userPageInfo = new PageInfo<>(this.userMapper.selectAll());
        log.info("[普通写法] - [{}]", userPageInfo);
    }
}
```

# 总结

1. **Mybatis官方文档：** <http://www.mybatis.org/mybatis-3/zh/index.html>
2. **通用Mapper文档：** <https://gitee.com/free/Mapper/wikis/1.1-java?parent=1.integration>
3. **分页插件文档：** <https://github.com/pagehelper/Mybatis-PageHelper/blob/master/wikis/zh/HowToUse.md>

目前很多大佬都写过关于 **SpringBoot** 的教程了，如有雷同，请多多包涵，本教程基于最新的 `spring-boot-starter-parent：2.0.1.RELEASE`编写，包括新版本的特性都会一起介绍…

# 说点什么

全文代码：<https://github.com/battcn/spring-boot2-learning/tree/master/chapter7>

# 666. 彩蛋

如果你对 SpringBoot 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)