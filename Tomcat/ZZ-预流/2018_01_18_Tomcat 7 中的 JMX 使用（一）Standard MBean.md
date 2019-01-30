title: Tomcat 7 中的 JMX 使用（一）Standard MBean
date: 2018-01-18
tag: 
categories: Tomcat
permalink: Tomcat/yuliu/JMX-1-Standard-MBean
author: 预流
from_url: https://juejin.im/post/5a9f6be86fb9a028d82b4f3a
wechat_url: 

-------

摘要: 原创出处 https://juejin.im/post/5a9f6be86fb9a028d82b4f3a 「预流」欢迎转载，保留摘要，谢谢！


-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

做过 Java 平台下的应用服务器监控的对 JMX 应该不会陌生，简单说 JMX 就是提供了一个标准的管理方案的框架。这里所说的管理的含义包括监控平台运行状况、应用级别配置资源、收集应用统计数据、调试、监视服务器性能，JMX  允许你将所有的资源（硬件和软件）打包成 java  对象，然后将他们暴露在分布式环境中，并且 JMX  提供了一个机制，可以很简单的将既存的管理协议，如 SNMP ，映射到 JMX  自己的管理结构中。

本文重点不是介绍 JMX ，而是分析 Tomcat 7 中是如何用 JMX 来提供管理功能的，如果对 JMX 并不熟悉可以先 Google 一下，了解一下这个技术，网上已经有一些中文技术博客的介绍，如 BlogJava 里 **子在川上曰** 的[ JMX 一步步来](https://link.juejin.im?target=http%3A%2F%2Fwww.blogjava.net%2Fchengang%2Farchive%2F2013%2F10%2F07%2F34061.html)、《JMX IN ACTION》的一些翻译文章。当然，最权威的还是看看 oracle 的官方文档，这里提供[JMX 1.4 规范的官方链接](https://link.juejin.im?target=http%3A%2F%2Fdocs.oracle.com%2Fjavase%2F7%2Fdocs%2Ftechnotes%2Fguides%2Fjmx%2FJMX_1_4_specification.pdf)。

先来看下 Tomcat 7里由 JMX 提供的管理功能，在 Tomcat 启动完之后可以用 jconsole 来访问：

![img](https://user-gold-cdn.xitu.io/2018/3/7/161fec3917fdf25b?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 选择 Tomcat 启动后所属的进程，这里是我本机启动 Tomcat 的例子，当然也可以访问远程 Tomcat 。连接成功后会看到：

![img](https://user-gold-cdn.xitu.io/2018/3/7/161fec3ffb4caf32?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 在 MBean 这一页里可以看到 Tomcat 提供的当前可以使用的 MBean 。



JMX 标准提供了四种不同的 MBean ：

- **Standard MBean**，该 MBean 直接实现用于管理对象的方法，既可以通过实现一个由程序员定义的、类名以 “MBean” 结束的接口，也可以使用一个以一个类作为构造函数参数的 Standard MBean 实例，加上一个可选的接口类规范。这个接口可以开放用于管理的部分对象方法。
- **Dynamic MBean**，该 MBean 用属性访问器动态地访问属性，并用一个一般化的 invoke() 方法调用方法。可用的方法是在 MBeanInfo 接口中指定的。这种方式更灵活，但是不具有像 Standard MBean 那样的类型安全性。它极大地降低了耦合性，可管理的 POJO（纯粹的老式 Java 对象）不需要实现特定的接口。
- **Model MBean**，该 MBean 提供了一个改进的抽象层，并扩展了 Dynamic MBean 模型以进一步减少对给定实现的依赖性。这对于可能使用多个版本的 JVM 或者需要用松散耦合管理第三方类的情况会有帮助。Dynamic MBean 与 Model MBean 之间的主要区别是，在 Model MBean 中有额外的元数据。
- **Open MBean**，该 MBean 是受限的 Model MBean，它限制类型为固定的一组类型，以得到最大的可移植性。通过限制数据类型，可以使用更多的适配器，并且像 SMTP 这样的技术可以更容易适应 Java 应用程序的管理。这种变体还指定了数组和表等标准结构以改进复合对象的管理。

在 Tomcat 7 中可以看到标准 MBean（Standard MBean）和动态 MBean（Dynamic MBean）的使用，本文就介绍这两种 MBean。先来看下比较简单的标准 MBean：

在 Tomcat 的启动类`org.apache.catalina.startup.Bootstrap的createClassLoader` 方法最后一部分：

```Java
ClassLoader classLoader = ClassLoaderFactory.createClassLoader
    (repositories, parent);

// Retrieving MBean server
MBeanServer mBeanServer = null;
if (MBeanServerFactory.findMBeanServer(null).size() > 0) {
    mBeanServer = MBeanServerFactory.findMBeanServer(null).get(0);
} else {
    mBeanServer = ManagementFactory.getPlatformMBeanServer();
}

// Register the server classloader
ObjectName objectName =
    new ObjectName("Catalina:type=ServerClassLoader,name=" + name);
mBeanServer.registerMBean(classLoader, objectName);

```

从`ClassLoaderFactory.createClassLoader`方法的最后一部分实现代码：

```Java
return AccessController.doPrivileged(
        new PrivilegedAction<StandardClassLoader>() {
            @Override
            public StandardClassLoader run() {
                if (parent == null)
                    return new StandardClassLoader(array);
                else
                    return new StandardClassLoader(array, parent);
            }
        });

```

可以看出上面的 classLoader 对象实际是`org.apache.catalina.loader.StandardClassLoader`类的实例。看这个类的定义：

```Java
public class StandardClassLoader
    extends URLClassLoader
    implements StandardClassLoaderMBean

```

它实现了一个StandardClassLoaderMBean 接口。从这里就可以看出最上面的代码`mBeanServer.registerMBean`中注册的实际上就是一个**Standard MBean**。只是这个标准 MBean 很没意思，一个方法都没开放出去管理，所以 jconsole 里只能看到 MBean 的描述信息，看不到它的属性、方法：

![img](https://user-gold-cdn.xitu.io/2018/3/7/161fecaa52d28bdf?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)