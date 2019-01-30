title: 亚马逊抢甲骨文的 Java 饭碗，推出 Corretto
date: 2019-01-27
tags:
categories: 精进
permalink: Fight/Amazon-took-oracle-Java-jobs-and-launched-Corretto
author: 开源中国
from_url: http://www.10tiao.com/html/194/201811/2651482123/1.html
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247486021&idx=1&sn=ad9b7647f0fedc07622067c9317ebf16&chksm=fa4975f4cd3efce20de0ed04a5c5268d3bb36df9693e44bea2b2873931fb3bf6ccc3f64ed007&token=810316232&lang=zh_CN#rd

-------

摘要: 原创出处 http://www.10tiao.com/html/194/201811/2651482123/1.html 「开源中国」欢迎转载，保留摘要，谢谢！


-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------


![img](http://static.iocoder.cn/675b48e38f6f0535c0f15697664e9347)

14 日亚马逊发文宣布 Amazon Corretto 的预览版，这是一个免费的、跨平台生产就绪的 OpenJDK 发行版。

这是亚马逊继前不久重申对 Amazon Linux 中的 Java 进行长期支持后，其对 Java 用户提供支持的又一重大支持。

源码地址：https://github.com/corretto

亚马逊介绍，目前其在数千种生产服务上运行着 Amazon  Corretto，Corretto 的补丁和改进使亚马逊能够解决大规模、真实的服务问题，满足严苛的性能和可扩展性需求。

Corretto 支持多种平台，可以在云端与本地计算机上运行。目前，Amazon Linux 2、Windows、macOS 平台和 Docker 镜像都提供了与 OpenJDK 8 对应的 Corretto 8 预览版。每个 Corretto 版本上都会运行技术兼容性工具包（Technology Compatibility Kit，TCK），以确保与 Java SE 平台相兼容。在不使用到 OpenJDK 中没有的功能（例如 Java Flight Recorder）的情况下，Corretto 完全可以直接作为 Java SE 发行版的替代品。Amazon 计划在 2019 年将 Corretto 作为Amazon Linux 2 上的默认 OpenJDK。



Corretto 计划于 2019 年第一季度 GA，届时还将支持 Ubuntu 和 Red Hat Enterprise Linux 平台，在这些平台上对应于 Open JDK 11 的 Corretto 11 版本将在 2019 年 4 月之前有足够的时间进行测试。同时亚马逊将免费提供 Corretto 8 安全更新到 2023 年 6 月、Corretto 11 安全更新到 2024 年 8 月。



Java 之父 James Gosling（去年加入 AWS）表示 Amazon 与 Java 之间有着长久而深远的历史，他很高兴看到 Amazon 内部任务关键型 Java团队的工作正在服务世界上的其它地方。



> “Amazon has a long and deep history with Java. I’m thrilled to see the work of our internal mission-critical Java team being made available to the rest of the world” — James Gosling
