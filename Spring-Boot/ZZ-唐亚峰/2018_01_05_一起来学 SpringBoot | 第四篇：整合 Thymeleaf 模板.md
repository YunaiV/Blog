title: 一起来学 SpringBoot 2.x | 第四篇：整合 Thymeleaf 模板
date: 2018-01-05
tag: 
categories: Spring Boot
permalink: Spring-Boot/battcn/v2-web-thymeleaf/
author: 唐亚峰
from_url: http://blog.battcn.com/2018/04/28/springboot/v2-web-thymeleaf/
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484765&idx=1&sn=ab1d20faa09332e8ffb723114ed44414&chksm=fa497aeccd3ef3fad6b876803bbfe27434aa822f144edb717671ffd4377a1f01edaef0d4b288#rd

-------

摘要: 原创出处 http://blog.battcn.com/2018/04/28/springboot/v2-web-thymeleaf/ 「唐亚峰」欢迎转载，保留摘要，谢谢！

- [thymeleaf介绍](http://www.iocoder.cn/Spring-Boot/battcn/v2-web-thymeleaf//)
- [使用](http://www.iocoder.cn/Spring-Boot/battcn/v2-web-thymeleaf//)
- [小技巧](http://www.iocoder.cn/Spring-Boot/battcn/v2-web-thymeleaf//)
- [默认配置](http://www.iocoder.cn/Spring-Boot/battcn/v2-web-thymeleaf//)
- [总结](http://www.iocoder.cn/Spring-Boot/battcn/v2-web-thymeleaf//)
- [说点什么](http://www.iocoder.cn/Spring-Boot/battcn/v2-web-thymeleaf//)

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

在前面几章中已经介绍了如何创建一个`SpringBoot` 项目，同时简单的描述了`SpringBoot REST Web服务`。除此之外它也是支持如`JSP`、`Thymeleaf`、`FreeMarker`、`Mustache`、`Velocity` 等各种模板引擎，同时还为开发者提供了自定义模板扩展的支持。

> 使用嵌入式Servlet容器时，请避免使用JSP，因为使用JSP打包后会存在一些限制。

在`SpringBoot`使用上述模板，默认从 **src/main/resources/templates**下加载。

# thymeleaf介绍

`Thymeleaf`是现代化服务器端的Java模板引擎，不同与其它几种模板的是`Thymeleaf`的语法更加接近HTML，并且具有很高的扩展性。详细资料可以浏览[官网](https://www.thymeleaf.org/)。

> 特点

- 支持无网络环境下运行，由于它支持 html 原型，然后在 html 标签里增加额外的属性来达到模板+数据的展示方式。浏览器解释 html 时会忽略未定义的标签属性，所以 thymeleaf 的模板可以静态地运行；当有数据返回到页面时，Thymeleaf 标签会动态地替换掉静态内容，使页面动态显示。所以它可以让前端小姐姐在浏览器中查看页面的静态效果，又可以让程序员小哥哥在服务端查看带数据的动态页面效果。
- 开箱即用，为`Spring`提供方言，可直接套用模板实现`JSTL、 OGNL`表达式效果，避免每天因套用模板而修改`JSTL、 OGNL`标签的困扰。同时开发人员可以扩展自定义的方言。
- `SpringBoot`官方推荐模板，提供了可选集成模块(`spring-boot-starter-thymeleaf`)，可以快速的实现表单绑定、属性编辑器、国际化等功能。

# 使用

首先要在 `pom.xml` 中添加对 `thymeleaf` 模板依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

然后创建一个 `ThymeleafController` 用来映射HTTP请求与页面的跳转，下面写了两种方式，第一种比较直观和优雅，第二种相对普遍且代码较少，且迎合从`struts2`跳坑的朋友们…

- **Spring4.3以后为简化@RequestMapping(method = RequestMethod.XXX)的写法，故而将其做了一层包装，也就是现在的GetMapping、PostMapping、PutMapping、DeleteMapping、PatchMapping**

```java
package com.battcn.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.servlet.ModelAndView;
import javax.servlet.http.HttpServletRequest;

/**
 * @author Levin
 * @since 2018/4/23 0023
 */
@Controller
@RequestMapping
public class ThymeleafController {

    @GetMapping("/index")
    public ModelAndView index() {
        ModelAndView view = new ModelAndView();
        // 设置跳转的视图 默认映射到 src/main/resources/templates/{viewName}.html
        view.setViewName("index");
        // 设置属性
        view.addObject("title", "我的第一个WEB页面");
        view.addObject("desc", "欢迎进入battcn-web 系统");
        Author author = new Author();
        author.setAge(22);
        author.setEmail("1837307557@qq.com");
        author.setName("唐亚峰");
        view.addObject("author", author);
        return view;
    }

    @GetMapping("/index1")
    public String index1(HttpServletRequest request) {
        // TODO 与上面的写法不同，但是结果一致。
        // 设置属性
        request.setAttribute("title", "我的第一个WEB页面");
        request.setAttribute("desc", "欢迎进入battcn-web 系统");
        Author author = new Author();
        author.setAge(22);
        author.setEmail("1837307557@qq.com");
        author.setName("唐亚峰");
        request.setAttribute("author", author);
        // 返回的 index 默认映射到 src/main/resources/templates/xxxx.html
        return "index";
    }

    class Author {
        private int age;
        private String name;
        private String email;
		// 省略 get set
    }
}
```

最后在 `src/main/resources/templates` 目录下创建一个名 `index.html` 的模板文件，可以看到 `thymeleaf` 是通过在标签中添加额外属性动态绑定数据的

```xml
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml"
      xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <!-- 可以看到 thymeleaf 是通过在标签里添加额外属性来绑定动态数据的 -->
    <title th:text="${title}">Title</title>
    <!-- 在/resources/static/js目录下创建一个hello.js 用如下语法依赖即可-->
    <script type="text/javascript" th:src="@{/js/hello.js}"></script>
</head>
<body>
    <h1 th:text="${desc}">Hello World</h1>
    <h2>=====作者信息=====</h2>
        <p th:text="${author?.name}"></p>
        <p th:text="${author?.age}"></p>
        <p th:text="${author?.email}"></p>
</body>
</html>
```

> 静态效果

双击打开 `index.html` 既可以看到如下的静态效果，并未和其它模板一样显示一堆标签的内容，而是正常渲染静态页面

[![静态效果](https://image.battcn.com/article/images/20180428/springboot/v2-web-thymeleaf/1.png)](https://image.battcn.com/article/images/20180428/springboot/v2-web-thymeleaf/1.png)

> 动态效果

在浏览器输入：<http://localhost:8080/index> 可以看到渲染后的效果，真正意义上的动静分离了

[![动态效果](https://image.battcn.com/article/images/20180428/springboot/v2-web-thymeleaf/2.png)](https://image.battcn.com/article/images/20180428/springboot/v2-web-thymeleaf/2.png)

# 小技巧

> 模板热部署

在 `IntelliJ IDEA` 中使用 `thymeleaf` 模板的时候，发现每次修改静态页面都需要重启才生效，这点是很不友好的，百度了下发现原来是默认配置搞的鬼，为了提高响应速度，默认情况下会缓存模板。如果是在开发中请**将spring.thymeleaf.cache 属性设置成 false**。在每次修改静态内容时**按Ctrl+Shift+F9**即可重新加载了…

> 修改默认`favicon.ico` 图标

默认情况下使用`springboot`总能看到一片叶子，这是因为我们没配置自己的ico导致的，解决方法也很简单，只需要在`src/main/static/`目录下放置一张名为`favicon.ico`就可以了

# 默认配置

`SpringBoot` 默认情况下为我们做了如下的默认配置工作，熟悉默认配置在开发过程中可以更好的解决问题

[![SpringBoot为thymeleaf模板提供的默认配置项](https://image.battcn.com/article/images/20180428/springboot/v2-web-thymeleaf/3.png)](https://image.battcn.com/article/images/20180428/springboot/v2-web-thymeleaf/3.png)

# 总结

Thymeleaf参考手册：<https://blog.csdn.net/zrk1000/article/details/72667478>

WEB MVC详细的内容请参考官方文档：<https://docs.spring.io/spring/docs/5.0.5.RELEASE/spring-framework-reference/web.html#mvc>

目前很多大佬都写过关于 **SpringBoot** 的教程了，如有雷同，请多多包涵，本教程基于最新的 `spring-boot-starter-parent：2.0.1.RELEASE`编写，包括新版本的特性都会一起介绍…

# 说点什么

全文代码：<https://github.com/battcn/spring-boot2-learning/tree/master/chapter3>

# 666. 彩蛋

如果你对 SpringBoot 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)