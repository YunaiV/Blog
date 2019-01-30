title: 一起来学 SpringBoot 2.x | 第十四篇：强大的 Actuator 服务监控与管理
date: 2018-01-15
tag: 
categories: Spring Boot
permalink: Spring-Boot/battcn/v2-actuator-introduce/
author: 唐亚峰
from_url: http://blog.battcn.com/2018/05/24/springboot/v2-actuator-introduce/
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247485869&idx=2&sn=d3c333a4e49b0eb64b12bb3793be2d50&chksm=fa49761ccd3eff0ade64542fd57c3adcd78f267434c045f090fe3a1b011adc538dbbf3735e50&token=59602784&lang=zh_CN#rd

-------

摘要: 原创出处 http://blog.battcn.com/2018/05/24/springboot/v2-actuator-introduce/ 「唐亚峰」欢迎转载，保留摘要，谢谢！

- [Endpoints](http://www.iocoder.cn/Spring-Boot/battcn/v2-actuator-monitor//)
  - [内置Endpoints](http://www.iocoder.cn/Spring-Boot/battcn/v2-actuator-monitor//)
- [导入依赖](http://www.iocoder.cn/Spring-Boot/battcn/v2-actuator-monitor//)
- [属性配置](http://www.iocoder.cn/Spring-Boot/battcn/v2-actuator-monitor//)
- [描述信息](http://www.iocoder.cn/Spring-Boot/battcn/v2-actuator-monitor//)
- [自定义 - 重点](http://www.iocoder.cn/Spring-Boot/battcn/v2-actuator-monitor//)
  - [默认装配 HealthIndicators](http://www.iocoder.cn/Spring-Boot/battcn/v2-actuator-monitor//)
  - [健康端点（第一种方式）](http://www.iocoder.cn/Spring-Boot/battcn/v2-actuator-monitor//)
  - [健康端点（第二种方式）](http://www.iocoder.cn/Spring-Boot/battcn/v2-actuator-monitor//)
  - [定义自己的端点](http://www.iocoder.cn/Spring-Boot/battcn/v2-actuator-monitor//)
  - [主函数](http://www.iocoder.cn/Spring-Boot/battcn/v2-actuator-monitor//)
  - [测试](http://www.iocoder.cn/Spring-Boot/battcn/v2-actuator-monitor//)
- [总结](http://www.iocoder.cn/Spring-Boot/battcn/v2-actuator-monitor//)
- [说点什么](http://www.iocoder.cn/Spring-Boot/battcn/v2-actuator-monitor//)

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

`actuator`是`spring boot`项目中非常强大一个功能，有助于对应用程序进行监视和管理，通过 `restful api` 请求来监管、审计、收集应用的运行情况，针对微服务而言它是必不可少的一个环节…

# Endpoints

`actuator` 的核心部分，它用来监视应用程序及交互，`spring-boot-actuator`中已经内置了非常多的 **Endpoints（health、info、beans、httptrace、shutdown等等）**，同时也允许我们自己扩展自己的端点

**Spring Boot 2.0** 中的端点和之前的版本有较大不同,使用时需注意。另外端点的监控机制也有很大不同，启用了不代表可以直接访问，还需要将其暴露出来，传统的`management.security`管理已被标记为不推荐。

[![皮一下很开心](https://image.battcn.com/article/images/20180524/springboot/v2-actuator-introduce/1.jpg)](https://image.battcn.com/article/images/20180524/springboot/v2-actuator-introduce/1.jpg)

## 内置Endpoints

| id                 | desc                                                         | Sensitive |
| ------------------ | ------------------------------------------------------------ | --------- |
| **auditevents**    | 显示当前应用程序的审计事件信息                               | Yes       |
| **beans**          | 显示应用Spring Beans的完整列表                               | Yes       |
| **caches**         | 显示可用缓存信息                                             | Yes       |
| **conditions**     | 显示自动装配类的状态及及应用信息                             | Yes       |
| **configprops**    | 显示所有 @ConfigurationProperties 列表                       | Yes       |
| **env**            | 显示 ConfigurableEnvironment 中的属性                        | Yes       |
| **flyway**         | 显示 Flyway 数据库迁移信息                                   | Yes       |
| **health**         | 显示应用的健康信息（未认证只显示`status`，认证显示全部信息详情） | No        |
| **info**           | 显示任意的应用信息（在资源文件写info.xxx即可）               | No        |
| **liquibase**      | 展示Liquibase 数据库迁移                                     | Yes       |
| **metrics**        | 展示当前应用的 metrics 信息                                  | Yes       |
| **mappings**       | 显示所有 @RequestMapping 路径集列表                          | Yes       |
| **scheduledtasks** | 显示应用程序中的计划任务                                     | Yes       |
| **sessions**       | 允许从Spring会话支持的会话存储中检索和删除用户会话。         | Yes       |
| **shutdown**       | 允许应用以优雅的方式关闭（默认情况下不启用）                 | Yes       |
| **threaddump**     | 执行一个线程dump                                             | Yes       |
| **httptrace**      | 显示HTTP跟踪信息（默认显示最后100个HTTP请求 - 响应交换）     | Yes       |

# 导入依赖

在 `pom.xml` 中添加 `spring-boot-starter-actuator` 的依赖

```xml
 <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

> 注意事项

如果要访问`info`接口想获取`maven`中的属性内容请记得添加如下内容

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <executions>
                <execution>
                    <goals>
                        <goal>build-info</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

# 属性配置

在 `application.properties` 文件中配置`actuator`的相关配置，其中`info`开头的属性，就是访问`info`端点中显示的相关内容，值得注意的是`Spring Boot2.x`中，默认只开放了`info、health`两个端点，剩余的需要自己通过配置`management.endpoints.web.exposure.include`属性来加载（有`include`自然就有`exclude`，不做详细概述了）。如果想单独操作某个端点可以使用`management.endpoint.端点.enabled`属性进行启用或禁用

```properties
# 描述信息
info.blog-url=http://blog.battcn.com
info.author=Levin
info.version=@project.version@

# 加载所有的端点/默认只加载了 info / health
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always

# 可以关闭制定的端点
management.endpoint.shutdown.enabled=false

# 路径映射，将 health 路径映射成 rest_health 那么在访问 health 路径将为404，因为原路径已经变成 rest_health 了，一般情况下不建议使用
# management.endpoints.web.path-mapping.health=rest_health
```

> 简单测试

启动项目，访问 <http://localhost:8080/actuator/info> 看到如下内容代表配置成功

```json
{
  "blog-url": "http://blog.battcn.com",
  "author": "Levin",
  "version": "0.0.1-SNAPSHOT"
}
```

# 自定义 - 重点

上面讲了很多都是配置相关，以及自带的一些端点，在实际应用中有时候默认并不能满足我们的要求，比如`Spring Boot`默认的健康端点就很有可能不能满足

## 默认装配 HealthIndicators

下列是依赖`spring-boot-xxx-starter`后相关`HealthIndicator`的实现（通过`management.health.defaults.enabled` 属性可以禁用它们），但想要获取一些额外的信息时，自定义的作用就体现出来了…

| 名称                             | 描述                                |
| -------------------------------- | ----------------------------------- |
| **CassandraHealthIndicator**     | 检查 `Cassandra` 数据库是否启动。   |
| **DiskSpaceHealthIndicator**     | 检查磁盘空间不足。                  |
| **DataSourceHealthIndicator**    | 检查是否可以获得连接 `DataSource`。 |
| **ElasticsearchHealthIndicator** | 检查 `Elasticsearch` 集群是否启动。 |
| **InfluxDbHealthIndicator**      | 检查 `InfluxDB` 服务器是否启动。    |
| **JmsHealthIndicator**           | 检查 `JMS` 代理是否启动。           |
| **MailHealthIndicator**          | 检查邮件服务器是否启动。            |
| **MongoHealthIndicator**         | 检查 `Mongo` 数据库是否启动。       |
| **Neo4jHealthIndicator**         | 检查 `Neo4j` 服务器是否启动。       |
| **RabbitHealthIndicator**        | 检查 `Rabbit` 服务器是否启动。      |
| **RedisHealthIndicator**         | 检查 `Redis` 服务器是否启动。       |
| **SolrHealthIndicator**          | 检查 `Solr` 服务器是否已启动。      |

## 健康端点（第一种方式）

实现`HealthIndicator`接口，根据自己的需要判断返回的状态是`UP`还是`DOWN`，功能简单。

```java
package com.battcn.health;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

/**
 * <p>自定义健康端点</p>
 *
 * @author Levin
 * @since 2018/5/24 0024
 */
@Component("my1")
public class MyHealthIndicator implements HealthIndicator {

    private static final String VERSION = "v1.0.0";

    @Override
    public Health health() {
        int code = check();
        if (code != 0) {
            Health.down().withDetail("code", code).withDetail("version", VERSION).build();
        }
        return Health.up().withDetail("code", code)
                .withDetail("version", VERSION).up().build();
    }
    private int check() {
        return 0;
    }
}
```

> 简单测试

启动项目，访问 <http://localhost:8080/actuator/health> 看到如下内容代表配置成功

```json
{
  "status": "UP",
  "details": {
    "my1": {
      "status": "UP",
      "details": {
        "code": 0,
        "version": "v1.0.0"
      }
    },
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 100944310272,
        "free": 55071866880,
        "threshold": 10485760
      }
    }
  }
}
```

## 健康端点（第二种方式）

继承`AbstractHealthIndicator`抽象类，重写`doHealthCheck`方法，功能比第一种要强大一点点，默认的**DataSourceHealthIndicator 、 RedisHealthIndicator** 都是这种写法，内容回调中还做了异常的处理。

```java
package com.battcn.health;

import org.springframework.boot.actuate.health.AbstractHealthIndicator;
import org.springframework.boot.actuate.health.Health;
import org.springframework.stereotype.Component;

/**
 * <p>自定义健康端点</p>
 * <p>功能更加强大一点，DataSourceHealthIndicator / RedisHealthIndicator 都是这种写法</p>
 *
 * @author Levin
 * @since 2018/5/24 0024
 */
@Component("my2")
public class MyAbstractHealthIndicator extends AbstractHealthIndicator {

    private static final String VERSION = "v1.0.0";

    @Override
    protected void doHealthCheck(Health.Builder builder) throws Exception {
        int code = check();
        if (code != 0) {
            builder.down().withDetail("code", code).withDetail("version", VERSION).build();
        }
        builder.withDetail("code", code)
                .withDetail("version", VERSION).up().build();
    }

    private int check() {
        return 0;
    }
}
```

> 简单测试

启动项目，访问 <http://localhost:8080/actuator/health> 看到如下内容代表配置成功

```json
{
  "status": "UP",
  "details": {
    "my2": {
      "status": "UP",
      "details": {
        "code": 0,
        "version": "v1.0.0"
      }
    },
    "my1": {...},
    "diskSpace": {...}
  }
}
```

## 定义自己的端点

上面介绍的 **info、health** 都是`spring-boot-actuator`内置的，真正要实现自己的端点还得通过`@Endpoint、 @ReadOperation、@WriteOperation、@DeleteOperation`。

> 注解介绍

不同请求的操作，调用时缺少必需参数，或者使用无法转换为所需类型的参数，则不会调用操作方法，响应状态将为400（错误请求）

- **@Endpoint** 构建 rest api 的唯一路径
- **@ReadOperation** GET请求，响应状态为 200 如果没有返回值响应 404（资源未找到）
- **@WriteOperation** POST请求，响应状态为 200 如果没有返回值响应 204（无响应内容）
- **@DeleteOperation** DELETE请求，响应状态为 200 如果没有返回值响应 204（无响应内容）

```java
package com.battcn.endpoint;

import org.springframework.boot.actuate.endpoint.annotation.Endpoint;
import org.springframework.boot.actuate.endpoint.annotation.ReadOperation;

import java.util.HashMap;
import java.util.Map;

/**
 * <p>@Endpoint 是构建 rest 的唯一路径 </p>
 * 不同请求的操作，调用时缺少必需参数，或者使用无法转换为所需类型的参数，则不会调用操作方法，响应状态将为400（错误请求）
 * <P>@ReadOperation = GET 响应状态为 200 如果没有返回值响应 404（资源未找到） </P>
 * <P>@WriteOperation = POST 响应状态为 200 如果没有返回值响应 204（无响应内容） </P>
 * <P>@DeleteOperation = DELETE 响应状态为 200 如果没有返回值响应 204（无响应内容） </P>
 *
 * @author Levin
 * @since 2018/5/24 0024
 */
@Endpoint(id = "battcn")
public class MyEndPoint {

    @ReadOperation
    public Map<String, String> hello() {
        Map<String, String> result = new HashMap<>();
        result.put("author", "Levin");
        result.put("age", "24");
        result.put("email", "1837307557@qq.com");
        return result;
    }
}
```

以为这就大功告成了吗，现实告诉我的是`spring-boot`默认是不认识这玩意的，得申明成一个`Bean`（请看 **主函数**）

[![皮一下很开心](https://image.battcn.com/article/images/20180524/springboot/v2-actuator-introduce/2.png)](https://image.battcn.com/article/images/20180524/springboot/v2-actuator-introduce/2.png)

## 主函数

```java
package com.battcn;

import com.battcn.endpoint.MyEndPoint;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.actuate.autoconfigure.endpoint.condition.ConditionalOnEnabledEndpoint;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;


/**
 * @author Levin
 */
@SpringBootApplication
public class Chapter13Application {

    public static void main(String[] args) {
        SpringApplication.run(Chapter13Application.class, args);
    }

    @Configuration
    static class MyEndpointConfiguration {
        @Bean
        @ConditionalOnMissingBean
        @ConditionalOnEnabledEndpoint
        public MyEndPoint myEndPoint() {
            return new MyEndPoint();
        }
    }
}
```

## 测试

完成准备事项后，启动`Chapter13Application` 访问 <http://localhost:8080/actuator/battcn> 看到如下内容代表配置成功…

```json
{
  "author": "Levin",
  "age": "24",
  "email": "1837307557@qq.com"
}
```

# 总结

**参考文档：**<https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#production-ready>

目前很多大佬都写过关于 **SpringBoot** 的教程了，如有雷同，请多多包涵，本教程基于最新的 `spring-boot-starter-parent：2.0.2.RELEASE`编写，包括新版本的特性都会一起介绍…

# 说点什么

全文代码：<https://github.com/battcn/spring-boot2-learning/tree/master/chapter13>

# 666. 彩蛋

如果你对 SpringBoot 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)