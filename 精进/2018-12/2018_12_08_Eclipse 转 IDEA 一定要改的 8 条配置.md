title: Eclipse 转 IDEA 一定要改的 8 条配置
date: 2018-12-08
tags: IntelliJ IDEA
categories: 精进
permalink: Fight/Eclipse-to-IDEA-must-change-the-8-configuration
author: 孤独烟
from_url: https://www.cnblogs.com/rjzheng/p/9966801.html
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247485810&idx=2&sn=f6c6e0e232f66e231883c22cce0b5ba8&chksm=fa4976c3cd3effd5ea446b37282d94851671adfb0fb2d2df45763524b86ee387ee8503f16bc7&token=55862109&lang=zh_CN#rd

-------

摘要: 原创出处 https://www.cnblogs.com/rjzheng/p/9966801.html 「孤独烟」欢迎转载，保留摘要，谢谢！

- [引言](http://www.iocoder.cn/Fight/Eclipse-to-IDEA-must-change-the-8-configuration/)
- [正文](http://www.iocoder.cn/Fight/Eclipse-to-IDEA-must-change-the-8-configuration/)
  - [自动编译开关](http://www.iocoder.cn/Fight/Eclipse-to-IDEA-must-change-the-8-configuration/)
  - [忽略大小写开关](http://www.iocoder.cn/Fight/Eclipse-to-IDEA-must-change-the-8-configuration/)
  - [智能导包开关](http://www.iocoder.cn/Fight/Eclipse-to-IDEA-must-change-the-8-configuration/)
  - [悬浮提示开关](http://www.iocoder.cn/Fight/Eclipse-to-IDEA-must-change-the-8-configuration/)
  - [取消单行显示tabs的操作](http://www.iocoder.cn/Fight/Eclipse-to-IDEA-must-change-the-8-configuration/)
  - [项目文件编码](http://www.iocoder.cn/Fight/Eclipse-to-IDEA-must-change-the-8-configuration/)
  - [滚轴修改字体大小](http://www.iocoder.cn/Fight/Eclipse-to-IDEA-must-change-the-8-configuration/)
  - [设置行号显示](http://www.iocoder.cn/Fight/Eclipse-to-IDEA-must-change-the-8-configuration/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# 引言

坦白说，我很少写这种操作类型的文章。因为这种文章没啥新意，大家操作步骤肯定是一样的。然而，我答应了我的同事小阳，给她出一篇！毕竟人家打算从Eclipse转IDEA了，于是以示鼓励，写一篇给她！
那么是哪八条一定要改的配置呢！

- 自动编译开关
- 忽略大小写开关
- 智能导包开关
- 悬浮提示开关
- 取消单行显示tabs的操作
- 项目文件编码
- 滚轴修改字体大小
- 设置行号显示

# 正文

## 自动编译开关

在Eclipse中自动编译开关是开着的，如下所示
![image](http://static.iocoder.cn/c39aaeb708abcd6cc8a3c1db580d5d58)
那么，在IDEA中，务必要手动将其打开，非常重要！如下所示
![image](http://static.iocoder.cn/f5d6d00c18596181ad8009f2de46a17b)

## 忽略大小写开关

IDEA默认是匹配大小写，此开关如果未关。你输入字符一定要符合大小写。比如你敲string是不会出现代码提示或智能补充。
但是，如果你开了这个开关，你无论输入String或者string都会出现代码提示或者智能补充！
![image](http://static.iocoder.cn/507bdad7be2a3705c9f905a0a264ccab)

## 智能导包开关

如下图所示，将

- 自动导入不明确的结构
- 智能优化包

这两个选项勾上。那么有什么效果呢？
你在代码中，只要敲list，就会出现提示，自动导入`java.util.List`这个类。而这个特性，在eclipse中，是无法做到的。如下图所示
![image](http://static.iocoder.cn/5d418f58b5ae9dd0f12058c7387086fa)

## 悬浮提示开关

这个容易理解，打开这个开关后。只要把鼠标放在相应的类上，就会出现提示，如下图所示
![image](http://static.iocoder.cn/648e5485496f6d985b58ff05f27f2c1c)

## 取消单行显示tabs的操作

如下图所示，把该按钮去了
![image](http://static.iocoder.cn/17218b7efc4d444393a1565464da0fc3)
那么去掉后有什么效果呢？
打开多个文件的时候，会换行显示，非常直观。大大提升效率！
![image](http://static.iocoder.cn/2655c6a0a74d3d7cb9eef5acc236c5f9)

## 项目文件编码

如下图所示进行设置
![image](http://static.iocoder.cn/bcd12647ee016a071eef66b25ff9dd87)
Transparent native-to-ascii conversion的意思是：自动转换ASCII编码。
他的工作原理是：在文件中输入文字时他会自动的转换为Unicode编码，然后在idea中发开文件时他会自动转回文字来显示。
这样做是为了防止文件乱码。
这样你的properties文件，一般都不会出现中文乱码！

## 滚轴修改字体大小

是这样的，我一般在写代码的时候。都是设按住Ctrl+滚轴可以修改编辑器字体大小，这样其实很方便，大家不妨试试。
如下图所示
![image](http://static.iocoder.cn/01bd45eaf6bb7730c0546e288d128721)

## 设置行号显示

这个的重要性就不用多说了，勾上后代码中，会显示行数!
![image](http://static.iocoder.cn/c2050d6ffa70778bf94b53d6085b45ca)
