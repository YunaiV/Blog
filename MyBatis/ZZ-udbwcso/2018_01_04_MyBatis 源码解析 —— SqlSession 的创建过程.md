title: MyBatis 源码解析 —— SqlSession 的创建过程
date: 2018-01-04
tag: 
categories: MyBatis
permalink: MyBatis/udbwcso/SqlSession
author: udbwcso
from_url: https://my.oschina.net/u/657390/blog/663991
wechat_url: 

-------

摘要: 原创出处 https://my.oschina.net/u/657390/blog/663991 「udbwcso」欢迎转载，保留摘要，谢谢！

- [666. 彩蛋](http://www.iocoder.cn/MyBatis/udbwcso/SqlSession/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

以上是之前的分析,在[mybatis源码分析之事务管理器](http://my.oschina.net/u/657390/blog/663080)里分析到了事务管理器

```Java
SqlSession session = sqlSessionFactory.openSession();

//DefaultSqlSessionFactory里的openSession
public SqlSession openSession() {
  return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
}

private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
  Transaction tx = null;
  try {
    //根据配置获取环境
    final Environment environment = configuration.getEnvironment();
    //构建事务工厂
    final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
    //通过事务工厂创建事务Transaction对象
    tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
    //创建执行器Executor对象
    final Executor executor = configuration.newExecutor(tx, execType);
    //根据configuration,executor创建DefaultSqlSession对象
    return new DefaultSqlSession(configuration, executor, autoCommit);
  } catch (Exception e) {
    closeTransaction(tx); // may have fetched a connection so lets call close()
    throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}
```

之前已经分析过事务管理器,下面分析执行器Executor

![img](http://static.iocoder.cn/oschina/uploads/space/2016/0421/171053_YPoi_657390.png)

```Java
public enum ExecutorType {
  SIMPLE, REUSE, BATCH
}
```

执行器类型只有三种

SIMPLE:普通的执行器;

REUSE:执行器会重用预处理语句（prepared statements）;

BATCH:执行器将重用语句并执行批量更新。

```Java
configuration.newExecutor(tx, execType);

//默认执行器类型
protected ExecutorType defaultExecutorType = ExecutorType.SIMPLE;
//二级缓存的全局开关,默认开启缓存
protected boolean cacheEnabled = true;

public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
  //executorType为null时executorType=ExecutorType.SIMPLE
  executorType = executorType == null ? defaultExecutorType : executorType;
  executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
  Executor executor;
  //根据执行器类型创建执行器
  if (ExecutorType.BATCH == executorType) {
    executor = new BatchExecutor(this, transaction);
  } else if (ExecutorType.REUSE == executorType) {
    executor = new ReuseExecutor(this, transaction);
  } else {
    executor = new SimpleExecutor(this, transaction);
  }
  //当cacheEnabled为true时创建CachingExecutor对象
  if (cacheEnabled) {
    executor = new CachingExecutor(executor);
  }
  executor = (Executor) interceptorChain.pluginAll(executor);
  return executor;
}
```

二级缓存开关配置示例

```XML
<settings>
  <setting name="cacheEnabled" value="true"/>
</settings>
```

执行器创建后

```Java
new DefaultSqlSession(configuration, executor, autoCommit);

//DefaultSqlSession构造方法
public DefaultSqlSession(Configuration configuration, Executor executor, boolean autoCommit) {
  this.configuration = configuration;
  this.executor = executor;
  this.dirty = false;
  this.autoCommit = autoCommit;
}
```

至此SqlSession创建完成,从之前的几篇和这篇能清晰的看到从读取配置文件到SqlSession创建的整个过程.需要注意的是SqlSession 的实例不是线程安全的,是不能被共享的,所以它的最佳的范围是请求或方法范围.每个线程都应该有自己的 SqlSession 实例.

# 666. 彩蛋

如果你对 MyBatis 感兴趣，欢迎加入我的知识一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)