title: MyBatis 源码解析 —— SqlSessionFactory
date: 2018-01-01
tag: 
categories: MyBatis
permalink: MyBatis/udbwcso/SqlSessionFactory
author: udbwcso
from_url: https://my.oschina.net/u/657390/blog/653637
wechat_url: 

-------

摘要: 原创出处 https://my.oschina.net/u/657390/blog/653637 「udbwcso」欢迎转载，保留摘要，谢谢！

- [666. 彩蛋](http://www.iocoder.cn/MyBatis/udbwcso/SqlSessionFactory/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

版本:mybatis-3

主要分析SqlSessionFactory的构建过程

SqlSessionFactoryBuilder从XML中构建SqlSessionFactory

```Java
String resource = "org/mybatis/example/mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```



![img](http://static.iocoder.cn/oschina/uploads/space/2016/0406/144757_x3gI_657390.png)

```Java
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```

SqlSessionFactoryBuilder用来创建SqlSessionFactory实例,一旦创建了SqlSessionFactory,就不再需要它了

```Java
public SqlSessionFactory build(InputStream inputStream) {
  return build(inputStream, null, null);
}
```

调用

```Java
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties)
```

源码如下:

```Java
/**
   * 根据配置创建SqlSessionFactory
   *
   * @param inputStream 配置文件输入流
   * @param environment 环境名称
   * @param properties  外部配置
   * @return
   */
  public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
    try {
      XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
      //parser.parse()读取配置文件返回configuration
      //build()根据返回的configuration创建SqlSessionFactory
      return build(parser.parse());
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } finally {
      ErrorContext.instance().reset();
      try {
        inputStream.close();
      } catch (IOException e) {
        // Intentionally ignore. Prefer previous error.
      }
    }
  }
```

最终创建DefaultSqlSessionFactory实例

```Java
  public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
  }
```

其中

XMLConfigBuilder与Configuration

![img](http://static.iocoder.cn/oschina/uploads/space/2016/0406/153838_T1YK_657390.png)

XMLConfigBuilder的方法parse()

```Java
public Configuration parse() {
	if (parsed) {
	    throw new BuilderException("Each XMLConfigBuilder can only be used once.");
	}
	parsed = true;
	//读取mybatis-config.xml配置信息,"configuration"是根结点
	parseConfiguration(parser.evalNode("/configuration"));
	return configuration;
}
```

XMLConfigBuilder的方法parseConfiguration(XNode root)

```Java
/**
 * 读取配置文件组装configuration
 * @param root 配置文件的configuration节点
 */
private void parseConfiguration(XNode root) {
	try {
		//issue #117 read properties first
		propertiesElement(root.evalNode("properties"));
		typeAliasesElement(root.evalNode("typeAliases"));
		pluginElement(root.evalNode("plugins"));
		objectFactoryElement(root.evalNode("objectFactory"));
		objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
		reflectionFactoryElement(root.evalNode("reflectionFactory"));
		settingsElement(root.evalNode("settings"));
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

关于配置文件相关的源码分析参看:

<http://www.iocoder.cn/MyBatis/udbwcso/Configuration>

设计模式

从SqlSessionFactory和Configuration的创建过程中可以看出它们都使用了建造者模式.

# 666. 彩蛋

如果你对 MyBatis 感兴趣，欢迎加入我的知识一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)