title: Apollo 源码解析 —— Portal 发布配置
date: 2018-04-01
tags:
categories: Apollo
permalink: Apollo/portal-publish
wechat_url:
toutiao_url: https://www.toutiao.com/i6643382949297259016/

---

摘要: 原创出处 http://www.iocoder.cn/Apollo/portal-publish/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Apollo/portal-publish/)
- [2. 实体](http://www.iocoder.cn/Apollo/portal-publish/)
  - [2.1 Release](http://www.iocoder.cn/Apollo/portal-publish/)
  - [2.2 ReleaseHistory](http://www.iocoder.cn/Apollo/portal-publish/)
  - [2.3 ReleaseMessage](http://www.iocoder.cn/Apollo/portal-publish/)
- [3. Portal 侧](http://www.iocoder.cn/Apollo/portal-publish/)
  - [3.1 ReleaseController](http://www.iocoder.cn/Apollo/portal-publish/)
  - [3.2 ReleaseService](http://www.iocoder.cn/Apollo/portal-publish/)
  - [3.3 ReleaseAPI](http://www.iocoder.cn/Apollo/portal-publish/)
- [4. Admin Service 侧](http://www.iocoder.cn/Apollo/portal-publish/)
  - [4.1 ReleaseController](http://www.iocoder.cn/Apollo/portal-publish/)
  - [4.2 ReleaseService](http://www.iocoder.cn/Apollo/portal-publish/)
  - [4.3 ReleaseRepository](http://www.iocoder.cn/Apollo/portal-publish/)
  - [4.4 ReleaseHistoryService](http://www.iocoder.cn/Apollo/portal-publish/)
  - [4.5 ReleaseHistoryRepository](http://www.iocoder.cn/Apollo/portal-publish/)
- [666. 彩蛋](http://www.iocoder.cn/Apollo/portal-publish/)

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

从本文开始，我们进入 Apollo **最最最**核心的流程 [配置发布后的实时推送设计](https://github.com/ctripcorp/apollo/wiki/Apollo%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83%E8%AE%BE%E8%AE%A1#21-%E9%85%8D%E7%BD%AE%E5%8F%91%E5%B8%83%E5%90%8E%E7%9A%84%E5%AE%9E%E6%97%B6%E6%8E%A8%E9%80%81%E8%AE%BE%E8%AE%A1) 。

> 在配置中心中，一个重要的功能就是配置发布后实时推送到客户端。下面我们简要看一下这块是怎么设计实现的。
> 
> ![配置发布](http://www.iocoder.cn/images/Apollo/2018_04_01/01.png)
> 
> 上图简要描述了配置发布的大致过程：
> 
> 1. 用户在 Portal 操作配置发布
> 2. Portal 调用 Admin Service 的接口操作发布
> 3. Admin Service 发布配置后，发送 ReleaseMessage 给各个Config Service
> 4. Config Service 收到 ReleaseMessage 后，通知对应的客户端

本文分享 **Portal 发布配置**，对应上述第一、二步，大体流程如下：

![流程](http://www.iocoder.cn/images/Apollo/2018_04_01/02.png)

* 😈 这个流程过程中，我们先不考虑**灰度**发布，会涉及**配置合并**的过程。

> 老艿艿：因为 Portal 是管理后台，所以从代码实现上，和业务系统非常相像。也因此，本文会略显啰嗦。

# 2. 实体

## 2.1 Release

`com.ctrip.framework.apollo.biz.entity.Release` ，继承 BaseEntity 抽象类，Release **实体**。代码如下：

```Java
@Entity
@Table(name = "Release")
@SQLDelete(sql = "Update Release set isDeleted = 1 where id = ?") // 标记删除
@Where(clause = "isDeleted = 0")
public class Release extends BaseEntity {

    /**
     * Release Key
     *
     * 【TODO 6006】用途？
     */
    @Column(name = "ReleaseKey", nullable = false)
    private String releaseKey;
    /**
     * 标题
     */
    @Column(name = "Name", nullable = false)
    private String name;
    /**
     * 备注
     */
    @Column(name = "Comment", nullable = false)
    private String comment;
    /**
     * App 编号
     */
    @Column(name = "AppId", nullable = false)
    private String appId;
    /**
     * Cluster 名字
     */
    @Column(name = "ClusterName", nullable = false)
    private String clusterName;
    /**
     * Namespace 名字
     */
    @Column(name = "NamespaceName", nullable = false)
    private String namespaceName;
    /**
     * 配置 Map 字符串，使用 JSON 格式化成字符串
     */
    @Column(name = "Configurations", nullable = false)
    @Lob
    private String configurations;
    /**
     * 是否被回滚（放弃）
     */
    @Column(name = "IsAbandoned", columnDefinition = "Bit default '0'")
    private boolean isAbandoned;
}
```

* `releaseKey` 字段，【TODO 6006】用途？
* `name` 字段，发布标题。
* `comment` 字段，发布备注。
* `appId` + `clusterName` + `namespaceName` 字段，指向对应的 Namespace 记录。
* `configurations` 字段，发布时的**完整**配置 Map **字符串**，使用 JSON 格式化成字符串。
    * 和 `Commit.changeSets` 字段，格式**一致**，只是它是**变化**配置 Map **字符串**。
    * 例子如下：

        ```JSON
        {"huidu01":"huidu01"}
        ```
        * x

* `isAbandoned` 字段，是否被回滚（放弃）。

## 2.2 ReleaseHistory

`com.ctrip.framework.apollo.biz.entity.ReleaseHistory` ，继承 BaseEntity 抽象类，ReleaseHistory **实体**，记录每次 Release **相关**的操作日志。代码如下：

```Java
@Entity
@Table(name = "ReleaseHistory")
@SQLDelete(sql = "Update ReleaseHistory set isDeleted = 1 where id = ?") // 标记删除
@Where(clause = "isDeleted = 0")
public class ReleaseHistory extends BaseEntity {

    /**
     * App 编号
     */
    @Column(name = "AppId", nullable = false)
    private String appId;
    /**
     * Cluster 名字
     */
    @Column(name = "ClusterName", nullable = false)
    private String clusterName;
    /**
     * Namespace 名字
     */
    @Column(name = "NamespaceName", nullable = false)
    private String namespaceName;
    /**
     * Branch 名
     *
     * 主干，使用 Cluster 名字
     * 分支，使用子 Cluster 名字
     */
    @Column(name = "BranchName", nullable = false)
    private String branchName;
    /**
     * Release 编号
     */
    @Column(name = "ReleaseId")
    private long releaseId;
    /**
     * 上一次 Release 编号
     */
    @Column(name = "PreviousReleaseId")
    private long previousReleaseId;
    /**
     * 操作类型 {@link com.ctrip.framework.apollo.common.constants.ReleaseOperation}
     */
    @Column(name = "Operation")
    private int operation;
    /**
     * 操作 Context
     */
    @Column(name = "OperationContext", nullable = false)
    private String operationContext;
    
}
```

* `appId` + `clusterName` + `namespaceName` 字段，指向对应的 Namespace 记录。
* `branchName` 字段，Branch 名字。
    * **主干**，使用 Cluster 名字。
    * **分支**，使用**子** Cluster 名字。
* `releaseId` 字段，Release 编号。
* `previousReleaseId` 字段，**上一次** Release 编号。
* `operation` 类型，操作类型。在 `com.ctrip.framework.apollo.common.constants.ReleaseOperation` 类中，枚举了所有发布相关的操作类型。代码如下：

    ```Java
    public interface ReleaseOperation {
    
        int NORMAL_RELEASE = 0; // 主干发布
        int ROLLBACK = 1; // 回滚
        int GRAY_RELEASE = 2; // 灰度发布
        int APPLY_GRAY_RULES = 3; //
        int GRAY_RELEASE_MERGE_TO_MASTER = 4;
        int MASTER_NORMAL_RELEASE_MERGE_TO_GRAY = 5;
        int MATER_ROLLBACK_MERGE_TO_GRAY = 6;
        int ABANDON_GRAY_RELEASE = 7;
        int GRAY_RELEASE_DELETED_AFTER_MERGE = 8;
    
    }
    ```

* `operationContext` 字段，操作 Context 。

## 2.3 ReleaseMessage

下一篇文章，详细分享。

# 3. Portal 侧

## 3.1 ReleaseController

在 `apollo-portal` 项目中，`com.ctrip.framework.apollo.portal.controller.ReleaseController` ，提供 Release 的 **API** 。

在【发布】的界面中，点击【 发布 】按钮，调用**发布配置的 API** 。

![发布配置](http://www.iocoder.cn/images/Apollo/2018_04_01/03.png)

`#createRelease(appId, env, clusterName, namespaceName, NamespaceReleaseModel)` 方法，发布配置。代码如下：

```Java
  1: @Autowired
  2: private ReleaseService releaseService;
  3: @Autowired
  4: private ApplicationEventPublisher publisher;
  5: @Autowired
  6: private PortalConfig portalConfig;
  7: 
  8: @PreAuthorize(value = "@permissionValidator.hasReleaseNamespacePermission(#appId, #namespaceName)")
  9: @RequestMapping(value = "/apps/{appId}/envs/{env}/clusters/{clusterName}/namespaces/{namespaceName}/releases", method = RequestMethod.POST)
 10: public ReleaseDTO createRelease(@PathVariable String appId,
 11:                                 @PathVariable String env, @PathVariable String clusterName,
 12:                                 @PathVariable String namespaceName, @RequestBody NamespaceReleaseModel model) {
 13:     // 校验 NamespaceReleaseModel 非空
 14:     checkModel(Objects.nonNull(model));
 15:     // 设置 PathVariable 变量到 NamespaceReleaseModel 中
 16:     model.setAppId(appId);
 17:     model.setEnv(env);
 18:     model.setClusterName(clusterName);
 19:     model.setNamespaceName(namespaceName);
 20:     // 若是紧急发布，但是当前环境未允许该操作，抛出 BadRequestException 异常
 21:     if (model.isEmergencyPublish() && !portalConfig.isEmergencyPublishAllowed(Env.valueOf(env))) {
 22:         throw new BadRequestException(String.format("Env: %s is not supported emergency publish now", env));
 23:     }
 24:     // 发布配置
 25:     ReleaseDTO createdRelease = releaseService.publish(model);
 26: 
 27:     // 创建 ConfigPublishEvent 对象
 28:     ConfigPublishEvent event = ConfigPublishEvent.instance();
 29:     event.withAppId(appId)
 30:             .withCluster(clusterName)
 31:             .withNamespace(namespaceName)
 32:             .withReleaseId(createdRelease.getId())
 33:             .setNormalPublishEvent(true)
 34:             .setEnv(Env.valueOf(env));
 35:     // 发布 ConfigPublishEvent 事件
 36:     publisher.publishEvent(event);
 37: 
 38:     return createdRelease;
 39: }
```

* **POST `/apps/{appId}/envs/{env}/clusters/{clusterName}/namespaces/{namespaceName}/releases` 接口**，Request Body 传递 **JSON** 对象。
* `@PreAuthorize(...)` 注解，调用 `PermissionValidator#hasReleaseNamespacePermissio(appId, namespaceName)` 方法，**校验**是否有发布配置的权限。后续文章，详细分享。
* `com.ctrip.framework.apollo.portal.entity.model.NamespaceReleaseModel` ，Namespace 配置发布 Model 。代码如下：

    ```Java
    public class NamespaceReleaseModel implements Verifiable {
    
        /**
         * App 编号
         */
        private String appId;
        /**
         * Env 名字
         */
        private String env;
        /**
         * Cluster 名字
         */
        private String clusterName;
        /**
         * Namespace 名字
         */
        private String namespaceName;
        /**
         * 发布标题
         */
        private String releaseTitle;
        /**
         * 发布描述
         */
        private String releaseComment;
        /**
         * 发布人
         */
        private String releasedBy;
        /**
         * 是否紧急发布
         */
        private boolean isEmergencyPublish;
    
        @Override
        public boolean isInvalid() {
            return StringUtils.isContainEmpty(appId, env, clusterName, namespaceName, releaseTitle); // 校验非空
        }
        
    }
    ```

* 第 14 行：**校验** NamespaceReleaseModel 非空。
* 第 15 至 19 行：**设置** PathVariable 变量到 NamespaceReleaseModel 中。
* 第 20 至 23 行：**校验**若是紧急发布，但是当前环境未允许该操作，抛出 BadRequestException 异常。
    * **紧急发布**功能，可通过设置 **PortalDB** 的 ServerConfig 的`"emergencyPublish.supported.envs"` 配置开启对应的 **Env 们**。例如，`emergencyPublish.supported.envs = dev` 。
* 第 25 行：调用 `ReleaseService#publish(NamespaceReleaseModel)` 方法，调用 Admin Service API ，发布配置。
* 第 27 至 36 行：创建 ConfigPublishEvent 对象，并调用 `ApplicationEventPublisher#publishEvent(event)` 方法，发布 ConfigPublishEvent 事件。这部分，我们在后续文章分享。
* 第 38 行：返回 **ReleaseDTO** 对象。

## 3.2 ReleaseService

在 `apollo-portal` 项目中，`com.ctrip.framework.apollo.portal.service.ReleaseService` ，提供 Release 的 **Service** 逻辑。

`#publish(NamespaceReleaseModel)` 方法，调用 Admin Service API ，发布配置。代码如下：

```Java
  1: @Autowired
  2: private UserInfoHolder userInfoHolder;
  3: @Autowired
  4: private AdminServiceAPI.ReleaseAPI releaseAPI;
  5: 
  6: public ReleaseDTO publish(NamespaceReleaseModel model) {
  7:     Env env = model.getEnv();
  8:     boolean isEmergencyPublish = model.isEmergencyPublish();
  9:     String appId = model.getAppId();
 10:     String clusterName = model.getClusterName();
 11:     String namespaceName = model.getNamespaceName();
 12:     String releaseBy = StringUtils.isEmpty(model.getReleasedBy()) ? userInfoHolder.getUser().getUserId() : model.getReleasedBy();
 13: 
 14:     // 调用 Admin Service API ，发布 Namespace 的配置。
 15:     ReleaseDTO releaseDTO = releaseAPI.createRelease(appId, env, clusterName, namespaceName,
 16:             model.getReleaseTitle(), model.getReleaseComment(),
 17:             releaseBy, isEmergencyPublish);
 18:     // 【TODO 6001】Tracer 日志
 19:     Tracer.logEvent(TracerEventType.RELEASE_NAMESPACE, String.format("%s+%s+%s+%s", appId, env, clusterName, namespaceName));
 20:     return releaseDTO;
 21: }
```

* 第14 至 17 行：调用 `ReleaseAPI#createRelease(appId, env, clusterName, namespaceName, releaseTitle, releaseComment, releaseBy, isEmergencyPublish)` 方法，调用 Admin Service API ，发布配置。
* 第 19 行：【TODO 6001】Tracer 日志

## 3.3 ReleaseAPI

`com.ctrip.framework.apollo.portal.api.ReleaseAPI` ，实现 API 抽象类，封装对 Admin Service 的 Release 模块的 API 调用。代码如下：

![ReleaseAPI](http://www.iocoder.cn/images/Apollo/2018_04_01/04.png)

# 4. Admin Service 侧

## 4.1 ReleaseController

在 `apollo-adminservice` 项目中， `com.ctrip.framework.apollo.adminservice.controller.ReleaseController` ，提供 Release 的 **API** 。

`#publish(appId, env, clusterName, namespaceName, releaseTitle, releaseComment, releaseBy, isEmergencyPublish)` 方法，发布 Namespace 的配置。代码如下：

```Java
  1: @Autowired
  2: private ReleaseService releaseService;
  3: @Autowired
  4: private NamespaceService namespaceService;
  5: @Autowired
  6: private MessageSender messageSender;
  7: 
  8: @Transactional
  9: @RequestMapping(path = "/apps/{appId}/clusters/{clusterName}/namespaces/{namespaceName}/releases", method = RequestMethod.POST)
 10: public ReleaseDTO publish(@PathVariable("appId") String appId,
 11:                           @PathVariable("clusterName") String clusterName,
 12:                           @PathVariable("namespaceName") String namespaceName,
 13:                           @RequestParam("name") String releaseName,
 14:                           @RequestParam(name = "comment", required = false) String releaseComment,
 15:                           @RequestParam("operator") String operator,
 16:                           @RequestParam(name = "isEmergencyPublish", defaultValue = "false") boolean isEmergencyPublish) {
 17:     // 校验对应的 Namespace 对象是否存在。若不存在，抛出 NotFoundException 异常
 18:     Namespace namespace = namespaceService.findOne(appId, clusterName, namespaceName);
 19:     if (namespace == null) {
 20:         throw new NotFoundException(String.format("Could not find namespace for %s %s %s", appId, clusterName, namespaceName));
 21:     }
 22:     // 发布 Namespace 的配置
 23:     Release release = releaseService.publish(namespace, releaseName, releaseComment, operator, isEmergencyPublish);
 24: 
 25:     // send release message
 26:     // 获得 Cluster 名
 27:     Namespace parentNamespace = namespaceService.findParentNamespace(namespace);
 28:     String messageCluster;
 29:     if (parentNamespace != null) { // 灰度发布
 30:         messageCluster = parentNamespace.getClusterName();
 31:     } else {
 32:         messageCluster = clusterName; // 使用请求的 ClusterName
 33:     }
 34:     // 发送 Release 消息
 35:     messageSender.sendMessage(ReleaseMessageKeyGenerator.generate(appId, messageCluster, namespaceName), Topics.APOLLO_RELEASE_TOPIC);
 36: 
 37:     // 将 Release 转换成 ReleaseDTO 对象
 38:     return BeanUtils.transfrom(ReleaseDTO.class, release);
 39: }
```

* **POST `/apps/{appId}/clusters/{clusterName}/namespaces/{namespaceName}/releases` 接口**，Request Body 传递 **JSON** 对象。
* 第 17 至 21 行：**校验**对应的 Namespace 对象是否存在。若不存在，抛出 NotFoundException 异常。
* 第 23 行：调用 `ReleaseService#publish(namespace, releaseName, releaseComment, operator, isEmergencyPublish)` 方法，发布 Namespace 的配置，返回  **Release** 对象。
* 第 26 至 33 行：获得发布消息的 **Cluster** 名字。
    * 第 27 行：调用 `NamespaceService#findParentNamespace(namespace)` 方法，获得**父** Namespace 对象。代码如下：

        ```Java
        @Autowired
        private ClusterService clusterService;
        @Autowired
        private ClusterService clusterService;
            
        public Namespace findParentNamespace(Namespace namespace) {
            String appId = namespace.getAppId();
            String namespaceName = namespace.getNamespaceName();
            // 获得 Cluster
            Cluster cluster = clusterService.findOne(appId, namespace.getClusterName());
            // 若为子 Cluster
            if (cluster != null && cluster.getParentClusterId() > 0) {
                // 获得父 Cluster
                Cluster parentCluster = clusterService.findOne(cluster.getParentClusterId());
                // 获得父 Namespace
                return findOne(appId, parentCluster.getName(), namespaceName);
            }
            return null;
        }
        
        public Namespace findOne(String appId, String clusterName, String namespaceName) {
            return namespaceRepository.findByAppIdAndClusterNameAndNamespaceName(appId, clusterName, namespaceName);
        }
        ```
        * 这块胖友可以先跳过，等看完后面**灰度发布**相关的内容，在回过头理解。
    * 第 29 至 30 行：若有**父** Namespace 对象，说明是**子** Namespace ( 灰度发布 )，则使用**父** Namespace 的 Cluster 名字。因为，客户端即使在灰度发布的情况下，也是使用 **父** Namespace 的 Cluster 名字。也就说，灰度发布，对客户端是透明无感知的。
    * 第 32 行：使用**请求**的 Cluster 名字。
* 第 35 行：调用 `MessageSender#sendMessage(String message, String channel)` 方法，发送发布消息。详细实现，下一篇文章详细解析。
* 第 38 行：调用 `BeanUtils#transfrom(Class<T> clazz, Object src)` 方法，将 Release **转换**成 ReleaseDTO 对象。

## 4.2 ReleaseService

在 `apollo-biz` 项目中，`com.ctrip.framework.apollo.biz.service.ReleaseService` ，提供 Release  的 **Service** 逻辑给 Admin Service 和 Config Service 。

### 4.2.1 publish

`#publish(namespace, releaseName, releaseComment, operator, isEmergencyPublish)` 方法，发布 Namespace 的配置。代码如下：

```Java
  1: private Gson gson = new Gson();
  2: 
  3: @Autowired
  4: private ReleaseRepository releaseRepository;
  5: @Autowired
  6: private ItemService itemService;
  7: @Autowired
  8: private AuditService auditService;
  9: @Autowired
 10: private NamespaceLockService namespaceLockService;
 11: @Autowired
 12: private NamespaceService namespaceService;
 13: @Autowired
 14: private ReleaseHistoryService releaseHistoryService;
 15: 
 16: @Transactional
 17: public Release publish(Namespace namespace, String releaseName, String releaseComment, String operator, boolean isEmergencyPublish) {
 18:     // 校验锁定
 19:     checkLock(namespace, isEmergencyPublish, operator);
 20:     // 获得 Namespace 的普通配置 Map
 21:     Map<String, String> operateNamespaceItems = getNamespaceItems(namespace);
 22:     // 获得父 Namespace
 23:     Namespace parentNamespace = namespaceService.findParentNamespace(namespace);
 24:     // 若有父 Namespace ，则是子 Namespace ，进行灰度发布
 25:     // branch release
 26:     if (parentNamespace != null) {
 27:         return publishBranchNamespace(parentNamespace, namespace, operateNamespaceItems, releaseName, releaseComment, operator, isEmergencyPublish);
 28:     }
 29:     // 获得子 Namespace 对象
 30:     Namespace childNamespace = namespaceService.findChildNamespace(namespace);
 31:     // 获得上一次，并且有效的 Release 对象
 32:     Release previousRelease = null;
 33:     if (childNamespace != null) {
 34:         previousRelease = findLatestActiveRelease(namespace);
 35:     }
 36:     // 创建操作 Context
 37:     // master release
 38:     Map<String, Object> operationContext = Maps.newHashMap();
 39:     operationContext.put(ReleaseOperationContext.IS_EMERGENCY_PUBLISH, isEmergencyPublish);
 40:     // 主干发布
 41:     Release release = masterRelease(namespace, releaseName, releaseComment, operateNamespaceItems, operator, ReleaseOperation.NORMAL_RELEASE, operationContext); // 是否紧急发布。
 42:     // 若有子 Namespace 时，自动将主干合并到子 Namespace ，并进行一次子 Namespace 的发布
 43:     // merge to branch and auto release
 44:     if (childNamespace != null) {
 45:         mergeFromMasterAndPublishBranch(namespace, childNamespace, operateNamespaceItems,
 46:                 releaseName, releaseComment, operator, previousRelease,
 47:                 release, isEmergencyPublish);
 48:     }
 49:     return release;
 50: }
```

* 第 19 行：调用 `#checkLock(namespace, isEmergencyPublish, operator)` 方法，**校验** NamespaceLock 锁定。代码如下：

    ```Java
    private void checkLock(Namespace namespace, boolean isEmergencyPublish, String operator) {
        if (!isEmergencyPublish) { // 非紧急发布
            // 获得 NamespaceLock 对象
            NamespaceLock lock = namespaceLockService.findLock(namespace.getId());
            // 校验锁定人是否是当前管理员。若是，抛出 BadRequestException 异常
            if (lock != null && lock.getDataChangeCreatedBy().equals(operator)) {
                throw new BadRequestException("Config can not be published by yourself.");
            }
        }
    }
    ```

* 第 21 行：调用 `#getNamespaceItems(namespace)` 方法，获得 Namespace 的**普通**配置 Map 。代码如下：

    ```Java
    private Map<String, String> getNamespaceItems(Namespace namespace) {
        // 读取 Namespace 的 Item 集合
        List<Item> items = itemService.findItemsWithoutOrdered(namespace.getId());
        // 生成普通配置 Map 。过滤掉注释和空行的配置项
        Map<String, String> configurations = new HashMap<String, String>();
        for (Item item : items) {
            if (StringUtils.isEmpty(item.getKey())) {
                continue;
            }
            configurations.put(item.getKey(), item.getValue());
        }
        return configurations;
    }
    ```

* 第 23 行：调用 `#findParentNamespace(namespace)` 方法，获得**父** Namespace 对象。
* 第 26 至 28 行：若有**父** Namespace 对象，**灰度发布**。详细解析，见 [《Apollo 源码解析 —— Portal 灰度发布》](http://www.iocoder.cn/Apollo/portal-publish-namespace-branch?self) 。
* 第 30 行：调用 `NamespaceService#findChildNamespace(namespace)` 方法，获得子 Namespace 对象。详细解析，见 [《Apollo 源码解析 —— Portal 创建灰度》](http://www.iocoder.cn/Apollo/portal-create-namespace-branch/?slef) 。
* 第 31 至 35 行：调用 `#findLatestActiveRelease(Namespace)` 方法，获得**上一次**，并且有效的 Release 对象。代码如下：

    ```Java
    public Release findLatestActiveRelease(Namespace namespace) {
        return findLatestActiveRelease(namespace.getAppId(), namespace.getClusterName(), namespace.getNamespaceName());
    }
    
    public Release findLatestActiveRelease(String appId, String clusterName, String namespaceName) {
        return releaseRepository.findFirstByAppIdAndClusterNameAndNamespaceNameAndIsAbandonedFalseOrderByIdDesc(appId,
                clusterName, namespaceName); // IsAbandoned = False && Id DESC
    }
    ```

* 第 36 至 39 行：创建操作 **Context** 。
* 第 41 行：调用 `#masterRelease(namespace, releaseName, releaseComment, operateNamespaceItems, operator, releaseOperation, operationContext)` 方法，**主干**发布配置。🙂 创建的 Namespace ，默认就是**主干**，而**灰度**发布使用的是**分支**。
* 第 42 至 48 行：调用 `#mergeFromMasterAndPublishBranch(...)` 方法，若有子 Namespace 时，自动将主干合并到子 Namespace ，并进行一次子 Namespace 的发布。
* 第 49 行：返回 **Release** 对象。详细解析，见 [《Apollo 源码解析 —— Portal 灰度发布》](http://www.iocoder.cn/Apollo/portal-publish-namespace-branch?self) 。

### 4.2.2 masterRelease

`#masterRelease(namespace, releaseName, releaseComment, operateNamespaceItems, operator, releaseOperation, operationContext)` 方法，主干发布配置。代码如下：

```Java
  1: private Release masterRelease(Namespace namespace, String releaseName, String releaseComment,
  2:                               Map<String, String> configurations, String operator,
  3:                               int releaseOperation, Map<String, Object> operationContext) {
  4:     // 获得最后有效的 Release 对象
  5:     Release lastActiveRelease = findLatestActiveRelease(namespace);
  6:     long previousReleaseId = lastActiveRelease == null ? 0 : lastActiveRelease.getId();
  7:     // 创建 Release 对象，并保存
  8:     Release release = createRelease(namespace, releaseName, releaseComment, configurations, operator);
  9: 
 10:     // 创建 ReleaseHistory 对象，并保存
 11:     releaseHistoryService.createReleaseHistory(namespace.getAppId(), namespace.getClusterName(),
 12:             namespace.getNamespaceName(), namespace.getClusterName(),
 13:             release.getId(), previousReleaseId, releaseOperation,
 14:             operationContext, operator);
 15:     return release;
 16: }
```

* 第 5 行：调用 `#findLatestActiveRelease(namespace)` 方法，获得**最后**、**有效**的 Release 对象。代码如下：

    ```Java
    public Release findLatestActiveRelease(Namespace namespace) {
        return findLatestActiveRelease(namespace.getAppId(), namespace.getClusterName(), namespace.getNamespaceName());
    }
    
    public Release findLatestActiveRelease(String appId, String clusterName, String namespaceName) {
        return releaseRepository.findFirstByAppIdAndClusterNameAndNamespaceNameAndIsAbandonedFalseOrderByIdDesc(appId,
                clusterName, namespaceName); // IsAbandoned = False && Id DESC
    }
    ```

* 第 8 行：调用 `#createRelease(namespace, releaseName, releaseComment, configurations, operator)` 方法，创建 **Release** 对象，并保存。
* 第10 至 14 行：调用 `ReleaseHistoryService#createReleaseHistory(appId, clusterName, namespaceName, branchName, releaseId, previousReleaseId, operation, operationContext, operator)` 方法，创建 **ReleaseHistory** 对象，并保存。

### 4.2.3 createRelease

`#createRelease(namespace, releaseName, releaseComment, configurations, operator)` 方法，创建 **Release** 对象，并保存。代码如下：

```Java
  1: private Release createRelease(Namespace namespace, String name, String comment,
  2:                               Map<String, String> configurations, String operator) {
  3:     // 创建 Release 对象
  4:     Release release = new Release();
  5:     release.setReleaseKey(ReleaseKeyGenerator.generateReleaseKey(namespace)); //【TODO 6006】Release Key 用途？
  6:     release.setDataChangeCreatedTime(new Date());
  7:     release.setDataChangeCreatedBy(operator);
  8:     release.setDataChangeLastModifiedBy(operator);
  9:     release.setName(name);
 10:     release.setComment(comment);
 11:     release.setAppId(namespace.getAppId());
 12:     release.setClusterName(namespace.getClusterName());
 13:     release.setNamespaceName(namespace.getNamespaceName());
 14:     release.setConfigurations(gson.toJson(configurations)); // 使用 Gson ，将配置 Map 格式化成字符串。
 15:     // 保存 Release 对象
 16:     release = releaseRepository.save(release);
 17:     // 释放 NamespaceLock
 18:     namespaceLockService.unlock(namespace.getId());
 19:     // 记录 Audit 到数据库中
 20:     auditService.audit(Release.class.getSimpleName(), release.getId(), Audit.OP.INSERT, release.getDataChangeCreatedBy());
 21:     return release;
 22: }
```

* 第 4 至 14 行：创建 Release 对象，并设置对应的属性。
    * 第 5 行：【TODO 6006】Release Key 用途？
    * 第 14 行：调用 `Gson#toJson(src)` 方法，将**配置 Map** 格式化成字符串。
* 第 16 行：调用 `ReleaseRepository#save(Release)` 方法，保存 Release 对象。
* 第 18 行：调用 `NamespaceLockService#unlock(namespaceId)` 方法，释放 NamespaceLock 。
* 第 20 行：记录 Audit 到数据库中。

## 4.3 ReleaseRepository

`com.ctrip.framework.apollo.biz.repository.ReleaseRepository` ，继承 `org.springframework.data.repository.PagingAndSortingRepository` 接口，提供 Release 的**数据访问** 给 Admin Service 和 Config Service 。代码如下：

```Java
public interface ReleaseRepository extends PagingAndSortingRepository<Release, Long> {

  Release findFirstByAppIdAndClusterNameAndNamespaceNameAndIsAbandonedFalseOrderByIdDesc(@Param("appId") String appId, @Param("clusterName") String clusterName, @Param("namespaceName") String namespaceName);

  Release findByIdAndIsAbandonedFalse(long id);

  List<Release> findByAppIdAndClusterNameAndNamespaceNameOrderByIdDesc(String appId, String clusterName, String namespaceName, Pageable page);

  List<Release> findByAppIdAndClusterNameAndNamespaceNameAndIsAbandonedFalseOrderByIdDesc(String appId, String clusterName, String namespaceName, Pageable page);

  List<Release> findByReleaseKeyIn(Set<String> releaseKey);

  List<Release> findByIdIn(Set<Long> releaseIds);

  @Modifying
  @Query("update Release set isdeleted=1,DataChange_LastModifiedBy = ?4 where appId=?1 and clusterName=?2 and namespaceName = ?3")
  int batchDelete(String appId, String clusterName, String namespaceName, String operator);

  // For release history conversion program, need to delete after conversion it done
  List<Release> findByAppIdAndClusterNameAndNamespaceNameOrderByIdAsc(String appId, String clusterName, String namespaceName);

}
```

## 4.4 ReleaseHistoryService

在 `apollo-biz` 项目中，`com.ctrip.framework.apollo.biz.service.ReleaseHistoryService` ，提供 ReleaseHistory  的 **Service** 逻辑给 Admin Service 和 Config Service 。

`#createReleaseHistory(appId, clusterName, namespaceName, branchName, releaseId, previousReleaseId, operation, operationContext, operator)` 方法，创建 **ReleaseHistory** 对象，并保存。代码如下：

```Java
  1: private Gson gson = new Gson();
  2: 
  3: @Autowired
  4: private ReleaseHistoryRepository releaseHistoryRepository;
  5: @Autowired
  6: private AuditService auditService;
  7: 
  8: @Transactional
  9: public ReleaseHistory createReleaseHistory(String appId, String clusterName, String namespaceName, String branchName,
 10:                                            long releaseId, long previousReleaseId, int operation,
 11:                                            Map<String, Object> operationContext, String operator) {
 12:     // 创建 ReleaseHistory 对象
 13:     ReleaseHistory releaseHistory = new ReleaseHistory();
 14:     releaseHistory.setAppId(appId);
 15:     releaseHistory.setClusterName(clusterName);
 16:     releaseHistory.setNamespaceName(namespaceName);
 17:     releaseHistory.setBranchName(branchName);
 18:     releaseHistory.setReleaseId(releaseId); // Release 编号
 19:     releaseHistory.setPreviousReleaseId(previousReleaseId); // 上一个 Release 编号
 20:     releaseHistory.setOperation(operation);
 21:     if (operationContext == null) {
 22:         releaseHistory.setOperationContext("{}"); //default empty object
 23:     } else {
 24:         releaseHistory.setOperationContext(gson.toJson(operationContext));
 25:     }
 26:     releaseHistory.setDataChangeCreatedTime(new Date());
 27:     releaseHistory.setDataChangeCreatedBy(operator);
 28:     releaseHistory.setDataChangeLastModifiedBy(operator);
 29:     // 保存 ReleaseHistory 对象
 30:     releaseHistoryRepository.save(releaseHistory);
 31:     // 记录 Audit 到数据库中
 32:     auditService.audit(ReleaseHistory.class.getSimpleName(), releaseHistory.getId(), Audit.OP.INSERT, releaseHistory.getDataChangeCreatedBy());
 33:     return releaseHistory;
 34: }
```

* 第 12 至 28 行：创建 ReleaseHistory 对象，并设置对应的属性。
* 第 30 行：调用 `ReleaseHistoryRepository#save(ReleaseHistory)` 方法，保存 ReleaseHistory 对象。
* 第 32 行：记录 Audit 到数据库中。

## 4.5 ReleaseHistoryRepository

`com.ctrip.framework.apollo.biz.repository.ReleaseHistoryRepository` ，继承 `org.springframework.data.repository.PagingAndSortingRepository` 接口，提供 ReleaseHistory 的**数据访问** 给 Admin Service 和 Config Service 。代码如下：

```Java
public interface ReleaseHistoryRepository extends PagingAndSortingRepository<ReleaseHistory, Long> {

    Page<ReleaseHistory> findByAppIdAndClusterNameAndNamespaceNameOrderByIdDesc(String appId, String
            clusterName, String namespaceName, Pageable pageable);

    Page<ReleaseHistory> findByReleaseIdAndOperationOrderByIdDesc(long releaseId, int operation, Pageable pageable);

    Page<ReleaseHistory> findByPreviousReleaseIdAndOperationOrderByIdDesc(long previousReleaseId, int operation, Pageable pageable);

    @Modifying
    @Query("update ReleaseHistory set isdeleted=1,DataChange_LastModifiedBy = ?4 where appId=?1 and clusterName=?2 and namespaceName = ?3")
    int batchDelete(String appId, String clusterName, String namespaceName, String operator);

}
```

# 666. 彩蛋

T T 终于要到比较干的地方啦。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)


