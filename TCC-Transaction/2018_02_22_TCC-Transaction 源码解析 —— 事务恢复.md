title: TCC-Transaction 源码分析 —— 事务恢复
date: 2018-02-22
tags:
categories: TCC-Transaction
permalink: TCC-Transaction/transaction-recovery

---

**本文主要基于 TCC-Transaction 1.2.3.3 正式版**  

- [1. 概述](#)
- [2. 事务重试配置](#)
- [3. 事务重试定时任务](#)
- [4. 异常事务恢复](#)
	- [4.1 加载异常事务集合](#)
	- [4.2 恢复异常事务集合](#)
- [666. 彩蛋](#)

---

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

---

# 1. 概述

本文分享 **TCC 恢复**。主要涉及如下二个 package 路径下的类：

* `org.mengyun.tcctransaction.recover`
    * RecoverConfig，事务恢复配置**接口**  
    * TransactionRecovery，事务恢复逻辑
* `org.mengyun.tcctransaction.spring.recover` ：
    * DefaultRecoverConfig，默认事务恢复配置**实现**
    * RecoverScheduledJob，事务恢复定时任务

本文涉及到的类关系如下图( [打开大图](http://www.iocoder.cn/images/TCC-Transaction/2018_02_22/01.png) )：

![](http://www.iocoder.cn/images/TCC-Transaction/2018_02_22/01.png)

在[《TCC-Transaction 源码分析 —— 事务存储器》](http://www.iocoder.cn/TCC-Transaction/transaction-repository/?self)中，事务信息被持久化到外部的存储器中。**事务存储是事务恢复的基础**。通过读取外部存储器中的异常事务，定时任务会按照一定频率对事务进行重试，直到事务完成或超过最大重试次数。

![](http://www.iocoder.cn/images/TCC-Transaction/2018_02_15/01.png)

> 你行好事会因为得到赞赏而愉悦  
> 同理，开源项目贡献者会因为 Star 而更加有动力  
> 为 TCC-Transaction 点赞！[传送门](https://github.com/changmingxie/tcc-transaction)

ps：笔者假设你已经阅读过[《tcc-transaction 官方文档 —— 使用指南1.2.x》](https://github.com/changmingxie/tcc-transaction/wiki/%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%971.2.x)。

# 2. 事务重试配置

`org.mengyun.tcctransaction.recover.RecoverConfig`，事务恢复配置**接口**，实现代码如下：

```Java
public interface RecoverConfig {

    /**
     * @return 最大重试次数
     */
    int getMaxRetryCount();

    /**
     * @return 恢复间隔时间，单位：秒
     */
    int getRecoverDuration();

    /**
     * @return cron 表达式
     */
    String getCronExpression();

    /**
     * @return 延迟取消异常集合
     */
    Set<Class<? extends Exception>> getDelayCancelExceptions();

    /**
     * 设置延迟取消异常集合
     *
     * @param delayRecoverExceptions 延迟取消异常集合
     */
    void setDelayCancelExceptions(Set<Class<? extends Exception>> delayRecoverExceptions);
    
}
```

* `#getMaxRetryCount()`，单个事务恢复最大重试次数。超过最大重试次数后，目前仅打出错误日志，下文会看到实现。
* `#getRecoverDuration()`，单个事务恢复重试的间隔时间，单位：秒。
* `#getCronExpression()`，定时任务 cron 表达式。
* `#getDelayCancelExceptions()`，延迟取消异常集合。

-------

`org.mengyun.tcctransaction.spring.recover.DefaultRecoverConfig`，**默认**事务恢复配置实现，实现代码如下：

```Java
public class DefaultRecoverConfig implements RecoverConfig {

    public static final RecoverConfig INSTANCE = new DefaultRecoverConfig();

    /**
     * 最大重试次数
     */
    private int maxRetryCount = 30;

    /**
     * 恢复间隔时间，单位：秒
     */
    private int recoverDuration = 120;

    /**
     * cron 表达式
     */
    private String cronExpression = "0 */1 * * * ?";

    /**
     * 延迟取消异常集合
     */
    private Set<Class<? extends Exception>> delayCancelExceptions = new HashSet<Class<? extends Exception>>();

    public DefaultRecoverConfig() {
        delayCancelExceptions.add(OptimisticLockException.class);
        delayCancelExceptions.add(SocketTimeoutException.class);
    }
    
    @Override
    public void setDelayCancelExceptions(Set<Class<? extends Exception>> delayCancelExceptions) {
        this.delayCancelExceptions.addAll(delayCancelExceptions);
    }
    
}
```

* `maxRetryCount`，单个事务恢复最大重试次数 为 30。
* `recoverDuration`，单个事务恢复重试的间隔时间为 120 秒。
* `cronExpression`，定时任务 cron 表达式为 `"0 */1 * * * ?"`，每分钟执行一次。如果你希望定时任务执行的更频繁，可以修改 cron 表达式，例如 `0/30 * * * * ?`，每 30 秒执行一次。
* `delayCancelExceptions`，延迟取消异常集合。在 DefaultRecoverConfig 构造方法里，预先添加了 OptimisticLockException / SocketTimeoutException 。
    * 针对 **SocketTimeoutException** ：try 阶段，本地参与者调用远程参与者( 远程服务，例如 Dubbo，Http 服务)，远程参与者 try 阶段的方法逻辑执行时间较长，超过 Socket 等待时长，发生 SocketTimeoutException，如果立刻执行事务回滚，远程参与者 try 的方法未执行完成，可能导致 cancel 的方法实际未执行( try 的方法未执行完成，数据库事务【非 TCC 事务】未提交，cancel 的方法读取数据时发现未变更，导致方法实际未执行，最终 try 的方法执行完后，提交数据库事务【非 TCC 事务】，较为极端 )，最终引起数据不一致。在**事务恢复**时，会对这种情况的事务进行取消回滚，如果此时远程参与者的 try 的方法还未结束，还是可能发生数据不一致。
        * 官方解释：[为什么 tcc 事务切面中对乐观锁与socket超时异常不做回滚处理，只抛异常？](https://github.com/changmingxie/tcc-transaction/issues/87)
    * 针对 OptimisticLockException ：还是 SocketTimeoutException 的情况，事务恢复间隔时间小于 Socket 超时时间，此时事务恢复调用远程参与者取消回滚事务，远程参与者下次更新事务时，会因为乐观锁更新失败，抛出 OptimisticLockException。如果 CompensableTransactionInterceptor 此时立刻取消回滚，可能会和定时任务的取消回滚冲突，因此统一交给定时任务处理。
        * 官方解释：[事务恢复的疑问](https://github.com/changmingxie/tcc-transaction/issues/53)
        * 这块笔者还有一些疑问，如果有别的可能性导致这个情况，麻烦告知下笔者。谢谢。

# 3. 事务重试定时任务

`org.mengyun.tcctransaction.spring.recover.RecoverScheduledJob`，事务恢复定时任务，基于 Quartz 实现调度，不断不断不断执行事务恢复。实现代码如下：

```Java
public class RecoverScheduledJob {

    private TransactionRecovery transactionRecovery;

    private TransactionConfigurator transactionConfigurator;

    private Scheduler scheduler;

    public void init() {
        try {
            // Quartz JobDetail
            MethodInvokingJobDetailFactoryBean jobDetail = new MethodInvokingJobDetailFactoryBean();
            jobDetail.setTargetObject(transactionRecovery);
            jobDetail.setTargetMethod("startRecover");
            jobDetail.setName("transactionRecoveryJob");
            jobDetail.setConcurrent(false); // 禁止并发
            jobDetail.afterPropertiesSet();
            // Quartz CronTriggerFactoryBean
            CronTriggerFactoryBean cronTrigger = new CronTriggerFactoryBean();
            cronTrigger.setBeanName("transactionRecoveryCronTrigger");
            cronTrigger.setCronExpression(transactionConfigurator.getRecoverConfig().getCronExpression());
            cronTrigger.setJobDetail(jobDetail.getObject());
            cronTrigger.afterPropertiesSet();
            // 启动任务调度
            scheduler.scheduleJob(jobDetail.getObject(), cronTrigger.getObject());
            // 启动 Quartz Scheduler
            scheduler.start();
        } catch (Exception e) {
            throw new SystemException(e);
        }
    }
}
```

* 调用 `MethodInvokingJobDetailFactoryBean#setConcurrent(false)` 方法，禁用任务并发执行。
* 调用 `MethodInvokingJobDetailFactoryBean#setTargetObject(...)` + `MethodInvokingJobDetailFactoryBean#setTargetMethod(...)` 方法，设置任务调用 `TransactionRecovery#startRecover(...)` 方法执行。

**如果应用集群部署，会不会相同事务被多个定时任务同时重试**？

答案是不会，事务在重试时会乐观锁更新，同时只有一个应用节点能更新成功。

官方解释：[多机部署下，所有机器都宕机，从异常中恢复时，所有的机器岂不是都可以查询到所有的需要恢复的服务？](https://github.com/changmingxie/tcc-transaction/issues/98)

当然极端情况下，Socket 调用超时时间大于事务重试间隔，第一个节点在重试某个事务，一直未执行完成，第二个节点已经可以重试。

ps：建议，Socket 调用超时时间小于事务重试间隔。

**是否定时任务和应用服务器解耦**？

蚂蚁金服的分布式事务服务 DTS 采用 client-server 模式：

* xts-client ：负责事务的创建、提交、回滚、记录。
* xts-server ：负责异常事务的恢复。

> FROM [《蚂蚁金融云 DTS 文档》](https://www.cloud.alipay.com/docs/2/46887)   
> 分布式事务服务 (Distributed Transaction Service, DTS) 是一个分布式事务框架，用来保障在大规模分布式环境下事务的最终一致性。DTS 从架构上分为 xts-client 和 xts-server 两部分，前者是一个嵌入客户端应用的 JAR 包，主要负责事务数据的写入和处理；后者是一个独立的系统，主要负责异常事务的恢复。

# 4. 异常事务恢复

 `org.mengyun.tcctransaction.recover.TransactionRecovery`，异常事务恢复，实现主体代码如下：
 
 ```Java
public class TransactionRecovery {

    /**
     * 启动恢复事务逻辑
     */
    public void startRecover() {
        // 加载异常事务集合
        List<Transaction> transactions = loadErrorTransactions();
        // 恢复异常事务集合
        recoverErrorTransactions(transactions);
    }

}
 ```
 
## 4.1 加载异常事务集合

调用 `#loadErrorTransactions()` 方法，加载异常事务集合。实现代码如下：

```Java
private List<Transaction> loadErrorTransactions() {
   TransactionRepository transactionRepository = transactionConfigurator.getTransactionRepository();
   long currentTimeInMillis = Calendar.getInstance().getTimeInMillis();
   RecoverConfig recoverConfig = transactionConfigurator.getRecoverConfig();
   return transactionRepository.findAllUnmodifiedSince(new Date(currentTimeInMillis - recoverConfig.getRecoverDuration() * 1000));
}
```

* **异常事务的定义**：当前时间超过 - 事务变更时间( 最后执行时间 ) >= 事务恢复间隔( `RecoverConfig#getRecoverDuration()` )。这里有一点要注意，已完成的事务会从事务存储器删除。

## 4.2 恢复异常事务集合

调用 `#recoverErrorTransactions(...)` 方法，恢复异常事务集合。实现代码如下：

```Java
private void recoverErrorTransactions(List<Transaction> transactions) {
   for (Transaction transaction : transactions) {
       // 超过最大重试次数
       if (transaction.getRetriedCount() > transactionConfigurator.getRecoverConfig().getMaxRetryCount()) {
           logger.error(String.format("recover failed with max retry count,will not try again. txid:%s, status:%s,retried count:%d,transaction content:%s", transaction.getXid(), transaction.getStatus().getId(), transaction.getRetriedCount(), JSON.toJSONString(transaction)));
           continue;
       }
       // 分支事务超过最大可重试时间
       if (transaction.getTransactionType().equals(TransactionType.BRANCH)
               && (transaction.getCreateTime().getTime() +
               transactionConfigurator.getRecoverConfig().getMaxRetryCount() *
                       transactionConfigurator.getRecoverConfig().getRecoverDuration() * 1000
               > System.currentTimeMillis())) {
           continue;
       }
       // Confirm / Cancel
       try {
           // 增加重试次数
           transaction.addRetriedCount();
           // Confirm
           if (transaction.getStatus().equals(TransactionStatus.CONFIRMING)) {
               transaction.changeStatus(TransactionStatus.CONFIRMING);
               transactionConfigurator.getTransactionRepository().update(transaction);
               transaction.commit();
               transactionConfigurator.getTransactionRepository().delete(transaction);
           // Cancel
           } else if (transaction.getStatus().equals(TransactionStatus.CANCELLING)
                   || transaction.getTransactionType().equals(TransactionType.ROOT)) { // 处理延迟取消的情况
               transaction.changeStatus(TransactionStatus.CANCELLING);
               transactionConfigurator.getTransactionRepository().update(transaction);
               transaction.rollback();
               transactionConfigurator.getTransactionRepository().delete(transaction);
           }
       } catch (Throwable throwable) {
           if (throwable instanceof OptimisticLockException
                   || ExceptionUtils.getRootCause(throwable) instanceof OptimisticLockException) {
               logger.warn(String.format("optimisticLockException happened while recover. txid:%s, status:%s,retried count:%d,transaction content:%s", transaction.getXid(), transaction.getStatus().getId(), transaction.getRetriedCount(), JSON.toJSONString(transaction)), throwable);
           } else {
               logger.error(String.format("recover failed, txid:%s, status:%s,retried count:%d,transaction content:%s", transaction.getXid(), transaction.getStatus().getId(), transaction.getRetriedCount(), JSON.toJSONString(transaction)), throwable);
           }
       }
   }
}
```

* 当单个事务超过最大重试次数时，不再重试，只打印异常，此时需要**人工介入**解决。可以接入 ELK 收集日志监控报警。
* 当**分支事务**超过最大可重试时间时，不再重试。可能有同学和我一开始理解的是相同的，实际**分支事务**对应的应用服务器也可以重试**分支事务**，不是必须**根事务**发起重试，从而一起重试**分支事务**。这点要注意下。
* 当事务处于 TransactionStatus.CONFIRMING 状态时，提交事务，逻辑和 `TransactionManager#commit()` 类似。
* 当事务处于 TransactionStatus.CONFIRMING 状态，或者**事务类型为根事务**，回滚事务，逻辑和 `TransactionManager#rollback()` 类似。这里加判断的**事务类型为根事务**，用于处理延迟回滚异常的事务的回滚。

# 666. 彩蛋

在写本文的过程中，无意中翻到蚂蚁云的文档，分享给看到此处的真爱们。

真爱们，请猛击[《AntCloudPayPublic》](https://git.cloud.alipay.com/dx/AntCloudPayPublic)跳转。

![](http://www.iocoder.cn/images/TCC-Transaction/2018_02_22/02.png)

胖友，分享一个朋友圈可好？


