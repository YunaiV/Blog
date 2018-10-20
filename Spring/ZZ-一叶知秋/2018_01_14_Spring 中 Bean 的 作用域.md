title: Spring 中 Bean 的 作用域
date: 2018-01-14
tag: 
categories: Spring
permalink: Spring/Bean
author: 一叶知秋
from_url: https://muyinchen.github.io/2017/08/25/Spring%E4%B8%AD%E7%9A%84%E4%BD%9C%E7%94%A8%E5%9F%9F/
wechat_url: 

-------

摘要: 原创出处 https://muyinchen.github.io/2017/08/25/Spring%E4%B8%AD%E7%9A%84%E4%BD%9C%E7%94%A8%E5%9F%9F/ 「一叶知秋」欢迎转载，保留摘要，谢谢！

- [Spring中request请求作用域是什么？](http://www.iocoder.cn/Spring/Bean)
- [什么是Spring的Session作用域？](http://www.iocoder.cn/Spring/Bean)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

> Spring Bean，就像JavaBeans中一样，有其使用的作用域。前面的文章中我们已经看到其中的两个:singleton 和prototype。这次来讲讲另外2个作用域(总共六个，参考本人[Spring5文档翻译](https://github.com/muyinchen/Spring-Framework-5.0.0.M3-CN/blob/master/3.5-bean.md))。

本文将分为两部分。每个部分描述一个bean作用域。所以，在第一个，我们将探讨下`request请求`作用域。第二个描述的是`session`和`全局session`(此在Spring5文档中已经消失)的作用域。每一部分将由理论和实践组成。需要注意的是:这些概念仅在Web Spring应用程序上下文中有效。

## Spring中request请求作用域是什么？

每个请求初始化具有此作用域的Bean注解。这听起来像是原型作用域的描述，但它们有一些差异。第一个区别是原型作用域在Spring的上下文中可用。而请求作用域仅适用于Web应用程序。第二个是原型bean根据需求进行初始化，而请求bean是在每个请求下构建的。需要说的是，request作用域bean在其作用域内有且仅有一个实例。而你可以拥有一个或多个原型作用域bean实例。

在以下代码中，你可以看到请求作用域bean的示例：

```xml
<bean id="shoppingCartRequest" class="com.migo.scope.ShoppingCartRequest" scope="request">
    <aop:scoped-proxy/>
</bean>
```

当使用注解驱动组件或Java Config时，`@RequestScope`注解可以用于将一个组件分配给`request`作用域。

```Java
@RequestScope
@Component
public class ShoppingCartRequest {
	// ...
}
```

```Java
// request bean

// injection sample
@Controller
public class TestController {
    @Autowired
    private ShoppingCartRequest shoppingCartRequest;

    @RequestMapping(value = "/test", method = RequestMethod.GET)
    public String test(HttpServletRequest request) {
        LOGGER.debug("shoppingCartRequest is :"+shoppingCartRequest);
        // ...
    }
}
```

请注意**<bean>定义内**存在的**<aop: scoped-proxy />**标签。这代表着使用代理对象。所以实际上，TestController持有的是代理对象的引用。我们所有的调用该对象都会转发到真正的`ShoppingCartRequest`对象。

有时我们需要使用`DispatcherServlet`的另一个`servlet`来处理请求。在这种情况下，我们必须确保Spring中所有请求都可用(否则可以抛出与下面类似的异常)。为此，我们需要在`web.xml`中定义一个监听器:

```xml
<listener>
  <listener-class>org.springframework.web.context.request.RequestContextListener</listener-class>
</listener>
```

调用/测试URL后，你应该能在日志中的发现以下信息:

```Shell
shoppingCartRequest is :com.migo.scope.ShoppingCartRequest@2586b11c
shoppingCartRequest is :com.migo.scope.ShoppingCartRequest@3bd5b945
```

如果我们尝试在单例bean中使用request作用域的bean，则会在应用程序上下文加载阶段抛出一个`BeanCreationException`:

```shell
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'testController': Injection of autowired dependencies failed; nested exception is org.springframework.beans.factory.BeanCreationException: Could not autowire field: private com.migo.scope.ShoppingCartRequest com.migo.controller.TestController.shoppingCartRequest; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'shoppingCartRequest': Scope 'request' is not active for the current thread; consider defining a scoped proxy for this bean if you intend to refer to it from a singleton; nested exception is java.lang.IllegalStateException: No thread-bound request found: Are you referring to request attributes outside of an actual web request, or processing a request outside of the originally receiving thread? If you are actually operating within a web request and still receive this message, your code is probably running outside of DispatcherServlet/DispatcherPortlet: In this case, use RequestContextListener or RequestContextFilter to expose the current request.
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor.postProcessPropertyValues(AutowiredAnnotationBeanPostProcessor.java:292)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.populateBean(AbstractAutowireCapableBeanFactory.java:1185)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:537)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:475)
	at org.springframework.beans.factory.support.AbstractBeanFactory$1.getObject(AbstractBeanFactory.java:304)
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:228)
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:300)
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:195)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.preInstantiateSingletons(DefaultListableBeanFactory.java:700)
	at org.springframework.context.support.AbstractApplicationContext.finishBeanFactoryInitialization(AbstractApplicationContext.java:760)
	at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:482)
	at org.springframework.web.context.ContextLoader.configureAndRefreshWebApplicationContext(ContextLoader.java:381)
	at org.springframework.web.context.ContextLoader.initWebApplicationContext(ContextLoader.java:293)
	at org.springframework.web.context.ContextLoaderListener.contextInitialized(ContextLoaderListener.java:106)
	at org.apache.catalina.core.StandardContext.listenerStart(StandardContext.java:4701)
	at org.apache.catalina.core.StandardContext$1.call(StandardContext.java:5204)
	at org.apache.catalina.core.StandardContext$1.call(StandardContext.java:5199)
	at java.util.concurrent.FutureTask$Sync.innerRun(Unknown Source)
	at java.util.concurrent.FutureTask.run(Unknown Source)
	at java.util.concurrent.ThreadPoolExecutor$Worker.runTask(Unknown Source)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(Unknown Source)
	at java.lang.Thread.run(Unknown Source)
Caused by: org.springframework.beans.factory.BeanCreationException: Could not autowire field: private com.migo.scope.ShoppingCartRequest com.migo.controller.TestController.shoppingCartRequest; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'shoppingCartRequest': Scope 'request' is not active for the current thread; consider defining a scoped proxy for this bean if you intend to refer to it from a singleton; nested exception is java.lang.IllegalStateException: No thread-bound request found: Are you referring to request attributes outside of an actual web request, or processing a request outside of the originally receiving thread? If you are actually operating within a web request and still receive this message, your code is probably running outside of DispatcherServlet/DispatcherPortlet: In this case, use RequestContextListener or RequestContextFilter to expose the current request.
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor$AutowiredFieldElement.inject(AutowiredAnnotationBeanPostProcessor.java:508)
	at org.springframework.beans.factory.annotation.InjectionMetadata.inject(InjectionMetadata.java:87)
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor.postProcessPropertyValues(AutowiredAnnotationBeanPostProcessor.java:289)
	... 21 more
Caused by: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'shoppingCartRequest': Scope 'request' is not active for the current thread; consider defining a scoped proxy for this bean if you intend to refer to it from a singleton; nested exception is java.lang.IllegalStateException: No thread-bound request found: Are you referring to request attributes outside of an actual web request, or processing a request outside of the originally receiving thread? If you are actually operating within a web request and still receive this message, your code is probably running outside of DispatcherServlet/DispatcherPortlet: In this case, use RequestContextListener or RequestContextFilter to expose the current request.
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:353)
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:195)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.findAutowireCandidates(DefaultListableBeanFactory.java:1014)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.doResolveDependency(DefaultListableBeanFactory.java:957)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveDependency(DefaultListableBeanFactory.java:855)
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor$AutowiredFieldElement.inject(AutowiredAnnotationBeanPostProcessor.java:480)
	... 23 more
Caused by: java.lang.IllegalStateException: No thread-bound request found: Are you referring to request attributes outside of an actual web request, or processing a request outside of the originally receiving thread? If you are actually operating within a web request and still receive this message, your code is probably running outside of DispatcherServlet/DispatcherPortlet: In this case, use RequestContextListener or RequestContextFilter to expose the current request.
	at org.springframework.web.context.request.RequestContextHolder.currentRequestAttributes(RequestContextHolder.java:131)
	at org.springframework.web.context.request.AbstractRequestAttributesScope.get(AbstractRequestAttributesScope.java:41)
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:338)
	... 28 more
```

## 什么是Spring的Session作用域？

Session作用域的bean与request 作用域的bean没有太大的不同。它们也与纯Web应用程序上下文相关联。注解为Session作用域的Bean对于每个用户的会话仅创建一次。他们在会话结束时被破坏销毁掉。

由Session作用域限制的Bean可以被认为是面向Web的单例，因为给定环境(用户会话)仅存在一个实例。但请记住，你无法在Web应用程序上下文中使用它们(说个好理解点的，就是一个函数内部自定义变量所在的作用域，函数执行完就销毁了，没有什么逃逸，关于此处更深入的理解请看我的博文[由域联系到的逃逸分析](https://muyinchen.github.io/2016/11/08/%E7%94%B1%E5%9F%9F%E8%81%94%E7%B3%BB%E5%88%B0%E7%9A%84%E9%80%83%E9%80%B8%E5%88%86%E6%9E%90/))。

想知道Session作用域bean在Spring中的操作，我们需要在配置文件中定义一个bean:

```xml
<bean id="shoppingCartRequest" class="com.migo.scope.ShoppingCartSession" scope="session">
    <aop:scoped-proxy/>
</bean>
```

通过`@Autowired`注解，查找这个bean的方式与request 作用域的bean相同。可以看到以下结果:

```shell
shoppingCartSession is :com.migo.scope.ShoppingCartSession@3876e5d
shoppingCartSession is :com.migo.scope.ShoppingCartSession@3876e5d
shoppingCartSession is :com.migo.scope.ShoppingCartSession@3876e5d
shoppingCartSession is :com.migo.scope.ShoppingCartSession@3876e5d
shoppingCartSession is :com.migo.scope.ShoppingCartSession@3876e5d
shoppingCartSession is :com.migo.scope.ShoppingCartSession@2f87fafc
```

你可以看到，前5个打印输出代表相同的对象。最后一个是不同的。这是什么意思 ?简单来说，这代表 着一个新的用户使用自动注入的Session作用域访问该页面。我们可以通过打开两个浏览器的测试页(/test)来观察它。每个都将初始化一个新的会话Session，因此也就创建新的`ShoppingCartSession bean`实例。

关于全局会话作用域(Global session scope)属于4.3x的范畴了，Spring5已经没有了，Spring5文档是去掉了因为4的存在所以还是说两句，它保留给portlet应用程序。 是不是一脸懵逼，so，来解释一下portlet是什么。Portlet是能够生成语义代码(例如：HTML)片段的小型Java Web插件。它们基于portlet容器，可以像servlet一样处理HTTP请求。但是，与servlet不同，每个portlet都有不同的会话。在这种情况下，Spring提供了一个名为`global-session`的作用域。通过它，一个bean可以通过应用程序中的多个portlet共享。

```XML
<bean id="userPreferences" class="com.foo.UserPreferences" scope="globalSession"/>
```

至此，我们解释了请求和面向会话的作用域。第一个的作用是在每个request请求上创建新的bean。第二个在Session会话开始的时候初始化bean。

# 666. 彩蛋

如果你对 Spring 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)