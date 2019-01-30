title: Apollo 源码解析 —— Portal 创建 Item
date: 2018-03-20
tags:
categories: Apollo
permalink: Apollo/portal-create-item
wechat_url:
toutiao_url: https://www.toutiao.com/i6637842822575686157/

---

摘要: 原创出处 http://www.iocoder.cn/Apollo/portal-create-item/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Apollo/portal-create-item/)
- [2. 实体](http://www.iocoder.cn/Apollo/portal-create-item/)
  - [2.1 Item](http://www.iocoder.cn/Apollo/portal-create-item/)
  - [2.2 Commit](http://www.iocoder.cn/Apollo/portal-create-item/)
- [3. Portal 侧](http://www.iocoder.cn/Apollo/portal-create-item/)
  - [3.1 ItemController](http://www.iocoder.cn/Apollo/portal-create-item/)
  - [3.2 ItemService](http://www.iocoder.cn/Apollo/portal-create-item/)
  - [3.3 ItemAPI](http://www.iocoder.cn/Apollo/portal-create-item/)
- [4. Admin Service 侧](http://www.iocoder.cn/Apollo/portal-create-item/)
  - [4.1 ItemController](http://www.iocoder.cn/Apollo/portal-create-item/)
  - [4.2 ItemService](http://www.iocoder.cn/Apollo/portal-create-item/)
  - [4.3 ItemRepository](http://www.iocoder.cn/Apollo/portal-create-item/)
  - [4.4 CommitService](http://www.iocoder.cn/Apollo/portal-create-item/)
  - [4.5 CommitRepository](http://www.iocoder.cn/Apollo/portal-create-item/)
- [666. 彩蛋](http://www.iocoder.cn/Apollo/portal-create-item/)

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

Item ，配置项，是 Namespace 下**最小颗粒度**的单位。在 Namespace 分成**五种**类型：`properties` `yml` `yaml` `json` `xml` 。其中：

* **`properties`** ：**每一行**配置对应**一条** Item 记录。
* **后四者**：无法进行拆分，所以**一个** Namespace **仅仅**对应**一条** Item 记录。

本文先分享 **Portal 创建类型为 properties 的 Namespace 的 Item** 的流程，整个过程涉及 Portal、Admin Service ，如下图所示：

![流程](http://www.iocoder.cn/images/Apollo/2018_03_20/01.png)

> 老艿艿：因为 Portal 是管理后台，所以从代码实现上，和业务系统非常相像。也因此，本文会略显啰嗦。

# 2. 实体

## 2.1 Item

`com.ctrip.framework.apollo.biz.entity.Item` ，继承 BaseEntity 抽象类，Item **实体**。代码如下：

```Java
@Entity
@Table(name = "Item")
@SQLDelete(sql = "Update Item set isDeleted = 1 where id = ?")
@Where(clause = "isDeleted = 0")
public class Item extends BaseEntity {

    /**
     * Namespace 编号
     */
    @Column(name = "NamespaceId", nullable = false)
    private long namespaceId;
    /**
     * 键
     */
    @Column(name = "key", nullable = false)
    private String key;
    /**
     * 值
     */
    @Column(name = "value")
    @Lob
    private String value;
    /**
     * 注释
     */
    @Column(name = "comment")
    private String comment;
    /**
     * 行号，从一开始。
     *
     * 例如 Properties 中，多个配置项。每个配置项对应一行。
     */
    @Column(name = "LineNum")
    private Integer lineNum;
}
```

* `namespaceId` 字段，Namespace 编号，指向对应的 Namespace 记录。
* `key` 字段，键。
    * 对于  `properties` ，使用 Item 的 `key` ，对应**每条**配置项的键。
    * 对于 `yaml` 等等，使用 Item 的 `key = content` ，对应**整个**配置文件。
* `lineNum` 字段，行号，从**一**开始。主要用于 `properties` 类型的配置文件。

## 2.2 Commit

`com.ctrip.framework.apollo.biz.entity.Commit` ，继承 BaseEntity 抽象类，Commit **实体**，记录 Item 的 **KV** 变更历史。代码如下：

```Java
@Entity
@Table(name = "Commit")
@SQLDelete(sql = "Update Commit set isDeleted = 1 where id = ?")
@Where(clause = "isDeleted = 0")
public class Commit extends BaseEntity {

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
     * 变更集合。
     *
     * JSON 格式化，使用 {@link com.ctrip.framework.apollo.biz.utils.ConfigChangeContentBuilder} 生成
     */
    @Lob
    @Column(name = "ChangeSets", nullable = false)
    private String changeSets;
    /**
     * 备注
     */
    @Column(name = "Comment")
    private String comment;
    
}
```

* `appId` + `clusterName` + `namespaceName`  字段，可以确认唯一 Namespace 记录。
* `changeSets` 字段，Item 变更集合。JSON 格式化字符串，使用 ConfigChangeContentBuilder 构建。

### 2.2.1 ConfigChangeContentBuilder

`com.ctrip.framework.apollo.biz.utils.ConfigChangeContentBuilder` ，配置变更内容构建器。

**构造方法**

```Java
private static final Gson gson = new GsonBuilder().setDateFormat("yyyy-MM-dd HH:mm:ss").create();

/**
 * 创建 Item 集合
 */
private List<Item> createItems = new LinkedList<>();
/**
 * 更新 Item 集合
 */
private List<ItemPair> updateItems = new LinkedList<>();
/**
 * 删除 Item 集合
 */
private List<Item> deleteItems = new LinkedList<>();
```

* `createItems` 字段，添加代码如下：

    ```Java
    public ConfigChangeContentBuilder createItem(Item item) {
        if (!StringUtils.isEmpty(item.getKey())) {
            createItems.add(cloneItem(item));
        }
        return this;
    }
    ```
    * 调用 `#cloneItem(Item)` 方法，克隆 Item 对象。因为在 `#build()` 方法中，会修改 Item 对象的属性。代码如下：

        ```Java
        Item cloneItem(Item source) {
            Item target = new Item();
            BeanUtils.copyProperties(source, target);
            return target;
        }
        ```
        * x

* `updateItems` 字段，添加代码如下：

    ```Java
    public ConfigChangeContentBuilder updateItem(Item oldItem, Item newItem) {
        if (!oldItem.getValue().equals(newItem.getValue())) {
            ItemPair itemPair = new ItemPair(cloneItem(oldItem), cloneItem(newItem));
            updateItems.add(itemPair);
        }
        return this;
    }
    ```
    * ItemPair ，Item **组**，代码如下：

        ```Java
        static class ItemPair {
        
            Item oldItem; // 老
            Item newItem; // 新
        
            public ItemPair(Item oldItem, Item newItem) {
                this.oldItem = oldItem;
                this.newItem = newItem;
            }
        
        }
        ```
        * x

* `deleteItems` 字段，添加代码如下：

    ```Java
    public ConfigChangeContentBuilder deleteItem(Item item) {
        if (!StringUtils.isEmpty(item.getKey())) {
            deleteItems.add(cloneItem(item));
        }
        return this;
    }
    ```

**hasContent**

`#hasContent()` 方法，判断是否有变化。**当且仅当有变化才记录 Commit**。代码如下：

```Java
public boolean hasContent() {
    return !createItems.isEmpty() || !updateItems.isEmpty() || !deleteItems.isEmpty();
}
```

**build**

`#build()` 方法，构建 Item 变化的 JSON 字符串。代码如下：

```Java
public String build() {
    // 因为事务第一段提交并没有更新时间,所以build时统一更新
    Date now = new Date();
    for (Item item : createItems) {
        item.setDataChangeLastModifiedTime(now);
    }
    for (ItemPair item : updateItems) {
        item.newItem.setDataChangeLastModifiedTime(now);
    }
    for (Item item : deleteItems) {
        item.setDataChangeLastModifiedTime(now);
    }
    // JSON 格式化成字符串
    return gson.toJson(this);
}
```

* 例子如下：

    ```JSON
    // 已经使用 http://tool.oschina.net/codeformat/json/ 进行格式化，实际是**紧凑型**
    {
        "createItems": [ ], 
        "updateItems": [
            {
                "oldItem": {
                    "namespaceId": 32, 
                    "key": "key4", 
                    "value": "value4123", 
                    "comment": "123", 
                    "lineNum": 4, 
                    "id": 15, 
                    "isDeleted": false, 
                    "dataChangeCreatedBy": "apollo", 
                    "dataChangeCreatedTime": "2018-04-27 16:49:59", 
                    "dataChangeLastModifiedBy": "apollo", 
                    "dataChangeLastModifiedTime": "2018-04-27 22:37:52"
                }, 
                "newItem": {
                    "namespaceId": 32, 
                    "key": "key4", 
                    "value": "value41234", 
                    "comment": "123", 
                    "lineNum": 4, 
                    "id": 15, 
                    "isDeleted": false, 
                    "dataChangeCreatedBy": "apollo", 
                    "dataChangeCreatedTime": "2018-04-27 16:49:59", 
                    "dataChangeLastModifiedBy": "apollo", 
                    "dataChangeLastModifiedTime": "2018-04-27 22:38:58"
                }
            }
        ], 
        "deleteItems": [ ]
    }
    ```

# 3. Portal 侧

## 3.1 ItemController

在 `apollo-portal` 项目中，`com.ctrip.framework.apollo.portal.controller.ItemController` ，提供 Item 的 **API** 。

在【**添加配置项**】的界面中，点击【提交】按钮，调用**创建 Item 的 API** 。

![添加配置项](http://www.iocoder.cn/images/Apollo/2018_03_20/02.png)

`#createItem(appId, env, clusterName, namespaceName, ItemDTO)` 方法，创建 Item 对象。代码如下：

```Java
  1: @RestController
  2: public class ItemController {
  3: 
  4:     @Autowired
  5:     private ItemService configService;
  6:     @Autowired
  7:     private UserInfoHolder userInfoHolder;
  8:     
  9:     @PreAuthorize(value = "@permissionValidator.hasModifyNamespacePermission(#appId, #namespaceName)")
 10:     @RequestMapping(value = "/apps/{appId}/envs/{env}/clusters/{clusterName}/namespaces/{namespaceName}/item", method = RequestMethod.POST)
 11:     public ItemDTO createItem(@PathVariable String appId, @PathVariable String env,
 12:                               @PathVariable String clusterName, @PathVariable String namespaceName,
 13:                               @RequestBody ItemDTO item) {
 14:         // 校验 Item 格式正确
 15:         checkModel(isValidItem(item));
 16:         // protect
 17:         item.setLineNum(0);
 18:         item.setId(0);
 19:         // 设置 ItemDTO 的创建和修改人为当前管理员
 20:         String userId = userInfoHolder.getUser().getUserId();
 21:         item.setDataChangeCreatedBy(userId);
 22:         item.setDataChangeLastModifiedBy(userId);
 23:         // protect
 24:         item.setDataChangeCreatedTime(null);
 25:         item.setDataChangeLastModifiedTime(null);
 26:         // 保存 Item 到 Admin Service
 27:         return configService.createItem(appId, Env.valueOf(env), clusterName, namespaceName, item);
 28:     }
 29:     
 30:     // ... 省略 deleteCluster 接口
 31: }    
```

* **POST `/apps/{appId}/envs/{env}/clusters/{clusterName}/namespaces/{namespaceName}/item` 接口**，Request Body 传递 **JSON** 对象。
* `@PreAuthorize(...)` 注解，调用 `PermissionValidator#hasModifyNamespacePermission(appId, namespaceName)` 方法，校验是否有**修改** Namespace 的权限。后续文章，详细分享。
* `com.ctrip.framework.apollo.common.dto.ItemDTO` ，Item DTO 。代码如下：

    ```Java
    public class ItemDTO extends BaseDTO {
    
        /**
         * Item 编号
         */
        private long id;
        /**
         * Namespace 编号
         */
        private long namespaceId;
        /**
         * 键
         */
        private String key;
        /**
         * 值
         */
        private String value;
        /**
         * 备注
         */
        private String comment;
        /**
         * 行数
         */
        private int lineNum;
    }
    ```

* 第 14 行：调用 `#isValidItem(ItemDTO)` 方法，校验 Item 格式正确。代码如下：

    ```Java
    private boolean isValidItem(ItemDTO item) {
        return Objects.nonNull(item) // 非空
                && !StringUtils.isContainEmpty(item.getKey()); // 键非空
    }
    ```

* 第 16 至 18 行 && 第 23 至 25 行：防御性编程，这几个参数不需要从 Portal 传递。
* 第 19 至 22 行：设置 ItemDTO 的创建和修改人为当前管理员。
* 第 27 行：调用 `ConfigService#createItem(appId, Env, clusterName, namespaceName, ItemDTO)` 方法，保存 Item 到 Admin Service 中。

## 3.2 ItemService

在 `apollo-portal` 项目中，`com.ctrip.framework.apollo.portal.service.ItemService` ，提供 Item 的 **Service** 逻辑。

`#createItem(appId, env, clusterName, namespaceName, ItemDTO)` 方法，创建并保存 Item 到 Admin Service 。代码如下：

```Java
  1: @Autowired
  2: private AdminServiceAPI.NamespaceAPI namespaceAPI;
  3: @Autowired
  4: private AdminServiceAPI.ItemAPI itemAPI;
  5:     
  6: public ItemDTO createItem(String appId, Env env, String clusterName, String namespaceName, ItemDTO item) {
  7:     // 校验 NamespaceDTO 是否存在。若不存在，抛出 BadRequestException 异常
  8:     NamespaceDTO namespace = namespaceAPI.loadNamespace(appId, env, clusterName, namespaceName);
  9:     if (namespace == null) {
 10:         throw new BadRequestException("namespace:" + namespaceName + " not exist in env:" + env + ", cluster:" + clusterName);
 11:     }
 12:     // 设置 ItemDTO 的 `namespaceId`
 13:     item.setNamespaceId(namespace.getId());
 14:     // 保存 Item 到 Admin Service
 15:     ItemDTO itemDTO = itemAPI.createItem(appId, env, clusterName, namespaceName, item);
 16:     // 【TODO 6001】Tracer 日志
 17:     Tracer.logEvent(TracerEventType.MODIFY_NAMESPACE, String.format("%s+%s+%s+%s", appId, env, clusterName, namespaceName));
 18:     return itemDTO;
 19: } 
```

* 第 7 至11 行：调用 `NamespaceAPI#loadNamespace(appId, Env, clusterName, namespaceName)` 方法，**校验** Namespace 是否存在。若不存在，抛出 BadRequestException 异常。**注意**，此处是远程调用 Admin Service 的 API 。
* 第 12 行：设置 ItemDTO 的 `namespaceId` 。
* 第 15 行：调用 `NamespaceAPI#createItem(appId, Env, clusterName, namespaceName, ItemDTO)` 方法，保存 Item 到 Admin Service 。
* 第 17 行：【TODO 6001】Tracer 日志

## 3.3 ItemAPI

`com.ctrip.framework.apollo.portal.api.ItemAPI` ，实现 API 抽象类，封装对 Admin Service 的 Item 模块的 API 调用。代码如下：

![ItemAPI](http://www.iocoder.cn/images/Apollo/2018_03_20/03.png)

# 4. Admin Service 侧

## 4.1 ItemController

在 `apollo-adminservice` 项目中， `com.ctrip.framework.apollo.adminservice.controller.ItemController` ，提供 Item 的 **API** 。

`#create(appId, clusterName, namespaceName, ItemDTO)` 方法，创建 Item ，并记录 Commit 。代码如下：

```Java
  1: @RestController
  2: public class ItemController {
  3: 
  4:     @Autowired
  5:     private ItemService itemService;
  6:     @Autowired
  7:     private CommitService commitService;
  8: 
  9:     @PreAcquireNamespaceLock
 10:     @RequestMapping(path = "/apps/{appId}/clusters/{clusterName}/namespaces/{namespaceName}/items", method = RequestMethod.POST)
 11:     public ItemDTO create(@PathVariable("appId") String appId,
 12:                           @PathVariable("clusterName") String clusterName,
 13:                           @PathVariable("namespaceName") String namespaceName,
 14:                           @RequestBody ItemDTO dto) {
 15:         // 将 ItemDTO 转换成 Item 对象
 16:         Item entity = BeanUtils.transfrom(Item.class, dto);
 17:         // 创建 ConfigChangeContentBuilder 对象
 18:         ConfigChangeContentBuilder builder = new ConfigChangeContentBuilder();
 19:         // 校验对应的 Item 是否已经存在。若是，抛出 BadRequestException 异常。
 20:         Item managedEntity = itemService.findOne(appId, clusterName, namespaceName, entity.getKey());
 21:         if (managedEntity != null) {
 22:             throw new BadRequestException("item already exist");
 23:         } else {
 24:             // 保存 Item 对象
 25:             entity = itemService.save(entity);
 26:             // 添加到 ConfigChangeContentBuilder 中
 27:             builder.createItem(entity);
 28:         }
 29:         // 将 Item 转换成 ItemDTO 对象
 30:         dto = BeanUtils.transfrom(ItemDTO.class, entity);
 31:         // 创建 Commit 对象
 32:         Commit commit = new Commit();
 33:         commit.setAppId(appId);
 34:         commit.setClusterName(clusterName);
 35:         commit.setNamespaceName(namespaceName);
 36:         commit.setChangeSets(builder.build()); // ConfigChangeContentBuilder 构造变更
 37:         commit.setDataChangeCreatedBy(dto.getDataChangeLastModifiedBy());
 38:         commit.setDataChangeLastModifiedBy(dto.getDataChangeLastModifiedBy());
 39:         // 保存 Commit 对象
 40:         commitService.save(commit);
 41:         return dto;
 42:     }
 43:     
 44:     // ... 省略其他接口和属性
 45: }
```

* **POST `/apps/{appId}/clusters/{clusterName}/namespaces/{namespaceName}/items` 接口**，Request Body 传递 **JSON** 对象。
* 第 16 行：调用 `BeanUtils#transfrom(Class<T> clazz, Object src)` 方法，将 ItemDTO **转换**成 Item 对象。
* 第 18 行：创建 ConfigChangeContentBuilder 对象。
* 第 19 至 22 行：调用 `ItemService#findOne(appId, clusterName, namespaceName, key)` 方法，**校验**对应的 Item 是否已经存在。若是，抛出 BadRequestException 异常。
* 第 25 行：调用 `ItemService#save(Item)` 方法，保存 Item 对象。
* 第 27 行：调用 `ConfigChangeContentBuilder#createItem(Item)` 方法，添加到 ConfigChangeContentBuilder 中。
* 第 30 行：调用 `BeanUtils#transfrom(Class<T> clazz, Object src)` 方法，将 Item **转换**成 ItemDTO 对象。
* 第 31 至 38 行：创建 Commit 对象。
* 第 40 行：调用 `CommitService#save(Commit)` 方法，保存 Commit 对象。

## 4.2 ItemService

在 `apollo-biz` 项目中，`com.ctrip.framework.apollo.biz.service.ItemService` ，提供 Item 的 **Service** 逻辑给 Admin Service 和 Config Service 。

`#save(Item)` 方法，保存 Item 对象 。代码如下：

```Java
  1: @Autowired
  2: private ItemRepository itemRepository;
  3: @Autowired
  4: private AuditService auditService;
  5: 
  6: @Transactional
  7: public Item save(Item entity) {
  8:     // 校验 Key 长度
  9:     checkItemKeyLength(entity.getKey());
 10:     // 校验 Value 长度
 11:     checkItemValueLength(entity.getNamespaceId(), entity.getValue());
 12:     // protection
 13:     entity.setId(0);
 14:     // 设置 Item 的行号，以 Namespace 下的 Item 最大行号 + 1 。
 15:     if (entity.getLineNum() == 0) {
 16:         Item lastItem = findLastOne(entity.getNamespaceId());
 17:         int lineNum = lastItem == null ? 1 : lastItem.getLineNum() + 1;
 18:         entity.setLineNum(lineNum);
 19:     }
 20:     // 保存 Item
 21:     Item item = itemRepository.save(entity);
 22:     // 记录 Audit 到数据库中
 23:     auditService.audit(Item.class.getSimpleName(), item.getId(), Audit.OP.INSERT, item.getDataChangeCreatedBy());
 24:     return item;
 25: }
```

* 第 9 行：调用 `#checkItemKeyLength(key)` 方法，**校验** Key 长度。
    * 可配置 `"item.value.length.limit"` 在 ServerConfig 配置最大长度。
    * 默认最大长度为 128 。
* 第 11 行：调用 `#checkItemValueLength(namespaceId, value)` 方法，**校验** Value 长度。
    * 全局可配置 `"item.value.length.limit"` 在 ServerConfig 配置最大长度。
    * **自定义**配置 `"namespace.value.length.limit.override"` 在 ServerConfig 配置最大长度。
    * 默认最大长度为 20000 。
* 第 14 至 19 行：设置 Item 的行号，以 Namespace 下的 Item **最大**行号 + 1 。`#findLastOne(namespaceId)` 方法，获得最大行号的 Item 对象，代码如下：

    ```Java
    public Item findLastOne(long namespaceId) {
        return itemRepository.findFirst1ByNamespaceIdOrderByLineNumDesc(namespaceId);
    }
    ```

* 第 21 行：调用 `ItemRepository#save(Item)` 方法，保存 Item 。
* 第 23 行：记录 Audit 到数据库中

## 4.3 ItemRepository

`com.ctrip.framework.apollo.biz.repository.ItemRepository` ，继承 `org.springframework.data.repository.PagingAndSortingRepository` 接口，提供 Item 的**数据访问** 给 Admin Service 和 Config Service 。代码如下：

```Java
public interface ItemRepository extends PagingAndSortingRepository<Item, Long> {

  Item findByNamespaceIdAndKey(Long namespaceId, String key);

  List<Item> findByNamespaceIdOrderByLineNumAsc(Long namespaceId);

  List<Item> findByNamespaceId(Long namespaceId);

  List<Item> findByNamespaceIdAndDataChangeLastModifiedTimeGreaterThan(Long namespaceId, Date date);

  Item findFirst1ByNamespaceIdOrderByLineNumDesc(Long namespaceId);

  @Modifying
  @Query("update Item set isdeleted=1,DataChange_LastModifiedBy = ?2 where namespaceId = ?1")
  int deleteByNamespaceId(long namespaceId, String operator);

}
```

## 4.4 CommitService

在 `apollo-biz` 项目中，`com.ctrip.framework.apollo.biz.service.CommitService` ，提供 Commit 的 **Service** 逻辑给 Admin Service 和 Config Service 。

`#save(Commit)` 方法，保存 Item 对象 。代码如下：

```Java
@Autowired
private CommitRepository commitRepository;

@Transactional
public Commit save(Commit commit) {
    //protection
    commit.setId(0);
    // 保存 Commit
    return commitRepository.save(commit);
}
```

## 4.5 CommitRepository

`com.ctrip.framework.apollo.biz.repository.CommitRepository` ，继承 `org.springframework.data.repository.PagingAndSortingRepository` 接口，提供 Commit 的**数据访问** 给 Admin Service 和 Config Service 。代码如下：

```Java
public interface CommitRepository extends PagingAndSortingRepository<Commit, Long> {

  List<Commit> findByAppIdAndClusterNameAndNamespaceNameOrderByIdDesc(String appId, String clusterName,
                                                                      String namespaceName, Pageable pageable);

  @Modifying
  @Query("update Commit set isdeleted=1,DataChange_LastModifiedBy = ?4 where appId=?1 and clusterName=?2 and namespaceName = ?3")
  int batchDelete(String appId, String clusterName, String namespaceName, String operator);

}
```

# 666. 彩蛋

Commit 的设计，在我们日常的管理后台，对重要数据的变更，可以作为参考。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)


