title: MyBatis 源码解析 —— TypeHandler
date: 2018-01-07
tag: 
categories: MyBatis
permalink: MyBatis/udbwcso/TypeHandler
author: udbwcso
from_url: https://my.oschina.net/u/657390/blog/790456
wechat_url: 

-------

摘要: 原创出处 https://my.oschina.net/u/657390/blog/790456 「udbwcso」欢迎转载，保留摘要，谢谢！

- [1. 配置](http://www.iocoder.cn/MyBatis/udbwcso/TypeHandler/)
- [2. 设置参数](http://www.iocoder.cn/MyBatis/udbwcso/TypeHandler/)
- [666. 彩蛋](http://www.iocoder.cn/MyBatis/udbwcso/TypeHandler/)

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

[mybatis源码分析之MapperMethod](https://my.oschina.net/u/657390/blog/755787)

<https://my.oschina.net/u/657390/blog/755787>

分析了MapperMethod从创建到执行的过程,MapperMethod的执行包括执行sql返回结果.

在执行sql和返回结果的过程中就会涉及到参数类型的转换,这个过程是通过TypeHandler来处理的.关于TypeHandler官网有比较详细的文档http://www.mybatis.org/mybatis-3/zh/configuration.html#typeHandlers,文档主要说明了如何使用TypeHandler,在下面的分析中将重点分析与TypeHandler有关的源码.

#### 1.配置

MyBatis有默认的类型处理器,如果需要自定义配置也相当简单,在mybatis-config.xml里添加如下配置:

```XML
<typeHandlers>
  <typeHandler handler="org.mybatis.example.ExampleTypeHandler"/>
</typeHandlers>
```

下面分析配置读取设置的过程,在XMLConfigBuilder中

```Java
    /**
     * 读取配置文件组装configuration
     * @param root 配置文件的configuration节点
     */
  private void parseConfiguration(XNode root) {
    try {
      //issue #117 read properties first
      propertiesElement(root.evalNode("properties"));
      Properties settings = settingsAsProperties(root.evalNode("settings"));
      loadCustomVfs(settings);
      typeAliasesElement(root.evalNode("typeAliases"));
      pluginElement(root.evalNode("plugins"));
      objectFactoryElement(root.evalNode("objectFactory"));
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      reflectorFactoryElement(root.evalNode("reflectorFactory"));
      settingsElement(settings);
      // read it after objectFactory and objectWrapperFactory issue #631
      environmentsElement(root.evalNode("environments"));
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      typeHandlerElement(root.evalNode("typeHandlers"));
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }
```

在以上源码中有一行

```Java
typeHandlerElement(root.evalNode("typeHandlers"));
```

再来看typeHandlerElement这个方法

```Java
/**
   * 读取typeHandlers配置并注册
   * @param parent 配置文件typeHandlers节点
   * @throws Exception
   */
  private void typeHandlerElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        if ("package".equals(child.getName())) {
          String typeHandlerPackage = child.getStringAttribute("name");
          typeHandlerRegistry.register(typeHandlerPackage);
        } else {
          String javaTypeName = child.getStringAttribute("javaType");
          String jdbcTypeName = child.getStringAttribute("jdbcType");
          String handlerTypeName = child.getStringAttribute("handler");
          Class<?> javaTypeClass = resolveClass(javaTypeName);
          JdbcType jdbcType = resolveJdbcType(jdbcTypeName);
          Class<?> typeHandlerClass = resolveClass(handlerTypeName);
          if (javaTypeClass != null) {
            if (jdbcType == null) {
              typeHandlerRegistry.register(javaTypeClass, typeHandlerClass);
            } else {
              typeHandlerRegistry.register(javaTypeClass, jdbcType, typeHandlerClass);
            }
          } else {
            typeHandlerRegistry.register(typeHandlerClass);
          }
        }
      }
    }
  }
```

if和else中的代码逻辑对应了typeHandler的两种配置方式.最后都会调用

```Java
typeHandlerRegistry.register()
```

![img](http://static.iocoder.cn/oschina/uploads/space/2016/1118/150653_TtjR_657390.png)

以上是TypeHandler与TypeHandlerRegistry,Configuration,BaseTypeHandler之间的关系.

#### 2.设置参数

设置参数时先调用ParameterHandler.setParameters(),然后在setParameters()里获取相应的typeHandler,最后调用typeHandler.setParameter()

![img](http://static.iocoder.cn/oschina/uploads/space/2016/1118/162854_zO7R_657390.png)

再来看看BaseTypeHandler的setParameter方法

当parameter不为null时调用的是setNonNullParameter,也就是说子类需要实现setNonNullParameter

BigIntegerTypeHandler的源码:

```Java
  public void setNonNullParameter(PreparedStatement ps, int i, BigInteger parameter, JdbcType jdbcType) throws SQLException {
    ps.setBigDecimal(i, new BigDecimal(parameter));
  }
```

至此,TypeHandler的作用已经大致分析完毕了.

# 666. 彩蛋

如果你对 MyBatis 感兴趣，欢迎加入我的知识一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)