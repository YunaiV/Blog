title: Apollo 源码解析 —— Portal 关联 Namespace
date: 2018-03-15
tags:
categories: Apollo
permalink: Apollo/portal-associate-namespace
wechat_url:
toutiao_url: https://www.toutiao.com/i6634867649400537613/

---

摘要: 原创出处 http://www.iocoder.cn/Apollo/portal-associate-namespace/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Apollo/portal-associate-namespace/)
- [2. Portal 侧](http://www.iocoder.cn/Apollo/portal-associate-namespace/)
  - [2.1 NamespaceController](http://www.iocoder.cn/Apollo/portal-associate-namespace/)
  - [2.2 NamespaceService](http://www.iocoder.cn/Apollo/portal-associate-namespace/)
  - [2.3 NamespaceAPI](http://www.iocoder.cn/Apollo/portal-associate-namespace/)
- [3. Admin Service 侧](http://www.iocoder.cn/Apollo/portal-associate-namespace/)
  - [3.1 NamespaceController](http://www.iocoder.cn/Apollo/portal-associate-namespace/)
  - [3.2 NamespaceService](http://www.iocoder.cn/Apollo/portal-associate-namespace/)
  - [3.3 NamespaceRepository](http://www.iocoder.cn/Apollo/portal-associate-namespace/)
- [666. 彩蛋](http://www.iocoder.cn/Apollo/portal-associate-namespace/)

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

> 老艿艿：本系列假定胖友已经阅读过 [《Apollo 官方 wiki 文档》](https://github.com/ctripcorp/apollo/wiki/)  ，特别是 [《Apollo 官方 wiki 文档 —— 核心概念之“Namespace”》](https://github.com/ctripcorp/apollo/wiki/Apollo%E6%A0%B8%E5%BF%83%E6%A6%82%E5%BF%B5%E4%B9%8B%E2%80%9CNamespace%E2%80%9D) 。

本文分享 **Portal 关联 Namespace** 的流程，整个过程涉及 Portal、Admin Service ，如下图所示：

![流程](http://www.iocoder.cn/images/Apollo/2018_03_15/01.png)

> 老艿艿：因为 Portal 是管理后台，所以从代码实现上，和业务系统非常相像。也因此，本文会略显啰嗦。

# 2. Portal 侧

## 2.1 NamespaceController

在 `apollo-portal` 项目中，`com.ctrip.framework.apollo.portal.controller.NamespaceController` ，提供 AppNamespace 和 Namespace  的 **API** 。

在**关联 Namespace**的界面中，点击【提交】按钮，调用**创建 Namespace 的 API** 。

![关联 Namespace](http://www.iocoder.cn/images/Apollo/2018_03_15/02.png)

* 公用类型的 Namespace 的**名字**是**全局**唯一，所以关联时，只需要查看名字即可。

`#createNamespace(appId, List<NamespaceCreationModel>)` 方法，创建 Namespace 对象，支持**多个** Namespace 。代码如下：

```Java
  1: @Autowired
  2: private UserInfoHolder userInfoHolder;
  3: @Autowired
  4: private NamespaceService namespaceService;
  5: @Autowired
  6: private RoleInitializationService roleInitializationService;
  7: 
  8: @PreAuthorize(value = "@permissionValidator.hasCreateNamespacePermission(#appId)")
  9: @RequestMapping(value = "/apps/{appId}/namespaces", method = RequestMethod.POST)
 10: public ResponseEntity<Void> createNamespace(@PathVariable String appId,
 11:                                             @RequestBody List<NamespaceCreationModel> models) {
 12:     // 校验 `models` 非空
 13:     checkModel(!CollectionUtils.isEmpty(models));
 14:     // 初始化 Namespace 的 Role 们
 15:     String namespaceName = models.get(0).getNamespace().getNamespaceName();
 16:     String operator = userInfoHolder.getUser().getUserId();
 17:     roleInitializationService.initNamespaceRoles(appId, namespaceName, operator);
 18:     // 循环 `models` ，创建 Namespace 对象
 19:     for (NamespaceCreationModel model : models) {
 20:         NamespaceDTO namespace = model.getNamespace();
 21:         // 校验相关参数非空
 22:         RequestPrecondition.checkArgumentsNotEmpty(model.getEnv(), namespace.getAppId(), namespace.getClusterName(), namespace.getNamespaceName());
 23:         // 创建 Namespace 对象
 24:         try {
 25:             namespaceService.createNamespace(Env.valueOf(model.getEnv()), namespace);
 26:         } catch (Exception e) {
 27:             logger.error("create namespace fail.", e);
 28:             Tracer.logError(String.format("create namespace fail. (env=%s namespace=%s)", model.getEnv(), namespace.getNamespaceName()), e);
 29:         }
 30:     }
 31:     // 授予 Namespace Role 给当前管理员
 32:     assignNamespaceRoleToOperator(appId, namespaceName);
 33:     return ResponseEntity.ok().build();
 34: }
```

* **POST `/apps/{appId}/namespaces` 接口**，Request Body 传递 **JSON** 对象。
* `@PreAuthorize(...)` 注解，调用 `PermissionValidator#hasCreateNamespacePermission(appId)` 方法，校验是否有创建 Namespace 的权限。后续文章，详细分享。
* `com.ctrip.framework.apollo.portal.entity.model.NamespaceCreationModel` ，Namespace 创建 Model 。代码如下：

    ```Java
    public class NamespaceCreationModel {
    
        /**
         * 环境
         */
        private String env;
        /**
         * Namespace 信息
         */
        private NamespaceDTO namespace;
    }
    ```
    * `com.ctrip.framework.apollo.common.dto.NamespaceDTO` ，Namespace DTO 。代码如下：

        ```Java
        public class NamespaceDTO extends BaseDTO {
        
            private long id;
            /**
             * App 编号
             */
            private String appId;
            /**
             * Cluster 名字
             */
            private String clusterName;
            /**
             * Namespace 名字
             */
            private String namespaceName;
        }
        ```
        * x

* 第 13 行：校验 `models` 非空。
* 第 14 至 17 行：初始化 Namespace 的 Role 们。详解解析，见 [《Apollo 源码解析 —— Portal 认证与授权（二）之授权》](http://www.iocoder.cn/Apollo/portal-auth-2?self) 。
* 第 18 至 30 行：循环 `models` ，创建 Namespace 对象们。
    * 第 22 行：调用 `RequestPrecondition#checkArgumentsNotEmpty(String... args)` 方法，校验 NamespaceDTO 的 `env` `appId` `clusterName` `namespaceName` 非空。
    * 第 25 行：调用 `NamespaceService#createNamespace(Env, NamespaceDTO)` 方法，创建并保存 Namespace 到 Admin Service 中。
    * 第 26 至 29 行：当发生异常时，即创建**失败**，仅打印异常日志。也就是说，在 【第 33 行】，依然提示创建 Namespace 成功。
* 第 32 行：授予 Namespace Role 给当前管理员。详解解析，见 [《Apollo 源码解析 —— Portal 认证与授权（二）之授权》](http://www.iocoder.cn/Apollo/portal-auth-2?self) 。

## 2.2 NamespaceService

在 `apollo-portal` 项目中，`com.ctrip.framework.apollo.portal.service.NamespaceService` ，提供 Namespace 的 **Service** 逻辑。

`#createNamespace(Env env, NamespaceDTO namespace)` 方法，保存 Namespace 对象到 Admin Service 中。代码如下：

```Java
  1: @Autowired
  2: private UserInfoHolder userInfoHolder;
  3: @Autowired
  4: private AdminServiceAPI.NamespaceAPI namespaceAPI;
  5: 
  6: public NamespaceDTO createNamespace(Env env, NamespaceDTO namespace) {
  7:     // 设置 NamespaceDTO 的创建和修改人为当前管理员
  8:     if (StringUtils.isEmpty(namespace.getDataChangeCreatedBy())) {
  9:         namespace.setDataChangeCreatedBy(userInfoHolder.getUser().getUserId());
 10:     }
 11:     namespace.setDataChangeLastModifiedBy(userInfoHolder.getUser().getUserId());
 12:     // 创建 Namespace 到 Admin Service
 13:     NamespaceDTO createdNamespace = namespaceAPI.createNamespace(env, namespace);
 14:     // 【TODO 6001】Tracer 日志
 15:     Tracer.logEvent(TracerEventType.CREATE_NAMESPACE, String.format("%s+%s+%s+%s", namespace.getAppId(), env, namespace.getClusterName(), namespace.getNamespaceName()));
 16:     return createdNamespace;
 17: }
```

* 第 7 至 11 行：**设置** NamespaceDTO 的创建和修改人。
* 第 13 行：调用 `NamespaceAPI#createNamespace(Env, NamespaceDTO)` 方法，创建 Namespace 到 Admin Service 。
* 第 15 行：【TODO 6001】Tracer 日志

## 2.3 NamespaceAPI

`com.ctrip.framework.apollo.portal.api.NamespaceAPI` ，实现 API 抽象类，封装对 Admin Service 的 AppNamespace 和 Namespace **两个**模块的 API 调用。代码如下：

![NamespaceAPI](http://www.iocoder.cn/images/Apollo/2018_03_15/03.png)

* 使用 `restTemplate` ，调用对应的 API 接口。

# 3. Admin Service 侧

## 3.1 NamespaceController

在 `apollo-adminservice` 项目中， `com.ctrip.framework.apollo.adminservice.controller.NamespaceController` ，提供 Namespace 的 **API** 。

`#create(appId, clusterName, NamespaceDTO)` 方法，创建 Namespace 。代码如下：

```Java
  1: @RestController
  2: public class NamespaceController {
  3: 
  4:     @Autowired
  5:     private NamespaceService namespaceService;
  6: 
  7:     /**
  8:      * 创建 Namespace
  9:      *
 10:      * @param appId App 编号
 11:      * @param clusterName Cluster 名字
 12:      * @param dto NamespaceDTO 对象
 13:      * @return 创建成功的 NamespaceDTO 对象
 14:      */
 15:     @RequestMapping(path = "/apps/{appId}/clusters/{clusterName}/namespaces", method = RequestMethod.POST)
 16:     public NamespaceDTO create(@PathVariable("appId") String appId,
 17:                                @PathVariable("clusterName") String clusterName, @RequestBody NamespaceDTO dto) {
 18:         // 校验 NamespaceDTO 的 `namespaceName` 格式正确。
 19:         if (!InputValidator.isValidClusterNamespace(dto.getNamespaceName())) {
 20:             throw new BadRequestException(String.format("Namespace格式错误: %s", InputValidator.INVALID_CLUSTER_NAMESPACE_MESSAGE));
 21:         }
 22:         // 将 NamespaceDTO 转换成 Namespace 对象
 23:         Namespace entity = BeanUtils.transfrom(Namespace.class, dto);
 24:         // 判断 `name` 在 Cluster 下是否已经存在对应的 Namespace 对象。若已经存在，抛出 BadRequestException 异常。
 25:         Namespace managedEntity = namespaceService.findOne(appId, clusterName, entity.getNamespaceName());
 26:         if (managedEntity != null) {
 27:             throw new BadRequestException("namespace already exist.");
 28:         }
 29:         // 保存 Namespace 对象
 30:         entity = namespaceService.save(entity);
 31:         // 将保存的 Namespace 对象转换成 NamespaceDTO
 32:         dto = BeanUtils.transfrom(NamespaceDTO.class, entity);
 33:         return dto;
 34:     }
 35:     
 36:     // ... 省略其他接口和属性
 37: }
```

* **POST `/apps/{appId}/clusters/{clusterName}/namespaces` 接口**，Request Body 传递 **JSON** 对象。
* 第 18 至 21 行：调用 `InputValidator#isValidClusterNamespace(name)` 方法，**校验** NamespaceDTO 的 `namespaceName` 格式正确，符合 `[0-9a-zA-Z_.-]+"` 格式。
* 第 23 行：调用 `BeanUtils#transfrom(Class<T> clazz, Object src)` 方法，将 NamespaceDTO **转换**成 Namespace 对象。
* 第 20 至 23 行：调用 `NamespaceService#findOne(appId, clusterName, namespaceName)` 方法，**校验** `name` 在 Cluster 下是否已经存在对应的 Namespace 对象。若已经存在，抛出 BadRequestException 异常。
* 第 30 行：调用 `NamespaceService#save(Namespace)` 方法，保存 Namespace 对象到数据库。
* 第 30 至 32 行：调用 `BeanUtils#transfrom(Class<T> clazz, Object src)` 方法，将保存的 Namespace 对象，转换成 NamespaceDTO 返回。

## 3.2 NamespaceService

在 [《Apollo 源码解析 —— Portal 创建 Namespace》](http://www.iocoder.cn/Apollo/portal-create-namespace/?self) 的 [「4.4 NamespaceService」](#) ，已经详细解析。

## 3.3 NamespaceRepository

在 [《Apollo 源码解析 —— Portal 创建 Namespace》](http://www.iocoder.cn/Apollo/portal-create-namespace/?self) 的 [「4.5 NamespaceRepository」](#) ，已经详细解析。

# 666. 彩蛋

🙂 有点水更，写的自己都不好意思了。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

有一点需要补充，公用类型和关联类型的判定，差异点**仅仅**是 Namespace 和 其对应的 AppNamespace 的 `appId` 是否一致。

* 若**一致**，就是公用类型。
* 若**不同**，就是关联类型。

着意味着什么？如果我们在一个应用下给的 Cluster A 和 B 创建了一个名为 ns 的**公用类型**的 AppNamespace ，那么在 Cluster A 和 B 都会生成对应的 Namespace 。  
如果此处删除了 B 的 Namespace ，则在 B 下面就不存在该 Namespace 。  
如果我们再通过**关联**的方式，添加了 ns ，生成的 Namespace 是 **公用类型**，而不是**关联类型**。

一定要注意！！！

