title: 深入浅出高性能服务发现、配置框架 Nacos 系列 2 ：Nacos项目结构介绍
date: 2010-01-06
tag:
categories: Nacos
permalink: Nacos/wyp57801314/project-module-intro
author: wyp578013140
from_url: http://blog.51cto.com/2662087/2182342
wechat_url:

-------

摘要: 原创出处 http://blog.51cto.com/2662087/2182342 「wyp578013140」欢迎转载，保留摘要，谢谢！

- [console模块](http://www.iocoder.cn/Nacos/wyp57801314/project-module-intro/)
- [config模块](http://www.iocoder.cn/Nacos/wyp57801314/project-module-intro/)
- [naming](http://www.iocoder.cn/Nacos/wyp57801314/project-module-intro/)
- [core模块](http://www.iocoder.cn/Nacos/wyp57801314/project-module-intro/)
- [common模块](http://www.iocoder.cn/Nacos/wyp57801314/project-module-intro/)
- [client模块](http://www.iocoder.cn/Nacos/wyp57801314/project-module-intro/)
- [api模块](http://www.iocoder.cn/Nacos/wyp57801314/project-module-intro/)
- [总结](http://www.iocoder.cn/Nacos/wyp57801314/project-module-intro/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

今天，我们分析一下Nacos工程的包模块结构，都是负责什么功能的，从全局看一下整个工程，从整体到细节，还没下载源码的同学，赶紧动起来！<https://github.com/alibaba/nacos>，这个是github代码地址，开始之前先start关注一下，加上watch，后续[Nacos](https://github.com/alibaba/nacos)的邮件列表也会通知到你，可以关注到Nacos的最新实时消息，及各大牛之间的精彩讨论。

截止本文发出，代码最新是master分支上0.2.0版本的，新开源版迭代会比较频繁，很可能某个类，或者模块依赖关系，下个版本就不一样了，请不要疑惑。

通过分析[Nacos](https://github.com/alibaba/nacos)源码工程中的pom模块依赖，画出了如下每个模块的依赖关系，每个component就对应者工程里面的一个maven module，我们重点关注一下工程代码相关的模块的依赖关系，依赖关系从上往下，下图右边部分：

![img](https://cdn.nlark.com/lark/0/2018/png/4232/1537615415981-071b6819-6643-49c3-94fa-68a7e54908fd.png)

接下来，将介绍一下每个模块在[Nacos](https://github.com/alibaba/nacos)工程中的职责及依赖关系。

# console模块

目前里面是空的，根据[Nacos](https://github.com/alibaba/nacos)的版本规划，预计在0.3.0版本里面，会在这个模块里面放入[Nacos](https://github.com/alibaba/nacos)控制台相关代码，及前端资源，它依赖config和naming模块，由此可以看出，consule前端依赖的api，将直接由这2个模块进行提供

# config模块

大家知道，[Nacos](https://github.com/alibaba/nacos)目前有3大核心功能，其中一个是动态配置发现，这个包里面，放的就是配置中心的代码，打开模块，内部的包结构如下图：

![img](https://cdn.nlark.com/lark/0/2018/png/4232/1537618066888-24b2beae-64b0-4058-afb3-ba58211413a9.png)

基本上从名字都能猜出个大概，这里就不详细讲解里面的具体实现了，后续有章节介绍。工程是基于spring boot构建的，所以会有一个入口类Config，里面打开就是Spring boot玩家熟悉的spring boot加载入口方法了，SpringApplication.run(Config.class, args)

# naming

这个是[Nacos](https://github.com/alibaba/nacos)的另外一个核心功能的代码模块，动态服务发现，内部的包结构如下：

![img](https://cdn.nlark.com/lark/0/2018/png/4232/1537619090629-6bdab507-eec4-42f6-905d-dc766e64749c.png)

和config模块类似，NamingApp是spring boot的入口类，其他的都通过包隔离了，各个包的名字，大家也可以猜出大概是承担什么功能的

# core模块

从上图可以看出，由config直接依赖，目前里面是空的，看来还是属于规划之中，从core上里面，这里可能会抽取出一些核心服务，供多个功能模块使用，例如数据层，一致性协议等

# common模块

目前由config通过core间接依赖到，翻开了里面的代码，是一些公用的工具类，如下图所示：

![img](https://cdn.nlark.com/lark/0/2018/png/4232/1537619432951-b8a703c1-b8ff-4515-84a3-967dc52ce1ba.png)

目前只有这些工具类，naming目前没有依赖到，但是后续和config模块进行融合时，这里面后续会放入共同依赖的一些类，例如请求对象，异常，协议结构等

# client模块

这个里面放的是[Nacos](https://github.com/alibaba/nacos)客户端的代码，服务发现和配置管理2个功能的客户端，目前在工程模块上已经做了统一，但是，在代码包上面，还是分离到，可以从包结构里面看出，如下：

![img](https://cdn.nlark.com/lark/0/2018/png/4232/1537622541414-fefcc655-9c99-4a85-a430-4077419b2bde.png)

很明显分了2个模块，配置和naming，后续随着整合到深度，这些都会陆续合并掉，真正做到统一

# api模块

这个里面，主要是把naming和config的api进行了抽取，从结构上看更清晰一些，api的具体实现，都还在client模块里面，它包结构如下：

![img](https://cdn.nlark.com/lark/0/2018/png/4232/1537624566403-58840c48-85b0-42ff-a0df-156ef50c95ce.png)

在[Nacos](https://github.com/alibaba/nacos)的使用手册中，使用到的工厂类，NacosFactory在构建对应实例时，调用的就是api里面的，根据不同的方法，再通过反射构造出对应的实例：

```Java
public static ConfigService createConfigService(String serverAddr) throws NacosException {
   Properties properties = new Properties();
   properties.put(PropertyKeyConst.SERVER_ADDR, serverAddr);
   try {
      Class<?> driverImplClass = Class.forName("com.alibaba.nacos.client.config.NacosConfigService");
      Constructor constructor = driverImplClass.getConstructor(Properties.class);
      ConfigService vendorImpl = (ConfigService) constructor.newInstance(properties);
      return vendorImpl;
   } catch (Throwable e) {
      throw new NacosException(-400, e.getMessage());
   }}
```

- 其他非工程源码的模块，是用来放测试用例，文档，打包文件，启动脚本的，这个就不详细讲了，大家有兴趣可以翻开来看看。

# 总结

[Nacos](https://github.com/alibaba/nacos)的工程，包含了Nacos Server和Nacos Client2个工程的代码，后续会再console的代码也放入总工程中，这些代码都放在一起，对使用和学习者来说，是非常方便的，其中，Nacos Server，由配置和服务发现2个大功能，分别由config和naming承载，client和naming模块的代码，都没有共同依赖的类，由此可以看出，client与server之间的协议，是和语言没有关系，没有涉及到自己的私有应用层协议，这样的话，多语言客户端的接入，会简单很多，在客户端的模块里面，这2个功能就统一到了NacosClient模块里面，server目前还是不同的模块，后续这块需要看Nacos的架构融合演进。

总的来说，目前[Nacos](https://github.com/alibaba/nacos)的工程，将框架都搭好了，但是一些整合的实现和优化，都还未清晰，好几个包都是空的，功能和系统架构，还在规划之中，因此大家多参与进来，有很多的机会和空间给你们发挥，你的提议很可能会成为[Nacos](https://github.com/alibaba/nacos)的标准，服务于世界级工程应用！