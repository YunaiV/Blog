title: Apollo 源码解析 —— Portal 创建 Cluster
date: 2018-03-07
tags:
categories: Apollo
permalink: Apollo/portal-create-cluster
wechat_url:
toutiao_url: https://www.toutiao.com/i6634864500442923527/

---

摘要: 原创出处 http://www.iocoder.cn/Apollo/portal-create-cluster/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Apollo/portal-create-cluster/)
- [2. Cluster](http://www.iocoder.cn/Apollo/portal-create-cluster/)
- [3. Portal 侧](http://www.iocoder.cn/Apollo/portal-create-cluster/)
  - [3.1 ClusterController](http://www.iocoder.cn/Apollo/portal-create-cluster/)
  - [3.2 ClusterService](http://www.iocoder.cn/Apollo/portal-create-cluster/)
  - [3.3 ClusterAPI](http://www.iocoder.cn/Apollo/portal-create-cluster/)
- [4. Admin Service 侧](http://www.iocoder.cn/Apollo/portal-create-cluster/)
  - [4.1 ClusterController](http://www.iocoder.cn/Apollo/portal-create-cluster/)
  - [4.2 ClusterService](http://www.iocoder.cn/Apollo/portal-create-cluster/)
  - [4.3 ClusterRepository](http://www.iocoder.cn/Apollo/portal-create-cluster/)
- [666. 彩蛋](http://www.iocoder.cn/Apollo/portal-create-cluster/)

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

本文分享 **Portal 创建 Cluster** 的流程，整个过程涉及 Portal、Admin Service ，如下图所示：

![流程](http://www.iocoder.cn/images/Apollo/2018_03_07/01.png)

下面，我们先来看看 Cluster 的实体结构

> 老艿艿：因为 Portal 是管理后台，所以从代码实现上，和业务系统非常相像。也因此，本文会略显啰嗦。

# 2. Cluster

`com.ctrip.framework.apollo.biz.entity.Cluster` ，继承 BaseEntity 抽象类，Cluster **实体**。代码如下：

```Java
@Entity
@Table(name = "Cluster")
@SQLDelete(sql = "Update Cluster set isDeleted = 1 where id = ?")
@Where(clause = "isDeleted = 0")
public class Cluster extends BaseEntity implements Comparable<Cluster> {

    /**
     * 名字
     */
    @Column(name = "Name", nullable = false)
    private String name;
    /**
     * App 编号 {@link }
     */
    @Column(name = "AppId", nullable = false)
    private String appId;
    /**
     * 父 App 编号
     */
    @Column(name = "ParentClusterId", nullable = false)
    private long parentClusterId;
}
```

* `appId` 字段，App 编号，指向对应的 App 。App : Cluster = 1 : N 。
* `parentClusterId` 字段，父 App 编号。用于灰度发布，在 [《Apollo 源码解析 —— Portal 创建灰度》](http://www.iocoder.cn/Apollo/portal-create-namespace-branch/?self) 有详细解析。

# 3. Portal 侧

## 3.1 ClusterController

在 `apollo-portal` 项目中，`com.ctrip.framework.apollo.portal.controller.ClusterController` ，提供 Cluster 的 **API** 。

在**创建 Cluster**的界面中，点击【提交】按钮，调用**创建 Cluster 的 API** 。

![创建 Cluster](http://www.iocoder.cn/images/Apollo/2018_03_07/02.png)

代码如下：

```Java
  1: @RestController
  2: public class ClusterController {
  3: 
  4:     @Autowired
  5:     private ClusterService clusterService;
  6:     @Autowired
  7:     private UserInfoHolder userInfoHolder;
  8: 
  9:     @PreAuthorize(value = "@permissionValidator.hasCreateClusterPermission(#appId)")
 10:     @RequestMapping(value = "apps/{appId}/envs/{env}/clusters", method = RequestMethod.POST)
 11:     public ClusterDTO createCluster(@PathVariable String appId, @PathVariable String env,
 12:                                     @RequestBody ClusterDTO cluster) {
 13:         // 校验 ClusterDTO 非空
 14:         checkModel(Objects.nonNull(cluster));
 15:         // 校验 ClusterDTO 的 `appId` 和 `name` 非空。
 16:         RequestPrecondition.checkArgumentsNotEmpty(cluster.getAppId(), cluster.getName());
 17:         // 校验 ClusterDTO 的 `name` 格式正确。
 18:         if (!InputValidator.isValidClusterNamespace(cluster.getName())) {
 19:             throw new BadRequestException(String.format("Cluster格式错误: %s", InputValidator.INVALID_CLUSTER_NAMESPACE_MESSAGE));
 20:         }
 21:         // 设置 ClusterDTO 的创建和修改人为当前管理员
 22:         String operator = userInfoHolder.getUser().getUserId();
 23:         cluster.setDataChangeLastModifiedBy(operator);
 24:         cluster.setDataChangeCreatedBy(operator);
 25:         // 创建 Cluster 到 Admin Service
 26:         return clusterService.createCluster(Env.valueOf(env), cluster);
 27:     }
 28:     
 29:     // ... 省略 deleteCluster 接口
 30: }
```

* **POST `apps/{appId}/envs/{env}/cluster` 接口**，Request Body 传递 **JSON** 对象。
* `@PreAuthorize(...)` 注解，调用 `PermissionValidator#hasCreateClusterPermission(appId,)` 方法，校验是否有创建 Cluster 的权限。后续文章，详细分享。
* 第 14 行：校验 ClusterDTO 非空。**注意**，此处使用的接收请求参数是 Cluster**DTO** 。
* 第 16 行：调用 `RequestPrecondition#checkArgumentsNotEmpty(String... args)` 方法，校验 ClusterDTO 的 `appId` 和 `name` 非空。
* 第 16 至 21 行：调用 `InputValidator#isValidClusterNamespace(name)` 方法，校验 ClusterDTO 的 `name` 格式正确，符合 `[0-9a-zA-Z_.-]+"` 格式。
* 第 21 至 24 行：设置 ClusterDTO 的创建和修改人为当前管理员。
* 第 26 行：调用 `ClusterService#createCluster(Env, ClusterDTO)` 方法，创建并保存 Cluster 到 Admin Service 。在 [「3.2 ClusterService」](#) 中，详细解析。

## 3.2 ClusterService

在 `apollo-portal` 项目中，`com.ctrip.framework.apollo.portal.service.ClusterService` ，提供 Cluster 的 **Service** 逻辑。

`#createCluster(Env, ClusterDTO)` 方法，创建并保存 Cluster 到 Admin Service 。代码如下：

```Java
  1: @Autowired
  2: private AdminServiceAPI.ClusterAPI clusterAPI;
  3: 
  4: public ClusterDTO createCluster(Env env, ClusterDTO cluster) {
  5:     // 根据 `appId` 和 `name` 校验 Cluster 的唯一性
  6:     if (!clusterAPI.isClusterUnique(cluster.getAppId(), env, cluster.getName())) {
  7:         throw new BadRequestException(String.format("cluster %s already exists.", cluster.getName()));
  8:     }
  9:     // 创建 Cluster 到 Admin Service
 10:     ClusterDTO clusterDTO = clusterAPI.create(env, cluster);
 11:     // 【TODO 6001】Tracer 日志
 12:     Tracer.logEvent(TracerEventType.CREATE_CLUSTER, cluster.getAppId(), "0", cluster.getName());
 13:     return clusterDTO;
 14: }
```

* 第 5 至 8 行：调用 `ClusterAPI#isClusterUnique(appId, Env, clusterName)` 方法，根据 `appId` 和 `name` **校验** Cluster 的唯一性。**注意**，此处是远程调用 Admin Service 的 API 。
* 第 10 行：调用 `ClusterAPI#create(Env, ClusterDTO)` 方法，创建 Cluster 到 Admin Service 。
* 第 12 行：【TODO 6001】Tracer 日志

## 3.3 ClusterAPI

`com.ctrip.framework.apollo.portal.api.ClusterAPI` ，实现 API 抽象类，封装对 Admin Service 的 Cluster 模块的 API 调用。代码如下：

![ClusterAPI](http://www.iocoder.cn/images/Apollo/2018_03_07/03.png)

# 4. Admin Service 侧

## 4.1 ClusterController

在 `apollo-adminservice` 项目中， `com.ctrip.framework.apollo.adminservice.controller.ClusterController` ，提供 Cluster 的 **API** 。

`#create(appId, autoCreatePrivateNamespace, ClusterDTO)` 方法，创建 Cluster 。代码如下：

```Java
     @RestController
  1: public class ClusterController {
  2: 
  3:     @Autowired
  4:     private ClusterService clusterService;
  5: 
  6:     @RequestMapping(path = "/apps/{appId}/clusters", method = RequestMethod.POST)
  7:     public ClusterDTO create(@PathVariable("appId") String appId,
  8:                              @RequestParam(value = "autoCreatePrivateNamespace", defaultValue = "true") boolean autoCreatePrivateNamespace,
  9:                              @RequestBody ClusterDTO dto) {
 10:         // 校验 ClusterDTO 的 `name` 格式正确。
 11:         if (!InputValidator.isValidClusterNamespace(dto.getName())) {
 12:             throw new BadRequestException(String.format("Cluster格式错误: %s", InputValidator.INVALID_CLUSTER_NAMESPACE_MESSAGE));
 13:         }
 14:         // 将 ClusterDTO 转换成 Cluster 对象
 15:         Cluster entity = BeanUtils.transfrom(Cluster.class, dto);
 16:         // 判断 `name` 在 App 下是否已经存在对应的 Cluster 对象。若已经存在，抛出 BadRequestException 异常。
 17:         Cluster managedEntity = clusterService.findOne(appId, entity.getName());
 18:         if (managedEntity != null) {
 19:             throw new BadRequestException("cluster already exist.");
 20:         }
 21:         // 保存 Cluster 对象，并创建其 Namespace
 22:         if (autoCreatePrivateNamespace) {
 23:             entity = clusterService.saveWithInstanceOfAppNamespaces(entity);
 24:         // 保存 Cluster 对象，不创建其 Namespace
 25:         } else {
 26:             entity = clusterService.saveWithoutInstanceOfAppNamespaces(entity);
 27:         }
 28:         // 将保存的 Cluster 对象转换成 ClusterDTO
 29:         dto = BeanUtils.transfrom(ClusterDTO.class, entity);
 30:         return dto;
 31:     }
 32:      
 33:      // ... 省略其他接口和属性
 34: }
```

* **POST `/apps/{appId}/clusters` 接口**，Request Body 传递 **JSON** 对象。
* 第 10 至 13 行：调用 `InputValidator#isValidClusterNamespace(name)` 方法，**校验** ClusterDTO 的 `name` 格式正确，符合 `[0-9a-zA-Z_.-]+"` 格式。
* 第 15 行：调用 `BeanUtils#transfrom(Class<T> clazz, Object src)` 方法，将 ClusterDTO **转换**成 Cluster 对象。
* 第 16 至 20 行：调用 `ClusterService#findOne(appId, name)` 方法，**校验** `name`  在 App 下，是否已经存在对应的 Cluster 对象。若已经存在，抛出 BadRequestException 异常。
* 第 21 至 23 行：若 `autoCreatePrivateNamespace = true` 时，调用 `ClusterService#saveWithInstanceOfAppNamespaces(Cluster)` 方法，保存 Cluster 对象，**并**创建其 Namespace 。
* 第 24 至 27 行：若 `autoCreatePrivateNamespace = false` 时，调用 `ClusterService#saveWithoutInstanceOfAppNamespaces(Cluster)` 方法，保存 Cluster 对象，**不**创建其 Namespace 。
* 第 29 行：调用 `BeanUtils#transfrom(Class<T> clazz, Object src)` 方法，将 Cluster **转换**成 ClusterDTO 对象。

## 4.2 ClusterService

在 `apollo-biz` 项目中，`com.ctrip.framework.apollo.biz.service.ClusterService` ，提供 Cluster 的 **Service** 逻辑给 Admin Service 和 Config Service 。

`#saveWithInstanceOfAppNamespaces(Cluster)` 方法，保存 Cluster 对象，并创建其 Namespace 。代码如下：

```Java
  1: @Autowired
  2: private ClusterRepository clusterRepository;
  3: @Autowired
  4: private AuditService auditService;
  5: @Autowired
  6: private NamespaceService namespaceService;
  7: 
  8: @Transactional
  9: public Cluster saveWithInstanceOfAppNamespaces(Cluster entity) {
 10:     // 保存 Cluster 对象
 11:     Cluster savedCluster = saveWithoutInstanceOfAppNamespaces(entity);
 12:     // 创建 Cluster 的 Namespace 们
 13:     namespaceService.instanceOfAppNamespaces(savedCluster.getAppId(), savedCluster.getName(), savedCluster.getDataChangeCreatedBy());
 14:     return savedCluster;
 15: }
```

* 第 11 行：调用 `#saveWithoutInstanceOfAppNamespaces(Cluster)` 方法，保存 Cluster 对象。
* 第 13 行：调用 `NamespaceService#instanceOfAppNamespaces(appId, clusterName, createBy)` 方法，创建 Cluster 的 Namespace 们。在 [《Apollo 源码解析 —— Portal 创建 Namespace》](http://www.iocoder.cn/Apollo/portal-create-namespace/?self) 中，有详细解析。

-------

`#saveWithoutInstanceOfAppNamespaces(Cluster)` 方法，保存 Cluster 对象。代码如下：

```Java
@Transactional
public Cluster saveWithoutInstanceOfAppNamespaces(Cluster entity) {
    // 判断 `name` 在 App 下是否已经存在对应的 Cluster 对象。若已经存在，抛出 BadRequestException 异常。
    if (!isClusterNameUnique(entity.getAppId(), entity.getName())) {
        throw new BadRequestException("cluster not unique");
    }
    // 保存 Cluster 对象到数据库
    entity.setId(0);//protection
    Cluster cluster = clusterRepository.save(entity);
    // 【TODO 6001】Tracer 日志
    auditService.audit(Cluster.class.getSimpleName(), cluster.getId(), Audit.OP.INSERT, cluster.getDataChangeCreatedBy());
    return cluster;
}
```

## 4.3 ClusterRepository

`com.ctrip.framework.apollo.biz.repository.ClusterRepository` ，继承 `org.springframework.data.repository.PagingAndSortingRepository` 接口，提供 Cluster 的**数据访问** 给 Admin Service 和 Config Service 。代码如下：

```Java
public interface ClusterRepository extends PagingAndSortingRepository<Cluster, Long> {

  List<Cluster> findByAppIdAndParentClusterId(String appId, Long parentClusterId);

  List<Cluster> findByAppId(String appId);

  Cluster findByAppIdAndName(String appId, String name);

  List<Cluster> findByParentClusterId(Long parentClusterId);

}
```

# 666. 彩蛋

🙂 有点水更，写的自己都不好意思了。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

