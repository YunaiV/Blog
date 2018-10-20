title: Apollo 源码解析 —— Portal 创建灰度
date: 2018-05-01
tags:
categories: Apollo
permalink: Apollo/portal-create-namespace-branch

-------

摘要: 原创出处 http://www.iocoder.cn/Apollo/portal-create-namespace-branch/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Apollo/portal-create-namespace-branch/)
- [2. Portal](http://www.iocoder.cn/Apollo/portal-create-namespace-branch/)
  - [2.1 NamespaceBranchController](http://www.iocoder.cn/Apollo/portal-create-namespace-branch/)
  - [2.2 NamespaceBranchService](http://www.iocoder.cn/Apollo/portal-create-namespace-branch/)
  - [2.3 NamespaceBranchAPI](http://www.iocoder.cn/Apollo/portal-create-namespace-branch/)
- [3. Admin Service](http://www.iocoder.cn/Apollo/portal-create-namespace-branch/)
  - [3.1 NamespaceBranchController](http://www.iocoder.cn/Apollo/portal-create-namespace-branch/)
  - [3.2 NamespaceBranchService](http://www.iocoder.cn/Apollo/portal-create-namespace-branch/)

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

本文分享 **Portal 创建灰度** 的流程，整个过程涉及 Portal、Admin Service ，如下图所示：![流程](http://www.iocoder.cn/images/Apollo/2018_05_01/01.png)

创建灰度，调用的是创建 Namespace **分支** 的 API 。通过创建的子 Namespace ，可以关联其自己定义的 Cluster、Item、Release 等等。关系如下所图所示：![关系图](http://www.iocoder.cn/images/Apollo/2018_05_01/02.png)

* 创建 Namespace **分支**时：
    * 会创建**子** Cluster ，指向**父** Cluster 。
    * 会创建**子** Namespace ，关联**子** Namespace 。实际上，**子** Namespace 和 **父** Namespace 无**任何**数据字段上的关联。
* 向**子** Namespace 添加 Item 时，该 Item 指向**子** Namespace 。虽然，代码实现和**父** Namespace 是**一模一样**的。
* **子** Namespace 发布( *灰度发布* ) 和 **父** Namespace 发布( *普通发布* ) 在代码实现，有一些差距，后续文章分享。

> 老艿艿：在目前 Apollo 的实现上，胖友可以把**分支**和**灰度**等价。
> 
> * 所以下文在用词时，选择使用**分支**。
> * 所以下文在用词时，选择使用**分支**。
> * 所以下文在用词时，选择使用**分支**。

🙂 这样的设计，巧妙。

# 2. Portal 侧

## 2.1 NamespaceBranchController

在 `apollo-portal` 项目中，`com.ctrip.framework.apollo.portal.controller.NamespaceBranchController` ，提供 Namespace **分支**的 **API** 。

> 首先点击 application namespace 右上角的【创建灰度】按钮。
> 
> ![创建灰度](http://www.iocoder.cn/images/Apollo/2018_05_01/03.png)
> 
> 点击确定后，灰度版本就创建成功了，页面会自动切换到【灰度版本】 Tab 。
> 
> ![灰度版本](http://www.iocoder.cn/images/Apollo/2018_05_01/04.png)

`#createBranch(...)` 方法，创建 Namespace **分支**。代码如下：

```Java
@RestController
public class NamespaceBranchController {

    @Autowired
    private NamespaceBranchService namespaceBranchService;
    
    @PreAuthorize(value = "@permissionValidator.hasModifyNamespacePermission(#appId, #namespaceName)")
    @RequestMapping(value = "/apps/{appId}/envs/{env}/clusters/{clusterName}/namespaces/{namespaceName}/branches", method = RequestMethod.POST)
    public NamespaceDTO createBranch(@PathVariable String appId,
                                     @PathVariable String env,
                                     @PathVariable String clusterName,
                                     @PathVariable String namespaceName) {
        return namespaceBranchService.createBranch(appId, Env.valueOf(env), clusterName, namespaceName);
    }
    
    // ... 省略其他接口和属性
}
```

* * **POST `"/apps/{appId}/envs/{env}/clusters/{clusterName}/namespaces/{namespaceName}/branches` 接口** 。
* `@PreAuthorize(...)` 注解，调用 `PermissionValidator#hasModifyNamespacePermission(appId, namespaceName)` 方法，校验是否有**修改** Namespace 的权限。后续文章，详细分享。
* 调用 `NamespaceBranchService#createBranch(...)` 方法，创建 Namespace **分支**。

## 2.2 NamespaceBranchService

在 `apollo-portal` 项目中，`com.ctrip.framework.apollo.portal.service.NamespaceBranchService` ，提供 Namespace **分支**的 **Service** 逻辑。

`#createItem(appId, env, clusterName, namespaceName, ItemDTO)` 方法，创建并保存 Item 到 Admin Service 。代码如下：

```Java
  1: @Autowired
  2: private UserInfoHolder userInfoHolder;
  3: @Autowired
  4: private AdminServiceAPI.NamespaceBranchAPI namespaceBranchAPI;
  5: 
  6: @Transactional
  7: public NamespaceDTO createBranch(String appId, Env env, String parentClusterName, String namespaceName) {
  8:     // 创建 Namespace 分支
  9:     NamespaceDTO createdBranch = namespaceBranchAPI.createBranch(appId, env, parentClusterName, namespaceName, userInfoHolder.getUser().getUserId());
 10:     // 【TODO 6001】Tracer 日志
 11:     Tracer.logEvent(TracerEventType.CREATE_GRAY_RELEASE, String.format("%s+%s+%s+%s", appId, env, parentClusterName, namespaceName));
 12:     return createdBranch;
 13: }
```

* 第 9 行：调用 `NamespaceBranchAPI#createBranch(...)` 方法，创建 Namespace **分支**。
* 第 11 行：【TODO 6001】Tracer 日志

## 2.3 NamespaceBranchAPI

`com.ctrip.framework.apollo.portal.api.NamespaceBranchAPI` ，实现 API 抽象类，封装对 Admin Service 的 Namespace **分支**模块的 API 调用。代码如下：

![NamespaceBranchAPI](http://www.iocoder.cn/images/Apollo/2018_05_01/05.png)

# 3. Admin Service 侧

## 3.1 NamespaceBranchController

在 `apollo-adminservice` 项目中， `com.ctrip.framework.apollo.adminservice.controller.NamespaceBranchController` ，提供 Namespace **分支**的 **API** 。

`#createBranch(...)` 方法，创建 Namespace **分支**。代码如下：

```Java
  1: @RestController
  2: public class NamespaceBranchController {
  3: 
  4:     @Autowired
  5:     private MessageSender messageSender;
  6:     @Autowired
  7:     private NamespaceBranchService namespaceBranchService;
  8:     @Autowired
  9:     private NamespaceService namespaceService;
 10: 
 11:     @RequestMapping(value = "/apps/{appId}/clusters/{clusterName}/namespaces/{namespaceName}/branches", method = RequestMethod.POST)
 12:     public NamespaceDTO createBranch(@PathVariable String appId,
 13:                                      @PathVariable String clusterName,
 14:                                      @PathVariable String namespaceName,
 15:                                      @RequestParam("operator") String operator) {
 16:         // 校验 Namespace 是否存在
 17:         checkNamespace(appId, clusterName, namespaceName);
 18:         // 创建子 Namespace
 19:         Namespace createdBranch = namespaceBranchService.createBranch(appId, clusterName, namespaceName, operator);
 20:         // 将 Namespace 转换成 NamespaceDTO 对象
 21:         return BeanUtils.transfrom(NamespaceDTO.class, createdBranch);
 22:     }
 23:     
 24:     // ... 省略其他接口和属性
 25: }
```

* 第 17 行：调用 `#checkNamespace(appId, clusterName, namespaceName)` ，**校验**父 Namespace 是否存在。代码如下：

    ```Java
    private void checkNamespace(String appId, String clusterName, String namespaceName) {
        // 查询父 Namespace 对象
        Namespace parentNamespace = namespaceService.findOne(appId, clusterName, namespaceName);
        // 若父 Namespace 不存在，抛出 BadRequestException 异常
        if (parentNamespace == null) {
            throw new BadRequestException(String.format("Namespace not exist. AppId = %s, ClusterName = %s, NamespaceName = %s",
                    appId, clusterName, namespaceName));
        }
    }
    ```

* 第 19 行：调用 `NamespaceBranchService#createBranch(appId, clusterName, namespaceName, operator)` 方法，创建 Namespace **分支**。
* 第 21 行：调用 `BeanUtils#transfrom(Class<T> clazz, Object src)` 方法，将 Namespace **转换**成 NamespaceDTO 对象。

## 3.2 NamespaceBranchService

在 `apollo-biz` 项目中，`com.ctrip.framework.apollo.biz.service.NamespaceBranchService` ，提供 Namespace **分支**的 **Service** 逻辑给 Admin Service 和 Config Service 。

`#createBranch(appId, clusterName, namespaceName, operator)` 方法，创建 Namespace **分支**。即，新增**子** Cluster 和**子** Namespace 。代码如下：

```Java
  1: @Autowired
  2: private ClusterService clusterService;
  3: @Autowired
  4: private NamespaceService namespaceService;
  5: 
  6: @Transactional
  7: public Namespace createBranch(String appId, String parentClusterName, String namespaceName, String operator) {
  8:     // 获得子 Namespace 对象
  9:     Namespace childNamespace = findBranch(appId, parentClusterName, namespaceName);
 10:     // 若存在子 Namespace 对象，则抛出 BadRequestException 异常。一个 Namespace 有且仅允许有一个子 Namespace 。
 11:     if (childNamespace != null) {
 12:         throw new BadRequestException("namespace already has branch");
 13:     }
 14:     // 获得父 Cluster 对象
 15:     Cluster parentCluster = clusterService.findOne(appId, parentClusterName);
 16:     // 若父 Cluster 对象不存在，抛出 BadRequestException 异常
 17:     if (parentCluster == null || parentCluster.getParentClusterId() != 0) {
 18:         throw new BadRequestException("cluster not exist or illegal cluster");
 19:     }
 20: 
 21:     // 创建子 Cluster 对象
 22:     // create child cluster
 23:     Cluster childCluster = createChildCluster(appId, parentCluster, namespaceName, operator);
 24:     // 保存子 Cluster 对象
 25:     Cluster createdChildCluster = clusterService.saveWithoutInstanceOfAppNamespaces(childCluster);
 26: 
 27:     // 创建子 Namespace 对象
 28:     // create child namespace
 29:     childNamespace = createNamespaceBranch(appId, createdChildCluster.getName(), namespaceName, operator);
 30:     // 保存子 Namespace 对象
 31:     return namespaceService.save(childNamespace);
 32: }
```

* 第 9 行：调用 `#findBranch(appId, parentClusterName, namespaceName)` 方法，获得**子** Namespace 对象。详细解析，见 [「3.2.1 findBranch」](#) 。
* 第 10 至 13 行：**校验**若存在**子** Namespace 对象，则抛出 BadRequestException 异常。**一个 Namespace 有且仅允许有一个子 Namespace** 。
* 第 15 行：调用 `ClusterService#findOne(appId, parentClusterName)` 方法，获得**父** Cluster 对象。
* 第 16 至 19 行：**校验**若父 Cluster 对象不存在，则抛出 BadRequestException 异常。
* ========== 子 Cluster ========== 
* 第 23 行：调用 `#createChildCluster(appId, parentCluster, namespaceName, operator)` 方法，创建**子** Cluster 对象。详细解析，见 [「3.2.2 createChildCluster」](#) 。
* 第 25 行：调用 `ClusterService#saveWithoutInstanceOfAppNamespaces(Cluster)` 方法，保存**子** Cluster 对象。
* ========== 子 Namespace ========== 
* 第 29 行：调用 `#createNamespaceBranch(appId, createdChildClusterName, namespaceName, operator)` 方法，创建**子** Namespace 对象。详细解析，见 [「3.2.3 createNamespaceBranch」](#) 。
* 第 31 行：调用 `NamespaceService#save(childNamespace)` 方法，保存**子** Namespace 对象。

### 3.2.1 findBranch

`#findBranch(appId, parentClusterName, namespaceName)` 方法，获得**子** Namespace 对象。代码如下：

```Java
public Namespace findBranch(String appId, String parentClusterName, String namespaceName) {
    return namespaceService.findChildNamespace(appId, parentClusterName, namespaceName);
}
```

-------

`NamespaceService#findChildNamespace(appId, parentClusterName, namespaceName)` 方法，获得**子** Namespace 对象。代码如下：

```Java
  1: /**
  2:  * 获得指定父 Namespace 的子 Namespace 对象
  3:  *
  4:  * @param appId App 编号
  5:  * @param parentClusterName 父 Cluster 的名字
  6:  * @param namespaceName 父 Namespace 的名字
  7:  * @return 子 Namespace 对象
  8:  */
  9: public Namespace findChildNamespace(String appId, String parentClusterName, String namespaceName) {
 10:     // 获得 Namespace 数组
 11:     List<Namespace> namespaces = findByAppIdAndNamespaceName(appId, namespaceName);
 12:     // 若只有一个 Namespace ，说明没有子 Namespace
 13:     if (CollectionUtils.isEmpty(namespaces) || namespaces.size() == 1) {
 14:         return null;
 15:     }
 16:     // 获得 Cluster 数组
 17:     List<Cluster> childClusters = clusterService.findChildClusters(appId, parentClusterName);
 18:     // 若无子 Cluster ，说明没有子 Namespace
 19:     if (CollectionUtils.isEmpty(childClusters)) {
 20:         return null;
 21:     }
 22:     // 创建子 Cluster 的名字的集合
 23:     Set<String> childClusterNames = childClusters.stream().map(Cluster::getName).collect(Collectors.toSet());
 24:     // 遍历 Namespace 数组，比较 Cluster 的名字。若符合，则返回该子 Namespace 对象。
 25:     // the child namespace is the intersection of the child clusters and child namespaces
 26:     for (Namespace namespace : namespaces) {
 27:         if (childClusterNames.contains(namespace.getClusterName())) {
 28:             return namespace;
 29:         }
 30:     }
 31:     // 无子 Namespace ，返回空。
 32:     return null;
 33: }
```

* 第 11 行：调用 `#findByAppIdAndNamespaceName(appId, namespaceName)` 方法，获得 **App 下所有的** Namespace 数组。代码如下：

    ```Java
    public List<Namespace> findByAppIdAndNamespaceName(String appId, String namespaceName) {
        return namespaceRepository.findByAppIdAndNamespaceName(appId, namespaceName);
    }
    ```

* 第12 至 15 行：若只有**一个** Namespace ，说明没有**子** Namespace 。
* 第 17 行：调用 `ClusterService#findChildClusters(appId, parentClusterName)` 方法，获得 Cluster 数组。代码如下：

    ```Java
    /**
     * 获得子 Cluster 数组
     *
     * @param appId App 编号
     * @param parentClusterName Cluster 名字
     * @return 子 Cluster 数组
     */
    public List<Cluster> findChildClusters(String appId, String parentClusterName) {
        // 获得父 Cluster 对象
        Cluster parentCluster = findOne(appId, parentClusterName);
        // 若不存在，抛出 BadRequestException 异常
        if (parentCluster == null) {
            throw new BadRequestException("parent cluster not exist");
        }
        // 获得子 Cluster 数组
        return clusterRepository.findByParentClusterId(parentCluster.getId());
    }
    ```

* 第 18 至 21 行：若无**子** Cluster ，说明没有**子** Namespace 。
* 第 23 行：创建**子** Cluster 的名字的集合。
* 第 24 至 30 行：遍历 Namespace 数组，若 Namespace 的 **Cluster 名字** 在 `childClusterNames` 中，返回该 Namespace 。因为【第 11 行】，获得 **App 下所有的** Namespace 数组。

### 3.2.2 createChildCluster

`#createChildCluster(...)` 方法，创建**子** Cluster 对象。代码如下：

```Java
private Cluster createChildCluster(String appId, Cluster parentCluster,
                                   String namespaceName, String operator) {
    Cluster childCluster = new Cluster();
    childCluster.setAppId(appId);
    childCluster.setParentClusterId(parentCluster.getId());
    childCluster.setName(UniqueKeyGenerator.generate(appId, parentCluster.getName(), namespaceName));
    childCluster.setDataChangeCreatedBy(operator);
    childCluster.setDataChangeLastModifiedBy(operator);
    return childCluster;
}
```

* `appId` 字段，指向和**父** Cluster 相同。
* `parentClusterId` 字段，指向**父** Cluster 编号。
* `name` 字段，调用 `UniqueKeyGenerator#generate(appId, parentClusterName, namespaceName)` 方法，创建唯一 KEY 。例如，`"20180422134118-dee27ba3456ff928"` 。

### 3.2.3 createNamespaceBranch

`#createNamespaceBranch(...)` 方法，创建**子** Namespace 对象。代码如下：

```Java
private Namespace createNamespaceBranch(String appId, String clusterName, String namespaceName, String operator) {
    Namespace childNamespace = new Namespace();
    childNamespace.setAppId(appId);
    childNamespace.setClusterName(clusterName);
    childNamespace.setNamespaceName(namespaceName);
    childNamespace.setDataChangeLastModifiedBy(operator);
    childNamespace.setDataChangeCreatedBy(operator);
    return childNamespace;
}
```

* `appId` 字段，指向和**父** Namespace 相同。
* `clusterName` 字段，指向和**子** Cluster 编号。
* `namespaceName` 字段，和 **父** Namespace 的名字相同。

# 666. 彩蛋

巧妙~~~

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)


