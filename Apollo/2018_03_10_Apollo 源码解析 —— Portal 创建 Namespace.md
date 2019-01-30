title: Apollo 源码解析 —— Portal 创建 Namespace
date: 2018-03-10
tags:
categories: Apollo
permalink: Apollo/portal-create-namespace
wechat_url:
toutiao_url: https://www.toutiao.com/i6634866252084937229/

---

摘要: 原创出处 http://www.iocoder.cn/Apollo/portal-create-namespace/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Apollo/portal-create-namespace/)
- [2. 实体](http://www.iocoder.cn/Apollo/portal-create-namespace/)
  - [2.1 AppNamespace](http://www.iocoder.cn/Apollo/portal-create-namespace/)
  - [2.2 Namespace](http://www.iocoder.cn/Apollo/portal-create-namespace/)
  - [2.3 AppNamespace vs. Namespace](http://www.iocoder.cn/Apollo/portal-create-namespace/)
  - [2.4 类型](http://www.iocoder.cn/Apollo/portal-create-namespace/)
- [3. Portal 侧](http://www.iocoder.cn/Apollo/portal-create-namespace/)
  - [3.1 NamespaceController](http://www.iocoder.cn/Apollo/portal-create-namespace/)
  - [3.2 AppNamespaceService](http://www.iocoder.cn/Apollo/portal-create-namespace/)
  - [3.3 AppNamespaceRepository](http://www.iocoder.cn/Apollo/portal-create-namespace/)
  - [3.4 AppNamespaceCreationEvent](http://www.iocoder.cn/Apollo/portal-create-namespace/)
  - [3.5 NamespaceAPI](http://www.iocoder.cn/Apollo/portal-create-namespace/)
- [4. Admin Service 侧](http://www.iocoder.cn/Apollo/portal-create-namespace/)
  - [4.1 AppNamespaceController](http://www.iocoder.cn/Apollo/portal-create-namespace/)
  - [4.2 AppNamespaceService](http://www.iocoder.cn/Apollo/portal-create-namespace/)
  - [4.3 AppNamespaceRepository](http://www.iocoder.cn/Apollo/portal-create-namespace/)
  - [4.4 NamespaceService](http://www.iocoder.cn/Apollo/portal-create-namespace/)
  - [4.5 NamespaceRepository](http://www.iocoder.cn/Apollo/portal-create-namespace/)
- [666. 彩蛋](http://www.iocoder.cn/Apollo/portal-create-namespace/)

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

本文分享 **Portal 创建 Namespace** 的流程，整个过程涉及 Portal、Admin Service ，如下图所示：

![流程](http://www.iocoder.cn/images/Apollo/2018_03_10/01.png)

下面，我们先来看看 AppNamespace 和 Namespace 的实体结构

> 老艿艿：因为 Portal 是管理后台，所以从代码实现上，和业务系统非常相像。也因此，本文会略显啰嗦。

# 2. 实体

## 2.1 AppNamespace

在 `apollo-common` 项目中，`com.ctrip.framework.apollo.common.entity.AppNamespace` ，继承 BaseEntity 抽象类，App Namespace **实体**。代码如下：

```Java
@Entity
@Table(name = "AppNamespace")
@SQLDelete(sql = "Update AppNamespace set isDeleted = 1 where id = ?")
@Where(clause = "isDeleted = 0")
public class AppNamespace extends BaseEntity {

    /**
     * AppNamespace 名
     */
    @Column(name = "Name", nullable = false)
    private String name;
    /**
     * App 编号
     */
    @Column(name = "AppId", nullable = false)
    private String appId;
    /**
     * 格式
     *
     * 参见 {@link ConfigFileFormat}
     */
    @Column(name = "Format", nullable = false)
    private String format;
    /**
     * 是否公用的
     */
    @Column(name = "IsPublic", columnDefinition = "Bit default '0'")
    private boolean isPublic = false;
    /**
     * 备注
     */
    @Column(name = "Comment")
    private String comment;
}
```
 
* `appId` 字段，App 编号，指向对应的 App 。App : AppNamespace = 1 : N 。
* `format` 字段，格式。在 `com.ctrip.framework.apollo.core.enums.ConfigFileFormat` **枚举类**中，定义了五种类型。代码如下：

    ```Java
    public enum ConfigFileFormat {
    
        Properties("properties"), XML("xml"), JSON("json"), YML("yml"), YAML("yaml");
    
        private String value;
    
        // ... 省略了无关的代码
    }
    ```

* `isPublic` 字段，是否公用的。

    > Namespace的获取权限分为两种：
    > 
    > * **private** （私有的）：private 权限的 Namespace ，只能被所属的应用获取到。一个应用尝试获取其它应用 private 的 Namespace ，Apollo 会报 “404” 异常。
    > * **public** （公共的）：public 权限的 Namespace ，能被任何应用获取。
    > 
    > *这里的获取权限是相对于 Apollo 客户端来说的。*

## 2.2 Namespace

在 `apollo-biz` 项目中， `com.ctrip.framework.apollo.biz.entity.Namespace` ，继承 BaseEntity 抽象类，Cluster Namespace **实体**，是配置项的**集合**，类似于一个配置文件的概念。代码如下：

```Java
@Entity
@Table(name = "Namespace")
@SQLDelete(sql = "Update Namespace set isDeleted = 1 where id = ?")
@Where(clause = "isDeleted = 0")
public class Namespace extends BaseEntity {

    /**
     * App 编号 {@link com.ctrip.framework.apollo.common.entity.App#appId}
     */
    @Column(name = "appId", nullable = false)
    private String appId;
    /**
     * Cluster 名 {@link Cluster#name}
     */
    @Column(name = "ClusterName", nullable = false)
    private String clusterName;
    /**
     * AppNamespace 名 {@link com.ctrip.framework.apollo.common.entity.AppNamespace#name}
     */
    @Column(name = "NamespaceName", nullable = false)
    private String namespaceName;
    
}    
```

## 2.3 AppNamespace vs. Namespace

**关系图**如下：

![ER 图](http://www.iocoder.cn/images/Apollo/2018_03_10/06.png)

**数据流**向如下：

1. 在 App 下创建 **App**Namespace 后，自动给 App 下每个 Cluster 创建 Namespace 。
2. 在 App 下创建 Cluster 后，根据 App 下 每个 **App**Namespace 创建 Namespace 。
3. 可删除 Cluster 下的 Namespace 。

**总结**来说：

1. **App**Namespace 是 App 下的每个 Cluster **默认**创建的 Namespace 。
2.  Namespace 是 每个 Cluster **实际**拥有的 Namespace 。

## 2.4 类型

> Namespace 类型有三种：
> 
> * 私有类型：私有类型的 Namespace 具有 **private** 权限。
> * 公共类型：公共类型的 Namespace 具有 **public** 权限。公共类型的 Namespace 相当于游离于应用之外的配置，且通过 Namespace 的名称去标识公共 Namespace ，所以公共的 Namespace 的名称必须**全局唯一**。
> * 关联类型：关联类型又可称为继承类型，关联类型具有 **private** 权限。关联类型的Namespace 继承于公共类型的Namespace，用于覆盖公共 Namespace 的某些配置。

在 Namespace 实体中，**找不到** 类型的字段呀？！通过如下逻辑判断：

```Java
Namespace => AppNamespace
if (AppNamespace.isPublic) {
    return "公共类型";
}
if (Namespace.appId == AppNamespace.appId) {
    return "私有类型";
}
return "关联类型";
```

# 3. Portal 侧

## 3.1 NamespaceController

在 `apollo-portal` 项目中，`com.ctrip.framework.apollo.portal.controller.NamespaceController` ，提供 AppNamespace 和 Namespace  的 **API** 。

在**创建 Namespace**的界面中，点击【提交】按钮，调用**创建 AppNamespace 的 API** 。

![创建 Namespace](http://www.iocoder.cn/images/Apollo/2018_03_10/02.png)

代码如下：

```Java
  1: @RestController
  2: public class NamespaceController {
  3: 
  4:     @Autowired
  5:     private ApplicationEventPublisher publisher;
  6:     @Autowired
  7:     private AppNamespaceService appNamespaceService;
  8:     @Autowired
  9:     private PortalConfig portalConfig;
 10: 
 11:     @PreAuthorize(value = "@permissionValidator.hasCreateAppNamespacePermission(#appId, #appNamespace)")
 12:     @RequestMapping(value = "/apps/{appId}/appnamespaces", method = RequestMethod.POST)
 13:     public AppNamespace createAppNamespace(@PathVariable String appId, @RequestBody AppNamespace appNamespace) {
 14:         // 校验 AppNamespace 的 `appId` 和 `name` 非空。
 15:         RequestPrecondition.checkArgumentsNotEmpty(appNamespace.getAppId(), appNamespace.getName());
 16:         // 校验 AppNamespace 的 `name` 格式正确。
 17:         if (!InputValidator.isValidAppNamespace(appNamespace.getName())) {
 18:             throw new BadRequestException(String.format("Namespace格式错误: %s",
 19:                     InputValidator.INVALID_CLUSTER_NAMESPACE_MESSAGE + " & "
 20:                             + InputValidator.INVALID_NAMESPACE_NAMESPACE_MESSAGE));
 21:         }
 22:         // 保存 AppNamespace 对象到数据库
 23:         AppNamespace createdAppNamespace = appNamespaceService.createAppNamespaceInLocal(appNamespace);
 24:         // 赋予权限，若满足如下任一条件：
 25:         // 1. 公开类型的 AppNamespace 。
 26:         // 2. 私有类型的 AppNamespace ，并且允许 App 管理员创建私有类型的 AppNamespace 。
 27:         if (portalConfig.canAppAdminCreatePrivateNamespace() || createdAppNamespace.isPublic()) {
 28:             // 授予 Namespace Role
 29:             assignNamespaceRoleToOperator(appId, appNamespace.getName());
 30:         }
 31:         // 发布 AppNamespaceCreationEvent 创建事件
 32:         publisher.publishEvent(new AppNamespaceCreationEvent(createdAppNamespace));
 33:         // 返回创建的 AppNamespace 对象
 34:         return createdAppNamespace;
 35:     }
 36: 
 37: }
```

* **POST `apps/{appId}/appnamespaces` 接口**，Request Body 传递 **JSON** 对象。
* `@PreAuthorize(...)` 注解，调用 `PermissionValidator#hasCreateAppNamespacePermission(appId, appNamespace)` 方法，校验是否有创建 AppNamespace 的权限。后续文章，详细分享。
* 第 15 行：调用 `RequestPrecondition#checkArgumentsNotEmpty(String... args)` 方法，校验 AppNamespace 的 `appId` 和 `name` 非空。
* 第 16 至 21 行：调用 `InputValidator#isValidAppNamespace(name)` 方法，校验 AppNamespace 的 `name` 格式正确，符合 `[0-9a-zA-Z_.-]+"` 和 `[a-zA-Z0-9._-]+(?<!\.(json|yml|yaml|xml|properties))$` 格式。
* 第 23 行：调用 `AppNamespaceService#createAppNamespaceInLocal(AppNamespace)` 方法，保存 AppNamespace 对象到 **Portal DB** 数据库。在 [「3.2 AppNamespaceService」](#) 中，详细解析。
* 第 27 至 30 行：调用 `#assignNamespaceRoleToOperator(String appId, String namespaceName)` 方法，授予 Namespace Role ，需要满足如下**任一**条件。
    * 1、 **公开**类型的 AppNamespace 。
    * 2、**私有**类型的 AppNamespace ，并且允许 App 管理员创建私有类型的 AppNamespace 。

        > **admin.createPrivateNamespace.switch** 【在 ServerConfig 表】
        > 
        > 是否允许项目管理员创建 **private namespace** 。设置为 `true` 允许创建，设置为 `false` 则项目管理员在页面上看不到创建 **private namespace** 的选项。并且，项目管理员不允许创建 **private namespace** 。 

* 第 32 行：调用 `ApplicationEventPublisher#publishEvent(AppNamespaceCreationEvent)` 方法，发布 `com.ctrip.framework.apollo.portal.listener.AppNamespaceCreationEvent` 事件。
* 第 38 行：返回创建的 AppNamespace 对象。

## 3.2 AppNamespaceService

在 `apollo-portal` 项目中，`com.ctrip.framework.apollo.portal.service.AppNamespaceService` ，提供 AppNamespace 的 **Service** 逻辑。

`#createAppNamespaceInLocal(AppNamespace)` 方法，保存 AppNamespace 对象到 **Portal DB** 数据库。代码如下：

```Java
  1: @Autowired
  2: private UserInfoHolder userInfoHolder;
  3: @Autowired
  4: private AppNamespaceRepository appNamespaceRepository;
  5: @Autowired
  6: private RoleInitializationService roleInitializationService;
  7: @Autowired
  8: private AppService appService;
  9: 
 10: @Transactional
 11: public AppNamespace createAppNamespaceInLocal(AppNamespace appNamespace) {
 12:     String appId = appNamespace.getAppId();
 13:     // 校验对应的 App 是否存在。若不存在，抛出 BadRequestException 异常
 14:     // add app org id as prefix
 15:     App app = appService.load(appId);
 16:     if (app == null) {
 17:         throw new BadRequestException("App not exist. AppId = " + appId);
 18:     }
 19:     // 拼接 AppNamespace 的 `name` 属性。
 20:     StringBuilder appNamespaceName = new StringBuilder();
 21:     // add prefix postfix
 22:     appNamespaceName
 23:             .append(appNamespace.isPublic() ? app.getOrgId() + "." : "") // 公用类型，拼接组织编号
 24:             .append(appNamespace.getName())
 25:             .append(appNamespace.formatAsEnum() == ConfigFileFormat.Properties ? "" : "." + appNamespace.getFormat());
 26:     appNamespace.setName(appNamespaceName.toString());
 27:     // 设置 AppNamespace 的 `comment` 属性为空串，若为 null 。
 28:     if (appNamespace.getComment() == null) {
 29:         appNamespace.setComment("");
 30:     }
 31:     // 校验 AppNamespace 的 `format` 是否合法
 32:     if (!ConfigFileFormat.isValidFormat(appNamespace.getFormat())) {
 33:         throw new BadRequestException("Invalid namespace format. format must be properties、json、yaml、yml、xml");
 34:     }
 35:     // 设置 AppNamespace 的创建和修改人
 36:     String operator = appNamespace.getDataChangeCreatedBy();
 37:     if (StringUtils.isEmpty(operator)) {
 38:         operator = userInfoHolder.getUser().getUserId(); // 当前登录管理员
 39:         appNamespace.setDataChangeCreatedBy(operator);
 40:     }
 41:     appNamespace.setDataChangeLastModifiedBy(operator);
 42:     // 公用类型，校验 `name` 在全局唯一
 43:     // unique check
 44:     if (appNamespace.isPublic() && findPublicAppNamespace(appNamespace.getName()) != null) {
 45:         throw new BadRequestException(appNamespace.getName() + "已存在");
 46:     }
 47:     // 私有类型，校验 `name` 在 App 下唯一
 48:     if (!appNamespace.isPublic() && appNamespaceRepository.findByAppIdAndName(appNamespace.getAppId(), appNamespace.getName()) != null) {
 49:         throw new BadRequestException(appNamespace.getName() + "已存在");
 50:     }
 51:     // 保存 AppNamespace 到数据库
 52:     AppNamespace createdAppNamespace = appNamespaceRepository.save(appNamespace);
 53:     // 初始化 Namespace 的 Role 们
 54:     roleInitializationService.initNamespaceRoles(appNamespace.getAppId(), appNamespace.getName(), operator);
 55:     return createdAppNamespace;
 56: }
```

* 第 15 至 18 行：调用 `AppService.load(appId)` 方法，获得对应的 App 对象。当**校验** App 不存在时，抛出 BadRequestException 异常。
* 第 19 至 26 行：拼接并设置 AppNamespace 的 `name` 属性。
* 第 27 至 30 行：**设置** AppNamespace 的 `comment` 属性为空串，若为 null 。
* 第 31 至 34 行：**校验** AppNamespace 的 `format` 是否合法。
* 第 35 至 41 行：**设置** AppNamespace 的创建和修改人。
* 第 42 至 46 行：若 AppNamespace 为公用类型，**校验** `name` 在**全局**唯一，否则抛出 BadRequestException 异常。`#findPublicAppNamespace(name)` 方法，代码如下：

    ```Java
    public AppNamespace findPublicAppNamespace(String namespaceName) {
        return appNamespaceRepository.findByNameAndIsPublic(namespaceName, true);
    }
    ```

* 第 47 至 50 行：若 AppNamespace 为私有类型，**校验** `name` 在 **App** 唯一否则抛出 BadRequestException 异常。
* 第 52 行：调用 `AppNamespaceRepository#save(AppNamespace)` 方法，保存 AppNamespace 到数据库。
* 第 54 行：初始化 Namespace 的 Role 们。详解解析，见 [《Apollo 源码解析 —— Portal 认证与授权（二）之授权》](http://www.iocoder.cn/Apollo/portal-auth-2?self) 。

-------

`#createDefaultAppNamespace(appId)` 方法，创建并保存 App 下默认的 `"application"` 的 AppNamespace 到数据库。代码如下：

```Java
@Transactional
public void createDefaultAppNamespace(String appId) {
    // 校验 `name` 在 App 下唯一
    if (!isAppNamespaceNameUnique(appId, ConfigConsts.NAMESPACE_APPLICATION)) {
        throw new BadRequestException(String.format("App already has application namespace. AppId = %s", appId));
    }
    // 创建 AppNamespace 对象
    AppNamespace appNs = new AppNamespace();
    appNs.setAppId(appId);
    appNs.setName(ConfigConsts.NAMESPACE_APPLICATION); // `application`
    appNs.setComment("default app namespace");
    appNs.setFormat(ConfigFileFormat.Properties.getValue());
    // 设置 AppNamespace 的创建和修改人为当前管理员
    String userId = userInfoHolder.getUser().getUserId();
    appNs.setDataChangeCreatedBy(userId);
    appNs.setDataChangeLastModifiedBy(userId);
    // 保存 AppNamespace 到数据库
    appNamespaceRepository.save(appNs);
}
```

* 在 App **创建**时，会调用该方法。

## 3.3 AppNamespaceRepository

在 `apollo-portal` 项目中，`com.ctrip.framework.apollo.common.entity.App.AppNamespaceRepository` ，继承 `org.springframework.data.repository.PagingAndSortingRepository` 接口，提供 AppNamespace 的**数据访问**，即 **DAO** 。

代码如下：

```Java
public interface AppNamespaceRepository extends PagingAndSortingRepository<AppNamespace, Long> {

  AppNamespace findByAppIdAndName(String appId, String namespaceName);

  AppNamespace findByName(String namespaceName);

  AppNamespace findByNameAndIsPublic(String namespaceName, boolean isPublic);

  List<AppNamespace> findByIsPublicTrue();

}
```

## 3.4 AppNamespaceCreationEvent

`com.ctrip.framework.apollo.portal.listener.AppNamespaceCreationEvent` ，实现 `org.springframework.context.ApplicationEvent` 抽象类，AppNamespace **创建**事件。

代码如下：

```Java
public class AppNamespaceCreationEvent extends ApplicationEvent {

  public AppNamespaceCreationEvent(Object source) {
    super(source);
  }

  public AppNamespace getAppNamespace() {
    Preconditions.checkState(source != null);
    return (AppNamespace) this.source;
  }

}
```

* **构造方法**，将 AppNamespace 对象作为*方法参数*传入。
* `#getAppNamespace()` 方法，获得事件对应的 AppNamespace 对象。

### 3.4.1 CreationListener

`com.ctrip.framework.apollo.portal.listener.CreationListener` ，**对象创建**监听器，目前监听 AppCreationEvent 和 AppNamespaceCreationEvent 事件。

我们以  AppNamespaceCreationEvent 举例子，代码如下：

```Java
@EventListener
public void onAppNamespaceCreationEvent(AppNamespaceCreationEvent event) {
    // 将 AppNamespace 转成 AppNamespaceDTO 对象
    AppNamespaceDTO appNamespace = BeanUtils.transfrom(AppNamespaceDTO.class, event.getAppNamespace());
    // 获得有效的 Env 数组
    List<Env> envs = portalSettings.getActiveEnvs();
    // 循环 Env 数组，调用对应的 Admin Service 的 API ，创建 AppNamespace 对象。
    for (Env env : envs) {
        try {
            namespaceAPI.createAppNamespace(env, appNamespace);
        } catch (Throwable e) {
            logger.error("Create appNamespace failed. appId = {}, env = {}", appNamespace.getAppId(), env, e);
            Tracer.logError(String.format("Create appNamespace failed. appId = %s, env = %s", appNamespace.getAppId(), env), e);
        }
    }
}
```

## 3.5 NamespaceAPI

`com.ctrip.framework.apollo.portal.api.NamespaceAPI` ，实现 API 抽象类，封装对 Admin Service 的 AppNamespace 和 Namespace **两个**模块的 API 调用。代码如下：

![NamespaceAPI](http://www.iocoder.cn/images/Apollo/2018_03_10/07.png)

* 使用 `restTemplate` ，调用对应的 API 接口。

# 4. Admin Service 侧

## 4.1 AppNamespaceController

在 `apollo-adminservice` 项目中， `com.ctrip.framework.apollo.adminservice.controller.AppNamespaceController` ，提供 AppNamespace 的 **API** 。

`#create(AppNamespaceDTO)` 方法，创建 AppNamespace 。代码如下：

```Java
  1: @RestController
  2: public class AppNamespaceController {
  3: 
  4:     @Autowired
  5:     private AppNamespaceService appNamespaceService;
  8: 
  9:     /**
 10:      * 创建 AppNamespace
 11:      *
 12:      * @param appNamespace AppNamespaceDTO 对象
 13:      * @return AppNamespace 对象
 14:      */
 15:     @RequestMapping(value = "/apps/{appId}/appnamespaces", method = RequestMethod.POST)
 16:     public AppNamespaceDTO create(@RequestBody AppNamespaceDTO appNamespace) {
 17:         // 将 AppNamespaceDTO 转换成 AppNamespace 对象
 18:         AppNamespace entity = BeanUtils.transfrom(AppNamespace.class, appNamespace);
 19:         // 判断 `name` 在 App 下是否已经存在对应的 AppNamespace 对象。若已经存在，抛出 BadRequestException 异常。
 20:         AppNamespace managedEntity = appNamespaceService.findOne(entity.getAppId(), entity.getName());
 21:         if (managedEntity != null) {
 22:             throw new BadRequestException("app namespaces already exist.");
 23:         }
 24:         // 设置 AppNamespace 的 format 属性为 "properties"，若为 null 。
 25:         if (StringUtils.isEmpty(entity.getFormat())) {
 26:             entity.setFormat(ConfigFileFormat.Properties.getValue());
 27:         }
 28:         // 保存 AppNamespace 对象到数据库
 29:         entity = appNamespaceService.createAppNamespace(entity);
 30:         // 将保存的 AppNamespace 对象，转换成 AppNamespaceDTO 返回
 31:         return BeanUtils.transfrom(AppNamespaceDTO.class, entity);
 32:     }
 33:     
 34:     // ... 省略其他接口和属性
 35: }
```

* **POST `/apps/{appId}/appnamespaces` 接口**，Request Body 传递 **JSON** 对象。
* 第 22 行：调用 `BeanUtils#transfrom(Class<T> clazz, Object src)` 方法，将 AppNamespaceDTO **转换**成 AppNamespace对象。
* 第 20 至 23 行：调用 `AppNamespaceService#findOne(appId, name)` 方法，**校验** `name`  在 App 下，是否已经存在对应的 AppNamespace 对象。若已经存在，抛出 BadRequestException 异常。
* 第 24 至 27 行：**设置** AppNamespace 的 `format` 属性为 `"properties"`，若为 null 。
* 第 29 行：调用 `AppNamespaceService#createAppNamespace(AppNamespace)` 方法，保存 AppNamespace 对象到数据库。
* 第 30 至 32 行：调用 `BeanUtils#transfrom(Class<T> clazz, Object src)` 方法，将保存的 AppNamespace 对象，转换成 AppNamespaceDTO 返回。

## 4.2 AppNamespaceService

在 `apollo-biz` 项目中，`com.ctrip.framework.apollo.biz.service.AppNamespaceService` ，提供 AppNamespace 的 **Service** 逻辑给 Admin Service 和 Config Service 。

`#save(AppNamespace)` 方法，保存 AppNamespace 对象到数据库中。代码如下：

```Java
  1: @Autowired
  2: private AppNamespaceRepository appNamespaceRepository;
  3: @Autowired
  4: private NamespaceService namespaceService;
  5: @Autowired
  6: private ClusterService clusterService;
  7: @Autowired
  8: private AuditService auditService;
  9: 
 10: @Transactional
 11: public AppNamespace createAppNamespace(AppNamespace appNamespace) {
 12:     // 判断 `name` 在 App 下是否已经存在对应的 AppNamespace 对象。若已经存在，抛出 ServiceException 异常。
 13:     String createBy = appNamespace.getDataChangeCreatedBy();
 14:     if (!isAppNamespaceNameUnique(appNamespace.getAppId(), appNamespace.getName())) {
 15:         throw new ServiceException("appnamespace not unique");
 16:     }
 17:     // 保护代码，避免 App 对象中，已经有 id 属性。
 18:     appNamespace.setId(0);// protection
 19:     appNamespace.setDataChangeCreatedBy(createBy);
 20:     appNamespace.setDataChangeLastModifiedBy(createBy);
 21:     // 保存 AppNamespace 到数据库
 22:     appNamespace = appNamespaceRepository.save(appNamespace);
 23:     // 创建 AppNamespace 在 App 下，每个 Cluster 的 Namespace 对象。
 24:     instanceOfAppNamespaceInAllCluster(appNamespace.getAppId(), appNamespace.getName(), createBy);
 25:     // 记录 Audit 到数据库中
 26:     auditService.audit(AppNamespace.class.getSimpleName(), appNamespace.getId(), Audit.OP.INSERT, createBy);
 27:     return appNamespace;
 28: }
```

* 第 12 至 16 行：调用 `#isAppNamespaceNameUnique(appId, name)` 方法，判断 `name` 在 App 下是否已经存在对应的 AppNamespace 对象。若已经存在，抛出 ServiceException 异常。代码如下：

    ```Java
    public boolean isAppNamespaceNameUnique(String appId, String namespaceName) {
        Objects.requireNonNull(appId, "AppId must not be null");
        Objects.requireNonNull(namespaceName, "Namespace must not be null");
        return Objects.isNull(appNamespaceRepository.findByAppIdAndName(appId, namespaceName));
    }
    ```

* 第 18 行：置“**空**” AppNamespace 的编号，防御性编程，避免 AppNamespace 对象中，已经有 `id` 属性。
* 第 22 行：调用 `AppNamespaceRepository#save(AppNamespace)` 方法，保存 AppNamespace 对象到数据库中。
* 第 24 行：调用 `#instanceOfAppNamespaceInAllCluster(appId, namespaceName, createBy)` 方法，创建 AppNamespace 在 App 下，**每个** Cluster 的 Namespace 对象。代码如下：

    ```Java
    private void instanceOfAppNamespaceInAllCluster(String appId, String namespaceName, String createBy) {
        // 获得 App 下所有的 Cluster 数组
        List<Cluster> clusters = clusterService.findParentClusters(appId);
        // 循环 Cluster 数组，创建并保存 Namespace 到数据库
        for (Cluster cluster : clusters) {
            Namespace namespace = new Namespace();
            namespace.setClusterName(cluster.getName());
            namespace.setAppId(appId);
            namespace.setNamespaceName(namespaceName);
            namespace.setDataChangeCreatedBy(createBy);
            namespace.setDataChangeLastModifiedBy(createBy);
            namespaceService.save(namespace);
        }
    }
    ```

* 第 26 行：记录 Audit 到数据库中。

## 4.3 AppNamespaceRepository

`com.ctrip.framework.apollo.biz.repository.AppNamespaceRepository` ，继承 `org.springframework.data.repository.PagingAndSortingRepository` 接口，提供 AppNamespace 的**数据访问** 给 Admin Service 和 Config Service 。代码如下：

```Java
public interface AppNamespaceRepository extends PagingAndSortingRepository<AppNamespace, Long>{

  AppNamespace findByAppIdAndName(String appId, String namespaceName);

  List<AppNamespace> findByAppIdAndNameIn(String appId, Set<String> namespaceNames);

  AppNamespace findByNameAndIsPublicTrue(String namespaceName);

  List<AppNamespace> findByNameInAndIsPublicTrue(Set<String> namespaceNames);

  List<AppNamespace> findByAppIdAndIsPublic(String appId, boolean isPublic);

  List<AppNamespace> findByAppId(String appId);

  List<AppNamespace> findFirst500ByIdGreaterThanOrderByIdAsc(long id);

}
```

## 4.4 NamespaceService

在 `apollo-biz` 项目中，`com.ctrip.framework.apollo.biz.service.NamespaceService` ，提供 Namespace 的 **Service** 逻辑给 Admin Service 和 Config Service 。

`#save(Namespace)` 方法，保存 Namespace 对象到数据库中。代码如下：

```Java
  1: @Autowired
  2: private NamespaceRepository namespaceRepository;
  3: @Autowired
  4: private AuditService auditService;
  5: 
  6: @Transactional
  7: public Namespace save(Namespace entity) {
  8:     // 判断是否已经存在。若是，抛出 ServiceException 异常。
  9:     if (!isNamespaceUnique(entity.getAppId(), entity.getClusterName(), entity.getNamespaceName())) {
 10:         throw new ServiceException("namespace not unique");
 11:     }
 12:     // 保护代码，避免 Namespace 对象中，已经有 id 属性。
 13:     entity.setId(0);//protection
 14:     // 保存 Namespace 到数据库
 15:     Namespace namespace = namespaceRepository.save(entity);
 16:     // 记录 Audit 到数据库中
 17:     auditService.audit(Namespace.class.getSimpleName(), namespace.getId(), Audit.OP.INSERT, namespace.getDataChangeCreatedBy());
 18:     return namespace;
 19: }
```

* 第 8 至 11 行：调用 `#isNamespaceUnique(appId, cluster, namespace)` 方法，**校验**是否已经存在。若是，抛出 ServiceException 异常。代码如下：

    ```Java
    public boolean isNamespaceUnique(String appId, String cluster, String namespace) {
        Objects.requireNonNull(appId, "AppId must not be null");
        Objects.requireNonNull(cluster, "Cluster must not be null");
        Objects.requireNonNull(namespace, "Namespace must not be null");
        return Objects.isNull(namespaceRepository.findByAppIdAndClusterNameAndNamespaceName(appId, cluster, namespace));
    }
    ```

* 第 12 行：置“**空**” Namespace 的编号，防御性编程，避免 Namespace 对象中，已经有 `id` 属性。
* 第 15 行：调用 `NamespaceRepository#save(AppNamespace)` 方法，保存 Namespace 对象到数据库中。
* 第 17 行：记录 Audit 到数据库中。

-------

`#instanceOfAppNamespaces(appId, clusterName, createBy)` 方法，创建并保存 App 下**指定** Cluster 的 Namespace 到数据库。代码如下：

```Java
@Transactional
public void instanceOfAppNamespaces(String appId, String clusterName, String createBy) {
    // 获得所有的 AppNamespace 对象
    List<AppNamespace> appNamespaces = appNamespaceService.findByAppId(appId);
    // 循环 AppNamespace 数组，创建并保存 Namespace 到数据库
    for (AppNamespace appNamespace : appNamespaces) {
        Namespace ns = new Namespace();
        ns.setAppId(appId);
        ns.setClusterName(clusterName);
        ns.setNamespaceName(appNamespace.getName());
        ns.setDataChangeCreatedBy(createBy);
        ns.setDataChangeLastModifiedBy(createBy);
        namespaceRepository.save(ns);
        // 记录 Audit 到数据库中
        auditService.audit(Namespace.class.getSimpleName(), ns.getId(), Audit.OP.INSERT, createBy);
    }
}
```

* 在 App **创建**时，传入 Cluster 为 `default` ，此时只有 **1** 个 AppNamespace 对象。
* 在 Cluster **创建**时，传入**自己**，此处可以有**多**个 AppNamespace 对象。

## 4.5 NamespaceRepository

`com.ctrip.framework.apollo.biz.repository.NamespaceRepository` ，继承 `org.springframework.data.repository.PagingAndSortingRepository` 接口，提供 Namespace 的**数据访问** 给 Admin Service 和 Config Service 。代码如下：

```Java
public interface NamespaceRepository extends PagingAndSortingRepository<Namespace, Long> {

  List<Namespace> findByAppIdAndClusterNameOrderByIdAsc(String appId, String clusterName);

  Namespace findByAppIdAndClusterNameAndNamespaceName(String appId, String clusterName, String namespaceName);

  @Modifying
  @Query("update Namespace set isdeleted=1,DataChange_LastModifiedBy = ?3 where appId=?1 and clusterName=?2")
  int batchDelete(String appId, String clusterName, String operator);

  List<Namespace> findByAppIdAndNamespaceName(String appId, String namespaceName);

  List<Namespace> findByNamespaceName(String namespaceName, Pageable page);

  int countByNamespaceNameAndAppIdNot(String namespaceName, String appId);

}
```

# 666. 彩蛋

类似于 App 的创建，AppNamespace 也存在**跨系统**同步的一致性问题。但是，目前暂未提供**补偿**机制，如果 Portal 创建 AppNamespace 成功，而调用远程 Admin Service 失败，则会出现不一致的情况。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

