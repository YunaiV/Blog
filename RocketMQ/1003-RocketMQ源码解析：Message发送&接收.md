# 1、概述

1. `Producer` 发送消息。主要是**同步**发送消息源码，涉及到 异步/Oneway发送消息，事务消息会跳过。
2. `Broker` 接收消息。(*存储消息在[《RocketMQ源码解析：Message存储》](https://github.com/YunaiV/Blog/blob/master/RocketMQ/1004-RocketMQ源码解析：Message存储.md)解析*)

> ![Producer发送消息全局顺序图](images/1003/Producer发送消息全局顺序图.png)

# 2、Producer 发送消息

 > ![Producer发送消息顺序图](images/1003/Producer发送消息顺序图.png)

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
* 第 30 & 34 行 ：计算调用发送消息到成功为止的最大次数，并进行循环。当且仅当同步发送消息会调用多次，默认配置为3次。
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
* 第 13 至 17 行 ：当从 `Namesrv` 无法获取时，使用 `{@link DefaultMQProducer#createTopicKey}` 对应的 Topic发布信息。目的是当 `Broker` 开启自动创建 Topic开关时，`Broker` 接收到消息后自动创建Topic，详细解析见[《RocketMQ源码解析：Topic》](https://github.com/YunaiV/Blog/blob/master/RocketMQ/1001-RocketMQ源码解析：Topic.md)。

### MQFaultStrategy

> ![Latency类图](images/1003/Client之Latency类图.png)

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
* 第 74 至 79 行 ：更新延迟容错信息。当 `Producer` 发送消息时间过长，则逻辑认为N秒内部可用。按照`latencyMax`，`notAvailableDuration`的配置，对应如下：

| Producer发送消息消耗时长 | Broker不可用时间 |
| --- | --- |
| < 50 ms | 0 ms |
| < 100 ms | 0 ms |
| < 550 ms | 30 * 1000 ms |
| < 1000 ms | 60 * 1000 ms  |
| < 2000 ms | 120 * 1000 ms  |
| < 3000 ms | 180 * 1000 ms  |
| < 15000 ms | 600 * 1000 ms  |


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

TODO

# 3、Broker 接收消息

> ![接收发送消息API顺序图](images/1003/Broker接收发送消息API顺序图.png)








