title: Apollo 源码解析 —— Portal 灰度发布
date: 2018-05-10
tags:
categories: Apollo
permalink: Apollo/portal-publish-namespace-branch

-------

摘要: 原创出处 http://www.iocoder.cn/Apollo/portal-publish-namespace-branch/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Apollo/portal-publish-namespace-branch/)
- [2. ReleaseService](http://www.iocoder.cn/Apollo/portal-publish-namespace-branch/)
  - [2.1 publishBranchNamespace](http://www.iocoder.cn/Apollo/portal-publish-namespace-branch/)
  - [2.2 mergeFromMasterAndPublishBranch](http://www.iocoder.cn/Apollo/portal-publish-namespace-branch/)
- [3. 加载灰度配置](http://www.iocoder.cn/Apollo/portal-publish-namespace-branch/)
  - [3.1 GrayReleaseRulesHolder](http://www.iocoder.cn/Apollo/portal-publish-namespace-branch/)
  - [3.2 GrayReleaseRuleCache](http://www.iocoder.cn/Apollo/portal-publish-namespace-branch/)
- [666. 彩蛋](http://www.iocoder.cn/Apollo/portal-publish-namespace-branch/)

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

**灰度发布**，实际上是**子** Namespace ( **分支** Namespace )发布 Release 。所以，调用的接口和 [《Apollo 源码解析 —— Portal 发布配置》](http://www.iocoder.cn/Apollo/portal-publish/?self) 是**一样的**。

**差异点**，在于 `apollo-biz` 项目中，`ReleaseService#publish(...)` 方法中，多了一个处理**灰度发布**的分支逻辑。

# 2. ReleaseService

## 2.1 publishBranchNamespace

`#publishBranchNamespace(...)` 方法，**子** Namespace 发布 Release 。**子** Namespace 会**自动继承** 父 Namespace **已经发布**的配置。若有相同的配置项，使用 **子** Namespace 的。配置处理的逻辑上，和**关联** Namespace 是一致的。代码如下：

```Java
  1: private Release publishBranchNamespace(Namespace parentNamespace, Namespace childNamespace,
  2:                                        Map<String, String> childNamespaceItems,
  3:                                        String releaseName, String releaseComment,
  4:                                        String operator, boolean isEmergencyPublish) {
  5:     // 获得父 Namespace 的最后有效 Release 对象
  6:     Release parentLatestRelease = findLatestActiveRelease(parentNamespace);
  7:     // 获得父 Namespace 的配置项
  8:     Map<String, String> parentConfigurations = parentLatestRelease != null ? gson.fromJson(parentLatestRelease.getConfigurations(), GsonType.CONFIG) : new HashMap<>();
  9:     // 获得父 Namespace 的 releaseId 属性
 10:     long baseReleaseId = parentLatestRelease == null ? 0 : parentLatestRelease.getId();
 11:     // 合并配置项
 12:     Map<String, String> childNamespaceToPublishConfigs = mergeConfiguration(parentConfigurations, childNamespaceItems);
 13:     // 发布子 Namespace 的 Release
 14:     return branchRelease(parentNamespace, childNamespace, releaseName, releaseComment,
 15:             childNamespaceToPublishConfigs, baseReleaseId, operator,
 16:             ReleaseOperation.GRAY_RELEASE, isEmergencyPublish);
 17: 
 18: }
```

* 第 5 至 12 行：获得最终的配置 Map 。
    * 第 6 行：调用 `#findLatestActiveRelease(parentNamespace)`  方法，获得**父** Namespace 的**最后有效** Release 对象。
    * 第 8 行：获得**父** Namespace 的配置 Map 。
    * 第 10 行：获得**父** Namespace 的 `releaseId` 属性。
    * 第 12 行：调用 `#mergeConfiguration(parentConfigurations, childNamespaceItems)` 方法，合并**父子** Namespace 的配置 Map 。代码如下：

        ```Java
        private Map<String, String> mergeConfiguration(Map<String, String> baseConfigurations, Map<String, String> coverConfigurations) {
            Map<String, String> result = new HashMap<>();
            // copy base configuration
            // 父 Namespace 的配置项
            for (Map.Entry<String, String> entry : baseConfigurations.entrySet()) {
                result.put(entry.getKey(), entry.getValue());
            }
            // update and publish
            // 子 Namespace 的配置项
            for (Map.Entry<String, String> entry : coverConfigurations.entrySet()) {
                result.put(entry.getKey(), entry.getValue());
            }
            // 返回合并后的配置项
            return result;
        }
        ```
        * x

* 第 14 行：调用 `#branchRelease(...)` 方法，发布**子** Namespace 的 Release 。代码如下： 

    ```Java
      1: private Release branchRelease(Namespace parentNamespace, Namespace childNamespace,
      2:                               String releaseName, String releaseComment,
      3:                               Map<String, String> configurations, long baseReleaseId,
      4:                               String operator, int releaseOperation, boolean isEmergencyPublish) {
      5:     // 获得父 Namespace 最后有效的 Release 对象
      6:     Release previousRelease = findLatestActiveRelease(childNamespace.getAppId(), childNamespace.getClusterName(), childNamespace.getNamespaceName());
      7:     // 获得父 Namespace 最后有效的 Release 对象的编号
      8:     long previousReleaseId = previousRelease == null ? 0 : previousRelease.getId();
      9: 
     10:     // 创建 Map ，用于 ReleaseHistory 对象的 `operationContext` 属性。
     11:     Map<String, Object> releaseOperationContext = Maps.newHashMap();
     12:     releaseOperationContext.put(ReleaseOperationContext.BASE_RELEASE_ID, baseReleaseId);
     13:     releaseOperationContext.put(ReleaseOperationContext.IS_EMERGENCY_PUBLISH, isEmergencyPublish);
     14: 
     15:     // 创建子 Namespace 的 Release 对象，并保存
     16:     Release release = createRelease(childNamespace, releaseName, releaseComment, configurations, operator);
     17: 
     18:     // 更新 GrayReleaseRule 的 releaseId 属性
     19:     // update gray release rules
     20:     GrayReleaseRule grayReleaseRule = namespaceBranchService.updateRulesReleaseId(childNamespace.getAppId(),
     21:             parentNamespace.getClusterName(),
     22:             childNamespace.getNamespaceName(),
     23:             childNamespace.getClusterName(),
     24:             release.getId(), operator);
     25: 
     26:     // 创建 ReleaseHistory 对象，并保存
     27:     if (grayReleaseRule != null) {
     28:         releaseOperationContext.put(ReleaseOperationContext.RULES, GrayReleaseRuleItemTransformer.batchTransformFromJSON(grayReleaseRule.getRules()));
     29:     }
     30:     releaseHistoryService.createReleaseHistory(parentNamespace.getAppId(), parentNamespace.getClusterName(),
     31:             parentNamespace.getNamespaceName(), childNamespace.getClusterName(),
     32:             release.getId(),
     33:             previousReleaseId, releaseOperation, releaseOperationContext, operator);
     34:     return release;
     35: }
    ```
    * 第 6 行：获得**父** Namespace **最后有效**的 Release 对象。
    * 第 8 行：获得**父** Namespace 的 `releaseId`  属性。
    * 第 10 至 13 行：创建 Map ，用于 ReleaseHistory 对象的 `operationContext` 属性。
    * 第 16 行：调用 `#createRelease(...)` 方法，创建**子** Namespace 的 Release 对象，并**保存**到数据库中。
    * 第 18 至 24 行：**更新** GrayReleaseRule 的 `releaseId` 属性到数据库中。代码如下：

        ```Java
        @Transactional
        public GrayReleaseRule updateRulesReleaseId(String appId, String clusterName, String namespaceName, String branchName, long latestReleaseId, String operator) {
            // 获得老的 GrayReleaseRule 对象
            GrayReleaseRule oldRules = grayReleaseRuleRepository.findTopByAppIdAndClusterNameAndNamespaceNameAndBranchNameOrderByIdDesc(appId, clusterName, namespaceName, branchName);
            if (oldRules == null) {
                return null;
            }
        
            // 创建新的 GrayReleaseRule 对象
            GrayReleaseRule newRules = new GrayReleaseRule();
            newRules.setBranchStatus(NamespaceBranchStatus.ACTIVE);
            newRules.setReleaseId(latestReleaseId); // update
            newRules.setRules(oldRules.getRules());
            newRules.setAppId(oldRules.getAppId());
            newRules.setClusterName(oldRules.getClusterName());
            newRules.setNamespaceName(oldRules.getNamespaceName());
            newRules.setBranchName(oldRules.getBranchName());
            newRules.setDataChangeCreatedBy(operator); // update
            newRules.setDataChangeLastModifiedBy(operator); // update
        
            // 保存新的 GrayReleaseRule 对象
            grayReleaseRuleRepository.save(newRules);
            // 删除老的 GrayReleaseRule 对象
            grayReleaseRuleRepository.delete(oldRules);
            return newRules;
        }
        ```
        * 删除**老的** GrayReleaseRule 对象。
        * 保存**新的** GrayReleaseRule 对象。
     
    * 第 26 至 33 行：调用 `ReleaseHistoryService#createReleaseHistory(...)` 方法，创建 ReleaseHistory 对象，并**保存**到数据库中。

## 2.2 mergeFromMasterAndPublishBranch

> 本小节不属于本文，考虑到和灰度发布相关，所以放在此处。

在**父** Namespace 发布 Release 后，会调用 `#mergeFromMasterAndPublishBranch(...)` 方法，自动将 *父* Namespace (主干) 合并到**子** Namespace (分支)，并进行一次子 Namespace 的发布。代码如下：

```Java
  1: private void mergeFromMasterAndPublishBranch(Namespace parentNamespace, Namespace childNamespace,
  2:                                              Map<String, String> parentNamespaceItems,
  3:                                              String releaseName, String releaseComment,
  4:                                              String operator, Release masterPreviousRelease,
  5:                                              Release parentRelease, boolean isEmergencyPublish) {
  6:     // 获得子 Namespace 的配置 Map
  7:     // create release for child namespace
  8:     Map<String, String> childReleaseConfiguration = getNamespaceReleaseConfiguration(childNamespace);
  9:     // 获得父 Namespace 的配置 Map
 10:     Map<String, String> parentNamespaceOldConfiguration = masterPreviousRelease == null ? null : gson.fromJson(masterPreviousRelease.getConfigurations(), GsonType.CONFIG);
 11: 
 12:     // 计算合并最新父 Namespace 的配置 Map 后的子 Namespace 的配置 Map
 13:     Map<String, String> childNamespaceToPublishConfigs = calculateChildNamespaceToPublishConfiguration(parentNamespaceOldConfiguration,
 14:                     parentNamespaceItems, childNamespace);
 15: 
 16:     // compare
 17:     // 若发生了变化，则进行一次子 Namespace 的发布
 18:     if (!childNamespaceToPublishConfigs.equals(childReleaseConfiguration)) {
 19:         branchRelease(parentNamespace, childNamespace, releaseName, releaseComment,
 20:                 childNamespaceToPublishConfigs, parentRelease.getId(), operator,
 21:                 ReleaseOperation.MASTER_NORMAL_RELEASE_MERGE_TO_GRAY, isEmergencyPublish);
 22:     }
 23: }
```

* 第 8 行：调用 `#getNamespaceReleaseConfiguration(childNamespace)` 方法，获得**子** Namespace 的**最新且有效的** Release 的配置 Map 。代码如下：

    ```Java
    private Map<String, String> getNamespaceReleaseConfiguration(Namespace namespace) {
        // 获得最后有效的 Release 对象
        Release release = findLatestActiveRelease(namespace);
        Map<String, String> configuration = new HashMap<>();
        // 获得配置 Map
        if (release != null) {
            configuration = new Gson().fromJson(release.getConfigurations(), GsonType.CONFIG);
        }
        return configuration;
    }
    ```

* 第 10 行：获得**父** Namespace 的配置 Map 。
* 第 12 至 14 行：计算**合并**最新父 Namespace 的配置 Map 后，子 Namespace 的配置 Map 。代码如下：

    ```Java
    // 计算合并最新父 Namespace 的配置 Map 后的子 Namespace 的配置 Map
    private Map<String, String> calculateChildNamespaceToPublishConfiguration(
            Map<String, String> parentNamespaceOldConfiguration, Map<String, String> parentNamespaceNewConfiguration,
            Namespace childNamespace) {
        // 获得子 Namespace 的最后有效的 Release 对象
        // first. calculate child namespace modified configs
        Release childNamespaceLatestActiveRelease = findLatestActiveRelease(childNamespace);
        // 获得子 Namespace 的配置 Map
        Map<String, String> childNamespaceLatestActiveConfiguration = childNamespaceLatestActiveRelease == null ? null :
                gson.fromJson(childNamespaceLatestActiveRelease.getConfigurations(), GsonType.CONFIG);
    
        // 以子 Namespace 的配置 Map 为基础，计算出差异的 Map
        Map<String, String> childNamespaceModifiedConfiguration = calculateBranchModifiedItemsAccordingToRelease(parentNamespaceOldConfiguration,
                childNamespaceLatestActiveConfiguration);
    
        // second. append child namespace modified configs to parent namespace new latest configuration
        return mergeConfiguration(parentNamespaceNewConfiguration, childNamespaceModifiedConfiguration);
    }
    
    // 以子 Namespace 的配置 Map 为基础，计算出差异的 Map
    private Map<String, String> calculateBranchModifiedItemsAccordingToRelease(
            Map<String, String> masterReleaseConfigs,
            Map<String, String> branchReleaseConfigs) {
        // 差异 Map
        Map<String, String> modifiedConfigs = new HashMap<>();
        // 若子 Namespace 的配置 Map 为空，直接返回空 Map
        if (CollectionUtils.isEmpty(branchReleaseConfigs)) {
            return modifiedConfigs;
        }
        // 若父 Namespace 的配置 Map 为空，直接返回子 Namespace 的配置 Map
        if (CollectionUtils.isEmpty(masterReleaseConfigs)) {
            return branchReleaseConfigs;
        }
    
        // 以子 Namespace 的配置 Map 为基础，计算出差异的 Map
        for (Map.Entry<String, String> entry : branchReleaseConfigs.entrySet()) {
            if (!Objects.equals(entry.getValue(), masterReleaseConfigs.get(entry.getKey()))) { // 对比
                modifiedConfigs.put(entry.getKey(), entry.getValue());
            }
        }
        return modifiedConfigs;
    }
    ```
    * 【第一步】逻辑看起来比较冗长和“绕”。简单的说，**子** Namespace 的配置 Map 是包含**老**的**父** Namespace 的配置 Map ，所以需要**剔除**。但是呢，剔除的过程中，又需要保留**子** Namespace 的**自定义**的配置项。这就是第二个方法，`#calculateBranchModifiedItemsAccordingToRelease(...)` 的逻辑。
    * 【第二步】做完上面的步骤后，就可以调用 `#mergeConfiguration(...)` 方法，合并**新**的**父** Namespace 的配置 Map 。
    * 胖友好好理解下。

* 第 17 至 22 行：若发生了变化，则调用 `#branchRelease(...)` 方法，进行一次**子** Namespace 的发布。这块就和 [「2.1 publishBranchNamespace」](#) 一致了。
    * 什么情况下会未发生变化呢？例如，**父** Namespace 修改配置项 `timeout: 2000=> 3000`  ，而恰好**子** Namespace 修改配置项 `timeout: 2000=> 3000` 并且已经灰度发布。

# 3. 加载灰度配置

在 [《Apollo 源码解析 —— Config Service 配置读取接口》](http://www.iocoder.cn/Apollo/config-service-config-query-api/?self) 中，我们看到 `AbstractConfigService#findRelease(...)` 方法中，会读取根据客户端的情况，匹配是否有**灰度 Release** ，代码如下：

```Java
  1: /**
  2:  * Find release
  3:  *
  4:  * 获得 Release 对象
  5:  *
  6:  * @param clientAppId       the client's app id
  7:  * @param clientIp          the client ip
  8:  * @param configAppId       the requested config's app id
  9:  * @param configClusterName the requested config's cluster name
 10:  * @param configNamespace   the requested config's namespace name
 11:  * @param clientMessages    the messages received in client side
 12:  * @return the release
 13:  */
 14: private Release findRelease(String clientAppId, String clientIp, String configAppId, String configClusterName,
 15:                             String configNamespace, ApolloNotificationMessages clientMessages) {
 16:     // 读取灰度发布编号
 17:     Long grayReleaseId = grayReleaseRulesHolder.findReleaseIdFromGrayReleaseRule(clientAppId, clientIp, configAppId, configClusterName, configNamespace);
 18:     // 读取灰度 Release 对象
 19:     Release release = null;
 20:     if (grayReleaseId != null) {
 21:         release = findActiveOne(grayReleaseId, clientMessages);
 22:     }
 23:     // 非灰度，获得最新的，并且有效的 Release 对象
 24:     if (release == null) {
 25:         release = findLatestActiveRelease(configAppId, configClusterName, configNamespace, clientMessages);
 26:     }
 27:     return release;
 28: }
```

* 第 17 行：调用 `GrayReleaseRulesHolder#findReleaseIdFromGrayReleaseRule(...)` 方法，读取灰度发布编号，即 `GrayReleaseRule.releaseId` 属性。详细解析，在 [「3.1 GrayReleaseRulesHolder」](#) 中。
* 第 18 至 22 行：调用 `#findActiveOne(grayReleaseId, clientMessages)` 方法，读取**灰度** Release 对象。

## 3.1 GrayReleaseRulesHolder

`com.ctrip.framework.apollo.biz.grayReleaseRule.GrayReleaseRulesHolder` ，实现 InitializingBean 和 ReleaseMessageListener 接口，GrayReleaseRule **缓存** Holder ，用于提高对 GrayReleaseRule 的读取速度。

### 3.1.1 构造方法

```Java
private static final Logger logger = LoggerFactory.getLogger(GrayReleaseRulesHolder.class);
    
private static final Joiner STRING_JOINER = Joiner.on(ConfigConsts.CLUSTER_NAMESPACE_SEPARATOR);
private static final Splitter STRING_SPLITTER = Splitter.on(ConfigConsts.CLUSTER_NAMESPACE_SEPARATOR).omitEmptyStrings();

@Autowired
private GrayReleaseRuleRepository grayReleaseRuleRepository;
@Autowired
private BizConfig bizConfig;

/**
 * 数据库扫描频率，单位：秒
 */
private int databaseScanInterval;
/**
 * ExecutorService 对象
 */
private ScheduledExecutorService executorService;
/**
 * GrayReleaseRuleCache 缓存
 *
 * KEY：configAppId+configCluster+configNamespace ，通过 {@link #assembleGrayReleaseRuleKey(String, String, String)} 生成
 *      注意，KEY 中不包含 BranchName
 * VALUE：GrayReleaseRuleCache 数组
 */
//store configAppId+configCluster+configNamespace -> GrayReleaseRuleCache map
private Multimap<String, GrayReleaseRuleCache> grayReleaseRuleCache;
/**
 * GrayReleaseRuleCache 缓存2
 *
 * KEY：clientAppId+clientNamespace+ip ，通过 {@link #assembleReversedGrayReleaseRuleKey(String, String, String)} 生成
 *      注意，KEY 中不包含 ClusterName
 * VALUE：{@link GrayReleaseRule#id} 数组
 */
//store clientAppId+clientNamespace+ip -> ruleId map
private Multimap<String, Long> reversedGrayReleaseRuleCache;
/**
 * 加载版本号
 */
// an auto increment version to indicate the age of rules
private AtomicLong loadVersion;

public GrayReleaseRulesHolder() {
    loadVersion = new AtomicLong();
    grayReleaseRuleCache = Multimaps.synchronizedSetMultimap(HashMultimap.create());
    reversedGrayReleaseRuleCache = Multimaps.synchronizedSetMultimap(HashMultimap.create());
    executorService = Executors.newScheduledThreadPool(1, ApolloThreadFactory.create("GrayReleaseRulesHolder", true));
}
```

* 缓存相关
    * GrayReleaseRuleCache ，胖友先去 [「3.2 GrayReleaseRuleCache」](#) ，在回过来。  
    * `grayReleaseRuleCache` 属性， GrayReleaseRuleCache 缓存。
        * KEY： `configAppId` + `configCluster` + `configNamespace` 拼接成，不包含 `branchName` 。因为我们在**匹配**灰度规则时，不关注 `branchName` 属性。
        * VALUE：GrayReleaseRuleCache 数组。因为 `branchName` 不包含在 KEY 中，而同一个 Namespace 可以创建多次灰度( 创建下一个需要将前一个灰度放弃 )版本，所以就会形成数组。
    * `reversedGrayReleaseRuleCache` 属性，**反转**的 GrayReleaseRuleCache 缓存。
        * KEY：`clientAppId` + `clientNamespace` + `ip` 。**注意**，不包含 `clusterName` 属性。具体原因，我们下面的 `#hasGrayReleaseRule(clientAppId, clientIp, namespaceName)` 方法中，详细分享。
        * VALUE：GrayReleaseRule 的编号数组。
        * 为什么叫做**反转**呢？因为使用 GrayReleaseRule 的具体属性作为键，而使用 GrayReleaseRule 的编号作为值。
    * 通过**定时**扫描 + ReleaseMessage **近实时**通知，更新缓存。

* 定时任务相关
    * `executorService` 属性，ExecutorService 对象。
    * `databaseScanInterval` 属性，数据库扫描频率，单位：秒。
    * `loadVersion` 属性，加载版本。

### 3.1.2 初始化

`#afterPropertiesSet()` 方法，通过 Spring 调用，初始化 Scan 任务。代码如下：

```Java
@Override
  1: public void afterPropertiesSet() throws Exception {
  2:     // 从 ServerConfig 中，读取任务的周期配置
  3:     populateDataBaseInterval();
  4:     // 初始拉取 GrayReleaseRuleCache 到缓存
  5:     // force sync load for the first time
  6:     periodicScanRules();
  7:     // 定时拉取 GrayReleaseRuleCache 到缓存
  8:     executorService.scheduleWithFixedDelay(this::periodicScanRules,
  9:             getDatabaseScanIntervalSecond(), getDatabaseScanIntervalSecond(), getDatabaseScanTimeUnit()
 10:     );
 11: }
```

* 第 3 行：调用 `#populateDataBaseInterval()` 方法，从 ServerConfig 中，读取定时任务的周期配置。代码如下：

    ```Java
    private void populateDataBaseInterval() {
        databaseScanInterval = bizConfig.grayReleaseRuleScanInterval(); // "apollo.gray-release-rule-scan.interval" ，默认为 60 。
    }
    ```

* 第 6 行：调用 `#periodicScanRules()` 方法，**初始**拉取 GrayReleaseRuleCache 到缓存。代码如下：

    ```Java
    private void periodicScanRules() {
        // 【TODO 6001】Tracer 日志
        Transaction transaction = Tracer.newTransaction("Apollo.GrayReleaseRulesScanner", "scanGrayReleaseRules");
        try {
            // 递增加载版本号
            loadVersion.incrementAndGet();
            // 从数据卷库中，扫描所有 GrayReleaseRules ，并合并到缓存中
            scanGrayReleaseRules();
            // 【TODO 6001】Tracer 日志
            transaction.setStatus(Transaction.SUCCESS);
        } catch (Throwable ex) {
            // 【TODO 6001】Tracer 日志
            transaction.setStatus(ex);
            logger.error("Scan gray release rule failed", ex);
        } finally {
            // 【TODO 6001】Tracer 日志
            transaction.complete();
        }
    }
    
    private void scanGrayReleaseRules() {
        long maxIdScanned = 0;
        boolean hasMore = true;
        // 循环顺序分批加载 GrayReleaseRule ，直到结束或者线程打断
        while (hasMore && !Thread.currentThread().isInterrupted()) {
            // 顺序分批加载 GrayReleaseRule 500 条
            List<GrayReleaseRule> grayReleaseRules = grayReleaseRuleRepository.findFirst500ByIdGreaterThanOrderByIdAsc(maxIdScanned);
            if (CollectionUtils.isEmpty(grayReleaseRules)) {
                break;
            }
            // 合并到 GrayReleaseRule 缓存
            mergeGrayReleaseRules(grayReleaseRules);
            // 获得新的 maxIdScanned ，取最后一条记录
            int rulesScanned = grayReleaseRules.size();
            maxIdScanned = grayReleaseRules.get(rulesScanned - 1).getId();
            // batch is 500
            // 若拉取不足 500 条，说明无 GrayReleaseRule 了
            hasMore = rulesScanned == 500;
        }
    }
    ```
    * 循环**顺序**、**分批**加载 GrayReleaseRule ，直到**全部加载完**或者线程打断。
    * `loadVersion` 属性，**递增**加载版本号。
    * 调用 `#mergeGrayReleaseRules(List<GrayReleaseRule>)` 方法，**合并** GrayReleaseRule 数组，到缓存中。详细解析，见 [「3.1.4 mergeGrayReleaseRules」](#) 。
    * 🙂 其他代码比较简单，胖友自己看代码注释。
* 第 7 至 10 行：创建定时任务，定时调用 `#scanGrayReleaseRules()` 方法，**重新全量**拉取 GrayReleaseRuleCache 到缓存。

### 3.1.3 handleMessage

`#handleMessage(ReleaseMessage, channel)` **实现**方法，基于 ReleaseMessage **近实时**通知，更新缓存。代码如下：

```Java
  1: @Override
  2: public void handleMessage(ReleaseMessage message, String channel) {
  3:     logger.info("message received - channel: {}, message: {}", channel, message);
  4:     String releaseMessage = message.getMessage();
  5:     // 只处理 APOLLO_RELEASE_TOPIC 的消息
  6:     if (!Topics.APOLLO_RELEASE_TOPIC.equals(channel) || Strings.isNullOrEmpty(releaseMessage)) {
  7:         return;
  8:     }
  9:     // 获得 appId cluster namespace 参数
 10:     List<String> keys = STRING_SPLITTER.splitToList(releaseMessage);
 11:     //message should be appId+cluster+namespace
 12:     if (keys.size() != 3) {
 13:         logger.error("message format invalid - {}", releaseMessage);
 14:         return;
 15:     }
 16:     String appId = keys.get(0);
 17:     String cluster = keys.get(1);
 18:     String namespace = keys.get(2);
 19: 
 20:     // 获得对应的 GrayReleaseRule 数组
 21:     List<GrayReleaseRule> rules = grayReleaseRuleRepository.findByAppIdAndClusterNameAndNamespaceName(appId, cluster, namespace);
 22:     // 合并到 GrayReleaseRule 缓存中
 23:     mergeGrayReleaseRules(rules);
 24: }
```

* 第 5 至 8 行：只处理 **APOLLO_RELEASE_TOPIC** 的消息。
* 第 9 至 18 行：获得 `appId` `cluster` `namespace` 参数。
* 第 21 行：调用 `grayReleaseRuleRepository#findByAppIdAndClusterNameAndNamespaceName(appId, cluster, namespace)` 方法，获得对应的 GrayReleaseRule 数组。
* 第 23 行：调用 `#mergeGrayReleaseRules(List<GrayReleaseRule>)` 方法，合并到 GrayReleaseRule 缓存中。详细解析，见 [「3.1.4 mergeGrayReleaseRules」](#) 。

### 3.1.4 mergeGrayReleaseRules

`#mergeGrayReleaseRules(List<GrayReleaseRule>)` 方法，合并 GrayReleaseRule 到缓存中。代码如下：

```Java
  1: private void mergeGrayReleaseRules(List<GrayReleaseRule> grayReleaseRules) {
  2:     if (CollectionUtils.isEmpty(grayReleaseRules)) {
  3:         return;
  4:     }
  5:     // !!! 注意，下面我们说的“老”，指的是已经在缓存中，但是实际不一定“老”。
  6:     for (GrayReleaseRule grayReleaseRule : grayReleaseRules) {
  7:         // 无对应的 Release 编号，记未灰度发布，则无视
  8:         if (grayReleaseRule.getReleaseId() == null || grayReleaseRule.getReleaseId() == 0) {
  9:             // filter rules with no release id, i.e. never released
 10:             continue;
 11:         }
 12:         // 创建 `grayReleaseRuleCache` 的 KEY
 13:         String key = assembleGrayReleaseRuleKey(grayReleaseRule.getAppId(), grayReleaseRule.getClusterName(), grayReleaseRule.getNamespaceName());
 14:         // 从缓存 `grayReleaseRuleCache` 读取，并创建数组，避免并发
 15:         // create a new list to avoid ConcurrentModificationException
 16:         List<GrayReleaseRuleCache> rules = Lists.newArrayList(grayReleaseRuleCache.get(key));
 17:         // 获得子 Namespace 对应的老的 GrayReleaseRuleCache 对象
 18:         GrayReleaseRuleCache oldRule = null;
 19:         for (GrayReleaseRuleCache ruleCache : rules) {
 20:             if (ruleCache.getBranchName().equals(grayReleaseRule.getBranchName())) {
 21:                 oldRule = ruleCache;
 22:                 break;
 23:             }
 24:         }
 25: 
 26:         // 忽略，若不存在老的 GrayReleaseRuleCache ，并且当前 GrayReleaseRule 对应的分支不处于激活( 有效 )状态
 27:         // if old rule is null and new rule's branch status is not active, ignore
 28:         if (oldRule == null && grayReleaseRule.getBranchStatus() != NamespaceBranchStatus.ACTIVE) {
 29:             continue;
 30:         }
 31: 
 32:         // 若新的 GrayReleaseRule 为新增或更新，进行缓存更新
 33:         // use id comparison to avoid synchronization
 34:         if (oldRule == null || grayReleaseRule.getId() > oldRule.getRuleId()) {
 35:             // 添加新的 GrayReleaseRuleCache 到缓存中
 36:             addCache(key, transformRuleToRuleCache(grayReleaseRule));
 37:             // 移除老的 GrayReleaseRuleCache 出缓存中
 38:             if (oldRule != null) {
 39:                 removeCache(key, oldRule);
 40:             }
 41:         } else {
 42:             // 老的 GrayReleaseRuleCache 对应的分支处于激活( 有效 )状态，更新加载版本号。
 43:             // 例如，定时轮询，有可能，早于 `#handleMessage(...)` 拿到对应的新的 GrayReleaseRule 记录，那么此时规则编号是相等的，不符合上面的条件，但是符合这个条件。
 44:             // 再例如，两次定时轮询，第二次和第一次的规则编号是相等的，不符合上面的条件，但是符合这个条件。
 45:             if (oldRule.getBranchStatus() == NamespaceBranchStatus.ACTIVE) {
 46:                 // update load version
 47:                 oldRule.setLoadVersion(loadVersion.get());
 48:             // 保留两轮，
 49:             // 适用于，`GrayReleaseRule.branchStatus` 为 DELETED 或 MERGED 的情况。
 50:             } else if ((loadVersion.get() - oldRule.getLoadVersion()) > 1) {
 51:                 // remove outdated inactive branch rule after 2 update cycles
 52:                 removeCache(key, oldRule);
 53:             }
 54:         }
 55:     }
 56: }
```

* 第 5 行：**!!! 注意**，下面我们说的“老”，指的是已经在缓存中，但是实际不一定“老”。
* 第 6 行：**循环** GrayReleaseRule 数组，合并到缓存中。被缓存到的 GrayReleaseRule 对象，我们成为“**新**”的。
* 第 7 至 11 行：**无视**，若 GrayReleaseRule 无对应的 Release 编号，说明该**子** Namespace 还未灰度发布。
* 第 12 至 24 行：获得子 Namespace 对应的**老**的 GrayReleaseRuleCache 对象。此处的“老”，指的是**缓存中**的。
* 第 26 至 30 行：**无视**，若不存在老的 GrayReleaseRuleCache ，并且当前 GrayReleaseRule 对应的分支不处于激活( **ACTIVE** 有效 )状态。
* 第 32 至 40 行：若**新**的 GrayReleaseRule 为新增或更新( 编号**更大** )，进行缓存更新，并移除**老**的 GrayReleaseRule 出缓存。
    * 第 36 行：调用 `transformRuleToRuleCache(GrayReleaseRule)` 方法，将 GrayReleaseRule 转换成 GrayReleaseRuleCache 对象。详细解析，见 [「3.1.4.1 transformRuleToRuleCache」](#) 。
    * 第 36 行：调用 `#addCache(key, GrayReleaseRuleCache)` 方法，添加**新**的 GrayReleaseRuleCache 到缓存中。详细解析，见 [「3.1.4.2 addCache」](#) 。
    * 第 37 至 40 行：调用 `#remove(key, oldRule)` 方法，移除**老** 的 GrayReleaseRuleCache 出缓存。详细解析，见 [「3.1.4.3 removeCache」](#) 。
* 第 42 至 47 行：**老**的 GrayReleaseRuleCache 对应的分支处于激活( 有效 )状态，更新加载版本号。
    * 例如，定时轮询，有可能，早于 `#handleMessage(...)` 拿到对应的新的 GrayReleaseRule 记录，那么此时规则编号是相等的，不符合上面的条件，但是符合这个条件。
    *  再例如，两次定时轮询，第二次和第一次的规则编号是相等的，不符合上面的条件，但是符合这个条件。
    *  **总结**，刷新有效的 GrayReleaseRuleCache 对象的 `loadVersion` 。
*  第 50 至 53 行：若 `GrayReleaseRule.branchStatus` 为 DELETED 或 MERGED 的情况，保留两轮定时扫描，后调用 `#remove(key, oldRule)` 方法，移除出缓存。
    * 例如，灰度全量发布时，会添加 `GrayReleaseRule.branchStatus` 为 **MERGED** 到缓存中。保留两轮，进行移除出缓存。
    * 为什么是**两轮**？笔者请教了宋老师( Apollo 的作者之一 )，解答如下：

        > 这个是把已经inactive的rule删除，至于为啥保留两轮，这个可能只是个选择问题
        * T T 笔者表示还是不太明白，继续思考ing 。如果有知道的胖友，烦请告知。

#### 3.1.4.1 transformRuleToRuleCache

`#transformRuleToRuleCache(GrayReleaseRule)` 方法，将 GrayReleaseRule 转换成 GrayReleaseRuleCache 对象。代码如下：

```Java
private GrayReleaseRuleCache transformRuleToRuleCache(GrayReleaseRule grayReleaseRule) {
    // 转换出 GrayReleaseRuleItemDTO 数组
    Set<GrayReleaseRuleItemDTO> ruleItems;
    try {
        ruleItems = GrayReleaseRuleItemTransformer.batchTransformFromJSON(grayReleaseRule.getRules());
    } catch (Throwable ex) {
        ruleItems = Sets.newHashSet();
        Tracer.logError(ex);
        logger.error("parse rule for gray release rule {} failed", grayReleaseRule.getId(), ex);
    }
    // 创建 GrayReleaseRuleCache 对象，并返回
    return new GrayReleaseRuleCache(grayReleaseRule.getId(),
            grayReleaseRule.getBranchName(), grayReleaseRule.getNamespaceName(), grayReleaseRule
            .getReleaseId(), grayReleaseRule.getBranchStatus(), loadVersion.get(), ruleItems);
}
```

#### 3.1.4.2 addCache

`#addCache(key, GrayReleaseRuleCache)` 方法，添加**新**的 GrayReleaseRuleCache 到缓存中。代码如下：

```Java
  1: private void addCache(String key, GrayReleaseRuleCache ruleCache) {
  2:     // 添加到 reversedGrayReleaseRuleCache 中
  3:     // 为什么这里判断状态？因为删除灰度，或者灰度全量发布的情况下，是无效的，所以不添加到 reversedGrayReleaseRuleCache 中
  4:     if (ruleCache.getBranchStatus() == NamespaceBranchStatus.ACTIVE) {
  5:         for (GrayReleaseRuleItemDTO ruleItemDTO : ruleCache.getRuleItems()) {
  6:             for (String clientIp : ruleItemDTO.getClientIpList()) {
  7:                 reversedGrayReleaseRuleCache.put(assembleReversedGrayReleaseRuleKey(ruleItemDTO.getClientAppId(), ruleCache.getNamespaceName(), clientIp),
  8:                         ruleCache.getRuleId());
  9:             }
 10:         }
 11:     }
 12:     // 添加到 grayReleaseRuleCache
 13:     // 这里为什么可以添加？因为添加到 grayReleaseRuleCache 中是个对象，可以判断状态
 14:     grayReleaseRuleCache.put(key, ruleCache);
 15: }
```

* 第 2 至 11 行：添加到 `reversedGrayReleaseRuleCache` 中。
    * 为什么这里**判断状态**？因为删除灰度，或者灰度全量发布的情况下，是无效的，所以不添加到 `reversedGrayReleaseRuleCache` 中。
* 第 14 行：添加到 `grayReleaseRuleCache` 中。
    *  为什么这里**可以添加**？因为添加到 `grayReleaseRuleCache` 中是个对象，可以判断状态。

#### 3.1.4.3 removeCache

`#remove(key, oldRule)` 方法，移除**老** 的 GrayReleaseRuleCache 出缓存。代码如下：

```Java
private void removeCache(String key, GrayReleaseRuleCache ruleCache) {
    // 移除出 grayReleaseRuleCache
    grayReleaseRuleCache.remove(key, ruleCache);
    // 移除出 reversedGrayReleaseRuleCache
    for (GrayReleaseRuleItemDTO ruleItemDTO : ruleCache.getRuleItems()) {
        for (String clientIp : ruleItemDTO.getClientIpList()) {
            reversedGrayReleaseRuleCache.remove(assembleReversedGrayReleaseRuleKey(ruleItemDTO.getClientAppId(), ruleCache.getNamespaceName(), clientIp),
                    ruleCache.getRuleId());
        }
    }
}
```

### 3.1.5 findReleaseIdFromGrayReleaseRule

`#findReleaseIdFromGrayReleaseRule(clientAppId, clientIp, configAppId, configCluster, configNamespaceName)` 方法，若匹配上灰度规则，返回对应的 Release 编号。代码如下：

```Java
public Long findReleaseIdFromGrayReleaseRule(String clientAppId, String clientIp, String
        configAppId, String configCluster, String configNamespaceName) {
    // 判断 grayReleaseRuleCache 中是否存在
    String key = assembleGrayReleaseRuleKey(configAppId, configCluster, configNamespaceName);
    if (!grayReleaseRuleCache.containsKey(key)) {
        return null;
    }
    // 循环 GrayReleaseRuleCache 数组，获得匹配的 Release 编号
    // create a new list to avoid ConcurrentModificationException
    List<GrayReleaseRuleCache> rules = Lists.newArrayList(grayReleaseRuleCache.get(key));
    for (GrayReleaseRuleCache rule : rules) {
        // 校验 GrayReleaseRuleCache 对应的子 Namespace 的状态是否为有效
        //check branch status
        if (rule.getBranchStatus() != NamespaceBranchStatus.ACTIVE) {
            continue;
        }
        // 是否匹配灰度规则。若是，则返回。
        if (rule.matches(clientAppId, clientIp)) {
            return rule.getReleaseId();
        }
    }
    return null;
}
```

* 🙂 代码比较易懂，胖友自己看代码注释哈。

### 3.1.6 hasGrayReleaseRule

```Java
/**
 * Check whether there are gray release rules for the clientAppId, clientIp, namespace
 * combination. Please note that even there are gray release rules, it doesn't mean it will always
 * load gray releases. Because gray release rules actually apply to one more dimension - cluster.
 */
public boolean hasGrayReleaseRule(String clientAppId, String clientIp, String namespaceName) {
    return reversedGrayReleaseRuleCache.containsKey(assembleReversedGrayReleaseRuleKey(clientAppId, namespaceName, clientIp))
            || reversedGrayReleaseRuleCache.containsKey(assembleReversedGrayReleaseRuleKey(clientAppId, namespaceName, GrayReleaseRuleItemDTO.ALL_IP));
}
```

* 我们来翻一下英文注释哈，非**直译**哈。
* 【一】Check whether there are gray release rules for the clientAppId, clientIp, namespace combination. 针对 `clientAppId` + `clientIp` + `namespaceName` ，校验是否有灰度规则。
* 【二】Please note that even there are gray release rules, it doesn't mean it will always load gray releases. 请注意，即使返回 `true` ，也不意味着调用方能加载到灰度发布的配置。
* 【三】 Because gray release rules actually apply to one more dimension - cluster. 因为，`reversedGrayReleaseRuleCache` 的 KEY 不包含 `branchName` ，所以 `reversedGrayReleaseRuleCache` 的 VALUE 为**多个** `branchName` 的 Release 编号的**集合**。
* 那么为什么不包含 `branchName` 呢？在 [《Apollo 源码解析 —— Config Service 配置读取接口》](http://www.iocoder.cn/Apollo/config-service-config-query-api/?self) 一文中，我们看到 AbstractConfigService 中，`#loadConfig(...)` 方法中，是按照**集群**的**优先级**加载，代码如下：

    ```Java
    @Override
    public Release loadConfig(String clientAppId, String clientIp, String configAppId, String configClusterName,
                              String configNamespace, String dataCenter, ApolloNotificationMessages clientMessages) {
        // 优先，获得指定 Cluster 的 Release 。若存在，直接返回。
        // load from specified cluster fist
        if (!Objects.equals(ConfigConsts.CLUSTER_NAME_DEFAULT, configClusterName)) {
            Release clusterRelease = findRelease(clientAppId, clientIp, configAppId, configClusterName, configNamespace,
                    clientMessages);
            if (!Objects.isNull(clusterRelease)) {
                return clusterRelease;
            }
        }
    
        // 其次，获得所属 IDC 的 Cluster 的 Release 。若存在，直接返回
        // try to load via data center
        if (!Strings.isNullOrEmpty(dataCenter) && !Objects.equals(dataCenter, configClusterName)) {
            Release dataCenterRelease = findRelease(clientAppId, clientIp, configAppId, dataCenter, configNamespace, clientMessages);
            if (!Objects.isNull(dataCenterRelease)) {
                return dataCenterRelease;
            }
        }
    
        // 最后，获得默认 Cluster 的 Release 。
        // fallback to default release
        return findRelease(clientAppId, clientIp, configAppId, ConfigConsts.CLUSTER_NAME_DEFAULT, configNamespace, clientMessages);
    }
    ```
    * 但是，笔者又想了想，应该也不是这个方法的原因，因为这个方法里，每个调用的方法，`clusterName` 是明确的，那么把 `clusterName` 融入到缓存 KEY 也是可以的。**所以应该不是这个原因**。
    
* 目前 `#hasGrayReleaseRule(clientAppId, clientIp, namespaceName)` 方法，仅仅被 ConfigFileController 调用。而 ConfigFileController 在调用时，确实是不知道自己使用哪个 `clusterName` 。恩恩，应该是这个原因。

## 3.2 GrayReleaseRuleCache

`com.ctrip.framework.apollo.biz.grayReleaseRule.GrayReleaseRuleCache` ，GrayReleaseRule 的缓存类。代码如下：

```Java
private long ruleId;

// 缺少 appId
// 缺少 clusterName

private String namespaceName;
private String branchName;
private Set<GrayReleaseRuleItemDTO> ruleItems;
private long releaseId;
private int branchStatus;
/**
 * 加载版本
 */
private long loadVersion;
    
// 匹配 clientAppId + clientIp
public boolean matches(String clientAppId, String clientIp) {
    for (GrayReleaseRuleItemDTO ruleItem : ruleItems) {
        if (ruleItem.matches(clientAppId, clientIp)) {
            return true;
        }
    }
    return false;
}
```

相比 GrayReleaseRule 来说：

* **少**了 `appId` + `clusterName` 字段，因为在 GrayReleaseRulesHolder 中，缓存 KEY 会根据需要包含这两个字段。
* **多**了 `loadVersion` 字段，用于记录 GrayReleaseRuleCache 的加载版本，用于自动过期逻辑。

# 666. 彩蛋

T T ，GrayReleaseRulesHolder 看的还是有点懵逼，后面自己有机会写配置中心的灰度发布功能的时候，在捉摸捉摸。如果有些写的不对的地方，欢迎指正。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)



