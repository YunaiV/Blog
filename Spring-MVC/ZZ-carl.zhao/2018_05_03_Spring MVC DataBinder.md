title: 【carl.zhao】Spring MVC DataBinder
date: 2018-05-03
tag: 
categories: Spring-MVC
permalink: Spring-MVC/carlzhao/DataBinder
author: carl.zhao
from_url: http://blog.csdn.net/u012410733/article/details/53368351
wechat_url: 

-------

摘要: 原创出处 http://blog.csdn.net/u012410733/article/details/53368351 「carl.zhao」欢迎转载，保留摘要，谢谢！

- [1、DataBinder](http://www.iocoder.cn/Spring-MVC/carlzhao/DataBinder/)
- [2、setPropertyValues](http://www.iocoder.cn/Spring-MVC/carlzhao/DataBinder/)
- [3、DataBinder类图](http://www.iocoder.cn/Spring-MVC/carlzhao/DataBinder/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

我们都知道Spring MVC在处理HTTP请求的时候的数据都是来自于HTTP 请求。这不是废话吗，：）！ 那么Spring MVC是如何把HTTP中的请求中的数据纳入到其中呢？我们都知道Spring MVC处理HTTP请求是通过DispatcherServlet来做为拦截请求的。DispatcherServlet是一个Ja va EE中的Servlet，可以从Servlet容器(e.g. Tomcat) 获取到HTTP请求传过来的报文封装的HttpServletRequest 。

由之前的BLOG – [Spring MVC DispatcherServlet](http://blog.csdn.net/u012410733/article/details/51920055),我们可以知道Spring MVC最终是通过反射来调用定义在Controller中标注了@RequestMapping的方法。其中数据绑定的入口是InvocableHandlerMethod#getMethodArgumentValues方法。

下面先把调用的时序图弄出来。由于方法调用过深就画了2个时序图：

1. DataBinder:数据绑定,把HttpServletRequest中的数据纳入到Spring的管理。
2. setPropertyValues:设置property到Controller标注了@RequestMapping的方法参数中。

# 1、DataBinder

把HttpServletRequest中的参数放入到Spring中的DataBinder对象中。

![这里写图片描述](http://static.iocoder.cn/csdn/20161127214219417)

在这个过程中在ModelAttributeMethodProcessor#resolveArgument通过下面的代码创建空的方法参数对象实例:

```Java
Object attribute = (mavContainer.containsAttribute(name) ? mavContainer.getModel().get(name) :
        createAttribute(name, parameter, binderFactory, webRequest));12
```

创建的这个空的实例是作为下一步setPropertyValue利用内省来进行塞值 作准备的。

# 2、setPropertyValues

![这里写图片描述](http://static.iocoder.cn/csdn/20161127214405950)

通过第一步中实例化这一个需要把HTTP请求中的值放入到Spring MVC中的参数对象当中。当然最外层是foreach的把Spring MVC Controller中的方法参数创建出来给后面的反射调用。

# 3、DataBinder类图

在Spring MVC中使用DataBinder的实例是ExtendedServletRequestDataBinder。我们先来看一下它的类图:

![这里写图片描述](http://static.iocoder.cn/csdn/20161127220921377)

这个类图是不是和之前Spring IOC依赖注入中的[BeanWrapper](http://blog.csdn.net/u012410733/article/details/53346345)的有点差不多，只是它是简化版本的依赖注入.

而且Spring MVC当中创建WebDataBinder是通过工厂方法模式来创建的，下面WebDataBinderFactory的类继承图:

![这里写图片描述](http://static.iocoder.cn/csdn/20161127221928343)

下面来分析这个Spring MVC中是何时把数据真正纳入到DataBinder当中的。具体是在第一步DataBinder当中ModelAttributeMethodProcessor#resolveArgument中的下面的方法当中创建的:

```Java
WebDataBinder binder = binderFactory.createBinder(webRequest, attribute, name);1
```

当然这个创建过程还有很多细节，我这个只是给大家给个抛砖引玉的作用。如果大家感兴趣，可以自己去看看源代码。

这下子就把Spring MVC就把HttpServletRequest当中的参数纳入到Spring当中，然后再把数据通过内省把数据塞到Spring MVC当中Controller当中标注了@RequestMapping的方法中。我们就可以来分析一下Spring MVC处理Servlet的步骤:

1. @RequestMapping:初始化数据把URL与Controller当中的方法创建出来。
2. DispacherServlet：通过doService统一处理HTTP请求通过URL。
3. DataBinder:在HttpServletRequest中的参数塞值到@RequestMapping的方法中。
4. 利用反射调用@RequestMapping方法，返回值渲染到HttpServletRespinse中展现到页面。

# 666. 彩蛋

如果你对 Java 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)