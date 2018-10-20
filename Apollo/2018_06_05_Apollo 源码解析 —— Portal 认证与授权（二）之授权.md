title: Apollo 源码解析 —— Portal 认证与授权（二）之授权
date: 2018-06-05
tags:
categories: Apollo
permalink: Apollo/portal-auth-2

-------

摘要: 原创出处 http://www.iocoder.cn/Apollo/portal-auth-2/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Apollo/portal-auth-2/)
- [2. 权限模型](http://www.iocoder.cn/Apollo/portal-auth-2/)
  - [2.1 Role](http://www.iocoder.cn/Apollo/portal-auth-2/)
  - [2.2 UserRole](http://www.iocoder.cn/Apollo/portal-auth-2/)
  - [2.3 Permission](http://www.iocoder.cn/Apollo/portal-auth-2/)
  - [2.4 RolePermission](http://www.iocoder.cn/Apollo/portal-auth-2/)
- [3. RolePermissionService](http://www.iocoder.cn/Apollo/portal-auth-2/)
  - [3.1 DefaultRolePermissionService](http://www.iocoder.cn/Apollo/portal-auth-2/)
- [4. RoleInitializationService](http://www.iocoder.cn/Apollo/portal-auth-2/)
  - [4.1 DefaultRoleInitializationService](http://www.iocoder.cn/Apollo/portal-auth-2/)
- [5. PermissionValidator](http://www.iocoder.cn/Apollo/portal-auth-2/)
- [6. PermissionController](http://www.iocoder.cn/Apollo/portal-auth-2/)
- [666. 彩蛋](http://www.iocoder.cn/Apollo/portal-auth-2/)

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

> 老艿艿：本系列假定胖友已经阅读过 [《Apollo 官方 wiki 文档》](https://github.com/ctripcorp/apollo/wiki/) ，特别是 [《Portal 实现用户登录功能》](https://github.com/ctripcorp/apollo/wiki/Portal-%E5%AE%9E%E7%8E%B0%E7%94%A8%E6%88%B7%E7%99%BB%E5%BD%95%E5%8A%9F%E8%83%BD) 。

本文接 [《Apollo 源码解析 —— Portal 认证与授权（一）之认证》](http://www.iocoder.cn/Apollo/portal-auth-1/?self) ，**侧重在授权部分**。在上一文中，我们提到：

> 具体**每个** URL 的权限校验，通过在对应的方法上，添加 `@PreAuthorize` 方法注解，配合具体的方法参数，一起校验**功能 + 数据级**的权限校验。

# 2. 权限模型

常见的权限模型，有两种：RBAC 和 ACL 。如果不了解的胖友，可以看下 [《基于AOP实现权限管理：访问控制模型 RBAC 和 ACL 》](https://blog.csdn.net/tch918/article/details/18449043) 。

笔者一开始看到 Role + UserRole + Permission + RolePermission 四张表，认为是 **RBAC** 权限模型。但是看了 Permission 的数据结构，以及 PermissionValidator 的权限判断方式，又感受到几分 **ACL** 权限模型的味道。

所以，很难完全说，Apollo  属于 RBAC 还是 ACL 权限模型。或者说，权限模型，本身会根据实际业务场景的业务需要，做一些变种和改造。权限模型，提供给我们的是指导和借鉴，不需要过于拘泥。

关系如下图：![关系](http://www.iocoder.cn/images/Apollo/2018_06_05/01.png)

## 2.1 Role

**Role** 表，角色表，对应实体 `com.ctrip.framework.apollo.portal.entity.po.Role` ，代码如下：

```Java
@Entity
@Table(name = "Role")
@SQLDelete(sql = "Update Role set isDeleted = 1 where id = ?")
@Where(clause = "isDeleted = 0")
public class Role extends BaseEntity {

    /**
     * 角色名
     */
    @Column(name = "RoleName", nullable = false)
    private String roleName;

}
```

* `roleName` 字段，**角色名**，通过系统**自动生成**。目前有**三种类型**( 不是三个 )角色：
    * App 管理员，格式为 `"Master + AppId"` ，例如：`"Master+100004458"` 。
    * Namespace **修改**管理员，格式为 `"ModifyNamespace + AppId + NamespaceName"` ，例如：`"ModifyNamespace+100004458+application"` 。
    * Namespace **发布**管理员，格式为 `"ReleaseNamespace + AppId + NamespaceName"` ，例如：`"ReleaseNamespace+100004458+application"` 。
* 例子如下图：![例子](http://www.iocoder.cn/images/Apollo/2018_06_05/02.png)

## 2.2 UserRole

**UserRole** 表，用户与角色的**关联**表，对应实体 `com.ctrip.framework.apollo.portal.entity.po.UserRole` ，代码如下：

```Java
@Entity
@Table(name = "UserRole")
@SQLDelete(sql = "Update UserRole set isDeleted = 1 where id = ?")
@Where(clause = "isDeleted = 0")
public class UserRole extends BaseEntity {

    /**
     * 账号 {@link UserPO#username}
     */
    @Column(name = "UserId", nullable = false)
    private String userId;
    /**
     * 角色编号 {@link Role#id}
     */
    @Column(name = "RoleId", nullable = false)
    private long roleId;

}
```

* `userId` 字段，用户编号，指向对应的 User 。目前使用 `UserPO.username` 。当然，我们自己的业务系统里，推荐使用 `UserPO.id` 。
* `roleId` 字段，角色编号，指向对应的 Role 。
* 例子如下图：![例子](http://www.iocoder.cn/images/Apollo/2018_06_05/03.png)

## 2.3 Permission

**Permission** 表，权限表，对应实体 `com.ctrip.framework.apollo.portal.entity.po.Permission` ，代码如下：

```Java
@Entity
@Table(name = "Permission")
@SQLDelete(sql = "Update Permission set isDeleted = 1 where id = ?")
@Where(clause = "isDeleted = 0")
public class Permission extends BaseEntity {

    /**
     * 权限类型
     */
    @Column(name = "PermissionType", nullable = false)
    private String permissionType;
    /**
     * 目标编号
     */
    @Column(name = "TargetId", nullable = false)
    private String targetId;
    
}
```

* 
* `permissionType` 字段，权限类型。在 `com.ctrip.framework.apollo.portal.constant.PermissionType` 中枚举，代码如下：

    ```Java
    public interface PermissionType {
    
        // ========== APP level permission ==========
        String CREATE_NAMESPACE = "CreateNamespace"; // 创建 Namespace
        String CREATE_CLUSTER = "CreateCluster"; // 创建 Cluster
        String ASSIGN_ROLE = "AssignRole"; // 分配用户权限的权限
    
        // ========== namespace level permission =========
        String MODIFY_NAMESPACE = "ModifyNamespace"; // 修改 Namespace
        String RELEASE_NAMESPACE = "ReleaseNamespace"; // 发布 Namespace
    
    }
    ```
    * 分成 App 和 Namespace **两种**级别的权限类型。
* `targetId` 字段，目标编号。
* 例子如下图：![例子](http://www.iocoder.cn/images/Apollo/2018_06_05/04.png)
    * **App** 级别时，`targetId` 指向 "**App 编号**"。
    * **Namespace** 级别时，`targetId` 指向 "**App 编号** + **Namespace 名字**"。
        * **为什么**不是 Namespace 的编号？ **Namespace** 级别，是所有环境 + 所有集群都有权限，所以不能具体某个 Namespace 。

## 2.4 RolePermission

**RolePermission** 表，角色与权限的**关联**表，对应实体 `com.ctrip.framework.apollo.portal.entity.po.RolePermission` ，代码如下：

```Java
@Entity
@Table(name = "RolePermission")
@SQLDelete(sql = "Update RolePermission set isDeleted = 1 where id = ?")
@Where(clause = "isDeleted = 0")
public class RolePermission extends BaseEntity {

    /**
     * 角色编号 {@link Role#id}
     */
    @Column(name = "RoleId", nullable = false)
    private long roleId;
    /**
     * 权限编号 {@link Permission#id}
     */
    @Column(name = "PermissionId", nullable = false)
    private long permissionId;

}
```

* `roleId` 字段，角色编号，指向对应的 Role 。
* `permissionId` 字段，权限编号，指向对应的 Permission 。
* 例子如下图：![例子](http://www.iocoder.cn/images/Apollo/2018_06_05/05.png)

# 3. RolePermissionService

`com.ctrip.framework.apollo.portal.service.RolePermissionService` ，提供 Role、UserRole、Permission、UserPermission 相关的操作。代码如下：

```Java
public interface RolePermissionService {

    // ========== Role 相关 ==========
    /**
     * Create role with permissions, note that role name should be unique
     */
    Role createRoleWithPermissions(Role role, Set<Long> permissionIds);
    /**
     * Find role by role name, note that roleName should be unique
     */
    Role findRoleByRoleName(String roleName);

    // ========== UserRole 相关 ==========
    /**
     * Assign role to users
     *
     * @return the users assigned roles
     */
    Set<String> assignRoleToUsers(String roleName, Set<String> userIds, String operatorUserId);
    /**
     * Remove role from users
     */
    void removeRoleFromUsers(String roleName, Set<String> userIds, String operatorUserId);
    /**
     * Query users with role
     */
    Set<UserInfo> queryUsersWithRole(String roleName);

    // ========== UserPermission 相关 ==========
    /**
     * Check whether user has the permission
     */
    boolean userHasPermission(String userId, String permissionType, String targetId);
    /**
     * 校验是否为超级管理员
     */
    boolean isSuperAdmin(String userId);

    // ========== Permission 相关 ==========
    /**
     * Create permission, note that permissionType + targetId should be unique
     */
    Permission createPermission(Permission permission);
    /**
     * Create permissions, note that permissionType + targetId should be unique
     */
    Set<Permission> createPermissions(Set<Permission> permissions);

}
```

## 3.1 DefaultRolePermissionService

`com.ctrip.framework.apollo.portal.spi.defaultimpl.DefaultRolePermissionService` ，实现 RolePermissionService 接口，默认 RolePermissionService 实现类。

> 老艿艿：下面的方法比较易懂，胖友看着代码注释理解。

### 3.1.1 createRoleWithPermissions

```Java
@Override
@Transactional
public Role createRoleWithPermissions(Role role, Set<Long> permissionIds) {
    // 获得 Role 对象，校验 Role 不存在
    Role current = findRoleByRoleName(role.getRoleName());
    Preconditions.checkState(current == null, "Role %s already exists!", role.getRoleName());

    // 新增 Role
    Role createdRole = roleRepository.save(role);

    // 授权给 Role
    if (!CollectionUtils.isEmpty(permissionIds)) {
        // 创建 RolePermission 数组
        Iterable<RolePermission> rolePermissions = permissionIds.stream().map(permissionId -> {
            RolePermission rolePermission = new RolePermission();
            rolePermission.setRoleId(createdRole.getId()); // Role 编号
            rolePermission.setPermissionId(permissionId);
            rolePermission.setDataChangeCreatedBy(createdRole.getDataChangeCreatedBy());
            rolePermission.setDataChangeLastModifiedBy(createdRole.getDataChangeLastModifiedBy());
            return rolePermission;
        }).collect(Collectors.toList());
        // 保存 RolePermission 数组
        rolePermissionRepository.save(rolePermissions);
    }

    return createdRole;
}
```

### 3.1.2 assignRoleToUsers

```Java
@Override
@Transactional
public Set<String> assignRoleToUsers(String roleName, Set<String> userIds, String operatorUserId) {
    // 获得 Role 对象，校验 Role 存在
    Role role = findRoleByRoleName(roleName);
    Preconditions.checkState(role != null, "Role %s doesn't exist!", roleName);

    // 获得已存在的 UserRole 数组
    List<UserRole> existedUserRoles = userRoleRepository.findByUserIdInAndRoleId(userIds, role.getId());
    Set<String> existedUserIds = existedUserRoles.stream().map(UserRole::getUserId).collect(Collectors.toSet());
    // 排除已经存在的
    Set<String> toAssignUserIds = Sets.difference(userIds, existedUserIds);

    // 创建需要新增的 UserRole 数组
    Iterable<UserRole> toCreate = toAssignUserIds.stream().map(userId -> {
        UserRole userRole = new UserRole();
        userRole.setRoleId(role.getId());
        userRole.setUserId(userId);
        userRole.setDataChangeCreatedBy(operatorUserId);
        userRole.setDataChangeLastModifiedBy(operatorUserId);
        return userRole;
    }).collect(Collectors.toList());
    // 保存 RolePermission 数组
    userRoleRepository.save(toCreate);

    return toAssignUserIds;
}
```

### 3.1.3 removeRoleFromUsers

```Java
@Override
@Transactional
public void removeRoleFromUsers(String roleName, Set<String> userIds, String operatorUserId) {
    // 获得 Role 对象，校验 Role 存在
    Role role = findRoleByRoleName(roleName);
    Preconditions.checkState(role != null, "Role %s doesn't exist!", roleName);

    // 获得已存在的 UserRole 数组
    List<UserRole> existedUserRoles = userRoleRepository.findByUserIdInAndRoleId(userIds, role.getId());
    // 标记删除
    for (UserRole userRole : existedUserRoles) {
        userRole.setDeleted(true); // 标记删除
        userRole.setDataChangeLastModifiedTime(new Date());
        userRole.setDataChangeLastModifiedBy(operatorUserId);
    }

    // 保存 RolePermission 数组 【标记删除】
    userRoleRepository.save(existedUserRoles);
}
```

### 3.1.4 queryUsersWithRole

```Java
@Override
public Set<UserInfo> queryUsersWithRole(String roleName) {
    // 获得 Role 对象，校验 Role 存在
    Role role = findRoleByRoleName(roleName);

    // Role 不存在时，返回空数组
    if (role == null) {
        return Collections.emptySet();
    }

    // 获得 UserRole 数组
    List<UserRole> userRoles = userRoleRepository.findByRoleId(role.getId());
    // 转换成 UserInfo 数组
    return userRoles.stream().map(userRole -> {
        UserInfo userInfo = new UserInfo();
        userInfo.setUserId(userRole.getUserId());
        return userInfo;
    }).collect(Collectors.toSet());
}
```

### 3.1.5 findRoleByRoleName

```Java
public Role findRoleByRoleName(String roleName) {
    return roleRepository.findTopByRoleName(roleName);
}
```

### 3.1.6 userHasPermission 【重要】

```Java
@Override
public boolean userHasPermission(String userId, String permissionType, String targetId) {
    // 获得 Permission 对象
    Permission permission = permissionRepository.findTopByPermissionTypeAndTargetId(permissionType, targetId);
    // 若 Permission 不存在，返回 false
    if (permission == null) {
        return false;
    }

    // 若是超级管理员，返回 true 【有权限】
    if (isSuperAdmin(userId)) {
        return true;
    }

    // 获得 UserRole 数组
    List<UserRole> userRoles = userRoleRepository.findByUserId(userId);
    // 若数组为空，返回 false
    if (CollectionUtils.isEmpty(userRoles)) {
        return false;
    }

    // 获得 RolePermission 数组
    Set<Long> roleIds = userRoles.stream().map(UserRole::getRoleId).collect(Collectors.toSet());
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
```

* 从目前的代码看下来，这个权限判断的过程，是 **ACL** 的方式。
* 如果是 **RBAC** 的方式，获得 Permission 后，再获得 Permission 对应的 RolePermission 数组，最后和 User 对应的 UserRole 数组，求 `roleId` 是否相交。

### 3.1.7 isSuperAdmin

```Java
@Override
public boolean isSuperAdmin(String userId) {
    return portalConfig.superAdmins().contains(userId);
}
```

* 通过 ServerConfig 的 `"superAdmin"` 配置项，判断是否存在该账号。

### 3.1.8 createPermissions

```Java
@Override
@Transactional
public Permission createPermission(Permission permission) {
    String permissionType = permission.getPermissionType();
    String targetId = permission.getTargetId();
    // 获得 Permission 对象，校验 Permission 为空
    Permission current = permissionRepository.findTopByPermissionTypeAndTargetId(permissionType, targetId);
    Preconditions.checkState(current == null, "Permission with permissionType %s targetId %s already exists!", permissionType, targetId);

    // 保存 Permission
    return permissionRepository.save(permission);
}
```

### 3.1.8 createPermissions

```Java
@Override
@Transactional
public Set<Permission> createPermissions(Set<Permission> permissions) {
    // 创建 Multimap 对象，用于下面校验的分批的批量查询
    Multimap<String, String> targetIdPermissionTypes = HashMultimap.create();
    for (Permission permission : permissions) {
        targetIdPermissionTypes.put(permission.getTargetId(), permission.getPermissionType());
    }

    // 查询 Permission 集合，校验都不存在
    for (String targetId : targetIdPermissionTypes.keySet()) {
        Collection<String> permissionTypes = targetIdPermissionTypes.get(targetId);
        List<Permission> current = permissionRepository.findByPermissionTypeInAndTargetId(permissionTypes, targetId);
        Preconditions.checkState(CollectionUtils.isEmpty(current), "Permission with permissionType %s targetId %s already exists!", permissionTypes, targetId);
    }

    // 保存 Permission 集合
    Iterable<Permission> results = permissionRepository.save(permissions);
    // 转成 Permission 集合，返回
    return StreamSupport.stream(results.spliterator(), false).collect(Collectors.toSet());
}
```

# 4. RoleInitializationService

`com.ctrip.framework.apollo.portal.service.RoleInitializationService` ，提供角色初始化相关的操作。代码如下：

```Java
public interface RoleInitializationService {

    /**
     * 初始化 App 级的 Role
     */
    void initAppRoles(App app);

    /**
     * 初始化 Namespace 级的 Role
     */
    void initNamespaceRoles(String appId, String namespaceName, String operator);

}
```

## 4.1 DefaultRoleInitializationService

`com.ctrip.framework.apollo.portal.spi.defaultimpl.DefaultRoleInitializationService` ，实现 RoleInitializationService 接口，默认 RoleInitializationService 实现类。

### 4.1.1 initAppRoles

```Java
  1: @Override
  2: @Transactional
  3: public void initAppRoles(App app) {
  4:     String appId = app.getAppId();
  5: 
  6:     // 创建 App 拥有者的角色名
  7:     String appMasterRoleName = RoleUtils.buildAppMasterRoleName(appId);
  8:     // has created before
  9:     // 校验角色是否已经存在。若是，直接返回
 10:     if (rolePermissionService.findRoleByRoleName(appMasterRoleName) != null) {
 11:         return;
 12:     }
 13:     String operator = app.getDataChangeCreatedBy();
 14:     //create app permissions
 15:     // 创建 App 角色
 16:     createAppMasterRole(appId, operator);
 17:     // 授权 Role 给 App 拥有者
 18:     // assign master role to user
 19:     rolePermissionService.assignRoleToUsers(RoleUtils.buildAppMasterRoleName(appId), Sets.newHashSet(app.getOwnerName()), operator);
 20: 
 21:     // 初始化 Namespace 角色
 22:     initNamespaceRoles(appId, ConfigConsts.NAMESPACE_APPLICATION, operator);
 23:     // 授权 Role 给 App 创建者
 24:     //assign modify、release namespace role to user
 25:     rolePermissionService.assignRoleToUsers(RoleUtils.buildNamespaceRoleName(appId, ConfigConsts.NAMESPACE_APPLICATION, RoleType.MODIFY_NAMESPACE), Sets.newHashSet(operator), operator);
 26:     rolePermissionService.assignRoleToUsers(RoleUtils.buildNamespaceRoleName(appId, ConfigConsts.NAMESPACE_APPLICATION, RoleType.RELEASE_NAMESPACE), Sets.newHashSet(operator), operator);
 27: }
```

* 在 Portal 创建完**本地** App 后，自动初始化对应的 Role 们。调用如下图：![createLocalApp](http://www.iocoder.cn/images/Apollo/2018_06_05/06.png)
* =========== 初始化 App 级的 Role ===========
* 第 7 行：调用 `RoleUtils#buildAppMasterRoleName(appId)` 方法，创建 App **拥有者**的角色名。代码如下：

    ```Java
    // RoleUtils.java
    private static final Joiner STRING_JOINER = Joiner.on(ConfigConsts.CLUSTER_NAMESPACE_SEPARATOR);
    
    public static String buildAppMasterRoleName(String appId) {
        return STRING_JOINER.join(RoleType.MASTER, appId);
    }
    
    // RoleType.java
    public static final String MASTER = "Master";
    ```

* 第 9 至 12 行：调用 `RolePermissionService#findRoleByRoleName(appMasterRoleName)` 方法，**校验**角色是否已经存在。若是，直接返回。
* 第 16 行：调用 `#createAppMasterRole(appId, operator)` 方法，创建 App **拥有者**角色。代码如下：

    ```Java
    private void createAppMasterRole(String appId, String operator) {
        // 创建 App 对应的 Permission 集合，并保存到数据库
        Set<Permission> appPermissions = Lists.newArrayList(PermissionType.CREATE_CLUSTER, PermissionType.CREATE_NAMESPACE, PermissionType.ASSIGN_ROLE)
                .stream().map(permissionType -> createPermission(appId, permissionType, operator) /* 创建 Permission 对象 */ ).collect(Collectors.toSet());
        Set<Permission> createdAppPermissions = rolePermissionService.createPermissions(appPermissions);
        Set<Long> appPermissionIds = createdAppPermissions.stream().map(BaseEntity::getId).collect(Collectors.toSet());
    
        // 创建 App 对应的 Role 对象，并保存到数据库
        // create app master role
        Role appMasterRole = createRole(RoleUtils.buildAppMasterRoleName(appId), operator);
        rolePermissionService.createRoleWithPermissions(appMasterRole, appPermissionIds);
    }
    ```
    * 创建并保存 App 对应的 Permission 集合。`#createPermission(targetId, permissionType, operator)` 方法，创建 Permission 对象。代码如下：

        ```Java
        private Permission createPermission(String targetId, String permissionType, String operator) {
            Permission permission = new Permission();
            permission.setPermissionType(permissionType);
            permission.setTargetId(targetId);
            permission.setDataChangeCreatedBy(operator);
            permission.setDataChangeLastModifiedBy(operator);
            return permission;
        }
        ```
        * x
    
    * 创建并保存 App 对应的 Role 对象，并授权对应的 Permission 集合。`#createRole(roleName, operator)` 方法，创建 Role 对象。代码如下：

        ```Java
        private Role createRole(String roleName, String operator) {
            Role role = new Role();
            role.setRoleName(roleName);
            role.setDataChangeCreatedBy(operator);
            role.setDataChangeLastModifiedBy(operator);
            return role;
        }
        ```
        * x

* 第 19 行：调用 `rolePermissionService.assignRoleToUsers(roleName, userIds, operatorUserId)` 方法，授权 Role 给 App **拥有者**。
* =========== 初始化 Namespace 级的 Role ===========
* 第 22 行：调用 `#initNamespaceRoles(appId, namespaceName, operator)` 方法，初始化 Namespace 的角色。详细解析，见 [「4.2 initNamespaceRoles」](#) 。
* 第 23 至 26 行：调用 `rolePermissionService.assignRoleToUsers(roleName, userIds, operatorUserId)` 方法，授权 Role 给 App **创建者**。**注意**，此处不是“拥有者”噢。为什么？因为，Namespace 是自动创建的，并且是通过**创建人**来操作的。

### 4.1.2 initNamespaceRoles

```Java
@Override
@Transactional
public void initNamespaceRoles(String appId, String namespaceName, String operator) {
    // 创建 Namespace 修改的角色名
    String modifyNamespaceRoleName = RoleUtils.buildModifyNamespaceRoleName(appId, namespaceName);
    // 若不存在对应的 Role ，进行创建
    if (rolePermissionService.findRoleByRoleName(modifyNamespaceRoleName) == null) {
        createNamespaceRole(appId, namespaceName, PermissionType.MODIFY_NAMESPACE, RoleUtils.buildModifyNamespaceRoleName(appId, namespaceName), operator);
    }

    // 创建 Namespace 发布的角色名
    String releaseNamespaceRoleName = RoleUtils.buildReleaseNamespaceRoleName(appId, namespaceName);
    // 若不存在对应的 Role ，进行创建
    if (rolePermissionService.findRoleByRoleName(releaseNamespaceRoleName) == null) {
        createNamespaceRole(appId, namespaceName, PermissionType.RELEASE_NAMESPACE,
                RoleUtils.buildReleaseNamespaceRoleName(appId, namespaceName), operator);
    }
}
```

* 在 Portal 创建完 Namespace 后，自动初始化对应的 Role 们。调用如下图：![调用方](http://www.iocoder.cn/images/Apollo/2018_06_05/07.png)
* 创建并保存 Namespace **修改**和**发布**对应的 Role 。
* `RoleUtils#buildModifyNamespaceRoleName(appId, namespaceName)` 方法，创建 Namespace **修改**的角色名。代码如下：

    ```Java
    // RoleUtils.java
    public static String buildModifyNamespaceRoleName(String appId, String namespaceName) {
        return STRING_JOINER.join(RoleType.MODIFY_NAMESPACE, appId, namespaceName);
    }
    
    // RoleType.java
    public static final String MODIFY_NAMESPACE = "ModifyNamespace";
    ```

* `RoleUtils#buildReleaseNamespaceRoleName(appId, namespaceName)` 方法，创建 Namespace **发布**的角色名。代码如下：

    ```Java
    // RoleUtils.java
    public static String buildReleaseNamespaceRoleName(String appId, String namespaceName) {
        return STRING_JOINER.join(RoleType.RELEASE_NAMESPACE, appId, namespaceName);
    }
    
    // RoleType.java
    public static final String RELEASE_NAMESPACE = "ReleaseNamespace";
    ```

* `#createNamespaceRole(...)` 方法，创建 Namespace 的角色。代码如下：

    ```Java
    private void createNamespaceRole(String appId, String namespaceName, String permissionType,
                                     String roleName, String operator) {
        // 创建 Namespace 对应的 Permission 对象，并保存到数据库
        Permission permission = createPermission(RoleUtils.buildNamespaceTargetId(appId, namespaceName), permissionType, operator);
        Permission createdPermission = rolePermissionService.createPermission(permission);
    
        // 创建 Namespace 对应的 Role 对象，并保存到数据库
        Role role = createRole(roleName, operator);
        rolePermissionService.createRoleWithPermissions(role, Sets.newHashSet(createdPermission.getId()));
    }
    ```
    * 创建并保存 Namespace 对应的 Permission 对象。
    * 创建并保存 Namespace 对应的 Role 对象，并授权对应的 Permission 。
    * `RoleUtils#buildNamespaceTargetId(appId, namespaceName)` 方法，创建 Namespace 的目标编号。代码如下：

        ```Java
        public static String buildNamespaceTargetId(String appId, String namespaceName) {
            return STRING_JOINER.join(appId, namespaceName);
        }
        ```
        * x

# 5. PermissionValidator

`com.ctrip.framework.apollo.portal.component.PermissionValidator` ，权限校验器。代码如下：

```Java
@Component("permissionValidator")
public class PermissionValidator {

    @Autowired
    private UserInfoHolder userInfoHolder;
    @Autowired
    private RolePermissionService rolePermissionService;
    @Autowired
    private PortalConfig portalConfig;

    // ========== Namespace 级别 ==========

    public boolean hasModifyNamespacePermission(String appId, String namespaceName) {
        return rolePermissionService.userHasPermission(userInfoHolder.getUser().getUserId(),
                PermissionType.MODIFY_NAMESPACE,
                RoleUtils.buildNamespaceTargetId(appId, namespaceName));
    }

    public boolean hasReleaseNamespacePermission(String appId, String namespaceName) {
        return rolePermissionService.userHasPermission(userInfoHolder.getUser().getUserId(),
                PermissionType.RELEASE_NAMESPACE,
                RoleUtils.buildNamespaceTargetId(appId, namespaceName));
    }

    public boolean hasDeleteNamespacePermission(String appId) {
        return hasAssignRolePermission(appId) || isSuperAdmin();
    }

    public boolean hasOperateNamespacePermission(String appId, String namespaceName) {
        return hasModifyNamespacePermission(appId, namespaceName) || hasReleaseNamespacePermission(appId, namespaceName);
    }

    // ========== App 级别 ==========

    public boolean hasAssignRolePermission(String appId) {
        return rolePermissionService.userHasPermission(userInfoHolder.getUser().getUserId(),
                PermissionType.ASSIGN_ROLE,
                appId);
    }

    public boolean hasCreateNamespacePermission(String appId) {
        return rolePermissionService.userHasPermission(userInfoHolder.getUser().getUserId(),
                PermissionType.CREATE_NAMESPACE,
                appId);
    }

    public boolean hasCreateAppNamespacePermission(String appId, AppNamespace appNamespace) {
        boolean isPublicAppNamespace = appNamespace.isPublic();
        // 若满足如下任一条件：
        // 1. 公开类型的 AppNamespace 。
        // 2. 私有类型的 AppNamespace ，并且允许 App 管理员创建私有类型的 AppNamespace 。
        if (portalConfig.canAppAdminCreatePrivateNamespace() || isPublicAppNamespace) {
            return hasCreateNamespacePermission(appId);
        }
        // 超管
        return isSuperAdmin();
    }

    public boolean hasCreateClusterPermission(String appId) {
        return rolePermissionService.userHasPermission(userInfoHolder.getUser().getUserId(),
                PermissionType.CREATE_CLUSTER,
                appId);
    }

    public boolean isAppAdmin(String appId) {
        return isSuperAdmin() || hasAssignRolePermission(appId);
    }

    // ========== 超管 级别 ==========

    public boolean isSuperAdmin() {
        return rolePermissionService.isSuperAdmin(userInfoHolder.getUser().getUserId());
    }

}
```

在每个需要校验权限的方法上，添加 `@PreAuthorize` 注解，并在 `value` 属性上写 EL 表达式，调用 PermissionValidator 的校验方法。例如：

* 创建 Namespace 的方法，添加了 `@PreAuthorize(value = "@permissionValidator.hasCreateNamespacePermission(#appId)")` 。
* 删除 Namespace 的方法，添加了 ` @PreAuthorize(value = "@permissionValidator.hasDeleteNamespacePermission(#appId)")` 。

通过这样的方式，达到**功能 + 数据级**的权限控制。

# 6. PermissionController

`com.ctrip.framework.apollo.portal.controller.PermissionController` ，提供**权限相关**的 **API** 。如下图所示：![PermissionController](http://www.iocoder.cn/images/Apollo/2018_06_05/08.png)

* 每个方法，调用 RolePermissionService 的方法，提供 API 服务。
* 🙂 代码比较简单，胖友自己查看。

对应界面为

* **App** 级权限管理：![项目管理](http://www.iocoder.cn/images/Apollo/2018_06_05/09.png)
* **Namespace** 级别权限管理：![权限管理](http://www.iocoder.cn/images/Apollo/2018_06_05/10.png)

# 666. 彩蛋

T T 老长一篇。哈哈哈，有种把所有代码 copy 过来的感觉。

突然发现没分享 `com.ctrip.framework.apollo.portal.spi.configurationRoleConfiguration` ，**Role** Spring Java 配置。代码如下：

```Java
@Configuration
public class RoleConfiguration {

    @Bean
    public RoleInitializationService roleInitializationService() {
        return new DefaultRoleInitializationService();
    }

    @Bean
    public RolePermissionService rolePermissionService() {
        return new DefaultRolePermissionService();
    }

}
```

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

