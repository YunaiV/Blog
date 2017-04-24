# 1、概述
# 2、Broker 提供[拉取消息]接口

## PullMessageRequestHeader

```Java
  1: public class PullMessageRequestHeader implements CommandCustomHeader {
  2:     /**
  3:      * 消费者分组
  4:      */
  5:     @CFNotNull
  6:     private String consumerGroup;
  7:     /**
  8:      * Topic
  9:      */
 10:     @CFNotNull
 11:     private String topic;
 12:     /**
 13:      * 队列编号
 14:      */
 15:     @CFNotNull
 16:     private Integer queueId;
 17:     /**
 18:      * 队列开始位置
 19:      */
 20:     @CFNotNull
 21:     private Long queueOffset;
 22:     /**
 23:      * 消息数量
 24:      */
 25:     @CFNotNull
 26:     private Integer maxMsgNums;
 27:     /**
 28:      * 系统标识
 29:      */
 30:     @CFNotNull
 31:     private Integer sysFlag;
 32:     /**
 33:      * 提交消费进度位置
 34:      */
 35:     @CFNotNull
 36:     private Long commitOffset;
 37:     /**
 38:      * 挂起超时时间
 39:      */
 40:     @CFNotNull
 41:     private Long suspendTimeoutMillis;
 42:     /**
 43:      * 订阅表达式
 44:      */
 45:     @CFNullable
 46:     private String subscription;
 47:     /**
 48:      * 订阅版本号
 49:      */
 50:     @CFNotNull
 51:     private Long subVersion;
 52: }
```

* 说明：拉取消息请求Header
* topic +  queueId + queueOffset + maxMsgNums
* sysFlag ：系统标识。
    * 第 0 位 `FLAG_COMMIT_OFFSET` ：标记请求提交消费进度位置，和 `commitOffset` 配合。
    * 第 1 位 `FLAG_SUSPEND` ：标记请求是否挂起请求，和 `suspendTimeoutMillis` 配合。当拉取不到消息时， `Broker` 会挂起请求，直到有消息。最大挂起时间：`suspendTimeoutMillis` 毫秒。
    * 第 2 位 `FLAG_SUBSCRIPTION` ：是否过滤订阅表达式，和 `subscription` 配置。
* subVersion ：订阅版本号。请求时，如果版本号不对，则无法拉取到消息，需要重新获取订阅信息，使用最新的订阅版本号。

## PullMessageProcessor#processRequest(...)

