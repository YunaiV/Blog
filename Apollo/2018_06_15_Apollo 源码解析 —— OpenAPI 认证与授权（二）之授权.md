title: Apollo 源码解析 —— OpenAPI 认证与授权（二）之授权
date: 2018-06-15
tags:
categories: Apollo
permalink: Apollo/openapi-auth-2

-------

摘要: 原创出处 http://www.iocoder.cn/Apollo/openapi-auth-2/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Apollo/openapi-auth-2/)
- [2. 权限模型](http://www.iocoder.cn/Apollo/openapi-auth-2/)
  - [2.1 ConsumerRole](http://www.iocoder.cn/Apollo/openapi-auth-2/)
- [3. ConsumerService](http://www.iocoder.cn/Apollo/openapi-auth-2/)
  - [3.1 createConsumerRole](http://www.iocoder.cn/Apollo/openapi-auth-2/)
  - [3.2 assignAppRoleToConsumer](http://www.iocoder.cn/Apollo/openapi-auth-2/)
  - [3.3 assignNamespaceRoleToConsumer](http://www.iocoder.cn/Apollo/openapi-auth-2/)
- [4. ConsumerRolePermissionService](http://www.iocoder.cn/Apollo/openapi-auth-2/)
- [5. ConsumerPermissionValidator](http://www.iocoder.cn/Apollo/openapi-auth-2/)
- [6. ConsumerController](http://www.iocoder.cn/Apollo/openapi-auth-2/)
- [666. 彩蛋](http://www.iocoder.cn/Apollo/openapi-auth-2/)

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

> 老艿艿：本系列假定胖友已经阅读过 [《Apollo 官方 wiki 文档》](https://github.com/ctripcorp/apollo/wiki/) ，特别是 [《Apollo 开放平台》](https://github.com/ctripcorp/apollo/wiki/Apollo%E5%BC%80%E6%94%BE%E5%B9%B3%E5%8F%B0) 。

本文接 [《Apollo 源码解析 —— OpenAPI 认证与授权（一）之认证》](http://www.iocoder.cn/Apollo/openapi-auth-1/?self) ，**侧重在授权部分**。和 Portal 的**授权**一样：

> 具体**每个** URL 的权限校验，通过在对应的方法上，添加 `@PreAuthorize` 方法注解，配合具体的方法参数，一起校验**功能 + 数据级**的权限校验。

# 2. 权限模型

和 Portal 使用**相同**的权限模型，**差别**在于 UserRole 换成了 ConsumerRole 。所以，关系如下图：![关系](http://www.iocoder.cn/images/Apollo/2018_06_15/01.png)

## 2.1 ConsumerRole

**ConsumerRole** 表，Consumer 与角色的**关联**表，对应实体 `com.ctrip.framework.apollo.openapi.entity.ConsumerRole` ，代码如下：

```Java
@Entity
@Table(name = "ConsumerRole")
@SQLDelete(sql = "Update ConsumerRole set isDeleted = 1 where id = ?")
@Where(clause = "isDeleted = 0")
public class ConsumerRole extends BaseEntity {

    /**
     * Consumer 编号 {@link Consumer#id}
     */
    @Column(name = "ConsumerId", nullable = false)
    private long consumerId;
    /**
     * Role 编号 {@link com.ctrip.framework.apollo.portal.entity.po.Role#id}
     */
    @Column(name = "RoleId", nullable = false)
    private long roleId;
}
```

* 字段比较简单，胖友自己看注释。

# 3. ConsumerService

`com.ctrip.framework.apollo.openapi.service.ConsumerService` ，提供 Consumer、ConsumerToken、ConsumerAudit、ConsumerRole 相关的 **Service** 逻辑。

## 3.1 createConsumerRole

`#createConsumerRole(consumerId, roleId, operator)` 方法，创建 Consumer 对象。代码如下：

```Java
ConsumerRole createConsumerRole(Long consumerId, Long roleId, String operator) {
    ConsumerRole consumerRole = new ConsumerRole();
    consumerRole.setConsumerId(consumerId);
    consumerRole.setRoleId(roleId);
    consumerRole.setDataChangeCreatedBy(operator);
    consumerRole.setDataChangeLastModifiedBy(operator);
    return consumerRole;
}
```

## 3.2 assignAppRoleToConsumer

`#assignAppRoleToConsumer(token, appId)` 方法，授权 App 的 Role 给 Consumer 。代码如下：

```Java
@Transactional
public ConsumerRole assignAppRoleToConsumer(String token, String appId) {
    // 校验 Token 是否有对应的 Consumer 。若不存在，抛出 BadRequestException 异常
    Long consumerId = getConsumerIdByToken(token);
    if (consumerId == null) {
        throw new BadRequestException("Token is Illegal");
    }

    // 获得 App 对应的 Role 对象
    Role masterRole = rolePermissionService.findRoleByRoleName(RoleUtils.buildAppMasterRoleName(appId));
    if (masterRole == null) {
        throw new BadRequestException("App's role does not exist. Please check whether app has created.");
    }

    // 获得 Consumer 对应的 ConsumerRole 对象。若已存在，返回 ConsumerRole 对象
    long roleId = masterRole.getId();
    ConsumerRole managedModifyRole = consumerRoleRepository.findByConsumerIdAndRoleId(consumerId, roleId);
    if (managedModifyRole != null) {
        return managedModifyRole;
    }

    // 创建 Consumer 对应的 ConsumerRole 对象
    String operator = userInfoHolder.getUser().getUserId();
    ConsumerRole consumerRole = createConsumerRole(consumerId, roleId, operator);
    // 保存 Consumer 对应的 ConsumerRole 对象
    return consumerRoleRepository.save(consumerRole);
}
```

## 3.3 assignNamespaceRoleToConsumer

`#assignNamespaceRoleToConsumer(token, appId, namespaceName)` 方法，授权 Namespace 的 Role 给 Consumer 。对吗如下：

```Java
@Transactional
public List<ConsumerRole> assignNamespaceRoleToConsumer(String token, String appId, String namespaceName) {
    // 校验 Token 是否有对应的 Consumer 。若不存在，抛出 BadRequestException 异常
    Long consumerId = getConsumerIdByToken(token);
    if (consumerId == null) {
        throw new BadRequestException("Token is Illegal");
    }

    // 获得 Namespace 对应的 Role 们。若有任一不存在，抛出 BadRequestException 异常
    Role namespaceModifyRole = rolePermissionService.findRoleByRoleName(RoleUtils.buildModifyNamespaceRoleName(appId, namespaceName));
    Role namespaceReleaseRole = rolePermissionService.findRoleByRoleName(RoleUtils.buildReleaseNamespaceRoleName(appId, namespaceName));
    if (namespaceModifyRole == null || namespaceReleaseRole == null) {
        throw new BadRequestException("Namespace's role does not exist. Please check whether namespace has created.");
    }
    long namespaceModifyRoleId = namespaceModifyRole.getId();
    long namespaceReleaseRoleId = namespaceReleaseRole.getId();

    // 获得 Consumer 对应的 ConsumerRole 们。若都存在，返回 ConsumerRole 数组
    ConsumerRole managedModifyRole = consumerRoleRepository.findByConsumerIdAndRoleId(consumerId, namespaceModifyRoleId);
    ConsumerRole managedReleaseRole = consumerRoleRepository.findByConsumerIdAndRoleId(consumerId, namespaceReleaseRoleId);
    if (managedModifyRole != null && managedReleaseRole != null) {
        return Arrays.asList(managedModifyRole, managedReleaseRole);
    }

    // 创建 Consumer 对应的 ConsumerRole 们
    String operator = userInfoHolder.getUser().getUserId();
    ConsumerRole namespaceModifyConsumerRole = createConsumerRole(consumerId, namespaceModifyRoleId, operator);
    ConsumerRole namespaceReleaseConsumerRole = createConsumerRole(consumerId, namespaceReleaseRoleId, operator);
    // 保存 Consumer 对应的 ConsumerRole 们到数据库中
    ConsumerRole createdModifyConsumerRole = consumerRoleRepository.save(namespaceModifyConsumerRole);
    ConsumerRole createdReleaseConsumerRole = consumerRoleRepository.save(namespaceReleaseConsumerRole);
    // 返回 ConsumerRole 数组
    return Arrays.asList(createdModifyConsumerRole, createdReleaseConsumerRole);
}
```

# 4. ConsumerRolePermissionService

`com.ctrip.framework.apollo.openapi.service.ConsumerRolePermissionService` ，ConsumerRole 权限**校验** Service 。代码如下：

```Java
@Service
public class ConsumerRolePermissionService {

    @Autowired
    private PermissionRepository permissionRepository;
    @Autowired
    private ConsumerRoleRepository consumerRoleRepository;
    @Autowired
    private RolePermissionRepository rolePermissionRepository;

    /**
     * Check whether user has the permission
     */
    public boolean consumerHasPermission(long consumerId, String permissionType, String targetId) {
        // 获得 Permission 对象
        Permission permission = permissionRepository.findTopByPermissionTypeAndTargetId(permissionType, targetId);
        // 若 Permission 不存在，返回 false
        if (permission == null) {
            return false;
        }

        // 获得 ConsumerRole 数组
        List<ConsumerRole> consumerRoles = consumerRoleRepository.findByConsumerId(consumerId);
        // 若数组为空，返回 false
        if (CollectionUtils.isEmpty(consumerRoles)) {
            return false;
        }

        // 获得 RolePermission 数组
        Set<Long> roleIds = consumerRoles.stream().map(ConsumerRole::getRoleId).collect(Collectors.toSet());
        List<RolePermission> rolePermissions = rolePermissionRepository.findByRoleIdIn(roleIds);
        // 若数组为空，返回 false
        if (CollectionUtils.isEmpty(rolePermissions)) {
            return false;
        }

        // 判断是否有对应的 RolePermission 。若有，则返回 true 【有权限】
        for (RolePermission rolePermission : rolePermissions) {
            if (rolePermission.getPermissionId() == permission.getId()) {
                return true;
            }
        }
        
        return false;
    }

}
```

* 和 `DefaultRolePermissionService#userHasPermission(userId, permissionType, targetId)` 方法，**基本类似**。

# 5. ConsumerPermissionValidator

> ConsumerPermissionValidator 和 PermissionValidator **基本类似**。

`com.ctrip.framework.apollo.openapi.auth.ConsumerPermissionValidator` ，Consumer 权限校验器。代码如下：

```Java
@Component
public class ConsumerPermissionValidator {

    @Autowired
    private ConsumerRolePermissionService permissionService;
    @Autowired
    private ConsumerAuthUtil consumerAuthUtil;

    // ========== Namespace 级别 ==========

    public boolean hasModifyNamespacePermission(HttpServletRequest request, String appId, String namespaceName) {
        if (hasCreateNamespacePermission(request, appId)) {
            return true;
        }
        return permissionService.consumerHasPermission(consumerAuthUtil.retrieveConsumerId(request),
                PermissionType.MODIFY_NAMESPACE,
                RoleUtils.buildNamespaceTargetId(appId, namespaceName));

    }

    public boolean hasReleaseNamespacePermission(HttpServletRequest request, String appId, String namespaceName) {
        if (hasCreateNamespacePermission(request, appId)) {
            return true;
        }
        return permissionService.consumerHasPermission(consumerAuthUtil.retrieveConsumerId(request),
                PermissionType.RELEASE_NAMESPACE,
                RoleUtils.buildNamespaceTargetId(appId, namespaceName));

    }

    // ========== App 级别 ==========

    public boolean hasCreateNamespacePermission(HttpServletRequest request, String appId) {
        return permissionService.consumerHasPermission(consumerAuthUtil.retrieveConsumerId(request),
                PermissionType.CREATE_NAMESPACE,
                appId);
    }

}
```

在每个需要校验权限的方法上，添加 `@PreAuthorize` 注解，并在 `value` 属性上写 EL 表达式，调用 PermissionValidator 的校验方法。例如：

* 创建 Namespace 的方法，添加了 `@PreAuthorize(value = "@consumerPermissionValidator.hasCreateNamespacePermission(#request, #appId)")` 。
* 发布 Namespace 的方法，添加了 `@PreAuthorize(value = "@consumerPermissionValidator.hasReleaseNamespacePermission(#request, #appId, #namespaceName)")` 。

通过这样的方式，达到**功能 + 数据级**的权限控制。

# 6. ConsumerController

在 `apollo-portal` 项目中，`com.ctrip.framework.apollo.portal.controller.ConsumerController` ，提供 Consumer、ConsumerToken、ConsumerAudit 相关的 **API** 。

在**创建第三方应用**的界面中，点击【提交】按钮，调用**授权 Consumer 的 API** 。

![创建第三方应用](http://www.iocoder.cn/images/Apollo/2018_06_15/02.png)

代码如下：

```Java
@PreAuthorize(value = "@permissionValidator.isSuperAdmin()")
@RequestMapping(value = "/consumers/{token}/assign-role", method = RequestMethod.POST)
public List<ConsumerRole> assignNamespaceRoleToConsumer(@PathVariable String token, @RequestParam String type, @RequestBody NamespaceDTO namespace) {
    String appId = namespace.getAppId();
    String namespaceName = namespace.getNamespaceName();
    // 校验 appId 非空。若为空，抛出 BadRequestException 异常
    if (StringUtils.isEmpty(appId)) {
        throw new BadRequestException("Params(AppId) can not be empty.");
    }

    // 授权 App 的 Role 给 Consumer
    if (Objects.equals("AppRole", type)) {
        return Collections.singletonList(consumerService.assignAppRoleToConsumer(token, appId));
    // 授权 Namespace 的 Role 给 Consumer
    } else {
        if (StringUtils.isEmpty(namespaceName)) {
            throw new BadRequestException("Params(NamespaceName) can not be empty.");
        }
        return consumerService.assignNamespaceRoleToConsumer(token, appId, namespaceName);
    }
}
```

# 666. 彩蛋

OpenAPI 在 `v1/controller` 中，实现了自己的 API ，**共享**调用 Portal 中的 Service 。如下图所示：![OpenAPI Controller](http://www.iocoder.cn/images/Apollo/2018_06_15/03.png)

* 🙂 具体的 Controller 实现，胖友自己查看噢，笔者就不分享啦。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)


