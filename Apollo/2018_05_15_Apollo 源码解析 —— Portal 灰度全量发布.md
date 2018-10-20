title: Apollo 源码解析 —— Portal 灰度全量发布
date: 2018-05-10
tags:
categories: Apollo
permalink: Apollo/portal-publish-namespace-branch-to-master

-------

摘要: 原创出处 http://www.iocoder.cn/Apollo/portal-publish-namespace-branch-to-master/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Apollo/portal-publish-namespace-branch-to-master/)
- [2. Portal 侧](http://www.iocoder.cn/Apollo/portal-publish-namespace-branch-to-master/)
  - [2.1 NamespaceBranchController](http://www.iocoder.cn/Apollo/portal-publish-namespace-branch-to-master/)
  - [2.2 NamespaceBranchService](http://www.iocoder.cn/Apollo/portal-publish-namespace-branch-to-master/)
  - [2.3 ReleaseAPI](http://www.iocoder.cn/Apollo/portal-publish-namespace-branch-to-master/)
- [3. Admin Service 侧](http://www.iocoder.cn/Apollo/portal-publish-namespace-branch-to-master/)
  - [3.1 ReleaseController](http://www.iocoder.cn/Apollo/portal-publish-namespace-branch-to-master/)
  - [3.2 ReleaseService](http://www.iocoder.cn/Apollo/portal-publish-namespace-branch-to-master/)
  - [3.3  NamespaceBranchService](http://www.iocoder.cn/Apollo/portal-publish-namespace-branch-to-master/)
  - [3.4 ClusterService](http://www.iocoder.cn/Apollo/portal-publish-namespace-branch-to-master/)
- [666. 彩蛋](http://www.iocoder.cn/Apollo/portal-publish-namespace-branch-to-master/)

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

> 老艿艿：本系列假定胖友已经阅读过 [《Apollo 官方 wiki 文档》](https://github.com/ctripcorp/apollo/wiki/)  ，特别是 [《Apollo 官方 wiki 文档 —— 灰度发布使用指南》](https://github.com/ctripcorp/apollo/wiki/Apollo%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97#%E4%BA%94%E7%81%B0%E5%BA%A6%E5%8F%91%E5%B8%83%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97)。

本文接 [《Apollo 源码解析 —— Portal 灰度发布》](http://www.iocoder.cn/Apollo/portal-publish-namespace-branch?self) ，分享灰度**全量**发布。

我们先来看看官方文档对**灰度全量发布**的使用指南，来理解下它的定义和流程。

> 如果灰度的配置测试下来比较理想，符合预期，那么就可以操作【全量发布】。
> 
> 全量发布的效果是：
> 
> 1. 灰度版本的配置会合并回主版本，在这个例子中，就是主版本的 timeout 会被更新成 3000
> 2. 主版本的配置会自动进行一次发布
> 3. 在全量发布页面，可以选择是否保留当前灰度版本，默认为不保留。
> 
> ![灰度发布1](http://www.iocoder.cn/images/Apollo/2018_05_15/01.png)  
> ![全量发布1](http://www.iocoder.cn/images/Apollo/2018_05_15/02.png)  
> ![全量发布2](http://www.iocoder.cn/images/Apollo/2018_05_15/03.png)  
> 
> 我选择了不保留灰度版本，所以发布完的效果就是主版本的配置更新、灰度版本删除。点击主版本的实例列表，可以看到10.32.21.22和10.32.21.19都使用了主版本最新的配置。
> 
> ![灰度发布2](http://www.iocoder.cn/images/Apollo/2018_05_15/04.png)


灰度**全量**发布，和 [《Apollo 源码解析 —— Portal 发布配置》](http://www.iocoder.cn/Apollo/portal-publish/?self) ，差异点在于，**多了一步配置合并**，所以代码实现上，有很多相似度。整体系统流程如下：

 ![流程](http://www.iocoder.cn/images/Apollo/2018_05_15/06.png)  

# 2. Portal 侧

## 2.1 NamespaceBranchController

在 `apollo-portal` 项目中，`com.ctrip.framework.apollo.portal.controller.NamespaceBranchController` ，提供 Namespace **分支**的 **API** 。

`#merge(...)` 方法，灰度**全量**发布，**合并**子 Namespace 变更的配置 Map 到父 Namespace ，并进行一次 **Release** 。代码如下：

```Java
  1: @PreAuthorize(value = "@permissionValidator.hasReleaseNamespacePermission(#appId, #namespaceName)")
  2: @RequestMapping(value = "/apps/{appId}/envs/{env}/clusters/{clusterName}/namespaces/{namespaceName}/branches/{branchName}/merge", method = RequestMethod.POST)
  3: public ReleaseDTO merge(@PathVariable String appId, @PathVariable String env,
  4:                         @PathVariable String clusterName, @PathVariable String namespaceName,
  5:                         @PathVariable String branchName, @RequestParam(value = "deleteBranch", defaultValue = "true") boolean deleteBranch,
  6:                         @RequestBody NamespaceReleaseModel model) {
  7:     // 若是紧急发布，但是当前环境未允许该操作，抛出 BadRequestException 异常
  8:     if (model.isEmergencyPublish() && !portalConfig.isEmergencyPublishAllowed(Env.fromString(env))) {
  9:         throw new BadRequestException(String.format("Env: %s is not supported emergency publish now", env));
 10:     }
 11:     // 合并子 Namespace 变更的配置 Map 到父 Namespace ，并进行一次 Release
 12:     ReleaseDTO createdRelease = namespaceBranchService.merge(appId, Env.valueOf(env), clusterName, namespaceName, branchName,
 13:             model.getReleaseTitle(), model.getReleaseComment(),
 14:             model.isEmergencyPublish(), deleteBranch);
 15: 
 16:     // 创建 ConfigPublishEvent 对象
 17:     ConfigPublishEvent event = ConfigPublishEvent.instance();
 18:     event.withAppId(appId)
 19:             .withCluster(clusterName)
 20:             .withNamespace(namespaceName)
 21:             .withReleaseId(createdRelease.getId())
 22:             .setMergeEvent(true)
 23:             .setEnv(Env.valueOf(env));
 24:     // 发布 ConfigPublishEvent 事件
 25:     publisher.publishEvent(event);
 26:     return createdRelease;
 27: }
```

* **POST `/apps/{appId}/envs/{env}/clusters/{clusterName}/namespaces/{namespaceName}/branches/{branchName}/merge` 接口**，Request Body 传递 **JSON** 对象。
* `@PreAuthorize(...)` 注解，调用 `PermissionValidator#hasReleaseNamespacePermissio(appId, namespaceName)` 方法，**校验**是否有发布配置的权限。后续文章，详细分享。
* 第 7 至 10 行：**校验**若是紧急发布，但是当前环境未允许该操作，抛出 BadRequestException 异常。
* 第 11 至 14 行：调用 `NamespaceBranchService#merge(...)` 方法，合并子 Namespace 变更的配置 Map 到父 Namespace ，并进行一次 Release 。
* 第 16 至 25 行：创建 ConfigPublishEvent 对象，并调用 `ApplicationEventPublisher#publishEvent(event)` 方法，发布 ConfigPublishEvent 事件。这部分，我们在后续文章分享。
* 第 26 行：返回 **ReleaseDTO** 对象。

## 2.2 NamespaceBranchService

在 `apollo-portal` 项目中，`com.ctrip.framework.apollo.portal.service.NamespaceBranchService` ，提供 Namespace **分支**的 **Service** 逻辑。

`#merge(...)` 方法，调用 Admin Service API ，**合并**子 Namespace 变更的配置 Map 到父 Namespace ，并进行一次 **Release** 。代码如下：

```Java
  1: @Autowired
  2: private AdminServiceAPI.NamespaceBranchAPI namespaceBranchAPI;
  3: @Autowired
  4: private ReleaseService releaseService;
  5: 
  6: public ReleaseDTO merge(String appId, Env env, String clusterName, String namespaceName,
  7:                         String branchName, String title, String comment,
  8:                         boolean isEmergencyPublish, boolean deleteBranch) {
  9:     // 计算变化的 Item 集合
 10:     ItemChangeSets changeSets = calculateBranchChangeSet(appId, env, clusterName, namespaceName, branchName);
 11:     // 合并子 Namespace 变更的配置 Map 到父 Namespace ，并进行一次 Release
 12:     ReleaseDTO mergedResult = releaseService.updateAndPublish(appId, env, clusterName, namespaceName, title, comment,
 13:                     branchName, isEmergencyPublish, deleteBranch, changeSets);
 14:     // 【TODO 6001】Tracer 日志
 15:     Tracer.logEvent(TracerEventType.MERGE_GRAY_RELEASE, String.format("%s+%s+%s+%s", appId, env, clusterName, namespaceName));
 16:     return mergedResult;
 17: }
```

* 第 10 行：调用 `#calculateBranchChangeSet(appId, env, clusterName, namespaceName, branchName)` 方法，计算变化的 Item 集合。详细解析，见 [「2.2.1 calculateBranchChangeSet」](#) 。
* 第12 至 13 行：调用 `ReleaseService#updateAndPublish(...)` 方法，调用 Admin Service API ，**合并**子 Namespace 变更的配置 Map 到父 Namespace ，并进行一次 **Release** 。代码如下：

    ```Java
    @Autowired
    private AdminServiceAPI.ReleaseAPI releaseAPI;
    
    public ReleaseDTO updateAndPublish(String appId, Env env, String clusterName, String namespaceName,
                                       String releaseTitle, String releaseComment, String branchName,
                                       boolean isEmergencyPublish, boolean deleteBranch, ItemChangeSets changeSets) {
        return releaseAPI.updateAndPublish(appId, env, clusterName, namespaceName, releaseTitle, releaseComment, branchName,
                isEmergencyPublish, deleteBranch, changeSets);
    }
    ```
    * 方法内部，调用 `ReleaseAPI#updateAndPublish(...)` 方法，调用 Admin Service API ，合并子 Namespace 变更的配置 Map 到父 Namespace ，并进行一次 Release 。🙂 可能会有胖友会问，为什么不 NamespaceBranchService 直接调用 ReleaseAPI 呢？ReleaseAPI 属于 ReleaseService 模块，对外**透明**、**屏蔽**该细节。这样，未来 ReleaseService 想要改实现，可能不是调用 ReleaseAPI 的方法，而是别的方法，也是非常方便的。

* 第 15 行：【TODO 6001】Tracer 日志

### 2.2.1 calculateBranchChangeSet

```Java
  1: @Autowired
  2: private ItemsComparator itemsComparator;
  3: @Autowired
  4: private UserInfoHolder userInfoHolder;
  5: @Autowired
  6: private NamespaceService namespaceService;
  7: @Autowired
  8: private ItemService itemService;
  9: 
 10: private ItemChangeSets calculateBranchChangeSet(String appId, Env env, String clusterName, String namespaceName, String branchName) {
 11:     // 获得父 NamespaceBO 对象
 12:     NamespaceBO parentNamespace = namespaceService.loadNamespaceBO(appId, env, clusterName, namespaceName);
 13:     // 若父 Namespace 不存在，抛出 BadRequestException 异常。
 14:     if (parentNamespace == null) {
 15:         throw new BadRequestException("base namespace not existed");
 16:     }
 17:     // 若父 Namespace 有配置项的变更，不允许合并。因为，可能存在冲突。
 18:     if (parentNamespace.getItemModifiedCnt() > 0) {
 19:         throw new BadRequestException("Merge operation failed. Because master has modified items");
 20:     }
 21:     // 获得父 Namespace 的 Item 数组
 22:     List<ItemDTO> masterItems = itemService.findItems(appId, env, clusterName, namespaceName);
 23:     // 获得子 Namespace 的 Item 数组
 24:     List<ItemDTO> branchItems = itemService.findItems(appId, env, branchName, namespaceName);
 25:     // 计算变化的 Item 集合
 26:     ItemChangeSets changeSets = itemsComparator.compareIgnoreBlankAndCommentItem(parentNamespace.getBaseInfo().getId(), masterItems, branchItems);
 27:     // 设置 `ItemChangeSets.deleteItem` 为空。因为子 Namespace 从父 Namespace 继承配置，但是实际自己没有那些配置项，所以如果不清空，会导致这些配置项被删除。
 28:     changeSets.setDeleteItems(Collections.emptyList());
 29:     // 设置 `ItemChangeSets.dataChangeLastModifiedBy` 为当前管理员
 30:     changeSets.setDataChangeLastModifiedBy(userInfoHolder.getUser().getUserId());
 31:     return changeSets;
 32: }
```

* 第 11 至 20 行，父 Namespace 相关
    * 第 12 行：调用 [`namespaceService#loadNamespaceBO(appId, env, clusterName, namespaceName)`](https://github.com/YunaiV/apollo/blob/80166648912b03667cd92e234e93927e1bb096ff/apollo-portal/src/main/java/com/ctrip/framework/apollo/portal/service/NamespaceService.java#L147-L310) 方法，获得父 [**NamespaceBO**](https://github.com/YunaiV/apollo/blob/80166648912b03667cd92e234e93927e1bb096ff/apollo-portal/src/main/java/com/ctrip/framework/apollo/portal/entity/bo/NamespaceBO.java) 对象。该对象，包含了 Namespace 的**详细**数据，包括 Namespace 的基本信息、配置集合。详细解析，点击方法链接查看，笔者已经添加详细注释。方法比较**冗长**，胖友耐心阅读，其目的是为了【第 17 至 20 行】的判断，是否有**未发布**的配置变更。
    * 第 13 至 16 行：若**父** Namespace 不存在，抛出 BadRequestException 异常。
    * 第 17 至 20 行：若**父** Namespace 有**未发布**的配置变更，不允许合并。因为，可能存在冲突，无法自动解决。此时，需要在 Portal 上将**父** Namespace 的配置进行一次发布，或者回退回历史版本。
* 第 21 至 30 行：获得配置变更集合 ItemChangeSets 对象。该对象，我们在 [《Apollo 源码解析 —— Portal 批量变更 Item》](http://www.iocoder.cn/Apollo/portal-update-item-set/?self) 。
    * 第 22 行：调用 `ItemService#findItems(appId, env, clusterName, namespaceName)`  方法，获得**父** Namespace 的 ItemDTO 数组。
    * 第 24 行：调用 `ItemService#findItems(appId, env, branchName, namespaceName)`  方法，获得**子** Namespace 的 ItemDTO 数组。
    * 第 26 行：调用 [`ItemsComparator#compareIgnoreBlankAndCommentItem(baseNamespaceId, baseItems, targetItems)`](https://github.com/YunaiV/apollo/blob/80166648912b03667cd92e234e93927e1bb096ff/apollo-portal/src/main/java/com/ctrip/framework/apollo/portal/component/ItemsComparator.java) 方法，计算**变化**的 Item 集合。详细解析，点击方法链接查看，笔者已经添加详细注释。
    * 第 28 行：设置 `ItemChangeSets.deleteItem` 为**空**。因为**子** Namespace 从**父** Namespace 继承配置，但是实际自己没有那些配置项，所以如果不设置为空，会导致合并时，这些配置项被删除。

## 2.3 ReleaseAPI

`com.ctrip.framework.apollo.portal.api.ReleaseAPI` ，实现 API 抽象类，封装对 Admin Service 的 Release 模块的 API 调用。代码如下：

![ReleaseAPI](http://www.iocoder.cn/images/Apollo/2018_05_15/05.png)

# 3. Admin Service 侧

## 3.1 ReleaseController

在 `apollo-adminservice` 项目中， `com.ctrip.framework.apollo.adminservice.controller.ReleaseController` ，提供 Release 的 **API** 。

`#updateAndPublish(...)` 方法，**合并**子 Namespace 变更的配置 Map 到父 Namespace ，并进行一次 **Release** 。代码如下：

```Java
  1: /**
  2:  * merge branch items to master and publish master
  3:  *
  4:  * @return published result
  5:  */
  6: @Transactional
  7: @RequestMapping(path = "/apps/{appId}/clusters/{clusterName}/namespaces/{namespaceName}/updateAndPublish", method = RequestMethod.POST)
  8: public ReleaseDTO updateAndPublish(@PathVariable("appId") String appId,
  9:                                    @PathVariable("clusterName") String clusterName,
 10:                                    @PathVariable("namespaceName") String namespaceName,
 11:                                    @RequestParam("releaseName") String releaseName,
 12:                                    @RequestParam("branchName") String branchName,
 13:                                    @RequestParam(value = "deleteBranch", defaultValue = "true") boolean deleteBranch, // 是否删除 Namespace 分支
 14:                                    @RequestParam(name = "releaseComment", required = false) String releaseComment,
 15:                                    @RequestParam(name = "isEmergencyPublish", defaultValue = "false") boolean isEmergencyPublish,
 16:                                    @RequestBody ItemChangeSets changeSets) {
 17:     // 获得 Namespace
 18:     Namespace namespace = namespaceService.findOne(appId, clusterName, namespaceName);
 19:     if (namespace == null) {
 20:         throw new NotFoundException(String.format("Could not find namespace for %s %s %s", appId, clusterName, namespaceName));
 21:     }
 22:     // 合并子 Namespace 变更的配置 Map 到父 Namespace ，并进行一次 Release
 23:     Release release = releaseService.mergeBranchChangeSetsAndRelease(namespace, branchName, releaseName, releaseComment, isEmergencyPublish, changeSets);
 24:     // 若需要删除子 Namespace ，则进行删除
 25:     if (deleteBranch) {
 26:         namespaceBranchService.deleteBranch(appId, clusterName, namespaceName, branchName, NamespaceBranchStatus.MERGED, changeSets.getDataChangeLastModifiedBy());
 27:     }
 28:     // 发送 Release 消息
 29:     messageSender.sendMessage(ReleaseMessageKeyGenerator.generate(appId, clusterName, namespaceName), Topics.APOLLO_RELEASE_TOPIC);
 30:     // 将 Release 转换成 ReleaseDTO 对象
 31:     return BeanUtils.transfrom(ReleaseDTO.class, release);
 32: }
```

* 第 17 至 21 行：调用 `NamespaceService#findOne(ppId, clusterName, namespaceName)` 方法，获得**父** Namespace 对象。
    * 若校验到不存在，抛出  NotFoundException 异常。
* 第 23 行：调用 `ReleaseService#mergeBranchChangeSetsAndRelease(...)` 方法，合并**子** Namespace 变更的配置 Map 到**父** Namespace ，并进行一次 **Release** 。详细解析，见 [「3.2 ReleaseService」](#) 。
* 第 25 至 27 行：若需要**删除**子 Namespace ，即 Portal 中选择【删除灰度版本】，调用 `NamespaceBranchService#deleteBranch(...)` 方法，删除**子** Namespace 相关的记录。详细解析，见 [「3.3  NamespaceBranchService」](#) 。
* 第 29 行：调用 `MessageSender#sendMessage(String message, String channel)` 方法，发送发布消息。
* 第 31 行：调用 `BeanUtils#transfrom(Class<T> clazz, Object src)` 方法，将 Release **转换**成 ReleaseDTO 对象。

## 3.2 ReleaseService

在 `apollo-biz` 项目中，`com.ctrip.framework.apollo.biz.service.ReleaseService` ，提供 Release  的 **Service** 逻辑给 Admin Service 和 Config Service 。

### 3.2.1 mergeBranchChangeSetsAndRelease

`ReleaseService#mergeBranchChangeSetsAndRelease(...)` 方法，合并**子** Namespace 变更的配置 Map 到**父** Namespace ，并进行一次 **Release** 。代码如下：

```Java
  1: // 合并子 Namespace 变更的配置 Map 到父 Namespace ，并进行一次 Release
  2: @Transactional
  3: public Release mergeBranchChangeSetsAndRelease(Namespace namespace, String branchName, String releaseName,
  4:                                                String releaseComment, boolean isEmergencyPublish,
  5:                                                ItemChangeSets changeSets) {
  6:     // 校验锁定
  7:     checkLock(namespace, isEmergencyPublish, changeSets.getDataChangeLastModifiedBy());
  8:     // 变更的配置集 合 ItemChangeSets 对象，更新到父 Namespace 中。
  9:     itemSetService.updateSet(namespace, changeSets);
 10: 
 11:     // 获得子 Namespace 的最新且有效的 Release 对象
 12:     Release branchRelease = findLatestActiveRelease(namespace.getAppId(), branchName, namespace.getNamespaceName());
 13:     // 获得子 Namespace 的最新且有效的 Release 编号
 14:     long branchReleaseId = branchRelease == null ? 0 : branchRelease.getId();
 15: 
 16:     // 获得父 Namespace 的配置 Map
 17:     Map<String, String> operateNamespaceItems = getNamespaceItems(namespace);
 18: 
 19:     // 创建 Map ，用于 ReleaseHistory 对象的 `operationContext` 属性。
 20:     Map<String, Object> operationContext = Maps.newHashMap();
 21:     operationContext.put(ReleaseOperationContext.SOURCE_BRANCH, branchName);
 22:     operationContext.put(ReleaseOperationContext.BASE_RELEASE_ID, branchReleaseId);
 23:     operationContext.put(ReleaseOperationContext.IS_EMERGENCY_PUBLISH, isEmergencyPublish);
 24: 
 25:     // 父 Namespace 进行发布
 26:     return masterRelease(namespace, releaseName, releaseComment, operateNamespaceItems,
 27:             changeSets.getDataChangeLastModifiedBy(),
 28:             ReleaseOperation.GRAY_RELEASE_MERGE_TO_MASTER, operationContext);
 29: }
```

* 第 7 行：调用 `#checkLock(...)` 方法，**校验**锁定。
* 第 9 行：调用 `ItemService#updateSet(namespace, changeSets)` 方法，将变更的配置集 合 ItemChangeSets 对象，更新到**父** Namespace 中。详细解析，在 [《Apollo 源码解析 —— Portal 批量变更 Item》](http://www.iocoder.cn/Apollo/portal-update-item-set/?self) 中。
    * 第 17 行：调用 `#getNamespaceItems(namespace)` 方法，获得**父** Namespace 的配置 Map 。因为上面已经更新过，所以获得到的是**合并后**的结果。
* 第 11 至 23 行：创建 Map ，并设置需要的 KV ，用于 ReleaseHistory 对象的 `operationContext` 属性。
    * 第 12 行：调用 `#findLatestActiveRelease(...)` 方法，获得**子** Namespace 的**最新**且**有效**的 Release 对象。
    * 第 14 行：获得**子** Namespace 的**最新**且**有效**的 Release 编号。
    * 第 21 至 23 行：设置 KV 到 Map 中。
* 第 26 至 28 行：调用 `#masterRelease(...)` 方法，**父** Namespace 进行发布。这块，和 [《Apollo 源码解析 —— Portal 发布配置》](http://www.iocoder.cn/Apollo/portal-publish/?self) 的逻辑就统一了，所以详细解析，见该文。

## 3.3  NamespaceBranchService

在 `apollo-biz` 项目中，`com.ctrip.framework.apollo.biz.service.NamespaceBranchService` ，提供 Namespace **分支**的 **Service** 逻辑给 Admin Service 和 Config Service 。

### 3.3.1 deleteBranch

`#deleteBranch(...)` 方法，删除**子** Namespace 相关的记录。代码如下：

```Java
  1: @Transactional
  2: public void deleteBranch(String appId, String clusterName, String namespaceName,
  3:                          String branchName, int branchStatus, String operator) {
  4:     // 获得子 Cluster 对象
  5:     Cluster toDeleteCluster = clusterService.findOne(appId, branchName);
  6:     if (toDeleteCluster == null) {
  7:         return;
  8:     }
  9:     // 获得子 Namespace 的最后有效的 Release 对象
 10:     Release latestBranchRelease = releaseService.findLatestActiveRelease(appId, branchName, namespaceName);
 11:     // 获得子 Namespace 的最后有效的 Release 对象的编号
 12:     long latestBranchReleaseId = latestBranchRelease != null ? latestBranchRelease.getId() : 0;
 13: 
 14:     // 创建新的，用于表示删除的 GrayReleaseRule 的对象
 15:     // update branch rules
 16:     GrayReleaseRule deleteRule = new GrayReleaseRule();
 17:     deleteRule.setRules("[]");
 18:     deleteRule.setAppId(appId);
 19:     deleteRule.setClusterName(clusterName);
 20:     deleteRule.setNamespaceName(namespaceName);
 21:     deleteRule.setBranchName(branchName);
 22:     deleteRule.setBranchStatus(branchStatus); // Namespace 分支状态
 23:     deleteRule.setDataChangeLastModifiedBy(operator);
 24:     deleteRule.setDataChangeCreatedBy(operator);
 25:     // 更新 GrayReleaseRule
 26:     doUpdateBranchGrayRules(appId, clusterName, namespaceName, branchName, deleteRule, false, -1);
 27: 
 28:     // 删除子 Cluster
 29:     // delete branch cluster
 30:     clusterService.delete(toDeleteCluster.getId(), operator);
 31: 
 32:     // 创建 ReleaseHistory 对象，并保存
 33:     int releaseOperation = branchStatus == NamespaceBranchStatus.MERGED ? ReleaseOperation.GRAY_RELEASE_DELETED_AFTER_MERGE : ReleaseOperation.ABANDON_GRAY_RELEASE;
 34:     releaseHistoryService.createReleaseHistory(appId, clusterName, namespaceName, branchName, latestBranchReleaseId, latestBranchReleaseId,
 35:             releaseOperation, null, operator);
 36:     // 记录 Audit 到数据库中
 37:     auditService.audit("Branch", toDeleteCluster.getId(), Audit.OP.DELETE, operator);
 38: }
```

* 第 4 至 8 行：调用 `ClusterService#findOne(appId, branchName)` 方法，获得**子** Cluster 对象。
* 第 10 行：调用 `ReleaseService#findLatestActiveRelease(namespace)` 方法，获得**最后**、**有效**的 Release 对象。
    * 第 12 行：获得**最后**、**有效**的 Release 对象的编号。
* 第 14 至 24 行：创建**新的**，用于**表示**删除的 GrayReleaseRule 的对象。并且，当前场景，该 GrayReleaseRule 的 `branchStatus` 为 **MERGED** 。
    * 第 26 行：调用 `#doUpdateBranchGrayRules(...)` 方法，更新 GrayReleaseRule 。详细解析，见 [《Apollo 源码解析 —— Portal 配置灰度规则》](http://www.iocoder.cn/Apollo/portal-modify-namespace-branch-gray-rules/?self) 中。
* 第 30 行：调用 `ClusterService#delte(id, operator)` 方法，删除子 Cluster 相关。详细解析，见 [「3.4 ClusterService」](#) 。
* 第 32 至 35 行：调用 `ReleaseHistoryService#createReleaseHistory(...)` 方法，创建 **ReleaseHistory** 对象，并保存。
* 第 37 行：记录 Audit 到数据库中。

## 3.4 ClusterService

在 `apollo-biz` 项目中，`com.ctrip.framework.apollo.biz.service.ClusterService` ，提供 Cluster 的 **Service** 逻辑给 Admin Service 和 Config Service 。

### 3.4.1 delete

`#delete(...)` 方法，删除 Cluster 相关。代码如下：

```Java
@Transactional
public void delete(long id, String operator) {
    // 获得 Cluster 对象
    Cluster cluster = clusterRepository.findOne(id);
    if (cluster == null) {
        throw new BadRequestException("cluster not exist");
    }
    // 删除 Namespace
    // delete linked namespaces
    namespaceService.deleteByAppIdAndClusterName(cluster.getAppId(), cluster.getName(), operator);

    // 标记删除 Cluster
    cluster.setDeleted(true);
    cluster.setDataChangeLastModifiedBy(operator);
    clusterRepository.save(cluster);

    // 记录 Audit 到数据库中
    auditService.audit(Cluster.class.getSimpleName(), id, Audit.OP.DELETE, operator);
}
```

* 会**标记**删除 Cluster 和其相关的 Namespace 。代码比较简单，胖友自己看看哈。

# 666. 彩蛋

灰度发布结束~还有一些其他流程，胖友可以自己看看，例如**放弃灰度**。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

