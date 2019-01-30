title: Fescar 源码走读（2）之 Fescar 服务端
date: 2019-03-04
tags:
categories: Fescar
permalink: Fescar/yuanxiang/2-Fescar-Server
author: yuanxiang
from_url: https://zhuanlan.zhihu.com/p/54660611
wechat_url:

-------

摘要: 原创出处 https://zhuanlan.zhihu.com/p/54660611 「yuanxiang」欢迎转载，保留摘要，谢谢！


-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------


服务端启动：

```java
    public static void main(String[] args) throws IOException {
        RpcServer rpcServer = new RpcServer(WORKING_THREADS);

        if (args.length > 0) {
            int port = Integer.parseInt(args[0]);
            rpcServer.setListenPort(port);

        }
        String dataDir = null;
        if (args.length > 1) {
            dataDir = args[1];
        }
        SessionHolder.init(dataDir);
//协调者初始化，不清楚tc,tm,rm关系的可以简单看些git上的介绍
        DefaultCoordinator coordinator = new DefaultCoordinator(rpcServer);
        coordinator.init();
        rpcServer.setHandler(new DefaultCoordinator(rpcServer));
//全局事务发号器初始化
        UUIDGenerator.init(1);

        XID.setIpAddress(NetUtil.getLocalIp());
        XID.setPort(rpcServer.getListenPort());

        rpcServer.init();

        System.exit(0);
    }
```

RpcServer 是用netty实现的一个简单的rpc服务端



SessionHolder:

主要负责事务信息存储，对应的*ROOT_SESSION_MANAGER，ASYNC_COMMITTING_SESSION_MANAGER，RETRY_COMMITTING_SESSION_MANAGER，RETRY_ROLLBACKING_SESSION_MANAGER*分别对应相应的文件存储到本地

DefaultSessionManager内部采用队列的形式由一个线程从队列里读取session并写入本地，可以很好的避免竞争，但写入还是阻塞试等待并非异步



DefaultCoordinator :

init方法起了四个定时任务查询session并做处理，以rollback为例

```java
        retryRollbacking.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                try {
                    handleRetryRollbacking();
                } catch (Exception e) {
                    LOGGER.info("Exception retry rollbacking ... ", e);
                }

            }
        }, 0, 5, TimeUnit.MILLISECONDS);
```

从SessionHolder的*RETRY_ROLLBACKING_SESSION_MANAGER*里获取所有等待rollback的任务回滚。调用DefaultCore.doGlobalRollback->DefaultCoordinator.branchRollback

RMHandlerAT 事务提交或回滚的执行者rmRpcClient.setClientMessageListener(**new** RmMessageListener(**new** RMHandlerAT()));注册，具体的commit跟rollback实现都在这里不细说了

```java
    protected void doBranchCommit(BranchCommitRequest request, BranchCommitResponse response) throws TransactionException {
        String xid = request.getXid();
        long branchId = request.getBranchId();
        String resourceId = request.getResourceId();
        String applicationData = request.getApplicationData();
        LOGGER.info("AT Branch committing: " + xid + " " + branchId + " " + resourceId + " " + applicationData);
        BranchStatus status = dataSourceManager.branchCommit(xid, branchId, resourceId, applicationData);
        response.setBranchStatus(status);
        LOGGER.info("AT Branch commit result: " + status);

    }

    @Override
    protected void doBranchRollback(BranchRollbackRequest request, BranchRollbackResponse response) throws TransactionException {
        String xid = request.getXid();
        long branchId = request.getBranchId();
        String resourceId = request.getResourceId();
        String applicationData = request.getApplicationData();
        LOGGER.info("AT Branch rolling back: " + xid + " " + branchId + " " + resourceId);
        BranchStatus status = dataSourceManager.branchRollback(xid, branchId, resourceId, applicationData);
        response.setBranchStatus(status);
        LOGGER.info("AT Branch rollback result: " + status);

    }
```



branchRegister的全局锁机制：

