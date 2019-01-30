title: Tomcat 7 中 web 应用加载原理（二）web.xml 解析
date: 2018-01-15
tag: 
categories: Tomcat
permalink: Tomcat/yuliu/Web-application-loading-principle-2-web-xml-parsing
author: 预流
from_url: https://juejin.im/post/5a7aa6f4f265da4e7e10a0aa
wechat_url: 

-------

摘要: 原创出处 https://juejin.im/post/5a7aa6f4f265da4e7e10a0aa 「预流」欢迎转载，保留摘要，谢谢！


-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

[前一篇文章](http://www.iocoder.cn/Tomcat/yuliu/Web-application-loading-principle-1-Context-construction)讲了`org.apache.catalina.startup.HostConfig`的 lifecycleEvent 方法中所做的事情。最后看到在 Tomcat 启动时或启动后（后台线程定时扫描）会调用 HostConfig 类的 deployApps 方法：

![img](https://user-gold-cdn.xitu.io/2018/2/7/1616f1df2ceeeeb6?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 可以看到这里部署应用有三种方式：XML 文件描述符、WAR 包、文件目录。三种方式部署的总体流程很相似，都是一个 web 应用分配一个线程来处理，这里统一放到与 Host 内部的线程池对象中（ startStopExecutor ），所以有时会看到在默认配置下 Tomcat 启动后可能有一个叫

```
-startStop-
```

的线程还会运行一段时间才结束。但浏览这三种部署方式的实现代码，里面都是构建一个 Context 对象，并将构建好的 Context 对象与 Host 组件关联起来（即调用

```
host.addChild(context)
```

这句，具体代码在 HostConfig 类的

```
deployDescriptor(ContextName cn, File contextXml)
```

、

```
deployDirectory(ContextName cn, File dir)
```

、

```
deployWAR(ContextName cn, File war)
```

三个方法中，这里不再贴出代码来详细分析）。



前一篇文章只分析到这步，可以看出与一个 web 应用相对应的一个 Context 对象已经构建出来了，但如果容器只执行到这里根本无法响应一个浏览器的一次请求。就 web 服务器的实现来看一次请求过来除了需要根据内部 Context 构建找到这次请求访问的web应用具体所对应的 Context 对象，还需要包含 web 应用中具体的哪个 Servlet 来处理这次请求，中间是否还需要执行相应的过滤器( filter )、监听器( listener )等，做过 java 的 web 开发的同学都知道，这些信息是配置在一个 web 应用的`WEB-INF\web.xml`文件的（servlet3 中已经支持将这些配置信息放到 Java 文件的注解中，但万变不离其宗，总归要在 web 应用的某个地方说明，并在容器启动时加载，这样才能真正提供 web 服务，响应请求）。

看到这里可以猜到 Tomcat 容器加载 web 应用时必定会有对于每个应用的 web.xml 文件的解析过程，本文就来看看这个解析过程。

在本文开头提到的三种部署应用的实现代码中有一些共通的代码，这里摘出来说明一下：

```Java
Class<?> clazz = Class.forName(host.getConfigClass());
LifecycleListener listener =
    (LifecycleListener) clazz.newInstance();
context.addLifecycleListener(listener);

host.addChild(context);

```

第一段是在所有 Context 对象构建时会添加一个监听器，这里监听器的类名是 StandardHost 类的实例变量 configClass ，其默认值就是`org.apache.catalina.startup.ContextConfig`。第二段是将当前构建的 Context 对象添加到父容器 Host 对象中。

先看 StandardHost 的 addChild 方法的实现：

![img](https://user-gold-cdn.xitu.io/2018/2/7/1616f234c7d52aea?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 可以看到这段代码最后调用了父类的 addChild 方法：

![img](https://user-gold-cdn.xitu.io/2018/2/7/1616f23ebe7c71fe?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 这里看下 addChildInternal 方法的实现：

![img](https://user-gold-cdn.xitu.io/2018/2/7/1616f2469d4cc514?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 可以看到会调用子容器的 start 方法，就是指调用 StandardContext 的 start 方法。



即给 host 对象添加子容器时将会调用子容器的 start 方法，按照[前面文章](https://link.juejin.im?target=https%3A%2F%2Fjuejin.im%2Fpost%2F5a6d6f6751882573520da54d)的分析，调用 StandardContext 的 start 方法最终会调用`org.apache.catalina.core.StandardContext`类的 startInternal 方法（该方法代码较长，建议自己阅读，不再贴出），这里将会发布一系列事件，按调用前后顺序这些事件包括：`BEFORE_INIT_EVENT`、`AFTER_INIT_EVENT`、`BEFORE_START_EVENT`、`CONFIGURE_START_EVENT`、`START_EVENT`、`AFTER_START_EVENT`。

前面提到在构建 Context 对象时都会注册一个监听器`org.apache.catalina.startup.ContextConfig`，看下这个类的 lifecycleEvent 方法中（为什么会执行这个方法可以看[这篇文章](https://link.juejin.im?target=https%3A%2F%2Fjuejin.im%2Fpost%2F5a6d73a36fb9a01cba42d1d7)的分析）监听了哪些事件：

![img](https://user-gold-cdn.xitu.io/2018/2/7/1616f2757ad32363?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 与 Context 的 start 方法调用相关的事件监听前后顺序为：

```
AFTER_INIT_EVENT
```

（执行 init 方法）、

```
BEFORE_START_EVENT
```

（执行 beforeStart 方法）、

```
CONFIGURE_START_EVENT
```

（执行 configureStart 方法）。



在 configureStart 方法将直接调用 webConfig 方法，正是在这个方法中将会解析 web.xml 文件：

![img](https://user-gold-cdn.xitu.io/2018/2/7/1616f2a716667742?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/7/1616f2aa1948ad4b?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/7/1616f2ad0f0fe296?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/7/1616f2b0396c9998?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/7/1616f2b36258a6af?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/7/1616f2b5a7497d1b?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 这个方法里面做的事情，在英文注释中说的很清楚了，概括起来包括合并 Tomcat 全局 web.xml 、当前应用中的 web.xml 、web-fragment.xml 和 web 应用的注解中的配置信息，并将解析出的各种配置信息（如 servlet 配置、filter 配置等）关联到 Context 对象中（在上面的代码第 140 行：

```
webXml.configureContext(context)
```

）。



看下 configureContext 方法：

![img](https://user-gold-cdn.xitu.io/2018/2/7/1616f2fb18a7e0f9?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/7/1616f2fecccf4767?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/7/1616f3038a6ffec2?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/7/1616f307069e3d98?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/7/1616f30ac60438e6?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/7/1616f30e9abd47e1?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/7/1616f3111711b494?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 可以看到里面对 context 调用了各种 set、add 方法，从而将 web.xml 中的各种配置信息与表示一个 web 应用的 context 对象关联起来。