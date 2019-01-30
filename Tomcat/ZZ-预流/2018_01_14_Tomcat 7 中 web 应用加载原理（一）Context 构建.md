title: Tomcat 7 中 web 应用加载原理（一）Context 构建
date: 2018-01-14
tag: 
categories: Tomcat
permalink: Tomcat/yuliu/Web-application-loading-principle-1-Context-construction
author: 预流
from_url: https://juejin.im/post/5a7976a95188257a666ef0f1
wechat_url: 

-------

摘要: 原创出处 https://juejin.im/post/5a7976a95188257a666ef0f1 「预流」欢迎转载，保留摘要，谢谢！


-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

为什么关心 Tomcat 中一个 web 应用的加载过程？在前面的文章中看过多次 Tomcat 的组件结构图，这里再贴出来回顾一下：

![img](https://user-gold-cdn.xitu.io/2018/2/6/1616a782ea61d7b6?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 之前的

Tomcat 7 启动分析

系列文章中看到 Tomcat 启动的时候将会解析 server.xml，根据里面所配置的各个节点信息逐一初始化和启动相应组件（即分别调用它们的 init 和 start 方法），但浏览一下 Tomcat 7 源码中的 server.xml 的内容，里面对应上图中的已经默认配置的各级组件包括 Server、Service、Engine、Connector、Host、Valve 。上图中的 Context 组件实际就是我们通常所说的一个 web 应用，有趣的是在 server.xml 中并没有配置该组件，而我们默认启动的时候实际上已经有好几个 web 应用可以访问了： 这个到底是怎么回事？



看过前面的[Tomcat 7 的一次请求分析](https://link.juejin.im?target=https%3A%2F%2Fjuejin.im%2Fpost%2F5a7180f2f265da3e377c5620)系列文章的人应该知道，浏览器一次请求送到 Tomcat 服务器之后，最终会根据浏览器中的 url 路径找到相应的实际要访问的 web 应用的 context 对象（默认即`org.apache.catalina.core.StandardContext`类的实例）。比如访问的 url 为`http://localhost:8080/`，那么将会送到上图 ROOT 文件夹表示的 web 应用中，访问的 url 为`http://localhost:8080/docs`，那么将访问 docs 文件夹表示的 web 应用。所以能够猜到的是在 Tomcat 启动完成后，必定容器内部已经构造好了表示相应web应用的各个 context 对象。

本文就对这个问题一探究竟。在[Tomcat 7 服务器关闭原理](https://link.juejin.im?target=https%3A%2F%2Fjuejin.im%2Fpost%2F5a6d77916fb9a01c9c1f4440)的开头提到，默认的配置下 Tomcat 启动完之后会看到后台实际上总共有 6 个线程在运行：

![img](https://user-gold-cdn.xitu.io/2018/2/6/1616a7cb903f7687?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 前面的几篇文章中涉及了

`main`、`http-bio-8080-Acceptor-0`、`http-bio-8080-AsyncTimeout`、`ajp-bio-8009-Acceptor-0`、`ajp-bio-8009-AsyncTimeout`，已经谈到了这些线程的作用，它们是如何产生并响应请求的。但有一个线程没有说，即`ContainerBackgroundProcessor[StandardEngine[Catalina]]`，而本文要解答的问题奥秘就在这个线程之中。



先看看这个线程是如何产生的，其实从命名就可以看出一些端倪，它叫做容器后台处理器，并且跟 StandardEngine 关联起来，它的产生于作用也同样如此。

Tomcat 7 中所有的默认容器组件（ StandardEngine、StandardHost、StandardContext、StandardWrapper ）都会继承父类`org.apache.catalina.core.ContainerBase`，在这些容器组件启动时将会调用自己内部的 startInternal 方法，在该方法内部一般会调用父类的 startInternal 方法（ StandardContext 类的实现除外），比如`org.apache.catalina.core.StandardEngine`类中的 startInternal 方法：

![img](https://user-gold-cdn.xitu.io/2018/2/6/1616a7fd8c0dad07?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 最后的

`super.startInternal()`

即调用父类

`org.apache.catalina.core.ContainerBase的startInternal`

方法，在该方法最后：

![img](https://user-gold-cdn.xitu.io/2018/2/6/1616a80dd58f1b2d?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 第 6 行设置了

```
LifecycleState.STARTING
```

状态（这样将向容器发布一个

```
Lifecycle.START_EVENT
```

事件），这一行的作用本文后面会提到，暂且按下不表。第 9 行调用 threadStart 方法，看看 threadStart 方法的代码：

![img](https://user-gold-cdn.xitu.io/2018/2/6/1616a81cecaf30d2?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 这里可以看到如果两个前置校验条件通过的话将会启动一个线程，并且线程的名字即以

```
ContainerBackgroundProcessor[
```

开头，线程名字后面取的是对象的 toString 方法，以 StandardEngine 为例，看看

```
org.apache.catalina.core.StandardEngine
```

的 toString 方法实现：

![img](https://user-gold-cdn.xitu.io/2018/2/6/1616a879a890c1cd?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 以上解释了这个后台线程的来历。



但这里有一个问题，既然 StandardEngine、StandardHost 都会调用`super.startInternal()`方法，按默认配置，后台理应产生两个后台线程，实际为什么只有一个？

回到`org.apache.catalina.core.ContainerBase`的 threadStart 方法，在启动线程代码之前有两个校验条件：

![img](https://user-gold-cdn.xitu.io/2018/2/6/1616a889d17fbd9d?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 容器组件对象初始化时 thread 为

```
null
```

，backgroundProcessorDelay 是

```
-1
```

![img](https://user-gold-cdn.xitu.io/2018/2/6/1616a89715c12f06?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 但

```
org.apache.catalina.core.StandardEngine
```

在其自身构造函数中做了一点修改：

![img](https://user-gold-cdn.xitu.io/2018/2/6/1616a8a0b912e8ea?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 构造函数最后将父类的 backgroundProcessorDelay 的值由

```
-1
```

改成了

```
10
```

，所以 Tomcat 启动解析 xml 时碰到一个 Engine 节点就会对应产生一个后台处理线程。



讲完了这个后台处理线程的产生，看看这个线程所作的事情，再看下这个线程的启动代码：

![img](https://user-gold-cdn.xitu.io/2018/2/6/1616a8ad5d0bc758?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 所以这个线程将会执行 ContainerBase 的内部类 ContainerBackgroundProcessor 的 run 方法，看下 ContainerBackgroundProcessor 的全部实现代码：

![img](https://user-gold-cdn.xitu.io/2018/2/6/1616a8f89979cfbe?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/6/1616a8fafda1af4f?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 在它的 run 方法暂停一段时间之后会调用 processChildren 方法，而 processChildren 方法做了两件事，一是调用容器组件自身的 backgroundProcess 方法，而是取出该容器组件的所有子容器组件并调用它们的 processChildren 方法。归结起来这个线程的实现就是定期通过递归的方式调用当前容器及其所有子容器的 backgroundProcess 方法。



而这个 backgroundProcess 方法在 ContainerBase 内部已经给出了实现：

![img](https://user-gold-cdn.xitu.io/2018/2/6/1616a90d7d35375d?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/6/1616a918d26bcff1?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 这段代码就不一一解释了，概括起来说就是逐个调用与容器相关其它内部组件的 backgroundProcess 方法。最后注册一个

```
Lifecycle.PERIODIC_EVENT
```

事件。



上面就是 Tomcat 7 的后台处理线程所作的事情的概述，在 Tomcat 的早期版本中有一些后台处理的事情原来是在各个组件内部分别自定义一个线程并启动，在 Tomcat 5 中改成了所有后台处理共享同一线程的方式。

回到本文要解答的问题，web 应用如何加载到容器中的？在 ContainerBase 类的 backgroundProcess 方法的最后：

```Java
fireLifecycleEvent(Lifecycle.PERIODIC_EVENT, null);

```

向容器注册了一个`PERIODIC_EVENT`事件。前面说道默认的`ContainerBackgroundProcessor[StandardEngine[Catalina]]`线程会定期（默认为 10 秒）执行 Engine、Host、Context、Wrapper 各容器组件及与它们相关的其它组件的 backgroundProcess 方法，所以也会定期向 Host 组件发布一个`PERIODIC_EVENT`事件，这里看下 StandardHost 都会关联的一个监听器`org.apache.catalina.startup.HostConfig`：

> 在 Tomcat 启动解析 xml 时`org.apache.catalina.startup.Catalina`类的 386 行：`digester.addRuleSet(new HostRuleSet("Server/Service/Engine/"))`
>
> 在 HostRuleSet 类的 addRuleInstances 方法中：
>
> ![img](https://user-gold-cdn.xitu.io/2018/2/6/1616a944a79319bd?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)
>
>
>
> 第 9 到 12 行看到，所有 Host 节点都会添加一个`org.apache.catalina.startup.HostConfig`对象作为`org.apache.catalina.core.StandardHost`对象的监听器

在 HostConfig 的 lifecycleEvent 方法中可以看到如果 Host 组件收到了 Lifecycle.PERIODIC_EVENT 事件的发布所作出的响应（如果对 Tomcat 7 的 Lifecycle 机制不清楚可以看下[Tomcat 7 启动分析（五）Lifecycle 机制和实现原理](https://link.juejin.im?target=https%3A%2F%2Fjuejin.im%2Fpost%2F5a6d73a36fb9a01cba42d1d7)：

![img](https://user-gold-cdn.xitu.io/2018/2/6/1616a978854d7c84?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 第 17 行，如果发布的事件是

```
PERIODIC_EVENT
```

将会执行 check 方法。第 19 行，如果发布的事件是

```
START_EVENT
```

则执行 start 方法。check 方法和 start 方法最后都会调用 deployApps() 方法，看下这方法的实现：

![img](https://user-gold-cdn.xitu.io/2018/2/6/1616a99150c441a9?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 这里即各种不同方式发布 web 应用的代码。



本文前面提到默认情况下组件启动的时候会发布一个`Lifecycle.START_EVENT`事件（在`org.apache.catalina.core.ContainerBase`类的 startInternal 方法倒数第二行），回到 HostConfig 的 lifecycleEvent 方法中，所以默认启动时将会执行 HostConfig 的 start 方法，在该方法的最后：

```Java
if (host.getDeployOnStartup())
    deployApps();

```

因为默认配置 host.getDeployOnStartup() 返回 true ，这样容器就会在启动的时候直接加载相应的 web 应用。

当然，如果在 server.xml 中 Host 节点的 deployOnStartup 属性设置为 false ，则容器启动时不会加载应用，启动完之后不能立即提供 web 应用的服务。但因为有上面提到的后台处理线程在运行，会定期执行 HostConfig 的 check 方法：

![img](https://user-gold-cdn.xitu.io/2018/2/6/1616a9aeadd62a6c?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 如果 Host 节点的 autoDeploy 属性是 true（默认设置即为 true ），可以看到 check 方法最后同样会加载 web 应用。