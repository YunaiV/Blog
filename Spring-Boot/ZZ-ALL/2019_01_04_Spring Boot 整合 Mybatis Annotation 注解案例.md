title: Spring Boot 整合 Mybatis Annotation 注解案例
date: 2019-01-04
tag:
categories: Spring Boot
permalink: Spring-Boot/byscoket/spring-boot-mybatis-annotations-in-action
author: 泥瓦匠
from_url: http://www.bysocket.com
wechat_url:

-------

摘要: 原创出处 http://www.bysocket.com 「泥瓦匠」欢迎转载，保留摘要，谢谢！

- [一、前言](http://www.iocoder.cn/Spring-Boot/byscoket/spring-boot-mybatis-annotations-in-action/)
- [二、运行 springboot-mybatis-annotation 工程](http://www.iocoder.cn/Spring-Boot/byscoket/spring-boot-mybatis-annotations-in-action/)
- [三、springboot-mybatis-annotation 工程配置详解](http://www.iocoder.cn/Spring-Boot/byscoket/spring-boot-mybatis-annotations-in-action/)
- [四、小结](http://www.iocoder.cn/Spring-Boot/byscoket/spring-boot-mybatis-annotations-in-action/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

**运行环境**：JDK 7 或 8、Maven 3.0+
**技术栈**：SpringBoot 1.5+、SpringBoot Mybatis Starter 1.2+ 、MyBatis 3.4+

# 一、前言

距离第一篇 Spring Boot 系列的博文 3 个月了。《Springboot 整合 Mybatis 的完整 Web 案例》第一篇出来是 XML 配置 SQL 的形式。虽然 XML 形式是我比较推荐的，但是注解形式也是方便的。尤其一些小系统，快速的 CRUD 轻量级的系统。

这里感谢晓春 <http://xchunzhao.tk/> 的 Pull Request，提供了 springboot-mybatis-annotation 的实现。

# 二、运行 springboot-mybatis-annotation 工程

由于这篇文章和 《[Springboot 整合 Mybatis 的完整 Web 案例](http://www.iocoder.cn/Spring-Boot/bysocket/spring-boot-mybatis-with-web-in-action?self)》 类似，所以运行这块环境配置大家参考另外一篇兄弟文章。

然后Application 应用启动类的 main 函数，然后在浏览器访问：

```Shell
http://localhost:8080/api/city?cityName=温岭市
```

可以看到返回的 JSON 结果：

```Json
{
  "id": 1,
  "provinceId": 1,
  "cityName": "温岭市",
  "description": "我的家在温岭。"
}
```

# 三、springboot-mybatis-annotation 工程配置详解

1.pom 添加 Mybatis 依赖

```Xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>springboot</groupId>
    <artifactId>springboot-mybatis-annotation</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>springboot-mybatis-annotation</name>
    <description>Springboot-mybatis :: 整合Mybatis Annotation Demo</description>

    <!-- Spring Boot 启动父依赖 -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.1.RELEASE</version>
    </parent>

    <properties>
        <mybatis-spring-boot>1.2.0</mybatis-spring-boot>
        <mysql-connector>5.1.39</mysql-connector>
    </properties>

    <dependencies>

        <!-- Spring Boot Web 依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Spring Boot Test 依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <!-- Spring Boot Mybatis 依赖 -->
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>${mybatis-spring-boot}</version>
        </dependency>

        <!-- MySQL 连接驱动依赖 -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>${mysql-connector}</version>
        </dependency>

        <!-- Junit -->
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
        </dependency>
    </dependencies>


</project>
```

2.在 CityDao 城市数据操作层接口类添加注解 [@Mapper](https://github.com/Mapper)、[@Select](https://github.com/Select) 和 [@Results](https://github.com/Results)

```java
/**
* 城市 DAO 接口类
*
* Created by xchunzhao on 02/05/2017.
*/
@Mapper // 标志为 Mybatis 的 Mapper
public interface CityDao {

/**
* 根据城市名称，查询城市信息
*
* @param cityName 城市名
*/
@Select("SELECT * FROM city where city_name=${cityName}")
// 返回 Map 结果集
@Results({
@Result(property = "id", column = "id"),
@Result(property = "provinceId", column = "province_id"),
@Result(property = "cityName", column = "city_name"),
@Result(property = "description", column = "description"),
})
City findByName(@Param("cityName") String cityName);
}
```

[@Mapper](https://github.com/Mapper) 标志接口为 MyBatis Mapper 接口
[@Select](https://github.com/Select) 是 Select 操作语句
[@Results](https://github.com/Results) 标志结果集，以及与库表字段的映射关系

其他的注解可以看 org.apache.ibatis.annotations 包提供的，如图：

![img](https://www.bysocket.com/wp-content/uploads/2017/05/WX20170511-143439.png)

可以 git clone 下载工程 springboot-learning-example ，springboot-mybatis-annotation 工程代码注解很详细。 <https://github.com/JeffLi1993/springboot-learning-example>。

# 四、小结

注解不涉及到配置，更近贴近 0 配置。再次感谢晓春 <http://xchunzhao.tk/> 的 Pull Request~