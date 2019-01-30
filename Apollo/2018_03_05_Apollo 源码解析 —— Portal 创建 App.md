title: Apollo 源码解析 —— Portal 创建 App
date: 2018-03-05
tags:
categories: Apollo
permalink: Apollo/portal-create-app
wechat_url:
toutiao_url: https://www.toutiao.com/i6634860180750205454/

---

摘要: 原创出处 http://www.iocoder.cn/Apollo/portal-create-app/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Apollo/portal-create-app/)
- [2. App](http://www.iocoder.cn/Apollo/portal-create-app/)
  - [2.1 BaseEntity](http://www.iocoder.cn/Apollo/portal-create-app/)
  - [2.2 为什么需要同步](http://www.iocoder.cn/Apollo/portal-create-app/)
- [3. Portal 侧](http://www.iocoder.cn/Apollo/portal-create-app/)
  - [3.1 AppController](http://www.iocoder.cn/Apollo/portal-create-app/)
  - [3.2 AppService](http://www.iocoder.cn/Apollo/portal-create-app/)
  - [3.3 AppRepository](http://www.iocoder.cn/Apollo/portal-create-app/)
  - [3.4 AppCreationEvent](http://www.iocoder.cn/Apollo/portal-create-app/)
  - [3.5 AdminServiceAPI](http://www.iocoder.cn/Apollo/portal-create-app/)
- [4. Admin Service 侧](http://www.iocoder.cn/Apollo/portal-create-app/)
  - [4.1 AppController](http://www.iocoder.cn/Apollo/portal-create-app/)
  - [4.2 AdminService](http://www.iocoder.cn/Apollo/portal-create-app/)
  - [4.3 AppService](http://www.iocoder.cn/Apollo/portal-create-app/)
  - [4.4 AppRepository](http://www.iocoder.cn/Apollo/portal-create-app/)
- [666. 彩蛋](http://www.iocoder.cn/Apollo/portal-create-app/)

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

> 老艿艿：本系列假定胖友已经阅读过 [《Apollo 官方 wiki 文档》](https://github.com/ctripcorp/apollo/wiki/) 。

本文分享 **Portal 创建 App** 的流程，整个过程涉及 Portal、Admin Service ，如下图所示：

![流程](http://www.iocoder.cn/images/Apollo/2018_03_05/01.png)

下面，我们先来看看 App 的实体结构

> 老艿艿：因为 Portal 是管理后台，所以从代码实现上，和业务系统非常相像。也因此，本文会略显啰嗦。

# 2. App

在 `apollo-common` 项目中， `com.ctrip.framework.apollo.common.entity.App` ，继承 BaseEntity 抽象类，应用信息**实体**。代码如下：

```Java
@Entity
@Table(name = "App")
@SQLDelete(sql = "Update App set isDeleted = 1 where id = ?")
@Where(clause = "isDeleted = 0")
public class App extends BaseEntity {

    /**
     * App 名
     */
    @Column(name = "Name", nullable = false)
    private String name;
    /**
     * App 编号
     */
    @Column(name = "AppId", nullable = false)
    private String appId;
    /**
     * 部门编号
     */
    @Column(name = "OrgId", nullable = false)
    private String orgId;
    /**
     * 部门名
     *
     * 冗余字段
     */
    @Column(name = "OrgName", nullable = false)
    private String orgName;
    /**
     * 拥有人名
     *
     * 例如在 Portal 系统中，使用系统的管理员账号，即 UserPO.username 字段
     */
    @Column(name = "OwnerName", nullable = false)
    private String ownerName;
    /**
     * 拥有人邮箱
     *
     * 冗余字段
     */
    @Column(name = "OwnerEmail", nullable = false)
    private String ownerEmail;
}
```

* ORM 选用 **Hibernate** 框架。
* `@SQLDelete(...)` + `@Where(...)` 注解，配合 `BaseEntity.extends` 字段，实现 App 的**逻辑删除**。 
* 字段比较简单，胖友看下注释。

## 2.1 BaseEntity

`com.ctrip.framework.apollo.common.entity.BaseEntity` ，**基础**实体**抽象类**。代码如下：

```Java
@MappedSuperclass
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class BaseEntity {

    /**
     * 编号
     */
    @Id
    @GeneratedValue
    @Column(name = "Id")
    private long id;
    /**
     * 是否删除
     */
    @Column(name = "IsDeleted", columnDefinition = "Bit default '0'")
    protected boolean isDeleted = false;
    /**
     * 数据创建人
     *
     * 例如在 Portal 系统中，使用系统的管理员账号，即 UserPO.username 字段
     */
    @Column(name = "DataChange_CreatedBy", nullable = false)
    private String dataChangeCreatedBy;
    /**
     * 数据创建时间
     */
    @Column(name = "DataChange_CreatedTime", nullable = false)
    private Date dataChangeCreatedTime;
    /**
     * 数据最后更新人
     *
     * 例如在 Portal 系统中，使用系统的管理员账号，即 UserPO.username 字段
     */
    @Column(name = "DataChange_LastModifiedBy")
    private String dataChangeLastModifiedBy;
    /**
     * 数据最后更新时间
     */
    @Column(name = "DataChange_LastTime")
    private Date dataChangeLastModifiedTime;

    /**
     * 保存前置方法
     */
    @PrePersist
    protected void prePersist() {
        if (this.dataChangeCreatedTime == null) dataChangeCreatedTime = new Date();
        if (this.dataChangeLastModifiedTime == null) dataChangeLastModifiedTime = new Date();
    }

    /**
     * 更新前置方法
     */
    @PreUpdate
    protected void preUpdate() {
        this.dataChangeLastModifiedTime = new Date();
    }

    /**
     * 删除前置方法
     */
    @PreRemove
    protected void preRemove() {
        this.dataChangeLastModifiedTime = new Date();
    }
    
    // ... 省略 setting / getting 方法
}  
```

* `@MappedSuperclass` 注解，见 [《Hibernate 中 @MappedSuperclass 注解的使用说明》](https://blog.csdn.net/u012402177/article/details/78666532) 文章。
* `@Inheritance(...)` 注解，见 [《Hibernate（11）映射继承关系二之每个类对应一张表（@Inheritance(strategy=InheritanceType.TABLE_PER_CLASS）》](https://blog.csdn.net/jiangshangchunjiezi/article/details/78522924) 文章。
* `id` 字段，编号，Long 型，全局自增。
* `isDeleted` 字段，是否删除，用于**逻辑删除**的功能。
* `dataChangeCreatedBy` 和 `dataChangeCreatedTime` 字段，实现数据的创建人和时间的记录，方便追踪。
* `dataChangeLastModifiedBy` 和 `dataChangeLastModifiedTime` 字段，实现数据的更新人和时间的记录，方便追踪。
* `@PrePersist`、`@PreUpdate`、`@PreRemove` 注解，CRD 操作前，设置对应的**时间字段**。
* 在 Apollo 中，**所有**实体都会继承 BaseEntity ，实现**公用字段**的**统一**定义。这种设计值得**借鉴**，特别是**创建时间**和**更新时间**这两个字段，特别适合线上追踪问题和数据同步。

## 2.2 为什么需要同步

在文初的流程图中，我们看到 App 创建时，在 Portal Service 存储完成后，会**异步**同步到 Admin Service 中，这是为什么呢？

在 Apollo 的架构中，**一个**环境( Env ) 对应一套 Admin Service 和 Config Service 。  
而 Portal Service 会管理**所有**环境( Env ) 。因此，每次创建 App 后，需要进行同步。

或者说，App 在 Portal Service 中，表示需要**管理**的 App 。而在 Admin Service 和 Config Service 中，表示**存在**的 App 。

# 3. Portal 侧

## 3.1 AppController

在 `apollo-portal` 项目中，`com.ctrip.framework.apollo.portal.controller.AppController` ，提供 App 的 **API** 。

在**创建项目**的界面中，点击【提交】按钮，调用**创建 App 的 API** 。

![创建项目](http://www.iocoder.cn/images/Apollo/2018_03_05/02.png)

代码如下：

```Java
  1: @RestController
  2: @RequestMapping("/apps")
  3: public class AppController {
  4: 
  5:     @Autowired
  6:     private UserInfoHolder userInfoHolder;
  7:     @Autowired
  8:     private AppService appService;
  9:     /**
 10:      * Spring 事件发布者
 11:      */
 12:     @Autowired
 13:     private ApplicationEventPublisher publisher;
 14:     @Autowired
 15:     private RolePermissionService rolePermissionService;
 16: 
 17:     /**
 18:      * 创建 App
 19:      *
 20:      * @param appModel AppModel 对象
 21:      * @return App 对象
 22:      */
 23:     @RequestMapping(value = "", method = RequestMethod.POST)
 24:     public App create(@RequestBody AppModel appModel) {
 25:         // 将 AppModel 转换成 App 对象
 26:         App app = transformToApp(appModel);
 27:         // 保存 App 对象到数据库
 28:         App createdApp = appService.createAppInLocal(app);
 29:         // 发布 AppCreationEvent 创建事件
 30:         publisher.publishEvent(new AppCreationEvent(createdApp));
 31:         // 授予 App 管理员的角色
 32:         Set<String> admins = appModel.getAdmins();
 33:         if (!CollectionUtils.isEmpty(admins)) {
 34:             rolePermissionService.assignRoleToUsers(RoleUtils.buildAppMasterRoleName(createdApp.getAppId()),
 35:                     admins, userInfoHolder.getUser().getUserId());
 36:         }
 37:         // 返回 App 对象
 38:         return createdApp;
 39:     }
 40:     
 41:     // ... 省略其他接口和属性
 42: }    
```

* **POST `apps` 接口**，Request Body 传递 **JSON** 对象。
* [`com.ctrip.framework.apollo.portal.entity.model.AppModel`](https://github.com/YunaiV/apollo/blob/master/apollo-portal/src/main/java/com/ctrip/framework/apollo/portal/entity/model/AppModel.java) ，App Model 。在 `com.ctrip.framework.apollo.portal.entity.model` 包下，负责接收来自 Portal 界面的**复杂**请求对象。例如，AppModel 一方面带有创建 App 对象需要的属性，另外也带有需要授权管理员的编号集合 `admins` ，即存在**跨模块**的情况。
* 第 26 行：调用 [`#transformToApp(AppModel)`](https://github.com/YunaiV/apollo/blob/e7984de5d6ed8124184f8107e079f9d84462f037/apollo-portal/src/main/java/com/ctrip/framework/apollo/portal/controller/AppController.java#L171-L188) 方法，将 AppModel 转换成 App 对象。🙂 转换方法很简单，点击方法，直接查看。
* 第 28 行：调用 `AppService#createAppInLocal(App)` 方法，保存 App 对象到 **Portal DB** 数据库。在 [「3.2 AppService」](#) 中，详细解析。
* 第 30 行：调用 `ApplicationEventPublisher#publishEvent(AppCreationEvent)` 方法，发布 `com.ctrip.framework.apollo.portal.listener.AppCreationEvent` 事件。
* 第 31 至 36 行：授予 App 管理员的角色。详细解析，见 [《Apollo 源码解析 —— Portal 认证与授权（二）之授权》](http://www.iocoder.cn/Apollo/portal-auth-2?self) 。
* 第 38 行：返回创建的 App 对象。

## 3.2 AppService

在 `apollo-portal` 项目中，`com.ctrip.framework.apollo.portal.service.AppService` ，提供 App 的 **Service** 逻辑。

`#createAppInLocal(App)` 方法，保存 App 对象到 **Portal DB** 数据库。代码如下：

```Java
  1: @Autowired
  2: private UserInfoHolder userInfoHolder;
  3: @Autowired
  4: private AppRepository appRepository;
  5: @Autowired
  6: private AppNamespaceService appNamespaceService;
  7: @Autowired
  8: private RoleInitializationService roleInitializationService;
  9: @Autowired
 10: private UserService userService;
 11:     
 12: @Transactional
 13: public App createAppInLocal(App app) {
 14:     String appId = app.getAppId();
 15:     // 判断 `appId` 是否已经存在对应的 App 对象。若已经存在，抛出 BadRequestException 异常。
 16:     App managedApp = appRepository.findByAppId(appId);
 17:     if (managedApp != null) {
 18:         throw new BadRequestException(String.format("App already exists. AppId = %s", appId));
 19:     }
 20:     // 获得 UserInfo 对象。若不存在，抛出 BadRequestException 异常
 21:     UserInfo owner = userService.findByUserId(app.getOwnerName());
 22:     if (owner == null) {
 23:         throw new BadRequestException("Application's owner not exist.");
 24:     }
 25:     app.setOwnerEmail(owner.getEmail()); // Email
 26:     // 设置 App 的创建和修改人
 27:     String operator = userInfoHolder.getUser().getUserId();
 28:     app.setDataChangeCreatedBy(operator);
 29:     app.setDataChangeLastModifiedBy(operator);
 30:     // 保存 App 对象到数据库
 31:     App createdApp = appRepository.save(app);
 32:     // 创建 App 的默认命名空间 "application"
 33:     appNamespaceService.createDefaultAppNamespace(appId);
 34:     // 初始化 App 角色
 35:     roleInitializationService.initAppRoles(createdApp);
 36:     // 【TODO 6001】Tracer 日志
 37:     Tracer.logEvent(TracerEventType.CREATE_APP, appId);
 38:     return createdApp;
 39: }
```

* 第 15 至 19 行：调用 `AppRepository#findByAppId(appId)` 方法，判断 `appId` 是否已经存在对应的 App 对象。若已经存在，抛出 BadRequestException 异常。
* 第 20 至 25 行：调用 `UserService#findByUserId(userId)` 方法，获得 [`com.ctrip.framework.apollo.portal.entity.bo.UserInfo`](https://github.com/YunaiV/apollo/blob/master/apollo-portal/src/main/java/com/ctrip/framework/apollo/portal/entity/bo/UserInfo.java) 对象。`com.ctrip.framework.apollo.portal.entity.bo` 包下，负责返回 Service 的**业务**对象。例如，UserInfo 只包含 `com.ctrip.framework.apollo.portal.entity.po.UserPO` 的部分属性：`userId`、`username`、`email` 。
* 第 27 至 29 行：调用 `UserInfoHolder#getUser()#getUserId()` 方法，获得当前登录用户，并设置为 App 的创建和修改人。关于 UserInfoHolder ，后续文章，详细分享。
* 第 31 行：调用 `AppRepository#save(App)` 方法，保存 App 对象到数据库中。
* 第 33 行：调用 `AppNameSpaceService#createDefaultAppNamespace(appId)` 方法，创建 App 的**默认** Namespace (命名空间) `"application"` 。对于每个 App ，都会有一个默认 Namespace 。具体的代码实现，我们在 [《Apollo 源码解析 —— Portal 创建 Namespace》](http://www.iocoder.cn/Apollo/portal-create-namespace/?self)
* 第 35 行：初始化 App 角色。详解解析，见 [《Apollo 源码解析 —— Portal 认证与授权（二）之授权》](http://www.iocoder.cn/Apollo/portal-auth-2?self) 。
* 第 37 行：【TODO 6001】Tracer 日志

## 3.3 AppRepository

在 `apollo-portal` 项目中，`com.ctrip.framework.apollo.common.entity.App.AppRepository` ，继承 `org.springframework.data.repository.PagingAndSortingRepository` 接口，提供 App 的**数据访问**，即 **DAO** 。

代码如下：

```Java
public interface AppRepository extends PagingAndSortingRepository<App, Long> {

  App findByAppId(String appId);

  List<App> findByOwnerName(String ownerName, Pageable page);

  List<App> findByAppIdIn(Set<String> appIds);

}
```

基于 Spring Data JPA 框架，使用 Hibernate 实现。详细参见 [《Spring Data JPA、Hibernate、JPA 三者之间的关系》](https://www.cnblogs.com/xiaoheike/p/5150553.html) 文章。 

🙂 不熟悉 Spring Data JPA 的胖友，可以看下 [《Spring Data JPA 介绍和使用》](https://www.jianshu.com/p/633922bb189f) 文章。

## 3.4 AppCreationEvent

`com.ctrip.framework.apollo.portal.listener.AppCreationEvent` ，实现 `org.springframework.context.ApplicationEvent` 抽象类，App **创建**事件。

代码如下：

```Java
public class AppCreationEvent extends ApplicationEvent {

  public AppCreationEvent(Object source) {
    super(source);
  }

  public App getApp() {
    Preconditions.checkState(source != null);
    return (App) this.source;
  }

}
```

* **构造方法**，将 App 对象作为*方法参数*传入。
* `#getApp()` 方法，获得事件对应的 App 对象。

### 3.4.1 CreationListener

`com.ctrip.framework.apollo.portal.listener.CreationListener` ，**对象创建**监听器，目前监听 AppCreationEvent 和 AppNamespaceCreationEvent 事件。

我们以  AppCreationEvent 举例子，代码如下：

```Java
  1: @Autowired
  2: private PortalSettings portalSettings;
  3: @Autowired
  4: private AdminServiceAPI.AppAPI appAPI;
  5: 
  6: @EventListener
  7: public void onAppCreationEvent(AppCreationEvent event) {
  8:     // 将 App 转成 AppDTO 对象
  9:     AppDTO appDTO = BeanUtils.transfrom(AppDTO.class, event.getApp());
 10:     // 获得有效的 Env 数组
 11:     List<Env> envs = portalSettings.getActiveEnvs();
 12:     // 循环 Env 数组，调用对应的 Admin Service 的 API ，创建 App 对象。
 13:     for (Env env : envs) {
 14:         try {
 15:             appAPI.createApp(env, appDTO);
 16:         } catch (Throwable e) {
 17:             logger.error("Create app failed. appId = {}, env = {})", appDTO.getAppId(), env, e);
 18:             Tracer.logError(String.format("Create app failed. appId = %s, env = %s", appDTO.getAppId(), env), e);
 19:         }
 20:     }
 21: }
```

* `@EventListener` 注解 + 方法参数，表示 `#onAppCreationEvent(...)` 方法，监听 AppCreationEvent 事件。不了解的胖友，可以看下 [《Spring 4.2框架中注释驱动的事件监听器详解》](https://blog.csdn.net/chszs/article/details/49097919) 文章。
* 第 9 行：调用 `BeanUtils#transfrom(Class<T> clazz, Object src)` 方法，将 App 转换成 `com.ctrip.framework.apollo.common.dto.AppDTO` 对象。`com.ctrip.framework.apollo.common.dto` 包下，提供 Controller 和 Service 层的数据传输。😈 笔者思考了下，Apollo 中，Model 和 DTO 对象很**类似**，差异点在 Model 更侧重 UI 界面提交“**复杂**”业务请求。另外 Apollo 中，还有 VO 对象，侧重 UI 界面返回**复杂**业务响应。整理如下图：![各种 Entity 整理](http://www.iocoder.cn/images/Apollo/2018_03_05/03.png)
    * 老艿艿认为，PO 对象，可以考虑不暴露给 Controller 层，只在 Service 和 Repository 之间传递和返回。
    * 和彩笔老徐交流了下，实际项目可以简化，使用 VO + DTO + PO 。
* 第 11 行：调用 `PortalSettings#getActiveEnvs()`  方法，获得**有效**的 Env 数组，例如 `PROD` `UAT` 等。后续文章，详细分享该方法。
* 第 12 至 20 行：循环 Env 数组，调用 `AppAPI#createApp(Env, AppDTO)` 方法，调用对应的 Admin Service 的 **API** ，创建 App 对象，从而同步 App 到 **Config DB**。

## 3.5 AdminServiceAPI

`com.ctrip.framework.apollo.portal.api.AdminServiceAPI` ，Admin Service API **集合**，包含 Admin Service **所有模块** API 的调用封装。简化代码如下：

![代码](http://www.iocoder.cn/images/Apollo/2018_03_05/04.png)

### 3.5.1 API

`com.ctrip.framework.apollo.portal.api.API` ，API 抽象类。代码如下：

```Java
public abstract class API {

  @Autowired
  protected RetryableRestTemplate restTemplate;

}
```

* 提供统一的 `restTemplate` 的属性注入。对于 RetryableRestTemplate 的源码实现，我们放到后续文章分享。

### 3.5.2 AppAPI 

`com.ctrip.framework.apollo.portal.api.AdminServiceAPI.AppAPI` ，实现 API 抽象类，封装对 Admin Service 的 App 模块的 API 调用。代码如下：

```Java
@Service
public static class AppAPI extends API {

    public AppDTO loadApp(Env env, String appId) {
        return restTemplate.get(env, "apps/{appId}", AppDTO.class, appId);
    }

    public AppDTO createApp(Env env, AppDTO app) {
        return restTemplate.post(env, "apps", app, AppDTO.class);
    }

    public void updateApp(Env env, AppDTO app) {
        restTemplate.put(env, "apps/{appId}", app, app.getAppId());
    }

}
```

* 使用 `restTemplate` ，调用对应的 API 接口。

# 4. Admin Service 侧

## 4.1 AppController

在 `apollo-adminservice` 项目中， `com.ctrip.framework.apollo.adminservice.controller.AppController` ，提供 App 的 **API** 。

`#create(AppDTO)` 方法，创建 App 。代码如下：

```Java
  1: @RestController
  2: public class AppController {
  3: 
  4:     @Autowired
  5:     private AppService appService;
  6:     @Autowired
  7:     private AdminService adminService;
  8: 
  9:     /**
 10:      * 创建 App
 11:      *
 12:      * @param dto AppDTO 对象
 13:      * @return App 对象
 14:      */
 15:     @RequestMapping(path = "/apps", method = RequestMethod.POST)
 16:     public AppDTO create(@RequestBody AppDTO dto) {
 17:         // 校验 appId 格式。若不合法，抛出 BadRequestException 异常
 18:         if (!InputValidator.isValidClusterNamespace(dto.getAppId())) {
 19:             throw new BadRequestException(String.format("AppId格式错误: %s", InputValidator.INVALID_CLUSTER_NAMESPACE_MESSAGE));
 20:         }
 21:         // 将 AppDTO 转换成 App 对象
 22:         App entity = BeanUtils.transfrom(App.class, dto);
 23:         // 判断 `appId` 是否已经存在对应的 App 对象。若已经存在，抛出 BadRequestException 异常。
 24:         App managedEntity = appService.findOne(entity.getAppId());
 25:         if (managedEntity != null) {
 26:             throw new BadRequestException("app already exist.");
 27:         }
 28:         // 保存 App 对象到数据库
 29:         entity = adminService.createNewApp(entity);
 30:         // 将保存的 App 对象，转换成 AppDTO 返回
 31:         dto = BeanUtils.transfrom(AppDTO.class, entity);
 32:         return dto;
 33:     }
 34:     
 35:     // ... 省略其他接口和属性
 36: }   
```

* **POST `apps` 接口**，Request Body 传递 **JSON** 对象。
* 第 17 至 20 行：调用 [`InputValidator#isValidClusterNamespace(appId)`](https://github.com/YunaiV/apollo/blob/e7984de5d6ed8124184f8107e079f9d84462f037/apollo-common/src/main/java/com/ctrip/framework/apollo/common/utils/InputValidator.java#L22-L25) 方法，校验 `appId` 是否满足 `"[0-9a-zA-Z_.-]+"` 格式。若不合法，抛出 BadRequestException 异常。
* 第 22 行：调用 `BeanUtils#transfrom(Class<T> clazz, Object src)` 方法，将 AppDTO 转换成 App对象。
* 第 24 至 27 行：调用 `AppService#findOne(appId)` 方法，判断 `appId` 是否已经存在对应的 App 对象。若已经存在，抛出 BadRequestException 异常。
* 第 29 行：调用 `AdminService#createNewApp(App)` 方法，保存 App 对象到数据库。
* 第 30 至 32 行：调用 `BeanUtils#transfrom(Class<T> clazz, Object src)` 方法，将保存的 App 对象，转换成 AppDTO 返回。

## 4.2 AdminService

`com.ctrip.framework.apollo.biz.service.AdminService` ，😈 无法定义是什么模块的 Service ，目前仅有 `#createNewApp(App)` 方法，代码如下：

```Java
  1: @Service
  2: public class AdminService {
  3: 
  4:     @Autowired
  5:     private AppService appService;
  6:     @Autowired
  7:     private AppNamespaceService appNamespaceService;
  8:     @Autowired
  9:     private ClusterService clusterService;
 10:     @Autowired
 11:     private NamespaceService namespaceService;
 12: 
 13:     @Transactional
 14:     public App createNewApp(App app) {
 15:         // 保存 App 对象到数据库
 16:         String createBy = app.getDataChangeCreatedBy();
 17:         App createdApp = appService.save(app);
 18:         String appId = createdApp.getAppId();
 19:         // 创建 App 的默认命名空间 "application"
 20:         appNamespaceService.createDefaultAppNamespace(appId, createBy);
 21:         // 创建 App 的默认集群 "default"
 22:         clusterService.createDefaultCluster(appId, createBy);
 23:         // 创建 Cluster 的默认命名空间
 24:         namespaceService.instanceOfAppNamespaces(appId, ConfigConsts.CLUSTER_NAME_DEFAULT, createBy);
 25:         return app;
 26:     }
 27: 
 28: }
```

* 第 15 至 18 行：调用 `AppService#save(App)` 方法，保存 App 对象到数据库中。
* 第 20 行：调用 `AppNamespaceService#createDefaultAppNamespace(appId, createBy)` 方法，创建 App 的**默认** Namespace (命名空间) `"application"` 。具体的代码实现，我们在 [《Apollo 源码解析 —— Portal 创建 Namespace》](http://www.iocoder.cn/Apollo/portal-create-namespace/?self) 详细解析。
* ========== 如下部分，是 Admin Service 独有 ==========
* App 下有哪些 Cluster ，在 Portal 中是**不进行保存**，通过 Admin Service API 读取获得。
* 【AppNamespace】第 22 行：调用 `ClusterService#createDefaultCluster(appId, createBy)` 方法，创建 App 的**默认** Cluster `"default"` 。后续文章，详细分享。
* 【Namespace】第 24 行：调用 `NamespaceService#instanceOfAppNamespaces(appId, createBy)` 方法，创建 Cluster 的**默认**命名空间。

## 4.3 AppService

在 `apollo-biz` 项目中，`com.ctrip.framework.apollo.biz.service.AppService` ，提供 App 的 **Service** 逻辑给 Admin Service 和 Config Service 。

`#save(App)` 方法，保存 App 对象到数据库中。代码如下：

```Java
  1: @Autowired
  2: private AppRepository appRepository;
  3: @Autowired
  4: private AuditService auditService;
  5: 
  6: @Transactional
  7: public App save(App entity) {
  8:     // 判断是否已经存在。若是，抛出 ServiceException 异常。
  9:     if (!isAppIdUnique(entity.getAppId())) {
 10:         throw new ServiceException("appId not unique");
 11:     }
 12:     // 保护代码，避免 App 对象中，已经有 id 属性。
 13:     entity.setId(0); // protection
 14:     App app = appRepository.save(entity);
 15:     // 记录 Audit 到数据库中
 16:     auditService.audit(App.class.getSimpleName(), app.getId(), Audit.OP.INSERT, app.getDataChangeCreatedBy());
 17:     return app;
 18: }
```

* 第 8 至 11 行：调用 `#isAppIdUnique(appId)` 方法，判断是否已经存在。若是，抛出 ServiceException 异常。代码如下：

    ```Java
    public boolean isAppIdUnique(String appId) {
        Objects.requireNonNull(appId, "AppId must not be null");
        return Objects.isNull(appRepository.findByAppId(appId));
    }
    ```

* 第 13 行：置“**空**” App 对象，防御性编程，避免 App 对象中，已经有 `id` 属性。
* 第 14 行：调用 `AppRepository#save(App)` 方法，保存 App 对象到数据库中。
* 第 16 行：记录 Audit 到数据库中。

## 4.4 AppRepository

`com.ctrip.framework.apollo.biz.repository.AppRepository` ，继承 `org.springframework.data.repository.PagingAndSortingRepository` 接口，提供 App 的**数据访问** 给 Admin Service 和 Config Service 。代码如下：

```Java
public interface AppRepository extends PagingAndSortingRepository<App, Long> {

  @Query("SELECT a from App a WHERE a.name LIKE %:name%")
  List<App> findByName(@Param("name") String name);

  App findByAppId(String appId);

}
```

# 666. 彩蛋

我们知道，但凡涉及**跨系统**的同步，无可避免会有**事务**的问题，对于 App 创建也会碰到这样的问题，例如：

1. Portal 在同步 App 到 Admin Service 时，发生网络异常，**同步失败**。那么此时会出现该 App 存在于 Portal ，却不存在于 Admin Service 中。
2. 新增了一套环境( Env ) ，也会导致 Portal 和 Admin Service 不一致的情况。

那么 Apollo 是怎么解决这个问题的呢？😈 感兴趣的胖友，可以先自己翻翻源码。嘿嘿。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

