title: RocketMQ 源码分析 —— Message 拉取与消费（下）
date: 2017-05-11
tags:
categories: RocketMQ
permalink: RocketMQ/message-pull-and-consume-second

-------

![](http://www.yunai.me/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋艿的后端小屋】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

- [1、概述](#)
- [2、Consumer](#)
- [3、PushConsumer 一览](#)
- [4、PushConsumer 订阅](#)
	- [DefaultMQPushConsumerImpl#subscribe(...)](#)
		- [FilterAPI.buildSubscriptionData(...)](#)
	- [DefaultMQPushConsumer#registerMessageListener(...)](#)
- [5、PushConsumer 消息队列分配](#)
	- [RebalanceService](#)
	- [MQClientInstance#doRebalance(...)](#)
	- [DefaultMQPushConsumerImpl#doRebalance(...)](#)
	- [RebalanceImpl#doRebalance(...)](#)
		- [RebalanceImpl#rebalanceByTopic(...)](#)
		- [RebalanceImpl#removeUnnecessaryMessageQueue(...)](#)
			- [RebalancePushImpl#removeUnnecessaryMessageQueue(...)](#)
			- [[PullConsumer] RebalancePullImpl#removeUnnecessaryMessageQueue(...)](#)
		- [RebalancePushImpl#dispatchPullRequest(...)](#)
			- [DefaultMQPushConsumerImpl#executePullRequestImmediately(...)](#)
		- [AllocateMessageQueueStrategy](#)
			- [AllocateMessageQueueAveragely](#)
			- [AllocateMessageQueueByMachineRoom](#)
			- [AllocateMessageQueueAveragelyByCircle](#)
			- [AllocateMessageQueueByConfig](#)
- [5、PushConsumer 消费进度读取](#)
	- [RebalancePushImpl#computePullFromWhere(...)](#)
	- [[PullConsumer] RebalancePullImpl#computePullFromWhere(...)](#)
- [6、PushConsumer 拉取消息](#)
	- [PullMessageService](#)
	- [DefaultMQPushConsumerImpl#pullMessage(...)](#)
		- [PullAPIWrapper#pullKernelImpl(...)](#)
			- [PullAPIWrapper#recalculatePullFromWhichNode(...)](#)
			- [MQClientInstance#findBrokerAddressInSubscribe(...)](#)
		- [PullAPIWrapper#processPullResult(...)](#)
		- [ProcessQueue#putMessage(...)](#)
	- [总结](#)
- [6、PushConsumer 消费消息](#)
	- [ConsumeMessageConcurrentlyService 提交消费请求](#)
		- [ConsumeMessageConcurrentlyService#submitConsumeRequest(...)](#)
		- [ConsumeMessageConcurrentlyService#submitConsumeRequestLater](#)
	- [ConsumeRequest](#)
	- [ConsumeMessageConcurrentlyService#processConsumeResult(...)](#)
		- [ProcessQueue#removeMessage(...)](#)
	- [ConsumeMessageConcurrentlyService#cleanExpireMsg(...)](#)
		- [ProcessQueue#cleanExpiredMsg(...)](#)
- [7、PushConsumer 发回消费失败消息](#)
	- [DefaultMQPushConsumerImpl#sendMessageBack(...)](#)
		- [MQClientAPIImpl#consumerSendMessageBack(...)](#)
- [8、Consumer 消费进度](#)
	- [OffsetStore](#)
		- [OffsetStore#load(...)](#)
			- [LocalFileOffsetStore#load(...)](#)
				- [OffsetSerializeWrapper](#)
			- [RemoteBrokerOffsetStore#load(...)](#)
		- [OffsetStore#readOffset(...)](#)
			- [LocalFileOffsetStore#readOffset(...)](#)
			- [RemoteBrokerOffsetStore#readOffset(...)](#)
		- [OffsetStore#updateOffset(...)](#)
		- [OffsetStore#persistAll(...)](#)
			- [LocalFileOffsetStore#persistAll(...)](#)
			- [RemoteBrokerOffsetStore#persistAll(...)](#)
			- [MQClientInstance#persistAllConsumerOffset(...)](#)
- [9、结尾](#)

-------

# 1、概述

本文接：[《RocketMQ 源码分析 —— Message 拉取与消费（上）》](http://www.yunai.me/RocketMQ/message-pull-and-consume-first/)。

主要解析 `Consumer` 在 **消费** 逻辑涉及到的源码。

# 2、Consumer

MQ 提供了两类消费者：

* PushConsumer：
    * 在大多数场景下使用。
    * 名字虽然是 `Push` 开头，实际在实现时，使用 `Pull` 方式实现。通过 `Pull` **不断不断不断**轮询 `Broker` 获取消息。当不存在新消息时，`Broker` 会**挂起请求**，直到有新消息产生，取消挂起，返回新消息。这样，基本和 `Broker` 主动 `Push` 做到**接近**的实时性（当然，还是有相应的实时性损失）。原理类似 **[长轮询( `Long-Polling` )](https://www.ibm.com/developerworks/cn/web/wa-lo-comet/)**。
* PullConsumer

**本文主要讲解`PushConsumer`，部分讲解`PullConsumer`，跳过`顺序消费`。**  
**本文主要讲解`PushConsumer`，部分讲解`PullConsumer`，跳过`顺序消费`。**  
**本文主要讲解`PushConsumer`，部分讲解`PullConsumer`，跳过`顺序消费`。**  

# 3、PushConsumer 一览

先看一张 `PushConsumer` 包含的组件以及组件之间的交互图：

![PushConsumer手绘图.png](http://www.yunai.me/images/RocketMQ/2017_05_04/09.png)

* `RebalanceService`：均衡消息队列服务，负责分配当前 `Consumer` 可消费的消息队列( `MessageQueue` )。当有新的 `Consumer` 的加入或移除，都会重新分配消息队列。
* `PullMessageService`：拉取消息服务，**不断不断不断**从 `Broker` 拉取消息，并提交消费任务到 `ConsumeMessageService`。
* `ConsumeMessageService`：消费消息服务，**不断不断不断**消费消息，并处理消费结果。
* `RemoteBrokerOffsetStore`：`Consumer` 消费进度管理，负责从 `Broker` 获取消费进度，同步消费进度到 `Broker`。
* `ProcessQueue` ：消息处理队列。
* `MQClientInstance` ：封装对 `Namesrv`，`Broker` 的 API调用，提供给 `Producer`、`Consumer` 使用。

# 4、PushConsumer 订阅

## DefaultMQPushConsumerImpl#subscribe(...)

```Java
  1: public void subscribe(String topic, String subExpression) throws MQClientException {
  2:     try {
  3:         // 创建订阅数据
  4:         SubscriptionData subscriptionData = FilterAPI.buildSubscriptionData(this.defaultMQPushConsumer.getConsumerGroup(), //
  5:             topic, subExpression);
  6:         this.rebalanceImpl.getSubscriptionInner().put(topic, subscriptionData);
  7:         // 通过心跳同步Consumer信息到Broker
  8:         if (this.mQClientFactory != null) {
  9:             this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
 10:         }
 11:     } catch (Exception e) {
 12:         throw new MQClientException("subscription exception", e);
 13:     }
 14: }
```

* 说明 ：订阅 `Topic` 。
* 第 3 至 6 行 ：创建订阅数据。详细解析见：[FilterAPI.buildSubscriptionData(...)](#filterapibuildsubscriptiondata)。
* 第 7 至 10 行 ：通过心跳同步 `Consumer` 信息到 `Broker`。

### FilterAPI.buildSubscriptionData(...)

```Java
  1: public static SubscriptionData buildSubscriptionData(final String consumerGroup, String topic,
  2:     String subString) throws Exception {
  3:     SubscriptionData subscriptionData = new SubscriptionData();
  4:     subscriptionData.setTopic(topic);
  5:     subscriptionData.setSubString(subString);
  6:     // 处理订阅表达式
  7:     if (null == subString || subString.equals(SubscriptionData.SUB_ALL) || subString.length() == 0) {
  8:         subscriptionData.setSubString(SubscriptionData.SUB_ALL);
  9:     } else {
 10:         String[] tags = subString.split("\\|\\|");
 11:         if (tags.length > 0) {
 12:             for (String tag : tags) {
 13:                 if (tag.length() > 0) {
 14:                     String trimString = tag.trim();
 15:                     if (trimString.length() > 0) {
 16:                         subscriptionData.getTagsSet().add(trimString);
 17:                         subscriptionData.getCodeSet().add(trimString.hashCode());
 18:                     }
 19:                 }
 20:             }
 21:         } else {
 22:             throw new Exception("subString split error");
 23:         }
 24:     }
 25: 
 26:     return subscriptionData;
 27: }
```

* 说明 ：根据 `Topic` 和 订阅表达式 创建订阅数据
* subscriptionData.subVersion = System.currentTimeMillis()。

## DefaultMQPushConsumer#registerMessageListener(...)

```Java
  1: public void registerMessageListener(MessageListenerConcurrently messageListener) {
  2:     this.messageListener = messageListener;
  3:     this.defaultMQPushConsumerImpl.registerMessageListener(messageListener);
  4: }
```

* 说明 ：注册消息监听器。

# 5、PushConsumer 消息队列分配

![RebalanceService&PushConsumer分配队列](http://www.yunai.me/images/RocketMQ/2017_05_04/10.png)

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

* 说明 ：均衡消息队列服务，负责分配当前 `Consumer` 可消费的消息队列( `MessageQueue` )。
* 第 26 行 ：调用 `MQClientInstance#doRebalance(...)` 分配消息队列。目前有三种情况情况下触发：
    * 如 `第 25 行` 等待超时，每 20s 调用一次。
    * `PushConsumer` 启动时，调用 `rebalanceService#wakeup(...)` 触发。
    * `Broker` 通知 `Consumer` 加入 或 移除时，`Consumer` 响应通知，调用 `rebalanceService#wakeup(...)` 触发。

 详细解析见：[MQClientInstance#doRebalance(...)](#mqclientinstancedorebalance)。

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

* 说明 ：遍历当前 `Client` 包含的 `consumerTable`( `Consumer`集合 )，执行消息队列分配。
* **疑问**：目前代码调试下来，`consumerTable` 只包含 `Consumer` 自己。😈有大大对这个疑问有解答的，烦请解答下。
* 第 6 行 ：调用 `MQConsumerInner#doRebalance(...)` 进行队列分配。`DefaultMQPushConsumerImpl`、`DefaultMQPullConsumerImpl` 分别对该接口方法进行了实现。`DefaultMQPushConsumerImpl#doRebalance(...)` 详细解析见：[DefaultMQPushConsumerImpl#doRebalance(...)](defaultmqpushconsumerimpldorebalance)。

## DefaultMQPushConsumerImpl#doRebalance(...)

```Java
  1: public void doRebalance() {
  2:     if (!this.pause) {
  3:         this.rebalanceImpl.doRebalance(this.isConsumeOrderly());
  4:     }
  5: }
```

* 说明：执行消息队列分配。
* 第 3 行 ：调用 `RebalanceImpl#doRebalance(...)` 进行队列分配。详细解析见：[RebalancePushImpl#doRebalance(...)](#rebalancepushimpldorebalance)。

## RebalanceImpl#doRebalance(...)

```Java
  1: /**
  2:  * 执行分配消息队列
  3:  *
  4:  * @param isOrder 是否顺序消息
  5:  */
  6: public void doRebalance(final boolean isOrder) {
  7:     // 分配每个 topic 的消息队列
  8:     Map<String, SubscriptionData> subTable = this.getSubscriptionInner();
  9:     if (subTable != null) {
 10:         for (final Map.Entry<String, SubscriptionData> entry : subTable.entrySet()) {
 11:             final String topic = entry.getKey();
 12:             try {
 13:                 this.rebalanceByTopic(topic, isOrder);
 14:             } catch (Throwable e) {
 15:                 if (!topic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
 16:                     log.warn("rebalanceByTopic Exception", e);
 17:                 }
 18:             }
 19:         }
 20:     }
 21:     // 移除未订阅的topic对应的消息队列
 22:     this.truncateMessageQueueNotMyTopic();
 23: }
 24: 
 25: /**
 26:  * 移除未订阅的消息队列
 27:  */
 28: private void truncateMessageQueueNotMyTopic() {
 29:     Map<String, SubscriptionData> subTable = this.getSubscriptionInner();
 30:     for (MessageQueue mq : this.processQueueTable.keySet()) {
 31:         if (!subTable.containsKey(mq.getTopic())) {
 32: 
 33:             ProcessQueue pq = this.processQueueTable.remove(mq);
 34:             if (pq != null) {
 35:                 pq.setDropped(true);
 36:                 log.info("doRebalance, {}, truncateMessageQueueNotMyTopic remove unnecessary mq, {}", consumerGroup, mq);
 37:             }
 38:         }
 39:     }
 40: }
```

* `#doRebalance(...)` 说明 ：执行分配消息队列。
    * 第 7 至 20 行 ：循环订阅主题集合( `subscriptionInner` )，分配每一个 `Topic` 的消息队列。
    * 第 22 行 ：移除未订阅的 `Topic` 的消息队列。
* `#truncateMessageQueueNotMyTopic(...)` 说明 ：移除未订阅的消息队列。**当调用 `DefaultMQPushConsumer#unsubscribe(topic)` 时，只移除订阅主题集合( `subscriptionInner` )，对应消息队列移除在该方法。**

### RebalanceImpl#rebalanceByTopic(...)

```Java
  1: private void rebalanceByTopic(final String topic, final boolean isOrder) {
  2:     switch (messageModel) {
  3:         case BROADCASTING: {
  4:             Set<MessageQueue> mqSet = this.topicSubscribeInfoTable.get(topic);
  5:             if (mqSet != null) {
  6:                 boolean changed = this.updateProcessQueueTableInRebalance(topic, mqSet, isOrder);
  7:                 if (changed) {
  8:                     this.messageQueueChanged(topic, mqSet, mqSet);
  9:                     log.info("messageQueueChanged {} {} {} {}", //
 10:                         consumerGroup, //
 11:                         topic, //
 12:                         mqSet, //
 13:                         mqSet);
 14:                 }
 15:             } else {
 16:                 log.warn("doRebalance, {}, but the topic[{}] not exist.", consumerGroup, topic);
 17:             }
 18:             break;
 19:         }
 20:         case CLUSTERING: {
 21:             // 获取 topic 对应的 队列 和 consumer信息
 22:             Set<MessageQueue> mqSet = this.topicSubscribeInfoTable.get(topic);
 23:             List<String> cidAll = this.mQClientFactory.findConsumerIdList(topic, consumerGroup);
 24:             if (null == mqSet) {
 25:                 if (!topic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
 26:                     log.warn("doRebalance, {}, but the topic[{}] not exist.", consumerGroup, topic);
 27:                 }
 28:             }
 29: 
 30:             if (null == cidAll) {
 31:                 log.warn("doRebalance, {} {}, get consumer id list failed", consumerGroup, topic);
 32:             }
 33: 
 34:             if (mqSet != null && cidAll != null) {
 35:                 // 排序 消息队列 和 消费者数组。因为是在Client进行分配队列，排序后，各Client的顺序才能保持一致。
 36:                 List<MessageQueue> mqAll = new ArrayList<>();
 37:                 mqAll.addAll(mqSet);
 38: 
 39:                 Collections.sort(mqAll);
 40:                 Collections.sort(cidAll);
 41: 
 42:                 AllocateMessageQueueStrategy strategy = this.allocateMessageQueueStrategy;
 43: 
 44:                 // 根据 队列分配策略 分配消息队列
 45:                 List<MessageQueue> allocateResult;
 46:                 try {
 47:                     allocateResult = strategy.allocate(//
 48:                         this.consumerGroup, //
 49:                         this.mQClientFactory.getClientId(), //
 50:                         mqAll, //
 51:                         cidAll);
 52:                 } catch (Throwable e) {
 53:                     log.error("AllocateMessageQueueStrategy.allocate Exception. allocateMessageQueueStrategyName={}", strategy.getName(),
 54:                         e);
 55:                     return;
 56:                 }
 57: 
 58:                 Set<MessageQueue> allocateResultSet = new HashSet<>();
 59:                 if (allocateResult != null) {
 60:                     allocateResultSet.addAll(allocateResult);
 61:                 }
 62: 
 63:                 // 更新消息队列
 64:                 boolean changed = this.updateProcessQueueTableInRebalance(topic, allocateResultSet, isOrder);
 65:                 if (changed) {
 66:                     log.info(
 67:                         "rebalanced result changed. allocateMessageQueueStrategyName={}, group={}, topic={}, clientId={}, mqAllSize={}, cidAllSize={}, rebalanceResultSize={}, rebalanceResultSet={}",
 68:                         strategy.getName(), consumerGroup, topic, this.mQClientFactory.getClientId(), mqSet.size(), cidAll.size(),
 69:                         allocateResultSet.size(), allocateResultSet);
 70:                     this.messageQueueChanged(topic, mqSet, allocateResultSet);
 71:                 }
 72:             }
 73:             break;
 74:         }
 75:         default:
 76:             break;
 77:     }
 78: }
 79: 
 80: /**
 81:  * 当负载均衡时，更新 消息处理队列
 82:  * - 移除 在processQueueTable && 不存在于 mqSet 里的消息队列
 83:  * - 增加 不在processQueueTable && 存在于mqSet 里的消息队列
 84:  *
 85:  * @param topic Topic
 86:  * @param mqSet 负载均衡结果后的消息队列数组
 87:  * @param isOrder 是否顺序
 88:  * @return 是否变更
 89:  */
 90: private boolean updateProcessQueueTableInRebalance(final String topic, final Set<MessageQueue> mqSet, final boolean isOrder) {
 91:     boolean changed = false;
 92: 
 93:     // 移除 在processQueueTable && 不存在于 mqSet 里的消息队列
 94:     Iterator<Entry<MessageQueue, ProcessQueue>> it = this.processQueueTable.entrySet().iterator();
 95:     while (it.hasNext()) { // TODO 待读：
 96:         Entry<MessageQueue, ProcessQueue> next = it.next();
 97:         MessageQueue mq = next.getKey();
 98:         ProcessQueue pq = next.getValue();
 99: 
100:         if (mq.getTopic().equals(topic)) {
101:             if (!mqSet.contains(mq)) { // 不包含的队列
102:                 pq.setDropped(true);
103:                 if (this.removeUnnecessaryMessageQueue(mq, pq)) {
104:                     it.remove();
105:                     changed = true;
106:                     log.info("doRebalance, {}, remove unnecessary mq, {}", consumerGroup, mq);
107:                 }
108:             } else if (pq.isPullExpired()) { // 队列拉取超时，进行清理
109:                 switch (this.consumeType()) {
110:                     case CONSUME_ACTIVELY:
111:                         break;
112:                     case CONSUME_PASSIVELY:
113:                         pq.setDropped(true);
114:                         if (this.removeUnnecessaryMessageQueue(mq, pq)) {
115:                             it.remove();
116:                             changed = true;
117:                             log.error("[BUG]doRebalance, {}, remove unnecessary mq, {}, because pull is pause, so try to fixed it",
118:                                 consumerGroup, mq);
119:                         }
120:                         break;
121:                     default:
122:                         break;
123:                 }
124:             }
125:         }
126:     }
127: 
128:     // 增加 不在processQueueTable && 存在于mqSet 里的消息队列。
129:     List<PullRequest> pullRequestList = new ArrayList<>(); // 拉消息请求数组
130:     for (MessageQueue mq : mqSet) {
131:         if (!this.processQueueTable.containsKey(mq)) {
132:             if (isOrder && !this.lock(mq)) {
133:                 log.warn("doRebalance, {}, add a new mq failed, {}, because lock failed", consumerGroup, mq);
134:                 continue;
135:             }
136: 
137:             this.removeDirtyOffset(mq);
138:             ProcessQueue pq = new ProcessQueue();
139:             long nextOffset = this.computePullFromWhere(mq);
140:             if (nextOffset >= 0) {
141:                 ProcessQueue pre = this.processQueueTable.putIfAbsent(mq, pq);
142:                 if (pre != null) {
143:                     log.info("doRebalance, {}, mq already exists, {}", consumerGroup, mq);
144:                 } else {
145:                     log.info("doRebalance, {}, add a new mq, {}", consumerGroup, mq);
146:                     PullRequest pullRequest = new PullRequest();
147:                     pullRequest.setConsumerGroup(consumerGroup);
148:                     pullRequest.setNextOffset(nextOffset);
149:                     pullRequest.setMessageQueue(mq);
150:                     pullRequest.setProcessQueue(pq);
151:                     pullRequestList.add(pullRequest);
152:                     changed = true;
153:                 }
154:             } else {
155:                 log.warn("doRebalance, {}, add new mq failed, {}", consumerGroup, mq);
156:             }
157:         }
158:     }
159: 
160:     // 发起消息拉取请求
161:     this.dispatchPullRequest(pullRequestList);
162: 
163:     return changed;
164: }
```

* `#rebalanceByTopic(...)` 说明 ：分配 `Topic` 的消息队列。
    * 第 3 至 19 行 ：广播模式( `BROADCASTING` ) 下，分配 `Topic` 对应的**所有**消息队列。   
    * 第 20 至 74 行 ：集群模式( `CLUSTERING` ) 下，分配 `Topic` 对应的**部分**消息队列。
        * 第 21 至 40 行 ：获取 `Topic` 对应的消息队列和消费者们，并对其进行排序。因为各 `Consumer` 是在本地分配消息队列，排序后才能保证各 `Consumer` 顺序一致。
        *  第 42 至 61 行 ：根据 队列分配策略( `AllocateMessageQueueStrategy` ) 分配消息队列。详细解析见：[AllocateMessageQueueStrategy](#allocatemessagequeuestrategy)。
        *  第 63 至 72 行 ：更新 `Topic` 对应的消息队列。
* `#updateProcessQueueTableInRebalance(...)` 说明 ：当分配队列时，更新 `Topic` 对应的消息队列，并返回是否有变更。
    * 第 93 至 126 行 ：移除不存在于分配的消息队列( `mqSet` ) 的 消息处理队列( `processQueueTable` )。
        * 第 103 行 ：移除不需要的消息队列。详细解析见：[RebalancePushImpl#removeUnnecessaryMessageQueue(...)](#rebalancepushimplremoveunnecessarymessagequeue)。
        * 第 108 至 120 行 ：队列拉取超时，即 `当前时间 - 最后一次拉取消息时间 > 120s` ( 120s 可配置)，判定发生 **BUG**，过久未进行消息拉取，移除消息队列。移除后，下面**#新增队列逻辑#**可以重新加入新的该消息队列。
    * 第 128 至 158 行 ：增加 分配的消息队列( `mqSet` ) 新增的消息队列。
        * 第 132 至 135 行 ：`顺序消费` 相关跳过，详细解析见：[《RocketMQ 源码分析 —— Message 顺序发送与消费》](http://www.yunai.me/RocketMQ/message-send-and-consume-orderly/)。
        * 第 137 行 ：移除消息队列的消费进度。
        * 第 139 行 ：获取队列消费进度。详细解析见：[RebalancePushImpl#computePullFromWhere(...)](#rebalancepushimplcomputepullfromwhere)。
        * 第 140 至 156 行 ：**添加新消费处理队列，添加消费拉取消息请求**。
    * 第 161 行 ：**发起新增的消息队列消息拉取请求**。详细解析见：[RebalancePushImpl#dispatchPullRequest(...)](#rebalancepushimpldispatchpullrequest)。

### RebalanceImpl#removeUnnecessaryMessageQueue(...)

#### RebalancePushImpl#removeUnnecessaryMessageQueue(...)

```Java
  1: public boolean removeUnnecessaryMessageQueue(MessageQueue mq, ProcessQueue pq) {
  2:     // 同步队列的消费进度，并移除之。
  3:     this.defaultMQPushConsumerImpl.getOffsetStore().persist(mq);
  4:     this.defaultMQPushConsumerImpl.getOffsetStore().removeOffset(mq);
  5:     // TODO 顺序消费
  6:     if (this.defaultMQPushConsumerImpl.isConsumeOrderly()
  7:         && MessageModel.CLUSTERING.equals(this.defaultMQPushConsumerImpl.messageModel())) {
  8:         try {
  9:             if (pq.getLockConsume().tryLock(1000, TimeUnit.MILLISECONDS)) {
 10:                 try {
 11:                     return this.unlockDelay(mq, pq);
 12:                 } finally {
 13:                     pq.getLockConsume().unlock();
 14:                 }
 15:             } else {
 16:                 log.warn("[WRONG]mq is consuming, so can not unlock it, {}. maybe hanged for a while, {}", //
 17:                     mq, //
 18:                     pq.getTryUnlockTimes());
 19: 
 20:                 pq.incTryUnlockTimes();
 21:             }
 22:         } catch (Exception e) {
 23:             log.error("removeUnnecessaryMessageQueue Exception", e);
 24:         }
 25: 
 26:         return false;
 27:     }
 28:     return true;
 29: }
```

* 说明 ：移除不需要的消息队列相关的信息，并返回是否移除成功。
* 第 2 至 4 行 ：**同步**队列的消费进度，并移除之。
* 第 5 至 27 行 ：`顺序消费` 相关跳过，详细解析见：[《RocketMQ 源码分析 —— Message 顺序发送与消费》](http://www.yunai.me/RocketMQ/message-send-and-consume-orderly/)。

#### `[PullConsumer]` RebalancePullImpl#removeUnnecessaryMessageQueue(...)

```Java
  1: public boolean removeUnnecessaryMessageQueue(MessageQueue mq, ProcessQueue pq) {
  2:     this.defaultMQPullConsumerImpl.getOffsetStore().persist(mq);
  3:     this.defaultMQPullConsumerImpl.getOffsetStore().removeOffset(mq);
  4:     return true;
  5: }
```

* 说明 ：移除不需要的消息队列相关的信息，并返回移除成功。**和`RebalancePushImpl#removeUnnecessaryMessageQueue(...)`基本一致。**

### RebalancePushImpl#dispatchPullRequest(...)

```Java
  1: public void dispatchPullRequest(List<PullRequest> pullRequestList) {
  2:     for (PullRequest pullRequest : pullRequestList) {
  3:         this.defaultMQPushConsumerImpl.executePullRequestImmediately(pullRequest);
  4:         log.info("doRebalance, {}, add a new pull request {}", consumerGroup, pullRequest);
  5:     }
  6: }
```

* 说明 ：发起消息拉取请求。**该调用是`PushConsumer`不断不断不断拉取消息的起点**。

#### DefaultMQPushConsumerImpl#executePullRequestImmediately(...)

```Java
  1: public void executePullRequestImmediately(final PullRequest pullRequest) {
  2:     this.mQClientFactory.getPullMessageService().executePullRequestImmediately(pullRequest);
  3: }
```

* 说明 ：提交拉取请求。提交后，`PullMessageService` **异步执行**，**非阻塞**。详细解析见：[PullMessageService](pullmessageservice)。

### AllocateMessageQueueStrategy

![AllocateMessageQueueStrategy类图](http://www.yunai.me/images/RocketMQ/2017_05_04/01.png)

#### AllocateMessageQueueAveragely

```Java
  1: public class AllocateMessageQueueAveragely implements AllocateMessageQueueStrategy {
  2:     private final Logger log = ClientLogger.getLog();
  3: 
  4:     @Override
  5:     public List<MessageQueue> allocate(String consumerGroup, String currentCID, List<MessageQueue> mqAll,
  6:         List<String> cidAll) {
  7:         // 校验参数是否正确
  8:         if (currentCID == null || currentCID.length() < 1) {
  9:             throw new IllegalArgumentException("currentCID is empty");
 10:         }
 11:         if (mqAll == null || mqAll.isEmpty()) {
 12:             throw new IllegalArgumentException("mqAll is null or mqAll empty");
 13:         }
 14:         if (cidAll == null || cidAll.isEmpty()) {
 15:             throw new IllegalArgumentException("cidAll is null or cidAll empty");
 16:         }
 17: 
 18:         List<MessageQueue> result = new ArrayList<>();
 19:         if (!cidAll.contains(currentCID)) {
 20:             log.info("[BUG] ConsumerGroup: {} The consumerId: {} not in cidAll: {}",
 21:                 consumerGroup,
 22:                 currentCID,
 23:                 cidAll);
 24:             return result;
 25:         }
 26:         // 平均分配
 27:         int index = cidAll.indexOf(currentCID); // 第几个consumer。
 28:         int mod = mqAll.size() % cidAll.size(); // 余数，即多少消息队列无法平均分配。
 29:         int averageSize =
 30:             mqAll.size() <= cidAll.size() ? 1 : (mod > 0 && index < mod ? mqAll.size() / cidAll.size()
 31:                 + 1 : mqAll.size() / cidAll.size());
 32:         int startIndex = (mod > 0 && index < mod) ? index * averageSize : index * averageSize + mod; // 有余数的情况下，[0, mod) 平分余数，即每consumer多分配一个节点；第index开始，跳过前mod余数。
 33:         int range = Math.min(averageSize, mqAll.size() - startIndex); // 分配队列数量。之所以要Math.min()的原因是，mqAll.size() <= cidAll.size()，部分consumer分配不到消息队列。
 34:         for (int i = 0; i < range; i++) {
 35:             result.add(mqAll.get((startIndex + i) % mqAll.size()));
 36:         }
 37:         return result;
 38:     }
 39: 
 40:     @Override
 41:     public String getName() {
 42:         return "AVG";
 43:     }
 44: }
```

* 说明 ：**平均**分配队列策略。
* 第 7 至 25 行 ：参数校验。
* 第 26 至 36 行 ：平均分配消息队列。
    * 第 27 行 ：`index` ：当前 `Consumer` 在消费集群里是第几个。这里就是为什么需要对传入的 `cidAll` 参数必须进行排序的原因。如果不排序，`Consumer` 在本地计算出来的 `index` 无法一致，影响计算结果。
    * 第 28 行 ：`mod` ：余数，即多少消息队列无法平均分配。
    * 第 29 至 31 行 ：`averageSize` ：代码可以简化成 `(mod > 0 && index < mod ? mqAll.size() / cidAll.size() + 1 : mqAll.size() / cidAll.size())`。
        * `[ 0, mod )` ：`mqAll.size() / cidAll.size() + 1`。前面 `mod` 个 `Consumer` 平分余数，多获得 1 个消息队列。
        * `[ mod, cidAll.size() )` ：`mqAll.size() / cidAll.size()`。
    * 第 32 行 ：`startIndex` ：`Consumer` 分配消息队列开始位置。
    * 第 33 行 ：`range` ：分配队列数量。之所以要 `Math#min(...)` 的原因：当 `mqAll.size() <= cidAll.size()` 时，最后几个 `Consumer` 分配不到消息队列。
    * 第 34 至 36 行 ：生成分配消息队列结果。
* 举个例子：

固定消息队列长度为**4**。

|   | Consumer * 2 *可以整除* | Consumer * 3 *不可整除* | Consumer * 5 *无法都分配* |
| --- | --- | --- | --- |
| 消息队列[0] | Consumer[0] | Consumer[0] | Consumer[0] |
| 消息队列[1] | Consumer[0] | Consumer[0] | Consumer[1] |
| 消息队列[2] | Consumer[1] | Consumer[1] | Consumer[2] |
| 消息队列[3] | Consumer[1] | Consumer[2] | Consumer[3] |

#### AllocateMessageQueueByMachineRoom

```Java
  1: public class AllocateMessageQueueByMachineRoom implements AllocateMessageQueueStrategy {
  2:     /**
  3:      * 消费者消费brokerName集合
  4:      */
  5:     private Set<String> consumeridcs;
  6: 
  7:     @Override
  8:     public List<MessageQueue> allocate(String consumerGroup, String currentCID, List<MessageQueue> mqAll,
  9:         List<String> cidAll) {
 10:         // 参数校验
 11:         List<MessageQueue> result = new ArrayList<MessageQueue>();
 12:         int currentIndex = cidAll.indexOf(currentCID);
 13:         if (currentIndex < 0) {
 14:             return result;
 15:         }
 16:         // 计算符合当前配置的消费者数组('consumeridcs')对应的消息队列
 17:         List<MessageQueue> premqAll = new ArrayList<MessageQueue>();
 18:         for (MessageQueue mq : mqAll) {
 19:             String[] temp = mq.getBrokerName().split("@");
 20:             if (temp.length == 2 && consumeridcs.contains(temp[0])) {
 21:                 premqAll.add(mq);
 22:             }
 23:         }
 24:         // 平均分配
 25:         int mod = premqAll.size() / cidAll.size();
 26:         int rem = premqAll.size() % cidAll.size();
 27:         int startIndex = mod * currentIndex;
 28:         int endIndex = startIndex + mod;
 29:         for (int i = startIndex; i < endIndex; i++) {
 30:             result.add(mqAll.get(i));
 31:         }
 32:         if (rem > currentIndex) {
 33:             result.add(premqAll.get(currentIndex + mod * cidAll.size()));
 34:         }
 35:         return result;
 36:     }
 37: 
 38:     @Override
 39:     public String getName() {
 40:         return "MACHINE_ROOM";
 41:     }
 42: 
 43:     public Set<String> getConsumeridcs() {
 44:         return consumeridcs;
 45:     }
 46: 
 47:     public void setConsumeridcs(Set<String> consumeridcs) {
 48:         this.consumeridcs = consumeridcs;
 49:     }
 50: }
```

* 说明 ：**平均**分配**可消费的** `Broker` 对应的消息队列。
* 第 7 至 15 行 ：参数校验。
* 第 16 至 23 行 ：计算**可消费的** `Broker` 对应的消息队列。
* 第 25 至 34 行 ：平均分配消息队列。该**平均分配**方式和 `AllocateMessageQueueAveragely` 略有不同，其是将多余的结尾部分分配给前 `rem` 个 `Consumer`。
* 疑问：*使用该分配策略时，`Consumer` 和 `Broker` 分配需要怎么配置*。😈等研究**主从**相关源码时，仔细考虑下。

#### AllocateMessageQueueAveragelyByCircle

 ```Java
   1: public class AllocateMessageQueueAveragelyByCircle implements AllocateMessageQueueStrategy {
  2:     private final Logger log = ClientLogger.getLog();
  3: 
  4:     @Override
  5:     public List<MessageQueue> allocate(String consumerGroup, String currentCID, List<MessageQueue> mqAll,
  6:         List<String> cidAll) {
  7:         // 校验参数是否正确
  8:         if (currentCID == null || currentCID.length() < 1) {
  9:             throw new IllegalArgumentException("currentCID is empty");
 10:         }
 11:         if (mqAll == null || mqAll.isEmpty()) {
 12:             throw new IllegalArgumentException("mqAll is null or mqAll empty");
 13:         }
 14:         if (cidAll == null || cidAll.isEmpty()) {
 15:             throw new IllegalArgumentException("cidAll is null or cidAll empty");
 16:         }
 17: 
 18:         List<MessageQueue> result = new ArrayList<MessageQueue>();
 19:         if (!cidAll.contains(currentCID)) {
 20:             log.info("[BUG] ConsumerGroup: {} The consumerId: {} not in cidAll: {}",
 21:                 consumerGroup,
 22:                 currentCID,
 23:                 cidAll);
 24:             return result;
 25:         }
 26: 
 27:         // 环状分配
 28:         int index = cidAll.indexOf(currentCID);
 29:         for (int i = index; i < mqAll.size(); i++) {
 30:             if (i % cidAll.size() == index) {
 31:                 result.add(mqAll.get(i));
 32:             }
 33:         }
 34:         return result;
 35:     }
 36: 
 37:     @Override
 38:     public String getName() {
 39:         return "AVG_BY_CIRCLE";
 40:     }
 41: }
 ```
 
 * 说明 ：环状分配消息队列。

#### AllocateMessageQueueByConfig

```Java
  1: public class AllocateMessageQueueByConfig implements AllocateMessageQueueStrategy {
  2:     private List<MessageQueue> messageQueueList;
  3: 
  4:     @Override
  5:     public List<MessageQueue> allocate(String consumerGroup, String currentCID, List<MessageQueue> mqAll,
  6:         List<String> cidAll) {
  7:         return this.messageQueueList;
  8:     }
  9: 
 10:     @Override
 11:     public String getName() {
 12:         return "CONFIG";
 13:     }
 14: 
 15:     public List<MessageQueue> getMessageQueueList() {
 16:         return messageQueueList;
 17:     }
 18: 
 19:     public void setMessageQueueList(List<MessageQueue> messageQueueList) {
 20:         this.messageQueueList = messageQueueList;
 21:     }
 22: }
```

* 说明 ：分配**配置的**消息队列。
* 疑问 ：*该分配策略的使用场景。*

# 5、PushConsumer 消费进度读取

## RebalancePushImpl#computePullFromWhere(...)

```Java
  1: public long computePullFromWhere(MessageQueue mq) {
  2:     long result = -1;
  3:     final ConsumeFromWhere consumeFromWhere = this.defaultMQPushConsumerImpl.getDefaultMQPushConsumer().getConsumeFromWhere();
  4:     final OffsetStore offsetStore = this.defaultMQPushConsumerImpl.getOffsetStore();
  5:     switch (consumeFromWhere) {
  6:         case CONSUME_FROM_LAST_OFFSET_AND_FROM_MIN_WHEN_BOOT_FIRST: // 废弃
  7:         case CONSUME_FROM_MIN_OFFSET: // 废弃
  8:         case CONSUME_FROM_MAX_OFFSET: // 废弃
  9:         case CONSUME_FROM_LAST_OFFSET: {
 10:             long lastOffset = offsetStore.readOffset(mq, ReadOffsetType.READ_FROM_STORE);
 11:             if (lastOffset >= 0) {
 12:                 result = lastOffset;
 13:             }
 14:             // First start,no offset
 15:             else if (-1 == lastOffset) {
 16:                 if (mq.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
 17:                     result = 0L;
 18:                 } else {
 19:                     try {
 20:                         result = this.mQClientFactory.getMQAdminImpl().maxOffset(mq);
 21:                     } catch (MQClientException e) {
 22:                         result = -1;
 23:                     }
 24:                 }
 25:             } else {
 26:                 result = -1;
 27:             }
 28:             break;
 29:         }
 30:         case CONSUME_FROM_FIRST_OFFSET: {
 31:             long lastOffset = offsetStore.readOffset(mq, ReadOffsetType.READ_FROM_STORE);
 32:             if (lastOffset >= 0) {
 33:                 result = lastOffset;
 34:             } else if (-1 == lastOffset) {
 35:                 result = 0L;
 36:             } else {
 37:                 result = -1;
 38:             }
 39:             break;
 40:         }
 41:         case CONSUME_FROM_TIMESTAMP: {
 42:             long lastOffset = offsetStore.readOffset(mq, ReadOffsetType.READ_FROM_STORE);
 43:             if (lastOffset >= 0) {
 44:                 result = lastOffset;
 45:             } else if (-1 == lastOffset) {
 46:                 if (mq.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
 47:                     try {
 48:                         result = this.mQClientFactory.getMQAdminImpl().maxOffset(mq);
 49:                     } catch (MQClientException e) {
 50:                         result = -1;
 51:                     }
 52:                 } else {
 53:                     try {
 54:                         long timestamp = UtilAll.parseDate(this.defaultMQPushConsumerImpl.getDefaultMQPushConsumer().getConsumeTimestamp(),
 55:                             UtilAll.YYYY_MMDD_HHMMSS).getTime();
 56:                         result = this.mQClientFactory.getMQAdminImpl().searchOffset(mq, timestamp);
 57:                     } catch (MQClientException e) {
 58:                         result = -1;
 59:                     }
 60:                 }
 61:             } else {
 62:                 result = -1;
 63:             }
 64:             break;
 65:         }
 66: 
 67:         default:
 68:             break;
 69:     }
 70: 
 71:     return result;
 72: }
```

* 说明 ：计算消息队列开始消费位置。
* `PushConsumer` 读取消费进度有三种选项：
    * `CONSUME_FROM_LAST_OFFSET` ：第 6 至 29 行 ：一个新的消费集群第一次启动从**队列的最后位置**开始消费。**后续再启动接着上次消费的进度开始消费**。
    * `CONSUME_FROM_FIRST_OFFSET` ：第 30 至 40 行 ：一个新的消费集群第一次启动从队列的**最前位置**开始消费。**后续再启动接着上次消费的进度开始消费**。
    * `CONSUME_FROM_TIMESTAMP` ：第 41 至 65 行 ：一个新的消费集群第一次启动从**指定时间点**开始消费。**后续再启动接着上次消费的进度开始消费**。


## `[PullConsumer]` RebalancePullImpl#computePullFromWhere(...)

暂时跳过。😈

# 6、PushConsumer 拉取消息

![DefaultMQPushConsumerImpl拉取消息](http://www.yunai.me/images/RocketMQ/2017_05_04/05.png)

## PullMessageService

```Java
  1: public class PullMessageService extends ServiceThread {
  2:     private final Logger log = ClientLogger.getLog();
  3:     /**
  4:      * 拉取消息请求队列
  5:      */
  6:     private final LinkedBlockingQueue<PullRequest> pullRequestQueue = new LinkedBlockingQueue<>();
  7:     /**
  8:      * MQClient对象
  9:      */
 10:     private final MQClientInstance mQClientFactory;
 11:     /**
 12:      * 定时器。用于延迟提交拉取请求
 13:      */
 14:     private final ScheduledExecutorService scheduledExecutorService = Executors
 15:         .newSingleThreadScheduledExecutor(new ThreadFactory() {
 16:             @Override
 17:             public Thread newThread(Runnable r) {
 18:                 return new Thread(r, "PullMessageServiceScheduledThread");
 19:             }
 20:         });
 21: 
 22:     public PullMessageService(MQClientInstance mQClientFactory) {
 23:         this.mQClientFactory = mQClientFactory;
 24:     }
 25: 
 26:     /**
 27:      * 执行延迟拉取消息请求
 28:      *
 29:      * @param pullRequest 拉取消息请求
 30:      * @param timeDelay 延迟时长
 31:      */
 32:     public void executePullRequestLater(final PullRequest pullRequest, final long timeDelay) {
 33:         this.scheduledExecutorService.schedule(new Runnable() {
 34: 
 35:             @Override
 36:             public void run() {
 37:                 PullMessageService.this.executePullRequestImmediately(pullRequest);
 38:             }
 39:         }, timeDelay, TimeUnit.MILLISECONDS);
 40:     }
 41: 
 42:     /**
 43:      * 执行立即拉取消息请求
 44:      *
 45:      * @param pullRequest 拉取消息请求
 46:      */
 47:     public void executePullRequestImmediately(final PullRequest pullRequest) {
 48:         try {
 49:             this.pullRequestQueue.put(pullRequest);
 50:         } catch (InterruptedException e) {
 51:             log.error("executePullRequestImmediately pullRequestQueue.put", e);
 52:         }
 53:     }
 54: 
 55:     /**
 56:      * 执行延迟任务
 57:      *
 58:      * @param r 任务
 59:      * @param timeDelay 延迟时长
 60:      */
 61:     public void executeTaskLater(final Runnable r, final long timeDelay) {
 62:         this.scheduledExecutorService.schedule(r, timeDelay, TimeUnit.MILLISECONDS);
 63:     }
 64: 
 65:     public ScheduledExecutorService getScheduledExecutorService() {
 66:         return scheduledExecutorService;
 67:     }
 68: 
 69:     /**
 70:      * 拉取消息
 71:      *
 72:      * @param pullRequest 拉取消息请求
 73:      */
 74:     private void pullMessage(final PullRequest pullRequest) {
 75:         final MQConsumerInner consumer = this.mQClientFactory.selectConsumer(pullRequest.getConsumerGroup());
 76:         if (consumer != null) {
 77:             DefaultMQPushConsumerImpl impl = (DefaultMQPushConsumerImpl) consumer;
 78:             impl.pullMessage(pullRequest);
 79:         } else {
 80:             log.warn("No matched consumer for the PullRequest {}, drop it", pullRequest);
 81:         }
 82:     }
 83: 
 84:     @Override
 85:     public void run() {
 86:         log.info(this.getServiceName() + " service started");
 87: 
 88:         while (!this.isStopped()) {
 89:             try {
 90:                 PullRequest pullRequest = this.pullRequestQueue.take();
 91:                 if (pullRequest != null) {
 92:                     this.pullMessage(pullRequest);
 93:                 }
 94:             } catch (InterruptedException e) {
 95:             } catch (Exception e) {
 96:                 log.error("Pull Message Service Run Method exception", e);
 97:             }
 98:         }
 99: 
100:         log.info(this.getServiceName() + " service end");
101:     }
102: 
103:     @Override
104:     public String getServiceName() {
105:         return PullMessageService.class.getSimpleName();
106:     }
107: 
108: }
```

* 说明 ：拉取消息服务，不断不断不断从 `Broker` 拉取消息，并提交消费任务到 `ConsumeMessageService`。
* `#executePullRequestLater(...)` ：第 26 至 40 行 ： 提交**延迟**拉取消息请求。
* `#executePullRequestImmediately(...)` ：第 42 至 53 行 ：提交**立即**拉取消息请求。
* `#executeTaskLater(...)` ：第 55 至 63 行 ：提交**延迟任务**。
* `#pullMessage(...)` ：第 69 至 82 行 ：执行拉取消息逻辑。详细解析见：[DefaultMQPushConsumerImpl#pullMessage(...)](#defaultmqpushconsumerimplpullmessage)。
* `#run(...)` ：第 84 至 101 行 ：循环拉取消息请求队列( `pullRequestQueue` )，进行消息拉取。

## DefaultMQPushConsumerImpl#pullMessage(...)

```Java
  1: public void pullMessage(final PullRequest pullRequest) {
  2:     final ProcessQueue processQueue = pullRequest.getProcessQueue();
  3:     if (processQueue.isDropped()) {
  4:         log.info("the pull request[{}] is dropped.", pullRequest.toString());
  5:         return;
  6:     }
  7: 
  8:     // 设置队列最后拉取消息时间
  9:     pullRequest.getProcessQueue().setLastPullTimestamp(System.currentTimeMillis());
 10: 
 11:     // 判断consumer状态是否运行中。如果不是，则延迟拉取消息。
 12:     try {
 13:         this.makeSureStateOK();
 14:     } catch (MQClientException e) {
 15:         log.warn("pullMessage exception, consumer state not ok", e);
 16:         this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION);
 17:         return;
 18:     }
 19: 
 20:     // 判断是否暂停中。
 21:     if (this.isPause()) {
 22:         log.warn("consumer was paused, execute pull request later. instanceName={}, group={}", this.defaultMQPushConsumer.getInstanceName(), this.defaultMQPushConsumer.getConsumerGroup());
 23:         this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_SUSPEND);
 24:         return;
 25:     }
 26: 
 27:     // 判断是否超过最大持有消息数量。默认最大值为1000。
 28:     long size = processQueue.getMsgCount().get();
 29:     if (size > this.defaultMQPushConsumer.getPullThresholdForQueue()) {
 30:         this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL); // 提交延迟消息拉取请求。50ms。
 31:         if ((flowControlTimes1++ % 1000) == 0) {
 32:             log.warn(
 33:                 "the consumer message buffer is full, so do flow control, minOffset={}, maxOffset={}, size={}, pullRequest={}, flowControlTimes={}",
 34:                 processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), size, pullRequest, flowControlTimes1);
 35:         }
 36:         return;
 37:     }
 38: 
 39:     if (!this.consumeOrderly) { // 判断消息跨度是否过大。
 40:         if (processQueue.getMaxSpan() > this.defaultMQPushConsumer.getConsumeConcurrentlyMaxSpan()) {
 41:             this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL); // 提交延迟消息拉取请求。50ms。
 42:             if ((flowControlTimes2++ % 1000) == 0) {
 43:                 log.warn(
 44:                     "the queue's messages, span too long, so do flow control, minOffset={}, maxOffset={}, maxSpan={}, pullRequest={}, flowControlTimes={}",
 45:                     processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), processQueue.getMaxSpan(),
 46:                     pullRequest, flowControlTimes2);
 47:             }
 48:             return;
 49:         }
 50:     } else { // TODO 顺序消费
 51:         if (processQueue.isLocked()) {
 52:             if (!pullRequest.isLockedFirst()) {
 53:                 final long offset = this.rebalanceImpl.computePullFromWhere(pullRequest.getMessageQueue());
 54:                 boolean brokerBusy = offset < pullRequest.getNextOffset();
 55:                 log.info("the first time to pull message, so fix offset from broker. pullRequest: {} NewOffset: {} brokerBusy: {}",
 56:                     pullRequest, offset, brokerBusy);
 57:                 if (brokerBusy) {
 58:                     log.info("[NOTIFYME]the first time to pull message, but pull request offset larger than broker consume offset. pullRequest: {} NewOffset: {}",
 59:                         pullRequest, offset);
 60:                 }
 61: 
 62:                 pullRequest.setLockedFirst(true);
 63:                 pullRequest.setNextOffset(offset);
 64:             }
 65:         } else {
 66:             this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION);
 67:             log.info("pull message later because not locked in broker, {}", pullRequest);
 68:             return;
 69:         }
 70:     }
 71: 
 72:     // 获取Topic 对应的订阅信息。若不存在，则延迟拉取消息
 73:     final SubscriptionData subscriptionData = this.rebalanceImpl.getSubscriptionInner().get(pullRequest.getMessageQueue().getTopic());
 74:     if (null == subscriptionData) {
 75:         this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION);
 76:         log.warn("find the consumer's subscription failed, {}", pullRequest);
 77:         return;
 78:     }
 79: 
 80:     final long beginTimestamp = System.currentTimeMillis();
 81: 
 82:     PullCallback pullCallback = new PullCallback() {
 83:         @Override
 84:         public void onSuccess(PullResult pullResult) {
 85:             if (pullResult != null) {
 86:                 pullResult = DefaultMQPushConsumerImpl.this.pullAPIWrapper.processPullResult(pullRequest.getMessageQueue(), pullResult,
 87:                     subscriptionData);
 88: 
 89:                 switch (pullResult.getPullStatus()) {
 90:                     case FOUND:
 91:                         // 设置下次拉取消息队列位置
 92:                         long prevRequestOffset = pullRequest.getNextOffset();
 93:                         pullRequest.setNextOffset(pullResult.getNextBeginOffset());
 94: 
 95:                         // 统计
 96:                         long pullRT = System.currentTimeMillis() - beginTimestamp;
 97:                         DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullRT(pullRequest.getConsumerGroup(),
 98:                             pullRequest.getMessageQueue().getTopic(), pullRT);
 99: 
100:                         long firstMsgOffset = Long.MAX_VALUE;
101:                         if (pullResult.getMsgFoundList() == null || pullResult.getMsgFoundList().isEmpty()) {
102:                             DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
103:                         } else {
104:                             firstMsgOffset = pullResult.getMsgFoundList().get(0).getQueueOffset();
105: 
106:                             // 统计
107:                             DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullTPS(pullRequest.getConsumerGroup(),
108:                                 pullRequest.getMessageQueue().getTopic(), pullResult.getMsgFoundList().size());
109: 
110:                             // 提交拉取到的消息到消息处理队列
111:                             boolean dispathToConsume = processQueue.putMessage(pullResult.getMsgFoundList());
112: 
113:                             // 提交消费请求
114:                             DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(//
115:                                 pullResult.getMsgFoundList(), //
116:                                 processQueue, //
117:                                 pullRequest.getMessageQueue(), //
118:                                 dispathToConsume);
119: 
120:                             // 提交下次拉取消息请求
121:                             if (DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval() > 0) {
122:                                 DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest,
123:                                     DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval());
124:                             } else {
125:                                 DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
126:                             }
127:                         }
128: 
129:                         // 下次拉取消息队列位置小于上次拉取消息队列位置 或者 第一条消息的消息队列位置小于上次拉取消息队列位置，则判定为BUG，输出log
130:                         if (pullResult.getNextBeginOffset() < prevRequestOffset//
131:                             || firstMsgOffset < prevRequestOffset) {
132:                             log.warn(
133:                                 "[BUG] pull message result maybe data wrong, nextBeginOffset: {} firstMsgOffset: {} prevRequestOffset: {}", //
134:                                 pullResult.getNextBeginOffset(), //
135:                                 firstMsgOffset, //
136:                                 prevRequestOffset);
137:                         }
138: 
139:                         break;
140:                     case NO_NEW_MSG:
141:                         // 设置下次拉取消息队列位置
142:                         pullRequest.setNextOffset(pullResult.getNextBeginOffset());
143: 
144:                         // 持久化消费进度
145:                         DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);
146: 
147:                         // 立即提交拉取消息请求
148:                         DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
149:                         break;
150:                     case NO_MATCHED_MSG:
151:                         // 设置下次拉取消息队列位置
152:                         pullRequest.setNextOffset(pullResult.getNextBeginOffset());
153: 
154:                         // 持久化消费进度
155:                         DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);
156: 
157:                         // 提交立即拉取消息请求
158:                         DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
159:                         break;
160:                     case OFFSET_ILLEGAL:
161:                         log.warn("the pull request offset illegal, {} {}", //
162:                             pullRequest.toString(), pullResult.toString());
163:                         // 设置下次拉取消息队列位置
164:                         pullRequest.setNextOffset(pullResult.getNextBeginOffset());
165: 
166:                         // 设置消息处理队列为dropped
167:                         pullRequest.getProcessQueue().setDropped(true);
168: 
169:                         // 提交延迟任务，进行消费处理队列移除。不立即移除的原因：可能有地方正在使用，避免受到影响。
170:                         DefaultMQPushConsumerImpl.this.executeTaskLater(new Runnable() {
171: 
172:                             @Override
173:                             public void run() {
174:                                 try {
175:                                     // 更新消费进度，同步消费进度到Broker
176:                                     DefaultMQPushConsumerImpl.this.offsetStore.updateOffset(pullRequest.getMessageQueue(),
177:                                         pullRequest.getNextOffset(), false);
178:                                     DefaultMQPushConsumerImpl.this.offsetStore.persist(pullRequest.getMessageQueue());
179: 
180:                                     // 移除消费处理队列
181:                                     DefaultMQPushConsumerImpl.this.rebalanceImpl.removeProcessQueue(pullRequest.getMessageQueue());
182: 
183:                                     log.warn("fix the pull request offset, {}", pullRequest);
184:                                 } catch (Throwable e) {
185:                                     log.error("executeTaskLater Exception", e);
186:                                 }
187:                             }
188:                         }, 10000);
189:                         break;
190:                     default:
191:                         break;
192:                 }
193:             }
194:         }
195: 
196:         @Override
197:         public void onException(Throwable e) {
198:             if (!pullRequest.getMessageQueue().getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
199:                 log.warn("execute the pull request exception", e);
200:             }
201: 
202:             // 提交延迟拉取消息请求
203:             DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION);
204:         }
205:     };
206: 
207:     // 集群消息模型下，计算提交的消费进度。
208:     boolean commitOffsetEnable = false;
209:     long commitOffsetValue = 0L;
210:     if (MessageModel.CLUSTERING == this.defaultMQPushConsumer.getMessageModel()) {
211:         commitOffsetValue = this.offsetStore.readOffset(pullRequest.getMessageQueue(), ReadOffsetType.READ_FROM_MEMORY);
212:         if (commitOffsetValue > 0) {
213:             commitOffsetEnable = true;
214:         }
215:     }
216: 
217:     // 计算请求的 订阅表达式 和 是否进行filtersrv过滤消息
218:     String subExpression = null;
219:     boolean classFilter = false;
220:     SubscriptionData sd = this.rebalanceImpl.getSubscriptionInner().get(pullRequest.getMessageQueue().getTopic());
221:     if (sd != null) {
222:         if (this.defaultMQPushConsumer.isPostSubscriptionWhenPull() && !sd.isClassFilterMode()) {
223:             subExpression = sd.getSubString();
224:         }
225: 
226:         classFilter = sd.isClassFilterMode();
227:     }
228: 
229:     // 计算拉取消息系统标识
230:     int sysFlag = PullSysFlag.buildSysFlag(//
231:         commitOffsetEnable, // commitOffset
232:         true, // suspend
233:         subExpression != null, // subscription
234:         classFilter // class filter
235:     );
236: 
237:     // 执行拉取。如果拉取请求发生异常时，提交延迟拉取消息请求。
238:     try {
239:         this.pullAPIWrapper.pullKernelImpl(//
240:             pullRequest.getMessageQueue(), // 1
241:             subExpression, // 2
242:             subscriptionData.getSubVersion(), // 3
243:             pullRequest.getNextOffset(), // 4
244:             this.defaultMQPushConsumer.getPullBatchSize(), // 5
245:             sysFlag, // 6
246:             commitOffsetValue, // 7
247:             BROKER_SUSPEND_MAX_TIME_MILLIS, // 8
248:             CONSUMER_TIMEOUT_MILLIS_WHEN_SUSPEND, // 9
249:             CommunicationMode.ASYNC, // 10
250:             pullCallback// 11
251:         );
252:     } catch (Exception e) {
253:         log.error("pullKernelImpl exception", e);
254:         this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION);
255:     }
256: }
257: 
258: private void correctTagsOffset(final PullRequest pullRequest) {
259:     if (0L == pullRequest.getProcessQueue().getMsgCount().get()) {
260:         this.offsetStore.updateOffset(pullRequest.getMessageQueue(), pullRequest.getNextOffset(), true);
261:     }
262: }
```

* `#pullMessage(...)` 说明 ：拉取消息。
    * 第 3 至 6 行 ：消息处理队列已经终止，不进行消息拉取。
    * 第 9 行 ：设置消息处理队列最后拉取消息时间。
    * 第 11 至 18 行 ：`Consumer` 未处于运行中状态，不进行消息拉取，提交**延迟**拉取消息请求。
    * 第 20 至 25 行 ： `Consumer` 处于暂停中，不进行消息拉取，提交**延迟**拉取消息请求。
    * 第 27 至 37 行 ：消息处理队列持有消息超过最大允许值（默认：1000条），不进行消息拉取，提交**延迟**拉取消息请求。
    * 第 39 至 49 行 ：`Consumer` 为**并发消费** 并且 消息队列持有消息跨度过大（消息跨度 = 持有消息最后一条和第一条的消息位置差，默认：2000），不进行消息拉取，提交**延迟**拉取消息请求。
    * 第 50 至 70 行 ：`顺序消费` 相关跳过，详细解析见：[《RocketMQ 源码分析 —— Message 顺序发送与消费》](http://www.yunai.me/RocketMQ/message-send-and-consume-orderly/)。
    * 第 72 至 78 行 ：`Topic` 对应的订阅信息不存在，不进行消息拉取，提交**延迟**拉取消息请求。
    * 第 222 至 224 行 ：判断请求是否使用 `Consumer` **本地**的订阅信息( `SubscriptionData` )，而不使用 `Broker` 里的订阅信息。详细解析见：[PullMessageProcessor#processRequest(...) 第 64 至 110 行代码](http://www.yunai.me/RocketMQ/message-pull-and-consume-first/#PullMessageProcessor-processRequest-…)。
    * 第 226 行 ：是否开启过滤类过滤模式。详细解析见：[《RocketMQ 源码分析 —— Filtersrv》](http://www.yunai.me/RocketMQ/filtersrv/)。
    * 第 229 至 235 行 ：计算拉取消息请求系统标识。详细解析见：[PullMessageRequestHeader.sysFlag](http://www.yunai.me/RocketMQ/message-pull-and-consume-first/#PullMessageRequestHeader)。
    * 第 237 至 255 行 ：
        * 执行消息拉取**异步**请求。详细解析见：[PullAPIWrapper#pullKernelImpl(...)](#pullapiwrapperpullkernelimpl)。
        * 当发起请求产生异常时，提交**延迟**拉取消息请求。对应 `Broker` 处理拉取消息逻辑见：[PullMessageProcessor#processRequest(...)](http://www.yunai.me/RocketMQ/message-pull-and-consume-first/#PullMessageProcessor-processRequest-…)。
* `PullCallback` ：拉取消息回调：
   * 第 86 行 ：处理拉取结果。详细逻辑见：[PullAPIWrapper#processPullResult(...)](#pullapiwrapperprocesspullresult)。
   * 第 89 至 192 行 ：处理拉取状态结果：
        * 第 90 至 139 行 ：拉取到消息( `FOUND` ) ：
            * 第 91 至 93 行 ：设置下次拉取消息队列位置。
            * 第 95 至 97 行 ：统计。
            * 第 101 至 102 行 ：拉取到消息的消息列表为空，提交**立即**拉取消息请求。为什么会存在拉取到消息，但是消息结果未空呢？原因见：[PullAPIWrapper#processPullResult(...)](#pullapiwrapperprocesspullresult)。
            * 第 106 至 108 行 ：统计。
            * 第 111 行 ：提交拉取到的消息到消息处理队列。详细解析见：[ProcessQueue#putMessage(...)](#processqueueputmessage)。
            * 第 113 至 118 行 ：提交消费请求到 `ConsumeMessageService`。详细解析见：[ConsumeMessageConcurrentlyService](#consumemessageconcurrentlyservice)。
            * 第 120 至 126 行 ：根据拉取频率( `pullInterval` )，提交**立即或者延迟**拉取消息请求。默认拉取频率为 0ms ，提交**立即**拉取消息请求。
            * 第 129 至 137 行 ：下次拉取消息队列位置小于上次拉取消息队列位置 或者 第一条消息的消息队列位置小于上次拉取消息队列位置，则判定为**BUG**，输出警告日志。
       * 第 140 至 149 行 ：没有新消息( `NO_NEW_MSG` ) ：
            * 第 142 行 ： 设置下次拉取消息队列位置。
            * 第 145 行 ：更正消费进度。详细解析见：`#correctTagsOffset(...)`。
            * 第 148 行 ：提交**立即**拉取消息请求。
       * 第 150 至 159 行 ：有新消息但是不匹配( `NO_MATCHED_MSG` )。逻辑同 `NO_NEW_MSG` 。
       * 第 160 至 189 行 ：拉取请求的消息队列位置不合法 (`OFFSET_ILLEGAL`)。
            * 第 164 行 ：设置下次拉取消息队列位置。
            * 第 167 行 ：设置消息处理队列为 `dropped`。
            * 第 169 至 188 行 ：提交延迟任务，进行队列移除。
                * 第 175 至 178 行 ：更新消费进度，同步消费进度到 `Broker`。
                * 第 181 行 ：移除消费处理队列。
                    * 疑问：为什么不立即移除？？？ 
  * 第 196 至 204 行 ：发生异常，提交**延迟**拉取消息请求。
* `#correctTagsOffset(...)` ：更正消费进度。
    * 第 258 至 261 行 ： 当消费处理队列持有消息数量为 **0** 时，更新消费进度为拉取请求的拉取消息队列位置。

### PullAPIWrapper#pullKernelImpl(...)

```Java
  1: /**
  2:  * 拉取消息核心方法
  3:  *
  4:  * @param mq 消息队列
  5:  * @param subExpression 订阅表达式
  6:  * @param subVersion 订阅版本号
  7:  * @param offset 拉取队列开始位置
  8:  * @param maxNums 拉取消息数量
  9:  * @param sysFlag 拉取请求系统标识
 10:  * @param commitOffset 提交消费进度
 11:  * @param brokerSuspendMaxTimeMillis broker挂起请求最大时间
 12:  * @param timeoutMillis 请求broker超时时长
 13:  * @param communicationMode 通讯模式
 14:  * @param pullCallback 拉取回调
 15:  * @return 拉取消息结果。只有通讯模式为同步时，才返回结果，否则返回null。
 16:  * @throws MQClientException 当寻找不到 broker 时，或发生其他client异常
 17:  * @throws RemotingException 当远程调用发生异常时
 18:  * @throws MQBrokerException 当 broker 发生异常时。只有通讯模式为同步时才会发生该异常。
 19:  * @throws InterruptedException 当发生中断异常时
 20:  */
 21: protected PullResult pullKernelImpl(
 22:     final MessageQueue mq,
 23:     final String subExpression,
 24:     final long subVersion,
 25:     final long offset,
 26:     final int maxNums,
 27:     final int sysFlag,
 28:     final long commitOffset,
 29:     final long brokerSuspendMaxTimeMillis,
 30:     final long timeoutMillis,
 31:     final CommunicationMode communicationMode,
 32:     final PullCallback pullCallback
 33: ) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
 34:     // 获取Broker信息
 35:     FindBrokerResult findBrokerResult =
 36:         this.mQClientFactory.findBrokerAddressInSubscribe(mq.getBrokerName(),
 37:             this.recalculatePullFromWhichNode(mq), false);
 38:     if (null == findBrokerResult) {
 39:         this.mQClientFactory.updateTopicRouteInfoFromNameServer(mq.getTopic());
 40:         findBrokerResult =
 41:             this.mQClientFactory.findBrokerAddressInSubscribe(mq.getBrokerName(),
 42:                 this.recalculatePullFromWhichNode(mq), false);
 43:     }
 44: 
 45:     // 请求拉取消息
 46:     if (findBrokerResult != null) {
 47:         int sysFlagInner = sysFlag;
 48: 
 49:         if (findBrokerResult.isSlave()) {
 50:             sysFlagInner = PullSysFlag.clearCommitOffsetFlag(sysFlagInner);
 51:         }
 52: 
 53:         PullMessageRequestHeader requestHeader = new PullMessageRequestHeader();
 54:         requestHeader.setConsumerGroup(this.consumerGroup);
 55:         requestHeader.setTopic(mq.getTopic());
 56:         requestHeader.setQueueId(mq.getQueueId());
 57:         requestHeader.setQueueOffset(offset);
 58:         requestHeader.setMaxMsgNums(maxNums);
 59:         requestHeader.setSysFlag(sysFlagInner);
 60:         requestHeader.setCommitOffset(commitOffset);
 61:         requestHeader.setSuspendTimeoutMillis(brokerSuspendMaxTimeMillis);
 62:         requestHeader.setSubscription(subExpression);
 63:         requestHeader.setSubVersion(subVersion);
 64: 
 65:         String brokerAddr = findBrokerResult.getBrokerAddr();
 66:         if (PullSysFlag.hasClassFilterFlag(sysFlagInner)) { // TODO filtersrv
 67:             brokerAddr = computPullFromWhichFilterServer(mq.getTopic(), brokerAddr);
 68:         }
 69: 
 70:         PullResult pullResult = this.mQClientFactory.getMQClientAPIImpl().pullMessage(
 71:             brokerAddr,
 72:             requestHeader,
 73:             timeoutMillis,
 74:             communicationMode,
 75:             pullCallback);
 76: 
 77:         return pullResult;
 78:     }
 79: 
 80:     // Broker信息不存在，则抛出异常
 81:     throw new MQClientException("The broker[" + mq.getBrokerName() + "] not exist", null);
 82: }
```

* 说明 ：拉取消息核心方法。**该方法参数较多，可以看下代码注释上每个参数的说明**😈。
* 第 34 至 43 行 ：获取 `Broker` 信息(`Broker` 地址、是否为从节点)。
    * [#recalculatePullFromWhichNode(...)](#pullapiwrapperrecalculatepullfromwhichnode)
    * [#MQClientInstance#findBrokerAddressInSubscribe(...)](#mqclientinstancefindbrokeraddressinsubscribe)
* 第 45 至 78 行 ：**请求拉取消息**。
* 第 81 行 ：当 `Broker` 信息不存在，则抛出异常。

#### PullAPIWrapper#recalculatePullFromWhichNode(...)

```Java
  1: /**
  2:  * 消息队列 与 拉取Broker 的映射
  3:  * 当拉取消息时，会通过该映射获取拉取请求对应的Broker
  4:  */
  5: private ConcurrentHashMap<MessageQueue, AtomicLong/* brokerId */> pullFromWhichNodeTable =
  6:     new ConcurrentHashMap<MessageQueue, AtomicLong>(32);
  7: /**
  8:  * 是否使用默认Broker
  9:  */
 10: private volatile boolean connectBrokerByUser = false;
 11: /**
 12:  * 默认Broker编号
 13:  */
 14: private volatile long defaultBrokerId = MixAll.MASTER_ID;
 15: 
 16: /**
 17:  * 计算消息队列拉取消息对应的Broker编号
 18:  *
 19:  * @param mq 消息队列
 20:  * @return Broker编号
 21:  */
 22: public long recalculatePullFromWhichNode(final MessageQueue mq) {
 23:     // 若开启默认Broker开关，则返回默认Broker编号
 24:     if (this.isConnectBrokerByUser()) {
 25:         return this.defaultBrokerId;
 26:     }
 27: 
 28:     // 若消息队列映射拉取Broker存在，则返回映射Broker编号
 29:     AtomicLong suggest = this.pullFromWhichNodeTable.get(mq);
 30:     if (suggest != null) {
 31:         return suggest.get();
 32:     }
 33: 
 34:     // 返回Broker主节点编号
 35:     return MixAll.MASTER_ID;
 36: }
```

* 说明 ：计算消息队列拉取消息对应的 `Broker` 编号。

#### MQClientInstance#findBrokerAddressInSubscribe(...)

```Java
  1: /**
  2:  * Broker名字 和 Broker地址相关 Map
  3:  */
  4: private final ConcurrentHashMap<String/* Broker Name */, HashMap<Long/* brokerId */, String/* address */>> brokerAddrTable =
  5:         new ConcurrentHashMap<>();
  6: 
  7: /**
  8:  * 获得Broker信息
  9:  *
 10:  * @param brokerName broker名字
 11:  * @param brokerId broker编号
 12:  * @param onlyThisBroker 是否必须是该broker
 13:  * @return Broker信息
 14:  */
 15: public FindBrokerResult findBrokerAddressInSubscribe(//
 16:     final String brokerName, //
 17:     final long brokerId, //
 18:     final boolean onlyThisBroker//
 19: ) {
 20:     String brokerAddr = null; // broker地址
 21:     boolean slave = false; // 是否为从节点
 22:     boolean found = false; // 是否找到
 23: 
 24:     // 获得Broker信息
 25:     HashMap<Long/* brokerId */, String/* address */> map = this.brokerAddrTable.get(brokerName);
 26:     if (map != null && !map.isEmpty()) {
 27:         brokerAddr = map.get(brokerId);
 28:         slave = brokerId != MixAll.MASTER_ID;
 29:         found = brokerAddr != null;
 30: 
 31:         // 如果不强制获得，选择一个Broker
 32:         if (!found && !onlyThisBroker) {
 33:             Entry<Long, String> entry = map.entrySet().iterator().next();
 34:             brokerAddr = entry.getValue();
 35:             slave = entry.getKey() != MixAll.MASTER_ID;
 36:             found = true;
 37:         }
 38:     }
 39: 
 40:     // 找到broker，则返回信息
 41:     if (found) {
 42:         return new FindBrokerResult(brokerAddr, slave);
 43:     }
 44: 
 45:     // 找不到，则返回空
 46:     return null;
 47: }
```

* 说明 ：获取 `Broker` 信息(`Broker` 地址、是否为从节点)。

### PullAPIWrapper#processPullResult(...)

```Java
  1: /**
  2:  * 处理拉取结果
  3:  * 1. 更新消息队列拉取消息Broker编号的映射
  4:  * 2. 解析消息，并根据订阅信息消息tagCode匹配合适消息
  5:  *
  6:  * @param mq 消息队列
  7:  * @param pullResult 拉取结果
  8:  * @param subscriptionData 订阅信息
  9:  * @return 拉取结果
 10:  */
 11: public PullResult processPullResult(final MessageQueue mq, final PullResult pullResult,
 12:     final SubscriptionData subscriptionData) {
 13:     PullResultExt pullResultExt = (PullResultExt) pullResult;
 14: 
 15:     // 更新消息队列拉取消息Broker编号的映射
 16:     this.updatePullFromWhichNode(mq, pullResultExt.getSuggestWhichBrokerId());
 17: 
 18:     // 解析消息，并根据订阅信息消息tagCode匹配合适消息
 19:     if (PullStatus.FOUND == pullResult.getPullStatus()) {
 20:         // 解析消息
 21:         ByteBuffer byteBuffer = ByteBuffer.wrap(pullResultExt.getMessageBinary());
 22:         List<MessageExt> msgList = MessageDecoder.decodes(byteBuffer);
 23: 
 24:         // 根据订阅信息消息tagCode匹配合适消息
 25:         List<MessageExt> msgListFilterAgain = msgList;
 26:         if (!subscriptionData.getTagsSet().isEmpty() && !subscriptionData.isClassFilterMode()) {
 27:             msgListFilterAgain = new ArrayList<>(msgList.size());
 28:             for (MessageExt msg : msgList) {
 29:                 if (msg.getTags() != null) {
 30:                     if (subscriptionData.getTagsSet().contains(msg.getTags())) {
 31:                         msgListFilterAgain.add(msg);
 32:                     }
 33:                 }
 34:             }
 35:         }
 36: 
 37:         // Hook
 38:         if (this.hasHook()) {
 39:             FilterMessageContext filterMessageContext = new FilterMessageContext();
 40:             filterMessageContext.setUnitMode(unitMode);
 41:             filterMessageContext.setMsgList(msgListFilterAgain);
 42:             this.executeHook(filterMessageContext);
 43:         }
 44: 
 45:         // 设置消息队列当前最小/最大位置到消息拓展字段
 46:         for (MessageExt msg : msgListFilterAgain) {
 47:             MessageAccessor.putProperty(msg, MessageConst.PROPERTY_MIN_OFFSET,
 48:                 Long.toString(pullResult.getMinOffset()));
 49:             MessageAccessor.putProperty(msg, MessageConst.PROPERTY_MAX_OFFSET,
 50:                 Long.toString(pullResult.getMaxOffset()));
 51:         }
 52: 
 53:         // 设置消息列表
 54:         pullResultExt.setMsgFoundList(msgListFilterAgain);
 55:     }
 56: 
 57:     // 清空消息二进制数组
 58:     pullResultExt.setMessageBinary(null);
 59: 
 60:     return pullResult;
 61: }
```

* 说明 ：处理拉取结果。
    *  更新消息队列拉取消息 `Broker` 编号的映射。
    *  解析消息，并根据订阅信息消息 `tagCode `匹配合适消息。
* 第 16 行 ：更新消息队列拉取消息 `Broker` 编号的映射。下次拉取消息时，如果未设置默认拉取的 `Broker` 编号，会使用更新后的 `Broker` 编号。
* 第 18 至 55 行 ：解析消息，并根据订阅信息消息 `tagCode` 匹配合适消息。
    * 第 20 至 22 行 ：解析消息。详细解析见：[《RocketMQ 源码分析 —— Message基础》](http://www.yunai.me/RocketMQ/message/) 。
    * 第 24 至 35 行 ：根据订阅信息`tagCode` 匹配消息。
    * 第 37 至 43 行 ：`Hook`。
    * 第 45 至 51 行 ：设置消息队列当前最小/最大位置到消息拓展字段。
    * 第 54 行 ：设置消息队列。
* 第 58 行 ：清空消息二进制数组。

### ProcessQueue#putMessage(...)

```Java
  1:  /**
  2:  * 消息映射读写锁
  3:  */
  4: private final ReadWriteLock lockTreeMap = new ReentrantReadWriteLock();
  5: /**
  6:  * 消息映射
  7:  * key：消息队列位置
  8:  */
  9: private final TreeMap<Long, MessageExt> msgTreeMap = new TreeMap<>();
 10: /**
 11:  * 消息数
 12:  */
 13: private final AtomicLong msgCount = new AtomicLong();
 14: /**
 15:  * 添加消息最大队列位置
 16:  */
 17: private volatile long queueOffsetMax = 0L;
 18: /**
 19:  * 是否正在消费
 20:  */
 21: private volatile boolean consuming = false;
 22: /**
 23:  * Broker累计消息数量
 24:  * 计算公式 = queueMaxOffset - 新添加消息数组[n - 1].queueOffset
 25:  * Acc = Accumulation
 26:  * cnt = （猜测）对比度
 27:  */
 28: private volatile long msgAccCnt = 0;
 29: 
 30: /**
 31:  * 添加消息，并返回是否提交给消费者
 32:  * 返回true，当有新消息添加成功时，
 33:  *
 34:  * @param msgs 消息
 35:  * @return 是否提交给消费者
 36:  */
 37: public boolean putMessage(final List<MessageExt> msgs) {
 38:     boolean dispatchToConsume = false;
 39:     try {
 40:         this.lockTreeMap.writeLock().lockInterruptibly();
 41:         try {
 42:             // 添加消息
 43:             int validMsgCnt = 0;
 44:             for (MessageExt msg : msgs) {
 45:                 MessageExt old = msgTreeMap.put(msg.getQueueOffset(), msg);
 46:                 if (null == old) {
 47:                     validMsgCnt++;
 48:                     this.queueOffsetMax = msg.getQueueOffset();
 49:                 }
 50:             }
 51:             msgCount.addAndGet(validMsgCnt);
 52: 
 53:             // 计算是否正在消费
 54:             if (!msgTreeMap.isEmpty() && !this.consuming) {
 55:                 dispatchToConsume = true;
 56:                 this.consuming = true;
 57:             }
 58: 
 59:             // Broker累计消息数量
 60:             if (!msgs.isEmpty()) {
 61:                 MessageExt messageExt = msgs.get(msgs.size() - 1);
 62:                 String property = messageExt.getProperty(MessageConst.PROPERTY_MAX_OFFSET);
 63:                 if (property != null) {
 64:                     long accTotal = Long.parseLong(property) - messageExt.getQueueOffset();
 65:                     if (accTotal > 0) {
 66:                         this.msgAccCnt = accTotal;
 67:                     }
 68:                 }
 69:             }
 70:         } finally {
 71:             this.lockTreeMap.writeLock().unlock();
 72:         }
 73:     } catch (InterruptedException e) {
 74:         log.error("putMessage exception", e);
 75:     }
 76: 
 77:     return dispatchToConsume;
 78: }
```

## 总结

如果用最简单粗暴的方式描述 `PullConsumer` 拉取消息的过程，那就是如下的代码：

```Java
while (true) {
    if (不满足拉取消息) {
        Thread.sleep(间隔);
        continue;
    }
    主动拉取消息();
}
```

# 6、PushConsumer 消费消息

![DefaultMQPushConsumerImpl消费消息](http://www.yunai.me/images/RocketMQ/2017_05_04/06.png)

## ConsumeMessageConcurrentlyService 提交消费请求

### ConsumeMessageConcurrentlyService#submitConsumeRequest(...)

```Java
  1: /**
  2:  * 消费线程池队列
  3:  */
  4: private final BlockingQueue<Runnable> consumeRequestQueue;
  5: /**
  6:  * 消费线程池
  7:  */
  8: private final ThreadPoolExecutor consumeExecutor;
  9: 
 10: public void submitConsumeRequest(//
 11:     final List<MessageExt> msgs, //
 12:     final ProcessQueue processQueue, //
 13:     final MessageQueue messageQueue, //
 14:     final boolean dispatchToConsume) {
 15:     final int consumeBatchSize = this.defaultMQPushConsumer.getConsumeMessageBatchMaxSize();
 16:     if (msgs.size() <= consumeBatchSize) { // 提交消息小于批量消息数，直接提交消费请求
 17:         ConsumeRequest consumeRequest = new ConsumeRequest(msgs, processQueue, messageQueue);
 18:         try {
 19:             this.consumeExecutor.submit(consumeRequest);
 20:         } catch (RejectedExecutionException e) {
 21:             this.submitConsumeRequestLater(consumeRequest);
 22:         }
 23:     } else { // 提交消息大于批量消息数，进行分拆成多个消费请求
 24:         for (int total = 0; total < msgs.size(); ) {
 25:             // 计算当前拆分请求包含的消息
 26:             List<MessageExt> msgThis = new ArrayList<>(consumeBatchSize);
 27:             for (int i = 0; i < consumeBatchSize; i++, total++) {
 28:                 if (total < msgs.size()) {
 29:                     msgThis.add(msgs.get(total));
 30:                 } else {
 31:                     break;
 32:                 }
 33:             }
 34: 
 35:             // 提交拆分消费请求
 36:             ConsumeRequest consumeRequest = new ConsumeRequest(msgThis, processQueue, messageQueue);
 37:             try {
 38:                 this.consumeExecutor.submit(consumeRequest);
 39:             } catch (RejectedExecutionException e) {
 40:                 // 如果被拒绝，则将当前拆分消息+剩余消息提交延迟消费请求。
 41:                 for (; total < msgs.size(); total++) {
 42:                     msgThis.add(msgs.get(total));
 43:                 }
 44:                 this.submitConsumeRequestLater(consumeRequest);
 45:             }
 46:         }
 47:     }
 48: }
```

* 说明 ：提交**立即**消费请求。
* 第 16 至 22 行 ：提交消息小于等于批量消费数，直接提交消费请求。
* 第 23 至 47 行 ：当提交消息大于批量消费数，进行分拆成多个请求。
    * 第 25 至 33 行 ：计算当前拆分请求包含的消息。
    * 第 35 至 38 行 ：提交拆分消费请求。
    * 第 39 至 44 行 ：提交请求被拒绝，则将当前拆分消息 + 剩余消息提交延迟消费请求，结束拆分循环。

### ConsumeMessageConcurrentlyService#submitConsumeRequestLater
 
 ```Java
   1: /**
  2:  * 提交延迟消费请求
  3:  *
  4:  * @param msgs 消息列表
  5:  * @param processQueue 消息处理队列
  6:  * @param messageQueue 消息队列
  7:  */
  8: private void submitConsumeRequestLater(//
  9:     final List<MessageExt> msgs, //
 10:     final ProcessQueue processQueue, //
 11:     final MessageQueue messageQueue//
 12: ) {
 13: 
 14:     this.scheduledExecutorService.schedule(new Runnable() {
 15: 
 16:         @Override
 17:         public void run() {
 18:             ConsumeMessageConcurrentlyService.this.submitConsumeRequest(msgs, processQueue, messageQueue, true);
 19:         }
 20:     }, 5000, TimeUnit.MILLISECONDS);
 21: }
 22: 
 23: /**
 24:  * 提交延迟消费请求
 25:  * @param consumeRequest 消费请求
 26:  */
 27: private void submitConsumeRequestLater(final ConsumeRequest consumeRequest//
 28: ) {
 29: 
 30:     this.scheduledExecutorService.schedule(new Runnable() {
 31: 
 32:         @Override
 33:         public void run() {
 34:             ConsumeMessageConcurrentlyService.this.consumeExecutor.submit(consumeRequest); // TODO BUG ?
 35:         }
 36:     }, 5000, TimeUnit.MILLISECONDS);
 37: }
 ```
 
* 说明 ：提交延迟消费请求。
* 第 34 行 ：直接调用 `ConsumeMessageConcurrentlyService.this.consumeExecutor.submit(consumeRequest);`。如果消息数超过批量消费上限，会不会是**BUG**。
 
## ConsumeRequest

```Java
  1: class ConsumeRequest implements Runnable {
  2: 
  3:     /**
  4:      * 消费消息列表
  5:      */
  6:     private final List<MessageExt> msgs;
  7:     /**
  8:      * 消息处理队列
  9:      */
 10:     private final ProcessQueue processQueue;
 11:     /**
 12:      * 消息队列
 13:      */
 14:     private final MessageQueue messageQueue;
 15: 
 16:     public ConsumeRequest(List<MessageExt> msgs, ProcessQueue processQueue, MessageQueue messageQueue) {
 17:         this.msgs = msgs;
 18:         this.processQueue = processQueue;
 19:         this.messageQueue = messageQueue;
 20:     }
 21: 
 22:     @Override
 23:     public void run() {
 24:         // 废弃队列不进行消费
 25:         if (this.processQueue.isDropped()) {
 26:             log.info("the message queue not be able to consume, because it's dropped. group={} {}", ConsumeMessageConcurrentlyService.this.consumerGroup, this.messageQueue);
 27:             return;
 28:         }
 29: 
 30:         MessageListenerConcurrently listener = ConsumeMessageConcurrentlyService.this.messageListener; // 监听器
 31:         ConsumeConcurrentlyContext context = new ConsumeConcurrentlyContext(messageQueue); // 消费Context
 32:         ConsumeConcurrentlyStatus status = null; // 消费结果状态
 33: 
 34:         // Hook
 35:         ConsumeMessageContext consumeMessageContext = null;
 36:         if (ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.hasHook()) {
 37:             consumeMessageContext = new ConsumeMessageContext();
 38:             consumeMessageContext.setConsumerGroup(defaultMQPushConsumer.getConsumerGroup());
 39:             consumeMessageContext.setProps(new HashMap<String, String>());
 40:             consumeMessageContext.setMq(messageQueue);
 41:             consumeMessageContext.setMsgList(msgs);
 42:             consumeMessageContext.setSuccess(false);
 43:             ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.executeHookBefore(consumeMessageContext);
 44:         }
 45: 
 46:         long beginTimestamp = System.currentTimeMillis();
 47:         boolean hasException = false;
 48:         ConsumeReturnType returnType = ConsumeReturnType.SUCCESS; // 消费返回结果类型
 49:         try {
 50:             // 当消息为重试消息，设置Topic为原始Topic
 51:             ConsumeMessageConcurrentlyService.this.resetRetryTopic(msgs);
 52: 
 53:             // 设置开始消费时间
 54:             if (msgs != null && !msgs.isEmpty()) {
 55:                 for (MessageExt msg : msgs) {
 56:                     MessageAccessor.setConsumeStartTimeStamp(msg, String.valueOf(System.currentTimeMillis()));
 57:                 }
 58:             }
 59: 
 60:             // 进行消费
 61:             status = listener.consumeMessage(Collections.unmodifiableList(msgs), context);
 62:         } catch (Throwable e) {
 63:             log.warn("consumeMessage exception: {} Group: {} Msgs: {} MQ: {}",
 64:                 RemotingHelper.exceptionSimpleDesc(e), //
 65:                 ConsumeMessageConcurrentlyService.this.consumerGroup,
 66:                 msgs,
 67:                 messageQueue);
 68:             hasException = true;
 69:         }
 70: 
 71:         // 解析消费返回结果类型
 72:         long consumeRT = System.currentTimeMillis() - beginTimestamp;
 73:         if (null == status) {
 74:             if (hasException) {
 75:                 returnType = ConsumeReturnType.EXCEPTION;
 76:             } else {
 77:                 returnType = ConsumeReturnType.RETURNNULL;
 78:             }
 79:         } else if (consumeRT >= defaultMQPushConsumer.getConsumeTimeout() * 60 * 1000) {
 80:             returnType = ConsumeReturnType.TIME_OUT;
 81:         } else if (ConsumeConcurrentlyStatus.RECONSUME_LATER == status) {
 82:             returnType = ConsumeReturnType.FAILED;
 83:         } else if (ConsumeConcurrentlyStatus.CONSUME_SUCCESS == status) {
 84:             returnType = ConsumeReturnType.SUCCESS;
 85:         }
 86: 
 87:         // Hook
 88:         if (ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.hasHook()) {
 89:             consumeMessageContext.getProps().put(MixAll.CONSUME_CONTEXT_TYPE, returnType.name());
 90:         }
 91: 
 92:         // 消费结果状态为空时，则设置为稍后重新消费
 93:         if (null == status) {
 94:             log.warn("consumeMessage return null, Group: {} Msgs: {} MQ: {}",
 95:                 ConsumeMessageConcurrentlyService.this.consumerGroup,
 96:                 msgs,
 97:                 messageQueue);
 98:             status = ConsumeConcurrentlyStatus.RECONSUME_LATER;
 99:         }
100: 
101:         // Hook
102:         if (ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.hasHook()) {
103:             consumeMessageContext.setStatus(status.toString());
104:             consumeMessageContext.setSuccess(ConsumeConcurrentlyStatus.CONSUME_SUCCESS == status);
105:             ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.executeHookAfter(consumeMessageContext);
106:         }
107: 
108:         // 统计
109:         ConsumeMessageConcurrentlyService.this.getConsumerStatsManager()
110:             .incConsumeRT(ConsumeMessageConcurrentlyService.this.consumerGroup, messageQueue.getTopic(), consumeRT);
111: 
112:         // 处理消费结果
113:         if (!processQueue.isDropped()) {
114:             ConsumeMessageConcurrentlyService.this.processConsumeResult(status, context, this);
115:         } else {
116:             log.warn("processQueue is dropped without process consume result. messageQueue={}, msgs={}", messageQueue, msgs);
117:         }
118:     }
119: 
120: }
```

* 说明 ：消费请求。提交请求执行消费。
* 第 24 至 28 行 ：废弃处理队列不进行消费。
* 第 34 至 44 行 ：Hook。
* 第 51 行 ：当消息为重试消息，设置 `Topic`为原始 `Topic`。例如：原始 `Topic` 为 `TopicTest`，重试时 `Topic` 为 `%RETRY%please_rename_unique_group_name_4`，经过该方法，`Topic` 设置回 `TopicTest`。
* 第 53 至 58 行 ：设置开始消费时间。
* 第 61 行 ：**进行消费**。
* 第 71 至 85 行 ：解析消费返回结果类型
* 第 87 至 90 行 ：`Hook`。
* 第 92 至 99 行 ：消费结果状态未空时，则设置消费结果状态为稍后消费。
* 第 101 至 106 行 ：`Hook`。
* 第 108 至 110 行 ：统计。
* 第 112 至 117 行 ：处理消费结果。**如果消费处理队列被移除，恰好消息被消费，则可能导致消息重复消费，因此，消息消费要尽最大可能性实现幂等性**。详细解析见：[ConsumeMessageConcurrentlyService#processConsumeResult(...)](#consumemessageconcurrentlyserviceprocessconsumeresult)。

## ConsumeMessageConcurrentlyService#processConsumeResult(...)

```Java
  1: public void processConsumeResult(//
  2:     final ConsumeConcurrentlyStatus status, //
  3:     final ConsumeConcurrentlyContext context, //
  4:     final ConsumeRequest consumeRequest//
  5: ) {
  6:     int ackIndex = context.getAckIndex();
  7: 
  8:     // 消息为空，直接返回
  9:     if (consumeRequest.getMsgs().isEmpty())
 10:         return;
 11: 
 12:     // 计算从consumeRequest.msgs[0]到consumeRequest.msgs[ackIndex]的消息消费成功
 13:     switch (status) {
 14:         case CONSUME_SUCCESS:
 15:             if (ackIndex >= consumeRequest.getMsgs().size()) {
 16:                 ackIndex = consumeRequest.getMsgs().size() - 1;
 17:             }
 18:             // 统计成功/失败数量
 19:             int ok = ackIndex + 1;
 20:             int failed = consumeRequest.getMsgs().size() - ok;
 21:             this.getConsumerStatsManager().incConsumeOKTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), ok);
 22:             this.getConsumerStatsManager().incConsumeFailedTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), failed);
 23:             break;
 24:         case RECONSUME_LATER:
 25:             ackIndex = -1;
 26:             // 统计成功/失败数量
 27:             this.getConsumerStatsManager().incConsumeFailedTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(),
 28:                 consumeRequest.getMsgs().size());
 29:             break;
 30:         default:
 31:             break;
 32:     }
 33: 
 34:     // 处理消费失败的消息
 35:     switch (this.defaultMQPushConsumer.getMessageModel()) {
 36:         case BROADCASTING: // 广播模式，无论是否消费失败，不发回消息到Broker，只打印Log
 37:             for (int i = ackIndex + 1; i < consumeRequest.getMsgs().size(); i++) {
 38:                 MessageExt msg = consumeRequest.getMsgs().get(i);
 39:                 log.warn("BROADCASTING, the message consume failed, drop it, {}", msg.toString());
 40:             }
 41:             break;
 42:         case CLUSTERING:
 43:             // 发回消息失败到Broker。
 44:             List<MessageExt> msgBackFailed = new ArrayList<>(consumeRequest.getMsgs().size());
 45:             for (int i = ackIndex + 1; i < consumeRequest.getMsgs().size(); i++) {
 46:                 MessageExt msg = consumeRequest.getMsgs().get(i);
 47:                 boolean result = this.sendMessageBack(msg, context);
 48:                 if (!result) {
 49:                     msg.setReconsumeTimes(msg.getReconsumeTimes() + 1);
 50:                     msgBackFailed.add(msg);
 51:                 }
 52:             }
 53: 
 54:             // 发回Broker失败的消息，直接提交延迟重新消费
 55:             if (!msgBackFailed.isEmpty()) {
 56:                 consumeRequest.getMsgs().removeAll(msgBackFailed);
 57: 
 58:                 this.submitConsumeRequestLater(msgBackFailed, consumeRequest.getProcessQueue(), consumeRequest.getMessageQueue());
 59:             }
 60:             break;
 61:         default:
 62:             break;
 63:     }
 64: 
 65:     // 移除消费成功消息，并更新最新消费进度
 66:     long offset = consumeRequest.getProcessQueue().removeMessage(consumeRequest.getMsgs());
 67:     if (offset >= 0 && !consumeRequest.getProcessQueue().isDropped()) {
 68:         this.defaultMQPushConsumerImpl.getOffsetStore().updateOffset(consumeRequest.getMessageQueue(), offset, true);
 69:     }
 70: }
```

* 说明 ：处理消费结果。
* 第 8 至 10 行 ：消费请求消息未空时，直接返回。
* 第 12 至 32 行 ：计算 `ackIndex` 值。`consumeRequest.msgs[0 - ackIndex]`为消费成功，需要进行 `ack` 确认。
    * 第 14 至 23 行 ：`CONSUME_SUCCESS` ：`ackIndex = context.getAckIndex()`。
    * 第 24 至 29 行 ：`RECONSUME_LATER` ：`ackIndex = -1`。
* 第34 至 63 行 ：处理消费失败的消息。
    * 第 36 至 41 行 ：`BROADCASTING` ：广播模式，无论是否消费失败，不发回消息到 `Broker`，只打印日志。
    * 第 42 至 60 行 ：`CLUSTERING` ：集群模式，消费失败的消息发回到 `Broker`。
        * 第 43 至 52 行 ：发回消费失败的消息到 `Broker`。详细解析见：[DefaultMQPushConsumerImpl#sendMessageBack(...)](#defaultmqpushconsumerimplsendmessageback)。
        * 第 54 至 59 行 ：发回 `Broker` 失败的消息，直接提交延迟重新消费。
        * **如果发回 `Broker` 成功，结果因为例如网络异常，导致 `Consumer`以为发回失败，判定消费发回失败，会导致消息重复消费，因此，消息消费要尽最大可能性实现幂等性。**
* 第 65 至 69 行 ：移除**【消费成功】**和**【消费失败但发回`Broker`成功】**的消息，并更新最新消费进度。
    * 为什么会有**【消费失败但发回`Broker`成功】**的消息？见**第 56 行**。
    * [ProcessQueue#removeMessage(...)](#processqueueremovemessage)

### ProcessQueue#removeMessage(...)

```Java
  1: /**
  2:  * 移除消息，并返回第一条消息队列位置
  3:  *
  4:  * @param msgs 消息
  5:  * @return 消息队列位置
  6:  */
  7: public long removeMessage(final List<MessageExt> msgs) {
  8:     long result = -1;
  9:     final long now = System.currentTimeMillis();
 10:     try {
 11:         this.lockTreeMap.writeLock().lockInterruptibly();
 12:         this.lastConsumeTimestamp = now;
 13:         try {
 14:             if (!msgTreeMap.isEmpty()) {
 15:                 result = this.queueOffsetMax + 1; // 这里+1的原因是：如果msgTreeMap为空时，下一条获得的消息位置为queueOffsetMax+1
 16: 
 17:                 // 移除消息
 18:                 int removedCnt = 0;
 19:                 for (MessageExt msg : msgs) {
 20:                     MessageExt prev = msgTreeMap.remove(msg.getQueueOffset());
 21:                     if (prev != null) {
 22:                         removedCnt--;
 23:                     }
 24:                 }
 25:                 msgCount.addAndGet(removedCnt);
 26: 
 27:                 if (!msgTreeMap.isEmpty()) {
 28:                     result = msgTreeMap.firstKey();
 29:                 }
 30:             }
 31:         } finally {
 32:             this.lockTreeMap.writeLock().unlock();
 33:         }
 34:     } catch (Throwable t) {
 35:         log.error("removeMessage exception", t);
 36:     }
 37: 
 38:     return result;
 39: }
```

## ConsumeMessageConcurrentlyService#cleanExpireMsg(...)

```Java
  1: public void start() {
  2:     this.cleanExpireMsgExecutors.scheduleAtFixedRate(new Runnable() {
  3: 
  4:         @Override
  5:         public void run() {
  6:             cleanExpireMsg();
  7:         }
  8: 
  9:     }, this.defaultMQPushConsumer.getConsumeTimeout(), this.defaultMQPushConsumer.getConsumeTimeout(), TimeUnit.MINUTES);
 10: }
 11: 
 12: /**
 13:  * 清理过期消息
 14:  */
 15: private void cleanExpireMsg() {
 16:     Iterator<Map.Entry<MessageQueue, ProcessQueue>> it =
 17:         this.defaultMQPushConsumerImpl.getRebalanceImpl().getProcessQueueTable().entrySet().iterator();
 18:     while (it.hasNext()) {
 19:         Map.Entry<MessageQueue, ProcessQueue> next = it.next();
 20:         ProcessQueue pq = next.getValue();
 21:         pq.cleanExpiredMsg(this.defaultMQPushConsumer);
 22:     }
 23: }
```

* 说明 ：定时清理过期消息，默认周期：15min。

### ProcessQueue#cleanExpiredMsg(...)

```Java
  1: public void cleanExpiredMsg(DefaultMQPushConsumer pushConsumer) {
  2:     // 顺序消费时，直接返回
  3:     if (pushConsumer.getDefaultMQPushConsumerImpl().isConsumeOrderly()) {
  4:         return;
  5:     }
  6: 
  7:     // 循环移除消息
  8:     int loop = msgTreeMap.size() < 16 ? msgTreeMap.size() : 16; // 每次循环最多移除16条
  9:     for (int i = 0; i < loop; i++) {
 10:         // 获取第一条消息。判断是否超时，若不超时，则结束循环
 11:         MessageExt msg = null;
 12:         try {
 13:             this.lockTreeMap.readLock().lockInterruptibly();
 14:             try {
 15:                 if (!msgTreeMap.isEmpty() && System.currentTimeMillis() - Long.parseLong(MessageAccessor.getConsumeStartTimeStamp(msgTreeMap.firstEntry().getValue())) > pushConsumer.getConsumeTimeout() * 60 * 1000) {
 16:                     msg = msgTreeMap.firstEntry().getValue();
 17:                 } else {
 18:                     break;
 19:                 }
 20:             } finally {
 21:                 this.lockTreeMap.readLock().unlock();
 22:             }
 23:         } catch (InterruptedException e) {
 24:             log.error("getExpiredMsg exception", e);
 25:         }
 26: 
 27:         try {
 28:             // 发回超时消息
 29:             pushConsumer.sendMessageBack(msg, 3);
 30:             log.info("send expire msg back. topic={}, msgId={}, storeHost={}, queueId={}, queueOffset={}", msg.getTopic(), msg.getMsgId(), msg.getStoreHost(), msg.getQueueId(), msg.getQueueOffset());
 31: 
 32:             // 判断此时消息是否依然是第一条，若是，则进行移除
 33:             try {
 34:                 this.lockTreeMap.writeLock().lockInterruptibly();
 35:                 try {
 36:                     if (!msgTreeMap.isEmpty() && msg.getQueueOffset() == msgTreeMap.firstKey()) {
 37:                         try {
 38:                             msgTreeMap.remove(msgTreeMap.firstKey());
 39:                         } catch (Exception e) {
 40:                             log.error("send expired msg exception", e);
 41:                         }
 42:                     }
 43:                 } finally {
 44:                     this.lockTreeMap.writeLock().unlock();
 45:                 }
 46:             } catch (InterruptedException e) {
 47:                 log.error("getExpiredMsg exception", e);
 48:             }
 49:         } catch (Exception e) {
 50:             log.error("send expired msg exception", e);
 51:         }
 52:     }
 53: }
```

* 说明 ：移除过期消息。
* 第 2 至 5 行 ：顺序消费时，直接返回。
* 第 7 至 9 行 ：循环移除消息。默认最大循环次数：16次。
* 第 10 至 25 行 ：获取第一条消息。判断是否超时，若不超时，则结束循环。
* 第 29 行 ：**发回超时消息到`Broker`**。
* 第 32 至 48 行 ：判断此时消息是否依然是第一条，若是，则进行移除。

# 7、PushConsumer 发回消费失败消息

## DefaultMQPushConsumerImpl#sendMessageBack(...)

```Java
  1: public void sendMessageBack(MessageExt msg, int delayLevel, final String brokerName)
  2:     throws RemotingException, MQBrokerException, InterruptedException, MQClientException {
  3:     try {
  4:         // Consumer发回消息
  5:         String brokerAddr = (null != brokerName) ? this.mQClientFactory.findBrokerAddressInPublish(brokerName)
  6:             : RemotingHelper.parseSocketAddressAddr(msg.getStoreHost());
  7:         this.mQClientFactory.getMQClientAPIImpl().consumerSendMessageBack(brokerAddr, msg,
  8:             this.defaultMQPushConsumer.getConsumerGroup(), delayLevel, 5000, getMaxReconsumeTimes());
  9:     } catch (Exception e) { // TODO 疑问：什么情况下会发生异常
 10:         // 异常时，使用Client内置Producer发回消息
 11:         log.error("sendMessageBack Exception, " + this.defaultMQPushConsumer.getConsumerGroup(), e);
 12: 
 13:         Message newMsg = new Message(MixAll.getRetryTopic(this.defaultMQPushConsumer.getConsumerGroup()), msg.getBody());
 14: 
 15:         String originMsgId = MessageAccessor.getOriginMessageId(msg);
 16:         MessageAccessor.setOriginMessageId(newMsg, UtilAll.isBlank(originMsgId) ? msg.getMsgId() : originMsgId);
 17: 
 18:         newMsg.setFlag(msg.getFlag());
 19:         MessageAccessor.setProperties(newMsg, msg.getProperties());
 20:         MessageAccessor.putProperty(newMsg, MessageConst.PROPERTY_RETRY_TOPIC, msg.getTopic());
 21:         MessageAccessor.setReconsumeTime(newMsg, String.valueOf(msg.getReconsumeTimes() + 1));
 22:         MessageAccessor.setMaxReconsumeTimes(newMsg, String.valueOf(getMaxReconsumeTimes()));
 23:         newMsg.setDelayTimeLevel(3 + msg.getReconsumeTimes());
 24: 
 25:         this.mQClientFactory.getDefaultMQProducer().send(newMsg);
 26:     }
 27: }
```

* 说明 ：发回消息。
* 第 4 至 8 行 ：`Consumer` 发回消息。详细解析见：[MQClientAPIImpl#consumerSendMessageBack(...)](#mqclientapiimplconsumersendmessageback)。
* 第 10 至 25 行 ：发生异常时，`Consumer` 内置默认 `Producer` 发送消息。
    * 😈疑问：什么样的情况下会发生异常呢？

### MQClientAPIImpl#consumerSendMessageBack(...)

```Java
  1: /**
  2:  * Consumer发回消息
  3:  * @param addr Broker地址
  4:  * @param msg 消息
  5:  * @param consumerGroup 消费分组
  6:  * @param delayLevel 延迟级别
  7:  * @param timeoutMillis 超时
  8:  * @param maxConsumeRetryTimes 消费最大重试次数
  9:  * @throws RemotingException 当远程调用发生异常时
 10:  * @throws MQBrokerException 当Broker发生异常时
 11:  * @throws InterruptedException 当线程中断时
 12:  */
 13: public void consumerSendMessageBack(
 14:     final String addr,
 15:     final MessageExt msg,
 16:     final String consumerGroup,
 17:     final int delayLevel,
 18:     final long timeoutMillis,
 19:     final int maxConsumeRetryTimes
 20: ) throws RemotingException, MQBrokerException, InterruptedException {
 21:     ConsumerSendMsgBackRequestHeader requestHeader = new ConsumerSendMsgBackRequestHeader();
 22:     RemotingCommand request = RemotingCommand.createRequestCommand(RequestCode.CONSUMER_SEND_MSG_BACK, requestHeader);
 23: 
 24:     requestHeader.setGroup(consumerGroup);
 25:     requestHeader.setOriginTopic(msg.getTopic());
 26:     requestHeader.setOffset(msg.getCommitLogOffset());
 27:     requestHeader.setDelayLevel(delayLevel);
 28:     requestHeader.setOriginMsgId(msg.getMsgId());
 29:     requestHeader.setMaxReconsumeTimes(maxConsumeRetryTimes);
 30: 
 31:     RemotingCommand response = this.remotingClient.invokeSync(MixAll.brokerVIPChannel(this.clientConfig.isVipChannelEnabled(), addr),
 32:         request, timeoutMillis);
 33:     assert response != null;
 34:     switch (response.getCode()) {
 35:         case ResponseCode.SUCCESS: {
 36:             return;
 37:         }
 38:         default:
 39:             break;
 40:     }
 41: 
 42:     throw new MQBrokerException(response.getCode(), response.getRemark());
 43: }
```

# 8、Consumer 消费进度

## OffsetStore

![OffsetStore类图.png](http://www.yunai.me/images/RocketMQ/2017_05_04/07.png)

* `RemoteBrokerOffsetStore` ：`Consumer` **集群模式** 下，使用远程 `Broker` 消费进度。
* `LocalFileOffsetStore` ：`Consumer` **广播模式**下，使用本地 `文件` 消费进度。

### OffsetStore#load(...)

#### LocalFileOffsetStore#load(...)

```Java
  1: @Override
  2: public void load() throws MQClientException {
  3:     // 从本地硬盘读取消费进度
  4:     OffsetSerializeWrapper offsetSerializeWrapper = this.readLocalOffset();
  5:     if (offsetSerializeWrapper != null && offsetSerializeWrapper.getOffsetTable() != null) {
  6:         offsetTable.putAll(offsetSerializeWrapper.getOffsetTable());
  7: 
  8:         // 打印每个消息队列的消费进度
  9:         for (MessageQueue mq : offsetSerializeWrapper.getOffsetTable().keySet()) {
 10:             AtomicLong offset = offsetSerializeWrapper.getOffsetTable().get(mq);
 11:             log.info("load consumer's offset, {} {} {}",
 12:                 this.groupName,
 13:                 mq,
 14:                 offset.get());
 15:         }
 16:     }
 17: }
```

* 说明 ：从本地文件加载消费进度到内存。

##### OffsetSerializeWrapper

```Java
  1: public class OffsetSerializeWrapper extends RemotingSerializable {
  2:     private ConcurrentHashMap<MessageQueue, AtomicLong> offsetTable =
  3:             new ConcurrentHashMap<>();
  4: 
  5:     public ConcurrentHashMap<MessageQueue, AtomicLong> getOffsetTable() {
  6:         return offsetTable;
  7:     }
  8: 
  9:     public void setOffsetTable(ConcurrentHashMap<MessageQueue, AtomicLong> offsetTable) {
 10:         this.offsetTable = offsetTable;
 11:     }
 12: }
```

* 说明 ：本地 `Offset` 存储序列化。

```Bash
Yunai-MacdeMacBook-Pro-2:config yunai$ cat /Users/yunai/.rocketmq_offsets/192.168.17.0@DEFAULT/please_rename_unique_group_name_1/offsets.json
{
	"offsetTable":{{
			"brokerName":"broker-a",
			"queueId":3,
			"topic":"TopicTest"
		}:1470,{
			"brokerName":"broker-a",
			"queueId":2,
			"topic":"TopicTest"
		}:1471,{
			"brokerName":"broker-a",
			"queueId":1,
			"topic":"TopicTest"
		}:1470,{
			"brokerName":"broker-a",
			"queueId":0,
			"topic":"TopicTest"
		}:1470
	}
}
```

#### RemoteBrokerOffsetStore#load(...)

```Java
  1: @Override
  2: public void load() {
  3: }
```

* 说明 ：不进行加载，实际读取消费进度时，从 `Broker` 获取。

### OffsetStore#readOffset(...)

读取消费进度类型：

* `READ_FROM_MEMORY` ：从内存读取。
* `READ_FROM_STORE` ：从存储( `Broker` 或 `文件` )读取。
* `MEMORY_FIRST_THEN_STORE` ：优先从内存读取，读取不到，从存储读取。

#### LocalFileOffsetStore#readOffset(...)

```Java
  1: @Override
  2: public long readOffset(final MessageQueue mq, final ReadOffsetType type) {
  3:     if (mq != null) {
  4:         switch (type) {
  5:             case MEMORY_FIRST_THEN_STORE:
  6:             case READ_FROM_MEMORY: {
  7:                 AtomicLong offset = this.offsetTable.get(mq);
  8:                 if (offset != null) {
  9:                     return offset.get();
 10:                 } else if (ReadOffsetType.READ_FROM_MEMORY == type) {
 11:                     return -1;
 12:                 }
 13:             }
 14:             case READ_FROM_STORE: {
 15:                 OffsetSerializeWrapper offsetSerializeWrapper;
 16:                 try {
 17:                     offsetSerializeWrapper = this.readLocalOffset();
 18:                 } catch (MQClientException e) {
 19:                     return -1;
 20:                 }
 21:                 if (offsetSerializeWrapper != null && offsetSerializeWrapper.getOffsetTable() != null) {
 22:                     AtomicLong offset = offsetSerializeWrapper.getOffsetTable().get(mq);
 23:                     if (offset != null) {
 24:                         this.updateOffset(mq, offset.get(), false);
 25:                         return offset.get();
 26:                     }
 27:                 }
 28:             }
 29:             default:
 30:                 break;
 31:         }
 32:     }
 33: 
 34:     return -1;
 35: }
```

* 第 16 行 ：从 `文件` 读取消费进度。

#### RemoteBrokerOffsetStore#readOffset(...)

```Java
  1: @Override
  2: public long readOffset(final MessageQueue mq, final ReadOffsetType type) {
  3:     if (mq != null) {
  4:         switch (type) {
  5:             case MEMORY_FIRST_THEN_STORE:
  6:             case READ_FROM_MEMORY: {
  7:                 AtomicLong offset = this.offsetTable.get(mq);
  8:                 if (offset != null) {
  9:                     return offset.get();
 10:                 } else if (ReadOffsetType.READ_FROM_MEMORY == type) {
 11:                     return -1;
 12:                 }
 13:             }
 14:             case READ_FROM_STORE: {
 15:                 try {
 16:                     long brokerOffset = this.fetchConsumeOffsetFromBroker(mq);
 17:                     AtomicLong offset = new AtomicLong(brokerOffset);
 18:                     this.updateOffset(mq, offset.get(), false);
 19:                     return brokerOffset;
 20:                 }
 21:                 // No offset in broker
 22:                 catch (MQBrokerException e) {
 23:                     return -1;
 24:                 }
 25:                 //Other exceptions
 26:                 catch (Exception e) {
 27:                     log.warn("fetchConsumeOffsetFromBroker exception, " + mq, e);
 28:                     return -2;
 29:                 }
 30:             }
 31:             default:
 32:                 break;
 33:         }
 34:     }
 35: 
 36:     return -1;
 37: }
```

* 第 16 行 ：从 `Broker` 读取消费进度。

### OffsetStore#updateOffset(...)

该方法 `RemoteBrokerOffsetStore` 与 `LocalFileOffsetStore` 实现相同。

```Java
  1: @Override
  2: public void updateOffset(MessageQueue mq, long offset, boolean increaseOnly) {
  3:     if (mq != null) {
  4:         AtomicLong offsetOld = this.offsetTable.get(mq);
  5:         if (null == offsetOld) {
  6:             offsetOld = this.offsetTable.putIfAbsent(mq, new AtomicLong(offset));
  7:         }
  8: 
  9:         if (null != offsetOld) {
 10:             if (increaseOnly) {
 11:                 MixAll.compareAndIncreaseOnly(offsetOld, offset);
 12:             } else {
 13:                 offsetOld.set(offset);
 14:             }
 15:         }
 16:     }
 17: }
```

### OffsetStore#persistAll(...)

#### LocalFileOffsetStore#persistAll(...)

```Java
  1: @Override
  2: public void persistAll(Set<MessageQueue> mqs) {
  3:     if (null == mqs || mqs.isEmpty())
  4:         return;
  5: 
  6:     OffsetSerializeWrapper offsetSerializeWrapper = new OffsetSerializeWrapper();
  7:     for (Map.Entry<MessageQueue, AtomicLong> entry : this.offsetTable.entrySet()) {
  8:         if (mqs.contains(entry.getKey())) {
  9:             AtomicLong offset = entry.getValue();
 10:             offsetSerializeWrapper.getOffsetTable().put(entry.getKey(), offset);
 11:         }
 12:     }
 13: 
 14:     String jsonString = offsetSerializeWrapper.toJson(true);
 15:     if (jsonString != null) {
 16:         try {
 17:             MixAll.string2File(jsonString, this.storePath);
 18:         } catch (IOException e) {
 19:             log.error("persistAll consumer offset Exception, " + this.storePath, e);
 20:         }
 21:     }
 22: }
```

* 说明 ：持久化消费进度。**将消费进度写入文件**。

#### RemoteBrokerOffsetStore#persistAll(...)

```Java
  1: @Override
  2: public void persistAll(Set<MessageQueue> mqs) {
  3:     if (null == mqs || mqs.isEmpty())
  4:         return;
  5: 
  6:     // 持久化消息队列
  7:     final HashSet<MessageQueue> unusedMQ = new HashSet<>();
  8:     if (!mqs.isEmpty()) {
  9:         for (Map.Entry<MessageQueue, AtomicLong> entry : this.offsetTable.entrySet()) {
 10:             MessageQueue mq = entry.getKey();
 11:             AtomicLong offset = entry.getValue();
 12:             if (offset != null) {
 13:                 if (mqs.contains(mq)) {
 14:                     try {
 15:                         this.updateConsumeOffsetToBroker(mq, offset.get());
 16:                         log.info("[persistAll] Group: {} ClientId: {} updateConsumeOffsetToBroker {} {}",
 17:                             this.groupName,
 18:                             this.mQClientFactory.getClientId(),
 19:                             mq,
 20:                             offset.get());
 21:                     } catch (Exception e) {
 22:                         log.error("updateConsumeOffsetToBroker exception, " + mq.toString(), e);
 23:                     }
 24:                 } else {
 25:                     unusedMQ.add(mq);
 26:                 }
 27:             }
 28:         }
 29:     }
 30: 
 31:     // 移除不适用的消息队列
 32:     if (!unusedMQ.isEmpty()) {
 33:         for (MessageQueue mq : unusedMQ) {
 34:             this.offsetTable.remove(mq);
 35:             log.info("remove unused mq, {}, {}", mq, this.groupName);
 36:         }
 37:     }
 38: }
```

* 说明 ：持久化指定消息队列数组的消费进度到 `Broker`，并移除非指定消息队列。

#### MQClientInstance#persistAllConsumerOffset(...)

```Java
  1: private void startScheduledTask() {
  2:     // 定时同步消费进度
  3:     this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
  4: 
  5:         @Override
  6:         public void run() {
  7:             try {
  8:                 MQClientInstance.this.cleanOfflineBroker();
  9:                 MQClientInstance.this.sendHeartbeatToAllBrokerWithLock();
 10:             } catch (Exception e) {
 11:                 log.error("ScheduledTask sendHeartbeatToAllBroker exception", e);
 12:             }
 13:         }
 14:     }, 1000, this.clientConfig.getHeartbeatBrokerInterval(), TimeUnit.MILLISECONDS);
 15: }
```

* 说明 ：定时进行持久化，默认周期：5000ms。
* **重要说明 ：**
    * **消费进度持久化不仅仅只有定时持久化，拉取消息、分配消息队列等等操作，都会进行消费进度持久化。** 
    * **消费进度持久化不仅仅只有定时持久化，拉取消息、分配消息队列等等操作，都会进行消费进度持久化。** 
    * **消费进度持久化不仅仅只有定时持久化，拉取消息、分配消息队列等等操作，都会进行消费进度持久化。** 

# 9、结尾

😈可能是本系列最长的一篇文章，如有表达错误和不清晰，请多多见谅。  
感谢对本系列的阅读、收藏、点赞、分享，特别是翻到结尾。😜真的有丢丢长。


