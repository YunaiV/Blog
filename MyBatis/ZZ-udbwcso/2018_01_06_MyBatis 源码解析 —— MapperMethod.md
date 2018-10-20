title: MyBatis 源码解析 —— MapperMethod
date: 2018-01-06
tag: 
categories: MyBatis
permalink: MyBatis/udbwcso/MapperMethod
author: udbwcso
from_url: https://my.oschina.net/u/657390/blog/755787
wechat_url: 

-------

摘要: 原创出处 https://my.oschina.net/u/657390/blog/755787 「udbwcso」欢迎转载，保留摘要，谢谢！

- [666. 彩蛋](http://www.iocoder.cn/MyBatis/udbwcso/MapperMethod/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------


上一篇:

[﻿mybatis源码分析之mapper动态代理](https://my.oschina.net/u/657390/blog/748646)

<https://my.oschina.net/u/657390/blog/748646>

MapperMethod与MapperProxy,MapperProxyFactory,MapperRegistry,Configuration之间的关系

![img](http://static.iocoder.cn/oschina/uploads/space/2016/1009/153546_1XP3_657390.png)

在[﻿](https://my.oschina.net/u/657390/blog/748646)分析mapper动态代理的时候可以看出最终执行的是mapperMethod.execute

```Java
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      //当执行的方法是继承自Object时执行this里的相应方法
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    //最终执行的是mapperMethod.execute
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }
```

接着分析mapperMethod.execute()

首先看MapperProxy中cachedMapperMethod(method)

```Java
/**
 * methodCache中已经存在传入的参数method
 * 则从methodCache中取MapperMethod,
 * 否则根据method生成MapperMethod实例并存储到methodCache中
 *
 * @param method
 * @return
 */
  private MapperMethod cachedMapperMethod(Method method) {
    MapperMethod mapperMethod = methodCache.get(method);
    if (mapperMethod == null) {
      mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
      methodCache.put(method, mapperMethod);
    }
    return mapperMethod;
  }
```

MapperProxy中methodCache

```Java
private final Map<Method, MapperMethod> methodCache;
```

追溯methodCache的来源

```Java
//MapperProxy构造方法
public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, Map<Method, MapperMethod> methodCache) {
    this.sqlSession = sqlSession;
    this.mapperInterface = mapperInterface;
    this.methodCache = methodCache;
  }
```

MapperProxyFactory中

```Java
private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<Method, MapperMethod>();

public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }
```

从MapperProxyFactory的源码中可以看出methodCache最初是一个空的ConcurrentHashMap,

从MapperProxy中cachedMapperMethod(method)可以看出

 methodCache中已经存在传入的参数method, 则从methodCache中取MapperMethod, 否则根据method生成MapperMethod实例并存储到methodCache中.

接着看MapperMethod的源码

构造方法

```Java
  /**
   * @param mapperInterface 接口
   * @param method 调用的方法
   * @param config 配置
   */
  public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
    this.command = new SqlCommand(config, mapperInterface, method);
    this.method = new MethodSignature(config, mapperInterface, method);
  }
```

创建了SqlCommand和MethodSignature实例

command type

```Java
public enum SqlCommandType {
  UNKNOWN, INSERT, UPDATE, DELETE, SELECT, FLUSH;
}
```

MapperMethod中最重要的方法

```Java
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) {
      case INSERT: {
    	Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      case UPDATE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.update(command.getName(), param));
        break;
      }
      case DELETE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.delete(command.getName(), param));
        break;
      }
      case SELECT:
        if (method.returnsVoid() && method.hasResultHandler()) {
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else {
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
        }
        break;
      case FLUSH:
        result = sqlSession.flushStatements();
        break;
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName()
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
```

首先调用MethodSignature里的

```Java
public Object convertArgsToSqlCommandParam(Object[] args)
```

处理参数.然后,

如果command type是

```SQL
INSERT, UPDATE, DELETE
```

则直接调用sqlSession的相应方法

如果command type是

```SQL
SELECT
```

则根据返回结果调用sqlSession里相应的查询方法.

至此MapperMethod从创建到执行的过程都大致分析了一遍.

# 666. 彩蛋

如果你对 MyBatis 感兴趣，欢迎加入我的知识一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)