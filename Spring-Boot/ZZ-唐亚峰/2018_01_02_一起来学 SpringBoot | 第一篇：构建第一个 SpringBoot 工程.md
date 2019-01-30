title: 一起来学 SpringBoot 2.x | 第一篇：构建第一个 SpringBoot 工程
date: 2018-01-02
tag: 
categories: Spring Boot
permalink: Spring-Boot/battcn/v2-introducing/
author: 唐亚峰
from_url: http://blog.battcn.com/2018/04/20/springboot/v2-introducing/
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484608&idx=1&sn=4a6269490b4e8c001709f029d0640e98&chksm=fa497b71cd3ef26704ed0a937903d734a72b7d430e89393c9dd93fd3a7fc05aaf6aebd0b58ff#rd

-------

摘要: 原创出处 http://blog.battcn.com/2018/04/20/springboot/v2-introducing/ 「唐亚峰」欢迎转载，保留摘要，谢谢！

- [设计的目标](http://www.iocoder.cn/Spring-Boot/battcn/v2-introducing//)
- [前提](http://www.iocoder.cn/Spring-Boot/battcn/v2-introducing//)
- [创建项目](http://www.iocoder.cn/Spring-Boot/battcn/v2-introducing//)
  - [目录结果](http://www.iocoder.cn/Spring-Boot/battcn/v2-introducing//)
  - [pom.xml 依赖](http://www.iocoder.cn/Spring-Boot/battcn/v2-introducing//)
  - [主函数入口](http://www.iocoder.cn/Spring-Boot/battcn/v2-introducing//)
  - [初窥配置文件](http://www.iocoder.cn/Spring-Boot/battcn/v2-introducing//)
  - [测试](http://www.iocoder.cn/Spring-Boot/battcn/v2-introducing//)
- [拓展知识](http://www.iocoder.cn/Spring-Boot/battcn/v2-introducing//)
  - [自定义Banner](http://www.iocoder.cn/Spring-Boot/battcn/v2-introducing//)
- [总结](http://www.iocoder.cn/Spring-Boot/battcn/v2-introducing//)
- [说点什么](http://www.iocoder.cn/Spring-Boot/battcn/v2-introducing//)

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

未接触 `SpringBoot` 之前，搭建一个普通的 `WEB` 工程往往需要花费30分钟左右，如果遇到点奇葩的问题耽搁的时间会更长一点，但自从用了`SpringBoot` 后，真正体会到什么叫分分钟搭建一个`WEB`，让我拥有更多的时间跟我的小伙伴们唠嗑了。使用 `SpringBoot` 后发现一切是如此的简单（**还记得读书那会被JAR包，xml支配的恐惧吗，如今都可以说 good bye**）

[![吐槽](https://image.battcn.com/article/images/20180420/springboot/v2-Introducing/5.png)](https://image.battcn.com/article/images/20180420/springboot/v2-Introducing/5.png)

# 设计的目标

- 为所有使用 `Spring` 的开发者提供一个更简单，快速的入门体验
- 提供一些常见的功能、如监控、WEB容器，健康，安全等功能
- 干掉XML，遵循规范，开箱即用

# 前提

`SpringBoot` 为我们提供了一系列的依赖包，所以需要构建工具的支持：**Maven** 或 **Gradle**。由于本人更习惯使用**Maven**所以后续案例都是基于`Maven` 与 **IntelliJ IDEA**，同时这里是基于最新的**SpringBoot2编写的哦...**

# 创建项目

初次接触，我们先来看看如何创建一个`Spring Boot`项目，这里以`IntelliJ IDEA`为例，其他的IDE工具小伙伴们自行搜索创建方式。创建完项目后，各位小伙伴请认真、细心的对比下与传统的WEB工程有何区别（如：目录结构）。

> 点击**File -> Project**

如果用过 `Eclipse/IDEA` 等工具的，对创建项目肯定不会陌生，但为了照顾第一次使用的我贴上了图文

[![第一步](https://image.battcn.com/article/images/20180420/springboot/v2-Introducing/1.png)](https://image.battcn.com/article/images/20180420/springboot/v2-Introducing/1.png)

> 选择**Spring Initializr**

到这一步选择的时候，如图中选项的是`Spring Initializr(官方的构建插件，需要联网)`，第二个是自己选择`Maven`构建，为了更好的适合初学者，我们将在本章用插件构建

[![第二步](https://image.battcn.com/article/images/20180420/springboot/v2-Introducing/2.png)](https://image.battcn.com/article/images/20180420/springboot/v2-Introducing/2.png)

> 填写项目基本信息

- **Group：** 组织ID，一般分为多个段，这里我只说两段，第一段为域，第二段为公司名称。域又分为 `org、com、cn`等等，其中 **org为非营利组织，com为商业组织**。如阿里、淘宝（com.alibaba/com.taobao）
- **Artifact:** 唯一标识符，一般是项目名称

[![第三步](https://image.battcn.com/article/images/20180420/springboot/v2-Introducing/3.png)](https://image.battcn.com/article/images/20180420/springboot/v2-Introducing/3.png)

> 选择包

**Spring Initializr** 为我们提供了很多的选项，不同的选项有不同的作用，在初期我们只需要依赖**Web -> Web** 就可以了，选择好依赖包之后点击**Next -> Finish**

[![第四步](https://image.battcn.com/article/images/20180420/springboot/v2-Introducing/4.png)](https://image.battcn.com/article/images/20180420/springboot/v2-Introducing/4.png)

## 目录结果

```YAML
- src
    -main
        -java
            -package
                #主函数，启动类，运行它如果运行了 Tomcat、Jetty、Undertow 等容器
                -SpringbootApplication	
        -resouces
            #存放静态资源 js/css/images 等
            - statics
            #存放 html 模板文件
            - templates
            #主要的配置文件，SpringBoot启动时候会自动加载application.yml/application.properties		
            - application.yml
    #测试文件存放目录		
    -test
 # pom.xml 文件是Maven构建的基础，里面包含了我们所依赖JAR和Plugin的信息
- pom
```

## pom.xml 依赖

因为使用了 **Spring Initializr** 插件，所以如下的配置都不需要我们自己去写啦，需要注意的是版本要选择 **RELEASE** ，稳定版本BUG少

```XML
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.battcn</groupId>
	<artifactId>chapter1</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>chapter1</name>
	<description>我的用第一个SpringBoot工程</description>

	<!--版本采用的是最新的 2.0.1.RELEASE TODO 开发中请记得版本一定要选择 RELEASE 哦 -->
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.0.1.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
	</properties>

	<dependencies>
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
	</dependencies>

	<build>
		<plugins>
			<!-- 编译插件 -->
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>


</project>
```

其它的依赖可以参考：[官方文档](https://docs.spring.io/spring-boot/docs/2.0.1.RELEASE/reference/htmlsingle/#using-boot-starter)

## 主函数入口

**注意事项：** 一个项目中切记不要出现多个 `main` 函数，否在在打包的时候 `spring-boot-maven-plugin` 将找不到主函数（**主动指定打包主函数入口除外…**）

```Java
/**
 * 我的第一个SpringBoot程序
 * 其中 @RestController 等同于 （@Controller 与 @ResponseBody）
 *
 * @author Levin
 */
@RestController
@SpringBootApplication
public class Chapter1Application {

    public static void main(String[] args) {
        SpringApplication.run(Chapter1Application.class, args);
    }

    @GetMapping("/demo1")
    public String demo1() {
        return "Hello battcn";
    }

    @Bean
    public CommandLineRunner commandLineRunner(ApplicationContext ctx) {
        // 目的是
        return args -> {
            System.out.println("来看看 SpringBoot 默认为我们提供的 Bean：");
            String[] beanNames = ctx.getBeanDefinitionNames();
            Arrays.sort(beanNames);
            Arrays.stream(beanNames).forEach(System.out::println);
        };
    }
}
```

## 初窥配置文件

从启动日志中可以发现，`SpringBoot` 默认的端口是 8080 ，那么如果端口被占用了怎么办呢？不要慌，问题不大，配置文件分分钟解决你的困扰…

```SHELL
2018-04-20 16:14:46.725  INFO 11184 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
```

> 修改默认配置

```Properties
# 默认的 8080 我们将它改成 9090 
server.port=9090
# 未定义上下文路径之前 地址是 http://localhost:8080 定义了后 http://localhost:9090 你能在tomcat做的事情，配置文件都可以
server.servlet.context-path=/chapter1
```

在启动一次看看日志

```SHELL
2018-04-20 16:47:05.716  INFO 12108 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 9090 (http) with context path '/chapter1'
```

## 测试

本次测试采用 `junit` 进行，当然也可以启动项目后直接访问 `http://localhost:9090/chapter/demo1` 进行测试

```Java
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.boot.web.server.LocalServerPort;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.junit4.SpringRunner;

import java.net.URL;

import static org.junit.Assert.assertEquals;

@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class Chapter1ApplicationTests {


    @LocalServerPort
    private int port;

    private URL base;

    @Autowired
    private TestRestTemplate template;

    @Before
    public void setUp() throws Exception {
        // TODO 因为我们修改了 content-path 所以请求后面要带上
        this.base = new URL("http://localhost:" + port + "/chapter1/demo1");
    }

    @Test
    public void demo1() throws Exception {
        ResponseEntity<String> response = template.getForEntity(base.toString(), String.class);
        assertEquals(response.getBody(), "Hello battcn");
    }
}
```

# 拓展知识

## 自定义Banner

`SpringBoot` 启动的时候我们可以看到如下内容，这一块其实是可以自定义的哦，而且在 2.X 版本中，它支持的格式从文本扩展到**banner.txt、banner.jpg、banner.gif、banner.jpeg** 等等，只需要在 **resouces** 目录下添加指定命名的文件即可

```SHELL
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.0.1.RELEASE)
```

# 总结

目前很多大佬都写过关于 **SpringBoot** 的教程了，如有雷同，请多多包涵，本教程基于最新的 `spring-boot-starter-parent：2.0.1.RELEASE`编写，包括新版本的特性都会一起介绍…

# 说点什么

全文代码：<https://github.com/battcn/spring-boot2-learning/tree/master/chapter1>

# 666. 彩蛋

如果你对 SpringBoot 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)