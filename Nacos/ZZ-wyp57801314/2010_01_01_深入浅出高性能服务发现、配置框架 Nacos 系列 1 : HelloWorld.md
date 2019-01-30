title: 深入浅出高性能服务发现、配置框架 Nacos 系列 1 ：HelloWorld
date: 2010-01-01
tag:
categories: Nacos
permalink: Nacos/wyp57801314/hello-world
author: wyp578013140
from_url: http://blog.51cto.com/2662087/2177877
wechat_url:

-------

摘要: 原创出处 http://blog.51cto.com/2662087/2177877 「wyp578013140」欢迎转载，保留摘要，谢谢！

- [Nacos是什么？](http://www.iocoder.cn/Nacos/wyp57801314/publish-0.3.0/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

#  Nacos是什么？

引用官方的介绍，他主要提供以下几个功能点：

- 动态配置服务
- 服务发现及管理
- 动态DNS服务

**动态配置服务**

就是通过一个系统，管理系统中的配置项，在配置项需要更新的时候，可以通过管理系统去操作更新，更新完了之后，会主动推送到订阅了这个配置的客户端
具体的使用场景，例如，在系统中，会有数据库的链接串，账号密码等配置信息，常规的做法是写在配置文件里面，如果需要修改更新，需要重新打包编译，如果你是分布式集群，那成本太大了，通常我们都会将它抽取出来，存放到db，或者一个管理系统，[Nacos](https://github.com/alibaba/nacos)，就是这个管理系统，[Nacos](https://github.com/alibaba/nacos)还提供主动通知的能力，DB是没有的，在自己的系统代码里面，可以监听某个配置，如果在管理系统上修改了配置项，客户端的监听函数，会立刻执行，在里面，你可以拿到最新的配置，执行自己的业务逻辑

**服务发现及管理**

这个主要是针对分布式的微服务集群系统，某A集群提供服务出去，其他应用集群，需要消费到A集群的服务，需要一个系统去管理A集群的ip列表，其他应用集群，去这个系统才能获取到A集群的ip列表，进行调用，同时该系统需要能够自动将A集群中无法工作的ip进行去除掉，这样才能保证调用方调用成功，[Nacos](https://github.com/alibaba/nacos)就是提供这种能力的一个系统

**动态DNS服务**

这个理解起来也简单，我们平常在代码里面 ，访问一个http的api，通常是带一个域名的，请求的时候，一般会先去DNS域名服务器上面寻找该域名对应的ip，再发起http请求，[Nacos](https://github.com/alibaba/nacos)可以充当这个DNS域名服务器的角色的，优点是什么呢？[Nacos](https://github.com/alibaba/nacos)提供了比DNS域名服务器更方便的管理能力，新增一个域名，只需要在Nacos的控制台上面配置一下，同时它还提供了包括权重，健康检查，属性，路由等DNS服务器不具备的能力，比DNS的产品功能，稳定性，灵活性，超出太多了
[Nacos](https://github.com/alibaba/nacos)除了能够解决上面提到的一些场景，大家可以自由发挥，反正它的能力在那里，至于怎么用，就靠你们聪明的脑袋了。

下面，我们尝试从[Nacos](https://github.com/alibaba/nacos)的官方源码拉一份下来，进行一个服务的发布，订阅的简单流程，让大家更有体感去感受下Nacos的功能。

**下载源码**

[Nacos](https://github.com/alibaba/nacos)的代码是托管在github上，<https://github.com/alibaba/nacos>
开始之前**先start关注一下**，加上watch，后续Nacos的邮件列表也会通知到你，可以关注到Nacos的最新实时消息.

![深入浅出高性能服务发现、配置框架Nacos系列 1: HelloWorld](http://i2.51cto.com/images/blog/201809/20/bcd45781870919d7931acfda74da37e6.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

我们首先直接下载Nacos的源码到本地，下载地址如下图：
![深入浅出高性能服务发现、配置框架Nacos系列 1: HelloWorld](http://i2.51cto.com/images/blog/201809/20/0b0ad76c086e7d70d7dfad5ac0a53b5e.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

然后，要从github上把代码拉下来，命令如下：

```Bash
git clone git@github.com:alibaba/nacos.git
```

执行完之后，出现下图进度
![深入浅出高性能服务发现、配置框架Nacos系列 1: HelloWorld](http://i2.51cto.com/images/blog/201809/20/b3d0722457d1d0da20d4b6db850b539d.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

**编译启动**
下载完后，在本地会有一个nacos的文件夹，进去后，执行编译打包的命令：

```Bash
mvn -Prelease-nacos clean install -U -Dmaven.test.skip=true
```

执行完之后，在nacos/distribution/target文件夹里面，会生成如下几个工程文件：
![深入浅出高性能服务发现、配置框架Nacos系列 1: HelloWorld](http://i2.51cto.com/images/blog/201809/20/b42266d3915a1df8e1a89f71eae1de02.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)
忽略archive-tmp，将nacos-server-0.2.0.zip或者nacos-server-0.2.0.tar.gz直接copy到你的服务器上面，
然后解压开来，得到一个叫nacos的文件夹，里面的结构如下：
![深入浅出高性能服务发现、配置框架Nacos系列 1: HelloWorld](http://i2.51cto.com/images/blog/201809/20/8e1cb3e53a0b9ab162faa357855d8209.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

进入bin文件，里面有启动，关闭nacos server的脚本，我们执行一下启动脚本，注意，nacos server的启动，有2种方式，一种是集群模式，一种是单机模式，由于集群模式还需要一些其他的配置才能运行，今天我们就跑单机模式，需要加一个特殊参数，命令如下：

```Bash
sh startup.sh -m standalone
```

命令执行完之后，出现提示：

```Bash
nacos is starting，you can check the /home/caogu.wyp/test/nacos/logs/start.log
```

我们可以去/nacos/logs/start.log看下启动日志，是否有报错，如果有报错，自己分析一下，找不到原因，联系Nacos管理员 @超哥，他会很热心的帮你解决问题滴。
启动成功的日志如下：
![深入浅出高性能服务发现、配置框架Nacos系列 1: HelloWorld](http://i2.51cto.com/images/blog/201809/20/fb394df6b27bd6609a28eabaffd16397.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

同时， 我们也可以自己check下server的端口是否监听成功，执行

```Bash
netstat -ano|grep 8080
```

![深入浅出高性能服务发现、配置框架Nacos系列 1: HelloWorld](http://i2.51cto.com/images/blog/201809/20/19b8341e528ac67c584638929871fc3d.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

**发布一个服务**

我们现在来用Java客户端，发布和订阅一个服务，客户端，我们有2种方式运行，一个是在自己工程里面，通过maven坐标下载进来，另外一种，直接从下载的[Nacos](https://github.com/alibaba/nacos)源码里面，坐标如下：

```XML
<groupId>com.alibaba.nacos</groupId>
    <artifactId>nacos-client</artifactId>
<version>0.1.0</version>
```

由于我们有代码，我就直接在工程里面进行操作了，工程结构如下，直接在test模块里面，新建一个helloworld的包：
![深入浅出高性能服务发现、配置框架Nacos系列 1: HelloWorld](http://i2.51cto.com/images/blog/201809/20/ddcaa254a8d6e2c43dd4df8246761180.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)
Pub类里面，是发布一条数据，代码如下：

```Java
public class Pub {

    public static void main(String[] args) throws NacosException, InterruptedException {
        //发布的服务名
        String serviceName = "helloworld.services";
        //构造一个nacos实例，入参是nacos server的ip和服务端口
        NamingService naming = NamingFactory.createNamingService("100.81.0.34:8080");
        //发布一个服务，该服务对外提供的ip为127.0.0.1，端口为8888
        naming.registerInstance(serviceName, "127.0.0.1", 8888);
        Thread.sleep(Integer.MAX_VALUE);
    }
}
```

启动之后，服务已经发布上去了

**订阅一个服务**

下面是Sub的代码：

```Java
public class Sub {

    public static void main(String[] args) throws NacosException, InterruptedException {
        //订阅的服务名
        String serviceName = "helloworld.services";
        //创建一个nacos实例
        NamingService naming = NamingFactory.createNamingService("100.81.0.34:8080");
        //订阅一个服务
        naming.subscribe(serviceName, event -> {
            if (event instanceof NamingEvent) {

                System.out.println("订阅到数据");
                System.out.println(((NamingEvent) event).getInstances());
            }
        });
        System.out.println("订阅完成，准备等数来");
        Thread.sleep(Integer.MAX_VALUE);
    }
}
```

我们尝试启动，会订阅到刚才发布的数据：

![深入浅出高性能服务发现、配置框架Nacos系列 1: HelloWorld](http://i2.51cto.com/images/blog/201809/20/470c9ce5d5f13915f4d34d1916cfa277.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

**总结**

上面流程，就是一个helloworld的服务发布和订阅，够简单吧，在实际的生产环境中，一个服务下面往往挂有好几个ip提供服务，在订阅端，拿到服务下面的所有地址列表后，通过负载均衡策略，例如随机策略，选择到一个ip进行调用，这样就完成了一次远程调用。