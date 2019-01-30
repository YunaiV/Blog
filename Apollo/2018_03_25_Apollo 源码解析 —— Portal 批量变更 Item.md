title: Apollo 源码解析 —— Portal 批量变更 Item
date: 2018-03-25
tags:
categories: Apollo
permalink: Apollo/portal-update-item-set
wechat_url:
toutiao_url: https://www.toutiao.com/i6637844102853427720/

---

摘要: 原创出处 http://www.iocoder.cn/Apollo/portal-update-item-set/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Apollo/portal-update-item-set/)
- [2. ItemChangeSets](http://www.iocoder.cn/Apollo/portal-update-item-set/)
- [3. ConfigTextResolver](http://www.iocoder.cn/Apollo/portal-update-item-set/)
  - [3.1 FileTextResolver](http://www.iocoder.cn/Apollo/portal-update-item-set/)
  - [3.2 PropertyResolver](http://www.iocoder.cn/Apollo/portal-update-item-set/)
- [4. Portal 侧](http://www.iocoder.cn/Apollo/portal-update-item-set/)
  - [4.1 ItemController](http://www.iocoder.cn/Apollo/portal-update-item-set/)
  - [4.2 ItemService](http://www.iocoder.cn/Apollo/portal-update-item-set/)
  - [4.3 ItemAPI](http://www.iocoder.cn/Apollo/portal-update-item-set/)
- [5. Admin Service 侧](http://www.iocoder.cn/Apollo/portal-update-item-set/)
  - [5.1 ItemSetController](http://www.iocoder.cn/Apollo/portal-update-item-set/)
  - [5.2 ItemSetService](http://www.iocoder.cn/Apollo/portal-update-item-set/)
- [666. 彩蛋](http://www.iocoder.cn/Apollo/portal-update-item-set/)

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

本文接 [《Apollo 源码解析 —— Portal 创建 Item》](http://www.iocoder.cn/Apollo/portal-create-item/?self) 文章，分享 Item 的**批量变更**。

* 对于 `yaml` `yml` `json` `xml` 数据类型的 Namespace ，仅有一条 Item 记录，所以批量修改实际是修改**该条** Item 。
* 对于 `properties` 数据类型的 Namespace ，有多条 Item 记录，所以批量变更是**多条** Item 。

整体流程如下图：

![流程](http://www.iocoder.cn/images/Apollo/2018_03_25/01.png)

> 老艿艿：因为 Portal 是管理后台，所以从代码实现上，和业务系统非常相像。也因此，本文会略显啰嗦。

# 2. ItemChangeSets

`com.ctrip.framework.apollo.common.dto.ItemChangeSets` ，Item 变更**集合**。代码如下：

```Java
public class ItemChangeSets extends BaseDTO {

    /**
     * 新增 Item 集合
     */
    private List<ItemDTO> createItems = new LinkedList<>();
    /**
     * 修改 Item 集合
     */
    private List<ItemDTO> updateItems = new LinkedList<>();
    /**
     * 删除 Item 集合
     */
    private List<ItemDTO> deleteItems = new LinkedList<>();

    public void addCreateItem(ItemDTO item) {
        createItems.add(item);
    }

    public void addUpdateItem(ItemDTO item) {
        updateItems.add(item);
    }

    public void addDeleteItem(ItemDTO item) {
        deleteItems.add(item);
    }
    
    public boolean isEmpty() {
        return createItems.isEmpty() && updateItems.isEmpty() && deleteItems.isEmpty();
    }
    
    // ... 省略 setting / getting 方法
}
```

# 3. ConfigTextResolver

在 `apollo-portal` 项目中， `com.ctrip.framework.apollo.portal.component.txtresolver.ConfigTextResolver` ，配置文本解析器**接口**。代码如下：

```Java
public interface ConfigTextResolver {

    /**
     * 解析文本，创建 ItemChangeSets 对象
     *
     * @param namespaceId Namespace 编号
     * @param configText 配置文本
     * @param baseItems 已存在的 ItemDTO 们
     * @return ItemChangeSets 对象
     */
    ItemChangeSets resolve(long namespaceId, String configText, List<ItemDTO> baseItems);

}
```

## 3.1 FileTextResolver

`com.ctrip.framework.apollo.portal.component.txtresolver.FileTextResolver` ，实现 ConfigTextResolver 接口，**文件**配置文本解析器，适用于 `yaml`、`yml`、`json`、`xml` 格式。代码如下：

```Java
  1: @Override
  2: public ItemChangeSets resolve(long namespaceId, String configText, List<ItemDTO> baseItems) {
  3:     ItemChangeSets changeSets = new ItemChangeSets();
  4:     // 配置文本为空，不进行修改
  5:     if (StringUtils.isEmpty(configText)) {
  6:         return changeSets;
  7:     }
  8:     // 不存在已有配置，创建 ItemDTO 到 ItemChangeSets 新增项
  9:     if (CollectionUtils.isEmpty(baseItems)) {
 10:         changeSets.addCreateItem(createItem(namespaceId, 0, configText));
 11:     // 已存在配置，创建 ItemDTO 到 ItemChangeSets 修改项
 12:     } else {
 13:         ItemDTO beforeItem = baseItems.get(0);
 14:         if (!configText.equals(beforeItem.getValue())) { //update
 15:             changeSets.addUpdateItem(createItem(namespaceId, beforeItem.getId(), configText));
 16:         }
 17:     }
 18:     return changeSets;
 19: }
```

* 第 3 行：创建 ItemChangeSets 对象。
* 第 4 至 7 行：若配置文件为**空**，不进行修改。
* 第 8 至 10 行：不存在已有配置( `baseItems` ) ，创建 ItemDTO 到 ItemChangeSets 新增项。
* 第 11 至 17 行：已存在配置，并且配置值**不相等**，创建 ItemDTO 到 ItemChangeSets 修改项。**注意**，选择了第一条 ItemDTO 进行对比，因为 `yaml` 等，有且仅有一条。
* `#createItem(long namespaceId, long itemId, String value)` 方法，创建 ItemDTO 对象。代码如下：

    ```Java
    private ItemDTO createItem(long namespaceId, long itemId, String value) {
        ItemDTO item = new ItemDTO();
        item.setId(itemId);
        item.setNamespaceId(namespaceId);
        item.setValue(value);
        item.setLineNum(1);
        item.setKey(ConfigConsts.CONFIG_FILE_CONTENT_KEY);
        return item;
    }
    ```

## 3.2 PropertyResolver

`com.ctrip.framework.apollo.portal.component.txtresolver.PropertyResolver` ，实现 ConfigTextResolver 接口，`properties` 配置解析器。代码如下：

```Java

  1: private static final String KV_SEPARATOR = "=";
  2: private static final String ITEM_SEPARATOR = "\n";
  3: 
  4: @Override
  5: public ItemChangeSets resolve(long namespaceId, String configText, List<ItemDTO> baseItems) {
  6:     // 创建 Item Map ，以 lineNum 为 键
  7:     Map<Integer, ItemDTO> oldLineNumMapItem = BeanUtils.mapByKey("lineNum", baseItems);
  8:     // 创建 Item Map ，以 key 为 键
  9:     Map<String, ItemDTO> oldKeyMapItem = BeanUtils.mapByKey("key", baseItems);
 10:     oldKeyMapItem.remove(""); // remove comment and blank item map.
 11: 
 12:     // 按照拆分 Property 配置
 13:     String[] newItems = configText.split(ITEM_SEPARATOR);
 14:     // 校验是否存在重复配置 Key 。若是，抛出 BadRequestException 异常
 15:     if (isHasRepeatKey(newItems)) {
 16:         throw new BadRequestException("config text has repeat key please check.");
 17:     }
 18: 
 19:     // 创建 ItemChangeSets 对象，并解析配置文件到 ItemChangeSets 中。
 20:     ItemChangeSets changeSets = new ItemChangeSets();
 21:     Map<Integer, String> newLineNumMapItem = new HashMap<>();//use for delete blank and comment item
 22:     int lineCounter = 1;
 23:     for (String newItem : newItems) {
 24:         newItem = newItem.trim();
 25:         newLineNumMapItem.put(lineCounter, newItem);
 26:         // 使用行号，获得已存在的 ItemDTO
 27:         ItemDTO oldItemByLine = oldLineNumMapItem.get(lineCounter);
 28:         // comment item 注释 Item
 29:         if (isCommentItem(newItem)) {
 30:             handleCommentLine(namespaceId, oldItemByLine, newItem, lineCounter, changeSets);
 31:         // blank item 空白 Item
 32:         } else if (isBlankItem(newItem)) {
 33:             handleBlankLine(namespaceId, oldItemByLine, lineCounter, changeSets);
 34:         // normal item 普通 Item
 35:         } else {
 36:             handleNormalLine(namespaceId, oldKeyMapItem, newItem, lineCounter, changeSets);
 37:         }
 38:         // 行号计数 + 1
 39:         lineCounter++;
 40:     }
 41:     // 删除注释和空行配置项
 42:     deleteCommentAndBlankItem(oldLineNumMapItem, newLineNumMapItem, changeSets);
 43:     // 删除普通配置项
 44:     deleteNormalKVItem(oldKeyMapItem, changeSets);
 45:     return changeSets;
 46: }
```

* 第 7 行：调用 `BeanUtils#mapByKey(String key, List<? extends Object> list)` 方法，创建 ItemDTO Map `oldLineNumMapItem` ，以 `lineNum` 属性为键。
* 第 9 至 10 行：调用 `BeanUtils#mapByKey(String key, List<? extends Object> list)` 方法，创建 ItemDTO Map `oldKeyMapItem` ，以 `key` 属性为键。
    * 移除 `key =""` 的原因是，移除**注释**和**空行**的配置项。
* 第 13 行：按照 `"\n"` 拆分 `properties` 配置。
* 第 15 至 17 行：调用 `#isHasRepeatKey(newItems)` 方法，**校验**是否存在重复配置 Key 。若是，抛出 BadRequestException 异常。代码如下：

    ```Java
    private boolean isHasRepeatKey(String[] newItems) {
        Set<String> keys = new HashSet<>();
        int lineCounter = 1; // 记录行数，用于报错提示，无业务逻辑需要。
        int keyCount = 0; // 计数
        for (String item : newItems) {
            if (!isCommentItem(item) && !isBlankItem(item)) { // 排除注释和空行的配置项
                keyCount++;
                String[] kv = parseKeyValueFromItem(item);
                if (kv != null) {
                    keys.add(kv[0]);
                } else {
                    throw new BadRequestException("line:" + lineCounter + " key value must separate by '='");
                }
            }
            lineCounter++;
        }
        return keyCount > keys.size();
    }
    ```
    * **基于 Set 做排重判断**。

* 第 19 至 44 行：创建 ItemChangeSets 对象，并解析配置文本到 ItemChangeSets 中。
    *  第 23 行：**循环** `newItems` 。 
    *  第 27 行：使用**行号**，获得对应的**老的** ItemDTO 配置项。
    *  ========== **注释**配置项 【基于**行数**】 ==========
    *  第 29 行：调用 `#isCommentItem(newItem)` 方法，判断是否为**注释**配置文本。代码如下：

        ```Java
        private boolean isCommentItem(String line) {
            return line != null && (line.startsWith("#") || line.startsWith("!"));
        }
        ```
        * x
    * 第 30 行：调用 `#handleCommentLine(namespaceId, oldItemByLine, newItem, lineCounter, changeSets)` 方法，处理**注释**配置项。代码如下：

        ```Java
          1: private void handleCommentLine(Long namespaceId, ItemDTO oldItemByLine, String newItem, int lineCounter, ItemChangeSets changeSets) {
          2:     String oldComment = oldItemByLine == null ? "" : oldItemByLine.getComment();
          3:     // create comment. implement update comment by delete old comment and create new comment
          4:     // 创建注释 ItemDTO 到 ItemChangeSets 的新增项，若老的配置项不是注释或者不相等。另外，更新注释配置，通过删除 + 添加的方式。
          5:     if (!(isCommentItem(oldItemByLine) && newItem.equals(oldComment))) {
          6:         changeSets.addCreateItem(buildCommentItem(0L, namespaceId, newItem, lineCounter));
          7:     }
          8: }
        ```
        * 创建注释 ItemDTO 到 ItemChangeSets 的**新增项**，若老的配置项*不是注释*或者*不相等*。另外，更新注释配置，通过**删除 + 添加**的方式。
        * `#buildCommentItem(id, namespaceId, comment, lineNum)` 方法，创建**注释** ItemDTO 对象。代码如下：

            ```Java
            private ItemDTO buildCommentItem(Long id, Long namespaceId, String comment, int lineNum) {
                return buildNormalItem(id, namespaceId, ""/* key */, "" /* value */, comment, lineNum);
            }
            ```
            * `key` 和 `value` 的属性，使用 `""` 空串。
        
    * ========== **空行**配置项 【基于**行数**】 ==========
    * 第 32 行：调用 调用 `#isBlankItem(newItem)` 方法，判断是否为**空行**配置文本。代码如下：

        ```Java
        private boolean isBlankItem(String line) {
            return "".equals(line);
        }
        ```
        * x
    
    * 第 33 行：调用 `#handleBlankLine(namespaceId, oldItemByLine, lineCounter, changeSets)` 方法，处理**空行**配置项。代码如下：

        ```Java
          1: private void handleBlankLine(Long namespaceId, ItemDTO oldItem, int lineCounter, ItemChangeSets changeSets) {
          2:     // 创建空行 ItemDTO 到 ItemChangeSets 的新增项，若老的不是空行。另外，更新空行配置，通过删除 + 添加的方式
          3:     if (!isBlankItem(oldItem)) {
          4:         changeSets.addCreateItem(buildBlankItem(0L, namespaceId, lineCounter));
          5:     }
          6: }
        ```
        * 创建**空行** ItemDTO 到 ItemChangeSets 的**新增项**，若老的*不是空行*。另外，更新空行配置，通过**删除 + 添加**的方式。
        * `#buildBlankItem(id, namespaceId, lineNum)` 方法，处理**空行**配置项。代码如下：
    
            ```Java
            private ItemDTO buildBlankItem(Long id, Long namespaceId, int lineNum) {
                return buildNormalItem(id, namespaceId, "" /* key */, "" /* value */, "" /* comment */, lineNum);
            }
            ```
            * 和 `#buildCommentItem(...)` 的差异点是，`comment` 是 `""` 空串。
        
    * ========== **普通**配置项 【基于 **Key** 】 ==========
    * 第 36 行：调用 `#handleNormalLine(namespaceId, oldKeyMapItem, newItem, lineCounter, changeSets)` 方法，处理**普通**配置项。代码如下：

        ```Java
          1: private void handleNormalLine(Long namespaceId, Map<String, ItemDTO> keyMapOldItem, String newItem,
          2:                               int lineCounter, ItemChangeSets changeSets) {
          3:     // 解析一行，生成 [key, value]
          4:     String[] kv = parseKeyValueFromItem(newItem);
          5:     if (kv == null) {
          6:         throw new BadRequestException("line:" + lineCounter + " key value must separate by '='");
          7:     }
          8:     String newKey = kv[0];
          9:     String newValue = kv[1].replace("\\n", "\n"); //handle user input \n
         10:     // 获得老的 ItemDTO 对象
         11:     ItemDTO oldItem = keyMapOldItem.get(newKey);
         12:     // 不存在，则创建 ItemDTO 到 ItemChangeSets 的添加项
         13:     if (oldItem == null) {//new item
         14:         changeSets.addCreateItem(buildNormalItem(0L, namespaceId, newKey, newValue, "", lineCounter));
         15:     // 如果值或者行号不相等，则创建 ItemDTO 到 ItemChangeSets 的修改项
         16:     } else if (!newValue.equals(oldItem.getValue()) || lineCounter != oldItem.getLineNum()) {//update item
         17:         changeSets.addUpdateItem(buildNormalItem(oldItem.getId(), namespaceId, newKey, newValue, oldItem.getComment(), lineCounter));
         18:     }
         19:     // 移除老的 ItemDTO 对象
         20:     keyMapOldItem.remove(newKey);
         21: }
        ```
        * 第 3 至 9 行：调用 `#parseKeyValueFromItem(newItem)` 方法，解析一行，生成 `[key, value]` 。代码如下：

            ```Java
            private String[] parseKeyValueFromItem(String item) {
                int kvSeparator = item.indexOf(KV_SEPARATOR);
                if (kvSeparator == -1) {
                    return null;
                }
                String[] kv = new String[2];
                kv[0] = item.substring(0, kvSeparator).trim();
                kv[1] = item.substring(kvSeparator + 1, item.length()).trim();
                return kv;
            }
            ```
            * x
        * 第 11 行：获得老的 ItemDTO 对象。
        * 第 12 至 14 行：若老的 Item DTO 对象**不存在**，则创建 ItemDTO 到 ItemChangeSets 的**新增**项。
        * 第 15 至 18 行：若老的 Item DTO 对象**存在**，且*值*或者*行数*不相等，则创建 ItemDTO 到 ItemChangeSets 的**修改**项。
        * 第 20 行：移除老的 ItemDTO 对象。这样，最终 `keyMapOldItem` 保留的是，需要**删除**的普通配置项，详细见 `#deleteNormalKVItem(oldKeyMapItem, changeSets)` 方法。   
* 第 42 行：调用 `#deleteCommentAndBlankItem(oldLineNumMapItem, newLineNumMapItem, changeSets)` 方法，删除**注释**和**空行**配置项。代码如下：

    ```Java
    private void deleteCommentAndBlankItem(Map<Integer, ItemDTO> oldLineNumMapItem,
                                           Map<Integer, String> newLineNumMapItem,
                                           ItemChangeSets changeSets) {
        for (Map.Entry<Integer, ItemDTO> entry : oldLineNumMapItem.entrySet()) {
            int lineNum = entry.getKey();
            ItemDTO oldItem = entry.getValue();
            String newItem = newLineNumMapItem.get(lineNum);
            // 添加到 ItemChangeSets 的删除项
            // 1. old is blank by now is not
            // 2. old is comment by now is not exist or modified
            if ((isBlankItem(oldItem) && !isBlankItem(newItem)) // 老的是空行配置项，新的不是空行配置项
                    || isCommentItem(oldItem) && (newItem == null || !newItem.equals(oldItem.getComment()))) { // 老的是注释配置项，新的不相等
                changeSets.addDeleteItem(oldItem);
            }
        }
    }
    ```
    * 将需要删除( *具体条件看注释* ) 的注释和空白配置项，添加到 ItemChangeSets 的**删除项**中。

* 第 44 行：调用 #deleteNormalKVItem(oldKeyMapItem, changeSets) 方法，删除普通配置项。代码如下：

    ```Java
    private void deleteNormalKVItem(Map<String, ItemDTO> baseKeyMapItem, ItemChangeSets changeSets) {
        // 将剩余的配置项，添加到 ItemChangeSets 的删除项
        // surplus item is to be deleted
        for (Map.Entry<String, ItemDTO> entry : baseKeyMapItem.entrySet()) {
            changeSets.addDeleteItem(entry.getValue());
        }
    }
    ```
    * 将剩余的配置项( `oldLineNumMapItem` )，添加到 ItemChangeSets 的**删除项**。

🙂 整个方法比较冗长，建议胖友多多调试，有几个点特别需要注意：

* 对于**注释**和**空行**配置项，基于**行数**做比较。当发生变化时，使用**删除 + 创建**的方式。笔者的理解是，注释和空行配置项，是没有 Key ，每次变化都认为是**新的**。另外，这样也可以和**注释**和**空行**配置项被改成**普通**配置项，保持一致。例如，第一行原先是**注释**配置项，改成了**普通**配置项，从数据上也是**删除 + 创建**的方式。
* 对于**普通**配置项，基于 **Key** 做比较。例如，第一行原先是**普通**配置项，结果我们在敲了回车，在第一行添加了**注释**，那么认为是**普通**配置项修改了**行数**。

# 4. Portal 侧

## 4.1 ItemController

在 `apollo-portal` 项目中，`com.ctrip.framework.apollo.portal.controller.ItemController` ，提供 Item 的 **API** 。

在【**批量变更 Namespace 配置项**】的界面中，点击【 √ 】按钮，调用**批量变更 Namespace 的 Item 们的 API** 。

![批量变更 Namespace 配置项](http://www.iocoder.cn/images/Apollo/2018_03_20/02.png)

`#modifyItemsByText(appId, env, clusterName, namespaceName, NamespaceTextModel)` 方法，批量变更 Namespace 的 Item 们。代码如下：

```Java
  1: @Autowired
  2: private ItemService configService;
  3: 
  4: @PreAuthorize(value = "@permissionValidator.hasModifyNamespacePermission(#appId, #namespaceName)")
  5: @RequestMapping(value = "/apps/{appId}/envs/{env}/clusters/{clusterName}/namespaces/{namespaceName}/items", method = RequestMethod.PUT, consumes = {"application/json"})
  6: public void modifyItemsByText(@PathVariable String appId, @PathVariable String env,
  7:                           @PathVariable String clusterName, @PathVariable String namespaceName,
  8:                           @RequestBody NamespaceTextModel model) {
  9:    // 校验 `model` 非空
 10:    checkModel(model != null);
 11:    // 设置 PathVariable 到 `model` 中
 12:    model.setAppId(appId);
 13:    model.setClusterName(clusterName);
 14:    model.setEnv(env);
 15:    model.setNamespaceName(namespaceName);
 16:    // 批量更新一个 Namespace 下的 Item 们
 17:    configService.updateConfigItemByText(model);
 18: }
```

* **POST `/apps/{appId}/envs/{env}/clusters/{clusterName}/namespaces/{namespaceName}/items` 接口**，Request Body 传递 **JSON** 对象。
* `@PreAuthorize(...)` 注解，调用 `PermissionValidator#hasModifyNamespacePermission(appId, namespaceName)` 方法，校验是否有**修改** Namespace 的权限。后续文章，详细分享。
* `com.ctrip.framework.apollo.portal.entity.model.NamespaceTextModel` ，Namespace 下的配置文本 Model 。代码如下：

    ```Java
    public class NamespaceTextModel implements Verifiable {
    
        /**
         * App 编号
         */
        private String appId;
        /**
         * Env 名
         */
        private String env;
        /**
         * Cluster 名
         */
        private String clusterName;
        /**
         * Namespace 名
         */
        private String namespaceName;
        /**
         * Namespace 编号
         */
        private int namespaceId;
        /**
         * 格式
         */
        private String format;
        /**
         * 配置文本
         */
        private String configText;
        
        @Override
        public boolean isInvalid() {
            return StringUtils.isContainEmpty(appId, env, clusterName, namespaceName) || namespaceId <= 0;
        }
    }
    ```
    * 重点是 `configText` 属性，配置文本。

* 第 10 行：校验 NamespaceTextModel 非空。
* 第 11 至 15 行：设置 PathVariable 变量，到 NamespaceTextModel 中 。
* 第 17 行：调用 `ItemService#updateConfigItemByText(NamespaceTextModel)` 方法，批量更新一个 Namespace 下的 Item **们** 。

## 4.2 ItemService

在 `apollo-portal` 项目中，`com.ctrip.framework.apollo.portal.service.ItemService` ，提供 Item 的 **Service** 逻辑。

`#updateConfigItemByText(NamespaceTextModel)` 方法，解析配置文本，并批量更新 Namespace 的 Item 们。代码如下：

```Java
  1: @Autowired
  2: private UserInfoHolder userInfoHolder;
  3: @Autowired
  4: private AdminServiceAPI.ItemAPI itemAPI;
  5: 
  6: @Autowired
  7: @Qualifier("fileTextResolver")
  8: private ConfigTextResolver fileTextResolver;
  9: @Autowired
 10: @Qualifier("propertyResolver")
 11: private ConfigTextResolver propertyResolver;
 12: 
 13: public void updateConfigItemByText(NamespaceTextModel model) {
 14:     String appId = model.getAppId();
 15:     Env env = model.getEnv();
 16:     String clusterName = model.getClusterName();
 17:     String namespaceName = model.getNamespaceName();
 18:     long namespaceId = model.getNamespaceId();
 19:     String configText = model.getConfigText();
 20:     // 获得对应格式的 ConfigTextResolver 对象
 21:     ConfigTextResolver resolver = model.getFormat() == ConfigFileFormat.Properties ? propertyResolver : fileTextResolver;
 22:     // 解析成 ItemChangeSets
 23:     ItemChangeSets changeSets = resolver.resolve(namespaceId, configText, itemAPI.findItems(appId, env, clusterName, namespaceName));
 24:     if (changeSets.isEmpty()) {
 25:         return;
 26:     }
 27:     // 设置修改人为当前管理员
 28:     changeSets.setDataChangeLastModifiedBy(userInfoHolder.getUser().getUserId());
 29:     // 调用 Admin Service API ，批量更新 Item 们。
 30:     updateItems(appId, env, clusterName, namespaceName, changeSets);
 31:     // 【TODO 6001】Tracer 日志
 32:     Tracer.logEvent(TracerEventType.MODIFY_NAMESPACE_BY_TEXT, String.format("%s+%s+%s+%s", appId, env, clusterName, namespaceName));
 33:     Tracer.logEvent(TracerEventType.MODIFY_NAMESPACE, String.format("%s+%s+%s+%s", appId, env, clusterName, namespaceName));
 34: }
```

* 第 21 行：获得对应**格式**( `format` )的 ConfigTextResolver 对象。
* 第 23 行：调用 `ItemAPI#findItems(appId, env, clusterName, namespaceName)` 方法，获得 Namespace 下所有的 ItemDTO 配置项们。
* 第 23 行：调用 `ConfigTextResolver#resolve(...)` 方法，解析配置文本，生成 ItemChangeSets 对象。
* 第 24 至 26 行：调用 `ItemChangeSets#isEmpty()` 方法，若无变更项，直接返回。
* 第 30 行：调用 `#updateItems(appId, env, clusterName, namespaceName, changeSets)` 方法，调用 Admin Service API ，批量更新 Namespace 下的 Item 们。代码如下：

    ```Java
    public void updateItems(String appId, Env env, String clusterName, String namespaceName, ItemChangeSets changeSets) {
        itemAPI.updateItemsByChangeSet(appId, env, clusterName, namespaceName, changeSets);
    }
    ```

* 第 31 至 33 行：【TODO 6001】Tracer 日志

## 4.3 ItemAPI

`com.ctrip.framework.apollo.portal.api.ItemAPI` ，实现 API 抽象类，封装对 Admin Service 的 Item 模块的 API 调用。代码如下：

![ItemAPI](http://www.iocoder.cn/images/Apollo/2018_03_25/03.png)

# 5. Admin Service 侧

## 5.1 ItemSetController

在 `apollo-adminservice` 项目中， `com.ctrip.framework.apollo.adminservice.controller.ItemSetController` ，提供 Item **批量**的 **API** 。

`#create(appId, clusterName, namespaceName, ItemChangeSets)` 方法，批量更新 Namespace 下的 Item 们。代码如下：

```Java
@RestController
public class ItemSetController {

    @Autowired
    private ItemSetService itemSetService;

    @PreAcquireNamespaceLock
    @RequestMapping(path = "/apps/{appId}/clusters/{clusterName}/namespaces/{namespaceName}/itemset", method = RequestMethod.POST)
    public ResponseEntity<Void> create(@PathVariable String appId, @PathVariable String clusterName,
                                       @PathVariable String namespaceName, @RequestBody ItemChangeSets changeSet) {
        // 批量更新 Namespace 下的 Item 们
        itemSetService.updateSet(appId, clusterName, namespaceName, changeSet);
        return ResponseEntity.status(HttpStatus.OK).build();
    }

}
```

* **POST `/apps/{appId}/clusters/{clusterName}/namespaces/{namespaceName}/itemset` 接口**，Request Body 传递 **JSON** 对象。

## 5.2 ItemSetService

在 `apollo-biz` 项目中，`com.ctrip.framework.apollo.biz.service.ItemSetService` ，提供 Item  **批量** 的 **Service** 逻辑给 Admin Service 和 Config Service 。

`#updateSet(Namespace, ItemChangeSets)` 方法，批量更新 Namespace 下的 Item 们。代码如下：

```Java
  1: @Service
  2: public class ItemSetService {
  3: 
  4:     @Autowired
  5:     private AuditService auditService;
  6:     @Autowired
  7:     private CommitService commitService;
  8:     @Autowired
  9:     private ItemService itemService;
 10: 
 11:     @Transactional
 12:     public ItemChangeSets updateSet(Namespace namespace, ItemChangeSets changeSets) {
 13:         return updateSet(namespace.getAppId(), namespace.getClusterName(), namespace.getNamespaceName(), changeSets);
 14:     }
 15: 
 16:     @Transactional
 17:     public ItemChangeSets updateSet(String appId, String clusterName,
 18:                                     String namespaceName, ItemChangeSets changeSet) {
 19:         String operator = changeSet.getDataChangeLastModifiedBy();
 20:         ConfigChangeContentBuilder configChangeContentBuilder = new ConfigChangeContentBuilder();
 21:         // 保存 Item 们
 22:         if (!CollectionUtils.isEmpty(changeSet.getCreateItems())) {
 23:             for (ItemDTO item : changeSet.getCreateItems()) {
 24:                 Item entity = BeanUtils.transfrom(Item.class, item);
 25:                 entity.setDataChangeCreatedBy(operator);
 26:                 entity.setDataChangeLastModifiedBy(operator);
 27:                 // 保存 Item
 28:                 Item createdItem = itemService.save(entity);
 29:                 // 添加到 ConfigChangeContentBuilder 中
 30:                 configChangeContentBuilder.createItem(createdItem);
 31:             }
 32:             // 记录 Audit 到数据库中
 33:             auditService.audit("ItemSet", null, Audit.OP.INSERT, operator);
 34:         }
 35:         // 更新 Item 们
 36:         if (!CollectionUtils.isEmpty(changeSet.getUpdateItems())) {
 37:             for (ItemDTO item : changeSet.getUpdateItems()) {
 38:                 Item entity = BeanUtils.transfrom(Item.class, item);
 39:                 Item managedItem = itemService.findOne(entity.getId());
 40:                 if (managedItem == null) {
 41:                     throw new NotFoundException(String.format("item not found.(key=%s)", entity.getKey()));
 42:                 }
 43:                 Item beforeUpdateItem = BeanUtils.transfrom(Item.class, managedItem);
 44:                 // protect. only value,comment,lastModifiedBy,lineNum can be modified
 45:                 managedItem.setValue(entity.getValue());
 46:                 managedItem.setComment(entity.getComment());
 47:                 managedItem.setLineNum(entity.getLineNum());
 48:                 managedItem.setDataChangeLastModifiedBy(operator);
 49:                 // 更新 Item
 50:                 Item updatedItem = itemService.update(managedItem);
 51:                 // 添加到 ConfigChangeContentBuilder 中
 52:                 configChangeContentBuilder.updateItem(beforeUpdateItem, updatedItem);
 53:             }
 54:             // 记录 Audit 到数据库中
 55:             auditService.audit("ItemSet", null, Audit.OP.UPDATE, operator);
 56:         }
 57:         // 删除 Item 们
 58:         if (!CollectionUtils.isEmpty(changeSet.getDeleteItems())) {
 59:             for (ItemDTO item : changeSet.getDeleteItems()) {
 60:                 // 删除 Item
 61:                 Item deletedItem = itemService.delete(item.getId(), operator);
 62:                 // 添加到 ConfigChangeContentBuilder 中
 63:                 configChangeContentBuilder.deleteItem(deletedItem);
 64:             }
 65:             // 记录 Audit 到数据库中
 66:             auditService.audit("ItemSet", null, Audit.OP.DELETE, operator);
 67:         }
 68:         // 创建 Commit 对象，并保存
 69:         if (configChangeContentBuilder.hasContent()) {
 70:             createCommit(appId, clusterName, namespaceName, configChangeContentBuilder.build(), changeSet.getDataChangeLastModifiedBy());
 71:         }
 72:         return changeSet;
 73: 
 74:     }
 75: 
 76:     private void createCommit(String appId, String clusterName, String namespaceName, String configChangeContent,
 77:                               String operator) {
 78:         // 创建 Commit 对象
 79:         Commit commit = new Commit();
 80:         commit.setAppId(appId);
 81:         commit.setClusterName(clusterName);
 82:         commit.setNamespaceName(namespaceName);
 83:         commit.setChangeSets(configChangeContent);
 84:         commit.setDataChangeCreatedBy(operator);
 85:         commit.setDataChangeLastModifiedBy(operator);
 86:         // 保存 Commit 对象
 87:         commitService.save(commit);
 88:     }
 89: 
 90: }
```

* 第 21 至 34 行：**保存** Item 们。
* 第 35 至 56 行：**更新** Item 们。
    * 第 40 至 42 行：若更新的 Item 不存在，抛出  NotFoundException 异常，**事务回滚**。
* 第 57 至 67 行：**删除** Item 们。
    *  第 61 行：在 `ItemService#delete(long id, String operator)` 方法中，会**校验**删除的 Item 是否存在。若不存在，会抛出 IllegalArgumentException 异常，**事务回滚**。
* 第 69 至 71 行：调用 `ConfigChangeContentBuilder#hasContent()` 方法，判断若有变更，则调用 `#createCommit(appId, clusterName, namespaceName, configChangeContent, operator)` 方法，创建并保存 Commit 。

# 666. 彩蛋

ConfigTextResolver 的设计，值得我们在业务系统开发学习。🙂 很多时候，我们习惯性把大量的逻辑，全部写在 Service 类中。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)