```Java
  1: private RemotingCommand processRequest(final Channel channel, RemotingCommand request, boolean brokerAllowSuspend)
  2:     throws RemotingCommandException {
  3:     RemotingCommand response = RemotingCommand.createResponseCommand(PullMessageResponseHeader.class);
  4:     final PullMessageResponseHeader responseHeader = (PullMessageResponseHeader) response.readCustomHeader();
  5:     final PullMessageRequestHeader requestHeader =
  6:         (PullMessageRequestHeader) request.decodeCommandCustomHeader(PullMessageRequestHeader.class);
  7: 
  8:     response.setOpaque(request.getOpaque());
  9: 
 10:     if (LOG.isDebugEnabled()) {
 11:         LOG.debug("receive PullMessage request command, {}", request);
 12:     }
 13: 
 14:     // 校验 broker 是否可读
 15:     if (!PermName.isReadable(this.brokerController.getBrokerConfig().getBrokerPermission())) {
 16:         response.setCode(ResponseCode.NO_PERMISSION);
 17:         response.setRemark(String.format("the broker[%s] pulling message is forbidden", this.brokerController.getBrokerConfig().getBrokerIP1()));
 18:         return response;
 19:     }
 20: 
 21:     // 校验 consumer分组配置 是否存在
 22:     SubscriptionGroupConfig subscriptionGroupConfig = this.brokerController.getSubscriptionGroupManager().findSubscriptionGroupConfig(requestHeader.getConsumerGroup());
 23:     if (null == subscriptionGroupConfig) {
 24:         response.setCode(ResponseCode.SUBSCRIPTION_GROUP_NOT_EXIST);
 25:         response.setRemark(String.format("subscription group [%s] does not exist, %s", requestHeader.getConsumerGroup(), FAQUrl.suggestTodo(FAQUrl.SUBSCRIPTION_GROUP_NOT_EXIST)));
 26:         return response;
 27:     }
 28:     // 校验 consumer分组配置 是否可消费
 29:     if (!subscriptionGroupConfig.isConsumeEnable()) {
 30:         response.setCode(ResponseCode.NO_PERMISSION);
 31:         response.setRemark("subscription group no permission, " + requestHeader.getConsumerGroup());
 32:         return response;
 33:     }
 34: 
 35:     final boolean hasSuspendFlag = PullSysFlag.hasSuspendFlag(requestHeader.getSysFlag()); // 是否挂起请求，当没有消息时
 36:     final boolean hasCommitOffsetFlag = PullSysFlag.hasCommitOffsetFlag(requestHeader.getSysFlag()); // 是否提交消费进度
 37:     final boolean hasSubscriptionFlag = PullSysFlag.hasSubscriptionFlag(requestHeader.getSysFlag()); // 是否过滤订阅表达式(subscription)
 38:     final long suspendTimeoutMillisLong = hasSuspendFlag ? requestHeader.getSuspendTimeoutMillis() : 0; // 挂起请求超时时长
 39: 
 40:     // 校验 topic配置 存在
 41:     TopicConfig topicConfig = this.brokerController.getTopicConfigManager().selectTopicConfig(requestHeader.getTopic());
 42:     if (null == topicConfig) {
 43:         LOG.error("The topic {} not exist, consumer: {} ", requestHeader.getTopic(), RemotingHelper.parseChannelRemoteAddr(channel));
 44:         response.setCode(ResponseCode.TOPIC_NOT_EXIST);
 45:         response.setRemark(String.format("topic[%s] not exist, apply first please! %s", requestHeader.getTopic(), FAQUrl.suggestTodo(FAQUrl.APPLY_TOPIC_URL)));
 46:         return response;
 47:     }
 48:     // 校验 topic配置 权限可读
 49:     if (!PermName.isReadable(topicConfig.getPerm())) {
 50:         response.setCode(ResponseCode.NO_PERMISSION);
 51:         response.setRemark("the topic[" + requestHeader.getTopic() + "] pulling message is forbidden");
 52:         return response;
 53:     }
 54:     // 校验 读取队列 在 topic配置 队列范围内
 55:     if (requestHeader.getQueueId() < 0 || requestHeader.getQueueId() >= topicConfig.getReadQueueNums()) {
 56:         String errorInfo = String.format("queueId[%d] is illegal, topic:[%s] topicConfig.readQueueNums:[%d] consumer:[%s]",
 57:                 requestHeader.getQueueId(), requestHeader.getTopic(), topicConfig.getReadQueueNums(), channel.remoteAddress());
 58:         LOG.warn(errorInfo);
 59:         response.setCode(ResponseCode.SYSTEM_ERROR);
 60:         response.setRemark(errorInfo);
 61:         return response;
 62:     }
 63: 
 64:     // 校验 订阅关系
 65:     SubscriptionData subscriptionData;
 66:     if (hasSubscriptionFlag) {
 67:         try {
 68:             subscriptionData = FilterAPI.buildSubscriptionData(requestHeader.getConsumerGroup(), requestHeader.getTopic(),
 69:                 requestHeader.getSubscription());
 70:         } catch (Exception e) {
 71:             LOG.warn("Parse the consumer's subscription[{}] failed, group: {}", requestHeader.getSubscription(), //
 72:                     requestHeader.getConsumerGroup());
 73:             response.setCode(ResponseCode.SUBSCRIPTION_PARSE_FAILED);
 74:             response.setRemark("parse the consumer's subscription failed");
 75:             return response;
 76:         }
 77:     } else {
 78:         // 校验 消费分组信息 是否存在
 79:         ConsumerGroupInfo consumerGroupInfo = this.brokerController.getConsumerManager().getConsumerGroupInfo(requestHeader.getConsumerGroup());
 80:         if (null == consumerGroupInfo) {
 81:             LOG.warn("The consumer's group info not exist, group: {}", requestHeader.getConsumerGroup());
 82:             response.setCode(ResponseCode.SUBSCRIPTION_NOT_EXIST);
 83:             response.setRemark("the consumer's group info not exist" + FAQUrl.suggestTodo(FAQUrl.SAME_GROUP_DIFFERENT_TOPIC));
 84:             return response;
 85:         }
 86:         // 校验 消费分组信息 消息模型是否匹配
 87:         if (!subscriptionGroupConfig.isConsumeBroadcastEnable() //
 88:             && consumerGroupInfo.getMessageModel() == MessageModel.BROADCASTING) {
 89:             response.setCode(ResponseCode.NO_PERMISSION);
 90:             response.setRemark("the consumer group[" + requestHeader.getConsumerGroup() + "] can not consume by broadcast way");
 91:             return response;
 92:         }
 93: 
 94:         // 校验 订阅信息 是否存在
 95:         subscriptionData = consumerGroupInfo.findSubscriptionData(requestHeader.getTopic());
 96:         if (null == subscriptionData) {
 97:             LOG.warn("The consumer's subscription not exist, group: {}, topic:{}", requestHeader.getConsumerGroup(), requestHeader.getTopic());
 98:             response.setCode(ResponseCode.SUBSCRIPTION_NOT_EXIST);
 99:             response.setRemark("the consumer's subscription not exist" + FAQUrl.suggestTodo(FAQUrl.SAME_GROUP_DIFFERENT_TOPIC));
100:             return response;
101:         }
102:         // 校验 订阅信息版本 是否合法
103:         if (subscriptionData.getSubVersion() < requestHeader.getSubVersion()) {
104:             LOG.warn("The broker's subscription is not latest, group: {} {}", requestHeader.getConsumerGroup(),
105:                     subscriptionData.getSubString());
106:             response.setCode(ResponseCode.SUBSCRIPTION_NOT_LATEST);
107:             response.setRemark("the consumer's subscription not latest");
108:             return response;
109:         }
110:     }
111: 
112:     // 获取消息
113:     final GetMessageResult getMessageResult = this.brokerController.getMessageStore().getMessage(requestHeader.getConsumerGroup(), requestHeader.getTopic(),
114:             requestHeader.getQueueId(), requestHeader.getQueueOffset(), requestHeader.getMaxMsgNums(), subscriptionData);
115:     if (getMessageResult != null) {
116:         response.setRemark(getMessageResult.getStatus().name());
117:         responseHeader.setNextBeginOffset(getMessageResult.getNextBeginOffset());
118:         responseHeader.setMinOffset(getMessageResult.getMinOffset());
119:         responseHeader.setMaxOffset(getMessageResult.getMaxOffset());
120: 
121:         // TODO 待读
122:         // 计算建议读取brokerId
123:         if (getMessageResult.isSuggestPullingFromSlave()) {
124:             responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.getWhichBrokerWhenConsumeSlowly());
125:         } else {
126:             responseHeader.setSuggestWhichBrokerId(MixAll.MASTER_ID);
127:         }
128: 
129:         switch (this.brokerController.getMessageStoreConfig().getBrokerRole()) {
130:             case ASYNC_MASTER:
131:             case SYNC_MASTER:
132:                 break;
133:             case SLAVE:
134:                 if (!this.brokerController.getBrokerConfig().isSlaveReadEnable()) { // 从节点不允许读取，告诉consumer读取主节点。
135:                     response.setCode(ResponseCode.PULL_RETRY_IMMEDIATELY);
136:                     responseHeader.setSuggestWhichBrokerId(MixAll.MASTER_ID);
137:                 }
138:                 break;
139:         }
140: 
141:         if (this.brokerController.getBrokerConfig().isSlaveReadEnable()) {
142:             // consume too slow ,redirect to another machine
143:             if (getMessageResult.isSuggestPullingFromSlave()) {
144:                 responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.getWhichBrokerWhenConsumeSlowly());
145:             }
146:             // consume ok
147:             else {
148:                 responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.getBrokerId());
149:             }
150:         } else {
151:             responseHeader.setSuggestWhichBrokerId(MixAll.MASTER_ID);
152:         }
153: 
154:         switch (getMessageResult.getStatus()) {
155:             case FOUND:
156:                 response.setCode(ResponseCode.SUCCESS);
157:                 break;
158:             case MESSAGE_WAS_REMOVING:
159:                 response.setCode(ResponseCode.PULL_RETRY_IMMEDIATELY);
160:                 break;
161:             case NO_MATCHED_LOGIC_QUEUE: // TODO 待补充博客
162:             case NO_MESSAGE_IN_QUEUE: // TODO 待补充博客
163:                 if (0 != requestHeader.getQueueOffset()) {
164:                     response.setCode(ResponseCode.PULL_OFFSET_MOVED);
165: 
166:                     // XXX: warn and notify me
167:                     LOG.info("the broker store no queue data, fix the request offset {} to {}, Topic: {} QueueId: {} Consumer Group: {}", //
168:                         requestHeader.getQueueOffset(), //
169:                         getMessageResult.getNextBeginOffset(), //
170:                         requestHeader.getTopic(), //
171:                         requestHeader.getQueueId(), //
172:                         requestHeader.getConsumerGroup()//
173:                     );
174:                 } else {
175:                     response.setCode(ResponseCode.PULL_NOT_FOUND);
176:                 }
177:                 break;
178:             case NO_MATCHED_MESSAGE:
179:                 response.setCode(ResponseCode.PULL_RETRY_IMMEDIATELY);
180:                 break;
181:             case OFFSET_FOUND_NULL:
182:                 response.setCode(ResponseCode.PULL_NOT_FOUND);
183:                 break;
184:             case OFFSET_OVERFLOW_BADLY: // TODO 待补充博客
185:                 response.setCode(ResponseCode.PULL_OFFSET_MOVED);
186:                 // XXX: warn and notify me
187:                 LOG.info("The request offset:{} over flow badly, broker max offset:{} , consumer: {}", requestHeader.getQueueOffset(), getMessageResult.getMaxOffset(), channel.remoteAddress());
188:                 break;
189:             case OFFSET_OVERFLOW_ONE: // TODO 待补充博客
190:                 response.setCode(ResponseCode.PULL_NOT_FOUND);
191:                 break;
192:             case OFFSET_TOO_SMALL: // TODO 待补充博客
193:                 response.setCode(ResponseCode.PULL_OFFSET_MOVED);
194:                 LOG.info("The request offset is too small. group={}, topic={}, requestOffset={}, brokerMinOffset={}, clientIp={}",
195:                     requestHeader.getConsumerGroup(), requestHeader.getTopic(), requestHeader.getQueueOffset(),
196:                     getMessageResult.getMinOffset(), channel.remoteAddress());
197:                 break;
198:             default:
199:                 assert false;
200:                 break;
201:         }
202: 
203:         // hook：before
204:         if (this.hasConsumeMessageHook()) {
205:             ConsumeMessageContext context = new ConsumeMessageContext();
206:             context.setConsumerGroup(requestHeader.getConsumerGroup());
207:             context.setTopic(requestHeader.getTopic());
208:             context.setQueueId(requestHeader.getQueueId());
209: 
210:             String owner = request.getExtFields().get(BrokerStatsManager.COMMERCIAL_OWNER);
211: 
212:             switch (response.getCode()) {
213:                 case ResponseCode.SUCCESS:
214:                     int commercialBaseCount = brokerController.getBrokerConfig().getCommercialBaseCount();
215:                     int incValue = getMessageResult.getMsgCount4Commercial() * commercialBaseCount;
216: 
217:                     context.setCommercialRcvStats(BrokerStatsManager.StatsType.RCV_SUCCESS);
218:                     context.setCommercialRcvTimes(incValue);
219:                     context.setCommercialRcvSize(getMessageResult.getBufferTotalSize());
220:                     context.setCommercialOwner(owner);
221: 
222:                     break;
223:                 case ResponseCode.PULL_NOT_FOUND:
224:                     if (!brokerAllowSuspend) {
225: 
226:                         context.setCommercialRcvStats(BrokerStatsManager.StatsType.RCV_EPOLLS);
227:                         context.setCommercialRcvTimes(1);
228:                         context.setCommercialOwner(owner);
229: 
230:                     }
231:                     break;
232:                 case ResponseCode.PULL_RETRY_IMMEDIATELY:
233:                 case ResponseCode.PULL_OFFSET_MOVED:
234:                     context.setCommercialRcvStats(BrokerStatsManager.StatsType.RCV_EPOLLS);
235:                     context.setCommercialRcvTimes(1);
236:                     context.setCommercialOwner(owner);
237:                     break;
238:                 default:
239:                     assert false;
240:                     break;
241:             }
242: 
243:             this.executeConsumeMessageHookBefore(context);
244:         }
245: 
246:         switch (response.getCode()) {
247:             case ResponseCode.SUCCESS:
248: 
249:                 this.brokerController.getBrokerStatsManager().incGroupGetNums(requestHeader.getConsumerGroup(), requestHeader.getTopic(),
250:                     getMessageResult.getMessageCount());
251:                 this.brokerController.getBrokerStatsManager().incGroupGetSize(requestHeader.getConsumerGroup(), requestHeader.getTopic(),
252:                     getMessageResult.getBufferTotalSize());
253:                 this.brokerController.getBrokerStatsManager().incBrokerGetNums(getMessageResult.getMessageCount());
254:                 // 读取消息
255:                 if (this.brokerController.getBrokerConfig().isTransferMsgByHeap()) { // 内存中
256:                     final long beginTimeMills = this.brokerController.getMessageStore().now();
257: 
258:                     final byte[] r = this.readGetMessageResult(getMessageResult, requestHeader.getConsumerGroup(), requestHeader.getTopic(), requestHeader.getQueueId());
259: 
260:                     this.brokerController.getBrokerStatsManager().incGroupGetLatency(requestHeader.getConsumerGroup(),
261:                         requestHeader.getTopic(), requestHeader.getQueueId(),
262:                         (int) (this.brokerController.getMessageStore().now() - beginTimeMills));
263:                     response.setBody(r);
264:                 } else { // zero-copy
265:                     try {
266:                         FileRegion fileRegion = new ManyMessageTransfer(response.encodeHeader(getMessageResult.getBufferTotalSize()), getMessageResult);
267:                         channel.writeAndFlush(fileRegion).addListener(new ChannelFutureListener() {
268:                             @Override
269:                             public void operationComplete(ChannelFuture future) throws Exception {
270:                                 getMessageResult.release();
271:                                 if (!future.isSuccess()) {
272:                                     LOG.error("Fail to transfer messages from page cache to {}", channel.remoteAddress(), future.cause());
273:                                 }
274:                             }
275:                         });
276:                     } catch (Throwable e) {
277:                         LOG.error("Error occurred when transferring messages from page cache", e);
278:                         getMessageResult.release();
279:                     }
280: 
281:                     response = null;
282:                 }
283:                 break;
284:             case ResponseCode.PULL_NOT_FOUND:
285:                 // 消息未查询到 && broker允许挂起请求 && 请求允许挂起
286:                 if (brokerAllowSuspend && hasSuspendFlag) {
287:                     long pollingTimeMills = suspendTimeoutMillisLong;
288:                     if (!this.brokerController.getBrokerConfig().isLongPollingEnable()) {
289:                         pollingTimeMills = this.brokerController.getBrokerConfig().getShortPollingTimeMills();
290:                     }
291: 
292:                     String topic = requestHeader.getTopic();
293:                     long offset = requestHeader.getQueueOffset();
294:                     int queueId = requestHeader.getQueueId();
295:                     PullRequest pullRequest = new PullRequest(request, channel, pollingTimeMills,
296:                         this.brokerController.getMessageStore().now(), offset, subscriptionData);
297:                     this.brokerController.getPullRequestHoldService().suspendPullRequest(topic, queueId, pullRequest);
298:                     response = null;
299:                     break;
300:                 }
301: 
302:             case ResponseCode.PULL_RETRY_IMMEDIATELY:
303:                 break;
304:             case ResponseCode.PULL_OFFSET_MOVED:
305:                 if (this.brokerController.getMessageStoreConfig().getBrokerRole() != BrokerRole.SLAVE
306:                     || this.brokerController.getMessageStoreConfig().isOffsetCheckInSlave()) { // TODO 待博客补充
307:                     MessageQueue mq = new MessageQueue();
308:                     mq.setTopic(requestHeader.getTopic());
309:                     mq.setQueueId(requestHeader.getQueueId());
310:                     mq.setBrokerName(this.brokerController.getBrokerConfig().getBrokerName());
311: 
312:                     OffsetMovedEvent event = new OffsetMovedEvent();
313:                     event.setConsumerGroup(requestHeader.getConsumerGroup());
314:                     event.setMessageQueue(mq);
315:                     event.setOffsetRequest(requestHeader.getQueueOffset());
316:                     event.setOffsetNew(getMessageResult.getNextBeginOffset());
317:                     this.generateOffsetMovedEvent(event);
318:                     LOG.warn(
319:                         "PULL_OFFSET_MOVED:correction offset. topic={}, groupId={}, requestOffset={}, newOffset={}, suggestBrokerId={}",
320:                         requestHeader.getTopic(), requestHeader.getConsumerGroup(), event.getOffsetRequest(), event.getOffsetNew(),
321:                         responseHeader.getSuggestWhichBrokerId());
322:                 } else {
323:                     responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.getBrokerId());
324:                     response.setCode(ResponseCode.PULL_RETRY_IMMEDIATELY);
325:                     LOG.warn("PULL_OFFSET_MOVED:none correction. topic={}, groupId={}, requestOffset={}, suggestBrokerId={}",
326:                         requestHeader.getTopic(), requestHeader.getConsumerGroup(), requestHeader.getQueueOffset(),
327:                         responseHeader.getSuggestWhichBrokerId());
328:                 }
329: 
330:                 break;
331:             default:
332:                 assert false;
333:         }
334:     } else {
335:         response.setCode(ResponseCode.SYSTEM_ERROR);
336:         response.setRemark("store getMessage return null");
337:     }
338: 
339:     // 请求要求持久化进度 && broker非主，进行持久化进度。
340:     boolean storeOffsetEnable = brokerAllowSuspend;
341:     storeOffsetEnable = storeOffsetEnable && hasCommitOffsetFlag;
342:     storeOffsetEnable = storeOffsetEnable && this.brokerController.getMessageStoreConfig().getBrokerRole() != BrokerRole.SLAVE;
343:     if (storeOffsetEnable) {
344:         this.brokerController.getConsumerOffsetManager().commitOffset(RemotingHelper.parseChannelRemoteAddr(channel),
345:             requestHeader.getConsumerGroup(), requestHeader.getTopic(), requestHeader.getQueueId(), requestHeader.getCommitOffset());
346:     }
347:     return response;
348: }
```

# 3、Broker 提供[更新消费进度]接口
# 4、Broker 提供[发回消息]接口
# 5、Consumer 调用[拉取消息]接口
# 6、Consumer 消费消息
# 7、Consumer 调用[发回消息]接口
# 8、Consumer 调用[更新消费进度]接口

