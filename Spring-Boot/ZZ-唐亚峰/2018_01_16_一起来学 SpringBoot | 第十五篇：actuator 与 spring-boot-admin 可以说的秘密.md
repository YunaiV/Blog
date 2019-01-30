title: 一起来学 SpringBoot 2.x | 第十五篇：actuator 与 spring-boot-admin 可以说的秘密
date: 2018-01-16
tag: 
categories: Spring Boot
permalink: Spring-Boot/battcn/v2-actuator-monitor/
author: 唐亚峰
from_url: http://blog.battcn.com/2018/05/24/springboot/v2-actuator-monitor/
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247485928&idx=2&sn=1366e81da20840d0631a35fcbb83ae2c&chksm=fa497659cd3eff4fd7d50a9337c6b35d1466605f895dae91a2ec1a141525716ff0207fcfa698&token=396846451&lang=zh_CN#rd

-------

摘要: 原创出处 http://blog.battcn.com/2018/05/24/springboot/v2-actuator-monitor/ 「唐亚峰」欢迎转载，保留摘要，谢谢！

- [什么是SBA](http://www.iocoder.cn/Spring-Boot/battcn/v2-actuator-introduce//)
- [导入依赖](http://www.iocoder.cn/Spring-Boot/battcn/v2-actuator-introduce//)
- [属性配置](http://www.iocoder.cn/Spring-Boot/battcn/v2-actuator-introduce//)
- [描述信息](http://www.iocoder.cn/Spring-Boot/battcn/v2-actuator-introduce//)
  - [主函数](http://www.iocoder.cn/Spring-Boot/battcn/v2-actuator-introduce//)
  - [测试](http://www.iocoder.cn/Spring-Boot/battcn/v2-actuator-introduce//)
- [总结](http://www.iocoder.cn/Spring-Boot/battcn/v2-actuator-introduce//)
- [说点什么](http://www.iocoder.cn/Spring-Boot/battcn/v2-actuator-introduce//)

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

[一起来学SpringBoot | 第十四篇：强大的 actuator 服务监控与管理](http://www.iocoder.cn/Spring-Boot/battcn/v2-actuator-introduce/) 中介绍了`actuator` 的作用，细心的朋友可能会发现通过`http restful api`的方式查看信息过于繁琐也不够直观，效率低下，运维人员看到JSON数据更是一脸懵逼，当服务过多的时候查看起来就过于操蛋了，每个服务都需要调用不同的接口来查看监控信息，备受各种困扰因素的我默默翻了下`全球最大男性交友平台`找到了`spring-boot-admin`

# 什么是SBA

**SBA 全称 Spring Boot Admin 是一个管理和监控 Spring Boot 应用程序的开源项目。分为admin-server 与 admin-client 两个组件，admin-server通过采集 actuator 端点数据，显示在 spring-boot-admin-ui 上，已知的端点几乎都有进行采集，通过 spring-boot-admin 可以动态切换日志级别、导出日志、导出heapdump、监控各项指标 等等….**

`Spring Boot Admin` 在对单一应用服务监控的同时也提供了集群监控方案，支持通过`eureka`、`consul`、`zookeeper`等注册中心的方式实现多服务监控与管理…

# 导入依赖

在 `pom.xml` 中添加 `spring-boot-admin` 的相关依赖，这里只演示单机版本的，因此就自己监控自己了

```xml
<dependencies>
    <!-- 服务端：带UI界面 -->
    <dependency>
        <groupId>de.codecentric</groupId>
        <artifactId>spring-boot-admin-starter-server</artifactId>
        <version>2.0.0</version>
    </dependency>
    <!-- 客户端包 -->
    <dependency>
        <groupId>de.codecentric</groupId>
        <artifactId>spring-boot-admin-starter-client</artifactId>
        <version>2.0.0</version>
    </dependency>
    <!-- 安全认证 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <!-- 端点 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <!-- 在管理界面中与 JMX-beans 进行交互所需要被依赖的 JAR -->
    <dependency>
        <groupId>org.jolokia</groupId>
        <artifactId>jolokia-core</artifactId>
    </dependency>
</dependencies>
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

在 `application.properties` 文件中配置`actuator`的相关配置，其中`info`开头的属性，就是访问`info`端点中显示的相关内容，值得注意的是`Spring Boot2.x`中，默认只开放了`info、health`两个端点，剩余的需要自己通过配置`management.endpoints.web.exposure.include`属性来加载（有`include`自然就有`exclude`，不做详细概述了）。这个`management.endpoints.web.base-path`属性比较重要，因为`Spring Boot2.x`后每个端点默认的路径是`/actuator/endpointId`这样一来`Spring Boot Admin`是无法正常采集的

> application.properties

```properties
# 描述信息
info.blog-url=http://blog.battcn.com
info.author=Levin
# 如果 Maven 插件没配置此处请注释掉
info.version=@project.version@
info.name=@project.artifactId@
# 选择激活对应环境的配置，如果是dev则代表不用认证就能访问监控页，prod代表需要认证
spring.profiles.active=prod

# 日志文件
logging.file=./target/admin-server.log

# 加载所有的端点/默认只加载了 info / health
management.endpoints.web.exposure.include=*
# 比较重要,默认 /actuator spring-boot-admin 扫描不到
management.endpoints.web.base-path=/
management.endpoint.health.show-details=always

spring.boot.admin.client.url=http://localhost:8080
# 不配置老喜欢用主机名，看着不舒服....
spring.boot.admin.client.instance.prefer-ip=true
```

> application-dev.properties - 空

> application-prod.properties

为了安全起见，应采用认证的方式

```properties
# 登陆所需的账号密码
spring.security.user.name=battcn
spring.security.user.password=battcn
# 便于客户端可以在受保护的服务器上注册api
spring.boot.admin.client.username=battcn
spring.boot.admin.client.password=battcn
# 便服务器可以访问受保护的客户端端点
spring.boot.admin.client.instance.metadata.user.name=battcn
spring.boot.admin.client.instance.metadata.user.password=battcn
```

## 主函数

添加上 `@EnableAdminServer` 注解即代表是`Server`端，集成UI的

```java
package com.battcn;

import de.codecentric.boot.admin.server.config.AdminServerProperties;
import de.codecentric.boot.admin.server.config.EnableAdminServer;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.web.authentication.SavedRequestAwareAuthenticationSuccessHandler;


/**
 * @author Levin
 */
@SpringBootApplication
@EnableAdminServer
public class Chapter14Application {

    public static void main(String[] args) {
        SpringApplication.run(Chapter14Application.class, args);
    }

    /**
     * dev 环境加载
     */
    @Profile("dev")
    @Configuration
    public static class SecurityPermitAllConfig extends WebSecurityConfigurerAdapter {
        @Override
        protected void configure(HttpSecurity http) throws Exception {
            http.authorizeRequests().anyRequest().permitAll()
                    .and().csrf().disable();
        }
    }

    /**
     * prod 环境加载
     */
    @Profile("prod")
    @Configuration
    public static class SecuritySecureConfig extends WebSecurityConfigurerAdapter {
        private final String adminContextPath;

        public SecuritySecureConfig(AdminServerProperties adminServerProperties) {
            this.adminContextPath = adminServerProperties.getContextPath();
        }

        @Override
        protected void configure(HttpSecurity http) throws Exception {
            SavedRequestAwareAuthenticationSuccessHandler successHandler = new SavedRequestAwareAuthenticationSuccessHandler();
            successHandler.setTargetUrlParameter("redirectTo");

            http.authorizeRequests()
                    .antMatchers(adminContextPath + "/assets/**").permitAll()
                    .antMatchers(adminContextPath + "/login").permitAll()
                    .anyRequest().authenticated()
                    .and()
                    .formLogin().loginPage(adminContextPath + "/login").successHandler(successHandler).and()
                    .logout().logoutUrl(adminContextPath + "/logout").and()
                    .httpBasic().and()
                    .csrf().disable();
        }
    }
}
```

## 测试

完成准备事项后，启动`Chapter14Application` 访问 <http://localhost:8080/login> 看到登陆页面则代表一切正常，接着输入账号密码点击登陆即可…

[![首页](https://image.battcn.com/article/images/20180525/springboot/v2-actuator-monitor/3.png)](https://image.battcn.com/article/images/20180525/springboot/v2-actuator-monitor/3.png)

由于篇幅原因大图就不放太多了，有兴趣的朋友可以直接`fork`代码运行即可

# 总结

**参考文档：**<http://codecentric.github.io/spring-boot-admin/2.0.0/>

目前很多大佬都写过关于 **SpringBoot** 的教程了，如有雷同，请多多包涵，本教程基于最新的 `spring-boot-starter-parent：2.0.2.RELEASE`编写，包括新版本的特性都会一起介绍…

# 说点什么

全文代码：<https://github.com/battcn/spring-boot2-learning/tree/master/chapter14>

# 666. 彩蛋

如果你对 SpringBoot 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)