title: Spring 5 源码解析 —— Spring 中的 Context loader
date: 2018-01-17
tag: 
categories: Spring
permalink: Spring/Context-loader
author: 一叶知秋
from_url: https://muyinchen.github.io/2017/09/12/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— Spring%E4%B8%AD%E7%9A%84Context%20loader/
wechat_url: 

-------

摘要: 原创出处 https://muyinchen.github.io/2017/09/12/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— Spring%E4%B8%AD%E7%9A%84Context%20loader/ 「一叶知秋」欢迎转载，保留摘要，谢谢！

  - [什么是Spring的上下文加载器(context loader)？](http://www.iocoder.cn/Spring/Context-loader/)
  - [Spring的上下文加载器详解](http://www.iocoder.cn/Spring/Context-loader/)
  - [在Spring Web应用程序中实现上下文加载程序](http://www.iocoder.cn/Spring/Context-loader/)
  - [总结](http://www.iocoder.cn/Spring/Context-loader/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

我们已经知道，应用程序上下文是Spring管理的bean所在的容器。但是我们依然要问一个问题:这个上下文是如何创建的？那么在这篇文章中我们来探讨这个问题。

在第一部分中，会说下在[Spring的应用程序上下文中](https://muyinchen.github.io/2017/09/08/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-Spring%E4%B8%AD%E7%9A%84%E5%BA%94%E7%94%A8%E4%B8%8A%E4%B8%8B%E6%96%87/)所谓的**上下文加载器(context loader)**是什么。在第二部分，我们会讨论这个加载器的代码细节。最后一部分，老规矩，写我们自己的一个自定义的loader。在继续之前，需要说一下，loader(加载器) 将根据web application和dispatcher servlet来结合进行分析。其实这也是很多人一碰到源码就像无头苍蝇，不知道从何而起了，刚开始放下所有，从大体去思考该如何入手,这里对设计模式了解就很重要了，还有，源码的类注释很重要，不多说，接着走。

## 什么是Spring的上下文加载器(context loader)？

见名知意，上下文加载程序负责构建应用程序上下文。我们可以通过**org.springframework.web.context.ContextLoaderListener的**实例来对其分析(**从我之前的设计模式的文章可以看到，Spring通过观察者模式，其实我自己总结的是电影院模式，声音和画面通过broadcaster发送到listener，listener再调用相应的adapter来处理,所以，这里就直接从listener来找了**)，它继承并扩展了同一个包下的`ContextLoader`类。同时还实现了**javax.servlet.ServletContextListener**接口。该接口旨在接收有关servlet上下文中更改变化的通知。只有当它们在(`WEB-INF/web.xml`)中注册时，这个接口的实现才能接收这些通知。

在Spring Web应用程序中，会在servlet上下文创建时调用上下文加载程序(`context loader`)。之后，开始初始化根Web应用程序上下文(`Root WebApplicationContext`)。**Root**非常重要，因为在加载的时候，可以创建两个或更多的上下文。第一个，也是最重要的，定义了整个bean的生存空间，被称为**应用程序上下文(application context)**。另一个是**servlet应用程序上下文**,其包含更多的是面向Web的元素，比如控制器(controllers)或视图解析器。然而我们需要记住的是，`servlet`的上下文是根应用程序上下文(`Root WebApplicationContext`)的子集，也就是父子容器一说。这意味着`servlet`可以从根应用程序上下文继承所有的bean。这就是为什么你可以在根配置文件中定义一些常见资源(例如:services，这也是我们的Spring xml配置文件为什么要分service和MVC两个的原因)，并通过两个不同的servlet进行共享的原因。但是在另一方面，根应用程序上下文不能获取到特定于servlet的bean，看过我的[逃逸分析](https://muyinchen.github.io/2016/11/08/%E7%94%B1%E5%9F%9F%E8%81%94%E7%B3%BB%E5%88%B0%E7%9A%84%E9%80%83%E9%80%B8%E5%88%86%E6%9E%90/)的应该都清楚了吧。

我们可以将注意力拉回到关于上下文加载器的两个作用上:

- 将根Web应用程序上下文(`Root WebApplicationContext`)绑定到调度程序特定的上下文中
- 自动创建上下文(程序员不需要编写任何东西来使上下文工作)

## Spring的上下文加载器详解

我们已经了解了上下文加载器的作用。现在，我们来更详细地介绍这其中的细节。web上下文加载器(context loader)类位于**org.springframework.web.context**包中。主类是`ContextLoaderListener`，它扩展了`ContextLoader`类。同时实现了`ServletContextListener`接口。

在上下文创建时调用的方法是**public void contextInitialized(ServletContextEvent event)**。它通过传递给它所接收到的servlet上下文(从事件参数获取`event.getServletContext()`)来调用`ContextLoader`的`initWebApplicationContext`方法。`initWebApplicationContext`方法进行的第一个操作是检查是否有另一个根上下文存在。如果至少存在另一个，则抛出`IllegalStateException`，并且初始化失败。否则，它继续初始化**org.springframework.web.context.WebApplicationContext**实例。如果初始化的实例实现了`ConfigurableWebApplicationContext`接口，则在设置当前应用程序上下文之前，加载器将进行一些设置服务(父上下文，应用程序上下文，servlet上下文等)，并通过上下文的`refresh()`方法来准备bean，这已经在关于[应用程序上下文](https://muyinchen.github.io/2017/09/08/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-Spring%E4%B8%AD%E7%9A%84%E5%BA%94%E7%94%A8%E4%B8%8A%E4%B8%8B%E6%96%87/)的文章中介绍过了。

**org.springframework.web.context.ContextLoaderListener:**

```java
/**
 * Initialize the root web application context.
 */
@Override
public void contextInitialized(ServletContextEvent event) {
	initWebApplicationContext(event.getServletContext());
}
```

**org.springframework.web.context.ContextLoader:**

```java
/**
	 * Initialize Spring's web application context for the given servlet context,
	 * using the application context provided at construction time, or creating a new one
	 * according to the "{@link #CONTEXT_CLASS_PARAM contextClass}" and
	 * "{@link #CONFIG_LOCATION_PARAM contextConfigLocation}" context-params.
	 * @param servletContext current servlet context
	 * @return the new WebApplicationContext
	 * @see #ContextLoader(WebApplicationContext)
	 * @see #CONTEXT_CLASS_PARAM
	 * @see #CONFIG_LOCATION_PARAM
	 */
	public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
		if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
			throw new IllegalStateException(
					"Cannot initialize context because there is already a root application context present - " +
					"check whether you have multiple ContextLoader* definitions in your web.xml!");
		}

		Log logger = LogFactory.getLog(ContextLoader.class);
		servletContext.log("Initializing Spring root WebApplicationContext");
		if (logger.isInfoEnabled()) {
			logger.info("Root WebApplicationContext: initialization started");
		}
		long startTime = System.currentTimeMillis();

		try {

			// Store context in local instance variable, to guarantee that
			// it is available on ServletContext shutdown.
			if (this.context == null) {
				this.context = createWebApplicationContext(servletContext);
			}
          //此处判断下初始化的实例实现了ConfigurableWebApplicationContext接口
			if (this.context instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
				if (!cwac.isActive()) {
					// The context has not yet been refreshed -> provide services such as
					// setting the parent context, setting the application context id, etc
					if (cwac.getParent() == null) {
						// The context instance was injected without an explicit parent ->
						// determine parent for root web application context, if any.
						ApplicationContext parent = loadParentContext(servletContext);
						cwac.setParent(parent);
					}
                  	//refresh()准备生米煮熟饭了
					configureAndRefreshWebApplicationContext(cwac, servletContext);
				}
			}
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

			ClassLoader ccl = Thread.currentThread().getContextClassLoader();
			if (ccl == ContextLoader.class.getClassLoader()) {
				currentContext = this.context;
			}
			else if (ccl != null) {
				currentContextPerThread.put(ccl, this.context);
			}

			if (logger.isDebugEnabled()) {
				logger.debug("Published root WebApplicationContext as ServletContext attribute with name [" +
						WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE + "]");
			}
			if (logger.isInfoEnabled()) {
				long elapsedTime = System.currentTimeMillis() - startTime;
				logger.info("Root WebApplicationContext: initialization completed in " + elapsedTime + " ms");
			}

			return this.context;
		}
		catch (RuntimeException ex) {
			logger.error("Context initialization failed", ex);
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
			throw ex;
		}
		catch (Error err) {
			logger.error("Context initialization failed", err);
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, err);
			throw err;
		}
	}
```

`ContextLoaderListener`中第二个我们需要关注的方法是**public void contextDestroyed(ServletContextEvent event)**。每当加载程序的上下文关闭时都会调用它。这个方法干了两件事情:

- 通过`ContextLoader`中的`closeWebApplicationContext()`，它关闭应用程序上下文。通过`ConfigurableWebApplicationContext close()`方法完成上下文关闭。上下文的销毁的过程其实就是销毁bean和关闭bean工厂，此处参考**org.springframework.context.support.AbstractApplicationContext**中的源码，下面相关部分已贴出。
- 调用**ContextCleanupListener.cleanupAttributes(event.getServletContext())**，它将查找当前servlet上下文的所有实现**org.springframework.beans.factory.DisposableBean**接口的对象。之后，将调用它们的destroy()方法，以销毁不再使用的bean。

**org.springframework.web.context.ContextLoaderListener:**

```java
/**
 * Close the root web application context.
 */
@Override
public void contextDestroyed(ServletContextEvent event) {
	closeWebApplicationContext(event.getServletContext());
	ContextCleanupListener.cleanupAttributes(event.getServletContext());
}
```

**org.springframework.web.context.ContextLoader:**

```java
/**
	 * Close Spring's web application context for the given servlet context. If
	 * the default {@link #loadParentContext(ServletContext)} implementation,
	 * which uses ContextSingletonBeanFactoryLocator, has loaded any shared
	 * parent context, release one reference to that shared parent context.
	 * <p>If overriding {@link #loadParentContext(ServletContext)}, you may have
	 * to override this method as well.
	 * @param servletContext the ServletContext that the WebApplicationContext runs in
	 */
	public void closeWebApplicationContext(ServletContext servletContext) {
		servletContext.log("Closing Spring root WebApplicationContext");
		try {
			if (this.context instanceof ConfigurableWebApplicationContext) {
				((ConfigurableWebApplicationContext) this.context).close();
			}
		}
		finally {
			ClassLoader ccl = Thread.currentThread().getContextClassLoader();
			if (ccl == ContextLoader.class.getClassLoader()) {
				currentContext = null;
			}
			else if (ccl != null) {
				currentContextPerThread.remove(ccl);
			}
			servletContext.removeAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE);
			if (this.parentContextRef != null) {
				this.parentContextRef.release();
			}
		}
	}
```

**org.springframework.context.support.AbstractApplicationContext:**

```java
/**
	 * DisposableBean callback for destruction of this instance.
	 * Only called when the ApplicationContext itself is running
	 * as a bean in another BeanFactory or ApplicationContext,
	 * which is rather unusual.
	 * <p>The {@code close} method is the native way to
	 * shut down an ApplicationContext.
	 * @see #close()
	 * @see org.springframework.beans.factory.access.SingletonBeanFactoryLocator
	 */
	@Override
	public void destroy() {
		close();
	}

	/**
	 * Close this application context, destroying all beans in its bean factory.
	 * <p>Delegates to {@code doClose()} for the actual closing procedure.
	 * Also removes a JVM shutdown hook, if registered, as it's not needed anymore.
	 * @see #doClose()
	 * @see #registerShutdownHook()
	 */
	@Override
	public void close() {
		synchronized (this.startupShutdownMonitor) {
			doClose();
			// If we registered a JVM shutdown hook, we don't need it anymore now:
			// We've already explicitly closed the context.
			if (this.shutdownHook != null) {
				try {
					Runtime.getRuntime().removeShutdownHook(this.shutdownHook);
				}
				catch (IllegalStateException ex) {
					// ignore - VM is already shutting down
				}
			}
		}
	}

	/**
	 * Actually performs context closing: publishes a ContextClosedEvent and
	 * destroys the singletons in the bean factory of this application context.
	 * <p>Called by both {@code close()} and a JVM shutdown hook, if any.
	 * @see org.springframework.context.event.ContextClosedEvent
	 * @see #destroyBeans()
	 * @see #close()
	 * @see #registerShutdownHook()
	 */
	protected void doClose() {
		if (this.active.get() && this.closed.compareAndSet(false, true)) {
			if (logger.isInfoEnabled()) {
				logger.info("Closing " + this);
			}

			LiveBeansView.unregisterApplicationContext(this);

			try {
				// Publish shutdown event.
				publishEvent(new ContextClosedEvent(this));
			}
			catch (Throwable ex) {
				logger.warn("Exception thrown from ApplicationListener handling ContextClosedEvent", ex);
			}

			// Stop all Lifecycle beans, to avoid delays during individual destruction.
			try {
				getLifecycleProcessor().onClose();
			}
			catch (Throwable ex) {
				logger.warn("Exception thrown from LifecycleProcessor on context close", ex);
			}

			// Destroy all cached singletons in the context's BeanFactory.
			destroyBeans();

			// Close the state of this context itself.
			closeBeanFactory();

			// Let subclasses do some final clean-up if they wish...
			onClose();

			this.active.set(false);
		}
	}

	/**
	 * Template method for destroying all beans that this context manages.
	 * The default implementation destroy all cached singletons in this context,
	 * invoking {@code DisposableBean.destroy()} and/or the specified
	 * "destroy-method".
	 * <p>Can be overridden to add context-specific bean destruction steps
	 * right before or right after standard singleton destruction,
	 * while the context's BeanFactory is still active.
	 * @see #getBeanFactory()
	 * @see org.springframework.beans.factory.config.ConfigurableBeanFactory#destroySingletons()
	 */
	protected void destroyBeans() {
		getBeanFactory().destroySingletons();
	}
	/**
	 * Template method which can be overridden to add context-specific shutdown work.
	 * The default implementation is empty.
	 * <p>Called at the end of {@link #doClose}'s shutdown procedure, after
	 * this context's BeanFactory has been closed. If custom shutdown logic
	 * needs to execute while the BeanFactory is still active, override
	 * the {@link #destroyBeans()} method instead.
	 */
	protected void onClose() {
		// For subclasses: do nothing by default.
	}
```

## 在Spring Web应用程序中实现上下文加载程序

想象一下，你希望在系统的所有用户之间共享一个信息。你可以用传统的方式做到这一点，也可以使用你定义的上下文加载器。我们通过写一些简单的代码来达到这个目的。还有一个想要实现的功能会涉及多个上下文。我们的应用程序将同时处理`guest`和`connected`两种形式(请同时看下面源码)。可以看到他们的网页的URL匹配规则不一样。使用connected的用户将能够访问与guest规则下以.chtml扩展名结尾的相同的页面，也就是所谓的交集。需要说的是，他们不会共享相同的信息(两个不一样的上下文当然不会一样了)。还不懂的话看下面源码，对于这两者，我们将分别 指定两个servlet上下文。你会看到，因为它，访问connected用户将不会与访问guest共享相同的bean。

我们将从`web.xml`文件开始,请对比上面说的:

```xml
<!--?xml version="1.0" encoding="UTF-8"?-->
<web-app id="WebApp_ID" version="2.4" xmlns="http://java.sun.com/xml/ns/j2ee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemalocation="http://java.sun.com/xml/ns/j2ee http://java.sun.com/xml/ns/j2ee/web-app_2_4.xsd">
  <servlet>
    <servlet-name>guest</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:spring/guest-servlet.xml</param-value>
     </init-param>
    <load-on-startup>1</load-on-startup>
  </servlet>

  <-- guest is the default servlet -->
  <servlet-mapping>
    <servlet-name>guest</servlet-name>
    <url-pattern>/</url-pattern>
  </servlet-mapping>

  <context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>
    /WEB-INF/applicationContext.xml
    </param-value>
  </context-param>
  <-- Customized listener which will put some personnalized data into servlet's context -->
  <listener>
    <listener-class>com.mysite.servlet.CustomizedContextLoader</listener-class>
  </listener>

  <servlet>
    <servlet-name>connected</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
      <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:spring/connected-servlet.xml</param-value>
       </init-param>
    <load-on-startup>2</load-on-startup>
  </servlet>

  <servlet-mapping>
    <servlet-name>connected</servlet-name>
    <url-pattern>**.chtml</url-pattern>
  </servlet-mapping>
</web-app>
```

两个指定的servlet的bean配置文件几乎相同。唯一的区别是connected-servlet.xml包含一个没有与guest servlet共享的bean的定义。这个bean的名字是secretData:

```xml
<bean id="secretData" class="com.migo.secret.SecretData">
  <property name="question" value="How old are you ?"/>
  <property name="answer" value="33"/>
</bean>
<context:annotation-config/>
<context:component-scan base-package="com.migo"/>
<mvc:annotation-driven/>
```

神秘豆的内容主要由setter和toString方法组成:

```java
public class SecretData {

  private String question;
  private String answer;

  public void setQuestion(String question) {
    this.question = question;
  }

  public void setAnswer(String answer) {
    this.answer = answer;
  }

  @Override
  public String toString() {
    return "SecretData {question: "+this.question+", answer: "+this.answer+"}";
  }

}
```

其他Java代码也很简单。在`CustomizedContextLoader`中，我们重写`contextInitialized`方法来放置共享`servlet`的上下文属性:名字叫`webappVersion`。该属性是一个随机数，用于证明根应用程序上下文的加载程序仅被调用一次:

```java
public class CustomizedContextLoader extends ContextLoaderListener  {

  @Override
  public void contextInitialized(ServletContextEvent event) {
    System.out.println("[CustomizedContextLoader] Loading context");
    // this value could be read from data source, but for the simplicity reasons, we put it statically
    // number is random because we want to prove that the root context is loaded only once
    Random random = new Random();
    int version = random.nextInt(100001);
    System.out.println("Version set into servlet's context :"+version);
    event.getServletContext().setAttribute("webappVersion", version);
    super.contextInitialized(event);
  }
}
```

之后，我们传递给用来处理访问网址的`TestController`:

```java
@Controller
public class TestController    {

  @Autowired
  private ApplicationContext context;

  @RequestMapping(value = "/test.chtml", method = RequestMethod.GET)
  public String test(HttpServletRequest request) {
    LOGGER.debug("[TestController] Webapp version from servlet's context :"+request.getServletContext().getAttribute("webappVersion"));
    LOGGER.debug("[TestController] Found secretData bean :"+context.getBean("secretData"));
    return "test";
  }

  @RequestMapping(value = "/test.html", method = RequestMethod.GET)
  public String guestTest(HttpServletRequest request) {
    LOGGER.debug("[TestController] Webapp version from servlet's context :"+request.getServletContext().getAttribute("webappVersion"));
    LOGGER.debug("[TestController] Found secretData bean :"+context.getBean("secretData"));
    return "test";
  }
}
```

测试的时候，首先输入[http://localhost:8080/test.chtml，然后输入http://localhost:8080/test.html。然后通过查看日志](http://localhost:8080/test.chtml%EF%BC%8C%E7%84%B6%E5%90%8E%E8%BE%93%E5%85%A5http://localhost:8080/test.html%E3%80%82%E7%84%B6%E5%90%8E%E9%80%9A%E8%BF%87%E6%9F%A5%E7%9C%8B%E6%97%A5%E5%BF%97):

```shell
[CustomizedContextLoader] Loading context
Version set into servlet's context :38023
// ... test.chtml
[TestController] Webapp version from servlet's context :38023
[TestController] Found secretData bean :SecretData {question: How old are you ?, answer: 33}
// ... test.html
[TestController] Webapp version from servlet's context :38023
3 avr. 2014 14:01:02 org.apache.catalina.core.StandardWrapperValve invoke
GRAVE: Servlet.service() for servlet [guestServlet] in context with path [] threw exception [Request processing failed; nested exception is org.springframework.beans.factory.NoSuchBeanDefinitionException: No bean named 'secretData' is defined] with root cause
org.springframework.beans.factory.NoSuchBeanDefinitionException: No bean named 'secretData' is defined
	  at org.springframework.beans.factory.support.DefaultListableBeanFactory.getBeanDefinition(DefaultListableBeanFactory.java:638)
  at org.springframework.beans.factory.support.AbstractBeanFactory.getMergedLocalBeanDefinition(AbstractBeanFactory.java:1159)
  at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:282)
  at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:200)
  at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:273)
  at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:195)
  at org.springframework.context.support.AbstractApplicationContext.getBean(AbstractApplicationContext.java:973)
  at com.mysite.controller.TestController.guestTest(TestController.java:114)
  at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
  at sun.reflect.NativeMethodAccessorImpl.invoke(Unknown Source)
  at sun.reflect.DelegatingMethodAccessorImpl.invoke(Unknown Source)
  at java.lang.reflect.Method.invoke(Unknown Source)
  at org.springframework.web.method.support.InvocableHandlerMethod.invoke(InvocableHandlerMethod.java:214)
  at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:132)
  at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:104)
  at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandleMethod(RequestMappingHandlerAdapter.java:748)
  at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:689)
  at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:83)
  at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:945)
  at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:876)
  at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:931)
  at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:822)
  at javax.servlet.http.HttpServlet.service(HttpServlet.java:668)
  at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:807)
  at javax.servlet.http.HttpServlet.service(HttpServlet.java:770)
  at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:304)
  at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:210)
  at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:240)
  at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:164)
  at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:462)
  at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:164)
  at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:100)
  at org.apache.catalina.valves.AccessLogValve.invoke(AccessLogValve.java:562)
  at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:118)
  at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:395)
  at org.apache.coyote.http11.Http11Processor.process(Http11Processor.java:250)
  at org.apache.coyote.http11.Http11Protocol$Http11ConnectionHandler.process(Http11Protocol.java:188)
  at org.apache.tomcat.util.net.JIoEndpoint$SocketProcessor.run(JIoEndpoint.java:302)
  at java.util.concurrent.ThreadPoolExecutor$Worker.runTask(Unknown Source)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(Unknown Source)
  at java.lang.Thread.run(Unknown Source)
```

首先，将一个信息(“Version set into servlet’s context :”+version)放在servlet上下文中，并由两个servlet上下文继承。第二点是bean的可见性。`Guest`的`servlet`没有看到`secretData bean`，因为它仅在`connected` (connected-servlet.xml)的配置中被定义。

## 总结

第一部分涉及了这个加载器的两个主要角色:将根Web应用程序上下文(`Root WebApplicationContext`)绑定到调度程序特定的上下文中并自动创建上下文。接下来，我们分析了关于上下文加载程序的代码的要点所涉及的细节，如所实现的接口和主要方法的细节实现。最后一部分是我们自定义扩展本地上下文加载器，然后对bean和servlet的属性继承方面进行一些测试。

# 666. 彩蛋

如果你对 Spring 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)