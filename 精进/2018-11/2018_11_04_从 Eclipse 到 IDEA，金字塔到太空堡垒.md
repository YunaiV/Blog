title: 从 Eclipse 到 IDEA，金字塔到太空堡垒
date: 2018-11-04
tags: IntelliJ IDEA
categories: 精进
permalink: Fight/From-Eclipse-to-IDEA-pyramids-to-battlestar
author: 大大枣
from_url: https://my.oschina.net/lizaizhong/blog/2051414
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247485509&idx=2&sn=358781ade936d135e9aeab4c612228dc&chksm=fa4977f4cd3efee2c88078bc4b9ee900d5327a83aa307ee2d543b40dcf0b0c3c0e3af105757e&token=696637778&lang=zh_CN#rd

-------

摘要: 原创出处 https://my.oschina.net/lizaizhong/blog/2051414 「大大枣」欢迎转载，保留摘要，谢谢！

- [1. 前言](http://www.iocoder.cn/Fight/From-Eclipse-to-IDEA-pyramids-to-battlestar/)
- [2. IDEA 有什么好？](http://www.iocoder.cn/Fight/From-Eclipse-to-IDEA-pyramids-to-battlestar/)
  - [2.1 插件安装](http://www.iocoder.cn/Fight/From-Eclipse-to-IDEA-pyramids-to-battlestar/)
  - [2.2 “专注”窗口](http://www.iocoder.cn/Fight/From-Eclipse-to-IDEA-pyramids-to-battlestar/)
  - [2.3 调试](http://www.iocoder.cn/Fight/From-Eclipse-to-IDEA-pyramids-to-battlestar/)
  - [2.4 Git 的使用](http://www.iocoder.cn/Fight/From-Eclipse-to-IDEA-pyramids-to-battlestar/)
- [3. 怎么走出 Eclipse 的舒适区](http://www.iocoder.cn/Fight/From-Eclipse-to-IDEA-pyramids-to-battlestar/)
  - [3.1 Maven 项目导入和启动 Tomcat](http://www.iocoder.cn/Fight/From-Eclipse-to-IDEA-pyramids-to-battlestar/)
  - [3.2 快捷键映射](http://www.iocoder.cn/Fight/From-Eclipse-to-IDEA-pyramids-to-battlestar/)
  - [3.3 快捷键被占用问题](http://www.iocoder.cn/Fight/From-Eclipse-to-IDEA-pyramids-to-battlestar/)
  - [3.4 关闭部分“建议”](http://www.iocoder.cn/Fight/From-Eclipse-to-IDEA-pyramids-to-battlestar/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# 1. 前言

工欲善其事，必先利其器。对于程序员来说，具有生产力的工具能让你事半功倍，心情大好。
两个月前从Eclipse转到了InteliJ IDEA，原先常用Eclipse如同身边的保温杯，如果不出毛病，大概我是不会考虑换掉他。中间想偶尔尝试一下IDEA，因为遇到各种不适应，就退回来了。这次要换掉Eclipse是因为一个莫名的bug导致我项目编译卡死，反复出现好几次后，决定弃用他。

Tips：下面博文中的图片都比较大，可以右键在新标签打开查看大图！

# 2. IDEA 有什么好？

换到IDEA并渐渐适应之后，用一个词形容这款IDE，就是：现代。
再看Eclipse，如同埃及金字塔，精巧但粗粝、宏伟但迟钝。而IDEA如同太空堡垒，精致大气、舒适内敛。
因为我对IDEA非常有好感，决定向你推销一下。让我们先来看看他那些让人不可自拨的功能吧：

## 2.1 插件安装

在Eclipse上安装一款插件，要到marketplace中搜索，点击install。因为跨国网络访问，所以一般下载插件会很慢。
在IDEA上安装插件，逻辑相似，Ctrl+Alt+S呼出Settings，找到Plugins进行搜索，如果没有找到会跳转到远程仓库，然后install即可。
不过对于有些插件来说，IDEA上的安装流程比Eclipse顺畅了不止一个级别，比如大家常用的lombok，Eclipse上安装过程繁琐不说了，IDEA上呢：
![img](http://static.iocoder.cn/86d0bea5888ecfc335ee2a1b5fbf37e0)
如果没有安装，点击右侧install，然后重启IDEA即可。

不光是lombok，诸如GoLang、C、Python的这种语言扩展插件，IDEA上的体验也比Eclipse好上一个等级。例如Go插件：
![img](http://static.iocoder.cn/0a1455125379e58ae76858eab783a32a)
IDEA自带了智能提示，不用配置，开箱即用。最重要的是，在IDEA中开发Go和开发Java一样顺手，快捷键是一致的，提示与代码生成、插入是一致的，这在Eclipse上几乎不可实现。我安装过Eclipse的Python语言扩展插件，和在Eclipse上编写Java是有一定落差的，而在IDEA上不会，他给你的体验是一致的，这个必须赞。

更棒的时，IDEA自带了一些非常好用的插件，如HTTP Client：
![img](http://static.iocoder.cn/f19c828bbddc5e7e4aa0790f490c8858)

## 2.2 “专注”窗口

再来看一个“现代”的例子，在IDEA中窗口都是可定制的，如同太空堡垒中的房价可移动一般。
在IDEA中你的界面可能是这样的：
![img](http://static.iocoder.cn/c0233cbc3925c34894e2ff236402755a)
只需要按下Ctrl+Shift+F12就会进入专注模式，变成了这样：
![img](http://static.iocoder.cn/1723a6c5b066e3862c4c4f34339cf45c)
所有窗口都隐藏起来了。
我为什么如此喜欢IDEA的窗口呢，是因为他可以层层叠叠如这样：
![img](http://static.iocoder.cn/f33a02f400f6f3a9212c7fcd977b5136)
也可以进入“专注”模式，清清爽爽。

## 2.3 调试

在Eclipse中调试，你是没有一种叫“即时窗口”的东西的，他是什么东西呢：
![img](http://static.iocoder.cn/c0c6c8b5a4db9635d4761314880778f9)
你可以在调试期间打开“即时窗口”，在其中输入变量和表达式，他会直接给出你答案，类似Chrome调试时的Console。

## 2.4 Git 的使用

在IDEA中使用Git，感觉非常现代，一反Eclipse中Git管理的笨重和粗糙。只需要Ctrl+K就会出现Commit界面，Revert、Diff等等让你体验非常自然。

# 3. 怎么走出 Eclipse 的舒适区

简单看了一下IDEA的优点，不知道你是否有点心动呢？Eclipse如同一条旧毛毯，有感情但却不舒适。如果你像我一样有不适应的顾虑，下面我会详细说一下要转到IDEA需要做的一些工作。

## 3.1 Maven 项目导入和启动 Tomcat

首先要转变的一个观念是IDEA中没有工作空间的概念，而用了模块（Modules）来代替。
当你第一次打开IDEA，走完前置配置之后，可以“Project From Existing Sources”--从现有源码导入项目，如果是Maven项目，直接选中pom.xml文件即可。
重要的是，如果你想如Eclipse中一样把很多项目放在一个空间下，你可以这么做：

- Ctrl+Shift+Alt+S打开项目结构
- 找到Modules，点击上面的加号，选择Import Modules，再选择pom.xml文件即可
- 如果你想一个项目一个窗口，那么还是可以选择“Project From Existing Sources”

如果你的项目用的是Spring Boot，那么打开启动类，点击右侧小图标：
![img](http://static.iocoder.cn/7ef4167793ce6c4d0c2081834b27f687)
就可以直接启动这个类了。

------

如果你的项目需要用Tomcat来启动，那么找到这个地方：
![img](http://static.iocoder.cn/1c4d073896350200b0fd788a2ef0dde9)
点击Edit Configurations...，在Templates中找到Tomcat Server，配置Local。
配置完之后，点击左上角的加号，选择Tomcat Server--Local，配置端口号等等。重点来了，你需要在Deployment选择卡中点击加号，配置Article，Article选择war exploded类型的war，点击ok完成配置。
选中你的配置，点击Run（或Shift+F10）即可运行Tomcat应用。

## 3.2 快捷键映射

要换一个IDE开发，最需要适应的莫过于快捷键的使用习惯，下面我对来个IDE常用的快捷键做了一个映射，供大家参考：

| 操作               | Eclipse           | IDEA                  |
| ------------------ | ----------------- | --------------------- |
| 删除一行           | Ctrl+D            | Ctrl+Y                |
| 关闭当前窗口       | Ctrl+W            | Ctrl+F4               |
| 上移、下移一行     | Alt+↑、↓          | Ctrl+Alt+↑、↓         |
| 回退操作           | Ctrl+Z            | Ctrl+Z                |
| 反向回退           | Ctrl+Y            | Ctrl+Shift+Z          |
| 回到上一处编辑     | Alt+←             | Ctrl+Alt+←            |
| 提取变量           | Ctrl+1 And Ctrl+L | Ctrl+Alt+V            |
| 添加或取消注释     | Ctrl+/            | Ctrl+/                |
| 生成Getter、Setter | Alt+Shift+S       | Alt+Insert            |
| 光标移到相同的变量 | Ctrl+K            | F3或Ctrl+F7           |
| 打开类结构         | Ctrl+O            | Ctrl+F12              |
| 显示类继承层次     | Ctrl+T            | Ctrl+H                |
| 查看方法调用链     | Ctrl+Shift+H      | Ctrl+Alt+H            |
| 文件内容搜索       | Ctrl+H            | Ctrl+Shift+F          |
| 按文件名搜索       | Ctrl+Shift+R      | Ctrl+Shift+N          |
| 格式化代码         | Ctrl+Shift+F      | Ctrl+Alt+L            |
| 代码折叠与打开     | Ctrl+Shift+*、/   | Ctrl+Shift+加号、减号 |
| try-catch包围      | Alt+Shift+S       | Ctrl+Shift+T          |

## 3.3 快捷键被占用问题

IDEA中的默认快捷键有可能被其他程序占用，例如Windows上IDEA的智能提示是Ctrl+Space，这个快捷键会被系统输入法切换中英文占用，建议修改为Alt+引号。Ctrl+Alt+S打开Settring，如图：
![img](http://static.iocoder.cn/39a15a0aa8ed71fb42665832ceaa88f5)
其他有可能被占用的快捷键还有调试的单步跳过F8，try-catch包围的Ctrl+Shift+T，我分别改为了F10和Alt+T。

## 3.4 关闭部分“建议”

使用IDEA过程中，你会发现一些如Office Word似的拼写检查，如果你想关闭他，如图：
![img](http://static.iocoder.cn/90e3d9a0d73c59fb5364413d48338d02)