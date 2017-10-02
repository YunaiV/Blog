title: RocketMQ 源码分析 —— Filtersrv
date: 2017-05-17
tags:
categories: RocketMQ
permalink: RocketMQ/filtersrv

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

- [1. 概述](#)
- [2. Filtersrv 注册到 Broker](#)
- [3. 过滤类](#)
	- [3.1 Consumer 订阅时设置 过滤类代码](#)
	- [3.2 Consumer 上传 过滤类代码](#)
	- [3.3 Filter 编译 过滤类代码](#)
- [4. 过滤消息](#)
	- [4.1 Consumer 从 Filtersrv 拉取消息](#)
	- [4.2 Filtersrv 从 Broker 拉取消息](#)
- [5. Filtersrv 高可用](#)

# 1. 概述

`Filtersrv` ，负责**自定义规则**过滤 `Consumer` 从 `Broker` 拉取的消息。

![Filtersrv.png](http://www.iocoder.cn/images/RocketMQ/2017_05_17/Filtersrv.png)

为什么 `Broker` 不提供过滤消息的功能呢？我们来看看官方的说法：

> * Broker 端消息过滤  
>  在 Broker 中，按照 Consumer 的要求做过滤，优点是减少了对于 Consumer 无用消息的网络传输。 缺点是增加了 Broker 的负担，实现相对复杂。  
> (1). 淘宝 Notify 支持多种过滤方式，包含直接按照消息类型过滤，灵活的语法表达式过滤，几乎可以满足最苛刻的过滤需求。  
> (2). 淘宝 RocketMQ 支持按照简单的 Message Tag 过滤，也支持按照 Message Header、body 进行过滤。  
(3). CORBA Notification 规范中也支持灵活的语法表达式过滤。  
> * Consumer 端消息过滤  
> 这种过滤方式可由应用完全自定义实现，但是缺点是很多无用的消息要传输到 Consumer 端。

**就是在这种考虑下，`Filtersrv` 出现了。减少了 `Broker` 的负担，又减少了 `Consumer` 接收无用的消息。当然缺点也是有的，多了一层 `Filtersrv` 网络开销。**
 
# 2. Filtersrv 注册到 Broker

* 🦅 一个 `Filtersrv` **只**对应一个 `Broker`。
* 🦅 一个 `Broker` 可以对应**多个** `Filtersrv`。**`Filtersrv` 的高可用通过启动多个 `Filtersrv` 实现**。
* 🦅 `Filtersrv` 注册失败时，主动**退出关闭**。

核心代码如下：

```Java
  1: // ⬇️⬇️⬇️【FiltersrvController.java】
  2: public boolean initialize() {
  3:     // ....(省略代码)
  4: 
  5:     // 固定间隔注册到Broker
  6:     this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
  7: 
  8:         @Override
  9:         public void run() {
 10:             FiltersrvController.this.registerFilterServerToBroker();
 11:         }
 12:     }, 15, 10, TimeUnit.SECONDS); // TODO edit by 芋艿：initialDelay时间太短，可能导致初始化失败。从3=》15
 13: 
 14:     // ....(省略代码)
 15: }
 16: 
 17: /**
 18:  * 注册Filtersrv 到 Broker
 19:  * ！！！如果注册失败，关闭Filtersrv
 20:  */
 21: public void registerFilterServerToBroker() {
 22:     try {
 23:         RegisterFilterServerResponseHeader responseHeader =
 24:             this.filterServerOuterAPI.registerFilterServerToBroker(
 25:                 this.filtersrvConfig.getConnectWhichBroker(), this.localAddr());
 26:         this.defaultMQPullConsumer.getDefaultMQPullConsumerImpl().getPullAPIWrapper()
 27:             .setDefaultBrokerId(responseHeader.getBrokerId());
 28: 
 29:         if (null == this.brokerName) {
 30:             this.brokerName = responseHeader.getBrokerName();
 31:         }
 32: 
 33:         log.info("register filter server<{}> to broker<{}> OK, Return: {} {}",
 34:             this.localAddr(),
 35:             this.filtersrvConfig.getConnectWhichBroker(),
 36:             responseHeader.getBrokerName(),
 37:             responseHeader.getBrokerId());
 38:     } catch (Exception e) {
 39:         log.warn("register filter server Exception", e);
 40: 
 41:         log.warn("access broker failed, kill oneself");
 42:         System.exit(-1); // 异常退出
 43:     }
 44: }
```

# 3. 过滤类 

![Filtersrv过滤类](http://www.iocoder.cn/images/RocketMQ/2017_05_17/03.png)

## 3.1 Consumer 订阅时设置 过滤类代码

* 🦅 `Consumer` 针对每个 `Topic` 可以订阅不同的 `过滤类代码`。

```Java
  1: // ⬇️⬇️⬇️【DefaultMQPushConsumer.java】
  2: @Override
  3: public void subscribe(String topic, String fullClassName, String filterClassSource) throws MQClientException {
  4:     this.defaultMQPushConsumerImpl.subscribe(topic, fullClassName, filterClassSource);
  5: }
```

## 3.2 Consumer 上传 过滤类代码

* 🦅 `Consumer` 心跳注册到 `Broker` 的同时，上传 `过滤类代码` 到 `Broker` 对应的**所有** `Filtersrv`。

```Java
  1: // ⬇️⬇️⬇️【MQClientInstance.java】
  2: /**
  3:  * 发送心跳到Broker，上传过滤类源码到Filtersrv
  4:  */
  5: public void sendHeartbeatToAllBrokerWithLock() {
  6:     if (this.lockHeartbeat.tryLock()) {
  7:         try {
  8:             this.sendHeartbeatToAllBroker();
  9:             this.uploadFilterClassSource();
 10:         } catch (final Exception e) {
 11:             log.error("sendHeartbeatToAllBroker exception", e);
 12:         } finally {
 13:             this.lockHeartbeat.unlock();
 14:         }
 15:     } else {
 16:         log.warn("lock heartBeat, but failed.");
 17:     }
 18: }
 19: 
 20: /**
 21:  * 上传过滤类到Filtersrv
 22:  */
 23: private void uploadFilterClassSource() {
 24:     Iterator<Entry<String, MQConsumerInner>> it = this.consumerTable.entrySet().iterator();
 25:     while (it.hasNext()) {
 26:         Entry<String, MQConsumerInner> next = it.next();
 27:         MQConsumerInner consumer = next.getValue();
 28:         if (ConsumeType.CONSUME_PASSIVELY == consumer.consumeType()) {
 29:             Set<SubscriptionData> subscriptions = consumer.subscriptions();
 30:             for (SubscriptionData sub : subscriptions) {
 31:                 if (sub.isClassFilterMode() && sub.getFilterClassSource() != null) {
 32:                     final String consumerGroup = consumer.groupName();
 33:                     final String className = sub.getSubString();
 34:                     final String topic = sub.getTopic();
 35:                     final String filterClassSource = sub.getFilterClassSource();
 36:                     try {
 37:                         this.uploadFilterClassToAllFilterServer(consumerGroup, className, topic, filterClassSource);
 38:                     } catch (Exception e) {
 39:                         log.error("uploadFilterClassToAllFilterServer Exception", e);
 40:                     }
 41:                 }
 42:             }
 43:         }
 44:     }
 45: }
```

## 3.3 Filter 编译 过滤类代码

* 🦅 `Filtersrv` 处理 `Consumer` 上传的 `过滤类代码`，并进行**编译**使用。

核心代码如下：

```Java
  1: // ⬇️⬇️⬇️【FilterClassManager.java】
  2: /**
  3:  * 注册过滤类
  4:  *
  5:  * @param consumerGroup 消费分组
  6:  * @param topic Topic
  7:  * @param className 过滤类名
  8:  * @param classCRC 过滤类源码CRC
  9:  * @param filterSourceBinary 过滤类源码
 10:  * @return 是否注册成功
 11:  */
 12: public boolean registerFilterClass(final String consumerGroup, final String topic,
 13:     final String className, final int classCRC, final byte[] filterSourceBinary) {
 14:     final String key = buildKey(consumerGroup, topic);
 15:     // 判断是否要注册新的过滤类
 16:     boolean registerNew = false;
 17:     FilterClassInfo filterClassInfoPrev = this.filterClassTable.get(key);
 18:     if (null == filterClassInfoPrev) {
 19:         registerNew = true;
 20:     } else {
 21:         if (this.filtersrvController.getFiltersrvConfig().isClientUploadFilterClassEnable()) {
 22:             if (filterClassInfoPrev.getClassCRC() != classCRC && classCRC != 0) { // 类有变化
 23:                 registerNew = true;
 24:             }
 25:         }
 26:     }
 27:     // 注册新的过滤类
 28:     if (registerNew) {
 29:         synchronized (this.compileLock) {
 30:             filterClassInfoPrev = this.filterClassTable.get(key);
 31:             if (null != filterClassInfoPrev && filterClassInfoPrev.getClassCRC() == classCRC) {
 32:                 return true;
 33:             }
 34:             try {
 35:                 FilterClassInfo filterClassInfoNew = new FilterClassInfo();
 36:                 filterClassInfoNew.setClassName(className);
 37:                 filterClassInfoNew.setClassCRC(0);
 38:                 filterClassInfoNew.setMessageFilter(null);
 39: 
 40:                 if (this.filtersrvController.getFiltersrvConfig().isClientUploadFilterClassEnable()) {
 41:                     String javaSource = new String(filterSourceBinary, MixAll.DEFAULT_CHARSET);
 42:                     // 编译新的过滤类
 43:                     Class<?> newClass = DynaCode.compileAndLoadClass(className, javaSource);
 44:                     // 创建新的过滤类对象
 45:                     Object newInstance = newClass.newInstance();
 46:                     filterClassInfoNew.setMessageFilter((MessageFilter) newInstance);
 47:                     filterClassInfoNew.setClassCRC(classCRC);
 48:                 }
 49: 
 50:                 this.filterClassTable.put(key, filterClassInfoNew);
 51:             } catch (Throwable e) {
 52:                 String info = String.format("FilterServer, registerFilterClass Exception, consumerGroup: %s topic: %s className: %s",
 53:                             consumerGroup, topic, className);
 54:                 log.error(info, e);
 55:                 return false;
 56:             }
 57:         }
 58:     }
 59: 
 60:     return true;
 61: }
```

-------

# 4. 过滤消息

![Filtersrv.png](http://www.iocoder.cn/images/RocketMQ/2017_05_17/Filtersrv.png)

## 4.1 Consumer 从 Filtersrv 拉取消息

* 🦅 `Consumer` 拉取 **使用过滤类方式订阅** 的消费消息时，从 `Broker` 对应的 `Filtersrv` 列表**随机**选择一个拉取消息。**如果选择不到 `Filtersrv`，则无法拉取消息。因此，`Filtersrv` 一定要做高可用**。

```Java
  1: // ⬇️⬇️⬇️【PullAPIWrapper.java】
  2: /**
  3:  * 拉取消息核心方法
  4:  *
  5:  * @param mq 消息嘟列
  6:  * @param subExpression 订阅表达式
  7:  * @param subVersion 订阅版本号
  8:  * @param offset 拉取队列开始位置
  9:  * @param maxNums 批量拉 取消息数量
 10:  * @param sysFlag 拉取系统标识
 11:  * @param commitOffset 提交消费进度
 12:  * @param brokerSuspendMaxTimeMillis broker挂起请求最大时间
 13:  * @param timeoutMillis 请求broker超时时间
 14:  * @param communicationMode 通讯模式
 15:  * @param pullCallback 拉取回调
 16:  * @return 拉取消息结果。只有通讯模式为同步时，才返回结果，否则返回null。
 17:  * @throws MQClientException 当寻找不到 broker 时，或发生其他client异常
 18:  * @throws RemotingException 当远程调用发生异常时
 19:  * @throws MQBrokerException 当 broker 发生异常时。只有通讯模式为同步时才会发生该异常。
 20:  * @throws InterruptedException 当发生中断异常时
 21:  */
 22: protected PullResult pullKernelImpl(
 23:     final MessageQueue mq,
 24:     final String subExpression,
 25:     final long subVersion,
 26:     final long offset,
 27:     final int maxNums,
 28:     final int sysFlag,
 29:     final long commitOffset,
 30:     final long brokerSuspendMaxTimeMillis,
 31:     final long timeoutMillis,
 32:     final CommunicationMode communicationMode,
 33:     final PullCallback pullCallback
 34: ) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
 35:     // // ....(省略代码)
 36:     // 请求拉取消息
 37:     if (findBrokerResult != null) {
 38:         // ....(省略代码)
 39:         // 若订阅topic使用过滤类，使用filtersrv获取消息
 40:         String brokerAddr = findBrokerResult.getBrokerAddr();
 41:         if (PullSysFlag.hasClassFilterFlag(sysFlagInner)) {
 42:             brokerAddr = computPullFromWhichFilterServer(mq.getTopic(), brokerAddr);
 43:         }
 44: 
 45:         PullResult pullResult = this.mQClientFactory.getMQClientAPIImpl().pullMessage(
 46:             brokerAddr,
 47:             requestHeader,
 48:             timeoutMillis,
 49:             communicationMode,
 50:             pullCallback);
 51: 
 52:         return pullResult;
 53:     }
 54: 
 55:     // Broker信息不存在，则抛出异常
 56:     throw new MQClientException("The broker[" + mq.getBrokerName() + "] not exist", null);
 57: }
 58: 
 59: /**
 60:  * 计算filtersrv地址。如果有多个filtersrv，随机选择一个。
 61:  *
 62:  * @param topic Topic
 63:  * @param brokerAddr broker地址
 64:  * @return filtersrv地址
 65:  * @throws MQClientException 当filtersrv不存在时
 66:  */
 67: private String computPullFromWhichFilterServer(final String topic, final String brokerAddr)
 68:     throws MQClientException {
 69:     ConcurrentHashMap<String, TopicRouteData> topicRouteTable = this.mQClientFactory.getTopicRouteTable();
 70:     if (topicRouteTable != null) {
 71:         TopicRouteData topicRouteData = topicRouteTable.get(topic);
 72:         List<String> list = topicRouteData.getFilterServerTable().get(brokerAddr);
 73:         if (list != null && !list.isEmpty()) {
 74:             return list.get(randomNum() % list.size());
 75:         }
 76:     }
 77:     throw new MQClientException("Find Filter Server Failed, Broker Addr: " + brokerAddr + " topic: "
 78:         + topic, null);
 79: }
```

## 4.2 Filtersrv 从 Broker 拉取消息

* 🦅 `Filtersrv` 拉取消息后，会建议 `Consumer` 向 `Broker主节点` 拉取消息。
* 🦅 `Filtersrv` 可以理解成一个 `Consumer`，向 `Broker` 拉取消息时，实际使用的 `DefaultMQPullConsumer.java` 的方法和逻辑。

```Java
  1: // ⬇️⬇️⬇️【DefaultRequestProcessor.java】
  2: /**
  3:  * 拉取消息
  4:  *
  5:  * @param ctx 拉取消息context
  6:  * @param request 拉取消息请求
  7:  * @return 响应
  8:  * @throws Exception 当发生异常时
  9:  */
 10: private RemotingCommand pullMessageForward(final ChannelHandlerContext ctx, final RemotingCommand request) throws Exception {
 11:     final RemotingCommand response = RemotingCommand.createResponseCommand(PullMessageResponseHeader.class);
 12:     final PullMessageResponseHeader responseHeader = (PullMessageResponseHeader) response.readCustomHeader();
 13:     final PullMessageRequestHeader requestHeader =
 14:         (PullMessageRequestHeader) request.decodeCommandCustomHeader(PullMessageRequestHeader.class);
 15: 
 16:     final FilterContext filterContext = new FilterContext();
 17:     filterContext.setConsumerGroup(requestHeader.getConsumerGroup());
 18: 
 19:     response.setOpaque(request.getOpaque());
 20: 
 21:     DefaultMQPullConsumer pullConsumer = this.filtersrvController.getDefaultMQPullConsumer();
 22: 
 23:     // 校验Topic过滤类是否完整
 24:     final FilterClassInfo findFilterClass = this.filtersrvController.getFilterClassManager().findFilterClass(requestHeader.getConsumerGroup(), requestHeader.getTopic());
 25:     if (null == findFilterClass) {
 26:         response.setCode(ResponseCode.SYSTEM_ERROR);
 27:         response.setRemark("Find Filter class failed, not registered");
 28:         return response;
 29:     }
 30:     if (null == findFilterClass.getMessageFilter()) {
 31:         response.setCode(ResponseCode.SYSTEM_ERROR);
 32:         response.setRemark("Find Filter class failed, registered but no class");
 33:         return response;
 34:     }
 35: 
 36:     // 设置下次请求从 Broker主节点。
 37:     responseHeader.setSuggestWhichBrokerId(MixAll.MASTER_ID);
 38: 
 39:     MessageQueue mq = new MessageQueue();
 40:     mq.setTopic(requestHeader.getTopic());
 41:     mq.setQueueId(requestHeader.getQueueId());
 42:     mq.setBrokerName(this.filtersrvController.getBrokerName());
 43:     long offset = requestHeader.getQueueOffset();
 44:     int maxNums = requestHeader.getMaxMsgNums();
 45: 
 46:     final PullCallback pullCallback = new PullCallback() {
 47: 
 48:         @Override
 49:         public void onSuccess(PullResult pullResult) {
 50:             responseHeader.setMaxOffset(pullResult.getMaxOffset());
 51:             responseHeader.setMinOffset(pullResult.getMinOffset());
 52:             responseHeader.setNextBeginOffset(pullResult.getNextBeginOffset());
 53:             response.setRemark(null);
 54: 
 55:             switch (pullResult.getPullStatus()) {
 56:                 case FOUND:
 57:                     response.setCode(ResponseCode.SUCCESS);
 58: 
 59:                     List<MessageExt> msgListOK = new ArrayList<MessageExt>();
 60:                     try {
 61:                         for (MessageExt msg : pullResult.getMsgFoundList()) {
 62:                             // 使用过滤类过滤消息
 63:                             boolean match = findFilterClass.getMessageFilter().match(msg, filterContext);
 64:                             if (match) {
 65:                                 msgListOK.add(msg);
 66:                             }
 67:                         }
 68: 
 69:                         if (!msgListOK.isEmpty()) {
 70:                             returnResponse(requestHeader.getConsumerGroup(), requestHeader.getTopic(), ctx, response, msgListOK);
 71:                             return;
 72:                         } else {
 73:                             response.setCode(ResponseCode.PULL_RETRY_IMMEDIATELY);
 74:                         }
 75:                     } catch (Throwable e) {
 76:                         final String error =
 77:                             String.format("do Message Filter Exception, ConsumerGroup: %s Topic: %s ",
 78:                                 requestHeader.getConsumerGroup(), requestHeader.getTopic());
 79:                         log.error(error, e);
 80: 
 81:                         response.setCode(ResponseCode.SYSTEM_ERROR);
 82:                         response.setRemark(error + RemotingHelper.exceptionSimpleDesc(e));
 83:                         returnResponse(requestHeader.getConsumerGroup(), requestHeader.getTopic(), ctx, response, null);
 84:                         return;
 85:                     }
 86: 
 87:                     break;
 88:                 case NO_MATCHED_MSG:
 89:                     response.setCode(ResponseCode.PULL_RETRY_IMMEDIATELY);
 90:                     break;
 91:                 case NO_NEW_MSG:
 92:                     response.setCode(ResponseCode.PULL_NOT_FOUND);
 93:                     break;
 94:                 case OFFSET_ILLEGAL:
 95:                     response.setCode(ResponseCode.PULL_OFFSET_MOVED);
 96:                     break;
 97:                 default:
 98:                     break;
 99:             }
100: 
101:             returnResponse(requestHeader.getConsumerGroup(), requestHeader.getTopic(), ctx, response, null);
102:         }
103: 
104:         @Override
105:         public void onException(Throwable e) {
106:             response.setCode(ResponseCode.SYSTEM_ERROR);
107:             response.setRemark("Pull Callback Exception, " + RemotingHelper.exceptionSimpleDesc(e));
108:             returnResponse(requestHeader.getConsumerGroup(), requestHeader.getTopic(), ctx, response, null);
109:             return;
110:         }
111:     };
112: 
113:     // 拉取消息
114:     pullConsumer.pullBlockIfNotFound(mq, null, offset, maxNums, pullCallback);
115:     return null;
116: }
``` 

# 5. Filtersrv 高可用

![Filtersrv过可用](http://www.iocoder.cn/images/RocketMQ/2017_05_17/02.png)


