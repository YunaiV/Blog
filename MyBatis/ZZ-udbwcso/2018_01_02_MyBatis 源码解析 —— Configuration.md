title: MyBatis 源码解析 —— Configuration
date: 2018-01-02
tag: 
categories: MyBatis
permalink: MyBatis/udbwcso/Configuration
author: udbwcso
from_url: https://my.oschina.net/u/657390/blog/661681
wechat_url: 

-------

摘要: 原创出处 https://my.oschina.net/u/657390/blog/661681 「udbwcso」欢迎转载，保留摘要，谢谢！

- [666. 彩蛋](http://www.iocoder.cn/MyBatis/udbwcso/Configuration/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

上一篇对mybatis中SqlSessionFactory的创建过程进行了分析,从之前的分析可以看出创建过程中比较重要的一部分是Configuration,这一篇将重点分析配置文件的读取.

以官方的例子来进行分析.

mybatis-config.xml是mybatis最开始的地方,dataSource,mappers等的配置都在这里.

```Java
String resource = "org/mybatis/example/mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```

进入SqlSessionFactoryBuilder,可以看出最终所有的build方法都是调用的以下方法:

```Java
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
  try {
    XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
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

environment:环境,MyBatis 可以配置成适应多种环境,例如开发,测试和生产.

XMLConfigBuilder与Configuration之间的关系

![img](http://static.iocoder.cn/oschina/uploads/space/2016/0415/180027_3Rv1_657390.png)

从这张图可以看出使用了建造者模式.

在build方法里涉及到与读取配置相关的代码

```Java
XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
//构造方法
//XMLMapperEntityResolver离线MyBatis dtd实体解析器
public XMLConfigBuilder(InputStream inputStream, String environment, Properties props) {
  this(new XPathParser(inputStream, true, props, new XMLMapperEntityResolver()), environment, props);
}
```

比较重要的部分是创建XPathParser实例

```Java
new XPathParser(inputStream, true, props, new XMLMapperEntityResolver())
//创建XPathParser实例
public XPathParser(InputStream inputStream, boolean validation, Properties variables, EntityResolver entityResolver) {
  commonConstructor(validation, variables, entityResolver);
  this.document = createDocument(new InputSource(inputStream));
}
```

commonConstructor完成的工作如下

```Java
private void commonConstructor(boolean validation, Properties variables, EntityResolver entityResolver) {
  this.validation = validation;
  this.entityResolver = entityResolver;
  this.variables = variables;
  //使用默认的对象模型得到一个新的XPathFactory实例
  XPathFactory factory = XPathFactory.newInstance();
  this.xpath = factory.newXPath();
}
```

XPathFactory,XPath,XPathFactoryImpl,XPath之间的关系

![img](http://static.iocoder.cn/oschina/uploads/space/2016/0418/111931_Pe52_657390.png)

可以看出使用了工厂方法模式.

接着就是调用createDocument创建document的过程

XMLConfigBuilder创建后调用parse()

```Java
parser.parse()
//parse()源码
public Configuration parse() {
  if (parsed) {
    throw new BuilderException("Each XMLConfigBuilder can only be used once.");
  }
  parsed = true;
  parseConfiguration(parser.evalNode("/configuration"));
  return configuration;
}
```

主要调用了parseConfiguration其中parser.evalNode("/configuration")获取到的是根结点

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

从上面的代码可以比较清晰的看出在读取配置文件的过程比较重要的类有:XNode,XPathParser

XNode是Node类的扩展,XPathParser是xml文件的解析器工具类.

XPathParser中比较重要的方法是:public XNode evalNode(String expression)

evalNode最终调用的是com.sun.org.apache.xpath.internal.jaxp.XPathImpl里的

public Object evaluate(String expression, Object item, QName returnType)

XPath的相关资料:

<http://www.w3school.com.cn/xpath/index.asp>

例子

```Java
import org.apache.ibatis.io.Resources;
import org.w3c.dom.Document;
import org.w3c.dom.NodeList;
import org.xml.sax.SAXException;

import javax.xml.parsers.DocumentBuilder;
import javax.xml.parsers.DocumentBuilderFactory;
import javax.xml.parsers.ParserConfigurationException;
import javax.xml.xpath.*;
import java.io.IOException;
import java.io.InputStream;

public class XPathExample {
    public static void main(String[] args) throws ParserConfigurationException, IOException, SAXException, XPathExpressionException {
        DocumentBuilderFactory domFactory = DocumentBuilderFactory.newInstance();
        domFactory.setNamespaceAware(true);
        DocumentBuilder builder = domFactory.newDocumentBuilder();
        String resource = "com/analyze/mybatis/mybatis-config.xml";
        InputStream inputStream = Resources.getResourceAsStream(resource);
        Document doc = builder.parse(inputStream);

        XPathFactory factory = XPathFactory.newInstance();
        XPath xpath = factory.newXPath();
        XPathExpression expr
                = xpath.compile("/configuration/mappers/mapper");
        Object result = expr.evaluate(doc, XPathConstants.NODESET);
        NodeList nodes = (NodeList) result;
        for (int i = 0; i < nodes.getLength(); i++) {
            System.out.println(nodes.item(i).getAttributes().getNamedItem("resource").getNodeValue());
        }

    }
}
```

xml

```XML
<?xml version="1.0" encoding="UTF-8" ?><!DOCTYPE configuration
  PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-config.dtd">
  <configuration>
      <environments default="development">
        <environment id="development">
          <transactionManager type="JDBC"/>
          <dataSource type="POOLED">
            <property name="driver" value="${driver}"/>
            <property name="url" value="${url}"/>
            <property name="username" value="${username}"/>
            <property name="password" value="${password}"/>
          </dataSource>
        </environment>
      </environments>
      <mappers>
        <mapper resource="org/mybatis/example/BlogMapper.xml"/>
      </mappers>
  </configuration>
```

# 666. 彩蛋

如果你对 MyBatis 感兴趣，欢迎加入我的知识一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)