```java
    public Long branchRegister(BranchType branchType, String resourceId, String clientId, String xid, String lockKeys) throws TransactionException {
        GlobalSession globalSession = assertGlobalSession(XID.getTransactionId(xid), GlobalStatus.Begin);

        BranchSession branchSession = new BranchSession();
        branchSession.setTransactionId(XID.getTransactionId(xid));
        branchSession.setBranchId(UUIDGenerator.generateUUID());
        branchSession.setApplicationId(globalSession.getApplicationId());
        branchSession.setTxServiceGroup(globalSession.getTransactionServiceGroup());
        branchSession.setBranchType(branchType);
        branchSession.setResourceId(resourceId);
        branchSession.setLockKey(lockKeys);
        branchSession.setClientId(clientId);
//锁检查
        if (!branchSession.lock()) {
            throw new TransactionException(LockKeyConflict);
        }
        try {
            globalSession.addBranch(branchSession);
        } catch (RuntimeException ex) {
            throw new TransactionException(FailedToAddBranch);

        }
        return branchSession.getBranchId();
    }
```

lockkey的机制是服务端维护的一个锁并非数据库层面的，所以这个锁仅针对fescar发起的事务有效。比如A事务包含BCD分支，需要修改TableA，后续事务如果想修改TableA就需要经过lockkey检查，直到A事务释放锁。如果有一个业务操作没有发起事务也去修改TableA，这个是不需要lockkey检查的（有可能）。

```java
    public boolean acquireLock(BranchSession branchSession) throws TransactionException {
        String resourceId = branchSession.getResourceId();
        long transactionId = branchSession.getTransactionId();
        ConcurrentHashMap<String, ConcurrentHashMap<Integer, Map<String, Long>>> dbLockMap = LOCK_MAP.get(resourceId);
        if (dbLockMap == null) {
            LOCK_MAP.putIfAbsent(resourceId, new ConcurrentHashMap<String, ConcurrentHashMap<Integer, Map<String, Long>>>());
            dbLockMap = LOCK_MAP.get(resourceId);
        }
        ConcurrentHashMap<Map<String, Long>, Set<String>> bucketHolder = branchSession.getLockHolder();
        String[] tableGroupedLockKeys = branchSession.getLockKey().split(";");
        for (String tableGroupedLockKey : tableGroupedLockKeys) {
            int idx = tableGroupedLockKey.indexOf(":");
            if (idx < 0) {
                branchSession.unlock();
                throw new ShouldNeverHappenException("Wrong format of LOCK KEYS: " + branchSession.getLockKey());
            }
            String tableName = tableGroupedLockKey.substring(0, idx);
            String mergedPKs = tableGroupedLockKey.substring(idx + 1);
            ConcurrentHashMap<Integer, Map<String, Long>> tableLockMap = dbLockMap.get(tableName);
            if (tableLockMap == null) {
                dbLockMap.putIfAbsent(tableName, new ConcurrentHashMap<Integer, Map<String, Long>>());
                tableLockMap = dbLockMap.get(tableName);
            }
            String[] pks = mergedPKs.split(",");
//检查主键是否还在tableLockMap里
            for (String pk : pks) {
                int bucketId = pk.hashCode() % BUCKET_PER_TABLE;
                Map<String, Long> bucketLockMap = tableLockMap.get(bucketId);
                if (bucketLockMap == null) {
                    tableLockMap.putIfAbsent(bucketId, new HashMap<String, Long>());
                    bucketLockMap = tableLockMap.get(bucketId);
                }
                synchronized (bucketLockMap) {
                    Long lockingTransactionId = bucketLockMap.get(pk);
                    if (lockingTransactionId == null) {
                        // No existing lock
                        bucketLockMap.put(pk, transactionId);
                        Set<String> keysInHolder = bucketHolder.get(bucketLockMap);
                        if (keysInHolder == null) {
                            bucketHolder.putIfAbsent(bucketLockMap, new ConcurrentSet<String>());
                            keysInHolder = bucketHolder.get(bucketLockMap);
                        }
                        keysInHolder.add(pk);

                    } else if (lockingTransactionId.longValue() == transactionId) {
                        // Locked by me
                        continue;
                    } else {
                        LOGGER.info("Global lock on [" + tableName + ":" + pk + "] is holding by " + lockingTransactionId);
                        branchSession.unlock(); // Release all acquired locks.
                        return false;
                    }
                }
            }
        }
        return true;
    }
```