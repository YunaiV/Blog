## 为什么阅读RocketMQ源码？

@time：2017-03-23

* 深入了解 MQ ，知其然知其所以然，如何实现高性能、高可用
* 最终一致行，是如何通过 MQ 进行实现
* 了解 Netty 在分布式中间件如何实现网络通信以及各种异常场景的处理
* 了解 MQ 消息存储，特别是磁盘 IO 部分
* **最重要的**，希望通过阅读源码，在技术上的认知和能力上，有新的突破

## 步骤

- [ ] namesrv、broker 启动
- [ ] producer 连接初始化
- [ ] consumer 连接初始化
- [ ] 消息模型
- [ ] producer 发消息
- [ ] broker 收消息
- [ ] broker 发消息
- [ ] consumer 收消息（多消费者）
- [ ] consuer 消息确认
- [ ] broker 队列模型
- [ ] broker store 消息存储
- [ ] broker store 消息同步
- [ ] 顺序消息
- [ ] 事务消息
- [ ] 定时消息
- [ ] pub/sub模型
- [ ] 主从
- [ ] filtersrv 过滤消息
