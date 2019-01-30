title: Apollo 源码解析 —— Admin Service 发送 ReleaseMessage
date: 2018-04-05
tags:
categories: Apollo
permalink: Apollo/admin-server-send-release-message
wechat_url:
toutiao_url: https://www.toutiao.com/i6643383570985386509/

---

摘要: 原创出处 http://www.iocoder.cn/Apollo/admin-server-send-release-message/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Apollo/admin-server-send-release-message/)
- [2. ReleaseMessage](http://www.iocoder.cn/Apollo/admin-server-send-release-message/)
  - [2.1 ReleaseMessageKeyGenerator](http://www.iocoder.cn/Apollo/admin-server-send-release-message/)
  - [2.2 ReleaseMessageRepository](http://www.iocoder.cn/Apollo/admin-server-send-release-message/)
- [3. MessageSender](http://www.iocoder.cn/Apollo/admin-server-send-release-message/)
  - [3.1 发布配置](http://www.iocoder.cn/Apollo/admin-server-send-release-message/)
  - [3.2 Topics](http://www.iocoder.cn/Apollo/admin-server-send-release-message/)
  - [3.3 DatabaseMessageSender](http://www.iocoder.cn/Apollo/admin-server-send-release-message/)
- [4. ReleaseMessageListener](http://www.iocoder.cn/Apollo/admin-server-send-release-message/)
  - [4.1 ReleaseMessageScanner](http://www.iocoder.cn/Apollo/admin-server-send-release-message/)
- [666. 彩蛋](http://www.iocoder.cn/Apollo/admin-server-send-release-message/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2018_05_18.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

# 1. 概述

> 老艿艿：本系列假定胖友已经阅读过 [《Apollo 官方 wiki 文档》](https://github.com/ctripcorp/apollo/wiki/)  。

本文接 [《Apollo 源码解析 —— Portal 发布配置》](http://www.iocoder.cn/Apollo/portal-publish/?self) 一文，分享配置发布的第三步，**Admin Service 发布配置后，发送 ReleaseMessage 给各个Config Service** 。

> FROM [《Apollo配置中心设计》](https://github.com/ctripcorp/apollo/wiki/Apollo%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83%E8%AE%BE%E8%AE%A1#211-%E5%8F%91%E9%80%81releasemessage%E7%9A%84%E5%AE%9E%E7%8E%B0%E6%96%B9%E5%BC%8F) 的 [2.1.1 发送ReleaseMessage的实现方式](#)
> 
> Admin Service 在配置发布后，需要通知所有的 Config Service 有配置发布，从而 Config Service 可以通知对应的客户端来拉取最新的配置。
> 
>从概念上来看，这是一个典型的**消息使用场景**，Admin Service 作为 **producer** 发出消息，各个Config Service 作为 **consumer** 消费消息。通过一个**消息组件**（Message Queue）就能很好的实现 Admin Service 和 Config Service 的解耦。
>
> 在实现上，考虑到 Apollo 的实际使用场景，以及为了**尽可能减少外部依赖**，我们没有采用外部的消息中间件，而是通过**数据库实现了一个简单的消息队列**。

实现方式如下：

> 1. Admin Service 在配置发布后会往 ReleaseMessage 表插入一条消息记录，消息内容就是配置发布的 AppId+Cluster+Namespace ，参见 DatabaseMessageSender 。
> 2. Config Service 有一个线程会每秒扫描一次 ReleaseMessage 表，看看是否有新的消息记录，参见 ReleaseMessageScanner 。
> 3. Config Service 如果发现有新的消息记录，那么就会通知到所有的消息监听器（ReleaseMessageListener），如 NotificationControllerV2 ，消息监听器的注册过程参见 ConfigServiceAutoConfiguration 。
> 4. NotificationControllerV2 得到配置发布的 **AppId+Cluster+Namespace** 后，会通知对应的客户端。

示意图如下：

> ![流程](http://www.iocoder.cn/images/Apollo/2018_04_05/01.png)

本文分享第 **1 + 2 + 3** 步骤，在 `apollo-biz` 项目的 `message` 模块实现。😏 第 4 步，我们在下一篇文章分享。

# 2. ReleaseMessage

`com.ctrip.framework.apollo.biz.entity.ReleaseMessage` ，**不继承** BaseEntity 抽象类，ReleaseMessage **实体**。代码如下：

```Java
@Entity
@Table(name = "ReleaseMessage")
public class ReleaseMessage {

    /**
     * 编号
     */
    @Id
    @GeneratedValue
    @Column(name = "Id")
    private long id;
    /**
     * 消息内容，通过 {@link com.ctrip.framework.apollo.biz.utils.ReleaseMessageKeyGenerator#generate(String, String, String)} 方法生成。
     */
    @Column(name = "Message", nullable = false)
    private String message;
    /**
     * 最后更新时间
     */
    @Column(name = "DataChange_LastTime")
    private Date dataChangeLastModifiedTime;
 
    @PrePersist
    protected void prePersist() {
        if (this.dataChangeLastModifiedTime == null) {
            dataChangeLastModifiedTime = new Date();
        }
    }   
}
```

* `id` 字段，编号，**自增**。
* `message` 字段，消息内容。通过 **ReleaseMessageKeyGenerator** 生成。胖友先跳到 [「2.1 ReleaseMessageKeyGenerator」](#) 看看具体实现。
* `#dataChangeLastModifiedTime` 字段，最后更新时间。
    * `#prePersist()` 方法，若保存时，未设置该字段，进行补全。

## 2.1 ReleaseMessageKeyGenerator

`com.ctrip.framework.apollo.biz.utils.ReleaseMessageKeyGenerator` ，ReleaseMessage **消息内容**( `ReleaseMessage.message` )生成器。代码如下：

```Java
public class ReleaseMessageKeyGenerator {

    private static final Joiner STRING_JOINER = Joiner.on(ConfigConsts.CLUSTER_NAMESPACE_SEPARATOR);

    public static String generate(String appId, String cluster, String namespace) {
        return STRING_JOINER.join(appId, cluster, namespace);
    }

}
```

`#generate(...)` 方法，将 `appId` + `cluster` + `namespace` 拼接，使用 `ConfigConsts.CLUSTER_NAMESPACE_SEPARATOR = "+"` 作为**间隔**，例如：`"test+default+application"` 。

因此，对于同一个 Namespace ，生成的**消息内容**是**相同**的。通过这样的方式，我们可以使用最新的 `ReleaseMessage` 的 **`id`** 属性，作为 Namespace 是否发生变更的标识。而 Apollo 确实是通过这样的方式实现，Client 通过不断使用**获得到 `ReleaseMessage` 的 `id` 属性**作为**版本号**，请求 Config Service 判断是否**配置**发生了变化。🙂 这里胖友先留有一个印象，后面我们会再详细介绍这个流程。

正因为，ReleaseMessage 设计的意图是作为配置发生变化的通知，所以对于同一个 Namespace ，仅需要保留**其最新的** ReleaseMessage 记录即可。所以，在 [「3.3 DatabaseMessageSender」](#) 中，我们会看到，有后台任务不断清理**旧的** ReleaseMessage 记录。

## 2.2 ReleaseMessageRepository

`com.ctrip.framework.apollo.biz.repository.ReleaseMessageRepository` ，继承 `org.springframework.data.repository.PagingAndSortingRepository` 接口，提供 ReleaseMessage 的**数据访问** 给 Admin Service 和 Config Service 。代码如下：

```Java
public interface ReleaseMessageRepository extends PagingAndSortingRepository<ReleaseMessage, Long> {

    List<ReleaseMessage> findFirst500ByIdGreaterThanOrderByIdAsc(Long id);

    ReleaseMessage findTopByOrderByIdDesc();

    ReleaseMessage findTopByMessageInOrderByIdDesc(Collection<String> messages);

    List<ReleaseMessage> findFirst100ByMessageAndIdLessThanOrderByIdAsc(String message, Long id);

    @Query("select message, max(id) as id from ReleaseMessage where message in :messages group by message")
    List<Object[]> findLatestReleaseMessagesGroupByMessages(@Param("messages") Collection<String> messages);

}
```

# 3. MessageSender

`com.ctrip.framework.apollo.biz.message.MessageSender` ，Message **发送者**接口。代码如下：

```Java
public interface MessageSender {

    /**
     * 发送 Message
     *
     * @param message 消息
     * @param channel 通道（主题）
     */
    void sendMessage(String message, String channel);

}
```

## 3.1 发布配置

在 ReleaseController 的 `#publish(...)` 方法中，会调用 `MessageSender#sendMessage(message, channel)` 方法，发送 Message 。**调用**简化代码如下：

```Java
// send release message
// 获得 Cluster 名
Namespace parentNamespace = namespaceService.findParentNamespace(namespace);
String messageCluster;
if (parentNamespace != null) { //  有父 Namespace ，说明是灰度发布，使用父 Namespace 的集群名
    messageCluster = parentNamespace.getClusterName();
} else {
    messageCluster = clusterName; // 使用请求的 ClusterName
}
// 发送 Release 消息
messageSender.sendMessage(ReleaseMessageKeyGenerator.generate(appId, messageCluster, namespaceName), Topics.APOLLO_RELEASE_TOPIC);
```

* 关于**父** Namespace 部分的代码，胖友看完**灰度发布**的内容，再回过头理解。
* `ReleaseMessageKeyGenerator#generate(appId, clusterName, namespaceName)` 方法，生成 ReleaseMessage 的**消息内容**。
* 使用 **Topic** 为 `Topics.APOLLO_RELEASE_TOPIC` 。

## 3.2 Topics

`com.ctrip.framework.apollo.biz.message.Topics` ，Topic **枚举**。代码如下：

```Java
public class Topics {

    /**
     * Apollo 配置发布 Topic
     */
    public static final String APOLLO_RELEASE_TOPIC = "apollo-release";

}
```

## 3.3 DatabaseMessageSender

`com.ctrip.framework.apollo.biz.message.DatabaseMessageSender` ，实现 MessageSender 接口，Message 发送者**实现类**，基于**数据库**实现。

### 3.3.1 构造方法

```Java
/**
 * 清理 Message 队列 最大容量
 */
private static final int CLEAN_QUEUE_MAX_SIZE = 100;

/**
 * 清理 Message 队列
 */
private BlockingQueue<Long> toClean = Queues.newLinkedBlockingQueue(CLEAN_QUEUE_MAX_SIZE);
/**
 * 清理 Message ExecutorService
 */
private final ExecutorService cleanExecutorService;
/**
 * 是否停止清理 Message 标识
 */
private final AtomicBoolean cleanStopped;

@Autowired
private ReleaseMessageRepository releaseMessageRepository;

public DatabaseMessageSender() {
    // 创建 ExecutorService 对象
    cleanExecutorService = Executors.newSingleThreadExecutor(ApolloThreadFactory.create("DatabaseMessageSender", true));
    // 设置 cleanStopped 为 false 
    cleanStopped = new AtomicBoolean(false);
}
```

* 主要和**清理** ReleaseMessage 相关的属性。

### 3.3.2 sendMessage

```Java
  1: @Override
  2: @Transactional
  3: public void sendMessage(String message, String channel) {
  4:     logger.info("Sending message {} to channel {}", message, channel);
  5:     // 仅允许发送 APOLLO_RELEASE_TOPIC
  6:     if (!Objects.equals(channel, Topics.APOLLO_RELEASE_TOPIC)) {
  7:         logger.warn("Channel {} not supported by DatabaseMessageSender!");
  8:         return;
  9:     }
 10:     // 【TODO 6001】Tracer 日志
 11:     Tracer.logEvent("Apollo.AdminService.ReleaseMessage", message);
 12:     // 【TODO 6001】Tracer 日志
 13:     Transaction transaction = Tracer.newTransaction("Apollo.AdminService", "sendMessage");
 14:     try {
 15:         // 保存 ReleaseMessage 对象
 16:         ReleaseMessage newMessage = releaseMessageRepository.save(new ReleaseMessage(message));
 17:         // 添加到清理 Message 队列。若队列已满，添加失败，不阻塞等待。
 18:         toClean.offer(newMessage.getId());
 19:         // 【TODO 6001】Tracer 日志
 20:         transaction.setStatus(Transaction.SUCCESS);
 21:     } catch (Throwable ex) {
 22:         // 【TODO 6001】Tracer 日志
 23:         logger.error("Sending message to database failed", ex);
 24:         transaction.setStatus(ex);
 25:         throw ex;
 26:     } finally {
 27:         // 【TODO 6001】Tracer 日志
 28:         transaction.complete();
 29:     }
 30: }
```

* 第 5 至 9 行：第 5 至 9 行：仅**允许**发送 APOLLO_RELEASE_TOPIC 。
* 第 16 行：调用 `ReleaseMessageRepository#save(ReleaseMessage)` 方法，保存 ReleaseMessage 对象。
* 第 18 行：调用 `toClean#offer(Long id)` 方法，添加到清理 Message 队列。**若队列已满，添加失败，不阻塞等待**。
    * 关于  BlockingQueue 的知识，胖友可以看看 [《阻塞队列（BlockingQueue）》](http://jiangzhengjun.iteye.com/blog/683593) 。

### 3.3.3 清理 ReleaseMessage 任务

`#initialize()` 方法，通知 Spring 调用，初始化**清理 ReleaseMessage 任务**。代码如下：

```Java
  1: @PostConstruct
  2: private void initialize() {
  3:     cleanExecutorService.submit(() -> {
  4:         // 若未停止，持续运行。
  5:         while (!cleanStopped.get() && !Thread.currentThread().isInterrupted()) {
  6:             try {
  7:                 // 拉取
  8:                 Long rm = toClean.poll(1, TimeUnit.SECONDS);
  9:                 // 队列非空，处理拉取到的消息
 10:                 if (rm != null) {
 11:                     cleanMessage(rm);
 12:                 // 队列为空，sleep ，避免空跑，占用 CPU
 13:                 } else {
 14:                     TimeUnit.SECONDS.sleep(5);
 15:                 }
 16:             } catch (Throwable ex) {
 17:                 // 【TODO 6001】Tracer 日志
 18:                 Tracer.logError(ex);
 19:             }
 20:         }
 21:     });
 22: }
```

* 第 3 至 21 行：调用 `ExecutorService#submit(Runnable)` 方法，提交**清理 ReleaseMessage 任务**
    * 第 5 行：**循环**，直到停止。
    * 第 8 行：调用 `BlockingQueue#poll(long timeout, TimeUnit unit)` 方法，拉取**队头**的**消息编号**。
        *  第 10 至 11 行：若拉取到消息编号，调用 `#cleanMessage(Long id)` 方法，处理拉取到的消息，即**清理老消息们**。
        *  第 13 至 15 行：若**未**拉取到消息编号，说明队列为**空**，**sleep** ，避免空跑，占用 CPU 。

-------

`#cleanMessage(Long id)` 方法，清理老消息们。代码如下：

```Java
  1: private void cleanMessage(Long id) {
  2:     boolean hasMore = true;
  3:     // 查询对应的 ReleaseMessage 对象，避免已经删除。因为，DatabaseMessageSender 会在多进程中执行。例如：1）Config Service + Admin Service ；2）N * Config Service ；3）N * Admin Service
  4:     // double check in case the release message is rolled back
  5:     ReleaseMessage releaseMessage = releaseMessageRepository.findOne(id);
  6:     if (releaseMessage == null) {
  7:         return;
  8:     }
  9:     // 循环删除相同消息内容( `message` )的老消息
 10:     while (hasMore && !Thread.currentThread().isInterrupted()) {
 11:         // 拉取相同消息内容的 100 条的老消息
 12:         // 老消息的定义：比当前消息编号小，即先发送的
 13:         // 按照 id 升序
 14:         List<ReleaseMessage> messages = releaseMessageRepository.findFirst100ByMessageAndIdLessThanOrderByIdAsc(
 15:                 releaseMessage.getMessage(), releaseMessage.getId());
 16:         // 删除老消息
 17:         releaseMessageRepository.delete(messages);
 18:         // 若拉取不足 100 条，说明无老消息了
 19:         hasMore = messages.size() == 100;
 20:         // 【TODO 6001】Tracer 日志
 21:         messages.forEach(toRemove -> Tracer.logEvent(
 22:                 String.format("ReleaseMessage.Clean.%s", toRemove.getMessage()), String.valueOf(toRemove.getId())));
 23:     }
 24: }
```

* 第 5 至 8 行：调用 `ReleaseMessageRepository#findOne(id)` 方法，查询对应的 ReleaseMessage 对象，避免已经删除。
    * 因为，**DatabaseMessageSender 会在多进程中执行**。例如：1）Config Service + Admin Service ；2）N * Config Service ；3）N * Admin Service 。
    * 为什么  Config Service 和 Admin Service 都会启动清理任务呢？😈 因为 DatabaseMessageSender 添加了 `@Component` 注解，而 NamespaceService 注入了 DatabaseMessageSender 。而 NamespaceService 被 `apollo-adminservice` 和 `apoll-configservice` 项目都引用了，所以都会启动该任务。
* 第 10 至 23 行：**循环删除**，**相同消息内容**( `ReleaseMessage.message` )的**老**消息，即 Namespace 的**老**消息。
    * 第 14 至 15 行：调用 `ReleaseMessageRepository#findFirst100ByMessageAndIdLessThanOrderByIdAsc(message, id)`  方法，拉取相同消息内容的 **100** 条的老消息，按照 id **升序**。 
        *  老消息的**定义**：比当前消息编号小，即先发送的。
    *  第 17 行：调用 `ReleaseMessageRepository#delete(messages)` 方法，**删除**老消息。
    *  第 19 行：若拉取**不足** 100 条，说明无老消息了。
    *  第 21 至 22 行：【TODO 6001】Tracer 日志

# 4. ReleaseMessageListener

`com.ctrip.framework.apollo.biz.message.ReleaseMessageListener` ，ReleaseMessage **监听器**接口。代码如下：

```Java
public interface ReleaseMessageListener {

    /**
     * 处理 ReleaseMessage
     *
     * @param message
     * @param channel 通道（主题）
     */
    void handleMessage(ReleaseMessage message, String channel);

}
```

ReleaseMessageListener 实现子类如下图：![子类](http://www.iocoder.cn/images/Apollo/2018_04_05/02.png)

例如，NotificationControllerV2 得到配置发布的 **AppId+Cluster+Namespace** 后，会通知对应的客户端。🙂 具体的代码实现，我们下一篇文章分享。

## 4.1 ReleaseMessageScanner

`com.ctrip.framework.apollo.biz.message.ReleaseMessageScanner` ，实现 `org.springframework.beans.factory.InitializingBean` 接口，ReleaseMessage 扫描器，**被 Config Service 使用**。

### 4.1.1 构造方法

```Java
@Autowired
private BizConfig bizConfig;
@Autowired
private ReleaseMessageRepository releaseMessageRepository;
/**
 * 从 DB 中扫描 ReleaseMessage 表的频率，单位：毫秒
 */
private int databaseScanInterval;
/**
 * 监听器数组
 */
private List<ReleaseMessageListener> listeners;
/**
 * 定时任务服务
 */
private ScheduledExecutorService executorService;
/**
 * 最后扫描到的 ReleaseMessage 的编号
 */
private long maxIdScanned;

public ReleaseMessageScanner() {
    // 创建监听器数组
    listeners = Lists.newCopyOnWriteArrayList();
    // 创建 ScheduledExecutorService 对象
    executorService = Executors.newScheduledThreadPool(1, ApolloThreadFactory.create("ReleaseMessageScanner", true));
}
```

* `listeners` 属性，监听器数组。通过 `#addMessageListener(ReleaseMessageListener)` 方法，注册 ReleaseMessageListener 。在 **MessageScannerConfiguration** 中，调用该方法，初始化 ReleaseMessageScanner 的监听器们。代码如下：

    ```Java
    @Configuration
    static class MessageScannerConfiguration {
    
        @Autowired
        private NotificationController notificationController;
        @Autowired
        private ConfigFileController configFileController;
        @Autowired
        private NotificationControllerV2 notificationControllerV2;
        @Autowired
        private GrayReleaseRulesHolder grayReleaseRulesHolder;
        @Autowired
        private ReleaseMessageServiceWithCache releaseMessageServiceWithCache;
        @Autowired
        private ConfigService configService;
    
        @Bean
        public ReleaseMessageScanner releaseMessageScanner() {
            ReleaseMessageScanner releaseMessageScanner = new ReleaseMessageScanner();
            // 0. handle release message cache
            releaseMessageScanner.addMessageListener(releaseMessageServiceWithCache);
            // 1. handle gray release rule
            releaseMessageScanner.addMessageListener(grayReleaseRulesHolder);
            // 2. handle server cache
            releaseMessageScanner.addMessageListener(configService);
            releaseMessageScanner.addMessageListener(configFileController);
            // 3. notify clients
            releaseMessageScanner.addMessageListener(notificationControllerV2);
            releaseMessageScanner.addMessageListener(notificationController);
            return releaseMessageScanner;
        }
    }
    ```

### 4.1.2 初始化 Scan 任务

`#afterPropertiesSet()` 方法，通过 Spring 调用，初始化 Scan 任务。代码如下：

```Java
  1: @Override
  2: public void afterPropertiesSet() {
  3:     // 从 ServerConfig 中获得频率
  4:     databaseScanInterval = bizConfig.releaseMessageScanIntervalInMilli();
  5:     // 获得最大的 ReleaseMessage 的编号
  6:     maxIdScanned = loadLargestMessageId();
  7:     // 创建从 DB 中扫描 ReleaseMessage 表的定时任务
  8:     executorService.scheduleWithFixedDelay((Runnable) () -> {
  9:         // 【TODO 6001】Tracer 日志
 10:         Transaction transaction = Tracer.newTransaction("Apollo.ReleaseMessageScanner", "scanMessage");
 11:         try {
 12:             // 从 DB 中，扫描 ReleaseMessage 们
 13:             scanMessages();
 14:             // 【TODO 6001】Tracer 日志
 15:             transaction.setStatus(Transaction.SUCCESS);
 16:         } catch (Throwable ex) {
 17:             // 【TODO 6001】Tracer 日志
 18:             transaction.setStatus(ex);
 19:             logger.error("Scan and send message failed", ex);
 20:         } finally {
 21:             // 【TODO 6001】Tracer 日志
 22:             transaction.complete();
 23:         }
 24:     }, databaseScanInterval, databaseScanInterval, TimeUnit.MILLISECONDS);
 25: }
```

* 第 4 行：调用 `BizConfig#releaseMessageScanIntervalInMilli()` 方法，从 ServerConfig 中获得频率，单位：毫秒。可通过 `"apollo.message-scan.interval"` 配置，默认：**1000** ms 。
* 第 6 行：调用 `#loadLargestMessageId()` 方法，获得**最大的** ReleaseMessage 的编号。代码如下：

    ```Java
    /**
     * find largest message id as the current start point
     *
     * @return current largest message id
     */
    private long loadLargestMessageId() {
        ReleaseMessage releaseMessage = releaseMessageRepository.findTopByOrderByIdDesc();
        return releaseMessage == null ? 0 : releaseMessage.getId();
    }
    ```
    
* 第 8 至 24 行：调用 `ExecutorService#scheduleWithFixedDelay(Runnable)` 方法，创建从 DB 中扫描 ReleaseMessage 表的定时任务。
    * 第 13 行：调用 `#scanMessages()` 方法，从 DB 中，扫描**新的** ReleaseMessage 们。

-------

`#scanMessages()` 方法，**循环**扫描消息，直到没有**新的** ReleaseMessage 为止。代码如下：

```Java
private void scanMessages() {
    boolean hasMoreMessages = true;
    while (hasMoreMessages && !Thread.currentThread().isInterrupted()) {
        hasMoreMessages = scanAndSendMessages();
    }
}
```

-------

`#scanAndSendMessages()` 方法，扫描消息，并返回是否继续有**新的** ReleaseMessage 可以继续扫描。代码如下：

```Java
  1: private boolean scanAndSendMessages() {
  2:     // 获得大于 maxIdScanned 的 500 条 ReleaseMessage 记录，按照 id 升序
  3:     // current batch is 500
  4:     List<ReleaseMessage> releaseMessages = releaseMessageRepository.findFirst500ByIdGreaterThanOrderByIdAsc(maxIdScanned);
  5:     if (CollectionUtils.isEmpty(releaseMessages)) {
  6:         return false;
  7:     }
  8:     // 触发监听器
  9:     fireMessageScanned(releaseMessages);
 10:     // 获得新的 maxIdScanned ，取最后一条记录
 11:     int messageScanned = releaseMessages.size();
 12:     maxIdScanned = releaseMessages.get(messageScanned - 1).getId();
 13:     // 若拉取不足 500 条，说明无新消息了
 14:     return messageScanned == 500;
 15: }
```

* 第 4 至 7 行：调用 `ReleaseMessageRepository#findFirst500ByIdGreaterThanOrderByIdAsc(maxIdScanned)` 方法，获得**大于 maxIdScanned** 的 **500** 条 ReleaseMessage 记录，**按照 id 升序**。
* 第 9 行：调用 `#fireMessageScanned(List<ReleaseMessage> messages)` 方法，触发监听器们。
* 第 10 至 12 行：获得**新的** `maxIdScanned` ，取**最后一条**记录。
* 第 14 行：若拉取**不足 500** 条，说明无新消息了。

### 4.1.3 fireMessageScanned

`#fireMessageScanned(List<ReleaseMessage> messages)` 方法，触发监听器，处理 ReleaseMessage 们。代码如下：

```Java
private void fireMessageScanned(List<ReleaseMessage> messages) {
    for (ReleaseMessage message : messages) { // 循环 ReleaseMessage
        for (ReleaseMessageListener listener : listeners) { // 循环 ReleaseMessageListener
            try {
                // 触发监听器
                listener.handleMessage(message, Topics.APOLLO_RELEASE_TOPIC);
            } catch (Throwable ex) {
                Tracer.logError(ex);
                logger.error("Failed to invoke message listener {}", listener.getClass(), ex);
            }
        }
    }
}
```

# 666. 彩蛋

美滋滋，小干货一篇。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

