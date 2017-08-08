title: RocketMQ 源码分析 —— Message 顺序发送与消费
date: 2017-05-13
tags:
categories: RocketMQ
permalink: RocketMQ/message-send-and-consume-orderly

-------

![](http://www.yunai.me/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋艿的后端小屋】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

- [1. 概述](#)
- [2. Producer 顺序发送](#)
- [3. Consumer 顺序消费](#)
	- [3.1 获得(锁定)消息队列](#)
	- [3.2 移除消息队列](#)
	- [3.3 消费消息队列](#)
		- [3.1.1 消费消息](#)
		- [3.1.2 处理消费结果](#)
		- [3.13 消息处理队列核心方法](#)

# 1. 概述

**建议**前置阅读内容：

* [《RocketMQ 源码分析 —— Message 发送与接收》](http://www.yunai.me/RocketMQ/message-send-and-receive/)
* [《RocketMQ 源码分析 —— Message 拉取与消费（下）》](http://www.yunai.me/RocketMQ/message-pull-and-consume-second/)

当然对 `Message` 发送与消费已经有一定了解的同学，可以选择跳过。

-------

`RocketMQ` 提供了两种顺序级别：

* 普通顺序消息 ：`Producer` 将相关联的消息发送到相同的消息队列。
* 完全严格顺序 ：在 `普通顺序消息` 的基础上，`Consumer` 严格顺序消费。

绝大部分场景下只需要用到**普通顺序消息**。  
例如说：给用户发送短信消息 + 发送推送消息，将两条消息发送到不同的消息队列，若其中一条消息队列消费较慢造成堵塞，用户可能会收到两条消息会存在一定的时间差，带来的体验会相对较差。当然类似这种场景，即使有一定的时间差，**不会产生系统逻辑上BUG**。另外，`普通顺序消息`性能能更加好。  
那么什么时候使用使用**完全严格顺序**？如下是来自官方文档的说明：
> 目前已知的应用只有数据库 `binlog` 同步强依赖严格顺序消息，其他应用绝大部分都可以容忍短暂乱序，推荐使用普通的顺序消息

-------

😈上代码！！！

# 2. `Producer` 顺序发送

官方发送顺序消息的**例子**：

```Java
  1: package org.apache.rocketmq.example.ordermessage;
  2: 
  3: import java.io.UnsupportedEncodingException;
  4: import java.util.List;
  5: import org.apache.rocketmq.client.exception.MQBrokerException;
  6: import org.apache.rocketmq.client.exception.MQClientException;
  7: import org.apache.rocketmq.client.producer.DefaultMQProducer;
  8: import org.apache.rocketmq.client.producer.MQProducer;
  9: import org.apache.rocketmq.client.producer.MessageQueueSelector;
 10: import org.apache.rocketmq.client.producer.SendResult;
 11: import org.apache.rocketmq.common.message.Message;
 12: import org.apache.rocketmq.common.message.MessageQueue;
 13: import org.apache.rocketmq.remoting.common.RemotingHelper;
 14: import org.apache.rocketmq.remoting.exception.RemotingException;
 15: 
 16: public class Producer {
 17:     public static void main(String[] args) throws UnsupportedEncodingException {
 18:         try {
 19:             MQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
 20:             producer.start();
 21: 
 22:             String[] tags = new String[] {"TagA", "TagB", "TagC", "TagD", "TagE"};
 23:             for (int i = 0; i < 100; i++) {
 24:                 int orderId = i % 10;
 25:                 Message msg =
 26:                     new Message("TopicTestjjj", tags[i % tags.length], "KEY" + i,
 27:                         ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
 28:                 SendResult sendResult = producer.send(msg, new MessageQueueSelector() {
 29:                     @Override
 30:                     public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
 31:                         Integer id = (Integer) arg;
 32:                         int index = id % mqs.size();
 33:                         return mqs.get(index);
 34:                     }
 35:                 }, orderId);
 36: 
 37:                 System.out.printf("%s%n", sendResult);
 38:             }
 39: 
 40:             producer.shutdown();
 41:         } catch (MQClientException | RemotingException | MQBrokerException | InterruptedException e) {
 42:             e.printStackTrace();
 43:         }
 44:     }
 45: }
```

* 第 28 至 35 行 ：实现了根据 `id % mqs.size()` 来进行消息队列的选择。当前例子，**我们传递 `orderId` 作为参数，那么相同的 `orderId` 能够进入相同的消息队列**。

-------

`MessageQueueSelector` 接口的**源码**：

```Java
  1: public interface MessageQueueSelector {
  2: 
  3:     /**
  4:      * 选择消息队列
  5:      *
  6:      * @param mqs 消息队列
  7:      * @param msg 消息
  8:      * @param arg 参数
  9:      * @return 消息队列
 10:      */
 11:     MessageQueue select(final List<MessageQueue> mqs, final Message msg, final Object arg);
 12: }
```
-------

`Producer` 选择队列发送消息方法的**源码**：

```Java
 16: private SendResult sendSelectImpl(//
 17:     Message msg, //
 18:     MessageQueueSelector selector, //
 19:     Object arg, //
 20:     final CommunicationMode communicationMode, //
 21:     final SendCallback sendCallback, final long timeout//
 22: ) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
 23:     this.makeSureStateOK();
 24:     Validators.checkMessage(msg, this.defaultMQProducer);
 25: 
 26:     TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
 27:     if (topicPublishInfo != null && topicPublishInfo.ok()) {
 28:         MessageQueue mq = null;
 29:         try {
 30:             mq = selector.select(topicPublishInfo.getMessageQueueList(), msg, arg);
 31:         } catch (Throwable e) {
 32:             throw new MQClientException("select message queue throwed exception.", e);
 33:         }
 34: 
 35:         if (mq != null) {
 36:             return this.sendKernelImpl(msg, mq, communicationMode, sendCallback, null, timeout);
 37:         } else {
 38:             throw new MQClientException("select message queue return null.", null);
 39:         }
 40:     }
 41: 
 42:     throw new MQClientException("No route info for this topic, " + msg.getTopic(), null);
 43: }
```

* 第 30 行 ：选择消息队列。
* 第 36 行 ：发送消息。

# 3. `Consumer` 严格顺序消费

`Consumer` 在严格顺序消费时，通过 **三** 把锁保证严格顺序消费。

* `Broker` 消息队列锁（**分布式锁**） ：
    * 集群模式下，`Consumer` 从 `Broker` 获得该锁后，才能进行消息拉取、消费。
    * 广播模式下，`Consumer` 无需该锁。
* `Consumer` 消息队列锁（**本地锁**） ：`Consumer` 获得该锁才能操作消息队列。
* `Consumer` 消息处理队列消费锁（**本地锁**） ：`Consumer` 获得该锁才能消费消息队列。

**可能同学有疑问，为什么有 `Consumer` 消息队列锁还需要有 `Consumer` 消息队列消费锁呢**？😈让我们带着疑问继续往下看。

-------

## 3.1 获得(锁定)消息队列

**集群模式**下，`Consumer` 更新属于自己的消息队列时，会向 `Broker` 锁定该消息队列（*广播模式下不需要*）。如果锁定失败，则更新失败，即该消息队列不属于自己，不能进行消费。核心代码如下：

```Java
  1: // ⬇️⬇️⬇️【RebalanceImpl.java】
  2: private boolean updateProcessQueueTableInRebalance(final String topic, final Set<MessageQueue> mqSet, final boolean isOrder) {
  3: // ..... 此处省略部分代码 
  4:     // 增加 不在processQueueTable && 存在于mqSet 里的消息队列。
  5:     List<PullRequest> pullRequestList = new ArrayList<>(); // 拉消息请求数组
  6:     for (MessageQueue mq : mqSet) {
  7:         if (!this.processQueueTable.containsKey(mq)) {
  8:             if (isOrder && !this.lock(mq)) { // 顺序消息锁定消息队列
  9:                 log.warn("doRebalance, {}, add a new mq failed, {}, because lock failed", consumerGroup, mq);
 10:                 continue;
 11:             }
 12: 
 13:             this.removeDirtyOffset(mq);
 14:             ProcessQueue pq = new ProcessQueue();
 15:             long nextOffset = this.computePullFromWhere(mq);
 16:             if (nextOffset >= 0) {
 17:                 ProcessQueue pre = this.processQueueTable.putIfAbsent(mq, pq);
 18:                 if (pre != null) {
 19:                     log.info("doRebalance, {}, mq already exists, {}", consumerGroup, mq);
 20:                 } else {
 21:                     log.info("doRebalance, {}, add a new mq, {}", consumerGroup, mq);
 22:                     PullRequest pullRequest = new PullRequest();
 23:                     pullRequest.setConsumerGroup(consumerGroup);
 24:                     pullRequest.setNextOffset(nextOffset);
 25:                     pullRequest.setMessageQueue(mq);
 26:                     pullRequest.setProcessQueue(pq);
 27:                     pullRequestList.add(pullRequest);
 28:                     changed = true;
 29:                 }
 30:             } else {
 31:                 log.warn("doRebalance, {}, add new mq failed, {}", consumerGroup, mq);
 32:             }
 33:         }
 34:     }
 35: 
 36: // ..... 此处省略部分代码 
 37: }
 38: 
 39: // ⬇️⬇️⬇️【RebalanceImpl.java】
 40: /**
 41:  * 请求Broker获得指定消息队列的分布式锁
 42:  *
 43:  * @param mq 队列
 44:  * @return 是否成功
 45:  */
 46: public boolean lock(final MessageQueue mq) {
 47:     FindBrokerResult findBrokerResult = this.mQClientFactory.findBrokerAddressInSubscribe(mq.getBrokerName(), MixAll.MASTER_ID, true);
 48:     if (findBrokerResult != null) {
 49:         LockBatchRequestBody requestBody = new LockBatchRequestBody();
 50:         requestBody.setConsumerGroup(this.consumerGroup);
 51:         requestBody.setClientId(this.mQClientFactory.getClientId());
 52:         requestBody.getMqSet().add(mq);
 53: 
 54:         try {
 55:             // 请求Broker获得指定消息队列的分布式锁
 56:             Set<MessageQueue> lockedMq =
 57:                 this.mQClientFactory.getMQClientAPIImpl().lockBatchMQ(findBrokerResult.getBrokerAddr(), requestBody, 1000);
 58: 
 59:             // 设置消息处理队列锁定成功。锁定消息队列成功，可能本地没有消息处理队列，设置锁定成功会在lockAll()方法。
 60:             for (MessageQueue mmqq : lockedMq) {
 61:                 ProcessQueue processQueue = this.processQueueTable.get(mmqq);
 62:                 if (processQueue != null) {
 63:                     processQueue.setLocked(true);
 64:                     processQueue.setLastLockTimestamp(System.currentTimeMillis());
 65:                 }
 66:             }
 67: 
 68:             boolean lockOK = lockedMq.contains(mq);
 69:             log.info("the message queue lock {}, {} {}",
 70:                 lockOK ? "OK" : "Failed",
 71:                 this.consumerGroup,
 72:                 mq);
 73:             return lockOK;
 74:         } catch (Exception e) {
 75:             log.error("lockBatchMQ exception, " + mq, e);
 76:         }
 77:     }
 78: 
 79:     return false;
 80: }
```

* ⬆️⬆️⬆️
* 第 8 至 11 行 ：顺序消费时，锁定消息队列。如果锁定失败，新增消息处理队列失败。

-------

`Broker` 消息队列锁会过期，默认配置 30s。因此，`Consumer` 需要不断向 `Broker` 刷新该锁过期时间，默认配置 20s 刷新一次。核心代码如下：

```Java
  1: // ⬇️⬇️⬇️【ConsumeMessageOrderlyService.java】
  2: public void start() {
  3:     if (MessageModel.CLUSTERING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())) {
  4:         this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
  5:             @Override
  6:             public void run() {
  7:                 ConsumeMessageOrderlyService.this.lockMQPeriodically();
  8:             }
  9:         }, 1000 * 1, ProcessQueue.REBALANCE_LOCK_INTERVAL, TimeUnit.MILLISECONDS);
 10:     }
 11: }
```

## 3.2 移除消息队列

集群模式下，`Consumer` 移除自己的消息队列时，会向 `Broker` 解锁该消息队列（广播模式下不需要）。核心代码如下：

```Java
  1: // ⬇️⬇️⬇️【RebalancePushImpl.java】
  2: /**
  3:  * 移除不需要的队列相关的信息
  4:  * 1. 持久化消费进度，并移除之
  5:  * 2. 顺序消费&集群模式，解锁对该队列的锁定
  6:  *
  7:  * @param mq 消息队列
  8:  * @param pq 消息处理队列
  9:  * @return 是否移除成功
 10:  */
 11: @Override
 12: public boolean removeUnnecessaryMessageQueue(MessageQueue mq, ProcessQueue pq) {
 13:     // 同步队列的消费进度，并移除之。
 14:     this.defaultMQPushConsumerImpl.getOffsetStore().persist(mq);
 15:     this.defaultMQPushConsumerImpl.getOffsetStore().removeOffset(mq);
 16:     // 集群模式下，顺序消费移除时，解锁对队列的锁定
 17:     if (this.defaultMQPushConsumerImpl.isConsumeOrderly()
 18:         && MessageModel.CLUSTERING.equals(this.defaultMQPushConsumerImpl.messageModel())) {
 19:         try {
 20:             if (pq.getLockConsume().tryLock(1000, TimeUnit.MILLISECONDS)) {
 21:                 try {
 22:                     return this.unlockDelay(mq, pq);
 23:                 } finally {
 24:                     pq.getLockConsume().unlock();
 25:                 }
 26:             } else {
 27:                 log.warn("[WRONG]mq is consuming, so can not unlock it, {}. maybe hanged for a while, {}", //
 28:                     mq, //
 29:                     pq.getTryUnlockTimes());
 30: 
 31:                 pq.incTryUnlockTimes();
 32:             }
 33:         } catch (Exception e) {
 34:             log.error("removeUnnecessaryMessageQueue Exception", e);
 35:         }
 36: 
 37:         return false;
 38:     }
 39:     return true;
 40: }
 41: 
 42: // ⬇️⬇️⬇️【RebalancePushImpl.java】
 43: /**
 44:  * 延迟解锁 Broker 消息队列锁
 45:  * 当消息处理队列不存在消息，则直接解锁
 46:  *
 47:  * @param mq 消息队列
 48:  * @param pq 消息处理队列
 49:  * @return 是否解锁成功
 50:  */
 51: private boolean unlockDelay(final MessageQueue mq, final ProcessQueue pq) {
 52:     if (pq.hasTempMessage()) { // TODO 疑问：为什么要延迟移除
 53:         log.info("[{}]unlockDelay, begin {} ", mq.hashCode(), mq);
 54:         this.defaultMQPushConsumerImpl.getmQClientFactory().getScheduledExecutorService().schedule(new Runnable() {
 55:             @Override
 56:             public void run() {
 57:                 log.info("[{}]unlockDelay, execute at once {}", mq.hashCode(), mq);
 58:                 RebalancePushImpl.this.unlock(mq, true);
 59:             }
 60:         }, UNLOCK_DELAY_TIME_MILLS, TimeUnit.MILLISECONDS);
 61:     } else {
 62:         this.unlock(mq, true);
 63:     }
 64:     return true;
 65: }
```

* ⬆️⬆️⬆️
* 第 20 至 32 行 ：获取**消息队列消费锁**，避免和消息队列消费冲突。如果获取锁失败，则移除消息队列失败，等待下次重新分配消费队列时，再进行移除。如果未获得锁而进行移除，则可能出现另外的  `Consumer` 和当前 `Consumer` 同时消费该消息队列，导致消息无法严格顺序消费。
* 第 51 至 64 行 ：解锁 `Broker` 消息队列锁。如果消息处理队列存在剩余消息，则延迟解锁 `Broker` 消息队列锁。❓为什么消息处理队列存在剩余消息不能直接解锁呢？😈我也不知道，百思不得其解。如果有知道的同学麻烦教育下俺。

## 3.3 消费消息队列

😏本节会类比**并发消费消费队列**，建议对照 [PushConsumer并发消费消息](http://www.yunai.me/RocketMQ/message-pull-and-consume-second/#6、PushConsumer-消费消息) 一起理解。

### 3.1.1 消费消息

![顺序消费活动图-消费消息](http://www.yunai.me/images/RocketMQ/2017_05_13/01.png)

```Java
  1: // ⬇️⬇️⬇️【ConsumeMessageOrderlyService.java】
  2: class ConsumeRequest implements Runnable {
  3: 
  4:     /**
  5:      * 消息处理队列
  6:      */
  7:     private final ProcessQueue processQueue;
  8:     /**
  9:      * 消息队列
 10:      */
 11:     private final MessageQueue messageQueue;
 12: 
 13:     @Override
 14:     public void run() {
 15:         if (this.processQueue.isDropped()) {
 16:             log.warn("run, the message queue not be able to consume, because it's dropped. {}", this.messageQueue);
 17:             return;
 18:         }
 19: 
 20:         // 获得 Consumer 消息队列锁
 21:         final Object objLock = messageQueueLock.fetchLockObject(this.messageQueue);
 22:         synchronized (objLock) {
 23:             // (广播模式) 或者 (集群模式 && Broker消息队列锁有效)
 24:             if (MessageModel.BROADCASTING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())
 25:                 || (this.processQueue.isLocked() && !this.processQueue.isLockExpired())) {
 26:                 final long beginTime = System.currentTimeMillis();
 27:                 // 循环
 28:                 for (boolean continueConsume = true; continueConsume; ) {
 29:                     if (this.processQueue.isDropped()) {
 30:                         log.warn("the message queue not be able to consume, because it's dropped. {}", this.messageQueue);
 31:                         break;
 32:                     }
 33: 
 34:                     // 消息队列分布式锁未锁定，提交延迟获得锁并消费请求
 35:                     if (MessageModel.CLUSTERING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())
 36:                         && !this.processQueue.isLocked()) {
 37:                         log.warn("the message queue not locked, so consume later, {}", this.messageQueue);
 38:                         ConsumeMessageOrderlyService.this.tryLockLaterAndReconsume(this.messageQueue, this.processQueue, 10);
 39:                         break;
 40:                     }
 41:                     // 消息队列分布式锁已经过期，提交延迟获得锁并消费请求
 42:                     if (MessageModel.CLUSTERING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())
 43:                         && this.processQueue.isLockExpired()) {
 44:                         log.warn("the message queue lock expired, so consume later, {}", this.messageQueue);
 45:                         ConsumeMessageOrderlyService.this.tryLockLaterAndReconsume(this.messageQueue, this.processQueue, 10);
 46:                         break;
 47:                     }
 48: 
 49:                     // 当前周期消费时间超过连续时长，默认：60s，提交延迟消费请求。默认情况下，每消费1分钟休息10ms。
 50:                     long interval = System.currentTimeMillis() - beginTime;
 51:                     if (interval > MAX_TIME_CONSUME_CONTINUOUSLY) {
 52:                         ConsumeMessageOrderlyService.this.submitConsumeRequestLater(processQueue, messageQueue, 10);
 53:                         break;
 54:                     }
 55: 
 56:                     // 获取消费消息。此处和并发消息请求不同，并发消息请求已经带了消费哪些消息。
 57:                     final int consumeBatchSize = ConsumeMessageOrderlyService.this.defaultMQPushConsumer.getConsumeMessageBatchMaxSize();
 58:                     List<MessageExt> msgs = this.processQueue.takeMessags(consumeBatchSize);
 59:                     if (!msgs.isEmpty()) {
 60:                         final ConsumeOrderlyContext context = new ConsumeOrderlyContext(this.messageQueue);
 61: 
 62:                         ConsumeOrderlyStatus status = null;
 63: 
 64:                         // ....省略代码：Hook：before
 65: 
 66:                         // 执行消费
 67:                         long beginTimestamp = System.currentTimeMillis();
 68:                         ConsumeReturnType returnType = ConsumeReturnType.SUCCESS;
 69:                         boolean hasException = false;
 70:                         try {
 71:                             this.processQueue.getLockConsume().lock(); // 锁定队列消费锁
 72: 
 73:                             if (this.processQueue.isDropped()) {
 74:                                 log.warn("consumeMessage, the message queue not be able to consume, because it's dropped. {}",
 75:                                     this.messageQueue);
 76:                                 break;
 77:                             }
 78: 
 79:                             status = messageListener.consumeMessage(Collections.unmodifiableList(msgs), context);
 80:                         } catch (Throwable e) {
 81:                             log.warn("consumeMessage exception: {} Group: {} Msgs: {} MQ: {}", //
 82:                                 RemotingHelper.exceptionSimpleDesc(e), //
 83:                                 ConsumeMessageOrderlyService.this.consumerGroup, //
 84:                                 msgs, //
 85:                                 messageQueue);
 86:                             hasException = true;
 87:                         } finally {
 88:                             this.processQueue.getLockConsume().unlock(); // 锁定队列消费锁
 89:                         }
 90: 
 91:                         // ....省略代码：解析消费结果状态
 92: 
 93:                         // ....省略代码：Hook：after
 94: 
 95:                         ConsumeMessageOrderlyService.this.getConsumerStatsManager()
 96:                             .incConsumeRT(ConsumeMessageOrderlyService.this.consumerGroup, messageQueue.getTopic(), consumeRT);
 97: 
 98:                         // 处理消费结果
 99:                         continueConsume = ConsumeMessageOrderlyService.this.processConsumeResult(msgs, status, context, this);
100:                     } else {
101:                         continueConsume = false;
102:                     }
103:                 }
104:             } else {
105:                 if (this.processQueue.isDropped()) {
106:                     log.warn("the message queue not be able to consume, because it's dropped. {}", this.messageQueue);
107:                     return;
108:                 }
109: 
110:                 ConsumeMessageOrderlyService.this.tryLockLaterAndReconsume(this.messageQueue, this.processQueue, 100);
111:             }
112:         }
113:     }
114: 
115: }
```

* ⬆️⬆️⬆️
* 第 20 行 ：获得 `Consumer` 消息队列锁。
* 第 58 行 ：从消息处理队列顺序获得消息。**和并发消费获得消息不同。并发消费请求在请求创建时，已经设置好消费哪些消息。**
* 第 71 行 ：获得 `Consumer` 消息处理队列消费锁。相比【`Consumer`消息队列锁】，其粒度较小。这就是上文提到的❓**为什么有`Consumer`消息队列锁还需要有 Consumer 消息队列消费锁呢**的原因。
* 第 79 行 ：**执行消费**。
* 第 99 行 ：处理消费结果。

### 3.1.2 处理消费结果

顺序消费消息结果 (`ConsumeOrderlyStatus`) 有四种情况：

* `SUCCESS` ：消费成功**但不提交**。
* `ROLLBACK` ：消费失败，消费回滚。
* `COMMIT` ：消费成功提交并且提交。
* `SUSPEND_CURRENT_QUEUE_A_MOMENT` ：消费失败，挂起消费队列一会会，稍后继续消费。

考虑到 `ROLLBACK` 、`COMMIT` 暂时只使用在 `MySQL binlog` 场景，官方将这两状态标记为 `@Deprecated`。当然，相应的实现逻辑依然保留。

在**并发消费**场景时，如果消费失败，`Consumer` 会将消费失败消息发回到 `Broker` 重试队列，跳过当前消息，等待下次拉取该消息再进行消费。

但是在**完全严格顺序消费**消费时，这样做显然不行。也因此，消费失败的消息，会挂起队列一会会，稍后继续消费。 

不过消费失败的消息一直失败，也不可能一直消费。当超过消费重试上限时，`Consumer` 会将消费失败超过上限的消息发回到 `Broker` 死信队列。

让我们来看看代码：

```Java
  1: // ⬇️⬇️⬇️【ConsumeMessageOrderlyService.java】
  2: /**
  3:  * 处理消费结果，并返回是否继续消费
  4:  *
  5:  * @param msgs 消息
  6:  * @param status 消费结果状态
  7:  * @param context 消费Context
  8:  * @param consumeRequest 消费请求
  9:  * @return 是否继续消费
 10:  */
 11: public boolean processConsumeResult(//
 12:     final List<MessageExt> msgs, //
 13:     final ConsumeOrderlyStatus status, //
 14:     final ConsumeOrderlyContext context, //
 15:     final ConsumeRequest consumeRequest//
 16: ) {
 17:     boolean continueConsume = true;
 18:     long commitOffset = -1L;
 19:     if (context.isAutoCommit()) {
 20:         switch (status) {
 21:             case COMMIT:
 22:             case ROLLBACK:
 23:                 log.warn("the message queue consume result is illegal, we think you want to ack these message {}", consumeRequest.getMessageQueue());
 24:             case SUCCESS:
 25:                 // 提交消息已消费成功到消息处理队列
 26:                 commitOffset = consumeRequest.getProcessQueue().commit();
 27:                 // 统计
 28:                 this.getConsumerStatsManager().incConsumeOKTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), msgs.size());
 29:                 break;
 30:             case SUSPEND_CURRENT_QUEUE_A_MOMENT:
 31:                 // 统计
 32:                 this.getConsumerStatsManager().incConsumeFailedTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), msgs.size());
 33:                 if (checkReconsumeTimes(msgs)) { // 计算是否暂时挂起（暂停）消费N毫秒，默认：10ms
 34:                     // 设置消息重新消费
 35:                     consumeRequest.getProcessQueue().makeMessageToCosumeAgain(msgs);
 36:                     // 提交延迟消费请求
 37:                     this.submitConsumeRequestLater(//
 38:                         consumeRequest.getProcessQueue(), //
 39:                         consumeRequest.getMessageQueue(), //
 40:                         context.getSuspendCurrentQueueTimeMillis());
 41:                     continueConsume = false;
 42:                 } else {
 43:                     commitOffset = consumeRequest.getProcessQueue().commit();
 44:                 }
 45:                 break;
 46:             default:
 47:                 break;
 48:         }
 49:     } else {
 50:         switch (status) {
 51:             case SUCCESS:
 52:                 this.getConsumerStatsManager().incConsumeOKTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), msgs.size());
 53:                 break;
 54:             case COMMIT:
 55:                 // 提交消息已消费成功到消息处理队列
 56:                 commitOffset = consumeRequest.getProcessQueue().commit();
 57:                 break;
 58:             case ROLLBACK:
 59:                 // 设置消息重新消费
 60:                 consumeRequest.getProcessQueue().rollback();
 61:                 this.submitConsumeRequestLater(//
 62:                     consumeRequest.getProcessQueue(), //
 63:                     consumeRequest.getMessageQueue(), //
 64:                     context.getSuspendCurrentQueueTimeMillis());
 65:                 continueConsume = false;
 66:                 break;
 67:             case SUSPEND_CURRENT_QUEUE_A_MOMENT: // 计算是否暂时挂起（暂停）消费N毫秒，默认：10ms
 68:                 this.getConsumerStatsManager().incConsumeFailedTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), msgs.size());
 69:                 if (checkReconsumeTimes(msgs)) {
 70:                     // 设置消息重新消费
 71:                     consumeRequest.getProcessQueue().makeMessageToCosumeAgain(msgs);
 72:                     // 提交延迟消费请求
 73:                     this.submitConsumeRequestLater(//
 74:                         consumeRequest.getProcessQueue(), //
 75:                         consumeRequest.getMessageQueue(), //
 76:                         context.getSuspendCurrentQueueTimeMillis());
 77:                     continueConsume = false;
 78:                 }
 79:                 break;
 80:             default:
 81:                 break;
 82:         }
 83:     }
 84: 
 85:     // 消息处理队列未dropped，提交有效消费进度
 86:     if (commitOffset >= 0 && !consumeRequest.getProcessQueue().isDropped()) {
 87:         this.defaultMQPushConsumerImpl.getOffsetStore().updateOffset(consumeRequest.getMessageQueue(), commitOffset, false);
 88:     }
 89: 
 90:     return continueConsume;
 91: }
 92: 
 93: private int getMaxReconsumeTimes() {
 94:     // default reconsume times: Integer.MAX_VALUE
 95:     if (this.defaultMQPushConsumer.getMaxReconsumeTimes() == -1) {
 96:         return Integer.MAX_VALUE;
 97:     } else {
 98:         return this.defaultMQPushConsumer.getMaxReconsumeTimes();
 99:     }
100: }
101: 
102: /**
103:  * 计算是否要暂停消费
104:  * 不暂停条件：存在消息都超过最大消费次数并且都发回broker成功
105:  *
106:  * @param msgs 消息
107:  * @return 是否要暂停
108:  */
109: private boolean checkReconsumeTimes(List<MessageExt> msgs) {
110:     boolean suspend = false;
111:     if (msgs != null && !msgs.isEmpty()) {
112:         for (MessageExt msg : msgs) {
113:             if (msg.getReconsumeTimes() >= getMaxReconsumeTimes()) {
114:                 MessageAccessor.setReconsumeTime(msg, String.valueOf(msg.getReconsumeTimes()));
115:                 if (!sendMessageBack(msg)) { // 发回失败，中断
116:                     suspend = true;
117:                     msg.setReconsumeTimes(msg.getReconsumeTimes() + 1);
118:                 }
119:             } else {
120:                 suspend = true;
121:                 msg.setReconsumeTimes(msg.getReconsumeTimes() + 1);
122:             }
123:         }
124:     }
125:     return suspend;
126: }
127: 
128: /**
129:  * 发回消息。
130:  * 消息发回broker后，对应的消息队列是死信队列。
131:  *
132:  * @param msg 消息
133:  * @return 是否发送成功
134:  */
135: public boolean sendMessageBack(final MessageExt msg) {
136:     try {
137:         // max reconsume times exceeded then send to dead letter queue.
138:         Message newMsg = new Message(MixAll.getRetryTopic(this.defaultMQPushConsumer.getConsumerGroup()), msg.getBody());
139:         String originMsgId = MessageAccessor.getOriginMessageId(msg);
140:         MessageAccessor.setOriginMessageId(newMsg, UtilAll.isBlank(originMsgId) ? msg.getMsgId() : originMsgId);
141:         newMsg.setFlag(msg.getFlag());
142:         MessageAccessor.setProperties(newMsg, msg.getProperties());
143:         MessageAccessor.putProperty(newMsg, MessageConst.PROPERTY_RETRY_TOPIC, msg.getTopic());
144:         MessageAccessor.setReconsumeTime(newMsg, String.valueOf(msg.getReconsumeTimes()));
145:         MessageAccessor.setMaxReconsumeTimes(newMsg, String.valueOf(getMaxReconsumeTimes()));
146:         newMsg.setDelayTimeLevel(3 + msg.getReconsumeTimes());
147: 
148:         this.defaultMQPushConsumer.getDefaultMQPushConsumerImpl().getmQClientFactory().getDefaultMQProducer().send(newMsg);
149:         return true;
150:     } catch (Exception e) {
151:         log.error("sendMessageBack exception, group: " + this.consumerGroup + " msg: " + msg.toString(), e);
152:     }
153: 
154:     return false;
155: }
```

* ⬆️⬆️⬆️
* 第 21 至 29 行 ：消费成功。在自动提交进度( `AutoCommit` )的情况下，`COMMIT`、`ROLLBACK`、`SUCCESS` 逻辑**已经统一**。
* 第 30 至 45 行 ：消费失败。当消息重试次数超过上限（默认 ：16次）时，将消息发送到 `Broker` 死信队列，跳过这些消息。此时，消息队列无需挂起，继续消费后面的消息。
* 第 85 至 88 行 ：提交消费进度。

### 3.13 消息处理队列核心方法

😈涉及到的四个核心方法的源码：

```Java
  1: // ⬇️⬇️⬇️【ProcessQueue.java】
  2: /**
  3:  * 消息映射
  4:  * key：消息队列位置
  5:  */
  6: private final TreeMap<Long, MessageExt> msgTreeMap = new TreeMap<>();    /**
  7:  * 消息映射临时存储（消费中的消息）
  8:  */
  9: private final TreeMap<Long, MessageExt> msgTreeMapTemp = new TreeMap<>();
 10: 
 11: /**
 12:  * 回滚消费中的消息
 13:  * 逻辑类似于{@link #makeMessageToCosumeAgain(List)}
 14:  */
 15: public void rollback() {
 16:     try {
 17:         this.lockTreeMap.writeLock().lockInterruptibly();
 18:         try {
 19:             this.msgTreeMap.putAll(this.msgTreeMapTemp);
 20:             this.msgTreeMapTemp.clear();
 21:         } finally {
 22:             this.lockTreeMap.writeLock().unlock();
 23:         }
 24:     } catch (InterruptedException e) {
 25:         log.error("rollback exception", e);
 26:     }
 27: }
 28: 
 29: /**
 30:  * 提交消费中的消息已消费成功，返回消费进度
 31:  *
 32:  * @return 消费进度
 33:  */
 34: public long commit() {
 35:     try {
 36:         this.lockTreeMap.writeLock().lockInterruptibly();
 37:         try {
 38:             // 消费进度
 39:             Long offset = this.msgTreeMapTemp.lastKey();
 40: 
 41:             //
 42:             msgCount.addAndGet(this.msgTreeMapTemp.size() * (-1));
 43: 
 44:             //
 45:             this.msgTreeMapTemp.clear();
 46: 
 47:             // 返回消费进度
 48:             if (offset != null) {
 49:                 return offset + 1;
 50:             }
 51:         } finally {
 52:             this.lockTreeMap.writeLock().unlock();
 53:         }
 54:     } catch (InterruptedException e) {
 55:         log.error("commit exception", e);
 56:     }
 57: 
 58:     return -1;
 59: }
 60: 
 61: /**
 62:  * 指定消息重新消费
 63:  * 逻辑类似于{@link #rollback()}
 64:  *
 65:  * @param msgs 消息
 66:  */
 67: public void makeMessageToCosumeAgain(List<MessageExt> msgs) {
 68:     try {
 69:         this.lockTreeMap.writeLock().lockInterruptibly();
 70:         try {
 71:             for (MessageExt msg : msgs) {
 72:                 this.msgTreeMapTemp.remove(msg.getQueueOffset());
 73:                 this.msgTreeMap.put(msg.getQueueOffset(), msg);
 74:             }
 75:         } finally {
 76:             this.lockTreeMap.writeLock().unlock();
 77:         }
 78:     } catch (InterruptedException e) {
 79:         log.error("makeMessageToCosumeAgain exception", e);
 80:     }
 81: }
 82: 
 83: /**
 84:  * 获得持有消息前N条
 85:  *
 86:  * @param batchSize 条数
 87:  * @return 消息
 88:  */
 89: public List<MessageExt> takeMessags(final int batchSize) {
 90:     List<MessageExt> result = new ArrayList<>(batchSize);
 91:     final long now = System.currentTimeMillis();
 92:     try {
 93:         this.lockTreeMap.writeLock().lockInterruptibly();
 94:         this.lastConsumeTimestamp = now;
 95:         try {
 96:             if (!this.msgTreeMap.isEmpty()) {
 97:                 for (int i = 0; i < batchSize; i++) {
 98:                     Map.Entry<Long, MessageExt> entry = this.msgTreeMap.pollFirstEntry();
 99:                     if (entry != null) {
100:                         result.add(entry.getValue());
101:                         msgTreeMapTemp.put(entry.getKey(), entry.getValue());
102:                     } else {
103:                         break;
104:                     }
105:                 }
106:             }
107: 
108:             if (result.isEmpty()) {
109:                 consuming = false;
110:             }
111:         } finally {
112:             this.lockTreeMap.writeLock().unlock();
113:         }
114:     } catch (InterruptedException e) {
115:         log.error("take Messages exception", e);
116:     }
117: 
118:     return result;
119: }
```


