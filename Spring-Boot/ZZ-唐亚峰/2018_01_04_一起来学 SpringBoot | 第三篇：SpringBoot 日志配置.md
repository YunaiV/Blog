title: 一起来学 SpringBoot 2.x | 第三篇：SpringBoot 日志配置
date: 2018-01-04
tag: 
categories: Spring Boot
permalink: Spring-Boot/battcn/v2-config-logs/
author: 唐亚峰
from_url: http://blog.battcn.com/2018/04/23/springboot/v2-config-logs/
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484678&idx=1&sn=7a38847952857e3b0f61455f2899da6b&chksm=fa497ab7cd3ef3a104e0463bd9781e89e30ce18df237a5a318c316839b60fcb09e7211c17226#rd

-------

摘要: 原创出处 http://blog.battcn.com/2018/04/23/springboot/v2-config-logs/ 「唐亚峰」欢迎转载，保留摘要，谢谢！

- [日志格式](http://www.iocoder.cn/Spring-Boot/battcn/v2-config-logs//)
- [日志输出](http://www.iocoder.cn/Spring-Boot/battcn/v2-config-logs//)
- [颜色编码](http://www.iocoder.cn/Spring-Boot/battcn/v2-config-logs//)
  - [编码对照表](http://www.iocoder.cn/Spring-Boot/battcn/v2-config-logs//)
- [文件保存](http://www.iocoder.cn/Spring-Boot/battcn/v2-config-logs//)
- [自定义日志配置](http://www.iocoder.cn/Spring-Boot/battcn/v2-config-logs//)
- [Logback扩展配置](http://www.iocoder.cn/Spring-Boot/battcn/v2-config-logs//)
  - [springProfile](http://www.iocoder.cn/Spring-Boot/battcn/v2-config-logs//)
  - [springProperty](http://www.iocoder.cn/Spring-Boot/battcn/v2-config-logs//)
- [总结](http://www.iocoder.cn/Spring-Boot/battcn/v2-config-logs//)
- [说点什么](http://www.iocoder.cn/Spring-Boot/battcn/v2-config-logs//)

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

`Spring Boot` 内部采用的是 `Commons Logging`进行日志记录，但在底层为 **Java Util Logging、Log4J2、Logback** 等日志框架提供了默认配置 。

> Java 虽然有很多可用的日志框架，但请不要担心，一般来说，使用 `SpringBoot` 默认的 `Logback` 就可以了。

# 日志格式

> `SpringBoot` 的默认输出的日志格式如下：

```shell
2014-03-05 10:57:51.112  INFO 45469 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet Engine: Apache Tomcat/7.0.52
2014-03-05 10:57:51.253  INFO 45469 --- [ost-startStop-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2014-03-05 10:57:51.253  INFO 45469 --- [ost-startStop-1] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 1358 ms
2014-03-05 10:57:51.698  INFO 45469 --- [ost-startStop-1] o.s.b.c.e.ServletRegistrationBean        : Mapping servlet: 'dispatcherServlet' to [/]
2014-03-05 10:57:51.702  INFO 45469 --- [ost-startStop-1] o.s.b.c.embedded.FilterRegistrationBean  : Mapping filter: 'hiddenHttpMethodFilter' to: [/*]
```

> 输出如下元素：

**Logback 是没有 FATAL级别的日志，它将被映射到 ERROR**

- 时间日期：精确到毫秒,可以用于排序
- 日志级别：ERROR、WARN、INFO、DEBUG、TRACE
- 进程ID
- 分隔符：采用 `---` 来标识日志开始部分
- 线程名：方括号括起来（可能会截断控制台输出）
- Logger名：通常使用源代码的类名
- 日志内容：我们输出的消息

# 日志输出

`SpringBoot` 默认为我们输出的日志级别为 `INFO`、`WARN`、`ERROR`，如需要输出更多日志的时候，可以通过以下方式开启

- 命令模式配置： `java -jar app.jar --debug=true` ， 这种命令会被 `SpringBoot` 解析，且优先级最高
- 资源文件配置： `application.properties` 配置 `debug=true` 即可。**该配置只对 嵌入式容器、Spring、Hibernate生效，我们自己的项目想要输出 DEBUG 需要额外配置（配置规则：logging.level.<logger-name>=<level>）**

> 日志输出级别配置

```properties
logging.level.root = WARN
logging.level.org.springframework.web = DEBUG
logging.level.org.hibernate = ERROR

#比如 mybatis sql日志
logging.level.org.mybatis = INFO
logging.level.mapper所在的包 = DEBUG
```

> 日志输出格式配置

- **logging.pattern.console：** 定义输出到控制台的格式（不支持JDK Logger）
- **logging.pattern.file：** 定义输出到文件的格式（不支持JDK Logger）

# 颜色编码

如果终端支持 `ANSI`，默认情况下会给日志上个色，提高可读性，可以在配置文件中设置 `spring.output.ansi.enabled` 来改变默认值

- **ALWAYS：** 启用 `ANSI` 颜色的输出。
- **DETECT：** 尝试检测 `ANSI` 着色功能是否可用。
- **NEVER：** 禁用 `ANSI` 颜色的输出。

## 编码对照表

| Level                    | Color  |
| ------------------------ | ------ |
| `WARN`                   | Yellow |
| `FATAL`、`ERROR`         | Red    |
| `INFO`、`DEBUG`、`TRACE` | Green  |

如果想修改日志默认色值，可以通过使用 `%clr` 关键字转换。比如想使文本变为黄色 `%clr(%d{yyyy-MM-dd HH:mm:ss.SSS}){yellow}`。目前支持的颜色有（`blue`、`cyan`、`faint`、`green`、`magenta`、`red`、`yellow`）

# 文件保存

默认情况下，`SpringBoot` 仅将日志输出到控制台，不会写入到日志文件中去。如果除了控制台输出之外还想写日志文件，则需要在`application.properties` 设置`logging.file` 或 `logging.path` 属性。

- **logging.file：** 将日志写入到指定的 **文件** 中，默认为相对路径，可以设置成绝对路径
- **logging.path：** 将名为 `spring.log` 写入到指定的 **文件夹** 中，如（`/var/log`）

日志文件在达到 `10MB` 时进行切割，产生一个新的日志文件（如：`spring.1.log、spring.2.log`），新的日志依旧输出到 `spring.log` 中去，默认情况下会记录 `ERROR`、`WARN`、`INFO` 级别消息。

- **logging.file.max-size：** 限制日志文件大小
- **logging.file.max-history：** 限制日志保留天数

# 自定义日志配置

由于日志在 `ApplicationContext` 之前就初始化好了，所以 `SpringBoot` 为我们提供了 `logging.config` 属性，方便我们配置自定义日志文件。默认情况它会根据日志的依赖自动加载。

| Logging System            | Customization                                                |
| ------------------------- | ------------------------------------------------------------ |
| `JDK (Java Util Logging)` | logging.properties                                           |
| `Log4j2`、`ERROR`         | log4j2-spring.xml 或 log4j2.xml                              |
| `Logback`                 | logback-spring.xml、logback-spring.groovy、logback.xml、logback.groovy |

# Logback扩展配置

该扩展配置仅适用 `logback-spring.xml` 或者设置 `logging.config` 属性的文件，因为 `logback.xml` 加载过早，因此无法获取 `SpringBoot`的一些扩展属性

使用扩展属性 `springProfile` 与 `springProperty` 让你的 `logback-spring.xml` 配置显得更有逼格，当别人还在苦苦挣扎弄logback-{profile}.xml的时候 你一个文件就搞定了…

## springProfile

`<springProfile>` 标签使我们让配置文件更加灵活，**它可以选择性的包含或排除部分配置**。

```xml
<springProfile name="dev">
    <!-- 开发环境时激活 -->
</springProfile>

<springProfile name="dev,test">
    <!-- 开发，测试的时候激活-->
</springProfile>

<springProfile name="!prod">
    <!-- 当 "生产" 环境时，该配置不激活-->
</springProfile>
```

### 案例

```xml
<!-- 开发环境日志级别为DEBUG/并且开发环境不写日志文件 -->
<springProfile name="dev">
    <root level="DEBUG">
        <appender-ref ref="STDOUT"/>
    </root>
</springProfile>

<!-- 测试环境日志级别为INFO/并且记录日志文件 -->
<springProfile name="test">
    <root level="INFO">
        <appender-ref ref="FILE"/>
        <appender-ref ref="STDOUT"/>
    </root>
</springProfile>
```

## springProperty

`<springProperty>` 标签可以让我们在 **Logback** 中使用 Spring Environment 中的属性。如果想在`logback-spring.xml`中回读 `application.properties` 配置的值时，这是一个非常好的解决方案

```xml
<!-- 读取 spring.application.name 属性来生成日志文件名
	scope：作用域
	name：在 logback-spring.xml 使用的键
	source：application.properties 文件中的键
	defaultValue：默认值
 -->
<springProperty scope="context" name="logName" source="spring.application.name" defaultValue="myapp.log"/>

<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>logs/${logName}.log</file>
</appender>
```

# 总结

更多细节请参考官方文档：<https://docs.spring.io/spring-boot/docs/2.0.1.RELEASE/reference/htmlsingle/#boot-features-custom-log-configuration>

目前很多大佬都写过关于 **SpringBoot** 的教程了，如有雷同，请多多包涵，本教程基于最新的 `spring-boot-starter-parent：2.0.1.RELEASE`编写，包括新版本的特性都会一起介绍…

# 666. 彩蛋

如果你对 SpringBoot 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)