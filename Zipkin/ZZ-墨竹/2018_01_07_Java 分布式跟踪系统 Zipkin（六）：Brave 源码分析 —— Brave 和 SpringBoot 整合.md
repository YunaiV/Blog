title: Java 分布式跟踪系统 Zipkin（六）：Brave 源码分析 —— Brave 和 SpringBoot 整合
date: 2018-01-07
tag: 
categories: Zipkin
permalink: Zipkin/mozhu/brace-with-spring-boot
author: v墨竹v
from_url: https://blog.csdn.net/apei830/article/details/78722253
wechat_url: 

-------

摘要: 原创出处 https://blog.csdn.net/apei830/article/details/78722253 「v墨竹v」欢迎转载，保留摘要，谢谢！

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

Zipkin是用当下最流行的SpringBoot开发的，SpringBoot将Spring项目的开发过程大大简化，一切主流的开发框架都可以通过添加jar包和配置，自动激活，现在越来越受广大Java开发人员的喜爱。
上一篇博文中，我们分析了Brave是如何在SpringMVC项目中使用的，这一篇博文我们继续分析Brave和SpringBoot项目的整合方法及原理。

相关代码在Chapter6/springboot中
pom.xml中添加依赖和插件

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
    <version>${springboot.version}</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <version>${springboot.version}</version>
</dependency>

<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <fork>true</fork>
    </configuration>
</plugin>
```

```java
package org.mozhu.zipkin.springboot;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@EnableAutoConfiguration
public class DefaultApplication {

    public static void main(String[] args) {
        SpringApplication.run(DefaultApplication.class, args);
    }

}
```

启动Zipkin，然后分别运行

```shell
mvn spring-boot:run -Drun.jvmArguments="-Dserver.port=9000 -Dzipkin.service=backend"
```

```shell
mvn spring-boot:run -Drun.jvmArguments="-Dserver.port=8081 -Dzipkin.service=frontend"
```

浏览器访问 <http://localhost:8081/> 会显示当前时间
在Zipkin的Web界面中，也能查询到这次跟踪信息

可见Brave和SpringBoot的整合更简单了，只添加了启动类DefaultApplication，其他类都没变化。至于SpringBoot的原理，这里就不展开了，网上优秀教程一大把。

在brave-instrumentation目录中，还有对其他框架的支持，有兴趣的可以看看其源代码实现。
grpc
httpasyncclient
httpclient
jaxrs2
kafka-clients
mysql
mysql6
p6spy
sparkjava

至此，我们Brave的源码分析即将告一段落，后续我们会逐步zipkin的高级用法及实现原理。

# 666. 彩蛋

如果你对 Zipkin 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)