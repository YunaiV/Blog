title: Apollo 源码解析 —— Config Service 通知配置变化
date: 2018-04-10
tags:
categories: Apollo
permalink: Apollo/config-service-notifications
wechat_url:
toutiao_url:

---

摘要: 原创出处 http://www.iocoder.cn/Apollo/config-service-notifications/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Apollo/config-service-notifications/)
- [2. NotificationControllerV2](http://www.iocoder.cn/Apollo/config-service-notifications/)
  - [2.1 构造方法](http://www.iocoder.cn/Apollo/config-service-notifications/)
  - [2.2 pollNotification](http://www.iocoder.cn/Apollo/config-service-notifications/)
  - [2.3 handleMessage](http://www.iocoder.cn/Apollo/config-service-notifications/)
- [3. ApolloConfigNotification](http://www.iocoder.cn/Apollo/config-service-notifications/)
  - [3.1 ApolloNotificationMessages](http://www.iocoder.cn/Apollo/config-service-notifications/)
- [4. DeferredResultWrapper](http://www.iocoder.cn/Apollo/config-service-notifications/)
  - [4.1 构造方法](http://www.iocoder.cn/Apollo/config-service-notifications/)
  - [4.2 onTimeout](http://www.iocoder.cn/Apollo/config-service-notifications/)
  - [4.3 onCompletion](http://www.iocoder.cn/Apollo/config-service-notifications/)
  - [4.4 setResult](http://www.iocoder.cn/Apollo/config-service-notifications/)
- [5. AppNamespaceServiceWithCache](http://www.iocoder.cn/Apollo/config-service-notifications/)
  - [5.1 构造方法](http://www.iocoder.cn/Apollo/config-service-notifications/)
  - [5.2 初始化定时任务](http://www.iocoder.cn/Apollo/config-service-notifications/)
  - [5.3 scanNewAppNamespaces](http://www.iocoder.cn/Apollo/config-service-notifications/)
  - [5.4 updateAndDeleteCache](http://www.iocoder.cn/Apollo/config-service-notifications/)
  - [5.5 findByAppIdAndNamespace](http://www.iocoder.cn/Apollo/config-service-notifications/)
  - [5.6 findByAppIdAndNamespaces](http://www.iocoder.cn/Apollo/config-service-notifications/)
  - [5.7 findPublicNamespaceByName](http://www.iocoder.cn/Apollo/config-service-notifications/)
  - [5.8 findPublicNamespacesByNames](http://www.iocoder.cn/Apollo/config-service-notifications/)
- [6. ReleaseMessageServiceWithCache](http://www.iocoder.cn/Apollo/config-service-notifications/)
  - [6.1 构造方法](http://www.iocoder.cn/Apollo/config-service-notifications/)
  - [6.2 初始化定时任务](http://www.iocoder.cn/Apollo/config-service-notifications/)
  - [6.3 loadReleaseMessages](http://www.iocoder.cn/Apollo/config-service-notifications/)
  - [6.4 handleMessage](http://www.iocoder.cn/Apollo/config-service-notifications/)
- [7. WatchKeysUtil](http://www.iocoder.cn/Apollo/config-service-notifications/)
  - [6.5 findLatestReleaseMessagesGroupByMessages](http://www.iocoder.cn/Apollo/config-service-notifications/)
  - [7.1 assembleWatchKeys](http://www.iocoder.cn/Apollo/config-service-notifications/)
  - [7.2 namespacesBelongToAppId](http://www.iocoder.cn/Apollo/config-service-notifications/)
  - [7.3 findPublicConfigWatchKeys](http://www.iocoder.cn/Apollo/config-service-notifications/)
  - [7.4 assembleKey](http://www.iocoder.cn/Apollo/config-service-notifications/)
- [8. EntityManagerUtil](http://www.iocoder.cn/Apollo/config-service-notifications/)
- [666. 彩蛋](http://www.iocoder.cn/Apollo/config-service-notifications/)

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

本文接 [《Apollo 源码解析 —— Admin Service 发送 ReleaseMessage》](http://www.iocoder.cn/Apollo/admin-server-send-release-message/?self) 一文，分享配置发布的第四步，**NotificationControllerV2 得到配置发布的 AppId+Cluster+Namespace 后，会通知对应的客户端** 。

> FROM [《Apollo配置中心设计》](https://github.com/ctripcorp/apollo/wiki/Apollo%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83%E8%AE%BE%E8%AE%A1#212-config-service%E9%80%9A%E7%9F%A5%E5%AE%A2%E6%88%B7%E7%AB%AF%E7%9A%84%E5%AE%9E%E7%8E%B0%E6%96%B9%E5%BC%8F) 的 [2.1.2 Config Service 通知客户端的实现方式](#)
>
> 1. 客户端会发起一个Http 请求到 Config Service 的 `notifications/v`2 接口，也就是NotificationControllerV2 ，参见 RemoteConfigLongPollService 。
> 2. NotificationControllerV2 不会立即返回结果，而是通过 [Spring DeferredResult](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/context/request/async/DeferredResult.html) 把请求挂起。
> 3. 如果在 60 秒内没有该客户端关心的配置发布，那么会返回 Http 状态码 304 给客户端。
> 4. 如果有该客户端关心的配置发布，NotificationControllerV2 会调用 DeferredResult 的 setResult 方法，传入有配置变化的 namespace 信息，同时该请求会立即返回。客户端从返回的结果中获取到配置变化的 namespace 后，会立即请求 Config Service 获取该 namespace 的最新配置。

* 本文**不分享**第 1 步的客户端部分，在下一篇文章分享。
* 关于 SpringMVC DeferredResult 的知识，推荐阅读 [《SpringMVC DeferredResult 的 Long Polling 的应用》](http://www.kailing.pub/article/index/arcid/163.html) .

> 友情提示：在目前 Apollo 的实现里，如下的名词是“等价”的：
>
> * 通知编号 = `ReleaseMessage.id`
> * Watch Key = `ReleaseMessage.message`
>
> 文章暂时未统一用词，所以胖友看的时候需要“脑补”下。


# 2. NotificationControllerV2

> 老艿艿：流程较长，代码较多，请耐心理解。

`com.ctrip.framework.apollo.configservice.controller.NotificationControllerV2` ，实现 ReleaseMessageListener 接口，通知 Controller ，**仅**提供 `notifications/v2` 接口。

## 2.1 构造方法

```Java
/**
 * Watch Key 与 DeferredResultWrapper 的 Multimap
 *
 * Key：Watch Key
 * Value：DeferredResultWrapper 数组
 */
private final Multimap<String, DeferredResultWrapper> deferredResults = Multimaps.synchronizedSetMultimap(HashMultimap.create());

private static final Splitter STRING_SPLITTER = Splitter.on(ConfigConsts.CLUSTER_NAMESPACE_SEPARATOR).omitEmptyStrings();
private static final Type notificationsTypeReference = new TypeToken<List<ApolloConfigNotification>>() {}.getType();

/**
 * 通过 ReleaseMessage 的消息内容，获得对应 Namespace 的名字
 */
private static final Function<String, String> retrieveNamespaceFromReleaseMessage =
        releaseMessage -> {
            if (Strings.isNullOrEmpty(releaseMessage)) {
                return null;
            }
            List<String> keys = STRING_SPLITTER.splitToList(releaseMessage);
            //message should be appId+cluster+namespace
            if (keys.size() != 3) {
                logger.error("message format invalid - {}", releaseMessage);
                return null;
            }
            return keys.get(2);
        };

/**
 * 大量通知分批执行 ExecutorService
 */
private final ExecutorService largeNotificationBatchExecutorService;
@Autowired
private WatchKeysUtil watchKeysUtil;
@Autowired
private ReleaseMessageServiceWithCache releaseMessageService;
@Autowired
private EntityManagerUtil entityManagerUtil;
@Autowired
private NamespaceUtil namespaceUtil;
@Autowired
private Gson gson;
@Autowired
private BizConfig bizConfig;

public NotificationControllerV2() {
    largeNotificationBatchExecutorService = Executors.newSingleThreadExecutor(ApolloThreadFactory.create("NotificationControllerV2", true));
}
```

* `deferredResults` 属性，**Watch Key** 与 DeferredResultWrapper 的 Multimap 。
    * 在下文中，我们会看到大量的 **Watch Key** 。实际上，目前 Apollo 的实现上，**Watch Key** 等价于 ReleaseMessage 的通知内容 `message` 字段。
    * Multimap 指的是 Google Guava Multimap ，不熟悉的胖友可以看看 [《Guava 学习笔记：Guava 新增集合类型 - Multimap》](http://www.cnblogs.com/peida/p/Guava_Multimap.html) 。推荐在项目中使用。
    * 在 `notifications/v2` 中，当请求的 Namespace **暂无新通知**时，会将该 Namespace 对应的 **Watch Key** 们，注册到 `deferredResults` 中。等到 Namespace **配置发生变更**时，在 `#handleMessage(...)` 中，进行通知。
* 其他属性，下文使用到，胖友可以回过头看看代码 + 注释。

## 2.2 pollNotification

```Java
  1: @RequestMapping(method = RequestMethod.GET)
  2: public DeferredResult<ResponseEntity<List<ApolloConfigNotification>>> pollNotification(
  3:         @RequestParam(value = "appId") String appId,
  4:         @RequestParam(value = "cluster") String cluster,
  5:         @RequestParam(value = "notifications") String notificationsAsString,
  6:         @RequestParam(value = "dataCenter", required = false) String dataCenter,
  7:         @RequestParam(value = "ip", required = false) String clientIp) {
  8:     // 解析 notificationsAsString 参数，创建 ApolloConfigNotification 数组。
  9:     List<ApolloConfigNotification> notifications = null;
 10:     try {
 11:         notifications = gson.fromJson(notificationsAsString, notificationsTypeReference);
 12:     } catch (Throwable ex) {
 13:         Tracer.logError(ex);
 14:     }
 15:     if (CollectionUtils.isEmpty(notifications)) {
 16:         throw new BadRequestException("Invalid format of notifications: " + notificationsAsString);
 17:     }
 18:
 19:     // 创建 DeferredResultWrapper 对象
 20:     DeferredResultWrapper deferredResultWrapper = new DeferredResultWrapper();
 21:     // Namespace 集合
 22:     Set<String> namespaces = Sets.newHashSet();
 23:     // 客户端的通知 Map 。key 为 Namespace 名，value 为通知编号。
 24:     Map<String, Long> clientSideNotifications = Maps.newHashMap();
 25:     // 过滤并创建 ApolloConfigNotification Map
 26:     Map<String, ApolloConfigNotification> filteredNotifications = filterNotifications(appId, notifications);
 27:     // 循环 ApolloConfigNotification Map ，初始化上述变量。
 28:     for (Map.Entry<String, ApolloConfigNotification> notificationEntry : filteredNotifications.entrySet()) {
 29:         String normalizedNamespace = notificationEntry.getKey();
 30:         ApolloConfigNotification notification = notificationEntry.getValue();
 31:         // 添加到 `namespaces` 中。
 32:         namespaces.add(normalizedNamespace);
 33:         // 添加到 `clientSideNotifications` 中。
 34:         clientSideNotifications.put(normalizedNamespace, notification.getNotificationId());
 35:         // 记录名字被归一化的 Namespace 。因为，最终返回给客户端，使用原始的 Namespace 名字，否则客户端无法识别。
 36:         if (!Objects.equals(notification.getNamespaceName(), normalizedNamespace)) {
 37:             deferredResultWrapper.recordNamespaceNameNormalizedResult(notification.getNamespaceName(), normalizedNamespace);
 38:         }
 39:     }
 40:     if (CollectionUtils.isEmpty(namespaces)) {
 41:         throw new BadRequestException("Invalid format of notifications: " + notificationsAsString);
 42:     }
 43:
 44:     // 组装 Watch Key Multimap
 45:     Multimap<String, String> watchedKeysMap = watchKeysUtil.assembleAllWatchKeys(appId, cluster, namespaces, dataCenter);
 46:     // 生成 Watch Key 集合
 47:     Set<String> watchedKeys = Sets.newHashSet(watchedKeysMap.values());
 48:     // 获得 Watch Key 集合中，每个 Watch Key 对应的 ReleaseMessage 记录。
 49:     List<ReleaseMessage> latestReleaseMessages = releaseMessageService.findLatestReleaseMessagesGroupByMessages(watchedKeys);
 50:
 51:     /**
 52:      * Manually close the entity manager.
 53:      * Since for async request, Spring won't do so until the request is finished,
 54:      * which is unacceptable since we are doing long polling - means the db connection would be hold
 55:      * for a very long time
 56:      */
 57:     // 手动关闭 EntityManager
 58:     // 因为对于 async 请求，Spring 在请求完成之前不会这样做
 59:     // 这是不可接受的，因为我们正在做长轮询——意味着 db 连接将被保留很长时间。
 60:     // 实际上，下面的过程，我们已经不需要 db 连接，因此进行关闭。
 61:     entityManagerUtil.closeEntityManager();
 62:     // 获得新的 ApolloConfigNotification 通知数组
 63:     List<ApolloConfigNotification> newNotifications = getApolloConfigNotifications(namespaces, clientSideNotifications, watchedKeysMap, latestReleaseMessages);
 64:     // 若有新的通知，直接设置结果。
 65:     if (!CollectionUtils.isEmpty(newNotifications)) {
 66:         deferredResultWrapper.setResult(newNotifications);
 67:         // 若无新的通知，
 68:     } else {
 69:         // 注册超时事件
 70:         deferredResultWrapper.onTimeout(() -> logWatchedKeys(watchedKeys, "Apollo.LongPoll.TimeOutKeys")); // 【TODO 6001】Tracer 日志
 71:         // 注册结束事件
 72:         deferredResultWrapper.onCompletion(() -> {
 73:             // 移除 Watch Key + DeferredResultWrapper 出 `deferredResults`
 74:             // unregister all keys
 75:             for (String key : watchedKeys) {
 76:                 deferredResults.remove(key, deferredResultWrapper);
 77:             }
 78:             // 【TODO 6001】Tracer 日志
 79:             logWatchedKeys(watchedKeys, "Apollo.LongPoll.CompletedKeys");
 80:         });
 81:
 82:         // 注册 Watch Key + DeferredResultWrapper 到 `deferredResults` 中，等待配置发生变化后通知。详见 `#handleMessage(...)` 方法。
 83:         // register all keys
 84:         for (String key : watchedKeys) {
 85:             this.deferredResults.put(key, deferredResultWrapper);
 86:         }
 87:
 88:         // 【TODO 6001】Tracer 日志
 89:         logWatchedKeys(watchedKeys, "Apollo.LongPoll.RegisteredKeys");
 90:         logger.debug("Listening {} from appId: {}, cluster: {}, namespace: {}, datacenter: {}", watchedKeys, appId, cluster, namespaces, dataCenter);
 91:     }
 92:
 93:     return deferredResultWrapper.getResult();
 94: }
```

* **GET `/notifications/v2` 接口**，具体 URL 在类上注明。
* `notificationsAsString` 请求参数，JSON 字符串，在【第 8 至 17 行】的代码，解析成 `List<ApolloConfigNotification>` ，表示**客户端**本地的配置通知信息。
    * 因为一个客户端可以订阅**多个** Namespace ，所以该参数是 **List** 。关于 ApolloConfigNotification 类，胖友先跳到 [「3. ApolloConfigNotification」](#) 看完在回来。
    * 我们可以注意到，该接口**真正**返回的结果也是 `List<ApolloConfigNotification>` ，**仅返回**配置发生变化的 Namespace 对应的 ApolloConfigNotification 。也就说，当有**几个** 配置发生变化的 Namespace ，返回**几个**对应的 ApolloConfigNotification 。另外，**客户端**接收到返回后，会**增量合并**到本地的配置通知信息。**客户端**下次请求时，使用**合并后**的配置通知信息。
    * **注意**，客户端请求时，只传递 ApolloConfigNotification 的 **`namespaceName` + `notificationId`** ，不传递 `messages` 。
* `clientIp` 请求参数，目前该接口暂时用不到，作为**预留**参数。🙂 万一未来在**灰度发布**需要呢。
* 第 20 行：创建 DeferredResultWrapper 对象。
* 第 22 行：创建 Namespace 的名字的集合。
* 第 24 行：创建**客户端**的通知信息 Map 。其中，KEY 为 Namespace 的名字，VALUE 为通知编号。
* 第 26 行：调用 `#filterNotifications(appId, notifications)` 方法，**过滤**并创建 ApolloConfigNotification Map 。胖友先跳到 [「2.2.1 filterNotifications」](#) 看完在回来。
* 第 27 至 39 行：循环 ApolloConfigNotification Map ，初始化上述变量。
    * 第 32 行：添加到 `namespaces` 中。
    * 第 34 行：添加到 `clientSideNotifications` 中。
    * 第 35 至 38 行：若 Namespace 的名字被归一化( normalized )了，则调用 `DeferredResultWrapper#recordNamespaceNameNormalizedResult(originalNamespaceName, normalizedNamespaceName)` 方法，记录名字被归一化的 Namespace 。因为，最终返回给客户端，使用原始的 Namespace 名字，否则客户端无法识别。
* 第 45 行：调用 `WatchKeysUtil#assembleAllWatchKeys(appId, cluster, namespaces, dataCenter)` 方法，组装 **Watch Key** Multimap 。胖友先跳到 [「7. WatchKeysUtil」](#) 看完在回来。
* 第 47 行：生成 **Watch Key** 集合。
* 第 49 行：调用 `ReleaseMessageServiceWithCache#findLatestReleaseMessagesGroupByMessages(watchedKeys)` 方法，获得 Watch Key 集合中，每个 Watch Key 对应的**最新的** ReleaseMessage 记录。胖友先跳到 [「6. ReleaseMessageServiceWithCache」](#) 看完在回来。
* 第 61 行：调用 `EntityManagerUtil#closeEntityManager()` 方法，**手动**关闭 EntityManager 。因为对于 **async** 请求，SpringMVC 在请求完成之前不会这样做。这是不可接受的，因为我们正在做长轮询——意味着 db 连接将被保留很长时间。实际上，下面的过程，我们已经**不需要** db 连接，因此进行关闭。[「8. EntityManagerUtil」](#) 看完在回来。
* 第 63 行：调用 `#getApolloConfigNotifications(namespaces, clientSideNotifications, watchedKeysMap, latestReleaseMessages)` 方法，获得**新的** ApolloConfigNotification 通知数组。胖友先跳到 [「2.2.2 getApolloConfigNotifications」](#) 看完在回来。
* 第 64 至 66 行：若有**新**的通知，调用 `DeferredResultWrapper#setResult(List<ApolloConfigNotification>)` 方法，直接设置 DeferredResult 的结果，从而**结束**长轮询。
* 第 67 至 91 行：若无**新**的通知，注册到 `deferredResults` 中，等到有配置变更或超时。
    * 第 70 行：调用 `DeferredResultWrapper#onTimeout(Runnable)` 方法，注册**超时**事件。
    * 第 71 至 80 行：调用 `DeferredResultWrapper#onCompletion(Runnable)` 方法，注册**结束**事件。在其内部，**移除**注册的 **Watch Key + DeferredResultWrapper** 出 `deferredResults` 。
    * 第 82 至 86 行：注册 **Watch Key + DeferredResultWrapper** 到 `deferredResults` 中，等待配置发生变化后通知。这样，任意一个 **Watch Key** 对应的 Namespace 对应的配置发生变化时，都可以进行通知，并结束轮询等待。详细解析，见 [「2.3 handleMessage」](#) 方法。
* 第 93 行：返回 DeferredResult 对象。

### 2.2.1 filterNotifications

`#filterNotifications(appId, notifications)` 方法，**过滤**并创建 ApolloConfigNotification Map 。其中，KEY 为 Namespace 的名字。代码如下：

```Java
  1: private Map<String, ApolloConfigNotification> filterNotifications(String appId, List<ApolloConfigNotification> notifications) {
  2:     Map<String, ApolloConfigNotification> filteredNotifications = Maps.newHashMap();
  3:     for (ApolloConfigNotification notification : notifications) {
  4:         if (Strings.isNullOrEmpty(notification.getNamespaceName())) {
  5:             continue;
  6:         }
  7:         // 若 Namespace 名以 .properties 结尾，移除该结尾，并设置到 ApolloConfigNotification 中。例如 application.properties => application 。
  8:         // strip out .properties suffix
  9:         String originalNamespace = namespaceUtil.filterNamespaceName(notification.getNamespaceName());
 10:         notification.setNamespaceName(originalNamespace);
 11:         // 获得归一化的 Namespace 名字。因为，客户端 Namespace 会填写错大小写。
 12:         // 例如，数据库中 Namespace 名为 Fx.Apollo ，而客户端 Namespace 名为 fx.Apollo
 13:         //      通过归一化后，统一为 Fx.Apollo
 14:         // fix the character case issue, such as FX.apollo <-> fx.apollo
 15:         String normalizedNamespace = namespaceUtil.normalizeNamespace(appId, originalNamespace);
 16:
 17:         // in case client side namespace name has character case issue and has difference notification ids
 18:         // such as FX.apollo = 1 but fx.apollo = 2, we should let FX.apollo have the chance to update its notification id
 19:         // which means we should record FX.apollo = 1 here and ignore fx.apollo = 2
 20:         // 如果客户端 Namespace 的名字有大小写的问题，并且恰好有不同的通知编号。
 21:         // 例如 Namespace 名字为 FX.apollo 的通知编号是 1 ，但是 fx.apollo 的通知编号为 2 。
 22:         //     我们应该让 FX.apollo 可以更新它的通知编号，
 23:         //     所以，我们使用 FX.apollo 的 ApolloConfigNotification 对象，添加到结果，而忽略 fx.apollo 。
 24:         if (filteredNotifications.containsKey(normalizedNamespace) &&
 25:                 filteredNotifications.get(normalizedNamespace).getNotificationId() < notification.getNotificationId()) {
 26:             continue;
 27:         }
 28:
 29:         filteredNotifications.put(normalizedNamespace, notification);
 30:     }
 31:     return filteredNotifications;
 32: }
```

* 🙂 这个方法的逻辑比较“绕”，目的是客户端传递的 Namespace 的名字不是正确的，例如大小写不对，需要做下**归一化**( normalized )处理。
* **循环** ApolloConfigNotification 数组。
* 第 9 至 10 行：调用 `NamespaceUtil#filterNamespaceName(namespaceName)` 方法，若 Namespace 名以 `".properties"` 结尾，移除该结尾，并设置到 ApolloConfigNotification 中。例如： `application.properties => application` 。代码如下：

    ```Java
    public String filterNamespaceName(String namespaceName) {
        // 若 Namespace 名以 .properties 结尾，移除该结尾，
        if (namespaceName.toLowerCase().endsWith(".properties")) {
            int dotIndex = namespaceName.lastIndexOf(".");
            return namespaceName.substring(0, dotIndex);
        }
        return namespaceName;
    }
    ```

* 第 15 行：调用 `NamespaceUtil#normalizeNamespace(appId, originalNamespace)` 方法，获得**归一化**的 Namespace 名字。因为，客户端 Namespace 会填写错大小写。
    * 例如，数据库中 Namespace 名为 `"Fx.Apollo"` ，而客户端 Namespace 名为 `"fx.Apollo"` 。通过归一化后，统一为 `"Fx.Apollo"` 。
    * 代码如下：

        ```Java
          1: @Autowired
          2: private AppNamespaceServiceWithCache appNamespaceServiceWithCache;
          3:
          4: public String normalizeNamespace(String appId, String namespaceName) {
          5:     // 获得 App 下的 AppNamespace 对象
          6:     AppNamespace appNamespace = appNamespaceServiceWithCache.findByAppIdAndNamespace(appId, namespaceName);
          7:     if (appNamespace != null) {
          8:         return appNamespace.getName();
          9:     }
         10:     // 获取不到，说明该 Namespace 可能是关联的
         11:     appNamespace = appNamespaceServiceWithCache.findPublicNamespaceByName(namespaceName);
         12:     if (appNamespace != null) {
         13:         return appNamespace.getName();
         14:     }
         15:     return namespaceName;
         16: }
        ```

        * 第 5 至 9 行：调用 `AppNamespaceServiceWithCache#findByAppIdAndNamespace(appId, namespaceName)` 方法，获得 **App 下**的 AppNamespace 对象。
        * 第 10 至 14 行：获取不到，说明该 Namespace 可能是**关联类型**的，所以调用 `AppNamespaceServiceWithCache#findPublicNamespaceByName(namespaceName)` 方法，查询**公用类型**的 AppNamespace 对象。
        * 第 15 行：都查询不到，直接返回。为什么呢？因为 AppNamespaceServiceWithCache 是基于**缓存**实现，可能对应的 AppNamespace 暂未缓存到内存中。
* 第 17 至 27 行：如果客户端 Namespace 的名字有大小写的问题，并且恰好有**不同的**通知编号。例如 Namespace 名字为 `"FX.apollo"` 的通知编号是 1 ，但是 `"fx.apollo"` 的通知编号为 2 。我们应该让 `"FX.apollo"` 可以更新它的通知编号，所以，我们使用 `"FX.apollo"` 的 ApolloConfigNotification 对象，添加到结果，而忽略 `"fx.apollo"` 。通过这样的方式，若此时服务器的通知编号为 3 ，那么 `"FX.apollo"` 的通知编号先更新成 3 ，**再下一次**长轮询时，`"fx.apollo"` 的通知编号再更新成 3 。🙂 比较“绕”，胖友细细品味下，大多数情况下，不会出现这样的情况。
* 第 29 行：添加到 `filteredNotifications` 中。

### 2.2.2 getApolloConfigNotifications

`#getApolloConfigNotifications(namespaces, clientSideNotifications, watchedKeysMap, latestReleaseMessages)` 方法，获得**新的** ApolloConfigNotification 通知数组。代码如下：

```Java
  1: private List<ApolloConfigNotification> getApolloConfigNotifications(Set<String> namespaces,
  2:                                                                     Map<String, Long> clientSideNotifications,
  3:                                                                     Multimap<String, String> watchedKeysMap,
  4:                                                                     List<ReleaseMessage> latestReleaseMessages) {
  5:     // 创建 ApolloConfigNotification 数组
  6:     List<ApolloConfigNotification> newNotifications = Lists.newArrayList();
  7:     if (!CollectionUtils.isEmpty(latestReleaseMessages)) {
  8:         // 创建最新通知的 Map 。其中 Key 为 Watch Key 。
  9:         Map<String, Long> latestNotifications = Maps.newHashMap();
 10:         for (ReleaseMessage releaseMessage : latestReleaseMessages) {
 11:             latestNotifications.put(releaseMessage.getMessage(), releaseMessage.getId());
 12:         }
 13:         // 循环 Namespace 的名字的集合，判断是否有配置更新
 14:         for (String namespace : namespaces) {
 15:             long clientSideId = clientSideNotifications.get(namespace);
 16:             long latestId = ConfigConsts.NOTIFICATION_ID_PLACEHOLDER;
 17:             // 获得 Namespace 对应的 Watch Key 集合
 18:             Collection<String> namespaceWatchedKeys = watchedKeysMap.get(namespace);
 19:             // 获得最大的通知编号
 20:             for (String namespaceWatchedKey : namespaceWatchedKeys) {
 21:                 long namespaceNotificationId = latestNotifications.getOrDefault(namespaceWatchedKey, ConfigConsts.NOTIFICATION_ID_PLACEHOLDER);
 22:                 if (namespaceNotificationId > latestId) {
 23:                     latestId = namespaceNotificationId;
 24:                 }
 25:             }
 26:             // 若服务器的通知编号大于客户端的通知编号，意味着有配置更新
 27:             if (latestId > clientSideId) {
 28:                 // 创建 ApolloConfigNotification 对象
 29:                 ApolloConfigNotification notification = new ApolloConfigNotification(namespace, latestId);
 30:                 // 循环添加通知编号到 ApolloConfigNotification 中。
 31:                 namespaceWatchedKeys.stream().filter(latestNotifications::containsKey).forEach(namespaceWatchedKey ->
 32:                         notification.addMessage(namespaceWatchedKey, latestNotifications.get(namespaceWatchedKey)));
 33:                 // 添加 ApolloConfigNotification 对象到结果
 34:                 newNotifications.add(notification);
 35:             }
 36:         }
 37:     }
 38:     return newNotifications;
 39: }
```

* 第 6 行：创建**新的** ApolloConfigNotification 数组。
* 第 8 至 12 行：创建**最新**通知的 Map 。其中，KEY 为 **Watch Key** 。
* 第 14 至 37 行：**循环** Namespace 的名字的集合，根据 `latestNotifications` 判断是否有配置更新。
    * 第 18 行：获得 Namespace 对应的 **Watch Key** 集合。
    * 第 19 至 25 行：获得**最大**的通知编号。
    * 第 26 至 35 行：若服务器的通知编号**大于**客户端的通知编号，意味着有配置更新。
        * 第 29 行：创建  ApolloConfigNotification 对象。
        * 第 30 至 32 行：**循环**调用 `ApolloConfigNotification#addMessage(String key, long notificationId)` 方法，添加通知编号到 ApolloConfigNotification 中。对于**关联类型**的 Namespace ，`details` 会是多个。
        * 第 34 行：添加 ApolloConfigNotification 对象到结果( `newNotifications` )。
* 第 38 行：返回 `newNotifications` 。若非空，说明有配置更新。

## 2.3 handleMessage

`#handleMessage(ReleaseMessage, channel)` 方法，当有**新的** ReleaseMessage 时，通知其对应的 Namespace 的，并且正在等待的请求。代码如下：

```Java
  1: @Override
  2: public void handleMessage(ReleaseMessage message, String channel) {
  3:     logger.info("message received - channel: {}, message: {}", channel, message);
  4:     // 【TODO 6001】Tracer 日志
  5:     String content = message.getMessage();
  6:     Tracer.logEvent("Apollo.LongPoll.Messages", content);
  7:
  8:     // 仅处理 APOLLO_RELEASE_TOPIC
  9:     if (!Topics.APOLLO_RELEASE_TOPIC.equals(channel) || Strings.isNullOrEmpty(content)) {
 10:         return;
 11:     }
 12:
 13:     // 获得对应的 Namespace 的名字
 14:     String changedNamespace = retrieveNamespaceFromReleaseMessage.apply(content);
 15:     if (Strings.isNullOrEmpty(changedNamespace)) {
 16:         logger.error("message format invalid - {}", content);
 17:         return;
 18:     }
 19:
 20:     // `deferredResults` 存在对应的 Watch Key
 21:     if (!deferredResults.containsKey(content)) {
 22:         return;
 23:     }
 24:
 25:     // create a new list to avoid ConcurrentModificationException
 26:     // 创建 DeferredResultWrapper 数组，避免并发问题。
 27:     List<DeferredResultWrapper> results = Lists.newArrayList(deferredResults.get(content));
 28:
 29:     // 创建 ApolloConfigNotification 对象
 30:     ApolloConfigNotification configNotification = new ApolloConfigNotification(changedNamespace, message.getId());
 31:     configNotification.addMessage(content, message.getId());
 32:
 33:     // do async notification if too many clients
 34:     // 若需要通知的客户端过多，使用 ExecutorService 异步通知，避免“惊群效应”
 35:     if (results.size() > bizConfig.releaseMessageNotificationBatch()) {
 36:         largeNotificationBatchExecutorService.submit(() -> {
 37:             logger.debug("Async notify {} clients for key {} with batch {}", results.size(), content,
 38:                     bizConfig.releaseMessageNotificationBatch());
 39:             for (int i = 0; i < results.size(); i++) {
 40:                 // 每 N 个客户端，sleep 一段时间。
 41:                 if (i > 0 && i % bizConfig.releaseMessageNotificationBatch() == 0) {
 42:                     try {
 43:                         TimeUnit.MILLISECONDS.sleep(bizConfig.releaseMessageNotificationBatchIntervalInMilli());
 44:                     } catch (InterruptedException e) {
 45:                         //ignore
 46:                     }
 47:                 }
 48:                 logger.debug("Async notify {}", results.get(i));
 49:                 // 设置结果
 50:                 results.get(i).setResult(configNotification);
 51:             }
 52:         });
 53:         return;
 54:     }
 55:
 56:     logger.debug("Notify {} clients for key {}", results.size(), content);
 57:     // 设置结果
 58:     for (DeferredResultWrapper result : results) {
 59:         result.setResult(configNotification);
 60:     }
 61:     logger.debug("Notification completed");
 62: }
```

* 第 8 至 11 行：仅处理 **APOLLO_RELEASE_TOPIC** 。
* 第 13 至 18 行：获得对应的 Namespace 的名字。
* 第 20 至 23 行：`deferredResults` 存在对应的 **Watch Key**。
* 第 27 行：从 `deferredResults` 中读取并创建 DeferredResultWrapper 数组，避免并发问题。
* 第 30 至 31 行：创建 ApolloConfigNotification 对象，并调用 `ApolloConfigNotification#addMessage(String key, long notificationId)` 方法，添加通知消息明细。此处，`details` 是**一个**。
* 【异步】当需要通知的客户端**过多**，使用 ExecutorService 异步通知，避免“**惊群效应**”。🙂 感谢作者( 宋顺大佬 )的解答：

    > 假设一个公共 Namespace 有10W 台机器使用，如果该公共 Namespace 发布时直接下发配置更新消息的话，就会导致这 10W 台机器一下子都来请求配置，这动静就有点大了，而且对 Config Service 的压力也会比较大。

    * 数量可通过 ServerConfig `"apollo.release-message.notification.batch"` 配置，默认 **100** 。
    * 第 40 至 47 行：每通知 `"apollo.release-message.notification.batch"` 个客户端，**sleep** 一段时间。可通过 ServerConfig `"apollo.release-message.notification.batch.interval"` 配置，默认 **100** 毫秒。
    * 第 50 行：调用 `DeferredResultWrapper#setResult(List<ApolloConfigNotification>)` 方法，设置 DeferredResult 的结果，从而结束长轮询。
    * 第 53 行：**return** 。
* 【同步】第 57 至 60 行：**循环**调用 `DeferredResultWrapper#setResult(List<ApolloConfigNotification>)` 方法，设置 DeferredResult 的结果，从而结束长轮询。

# 3. ApolloConfigNotification

`com.ctrip.framework.apollo.core.dto.ApolloConfigNotification` ，Apollo 配置通知 **DTO** 。代码如下：

```Java
public class ApolloConfigNotification {

    /**
     * Namespace 名字
     */
    private String namespaceName;
    /**
     * 最新通知编号
     *
     * 目前使用 `ReleaseMessage.id` 。
     */
    private long notificationId;
    /**
     * 通知消息集合
     */
    private volatile ApolloNotificationMessages messages;

    public ApolloConfigNotification(String namespaceName, long notificationId) {
        this.namespaceName = namespaceName;
        this.notificationId = notificationId;
    }

}
```

* `namespaceName` 字段，Namespace 名，指向对应的 Namespace 。因此，一个 Namespace 对应一个 ApolloConfigNotification 对象。
* `notificationId` 字段，**最新**通知编号，目前使用 `ReleaseMessage.id` 字段。
* `messages` 字段，通知消息集合。
    * `volatile` 修饰，因为存在多线程修改和读取。
    * `#addMessage(String key, long notificationId)` 方法，添加消息明细到 `message` 中。代码如下：

        ```Java
        public void addMessage(String key, long notificationId) {
            // 创建 ApolloNotificationMessages 对象
            if (this.messages == null) {
                synchronized (this) {
                    if (this.messages == null) {
                        this.messages = new ApolloNotificationMessages();
                    }
                }
            }
            // 添加到 `messages` 中
            this.messages.put(key, notificationId);
        }
        ```
        * x

## 3.1 ApolloNotificationMessages

`com.ctrip.framework.apollo.core.dto.ApolloNotificationMessages` ，Apollo 配置通知消息集合 **DTO** 。代码如下：

```Java
public class ApolloNotificationMessages {

    /**
     * 明细 Map
     *
     * KEY ：{appId} "+" {clusterName} "+" {namespace} ，例如：100004458+default+application
     * VALUE ：通知编号
     */
    private Map<String, Long> details;

    public ApolloNotificationMessages() {
        this(Maps.<String, Long>newHashMap());
    }

}
```

* `details` 字段，明细 Map 。其中，KEY 是 **Watch Key** 。

为什么 ApolloConfigNotification 中有 ApolloNotificationMessages ，而且 ApolloNotificationMessages 的 `details` 字段是 Map ？按道理说，对于一个 Namespace 的通知，使用 ApolloConfigNotification 的 **`namespaceName` + `notificationId`** 已经足够了。但是，在 `namespaceName` 对应的 Namespace 是**关联类型**时，会**同时**查询当前 Namespace + 关联的 Namespace **这两个 Namespace**，所以会是多个，使用 Map 数据结构。
当然，对于 `/notifications/v2` 接口，**仅有**【直接】获得到配置变化才可能出现 `ApolloNotificationMessages.details` 为**多个**的情况。为啥？在 `#handleMessage(...)` 方法中，一次只处理一条 ReleaseMessage ，因此只会有 `ApolloNotificationMessages.details` 只会有**一个**。

**put**

```Java
public void put(String key, long notificationId) {
    details.put(key, notificationId);
}
```

**mergeFrom**

```Java
public void mergeFrom(ApolloNotificationMessages source) {
    if (source == null) {
        return;
    }
    for (Map.Entry<String, Long> entry : source.getDetails().entrySet()) {
        // to make sure the notification id always grows bigger
        // 只合并新的通知编号大于的情况
        if (this.has(entry.getKey()) && this.get(entry.getKey()) >= entry.getValue()) {
            continue;
        }
        this.put(entry.getKey(), entry.getValue());
    }
}
```

* 在**客户端**中使用，将 Config Service 返回的结果，合并到本地的通知信息中。

# 4. DeferredResultWrapper

`com.ctrip.framework.apollo.configservice.wrapper.DeferredResultWrapper` ，DeferredResult 包装器，封装 DeferredResult 的**公用**方法。

## 4.1 构造方法

```Java
/**
 * 默认超时时间
 */
private static final long TIMEOUT = 60 * 1000; //60 seconds
/**
 * 未修改时的 ResponseEntity 响应，使用 302 状态码。
 */
private static final ResponseEntity<List<ApolloConfigNotification>> NOT_MODIFIED_RESPONSE_LIST = new ResponseEntity<>(HttpStatus.NOT_MODIFIED);

/**
 * 归一化和原始的 Namespace 的名字的 Map
 */
private Map<String, String> normalizedNamespaceNameToOriginalNamespaceName;
/**
 * 响应的 DeferredResult 对象
 */
private DeferredResult<ResponseEntity<List<ApolloConfigNotification>>> result;

public DeferredResultWrapper() {
    result = new DeferredResult<>(TIMEOUT, NOT_MODIFIED_RESPONSE_LIST);
}
```

* `TIMEOUT` **静态**属性，默认超时时间。
* `NOT_MODIFIED_RESPONSE_LIST` **静态**属性，**未修改**时的 ResponseEntity 响应，使用 **302** 状态码。
* `normalizedNamespaceNameToOriginalNamespaceName` 属性，归一化( normalized )和原始( original )的 Namespace 的名字的 Map 。因为**客户端**在填写 Namespace 时，写错了**名字**的**大小写**。在 Config Service 中，会进行归一化“修复”，方便逻辑的统一编写。但是，最终返回给**客户端**需要“还原”回原始( original )的 Namespace 的名字，避免客户端无法识别。
    * `#recordNamespaceNameNormalizedResult(String originalNamespaceName, String normalizedNamespaceName)`  方法，记录归一化和原始的 Namespace 的名字的映射。代码如下：

        ```Java
        public void recordNamespaceNameNormalizedResult(String originalNamespaceName, String normalizedNamespaceName) {
            if (normalizedNamespaceNameToOriginalNamespaceName == null) {
                normalizedNamespaceNameToOriginalNamespaceName = Maps.newHashMap();
            }
            // 添加到 `normalizedNamespaceNameToOriginalNamespaceName` 中
            normalizedNamespaceNameToOriginalNamespaceName.put(normalizedNamespaceName, originalNamespaceName); // 和参数的顺序，相反
        }
        ```
        * x

* `result` 属性，**响应**的 DeferredResult 对象，在**构造方法**中初始化。

## 4.2 onTimeout

```Java
public void onTimeout(Runnable timeoutCallback) {
    result.onTimeout(timeoutCallback);
}
```

## 4.3 onCompletion

```Java
public void setResult(ApolloConfigNotification notification) {
    setResult(Lists.newArrayList(notification));
}
```

## 4.4 setResult

```Java
public void setResult(ApolloConfigNotification notification) {
    setResult(Lists.newArrayList(notification));
}

public void setResult(List<ApolloConfigNotification> notifications) {
    // 恢复被归一化的 Namespace 的名字为原始的 Namespace 的名字
    if (normalizedNamespaceNameToOriginalNamespaceName != null) {
        notifications.stream().filter(notification -> normalizedNamespaceNameToOriginalNamespaceName.containsKey
                (notification.getNamespaceName())).forEach(notification -> notification.setNamespaceName(
                normalizedNamespaceNameToOriginalNamespaceName.get(notification.getNamespaceName())));
    }
    // 设置结果，并使用 200 状态码。
    result.setResult(new ResponseEntity<>(notifications, HttpStatus.OK));
}
```

* **两部分**工作，胖友看代码注释。

# 5. AppNamespaceServiceWithCache

`com.ctrip.framework.apollo.configservice.service.AppNamespaceServiceWithCache` ，实现 InitializingBean 接口，缓存 AppNamespace 的 Service 实现类。通过将 AppNamespace 缓存在内存中，提高查询性能。缓存实现方式如下：

1. 启动时，全量初始化 AppNamespace 到缓存
2. 考虑 AppNamespace 新增，后台定时任务，定时增量初始化 AppNamespace 到缓存
3. 考虑 AppNamespace 更新与删除，后台定时任务，定时全量重建 AppNamespace 到缓存

## 5.1 构造方法

```Java
private static final Joiner STRING_JOINER = Joiner.on(ConfigConsts.CLUSTER_NAMESPACE_SEPARATOR).skipNulls();

@Autowired
private AppNamespaceRepository appNamespaceRepository;
@Autowired
private BizConfig bizConfig;

/**
 * 增量初始化周期
 */
private int scanInterval;
/**
 * 增量初始化周期单位
 */
private TimeUnit scanIntervalTimeUnit;
/**
 * 重建周期
 */
private int rebuildInterval;
/**
 * 重建周期单位
 */
private TimeUnit rebuildIntervalTimeUnit;
/**
 * 定时任务 ExecutorService
 */
private ScheduledExecutorService scheduledExecutorService;
/**
 * 最后扫描到的 AppNamespace 的编号
 */
private long maxIdScanned;

/**
 * 公用类型的 AppNamespace 的缓存
 *
 * //store namespaceName -> AppNamespace
 */
private CaseInsensitiveMapWrapper<AppNamespace> publicAppNamespaceCache;
/**
 * App 下的 AppNamespace 的缓存
 *
 * store appId+namespaceName -> AppNamespace
 */
private CaseInsensitiveMapWrapper<AppNamespace> appNamespaceCache;
/**
 * AppNamespace 的缓存
 *
 * //store id -> AppNamespace
 */
private Map<Long, AppNamespace> appNamespaceIdCache;

public AppNamespaceServiceWithCache() {
    initialize();
}

private void initialize() {
    maxIdScanned = 0;
    // 创建缓存对象
    publicAppNamespaceCache = new CaseInsensitiveMapWrapper<>(Maps.newConcurrentMap());
    appNamespaceCache = new CaseInsensitiveMapWrapper<>(Maps.newConcurrentMap());
    appNamespaceIdCache = Maps.newConcurrentMap();
    // 创建 ScheduledExecutorService 对象，大小为 1 。
    scheduledExecutorService = Executors.newScheduledThreadPool(1, ApolloThreadFactory.create("AppNamespaceServiceWithCache", true));
}
```

* 属性都比较简单易懂，胖友看下注释哈。
* `appNamespaceCache` 的 KEY ，通过 `#assembleAppNamespaceKey(AppNamespace)` 方法，拼接 `appId` + `name` 。代码如下：

    ```Java
    private String assembleAppNamespaceKey(AppNamespace appNamespace) {
        return STRING_JOINER.join(appNamespace.getAppId(), appNamespace.getName());
    }
    ```

## 5.2 初始化定时任务

`#afterPropertiesSet()` 方法，通过 Spring 调用，初始化定时任务。代码如下：

```Java
  1: @Override
  2: public void afterPropertiesSet() throws Exception {
  3:     // 从 ServerConfig 中，读取定时任务的周期配置
  4:     populateDataBaseInterval();
  5:     // 全量初始化 AppNamespace 缓存
  6:     scanNewAppNamespaces(); // block the startup process until load finished
  7:     // 创建定时任务，全量重构 AppNamespace 缓存
  8:     scheduledExecutorService.scheduleAtFixedRate(() -> {
  9:         // 【TODO 6001】Tracer 日志
 10:         Transaction transaction = Tracer.newTransaction("Apollo.AppNamespaceServiceWithCache", "rebuildCache");
 11:         try {
 12:             // 全量重建 AppNamespace 缓存
 13:             this.updateAndDeleteCache();
 14:             // 【TODO 6001】Tracer 日志
 15:             transaction.setStatus(Transaction.SUCCESS);
 16:         } catch (Throwable ex) {
 17:             // 【TODO 6001】Tracer 日志
 18:             transaction.setStatus(ex);
 19:             logger.error("Rebuild cache failed", ex);
 20:         } finally {
 21:             // 【TODO 6001】Tracer 日志
 22:             transaction.complete();
 23:         }
 24:     }, rebuildInterval, rebuildInterval, rebuildIntervalTimeUnit);
 25:     // 创建定时任务，增量初始化 AppNamespace 缓存
 26:     scheduledExecutorService.scheduleWithFixedDelay(this::scanNewAppNamespaces, scanInterval, scanInterval, scanIntervalTimeUnit);
 27: }
```

* 第 4 行：调用 `#populateDataBaseInterval()` 方法，从 ServerConfig 中，读取定时任务的周期配置。代码如下：

    ```Java
    private void populateDataBaseInterval() {
        scanInterval = bizConfig.appNamespaceCacheScanInterval(); // "apollo.app-namespace-cache-scan.interval"
        scanIntervalTimeUnit = bizConfig.appNamespaceCacheScanIntervalTimeUnit(); // 默认秒，不可配置
        rebuildInterval = bizConfig.appNamespaceCacheRebuildInterval(); // "apollo.app-namespace-cache-rebuild.interval"
        rebuildIntervalTimeUnit = bizConfig.appNamespaceCacheRebuildIntervalTimeUnit(); // 默认秒，不可配置
    }
    ```

* 第 6 行：调用 `#scanNewAppNamespaces()` 方法，全量初始化 AppNamespace 缓存。在 [「5.3 scanNewAppNamespaces」](#) 中详细解析。
* 第 7 至 24 行：创建定时任务，全量重建 AppNamespace 缓存。
    * 第 13 行：调用 `#updateAndDeleteCache()` 方法，更新和删除 AppNamespace 缓存。 在 [「5.4 scanNewAppNamespaces」](#) 中详细解析。
* 第 26 行：创建定时任务，增量初始化 AppNamespace 缓存。其内部，调用的**也是** `#scanNewAppNamespaces()` 方法。

## 5.3 scanNewAppNamespaces

```Java
private void scanNewAppNamespaces() {
    // 【TODO 6001】Tracer 日志
    Transaction transaction = Tracer.newTransaction("Apollo.AppNamespaceServiceWithCache", "scanNewAppNamespaces");
    try {
        // 加载新的 AppNamespace 们
        this.loadNewAppNamespaces();
        // 【TODO 6001】Tracer 日志
        transaction.setStatus(Transaction.SUCCESS);
    } catch (Throwable ex) {
        // 【TODO 6001】Tracer 日志
        transaction.setStatus(ex);
        logger.error("Load new app namespaces failed", ex);
    } finally {
        // 【TODO 6001】Tracer 日志
        transaction.complete();
    }
}
```

* 调用 `#loadNewAppNamespaces()` 方法，加载新的 AppNamespace 们。代码如下：

    ```Java
    private void loadNewAppNamespaces() {
        boolean hasMore = true;
        while (hasMore && !Thread.currentThread().isInterrupted()) { // 循环，直到无新的 AppNamespace
            // current batch is 500
            // 获得大于 maxIdScanned 的 500 条 AppNamespace 记录，按照 id 升序
            List<AppNamespace> appNamespaces = appNamespaceRepository.findFirst500ByIdGreaterThanOrderByIdAsc(maxIdScanned);
            if (CollectionUtils.isEmpty(appNamespaces)) {
                break;
            }
            // 合并到 AppNamespace 缓存中
            mergeAppNamespaces(appNamespaces);
            // 获得新的 maxIdScanned ，取最后一条记录
            int scanned = appNamespaces.size();
            maxIdScanned = appNamespaces.get(scanned - 1).getId();
            // 若拉取不足 500 条，说明无新消息了
            hasMore = scanned == 500;
            logger.info("Loaded {} new app namespaces with startId {}", scanned, maxIdScanned);
        }
    }
    ```
    * 调用 `#mergeAppNamespaces(appNamespaces)` 方法，合并到 AppNamespace 缓存中。代码如下：

        ```Java
        private void mergeAppNamespaces(List<AppNamespace> appNamespaces) {
            for (AppNamespace appNamespace : appNamespaces) {
                // 添加到 `appNamespaceCache` 中
                appNamespaceCache.put(assembleAppNamespaceKey(appNamespace), appNamespace);
                // 添加到 `appNamespaceIdCache`
                appNamespaceIdCache.put(appNamespace.getId(), appNamespace);
                // 若是公用类型，则添加到 `publicAppNamespaceCache` 中
                if (appNamespace.isPublic()) {
                    publicAppNamespaceCache.put(appNamespace.getName(), appNamespace);
                }
            }
        }
        ```
        * x

## 5.4 updateAndDeleteCache

```Java
private void updateAndDeleteCache() {
    // 从缓存中，获得所有的 AppNamespace 编号集合
    List<Long> ids = Lists.newArrayList(appNamespaceIdCache.keySet());
    if (CollectionUtils.isEmpty(ids)) {
        return;
    }
    // 每 500 一批，从数据库中查询最新的 AppNamespace 信息
    List<List<Long>> partitionIds = Lists.partition(ids, 500);
    for (List<Long> toRebuild : partitionIds) {
        Iterable<AppNamespace> appNamespaces = appNamespaceRepository.findAll(toRebuild);
        if (appNamespaces == null) {
            continue;
        }
        // 处理更新的情况
        // handle updated
        Set<Long> foundIds = handleUpdatedAppNamespaces(appNamespaces);
        // 处理删除的情况
        // handle deleted
        handleDeletedAppNamespaces(Sets.difference(Sets.newHashSet(toRebuild), foundIds));
    }
}
```

* `#handleUpdatedAppNamespaces(appNamespaces)` 方法，处理更新的情况。代码如下：

    ```Java
    private Set<Long> handleUpdatedAppNamespaces(Iterable<AppNamespace> appNamespaces) {
        Set<Long> foundIds = Sets.newHashSet();
        for (AppNamespace appNamespace : appNamespaces) {
            foundIds.add(appNamespace.getId());
            // 获得缓存中的 AppNamespace 对象
            AppNamespace thatInCache = appNamespaceIdCache.get(appNamespace.getId());
            // 从 DB 中查询到的 AppNamespace 的更新时间更大，才认为是更新
            if (thatInCache != null && appNamespace.getDataChangeLastModifiedTime().after(thatInCache.getDataChangeLastModifiedTime())) {
                // 添加到 appNamespaceIdCache 中
                appNamespaceIdCache.put(appNamespace.getId(), appNamespace);
                // 添加到 appNamespaceCache 中
                String oldKey = assembleAppNamespaceKey(thatInCache);
                String newKey = assembleAppNamespaceKey(appNamespace);
                appNamespaceCache.put(newKey, appNamespace);
                // 当 appId 或 namespaceName 发生改变的情况，将老的移除出 appNamespaceCache
                // in case appId or namespaceName changes
                if (!newKey.equals(oldKey)) {
                    appNamespaceCache.remove(oldKey);
                }
                // 添加到 publicAppNamespaceCache 中
                if (appNamespace.isPublic()) { // 新的是公用类型
                    // 添加到 publicAppNamespaceCache 中
                    publicAppNamespaceCache.put(appNamespace.getName(), appNamespace);
                    // 当 namespaceName 发生改变的情况，将老的移除出 publicAppNamespaceCache
                    // in case namespaceName changes
                    if (!appNamespace.getName().equals(thatInCache.getName()) && thatInCache.isPublic()) {
                        publicAppNamespaceCache.remove(thatInCache.getName());
                    }
                } else if (thatInCache.isPublic()) { // 新的不是公用类型，需要移除
                    //just in case isPublic changes
                    publicAppNamespaceCache.remove(thatInCache.getName());
                }
                logger.info("Found AppNamespace changes, old: {}, new: {}", thatInCache, appNamespace);
            }
        }
        return foundIds;
    }
    ```
    * 🙂 相对复杂一些，胖友耐心看下代码注释。
* `#handleDeletedAppNamespaces(Set<Long> deletedIds)` 方法，处理删除的情况。代码如下：

    ```Java
    private void handleDeletedAppNamespaces(Set<Long> deletedIds) {
        if (CollectionUtils.isEmpty(deletedIds)) {
            return;
        }
        for (Long deletedId : deletedIds) {
            // 从 appNamespaceIdCache 中移除
            AppNamespace deleted = appNamespaceIdCache.remove(deletedId);
            if (deleted == null) {
                continue;
            }
            // 从 appNamespaceCache 中移除
            appNamespaceCache.remove(assembleAppNamespaceKey(deleted));
            // 从 publicAppNamespaceCache 移除
            if (deleted.isPublic()) {
                publicAppNamespaceCache.remove(deleted.getName());
            }
            logger.info("Found AppNamespace deleted, {}", deleted);
        }
    }
    ```

## 5.5 findByAppIdAndNamespace 

```Java
/**
 * 获得 AppNamespace 对象
 *
 * @param appId App 编号
 * @param namespaceName Namespace 名字
 * @return AppNamespace
 */
public AppNamespace findByAppIdAndNamespace(String appId, String namespaceName) {
    Preconditions.checkArgument(!StringUtils.isContainEmpty(appId, namespaceName), "appId and namespaceName must not be empty");
    return appNamespaceCache.get(STRING_JOINER.join(appId, namespaceName));
}
```

## 5.6 findByAppIdAndNamespaces

```Java
/**
 * 获得 AppNamespace 对象数组
 *
 * @param appId App 编号
 * @param namespaceNames Namespace 名字的集合
 * @return AppNamespace 数组
 */
public List<AppNamespace> findByAppIdAndNamespaces(String appId, Set<String> namespaceNames) {
    Preconditions.checkArgument(!Strings.isNullOrEmpty(appId), "appId must not be null");
    if (namespaceNames == null || namespaceNames.isEmpty()) {
        return Collections.emptyList();
    }
    List<AppNamespace> result = Lists.newArrayList();
    // 循环获取
    for (String namespaceName : namespaceNames) {
        AppNamespace appNamespace = appNamespaceCache.get(STRING_JOINER.join(appId, namespaceName));
        if (appNamespace != null) {
            result.add(appNamespace);
        }
    }
    return result;
}
```

## 5.7 findPublicNamespaceByName

```Java
/**
 * 获得公用类型的 AppNamespace 对象
 *
 * @param namespaceName Namespace 名字
 * @return AppNamespace
 */
public AppNamespace findPublicNamespaceByName(String namespaceName) {
    Preconditions.checkArgument(!Strings.isNullOrEmpty(namespaceName), "namespaceName must not be empty");
    return publicAppNamespaceCache.get(namespaceName);
}
```

## 5.8 findPublicNamespacesByNames

```Java
/**
 * 获得公用类型的 AppNamespace 对象数组
 *
 * @param namespaceNames Namespace 名字的集合
 * @return AppNamespace 数组
 */
public List<AppNamespace> findPublicNamespacesByNames(Set<String> namespaceNames) {
    if (namespaceNames == null || namespaceNames.isEmpty()) {
        return Collections.emptyList();
    }

    List<AppNamespace> result = Lists.newArrayList();
    // 循环获取
    for (String namespaceName : namespaceNames) {
        AppNamespace appNamespace = publicAppNamespaceCache.get(namespaceName);
        if (appNamespace != null) {
            result.add(appNamespace);
        }
    }
    return result;
}
```

# 6. ReleaseMessageServiceWithCache

`com.ctrip.framework.apollo.configservice.service.ReleaseMessageServiceWithCache` ，实现 InitializingBean 和 ReleaseMessageListener 接口，缓存 ReleaseMessage 的 Service 实现类。通过将 ReleaseMessage 缓存在内存中，提高查询性能。缓存实现方式如下：

1. 启动时，初始化 ReleaseMessage 到缓存。
2. 新增时，基于 ReleaseMessageListener ，通知有新的 ReleaseMessage ，根据是否有消息间隙，直接使用该 ReleaseMessage 或从数据库读取。

## 6.1 构造方法

```Java
@Autowired
private ReleaseMessageRepository releaseMessageRepository;
@Autowired
private BizConfig bizConfig;

/**
 * 扫描周期
 */
private int scanInterval;
/**
 * 扫描周期单位
 */
private TimeUnit scanIntervalTimeUnit;
/**
 * 最后扫描到的 ReleaseMessage 的编号
 */
private volatile long maxIdScanned;
/**
 * ReleaseMessage 缓存
 *
 * KEY：`ReleaseMessage.message`
 * VALUE：对应的最新的 ReleaseMessage 记录
 */
private ConcurrentMap<String, ReleaseMessage> releaseMessageCache;
/**
 * 是否执行扫描任务
 */
private AtomicBoolean doScan;
/**
 * ExecutorService 对象
 */
private ExecutorService executorService;

public ReleaseMessageServiceWithCache() {
    initialize();
}

private void initialize() {
    // 创建缓存对象
    releaseMessageCache = Maps.newConcurrentMap();
    // 设置 doScan 为 true
    doScan = new AtomicBoolean(true);
    // 创建 ScheduledExecutorService 对象，大小为 1 。
    executorService = Executors.newSingleThreadExecutor(ApolloThreadFactory.create("ReleaseMessageServiceWithCache", true));
}
```

* 属性都比较简单易懂，胖友看下注释哈。

## 6.2 初始化定时任务

`#afterPropertiesSet()` 方法，通知 Spring 调用，初始化定时任务。代码如下：

```Java
  1: @Override
  2: public void afterPropertiesSet() throws Exception {
  3:     // 从 ServerConfig 中，读取任务的周期配置
  4:     populateDataBaseInterval();
  5:     // 初始拉取 ReleaseMessage 到缓存
  6:     //block the startup process until load finished
  7:     //this should happen before ReleaseMessageScanner due to autowire
  8:     loadReleaseMessages(0);
  9:     // 创建定时任务，增量拉取 ReleaseMessage 到缓存，用以处理初始化期间，产生的 ReleaseMessage 遗漏的问题。
 10:     executorService.submit(() -> {
 11:         while (doScan.get() && !Thread.currentThread().isInterrupted()) {
 12:             // 【TODO 6001】Tracer 日志
 13:             Transaction transaction = Tracer.newTransaction("Apollo.ReleaseMessageServiceWithCache", "scanNewReleaseMessages");
 14:             try {
 15:                 // 增量拉取 ReleaseMessage 到缓存
 16:                 loadReleaseMessages(maxIdScanned);
 17:                 // 【TODO 6001】Tracer 日志
 18:                 transaction.setStatus(Transaction.SUCCESS);
 19:             } catch (Throwable ex) {
 20:                 // 【TODO 6001】Tracer 日志
 21:                 transaction.setStatus(ex);
 22:                 logger.error("Scan new release messages failed", ex);
 23:             } finally {
 24:                 transaction.complete();
 25:             }
 26:             try {
 27:                 scanIntervalTimeUnit.sleep(scanInterval);
 28:             } catch (InterruptedException e) {
 29:                 //ignore
 30:             }
 31:         }
 32:     });
 33: }
```

* 第 4 行：调用 `#populateDataBaseInterval()` 方法，从 ServerConfig 中，读取定时任务的周期配置。代码如下：

    ```Java
    private void populateDataBaseInterval() {
        scanInterval = bizConfig.releaseMessageCacheScanInterval(); // "apollo.release-message-cache-scan.interval" ，默认为 1 。
        scanIntervalTimeUnit = bizConfig.releaseMessageCacheScanIntervalTimeUnit(); // 默认秒，不可配置。
    }
    ```

* 第 8 行：调用 `#loadReleaseMessages(startId)` 方法，**初始**拉取 ReleaseMessage 到缓存。在 [「6.3 loadReleaseMessages」](#) 中详细解析。
* 第 10 至 32 行：创建创建定时任务，**增量**拉取 ReleaseMessage 到缓存，用以处理初始化期间，产生的 ReleaseMessage 遗漏的问题。为什么会遗漏呢？笔者又去请教作者，🙂 给 666 个赞。

    > 1. 20:00:00 程序启动过程中，当前 release message 有 5 条
    > 2. 20:00:01 loadReleaseMessages(0); 执行完成，获取到 5 条记录
    > 3. 20:00:02 有一条 release message 新产生，但是因为程序还没启动完，所以不会触发 handle message 操作
    > 4. 20:00:05 程序启动完成，但是第三步的这条新的 release message 漏了
    > 5. 20:10:00 假设这时又有一条 release message 产生，这次会触发 handle message ，同时会把第三步的那条 release message 加载到
    
    > 所以，定期刷的机制就是为了解决**第三步中**产生的release message问题。  
    > 当程序启动完，handleMessage生效后，就不需要再定期扫了

    * ReleaseMessageServiceWithCache 初始化在 ReleaseMessageScanner 之前，因此在第 3 步时，ReleaseMessageServiceWithCache 初始化完成之后，ReleaseMessageScanner 初始化之前，产生了一条心的 ReleaseMessage ，会导致 `ReleaseMessageScanner.maxIdScanned` **大于** `ReleaseMessageServiceWithCache.maxIdScanned` ，从而导致 ReleaseMessage 的遗漏。

## 6.3 loadReleaseMessages

`#loadReleaseMessages(startId)` 方法，增量拉取**新的** ReleaseMessage 们。代码如下：

```Java
private void loadReleaseMessages(long startId) {
    boolean hasMore = true;
    while (hasMore && !Thread.currentThread().isInterrupted()) {
        // current batch is 500
        // 获得大于 maxIdScanned 的 500 条 ReleaseMessage 记录，按照 id 升序
        List<ReleaseMessage> releaseMessages = releaseMessageRepository.findFirst500ByIdGreaterThanOrderByIdAsc(startId);
        if (CollectionUtils.isEmpty(releaseMessages)) {
            break;
        }
        // 合并到 ReleaseMessage 缓存
        releaseMessages.forEach(this::mergeReleaseMessage);
        // 获得新的 maxIdScanned ，取最后一条记录
        int scanned = releaseMessages.size();
        startId = releaseMessages.get(scanned - 1).getId();
        // 若拉取不足 500 条，说明无新消息了
        hasMore = scanned == 500;
        logger.info("Loaded {} release messages with startId {}", scanned, startId);
    }
}
```

* 调用 `#mergeAppNamespaces(appNamespaces)` 方法，合并到 ReleaseMessage 缓存中。代码如下：

    ```Java
    private synchronized void mergeReleaseMessage(ReleaseMessage releaseMessage) {
        // 获得对应的 ReleaseMessage 对象
        ReleaseMessage old = releaseMessageCache.get(releaseMessage.getMessage());
        // 若编号更大，进行更新缓存
        if (old == null || releaseMessage.getId() > old.getId()) {
            releaseMessageCache.put(releaseMessage.getMessage(), releaseMessage);
            maxIdScanned = releaseMessage.getId();
        }
    }
    ```
    * x

## 6.4 handleMessage

```Java
  1: @Override
  2: public void handleMessage(ReleaseMessage message, String channel) {
  3:     // Could stop once the ReleaseMessageScanner starts to work
  4:     // 关闭增量拉取定时任务的执行
  5:     doScan.set(false);
  6:     logger.info("message received - channel: {}, message: {}", channel, message);
  7: 
  8:     // 仅处理 APOLLO_RELEASE_TOPIC
  9:     String content = message.getMessage();
 10:     Tracer.logEvent("Apollo.ReleaseMessageService.UpdateCache", String.valueOf(message.getId()));
 11:     if (!Topics.APOLLO_RELEASE_TOPIC.equals(channel) || Strings.isNullOrEmpty(content)) {
 12:         return;
 13:     }
 14: 
 15:     // 计算 gap
 16:     long gap = message.getId() - maxIdScanned;
 17:     // 若无空缺 gap ，直接合并
 18:     if (gap == 1) {
 19:         mergeReleaseMessage(message);
 20:     // 如有空缺 gap ，增量拉取
 21:     } else if (gap > 1) {
 22:         // gap found!
 23:         loadReleaseMessages(maxIdScanned);
 24:     }
 25: }
```

* 第 5 行：**关闭**增量拉取定时任务的执行。后续通过 ReleaseMessageScanner 通知即可。
* 第 9 至 13 行：仅处理 **APOLLO_RELEASE_TOPIC** 。
* 第 16 行：计算 `gap` 。
* 第 18 至 20 行：若**无**空缺，调用 `#mergeReleaseMessage(message)` 方法，直接合并即可。
* 第 21 至 24 行：若**有**空缺，调用 `#loadReleaseMessages(maxIdScanned)` 方法，增量拉取。例如，上述的**第 3 步**，定时任务还**来不及**拉取( 即未执行 )，ReleaseMessageScanner 就已经通知，此处会产生**空缺**的 `gap` 。

# 7. WatchKeysUtil

`com.ctrip.framework.apollo.configservice.util.WatchKeysUtil` ，**Watch Key** 工具类。

**核心**的方法为 `#assembleAllWatchKeys(appId, clusterName, namespaces, dataCenter)` 方法，组装 **Watch Key** Multimap 。其中 KEY 为 Namespace 的名字，VALUE 为 **Watch Key** 集合。代码如下：

```Java
  1: /**
  2:  * 组装所有的 Watch Key Multimap 。其中 Key 为 Namespace 的名字，Value 为 Watch Key 集合。
  3:  *
  4:  * @param appId App 编号
  5:  * @param clusterName Cluster 名
  6:  * @param namespaces Namespace 的名字的数组
  7:  * @param dataCenter IDC 的 Cluster 名
  8:  * @return Watch Key Multimap
  9:  */
 10: public Multimap<String, String> assembleAllWatchKeys(String appId, String clusterName,
 11:                                                      Set<String> namespaces,
 12:                                                      String dataCenter) {
 13:     // 组装 Watch Key Multimap
 14:     Multimap<String, String> watchedKeysMap = assembleWatchKeys(appId, clusterName, namespaces, dataCenter);
 15:
 16:     // 如果不是仅监听 'application' Namespace ，处理其关联来的 Namespace 。
 17:     // Every app has an 'application' namespace
 18:     if (!(namespaces.size() == 1 && namespaces.contains(ConfigConsts.NAMESPACE_APPLICATION))) {
 19:         // 获得属于该 App 的 Namespace 的名字的集合
 20:         Set<String> namespacesBelongToAppId = namespacesBelongToAppId(appId, namespaces);
 21:         // 获得关联来的 Namespace 的名字的集合
 22:         Set<String> publicNamespaces = Sets.difference(namespaces, namespacesBelongToAppId);
 23:         // 添加到 Watch Key Multimap 中
 24:         // Listen on more namespaces if it's a public namespace
 25:         if (!publicNamespaces.isEmpty()) {
 26:             watchedKeysMap.putAll(findPublicConfigWatchKeys(appId, clusterName, publicNamespaces, dataCenter));
 27:         }
 28:     }
 29:
 30:     return watchedKeysMap;
 31: }
```

* 第 14 行：调用 `#assembleWatchKeys(appId, clusterName, namespaces, dataCenter)` 方法，组装 **App 下**的 **Watch Key** Multimap 。详细解析，见 [「7.1 assembleWatchKeys」](#)  。
* 第 18 至 28 行：判断 `namespaces` 中，可能存在**关联类型**的 Namespace ，因此需要进一步处理。在这里的判断会比较“绕”，如果 `namespaces` 仅仅是 `"application"` 时，那么肯定不存在**关联类型**的 Namespace 。
    * 第 20 行：调用 `#namespacesBelongToAppId(appId, namespaces)`  方法，获得属于该 **App** 的 Namespace 的名字的集合。详细解析，见 [「7.2 namespacesBelongToAppId」](#)  。
    * 第 22 行：通过 `Sets#difference(...)` 方法，进行集合差异计算，获得**关联类型**的 Namespace 的名字的集合。
    * 第 25 至 27 行：调用 `#findPublicConfigWatchKeys(...)` 方法，获得**关联类型**的 Namespace 的名字的集合的 **Watch Key** Multimap ，并添加到结果集中。详细解析，见 [「7.3 findPublicConfigWatchKeys」](#)  。
* 第 30 行：返回结果集。

## 6.5 findLatestReleaseMessagesGroupByMessages

```Java
/**
 * 获得每条消息内容对应的最新的 ReleaseMessage 对象
 *
 * @param messages 消息内容的集合
 * @return 集合
 */
public List<ReleaseMessage> findLatestReleaseMessagesGroupByMessages(Set<String> messages) {
    if (CollectionUtils.isEmpty(messages)) {
        return Collections.emptyList();
    }
    List<ReleaseMessage> releaseMessages = Lists.newArrayList();
    // 获得每条消息内容对应的最新的 ReleaseMessage 对象
    for (String message : messages) {
        ReleaseMessage releaseMessage = releaseMessageCache.get(message);
        if (releaseMessage != null) {
            releaseMessages.add(releaseMessage);
        }
    }
    return releaseMessages;
}
```

## 7.1 assembleWatchKeys

```Java
/**
 * 组装 Watch Key Multimap
 *
 * @param appId App 编号
 * @param clusterName Cluster 名
 * @param namespaces Namespace 的名字的集合
 * @param dataCenter IDC 的 Cluster 名字
 * @return Watch Key Multimap
 */
private Multimap<String, String> assembleWatchKeys(String appId, String clusterName, Set<String> namespaces, String dataCenter) {
    Multimap<String, String> watchedKeysMap = HashMultimap.create();
    // 循环 Namespace 的名字的集合
    for (String namespace : namespaces) {
        watchedKeysMap.putAll(namespace, assembleWatchKeys(appId, clusterName, namespace, dataCenter));
    }
    return watchedKeysMap;
}
```

* **循环** Namespace 的名字的集合，调用 `#assembleWatchKeys(appId, clusterName, namespace, dataCenter)` 方法，组装**指定** Namespace 的 Watch Key 数组。代码如下：

    ```Java
      1: private Set<String> assembleWatchKeys(String appId, String clusterName, String namespace, String dataCenter) {
      2:     if (ConfigConsts.NO_APPID_PLACEHOLDER.equalsIgnoreCase(appId)) {
      3:         return Collections.emptySet();
      4:     }
      5:     Set<String> watchedKeys = Sets.newHashSet();
      6:
      7:     // 指定 Cluster
      8:     // watch specified cluster config change
      9:     if (!Objects.equals(ConfigConsts.CLUSTER_NAME_DEFAULT, clusterName)) {
     10:         watchedKeys.add(assembleKey(appId, clusterName, namespace));
     11:     }
     12:
     13:     // 所属 IDC 的 Cluster
     14:     // https://github.com/ctripcorp/apollo/issues/952
     15:     // watch data center config change
     16:     if (!Strings.isNullOrEmpty(dataCenter) && !Objects.equals(dataCenter, clusterName)) {
     17:         watchedKeys.add(assembleKey(appId, dataCenter, namespace));
     18:     }
     19:
     20:     // 默认 Cluster
     21:     // watch default cluster config change
     22:     watchedKeys.add(assembleKey(appId, ConfigConsts.CLUSTER_NAME_DEFAULT, namespace));
     23:
     24:     return watchedKeys;
     25: }
    ```

    * 指定 Cluster  的 Namespace 的 **Watch Key** 。
    * 所属 IDC 的 Cluster 的 Namespace 的 **Watch Key** 。关于
    * 默认( `"default"` ) 的 Cluster 的 Namespace 的 **Watch Key** 。
    * `#assembleKey(appId, clusterName, namespace)` 方法，获得 **Watch Key** ，详细解析，见 [「7.4 assembleKey」](#)  。

关于多 Cluster 的读取顺序，可参见 [《Apollo 配置中心介绍 —— 4.4.1 应用自身配置的获取规则》](https://github.com/ctripcorp/apollo/wiki/Apollo%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83%E4%BB%8B%E7%BB%8D#442-%E5%85%AC%E5%85%B1%E7%BB%84%E4%BB%B6%E9%85%8D%E7%BD%AE%E7%9A%84%E8%8E%B7%E5%8F%96%E8%A7%84%E5%88%99) 。后续，我们也专门分享这块。

## 7.2 namespacesBelongToAppId

```Java
/**
 * 获得属于该 App 的 Namespace 的名字的集合
 *
 * @param appId App 编号
 * @param namespaces Namespace 名
 * @return 集合
 */
private Set<String> namespacesBelongToAppId(String appId, Set<String> namespaces) {
    if (ConfigConsts.NO_APPID_PLACEHOLDER.equalsIgnoreCase(appId)) {
        return Collections.emptySet();
    }
    // 获得属于该 App 的 AppNamespace 集合
    List<AppNamespace> appNamespaces = appNamespaceService.findByAppIdAndNamespaces(appId, namespaces);
    if (appNamespaces == null || appNamespaces.isEmpty()) {
        return Collections.emptySet();
    }
    // 返回 AppNamespace 的名字的集合
    return appNamespaces.stream().map(AppNamespace::getName).collect(Collectors.toSet());
}
```

## 7.3 findPublicConfigWatchKeys

```Java
@Autowired
private AppNamespaceServiceWithCache appNamespaceService;

/**
 * 获得 Namespace 类型为 public 对应的 Watch Key Multimap
 *
 * 重要：要求非当前 App 的 Namespace
 *
 * @param applicationId App 编号
 * @param clusterName Cluster 名
 * @param namespaces Namespace 的名字的集合
 * @param dataCenter  IDC 的 Cluster 名
 * @return Watch Key Map
 */
private Multimap<String, String> findPublicConfigWatchKeys(String applicationId, String clusterName, Set<String> namespaces, String dataCenter) {
    Multimap<String, String> watchedKeysMap = HashMultimap.create();
    // 获得 Namespace 为 public 的 AppNamespace 数组
    List<AppNamespace> appNamespaces = appNamespaceService.findPublicNamespacesByNames(namespaces);
    // 组装 Watch Key Map
    for (AppNamespace appNamespace : appNamespaces) {
        // 排除非关联类型的 Namespace
        // check whether the namespace's appId equals to current one
        if (Objects.equals(applicationId, appNamespace.getAppId())) {
            continue;
        }
        String publicConfigAppId = appNamespace.getAppId();
        // 组装指定 Namespace 的 Watch Key 数组
        watchedKeysMap.putAll(appNamespace.getName(), assembleWatchKeys(publicConfigAppId, clusterName, appNamespace.getName(), dataCenter));
    }
    return watchedKeysMap;
}
```

## 7.4 assembleKey

```Java
private static final Joiner STRING_JOINER = Joiner.on(ConfigConsts.CLUSTER_NAMESPACE_SEPARATOR);

/**
 * 拼接 Watch Key
 *
 * @param appId App 编号
 * @param cluster Cluster 名
 * @param namespace Namespace 名
 * @return Watch Key
 */
private String assembleKey(String appId, String cluster, String namespace) {
    return STRING_JOINER.join(appId, cluster, namespace);
}
```

* **Watch Key** 的格式和 `ReleaseMessage.message` 的格式是**一致**的。

# 8. EntityManagerUtil

`com.ctrip.framework.apollo.biz.utils.EntityManagerUtil` ，实现 `org.springframework.orm.jpa.EntityManagerFactoryAccessor` 抽象类，EntityManager 抽象类。代码如下：

```Java
@Component
public class EntityManagerUtil extends EntityManagerFactoryAccessor {

    private static final Logger logger = LoggerFactory.getLogger(EntityManagerUtil.class);

    /**
     * close the entity manager.
     * Use it with caution! This is only intended for use with async request, which Spring won't
     * close the entity manager until the async request is finished.
     */
    public void closeEntityManager() {
        // 获得 EntityManagerHolder 对象
        EntityManagerHolder emHolder = (EntityManagerHolder) TransactionSynchronizationManager.getResource(getEntityManagerFactory());
        if (emHolder == null) {
            return;
        }
        logger.debug("Closing JPA EntityManager in EntityManagerUtil");
        // 关闭 EntityManager
        EntityManagerFactoryUtils.closeEntityManager(emHolder.getEntityManager());
    }

}
```

# 666. 彩蛋

爽~爽爽~~爽爽爽~~~

感叹一句，比想象中长太多了。另外，RocketMQ 也是基于长轮询的方式，获取新的消息。实现上有一些差距，大的方向一致。感兴趣的胖友，可以看看老艿艿的 RocketMQ 源码解析系列。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)


