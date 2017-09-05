title: RocketMQ 源码分析 —— Message 发送与接收
date: 2017-04-18
tags:
categories: RocketMQ
permalink: RocketMQ/message-send-and-receive

-------

![](http://www.yunai.me/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

- [1、概述](#)
- [2、Producer 发送消息](#)
	- [DefaultMQProducer#send(Message)](#)
	- [DefaultMQProducerImpl#sendDefaultImpl()](#)
		- [DefaultMQProducerImpl#tryToFindTopicPublishInfo()](#)
		- [MQFaultStrategy](#)
			- [MQFaultStrategy](#)
			- [LatencyFaultTolerance](#)
			- [LatencyFaultToleranceImpl](#)
			- [FaultItem](#)
		- [DefaultMQProducerImpl#sendKernelImpl()](#)
- [3、Broker 接收消息](#)
	- [SendMessageProcessor#sendMessage](#)
		- [AbstractSendMessageProcessor#msgCheck](#)
	- [DefaultMessageStore#putMessage](#)
- [4、某种结尾](#)

# 1、概述

1. `Producer` 发送消息。主要是**同步**发送消息源码，涉及到 异步/Oneway发送消息，事务消息会跳过。
2. `Broker` 接收消息。(*存储消息在[《RocketMQ 源码分析 —— Message 存储》](http://www.yunai.me/RocketMQ/message-store/)解析*)

> ![Producer发送消息全局顺序图](http://www.yunai.me/images/RocketMQ/2017_04_18/01.png)

# 2、Producer 发送消息

 > ![Producer发送消息顺序图](http://www.yunai.me/images/RocketMQ/2017_04_18/02.png)

## DefaultMQProducer#send(Message)

```Java
  1: @Override
  2: public SendResult send(Message msg) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
  3:     return this.defaultMQProducerImpl.send(msg);
  4: }
```

* 说明：发送同步消息，`DefaultMQProducer#send(Message)` 对 `DefaultMQProducerImpl#send(Message)` 进行封装。  

## DefaultMQProducerImpl#sendDefaultImpl()

```Java
  1: public SendResult send(Message msg) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
  2:     return send(msg, this.defaultMQProducer.getSendMsgTimeout());
  3: }
  4: 
  5: public SendResult send(Message msg, long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
  6:     return this.sendDefaultImpl(msg, CommunicationMode.SYNC, null, timeout);
  7: }
  8: 
  9: private SendResult sendDefaultImpl(//
 10:     Message msg, //
 11:     final CommunicationMode communicationMode, //
 12:     final SendCallback sendCallback, //
 13:     final long timeout//
 14: ) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
 15:     // 校验 Producer 处于运行状态
 16:     this.makeSureStateOK();
 17:     // 校验消息格式
 18:     Validators.checkMessage(msg, this.defaultMQProducer);
 19:     //
 20:     final long invokeID = random.nextLong(); // 调用编号；用于下面打印日志，标记为同一次发送消息
 21:     long beginTimestampFirst = System.currentTimeMillis();
 22:     long beginTimestampPrev = beginTimestampFirst;
 23:     long endTimestamp = beginTimestampFirst;
 24:     // 获取 Topic路由信息
 25:     TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
 26:     if (topicPublishInfo != null && topicPublishInfo.ok()) {
 27:         MessageQueue mq = null; // 最后选择消息要发送到的队列
 28:         Exception exception = null;
 29:         SendResult sendResult = null; // 最后一次发送结果
 30:         int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1; // 同步多次调用
 31:         int times = 0; // 第几次发送
 32:         String[] brokersSent = new String[timesTotal]; // 存储每次发送消息选择的broker名
 33:         // 循环调用发送消息，直到成功
 34:         for (; times < timesTotal; times++) {
 35:             String lastBrokerName = null == mq ? null : mq.getBrokerName();
 36:             MessageQueue tmpmq = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName); // 选择消息要发送到的队列
 37:             if (tmpmq != null) {
 38:                 mq = tmpmq;
 39:                 brokersSent[times] = mq.getBrokerName();
 40:                 try {
 41:                     beginTimestampPrev = System.currentTimeMillis();
 42:                     // 调用发送消息核心方法
 43:                     sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout);
 44:                     endTimestamp = System.currentTimeMillis();
 45:                     // 更新Broker可用性信息
 46:                     this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
 47:                     switch (communicationMode) {
 48:                         case ASYNC:
 49:                             return null;
 50:                         case ONEWAY:
 51:                             return null;
 52:                         case SYNC:
 53:                             if (sendResult.getSendStatus() != SendStatus.SEND_OK) {
 54:                                 if (this.defaultMQProducer.isRetryAnotherBrokerWhenNotStoreOK()) { // 同步发送成功但存储有问题时 && 配置存储异常时重新发送开关 时，进行重试
 55:                                     continue;
 56:                                 }
 57:                             }
 58:                             return sendResult;
 59:                         default:
 60:                             break;
 61:                     }
 62:                 } catch (RemotingException e) { // 打印异常，更新Broker可用性信息，更新继续循环
 63:                     endTimestamp = System.currentTimeMillis();
 64:                     this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
 65:                     log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
 66:                     log.warn(msg.toString());
 67:                     exception = e;
 68:                     continue;
 69:                 } catch (MQClientException e) { // 打印异常，更新Broker可用性信息，继续循环
 70:                     endTimestamp = System.currentTimeMillis();
 71:                     this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
 72:                     log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
 73:                     log.warn(msg.toString());
 74:                     exception = e;
 75:                     continue;
 76:                 } catch (MQBrokerException e) { // 打印异常，更新Broker可用性信息，部分情况下的异常，直接返回，结束循环
 77:                     endTimestamp = System.currentTimeMillis();
 78:                     this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
 79:                     log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
 80:                     log.warn(msg.toString());
 81:                     exception = e;
 82:                     switch (e.getResponseCode()) {
 83:                         // 如下异常continue，进行发送消息重试
 84:                         case ResponseCode.TOPIC_NOT_EXIST:
 85:                         case ResponseCode.SERVICE_NOT_AVAILABLE:
 86:                         case ResponseCode.SYSTEM_ERROR:
 87:                         case ResponseCode.NO_PERMISSION:
 88:                         case ResponseCode.NO_BUYER_ID:
 89:                         case ResponseCode.NOT_IN_CURRENT_UNIT:
 90:                             continue;
 91:                         // 如果有发送结果，进行返回，否则，抛出异常；
 92:                         default:
 93:                             if (sendResult != null) {
 94:                                 return sendResult;
 95:                             }
 96:                             throw e;
 97:                     }
 98:                 } catch (InterruptedException e) {
 99:                     endTimestamp = System.currentTimeMillis();
100:                     this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
101:                     log.warn(String.format("sendKernelImpl exception, throw exception, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
102:                     log.warn(msg.toString());
103:                     throw e;
104:                 }
105:             } else {
106:                 break;
107:             }
108:         }
109:         // 返回发送结果
110:         if (sendResult != null) {
111:             return sendResult;
112:         }
113:         // 根据不同情况，抛出不同的异常
114:         String info = String.format("Send [%d] times, still failed, cost [%d]ms, Topic: %s, BrokersSent: %s", times, System.currentTimeMillis() - beginTimestampFirst,
115:                 msg.getTopic(), Arrays.toString(brokersSent)) + FAQUrl.suggestTodo(FAQUrl.SEND_MSG_FAILED);
116:         MQClientException mqClientException = new MQClientException(info, exception);
117:         if (exception instanceof MQBrokerException) {
118:             mqClientException.setResponseCode(((MQBrokerException) exception).getResponseCode());
119:         } else if (exception instanceof RemotingConnectException) {
120:             mqClientException.setResponseCode(ClientErrorCode.CONNECT_BROKER_EXCEPTION);
121:         } else if (exception instanceof RemotingTimeoutException) {
122:             mqClientException.setResponseCode(ClientErrorCode.ACCESS_BROKER_TIMEOUT);
123:         } else if (exception instanceof MQClientException) {
124:             mqClientException.setResponseCode(ClientErrorCode.BROKER_NOT_EXIST_EXCEPTION);
125:         }
126:         throw mqClientException;
127:     }
128:     // Namesrv找不到异常
129:     List<String> nsList = this.getmQClientFactory().getMQClientAPIImpl().getNameServerAddressList();
130:     if (null == nsList || nsList.isEmpty()) {
131:         throw new MQClientException(
132:             "No name server address, please set it." + FAQUrl.suggestTodo(FAQUrl.NAME_SERVER_ADDR_NOT_EXIST_URL), null).setResponseCode(ClientErrorCode.NO_NAME_SERVER_EXCEPTION);
133:     }
134:     // 消息路由找不到异常
135:     throw new MQClientException("No route info of this topic, " + msg.getTopic() + FAQUrl.suggestTodo(FAQUrl.NO_TOPIC_ROUTE_INFO),
136:         null).setResponseCode(ClientErrorCode.NOT_FOUND_TOPIC_EXCEPTION);
137: }
```
* 说明 ：发送消息。步骤：获取消息路由信息，选择要发送到的消息队列，执行消息发送核心方法，并对发送结果进行封装返回。
* 第 1  至 7 行：对`sendsendDefaultImpl(...)`进行封装。
* 第 20 行 ：`invokeID`仅仅用于打印日志，无实际的业务用途。
* 第 25 行 ：获取 Topic路由信息， 详细解析见：[DefaultMQProducerImpl#tryToFindTopicPublishInfo()](#defaultmqproducerimpltrytofindtopicpublishinfo)
* 第 30 & 34 行 ：计算调用发送消息到成功为止的最大次数，并进行循环。同步或异步发送消息会调用多次，默认配置为3次。
* 第 36 行 ：选择消息要发送到的队列，详细解析见：[MQFaultStrategy](#mqfaultstrategy)
* 第 43 行 ：调用发送消息核心方法，详细解析见：[DefaultMQProducerImpl#sendKernelImpl()](#defaultmqproducerimplsendkernelimpl)
* 第 46 行 ：更新`Broker`可用性信息。在选择发送到的消息队列时，会参考`Broker`发送消息的延迟，详细解析见：[MQFaultStrategy](#mqfaultstrategy)
* 第 62 至 68 行：当抛出`RemotingException`时，如果进行消息发送失败重试，则**可能导致消息发送重复**。例如，发送消息超时(`RemotingTimeoutException`)，实际`Broker`接收到该消息并处理成功。因此，`Consumer`在消费时，需要保证幂等性。

### DefaultMQProducerImpl#tryToFindTopicPublishInfo()

```Java
  1: private TopicPublishInfo tryToFindTopicPublishInfo(final String topic) {
  2:     // 缓存中获取 Topic发布信息
  3:     TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);
  4:     // 当无可用的 Topic发布信息时，从Namesrv获取一次
  5:     if (null == topicPublishInfo || !topicPublishInfo.ok()) {
  6:         this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
  7:         this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
  8:         topicPublishInfo = this.topicPublishInfoTable.get(topic);
  9:     }
 10:     // 若获取的 Topic发布信息时候可用，则返回
 11:     if (topicPublishInfo.isHaveTopicRouterInfo() || topicPublishInfo.ok()) {
 12:         return topicPublishInfo;
 13:     } else { // 使用 {@link DefaultMQProducer#createTopicKey} 对应的 Topic发布信息。用于 Topic发布信息不存在 && Broker支持自动创建Topic
 14:         this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer);
 15:         topicPublishInfo = this.topicPublishInfoTable.get(topic);
 16:         return topicPublishInfo;
 17:     }
 18: }
```
* 说明 ：获得 Topic发布信息。优先从缓存`topicPublishInfoTable`，其次从`Namesrv`中获得。
* 第 3 行 ：从缓存`topicPublishInfoTable`中获得 Topic发布信息。
* 第 5 至 9 行 ：从 `Namesrv` 中获得 Topic发布信息。
* 第 13 至 17 行 ：当从 `Namesrv` 无法获取时，使用 `{@link DefaultMQProducer#createTopicKey}` 对应的 Topic发布信息。目的是当 `Broker` 开启自动创建 Topic开关时，`Broker` 接收到消息后自动创建Topic，详细解析见[《RocketMQ 源码分析 —— Topic》](http://www.yunai.me/RocketMQ/topic/)。

### MQFaultStrategy

> ![Latency类图](http://www.yunai.me/images/RocketMQ/2017_04_18/03.png)

#### MQFaultStrategy

```Java
  1: public class MQFaultStrategy {
  2:     private final static Logger log = ClientLogger.getLog();
  3: 
  4:     /**
  5:      * 延迟故障容错，维护每个Broker的发送消息的延迟
  6:      * key：brokerName
  7:      */
  8:     private final LatencyFaultTolerance<String> latencyFaultTolerance = new LatencyFaultToleranceImpl();
  9:     /**
 10:      * 发送消息延迟容错开关
 11:      */
 12:     private boolean sendLatencyFaultEnable = false;
 13:     /**
 14:      * 延迟级别数组
 15:      */
 16:     private long[] latencyMax = {50L, 100L, 550L, 1000L, 2000L, 3000L, 15000L};
 17:     /**
 18:      * 不可用时长数组
 19:      */
 20:     private long[] notAvailableDuration = {0L, 0L, 30000L, 60000L, 120000L, 180000L, 600000L};
 21: 
 22:     /**
 23:      * 根据 Topic发布信息 选择一个消息队列
 24:      *
 25:      * @param tpInfo Topic发布信息
 26:      * @param lastBrokerName brokerName
 27:      * @return 消息队列
 28:      */
 29:     public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
 30:         if (this.sendLatencyFaultEnable) {
 31:             try {
 32:                 // 获取 brokerName=lastBrokerName && 可用的一个消息队列
 33:                 int index = tpInfo.getSendWhichQueue().getAndIncrement();
 34:                 for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++) {
 35:                     int pos = Math.abs(index++) % tpInfo.getMessageQueueList().size();
 36:                     if (pos < 0)
 37:                         pos = 0;
 38:                     MessageQueue mq = tpInfo.getMessageQueueList().get(pos);
 39:                     if (latencyFaultTolerance.isAvailable(mq.getBrokerName())) {
 40:                         if (null == lastBrokerName || mq.getBrokerName().equals(lastBrokerName))
 41:                             return mq;
 42:                     }
 43:                 }
 44:                 // 选择一个相对好的broker，并获得其对应的一个消息队列，不考虑该队列的可用性
 45:                 final String notBestBroker = latencyFaultTolerance.pickOneAtLeast();
 46:                 int writeQueueNums = tpInfo.getQueueIdByBroker(notBestBroker);
 47:                 if (writeQueueNums > 0) {
 48:                     final MessageQueue mq = tpInfo.selectOneMessageQueue();
 49:                     if (notBestBroker != null) {
 50:                         mq.setBrokerName(notBestBroker);
 51:                         mq.setQueueId(tpInfo.getSendWhichQueue().getAndIncrement() % writeQueueNums);
 52:                     }
 53:                     return mq;
 54:                 } else {
 55:                     latencyFaultTolerance.remove(notBestBroker);
 56:                 }
 57:             } catch (Exception e) {
 58:                 log.error("Error occurred when selecting message queue", e);
 59:             }
 60:             // 选择一个消息队列，不考虑队列的可用性
 61:             return tpInfo.selectOneMessageQueue();
 62:         }
 63:         // 获得 lastBrokerName 对应的一个消息队列，不考虑该队列的可用性
 64:         return tpInfo.selectOneMessageQueue(lastBrokerName);
 65:     }
 66: 
 67:     /**
 68:      * 更新延迟容错信息
 69:      *
 70:      * @param brokerName brokerName
 71:      * @param currentLatency 延迟
 72:      * @param isolation 是否隔离。当开启隔离时，默认延迟为30000。目前主要用于发送消息异常时
 73:      */
 74:     public void updateFaultItem(final String brokerName, final long currentLatency, boolean isolation) {
 75:         if (this.sendLatencyFaultEnable) {
 76:             long duration = computeNotAvailableDuration(isolation ? 30000 : currentLatency);
 77:             this.latencyFaultTolerance.updateFaultItem(brokerName, currentLatency, duration);
 78:         }
 79:     }
 80: 
 81:     /**
 82:      * 计算延迟对应的不可用时间
 83:      *
 84:      * @param currentLatency 延迟
 85:      * @return 不可用时间
 86:      */
 87:     private long computeNotAvailableDuration(final long currentLatency) {
 88:         for (int i = latencyMax.length - 1; i >= 0; i--) {
 89:             if (currentLatency >= latencyMax[i])
 90:                 return this.notAvailableDuration[i];
 91:         }
 92:         return 0;
 93:     }
```
* 说明 ：`Producer`消息发送容错策略。默认情况下容错策略关闭，即`sendLatencyFaultEnable=false`。
* 第 30 至 62 行 ：容错策略选择消息队列逻辑。优先获取可用队列，其次选择一个broker获取队列，最差返回任意broker的一个队列。
* 第 64 行 ：未开启容错策略选择消息队列逻辑。
* 第 74 至 79 行 ：更新延迟容错信息。当 `Producer` 发送消息时间过长，则逻辑认为N秒内不可用。按照`latencyMax`，`notAvailableDuration`的配置，对应如下：

    | Producer发送消息消耗时长 | Broker不可用时长 |
    | --- | --- |
    | >= 15000 ms | 600 * 1000 ms  |
    | >= 3000 ms | 180 * 1000 ms  |
    | >= 2000 ms | 120 * 1000 ms  |
    | >= 1000 ms | 60 * 1000 ms  |
    | >= 550 ms | 30 * 1000 ms |
    | >= 100 ms | 0 ms |
    | >= 50 ms | 0 ms |

#### LatencyFaultTolerance

```Java
  1: public interface LatencyFaultTolerance<T> {
  2: 
  3:     /**
  4:      * 更新对应的延迟和不可用时长
  5:      *
  6:      * @param name 对象
  7:      * @param currentLatency 延迟
  8:      * @param notAvailableDuration 不可用时长
  9:      */
 10:     void updateFaultItem(final T name, final long currentLatency, final long notAvailableDuration);
 11: 
 12:     /**
 13:      * 对象是否可用
 14:      *
 15:      * @param name 对象
 16:      * @return 是否可用
 17:      */
 18:     boolean isAvailable(final T name);
 19: 
 20:     /**
 21:      * 移除对象
 22:      *
 23:      * @param name 对象
 24:      */
 25:     void remove(final T name);
 26: 
 27:     /**
 28:      * 获取一个对象
 29:      *
 30:      * @return 对象
 31:      */
 32:     T pickOneAtLeast();
 33: }
```
* 说明 ：延迟故障容错接口

#### LatencyFaultToleranceImpl

```Java
  1: public class LatencyFaultToleranceImpl implements LatencyFaultTolerance<String> {
  2: 
  3:     /**
  4:      * 对象故障信息Table
  5:      */
  6:     private final ConcurrentHashMap<String, FaultItem> faultItemTable = new ConcurrentHashMap<>(16);
  7:     /**
  8:      * 对象选择Index
  9:      * @see #pickOneAtLeast()
 10:      */
 11:     private final ThreadLocalIndex whichItemWorst = new ThreadLocalIndex();
 12: 
 13:     @Override
 14:     public void updateFaultItem(final String name, final long currentLatency, final long notAvailableDuration) {
 15:         FaultItem old = this.faultItemTable.get(name);
 16:         if (null == old) {
 17:             // 创建对象
 18:             final FaultItem faultItem = new FaultItem(name);
 19:             faultItem.setCurrentLatency(currentLatency);
 20:             faultItem.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);
 21:             // 更新对象
 22:             old = this.faultItemTable.putIfAbsent(name, faultItem);
 23:             if (old != null) {
 24:                 old.setCurrentLatency(currentLatency);
 25:                 old.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);
 26:             }
 27:         } else { // 更新对象
 28:             old.setCurrentLatency(currentLatency);
 29:             old.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);
 30:         }
 31:     }
 32: 
 33:     @Override
 34:     public boolean isAvailable(final String name) {
 35:         final FaultItem faultItem = this.faultItemTable.get(name);
 36:         if (faultItem != null) {
 37:             return faultItem.isAvailable();
 38:         }
 39:         return true;
 40:     }
 41: 
 42:     @Override
 43:     public void remove(final String name) {
 44:         this.faultItemTable.remove(name);
 45:     }
 46: 
 47:     /**
 48:      * 选择一个相对优秀的对象
 49:      *
 50:      * @return 对象
 51:      */
 52:     @Override
 53:     public String pickOneAtLeast() {
 54:         // 创建数组
 55:         final Enumeration<FaultItem> elements = this.faultItemTable.elements();
 56:         List<FaultItem> tmpList = new LinkedList<>();
 57:         while (elements.hasMoreElements()) {
 58:             final FaultItem faultItem = elements.nextElement();
 59:             tmpList.add(faultItem);
 60:         }
 61:         //
 62:         if (!tmpList.isEmpty()) {
 63:             // 打乱 + 排序。TODO 疑问：应该只能二选一。猜测Collections.shuffle(tmpList)去掉。
 64:             Collections.shuffle(tmpList);
 65:             Collections.sort(tmpList);
 66:             // 选择顺序在前一半的对象
 67:             final int half = tmpList.size() / 2;
 68:             if (half <= 0) {
 69:                 return tmpList.get(0).getName();
 70:             } else {
 71:                 final int i = this.whichItemWorst.getAndIncrement() % half;
 72:                 return tmpList.get(i).getName();
 73:             }
 74:         }
 75:         return null;
 76:     }
 77: }
```

* 说明 ：延迟故障容错实现。维护每个对象的信息。

#### FaultItem

```Java
  1: class FaultItem implements Comparable<FaultItem> {
  2:     /**
  3:      * 对象名
  4:      */
  5:     private final String name;
  6:     /**
  7:      * 延迟
  8:      */
  9:     private volatile long currentLatency;
 10:     /**
 11:      * 开始可用时间
 12:      */
 13:     private volatile long startTimestamp;
 14: 
 15:     public FaultItem(final String name) {
 16:         this.name = name;
 17:     }
 18: 
 19:     /**
 20:      * 比较对象
 21:      * 可用性 > 延迟 > 开始可用时间
 22:      *
 23:      * @param other other
 24:      * @return 升序
 25:      */
 26:     @Override
 27:     public int compareTo(final FaultItem other) {
 28:         if (this.isAvailable() != other.isAvailable()) {
 29:             if (this.isAvailable())
 30:                 return -1;
 31: 
 32:             if (other.isAvailable())
 33:                 return 1;
 34:         }
 35: 
 36:         if (this.currentLatency < other.currentLatency)
 37:             return -1;
 38:         else if (this.currentLatency > other.currentLatency) {
 39:             return 1;
 40:         }
 41: 
 42:         if (this.startTimestamp < other.startTimestamp)
 43:             return -1;
 44:         else if (this.startTimestamp > other.startTimestamp) {
 45:             return 1;
 46:         }
 47: 
 48:         return 0;
 49:     }
 50: 
 51:     /**
 52:      * 是否可用：当开始可用时间大于当前时间
 53:      *
 54:      * @return 是否可用
 55:      */
 56:     public boolean isAvailable() {
 57:         return (System.currentTimeMillis() - startTimestamp) >= 0;
 58:     }
 59: 
 60:     @Override
 61:     public int hashCode() {
 62:         int result = getName() != null ? getName().hashCode() : 0;
 63:         result = 31 * result + (int) (getCurrentLatency() ^ (getCurrentLatency() >>> 32));
 64:         result = 31 * result + (int) (getStartTimestamp() ^ (getStartTimestamp() >>> 32));
 65:         return result;
 66:     }
 67: 
 68:     @Override
 69:     public boolean equals(final Object o) {
 70:         if (this == o)
 71:             return true;
 72:         if (!(o instanceof FaultItem))
 73:             return false;
 74: 
 75:         final FaultItem faultItem = (FaultItem) o;
 76: 
 77:         if (getCurrentLatency() != faultItem.getCurrentLatency())
 78:             return false;
 79:         if (getStartTimestamp() != faultItem.getStartTimestamp())
 80:             return false;
 81:         return getName() != null ? getName().equals(faultItem.getName()) : faultItem.getName() == null;
 82: 
 83:     }
 84: }
```
* 说明 ：对象故障信息。维护对象的名字、延迟、开始可用的时间。

### DefaultMQProducerImpl#sendKernelImpl()

```Java
  1: private SendResult sendKernelImpl(final Message msg, //
  2:     final MessageQueue mq, //
  3:     final CommunicationMode communicationMode, //
  4:     final SendCallback sendCallback, //
  5:     final TopicPublishInfo topicPublishInfo, //
  6:     final long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
  7:     // 获取 broker地址
  8:     String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
  9:     if (null == brokerAddr) {
 10:         tryToFindTopicPublishInfo(mq.getTopic());
 11:         brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
 12:     }
 13:     //
 14:     SendMessageContext context = null;
 15:     if (brokerAddr != null) {
 16:         // 是否使用broker vip通道。broker会开启两个端口对外服务。
 17:         brokerAddr = MixAll.brokerVIPChannel(this.defaultMQProducer.isSendMessageWithVIPChannel(), brokerAddr);
 18:         byte[] prevBody = msg.getBody(); // 记录消息内容。下面逻辑可能改变消息内容，例如消息压缩。
 19:         try {
 20:             // 设置唯一编号
 21:             MessageClientIDSetter.setUniqID(msg);
 22:             // 消息压缩
 23:             int sysFlag = 0;
 24:             if (this.tryToCompressMessage(msg)) {
 25:                 sysFlag |= MessageSysFlag.COMPRESSED_FLAG;
 26:             }
 27:             // 事务
 28:             final String tranMsg = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
 29:             if (tranMsg != null && Boolean.parseBoolean(tranMsg)) {
 30:                 sysFlag |= MessageSysFlag.TRANSACTION_PREPARED_TYPE;
 31:             }
 32:             // hook：发送消息校验
 33:             if (hasCheckForbiddenHook()) {
 34:                 CheckForbiddenContext checkForbiddenContext = new CheckForbiddenContext();
 35:                 checkForbiddenContext.setNameSrvAddr(this.defaultMQProducer.getNamesrvAddr());
 36:                 checkForbiddenContext.setGroup(this.defaultMQProducer.getProducerGroup());
 37:                 checkForbiddenContext.setCommunicationMode(communicationMode);
 38:                 checkForbiddenContext.setBrokerAddr(brokerAddr);
 39:                 checkForbiddenContext.setMessage(msg);
 40:                 checkForbiddenContext.setMq(mq);
 41:                 checkForbiddenContext.setUnitMode(this.isUnitMode());
 42:                 this.executeCheckForbiddenHook(checkForbiddenContext);
 43:             }
 44:             // hook：发送消息前逻辑
 45:             if (this.hasSendMessageHook()) {
 46:                 context = new SendMessageContext();
 47:                 context.setProducer(this);
 48:                 context.setProducerGroup(this.defaultMQProducer.getProducerGroup());
 49:                 context.setCommunicationMode(communicationMode);
 50:                 context.setBornHost(this.defaultMQProducer.getClientIP());
 51:                 context.setBrokerAddr(brokerAddr);
 52:                 context.setMessage(msg);
 53:                 context.setMq(mq);
 54:                 String isTrans = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
 55:                 if (isTrans != null && isTrans.equals("true")) {
 56:                     context.setMsgType(MessageType.Trans_Msg_Half);
 57:                 }
 58:                 if (msg.getProperty("__STARTDELIVERTIME") != null || msg.getProperty(MessageConst.PROPERTY_DELAY_TIME_LEVEL) != null) {
 59:                     context.setMsgType(MessageType.Delay_Msg);
 60:                 }
 61:                 this.executeSendMessageHookBefore(context);
 62:             }
 63:             // 构建发送消息请求
 64:             SendMessageRequestHeader requestHeader = new SendMessageRequestHeader();
 65:             requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());
 66:             requestHeader.setTopic(msg.getTopic());
 67:             requestHeader.setDefaultTopic(this.defaultMQProducer.getCreateTopicKey());
 68:             requestHeader.setDefaultTopicQueueNums(this.defaultMQProducer.getDefaultTopicQueueNums());
 69:             requestHeader.setQueueId(mq.getQueueId());
 70:             requestHeader.setSysFlag(sysFlag);
 71:             requestHeader.setBornTimestamp(System.currentTimeMillis());
 72:             requestHeader.setFlag(msg.getFlag());
 73:             requestHeader.setProperties(MessageDecoder.messageProperties2String(msg.getProperties()));
 74:             requestHeader.setReconsumeTimes(0);
 75:             requestHeader.setUnitMode(this.isUnitMode());
 76:             if (requestHeader.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) { // 消息重发Topic
 77:                 String reconsumeTimes = MessageAccessor.getReconsumeTime(msg);
 78:                 if (reconsumeTimes != null) {
 79:                     requestHeader.setReconsumeTimes(Integer.valueOf(reconsumeTimes));
 80:                     MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_RECONSUME_TIME);
 81:                 }
 82:                 String maxReconsumeTimes = MessageAccessor.getMaxReconsumeTimes(msg);
 83:                 if (maxReconsumeTimes != null) {
 84:                     requestHeader.setMaxReconsumeTimes(Integer.valueOf(maxReconsumeTimes));
 85:                     MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_MAX_RECONSUME_TIMES);
 86:                 }
 87:             }
 88:             // 发送消息
 89:             SendResult sendResult = null;
 90:             switch (communicationMode) {
 91:                 case ASYNC:
 92:                     sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(//
 93:                         brokerAddr, // 1
 94:                         mq.getBrokerName(), // 2
 95:                         msg, // 3
 96:                         requestHeader, // 4
 97:                         timeout, // 5
 98:                         communicationMode, // 6
 99:                         sendCallback, // 7
100:                         topicPublishInfo, // 8
101:                         this.mQClientFactory, // 9
102:                         this.defaultMQProducer.getRetryTimesWhenSendAsyncFailed(), // 10
103:                         context, //
104:                         this);
105:                     break;
106:                 case ONEWAY:
107:                 case SYNC:
108:                     sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
109:                         brokerAddr,
110:                         mq.getBrokerName(),
111:                         msg,
112:                         requestHeader,
113:                         timeout,
114:                         communicationMode,
115:                         context,
116:                         this);
117:                     break;
118:                 default:
119:                     assert false;
120:                     break;
121:             }
122:             // hook：发送消息后逻辑
123:             if (this.hasSendMessageHook()) {
124:                 context.setSendResult(sendResult);
125:                 this.executeSendMessageHookAfter(context);
126:             }
127:             // 返回发送结果
128:             return sendResult;
129:         } catch (RemotingException e) {
130:             if (this.hasSendMessageHook()) {
131:                 context.setException(e);
132:                 this.executeSendMessageHookAfter(context);
133:             }
134:             throw e;
135:         } catch (MQBrokerException e) {
136:             if (this.hasSendMessageHook()) {
137:                 context.setException(e);
138:                 this.executeSendMessageHookAfter(context);
139:             }
140:             throw e;
141:         } catch (InterruptedException e) {
142:             if (this.hasSendMessageHook()) {
143:                 context.setException(e);
144:                 this.executeSendMessageHookAfter(context);
145:             }
146:             throw e;
147:         } finally {
148:             msg.setBody(prevBody);
149:         }
150:     }
151:     // broker为空抛出异常
152:     throw new MQClientException("The broker[" + mq.getBrokerName() + "] not exist", null);
153: }
```
* 说明 ：发送消息核心方法。该方法真正发起网络请求，发送消息给 `Broker`。
* 第 21 行 ：生产消息编号，详细解析见[《RocketMQ 源码分析 —— Message 基础》](http://www.yunai.me/RocketMQ/message/)。
* 第 64 至 121 行 ：构建发送消息请求`SendMessageRequestHeader`。
* 第 107 至 117 行 ：执行 `MQClientInstance#sendMessage(...)` 发起网络请求。

# 3、Broker 接收消息

> ![接收发送消息API顺序图](http://www.yunai.me/images/RocketMQ/2017_04_18/04.png)

## SendMessageProcessor#sendMessage

```Java
  1: @Override
  2: public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException {
  3:     SendMessageContext mqtraceContext;
  4:     switch (request.getCode()) {
  5:         case RequestCode.CONSUMER_SEND_MSG_BACK:
  6:             return this.consumerSendMsgBack(ctx, request);
  7:         default:
  8:             // 解析请求
  9:             SendMessageRequestHeader requestHeader = parseRequestHeader(request);
 10:             if (requestHeader == null) {
 11:                 return null;
 12:             }
 13:             // 发送请求Context。在 hook 场景下使用
 14:             mqtraceContext = buildMsgContext(ctx, requestHeader);
 15:             // hook：处理发送消息前逻辑
 16:             this.executeSendMessageHookBefore(ctx, request, mqtraceContext);
 17:             // 处理发送消息逻辑
 18:             final RemotingCommand response = this.sendMessage(ctx, request, mqtraceContext, requestHeader);
 19:             // hook：处理发送消息后逻辑
 20:             this.executeSendMessageHookAfter(response, mqtraceContext);
 21:             return response;
 22:     }
 23: }
 24: 
 25: private RemotingCommand sendMessage(final ChannelHandlerContext ctx, //
 26:     final RemotingCommand request, //
 27:     final SendMessageContext sendMessageContext, //
 28:     final SendMessageRequestHeader requestHeader) throws RemotingCommandException {
 29: 
 30:     // 初始化响应
 31:     final RemotingCommand response = RemotingCommand.createResponseCommand(SendMessageResponseHeader.class);
 32:     final SendMessageResponseHeader responseHeader = (SendMessageResponseHeader) response.readCustomHeader();
 33:     response.setOpaque(request.getOpaque());
 34:     response.addExtField(MessageConst.PROPERTY_MSG_REGION, this.brokerController.getBrokerConfig().getRegionId());
 35:     response.addExtField(MessageConst.PROPERTY_TRACE_SWITCH, String.valueOf(this.brokerController.getBrokerConfig().isTraceOn()));
 36: 
 37:     if (log.isDebugEnabled()) {
 38:         log.debug("receive SendMessage request command, {}", request);
 39:     }
 40: 
 41:     // 如果未开始接收消息，抛出系统异常
 42:     @SuppressWarnings("SpellCheckingInspection")
 43:     final long startTimstamp = this.brokerController.getBrokerConfig().getStartAcceptSendRequestTimeStamp();
 44:     if (this.brokerController.getMessageStore().now() < startTimstamp) {
 45:         response.setCode(ResponseCode.SYSTEM_ERROR);
 46:         response.setRemark(String.format("broker unable to service, until %s", UtilAll.timeMillisToHumanString2(startTimstamp)));
 47:         return response;
 48:     }
 49: 
 50:     // 消息配置(Topic配置）校验
 51:     response.setCode(-1);
 52:     super.msgCheck(ctx, requestHeader, response);
 53:     if (response.getCode() != -1) {
 54:         return response;
 55:     }
 56: 
 57:     final byte[] body = request.getBody();
 58: 
 59:     // 如果队列小于0，从可用队列随机选择
 60:     int queueIdInt = requestHeader.getQueueId();
 61:     TopicConfig topicConfig = this.brokerController.getTopicConfigManager().selectTopicConfig(requestHeader.getTopic());
 62:     if (queueIdInt < 0) {
 63:         queueIdInt = Math.abs(this.random.nextInt() % 99999999) % topicConfig.getWriteQueueNums();
 64:     }
 65: 
 66:     //
 67:     int sysFlag = requestHeader.getSysFlag();
 68:     if (TopicFilterType.MULTI_TAG == topicConfig.getTopicFilterType()) {
 69:         sysFlag |= MessageSysFlag.MULTI_TAGS_FLAG;
 70:     }
 71: 
 72:     // 对RETRY类型的消息处理。如果超过最大消费次数，则topic修改成"%DLQ%" + 分组名，即加入 死信队列(Dead Letter Queue)
 73:     String newTopic = requestHeader.getTopic();
 74:     if (null != newTopic && newTopic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
 75:         // 获取订阅分组配置
 76:         String groupName = newTopic.substring(MixAll.RETRY_GROUP_TOPIC_PREFIX.length());
 77:         SubscriptionGroupConfig subscriptionGroupConfig =
 78:             this.brokerController.getSubscriptionGroupManager().findSubscriptionGroupConfig(groupName);
 79:         if (null == subscriptionGroupConfig) {
 80:             response.setCode(ResponseCode.SUBSCRIPTION_GROUP_NOT_EXIST);
 81:             response.setRemark("subscription group not exist, " + groupName + " " + FAQUrl.suggestTodo(FAQUrl.SUBSCRIPTION_GROUP_NOT_EXIST));
 82:             return response;
 83:         }
 84:         // 计算最大可消费次数
 85:         int maxReconsumeTimes = subscriptionGroupConfig.getRetryMaxTimes();
 86:         if (request.getVersion() >= MQVersion.Version.V3_4_9.ordinal()) {
 87:             maxReconsumeTimes = requestHeader.getMaxReconsumeTimes();
 88:         }
 89:         int reconsumeTimes = requestHeader.getReconsumeTimes() == null ? 0 : requestHeader.getReconsumeTimes();
 90:         if (reconsumeTimes >= maxReconsumeTimes) { // 超过最大消费次数
 91:             newTopic = MixAll.getDLQTopic(groupName);
 92:             queueIdInt = Math.abs(this.random.nextInt() % 99999999) % DLQ_NUMS_PER_GROUP;
 93:             topicConfig = this.brokerController.getTopicConfigManager().createTopicInSendMessageBackMethod(newTopic, //
 94:                 DLQ_NUMS_PER_GROUP, //
 95:                 PermName.PERM_WRITE, 0
 96:             );
 97:             if (null == topicConfig) {
 98:                 response.setCode(ResponseCode.SYSTEM_ERROR);
 99:                 response.setRemark("topic[" + newTopic + "] not exist");
100:                 return response;
101:             }
102:         }
103:     }
104: 
105:     // 创建MessageExtBrokerInner
106:     MessageExtBrokerInner msgInner = new MessageExtBrokerInner();
107:     msgInner.setTopic(newTopic);
108:     msgInner.setBody(body);
109:     msgInner.setFlag(requestHeader.getFlag());
110:     MessageAccessor.setProperties(msgInner, MessageDecoder.string2messageProperties(requestHeader.getProperties()));
111:     msgInner.setPropertiesString(requestHeader.getProperties());
112:     msgInner.setTagsCode(MessageExtBrokerInner.tagsString2tagsCode(topicConfig.getTopicFilterType(), msgInner.getTags()));
113:     msgInner.setQueueId(queueIdInt);
114:     msgInner.setSysFlag(sysFlag);
115:     msgInner.setBornTimestamp(requestHeader.getBornTimestamp());
116:     msgInner.setBornHost(ctx.channel().remoteAddress());
117:     msgInner.setStoreHost(this.getStoreHost());
118:     msgInner.setReconsumeTimes(requestHeader.getReconsumeTimes() == null ? 0 : requestHeader.getReconsumeTimes());
119: 
120:     // 校验是否不允许发送事务消息
121:     if (this.brokerController.getBrokerConfig().isRejectTransactionMessage()) {
122:         String traFlag = msgInner.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
123:         if (traFlag != null) {
124:             response.setCode(ResponseCode.NO_PERMISSION);
125:             response.setRemark(
126:                 "the broker[" + this.brokerController.getBrokerConfig().getBrokerIP1() + "] sending transaction message is forbidden");
127:             return response;
128:         }
129:     }
130: 
131:     // 添加消息
132:     PutMessageResult putMessageResult = this.brokerController.getMessageStore().putMessage(msgInner);
133:     if (putMessageResult != null) {
134:         boolean sendOK = false;
135: 
136:         switch (putMessageResult.getPutMessageStatus()) {
137:             // Success
138:             case PUT_OK:
139:                 sendOK = true;
140:                 response.setCode(ResponseCode.SUCCESS);
141:                 break;
142:             case FLUSH_DISK_TIMEOUT:
143:                 response.setCode(ResponseCode.FLUSH_DISK_TIMEOUT);
144:                 sendOK = true;
145:                 break;
146:             case FLUSH_SLAVE_TIMEOUT:
147:                 response.setCode(ResponseCode.FLUSH_SLAVE_TIMEOUT);
148:                 sendOK = true;
149:                 break;
150:             case SLAVE_NOT_AVAILABLE:
151:                 response.setCode(ResponseCode.SLAVE_NOT_AVAILABLE);
152:                 sendOK = true;
153:                 break;
154: 
155:             // Failed
156:             case CREATE_MAPEDFILE_FAILED:
157:                 response.setCode(ResponseCode.SYSTEM_ERROR);
158:                 response.setRemark("create mapped file failed, server is busy or broken.");
159:                 break;
160:             case MESSAGE_ILLEGAL:
161:             case PROPERTIES_SIZE_EXCEEDED:
162:                 response.setCode(ResponseCode.MESSAGE_ILLEGAL);
163:                 response.setRemark(
164:                     "the message is illegal, maybe msg body or properties length not matched. msg body length limit 128k, msg properties length limit 32k.");
165:                 break;
166:             case SERVICE_NOT_AVAILABLE:
167:                 response.setCode(ResponseCode.SERVICE_NOT_AVAILABLE);
168:                 response.setRemark(
169:                     "service not available now, maybe disk full, " + diskUtil() + ", maybe your broker machine memory too small.");
170:                 break;
171:             case OS_PAGECACHE_BUSY:
172:                 response.setCode(ResponseCode.SYSTEM_ERROR);
173:                 response.setRemark("[PC_SYNCHRONIZED]broker busy, start flow control for a while");
174:                 break;
175:             case UNKNOWN_ERROR:
176:                 response.setCode(ResponseCode.SYSTEM_ERROR);
177:                 response.setRemark("UNKNOWN_ERROR");
178:                 break;
179:             default:
180:                 response.setCode(ResponseCode.SYSTEM_ERROR);
181:                 response.setRemark("UNKNOWN_ERROR DEFAULT");
182:                 break;
183:         }
184: 
185:         String owner = request.getExtFields().get(BrokerStatsManager.COMMERCIAL_OWNER);
186:         if (sendOK) {
187:             // 统计
188:             this.brokerController.getBrokerStatsManager().incTopicPutNums(msgInner.getTopic());
189:             this.brokerController.getBrokerStatsManager().incTopicPutSize(msgInner.getTopic(), putMessageResult.getAppendMessageResult().getWroteBytes());
190:             this.brokerController.getBrokerStatsManager().incBrokerPutNums();
191: 
192:             // 响应
193:             response.setRemark(null);
194:             responseHeader.setMsgId(putMessageResult.getAppendMessageResult().getMsgId());
195:             responseHeader.setQueueId(queueIdInt);
196:             responseHeader.setQueueOffset(putMessageResult.getAppendMessageResult().getLogicsOffset());
197:             doResponse(ctx, request, response);
198: 
199:             // hook：设置发送成功到context
200:             if (hasSendMessageHook()) {
201:                 sendMessageContext.setMsgId(responseHeader.getMsgId());
202:                 sendMessageContext.setQueueId(responseHeader.getQueueId());
203:                 sendMessageContext.setQueueOffset(responseHeader.getQueueOffset());
204: 
205:                 int commercialBaseCount = brokerController.getBrokerConfig().getCommercialBaseCount();
206:                 int wroteSize = putMessageResult.getAppendMessageResult().getWroteBytes();
207:                 int incValue = (int) Math.ceil(wroteSize / BrokerStatsManager.SIZE_PER_COUNT) * commercialBaseCount;
208: 
209:                 sendMessageContext.setCommercialSendStats(BrokerStatsManager.StatsType.SEND_SUCCESS);
210:                 sendMessageContext.setCommercialSendTimes(incValue);
211:                 sendMessageContext.setCommercialSendSize(wroteSize);
212:                 sendMessageContext.setCommercialOwner(owner);
213:             }
214:             return null;
215:         } else {
216:             // hook：设置发送失败到context
217:             if (hasSendMessageHook()) {
218:                 int wroteSize = request.getBody().length;
219:                 int incValue = (int) Math.ceil(wroteSize / BrokerStatsManager.SIZE_PER_COUNT);
220: 
221:                 sendMessageContext.setCommercialSendStats(BrokerStatsManager.StatsType.SEND_FAILURE);
222:                 sendMessageContext.setCommercialSendTimes(incValue);
223:                 sendMessageContext.setCommercialSendSize(wroteSize);
224:                 sendMessageContext.setCommercialOwner(owner);
225:             }
226:         }
227:     } else {
228:         response.setCode(ResponseCode.SYSTEM_ERROR);
229:         response.setRemark("store putMessage return null");
230:     }
231: 
232:     return response;
233: }
```
* `#processRequest()` 说明 ：处理消息请求。
* `#sendMessage()` 说明 ：发送消息，并返回发送消息结果。
* 第 51 至 55 行 ：消息配置(Topic配置）校验，详细解析见：[AbstractSendMessageProcessor#msgCheck()](#abstractsendmessageprocessormsgcheck)。
* 第 60 至 64 行 ：消息队列编号小于0时，`Broker` 可以设置随机选择一个消息队列。
* 第 72 至 103 行 ：对RETRY类型的消息处理。如果超过最大消费次数，则topic修改成"%DLQ%" + 分组名， 即加  死信队 (Dead Letter Queue)，详细解析见：[《RocketMQ 源码分析 —— Topic》](http://www.yunai.me/RocketMQ/topic/)。
* 第 105 至 118 行 ：创建`MessageExtBrokerInner`。
* 第 132 ：存储消息，详细解析见：[DefaultMessageStore#putMessage()](defaultmessagestoreputmessage)。
* 第 133 至 183 行 ：处理消息发送结果，设置响应结果和提示。
* 第 186 至 214 行 ：发送成功，响应。这里`doResponse(ctx, request, response)`进行响应，最后`return null`，原因是：响应给 `Producer` 可能发生异常，`#doResponse(ctx, request, response)`捕捉了该异常并输出日志。这样做的话，我们进行排查 `Broker` 接收消息成功后响应是否存在异常会方便很多。

### AbstractSendMessageProcessor#msgCheck

```Java
  1: protected RemotingCommand msgCheck(final ChannelHandlerContext ctx,
  2:                                    final SendMessageRequestHeader requestHeader, final RemotingCommand response) {
  3:     // 检查 broker 是否有写入权限
  4:     if (!PermName.isWriteable(this.brokerController.getBrokerConfig().getBrokerPermission())
  5:         && this.brokerController.getTopicConfigManager().isOrderTopic(requestHeader.getTopic())) {
  6:         response.setCode(ResponseCode.NO_PERMISSION);
  7:         response.setRemark("the broker[" + this.brokerController.getBrokerConfig().getBrokerIP1()
  8:             + "] sending message is forbidden");
  9:         return response;
 10:     }
 11:     // 检查topic是否可以被发送。目前是{@link MixAll.DEFAULT_TOPIC}不被允许发送
 12:     if (!this.brokerController.getTopicConfigManager().isTopicCanSendMessage(requestHeader.getTopic())) {
 13:         String errorMsg = "the topic[" + requestHeader.getTopic() + "] is conflict with system reserved words.";
 14:         log.warn(errorMsg);
 15:         response.setCode(ResponseCode.SYSTEM_ERROR);
 16:         response.setRemark(errorMsg);
 17:         return response;
 18:     }
 19:     TopicConfig topicConfig = this.brokerController.getTopicConfigManager().selectTopicConfig(requestHeader.getTopic());
 20:     if (null == topicConfig) { // 不能存在topicConfig，则进行创建
 21:         int topicSysFlag = 0;
 22:         if (requestHeader.isUnitMode()) {
 23:             if (requestHeader.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
 24:                 topicSysFlag = TopicSysFlag.buildSysFlag(false, true);
 25:             } else {
 26:                 topicSysFlag = TopicSysFlag.buildSysFlag(true, false);
 27:             }
 28:         }
 29:         // 创建topic配置
 30:         log.warn("the topic {} not exist, producer: {}", requestHeader.getTopic(), ctx.channel().remoteAddress());
 31:         topicConfig = this.brokerController.getTopicConfigManager().createTopicInSendMessageMethod(//
 32:             requestHeader.getTopic(), //
 33:             requestHeader.getDefaultTopic(), //
 34:             RemotingHelper.parseChannelRemoteAddr(ctx.channel()), //
 35:             requestHeader.getDefaultTopicQueueNums(), topicSysFlag);
 36:         if (null == topicConfig) {
 37:             if (requestHeader.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
 38:                 topicConfig =
 39:                     this.brokerController.getTopicConfigManager().createTopicInSendMessageBackMethod(
 40:                         requestHeader.getTopic(), 1, PermName.PERM_WRITE | PermName.PERM_READ,
 41:                         topicSysFlag);
 42:             }
 43:         }
 44:         // 如果没配置
 45:         if (null == topicConfig) {
 46:             response.setCode(ResponseCode.TOPIC_NOT_EXIST);
 47:             response.setRemark("topic[" + requestHeader.getTopic() + "] not exist, apply first please!"
 48:                 + FAQUrl.suggestTodo(FAQUrl.APPLY_TOPIC_URL));
 49:             return response;
 50:         }
 51:     }
 52:     // 队列编号是否正确
 53:     int queueIdInt = requestHeader.getQueueId();
 54:     int idValid = Math.max(topicConfig.getWriteQueueNums(), topicConfig.getReadQueueNums());
 55:     if (queueIdInt >= idValid) {
 56:         String errorInfo = String.format("request queueId[%d] is illegal, %s Producer: %s",
 57:             queueIdInt,
 58:             topicConfig.toString(),
 59:             RemotingHelper.parseChannelRemoteAddr(ctx.channel()));
 60:         log.warn(errorInfo);
 61:         response.setCode(ResponseCode.SYSTEM_ERROR);
 62:         response.setRemark(errorInfo);
 63:         return response;
 64:     }
 65:     return response;
 66: }
```
* 说明：校验消息是否正确，主要是Topic配置方面，例如：`Broker` 是否有写入权限，topic配置是否存在，队列编号是否正确。
* 第 11 至 18 行 ：检查Topic是否可以被发送。目前是 `{@link MixAll.DEFAULT_TOPIC}` 不被允许发送。
* 第 20 至 51 行 ：当找不到Topic配置，则进行创建。当然，创建会存在不成功的情况，例如说：`defaultTopic` 的Topic配置不存在，又或者是 存在但是不允许继承，详细解析见[《RocketMQ 源码分析 —— Topic》](http://www.yunai.me/RocketMQ/topic/)。

## DefaultMessageStore#putMessage

```Java
  1: public PutMessageResult putMessage(MessageExtBrokerInner msg) {
  2:     if (this.shutdown) {
  3:         log.warn("message store has shutdown, so putMessage is forbidden");
  4:         return new PutMessageResult(PutMessageStatus.SERVICE_NOT_AVAILABLE, null);
  5:     }
  6: 
  7:     // 从节点不允许写入
  8:     if (BrokerRole.SLAVE == this.messageStoreConfig.getBrokerRole()) {
  9:         long value = this.printTimes.getAndIncrement();
 10:         if ((value % 50000) == 0) {
 11:             log.warn("message store is slave mode, so putMessage is forbidden ");
 12:         }
 13: 
 14:         return new PutMessageResult(PutMessageStatus.SERVICE_NOT_AVAILABLE, null);
 15:     }
 16: 
 17:     // store是否允许写入
 18:     if (!this.runningFlags.isWriteable()) {
 19:         long value = this.printTimes.getAndIncrement();
 20:         if ((value % 50000) == 0) {
 21:             log.warn("message store is not writeable, so putMessage is forbidden " + this.runningFlags.getFlagBits());
 22:         }
 23: 
 24:         return new PutMessageResult(PutMessageStatus.SERVICE_NOT_AVAILABLE, null);
 25:     } else {
 26:         this.printTimes.set(0);
 27:     }
 28: 
 29:     // 消息过长
 30:     if (msg.getTopic().length() > Byte.MAX_VALUE) {
 31:         log.warn("putMessage message topic length too long " + msg.getTopic().length());
 32:         return new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, null);
 33:     }
 34: 
 35:     // 消息附加属性过长
 36:     if (msg.getPropertiesString() != null && msg.getPropertiesString().length() > Short.MAX_VALUE) {
 37:         log.warn("putMessage message properties length too long " + msg.getPropertiesString().length());
 38:         return new PutMessageResult(PutMessageStatus.PROPERTIES_SIZE_EXCEEDED, null);
 39:     }
 40: 
 41:     if (this.isOSPageCacheBusy()) {
 42:         return new PutMessageResult(PutMessageStatus.OS_PAGECACHE_BUSY, null);
 43:     }
 44: 
 45:     long beginTime = this.getSystemClock().now();
 46:     // 添加消息到commitLog
 47:     PutMessageResult result = this.commitLog.putMessage(msg);
 48: 
 49:     long eclipseTime = this.getSystemClock().now() - beginTime;
 50:     if (eclipseTime > 500) {
 51:         log.warn("putMessage not in lock eclipse time(ms)={}, bodyLength={}", eclipseTime, msg.getBody().length);
 52:     }
 53:     this.storeStatsService.setPutMessageEntireTimeMax(eclipseTime);
 54: 
 55:     if (null == result || !result.isOk()) {
 56:         this.storeStatsService.getPutMessageFailedTimes().incrementAndGet();
 57:     }
 58: 
 59:     return result;
 60: }
```
* 说明：存储消息封装，最终存储需要 `CommitLog` 实现。
* 第 7 至 27 行 ：校验 `Broker` 是否可以写入。
* 第 29 至 39 行 ：消息格式与大小校验。
* 第 47 行 ：调用 `CommitLong` 进行存储，详细逻辑见：[《RocketMQ 源码分析 —— Message 存储》](http://www.yunai.me/RocketMQ/message-store/)

# 4、某种结尾

感谢阅读、收藏、点赞本文的工程师同学。

阅读源码是件令自己很愉悦的事情，编写源码解析是让自己脑细胞死伤无数的过程，痛并快乐着。

如果有内容写的存在错误，或是不清晰的地方，见笑了，🙂。欢迎加 QQ：7685413 我们一起探讨，共进步。

再次感谢阅读、收藏、点赞本文的工程师同学。









