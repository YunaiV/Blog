title: 树莓派上利用 Tensorflow 实现小车的自动驾驶
date: 2019-01-01-04
tags:
categories: 精进
permalink: Fight/The-raspberry-pie-USES-Tensorflow-to-realize-the-automatic-driving-of-the-car
author: Timthony
from_url: https://my.oschina.net/u/3858986/blog/2875027
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247486006&idx=3&sn=b5608d48097ca76c8f76db20fe5c2bd6&chksm=fa497587cd3efc91fc72a025628b708621e5bd0cea86694aaf0e0c6d266ad688e9e027729b6d&token=933039983&lang=zh_CN#rd

-------

摘要: 原创出处 https://my.oschina.net/u/3858986/blog/2875027 「Timthony」欢迎转载，保留摘要，谢谢！

- [整体流程](http://www.iocoder.cn/Fight/The-raspberry-pie-USES-Tensorflow-to-realize-the-automatic-driving-of-the-car/)
- [注意事项：](http://www.iocoder.cn/Fight/The-raspberry-pie-USES-Tensorflow-to-realize-the-automatic-driving-of-the-car/)
- [具体制作流程： ](http://www.iocoder.cn/Fight/The-raspberry-pie-USES-Tensorflow-to-realize-the-automatic-driving-of-the-car/)
- [正在进行一些改进：](http://www.iocoder.cn/Fight/The-raspberry-pie-USES-Tensorflow-to-realize-the-automatic-driving-of-the-car/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

先抛出大家最关心的——代码地址：

github传送门：<https://github.com/Timthony/self_drive>

码云传送门：<https://gitee.com/tiantianhang/self_drive>

基于树莓派的人工智能自动驾驶小车

# 整体流程

电机控制
摄像头调试
道路数据采集
搭建深度学习模型，参数调试
自动驾驶真实道路模拟
参数最终调试

使用方法：    

1. 先将树莓派小车硬件组装好
2. 使用zth_car_control.py来控制小车的前后左右移动，配合zth_collect_data.py来人工操作，使小车在自己制作的跑道进行数据采集。（该过程在树莓派进行）
3. 数据采集完成以后使用zth_process_img.py来对采集的数据进行处理，之前当前先完成一些数据清洗的工作。（电脑上执行）
4. 使用神经网络模型对数据进行训练zth_train.py，得到训练好的模型。（电脑上执行）
5. 在树莓派小车上使用zth_drive和训练好的模型，载入模型，即可实现在原先跑道的自动驾驶。（树莓派上执行）  
   注意：只需要使用上述提到的代码即可，别的都是一些初始版本或者正在增加的一些新模块。    



![img](http://static.iocoder.cn/536a5d271182a1d0388280c623935179)

# 注意事项：

1. 赛道需要自己制作，很重要，决定了数据质量。(我是在地板上，贴的有色胶带，然后贴成了跑道的形状)。
2. 赛道的宽度大约是车身的两倍。
3. 大约采集了五六万张图像，然后筛选出三四万张。
4. 摄像头角度问题

# 具体制作流程： 

1. 小车原始模型，某宝购买玩具车即可，比如：有电机，有自带电池盒（给电机供电）
2. 树莓派，摄像头，蓄电电池组（用于树莓派供电）
3. 使用一些螺栓，螺柱，亚克力板将树莓派，蓄电电池固定在小车上（具体方法，看手头的工具吧）
4. 组装好以后，树莓派通过VNC连接电脑，登陆树莓派，在树莓派安装keras环境，以便最后调用训练好的模型。
5. 关于小车的控制（电机控制，摄像头采集数据），都在源文件，有注释，大致思路就是通过方向键AWSD来控制方向，使用了pygame的工具包。
6. 通过电脑端的wasd方向键手动控制小车（已经VNC连接好）在制作好的赛道上进行图像采集，直线部分按w，左拐弯按a，右拐弯按d等，建议采集50000张以上。
   （采集的图像命名要求为，0_xxxx,1_xxxx,其中首位字母就代表了你按下的是哪个键，比如图像是0开头，那么这张图像就是直行，按下的是w键，这些0，1，2，3，4 数字就相当于数据的标签值）
7. 将图片从树莓派拷贝下来，进行数据清洗，使用电脑端的深度学习环境进行模型训练，使用的模型可以自行定义。
8. 将训练好的模型文件.h5拷贝到树莓派，然后通过树莓派调用载入模型，即可处理实时的图像，并且根据图像预测出是0，1，2，3，4等数字，也就表示了树莓派该怎么移动,通过树莓派控制电机即可。

# 正在进行一些改进：

1.使用迁移学习进行fine-tuning是否可以提高精度
2.处理光照问题
3.处理数据类别不平衡的问题
欢迎交流讨论

