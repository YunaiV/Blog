title: MyBatis 源码解析 —— mapper 动态代理
date: 2018-01-05
tag: 
categories: MyBatis
permalink: MyBatis/udbwcso/mapper
author: udbwcso
from_url: https://my.oschina.net/u/657390/blog/748646
wechat_url: 

-------

摘要: 原创出处 https://my.oschina.net/u/657390/blog/748646 「udbwcso」欢迎转载，保留摘要，谢谢！

- [666. 彩蛋](http://www.iocoder.cn/MyBatis/udbwcso/mapper/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

上一篇:[mybatis源码分析之SqlSession的创建过程](https://my.oschina.net/u/657390/blog/663991)

<https://my.oschina.net/u/657390/blog/663991>

重点分析了SqlSession的创建过程.SqlSession创建成功后:

```Java
String resource = "com/analyze/mybatis/mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
SqlSession sqlSession = sqlSessionFactory.openSession();
Map map = sqlSession.selectOne("com.analyze.mybatis.mapper.UserMapper.getUA");
sqlSession.close();

String resource = "com/analyze/mybatis/mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
SqlSession sqlSession = sqlSessionFactory.openSession();
UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
Map map = userMapper.getUA();
sqlSession.close();
```

以上两段代码最终将得到相同的结果.

对比可以发现两段代码不同之处为:

```Java
Map map = sqlSession.selectOne("com.analyze.mybatis.mapper.UserMapper.getUA");

UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
Map map = userMapper.getUA();
```

根据sqlSession.selectOne("com.analyze.mybatis.mapper.UserMapper.getUA");

可以想像selectOne会用com.analyze.mybatis.mapper.UserMapper.getUA查找相应的配置,然后执行sql.

接着分析sqlSession.selectOne()

```Java
DefaultSqlSession.java
public <T> T selectOne(String statement) {
    return this.<T>selectOne(statement, null);
}

public <T> T selectOne(String statement, Object parameter) {
    // Popular vote was to return null on 0 results and throw exception on too many.
    List<T> list = this.<T>selectList(statement, parameter);
    if (list.size() == 1) {
        return list.get(0);
    } else if (list.size() > 1) {
        throw new TooManyResultsException("Expected one result (or null) to be returned by selectOne(), but found: " + list.size());
    } else {
        return null;
    }
}

public <E> List<E> selectList(String statement, Object parameter) {
    return this.selectList(statement, parameter, RowBounds.DEFAULT);
}


public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
        //从配置中获取statement信息
        MappedStatement ms = configuration.getMappedStatement(statement);
        //调用执行器
        return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```

但UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
Map map = userMapper.getUA();这两行代码能够看出一些问题,在代码里并没有实现UserMapper这个接口,也就是说userMapper.getUA();并没实现,最终却能够被调用.那么这到底是怎么实现的呢?

进一步分析源码:

```Java
DefaultSqlSession.java
public <T> T getMapper(Class<T> type) {
	return configuration.<T>getMapper(type, this);
}

Configuration.java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
}

MapperRegistry.java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
	final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
	if (mapperProxyFactory == null) {
		throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
	}
	try {
		return mapperProxyFactory.newInstance(sqlSession);
	} catch (Exception e) {
		throw new BindingException("Error getting mapper instance. Cause: " + e, e);
	}
}
```

从以上代码可以看出最终调用的是mapperProxyFactory.newInstance()

接着分析MapperProxyFactory的源码

```Java
public class MapperProxyFactory<T> {

    private final Class<T> mapperInterface;
    private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<Method, MapperMethod>();

    public MapperProxyFactory(Class<T> mapperInterface) {
        this.mapperInterface = mapperInterface;
    }

    public Class<T> getMapperInterface() {
        return mapperInterface;
    }

    public Map<Method, MapperMethod> getMethodCache() {
        return methodCache;
    }

    @SuppressWarnings("unchecked")
    protected T newInstance(MapperProxy<T> mapperProxy) {
        return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[]{mapperInterface}, mapperProxy);
    }

    public T newInstance(SqlSession sqlSession) {
        final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
        return newInstance(mapperProxy);
    }

}
```

从newInstance里的MapperProxy很容易就可以看出使用了动态代理.

再来看看MapperProxy里的invoke

```Java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    //当执行的方法是继承自Object时执行this里的相应方法
    if (Object.class.equals(method.getDeclaringClass())) {
        try {
            return method.invoke(this, args);
        } catch (Throwable t) {
            throw ExceptionUtil.unwrapThrowable(t);
        }
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    //最终执行的是execute方法
    return mapperMethod.execute(sqlSession, args);
}
```

从以上代码可以看出调用mapper里的接口的时候执行的都是mapperMethod.execute(sqlSession, args);

继续跟进代码会发现上面的例子最终执行的是

```Java
sqlSession.selectOne(command.getName(), param);
```

可谓殊途同归.

mapper动态代理涉及到的类有MapperRegistry,MapperProxyFactory,MapperProxy,MapperMethod

![img](http://static.iocoder.cn/oschina/uploads/space/2016/0920/104134_RGFK_657390.png)

MapperRegistry的数据源头Configuration.java

MapperRegistry mapperRegistry = new MapperRegistry(this);

XMLConfigBuilder中的方法parseConfiguration()调用mapperElement(root.evalNode("mappers"));

mapperElement()会调用addMapper()最后将数据添加到MapperRegistry中的knownMappers

分析完源码可以仿照以上代码来实现自己的功能:

```Java
public interface UserService {
    Map getUser();
}


public class ServiceProxy implements InvocationHandler {

    public <T> T newInstance(Class<T> clz) {
        return (T) Proxy.newProxyInstance(clz.getClassLoader(), new Class[]{clz}, this);
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (Object.class.equals(method.getDeclaringClass())) {
            return method.invoke(this, args);
        }
        System.out.println("----proxy----" + method.getName());
        return null;
    }
}

//测试代码
public static void main(String[] args) {
    UserService userService = new ServiceProxy().newInstance(UserService.class);
    userService.toString();
    userService.getUser();
}

//输出结果
----proxy----getUser
```

# 666. 彩蛋

如果你对 MyBatis 感兴趣，欢迎加入我的知识一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)