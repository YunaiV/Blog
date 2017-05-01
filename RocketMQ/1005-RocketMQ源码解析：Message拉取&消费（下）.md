# 1、概述

本文接：[《Message拉取&消费（上）》](https://github.com/YunaiV/Blog/blob/master/RocketMQ/1005-RocketMQ源码解析：Message拉取&消费（上）.md)。

主要解析 `Consumer` 在 **消费** 逻辑涉及到的源码。

# 2、Consumer

MQ 提供了两类消费者：

* PullConsumer：TODO
* PushConsumer：
    * 在大多数场景下使用。
    * 名字虽然是 `Push` 开头，实际在实现时，使用 `Pull` 方式实现。通过 `Pull` **不断不断不断**轮询 `Broker` 获取消息。当不存在新消息时，`Broker` 会**挂起请求**，直到有新消息产生，取消挂起，返回新消息。这样，基本和 `Broker` 主动 `Push` 做到**接近**的实时性（当然，还是有相应的实时性损失）。原理类似 **[长轮询( `Long-Polling` )](https://www.ibm.com/developerworks/cn/web/wa-lo-comet/)**。


**本文主要以主要讲解`PushConsumer`，部分讲解`PullConsumer`。**
**本文主要以主要讲解`PushConsumer`，部分讲解`PullConsumer`。**
**本文主要以主要讲解`PushConsumer`，部分讲解`PullConsumer`。**

# 3、PushConsumer 一览

先看一张 `PushConsumer` 包含的组件以及组件之间的交互图：

![PushConsumer手绘图.png](images/1005/PushConsumer手绘图.png)

* `RebalanceService`：负责分配消费队列，即分配当前 `Consumer` 可消费的队列。当有新的 `Consumer` 的加入或移除，都会进行消费队列重新负载均衡，分配消费队列。
* `PullMessageService`：拉取消息线程服务，**不断不断不断**从 `Broker` 拉取消息，并提交消费任务到 `ConsumeMessageService`。
* `ConsumeMessageService`：消费消息线程服务，**不断不断不断**消费消息，并处理消费结果。
* `RemoteBrokerOffsetStore`：`Consumer` 消费进度管理，负责从 `Broker` 获取消费进度，更新消费进度到 `Broker`。
* `ProcessQueue` ：消息处理队列。
* `MQClientInstance` ：封装对 `Namesrv`，`Broker` 的 API调用，提供给 `Producer`、`Consumer` 使用。

# 4、PushConsumer 消费队列分配

![RebalanceService&PushConsumer分配队列](images/1005/RebalanceService&PushConsumer分配队列.png)

## RebalanceService

```Java
  1: public class RebalanceService extends ServiceThread {
  2: 
  3:     /**
  4:      * 等待间隔，单位：毫秒
  5:      */
  6:     private static long waitInterval =
  7:         Long.parseLong(System.getProperty(
  8:             "rocketmq.client.rebalance.waitInterval", "20000"));
  9: 
 10:     private final Logger log = ClientLogger.getLog();
 11:     /**
 12:      * MQClient对象
 13:      */
 14:     private final MQClientInstance mqClientFactory;
 15: 
 16:     public RebalanceService(MQClientInstance mqClientFactory) {
 17:         this.mqClientFactory = mqClientFactory;
 18:     }
 19: 
 20:     @Override
 21:     public void run() {
 22:         log.info(this.getServiceName() + " service started");
 23: 
 24:         while (!this.isStopped()) {
 25:             this.waitForRunning(waitInterval);
 26:             this.mqClientFactory.doRebalance();
 27:         }
 28: 
 29:         log.info(this.getServiceName() + " service end");
 30:     }
 31: 
 32:     @Override
 33:     public String getServiceName() {
 34:         return RebalanceService.class.getSimpleName();
 35:     }
 36: }
```

* 说明 ：分配消费队列线程服务。
* 第 26 行 ：调用 `MQClientInstance#doRebalance(...)` 分配消费队列。目前有三种情况情况下触发：
    * 如 `第 25 行` 等待超时，每 20s 调用一次。
    * `PushConsumer` 启动时，调用 `rebalanceService#wakeup(...)` 触发。
    * `Broker` 通知 `Consumer` 加入 或 移除时，`Consumer` 响应通知，调用 `rebalanceService#wakeup(...)` 触发。

 详细解析见：[MQClientInstance#doRebalance(...)](mqclientinstancedorebalance)。

## MQClientInstance#doRebalance(...)

```Java
  1: public void doRebalance() {
  2:     for (Map.Entry<String, MQConsumerInner> entry : this.consumerTable.entrySet()) {
  3:         MQConsumerInner impl = entry.getValue();
  4:         if (impl != null) {
  5:             try {
  6:                 impl.doRebalance();
  7:             } catch (Throwable e) {
  8:                 log.error("doRebalance exception", e);
  9:             }
 10:         }
 11:     }
 12: }
```

* 说明 ：遍历当前 `Client` 包含的 `consumerTable`( `Consumer`集合 )，执行消费队列分配。
* **疑问**：目前代码调试下来，`consumerTable` 只包含 `Consumer` 自己。😈有大大对这个疑问有解答的，烦请解答下。
* 第 6 行 ：调用 `MQConsumerInner#doRebalance(...)` 进行队列分配。`DefaultMQPushConsumerImpl` 和 `DefaultMQPullConsumerImpl` 分别对该接口方法进行了实现。`DefaultMQPushConsumerImpl#doRebalance(...)` 详细解析见：[DefaultMQPushConsumerImpl#doRebalance(...)](defaultmqpushconsumerimpldorebalance)。

## DefaultMQPushConsumerImpl#doRebalance(...)

# 7、Consumer 调用[拉取消息]接口
# 8、Consumer 消费消息
# 9、Consumer 调用[发回消息]接口
# 10、Consumer 调用[更新消费进度]接口


