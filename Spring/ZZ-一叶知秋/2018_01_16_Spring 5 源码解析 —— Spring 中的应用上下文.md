title: Spring 5 源码解析 —— Spring 中的应用上下文
date: 2018-01-16
tag: 
categories: Spring
permalink: Spring/ApplicationContext
author: 一叶知秋
from_url: https://muyinchen.github.io/2017/09/08/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— Spring%E4%B8%AD%E7%9A%84%E5%BA%94%E7%94%A8%E4%B8%8A%E4%B8%8B%E6%96%87/
wechat_url: 

-------

摘要: 原创出处 https://muyinchen.github.io/2017/09/08/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— Spring%E4%B8%AD%E7%9A%84%E5%BA%94%E7%94%A8%E4%B8%8A%E4%B8%8B%E6%96%87/ 「一叶知秋」欢迎转载，保留摘要，谢谢！

- [什么是Spring的应用程序上下文？](http://www.iocoder.cn/Spring/ApplicationContext/)
- [Spring的应用程序上下文类](http://www.iocoder.cn/Spring/ApplicationContext/)
- [关于Spring的应用程序环境的一些实践](http://www.iocoder.cn/Spring/ApplicationContext/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

之前讲到过，Spring中的 beans生活(用这俩字觉得更形象具体)在其应用程序的上下文环境中。在本文中，我们将详细介绍应用程序上下文,另外此篇同样是[由域联系到的逃逸分析](https://muyinchen.github.io/2016/11/08/%E7%94%B1%E5%9F%9F%E8%81%94%E7%B3%BB%E5%88%B0%E7%9A%84%E9%80%83%E9%80%B8%E5%88%86%E6%9E%90/)的关于Spring容器的续篇。

关于[Spring5源码解析-@Autowired](https://muyinchen.github.io/2017/08/23/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-@Autowired/)这篇文章讲了通过`@Autowired`注解进行依赖注入。这一次我们来探讨**应用程序上下文(application context)**的概念。在第一部分中，我们来看看所有Spring管理的bean生活在什么样的环境中。在第二部分，来分析下到负责上下文管理的类。在最后一部分中，我们来进行一些实践操作。

## 什么是Spring的应用程序上下文？

众所周知，Spring管理的这些类被称为bean，并且生活在Spring容器中。bean处理程序的最基本实现是**bean factory**。作为**org.springframework.beans.factory.BeanFactory**接口的**实现类**，这是一个初始化，配置和管理bean的容器。但通常在Spring应用程序中仅使用`BeanFactory`是不够的。它出现在**应用程序上下文中**。

应用程序上下文(Application context)是一种面向企业化(`其实Spring文档里也有面向企业这一说，不过这不就是企业里流水线的工厂里才能有的东西么`)的bean工厂。作为标准bean工厂，它是bean class生活的空间。但与标准bean工厂不同，应用程序上下文提供了一个补充企业层(也就是通用的东西了，比如企业里的胸牌，服装等)。又迷糊了吧，举个例子 :例如，通过提供国际化，转换服务或事件传播，使我们省去很多麻烦去亲自处理。通常，应用程序上下文优于bean工厂。但它的唯一缺点是内存消耗比bean工厂大，出现这种情况是由于补充的服务。如果内存的使用对于你的程序要求非常苛刻(例如在applet或移动环境中)，请考虑更多使用bean factory。否则，在更标准的应用程序中，应使用应用程序上下文(application context)。

## Spring的应用程序上下文类

想要了解Spring中应用程序上下文，关键部分就是**org.springframework.context.ApplicationContext**接口。它扩展了一些其他接口:

- **org.springframework.core.env.EnvironmentCapable**:用于标记对象来对外暴露自己说我实现了**Environment**接口。根据这个接口的注释可以知道，它主要用于完成类型的检查。

```java
/**
 * Interface indicating a component that contains and exposes an {@link Environment} reference.
 *  注释说了很清晰明了了，就不多废话了
 * <p>All Spring application contexts are EnvironmentCapable, and the interface is used primarily
 * for performing {@code instanceof} checks in framework methods that accept BeanFactory
 * instances that may or may not actually be ApplicationContext instances in order to interact
 * with the environment if indeed it is available.
 *
 * <p>As mentioned, {@link org.springframework.context.ApplicationContext ApplicationContext}
 * extends EnvironmentCapable, and thus exposes a {@link #getEnvironment()} method; however,
 * {@link org.springframework.context.ConfigurableApplicationContext ConfigurableApplicationContext}
 * redefines {@link org.springframework.context.ConfigurableApplicationContext#getEnvironment
 * getEnvironment()} and narrows the signature to return a {@link ConfigurableEnvironment}.
 * The effect is that an Environment object is 'read-only' until it is being accessed from
 * a ConfigurableApplicationContext, at which point it too may be configured.
 *
 * @author Chris Beams
 * @since 3.1
 * @see Environment
 * @see ConfigurableEnvironment
 * @see org.springframework.context.ConfigurableApplicationContext#getEnvironment()
 */
public interface EnvironmentCapable {

	/**
	 * Return the {@link Environment} associated with this component
	 * (may be {@code null} or a default environment).
	 */
	Environment getEnvironment();

}
```

- **org.springframework.beans.factory.ListableBeanFactory**:通过继承该interface可以列出所有bean，也可以只列出与预期类型相对应的bean。
- **org.springframework.beans.factory.HierarchicalBeanFactory**:支持分层bean的管理。
- **org.springframework.context.MessageSource**:用来解决消息支持国际化。
- **org.springframework.context.ApplicationEventPublisher**:通过该接口，可以允许通知所有类来监听到某些应用程序上下文事件。
- **org.springframework.core.io.support.ResourcePatternResolver**:是一个有助于将资源地址(例如:classpath:/WEB-INF/web.xml)解析到**org.springframework.core.io.Resource**对象中的策略接口。

```java
/**
 * Central interface to provide configuration for an application.
 * This is read-only while the application is running, but may be
 * reloaded if the implementation supports this.
 *
 * <p>An ApplicationContext provides:
 * <ul>  请看所扩展相关各个接口功能的描述
 * <li>Bean factory methods for accessing application components.
 * Inherited from {@link org.springframework.beans.factory.ListableBeanFactory}.
 * <li>The ability to load file resources in a generic fashion.
 * Inherited from the {@link org.springframework.core.io.ResourceLoader} interface.
 * <li>The ability to publish events to registered listeners.
 * Inherited from the {@link ApplicationEventPublisher} interface.
 * <li>The ability to resolve messages, supporting internationalization.
 * Inherited from the {@link MessageSource} interface.
 * <li>Inheritance from a parent context. Definitions in a descendant context
 * will always take priority. This means, for example, that a single parent
 * context can be used by an entire web application, while each servlet has
 * its own child context that is independent of that of any other servlet.
 * </ul>
 *
 * <p>In addition to standard {@link org.springframework.beans.factory.BeanFactory}
 * lifecycle capabilities, ApplicationContext implementations detect and invoke
 * {@link ApplicationContextAware} beans as well as {@link ResourceLoaderAware},
 * {@link ApplicationEventPublisherAware} and {@link MessageSourceAware} beans.
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @see ConfigurableApplicationContext
 * @see org.springframework.beans.factory.BeanFactory
 * @see org.springframework.core.io.ResourceLoader
 */
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,
		MessageSource, ApplicationEventPublisher, ResourcePatternResolver {
```

对于我们来说，实现这些接口使应用程序上下文比一个简单的bean工厂更有用。我们通过**org.springframework.web.context.support.XmlWebApplicationContext**这个实现类来看其在Web应用程序中使用。此类扩展了同一个包下`AbstractRefreshableWebApplicationContext`这个抽象类。

`XmlWebApplicationContext`实现了`AbstractRefreshableApplicationContext`中的抽象方法`loadBeanDefinitions`，用于读取所有bean。从这个方法实现，可以看出，所有的bean都是通过**org.springframework.beans.factory.xml.XmlBeanDefinitionReader**从XML文件读取的。

```java
/**
 * Loads the bean definitions via an XmlBeanDefinitionReader.
 * @see org.springframework.beans.factory.xml.XmlBeanDefinitionReader
 * @see #initBeanDefinitionReader
 * @see #loadBeanDefinitions
 */
@Override
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
	// Create a new XmlBeanDefinitionReader for the given BeanFactory.
     	//只能说Spring源码注释的太详细了，英文确实很重要
	XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

	// Configure the bean definition reader with this context's
	// resource loading environment.
	beanDefinitionReader.setEnvironment(getEnvironment());
	beanDefinitionReader.setResourceLoader(this);
	beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

	// Allow a subclass to provide custom initialization of the reader,
	// then proceed with actually loading the bean definitions.
	initBeanDefinitionReader(beanDefinitionReader);
	loadBeanDefinitions(beanDefinitionReader);
}
```

另一个有趣的方法，继承自`AbstractRefreshableWebApplicationContext`，是`postProcessBeanFactory`。它在加载所有bean定义之后并在其实例化之前被调用(`postProcess`就是bean构造函数之后调用即实例化之前)。`AbstractRefreshableWebApplicationContext`使用它来注册请求和会话作用域以及环境bean(具体看下面源码)。另外，这个抽象类实现了`ConfigurableWebApplicationContext`接口，这样一来就可以定义`servlet`的上下文和一些本地的配置。

```java
	/**
	 * Register request/session scopes, a {@link ServletContextAwareProcessor}, etc.
	 */
	@Override
	protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		beanFactory.addBeanPostProcessor(new ServletContextAwareProcessor(this.servletContext, this.servletConfig));
		beanFactory.ignoreDependencyInterface(ServletContextAware.class);
		beanFactory.ignoreDependencyInterface(ServletConfigAware.class);

		WebApplicationContextUtils.registerWebApplicationScopes(beanFactory, this.servletContext);
		WebApplicationContextUtils.registerEnvironmentBeans(beanFactory, this.servletContext, this.servletConfig);
	}

@Override
	@Nullable
	public ServletContext getServletContext() {
		return this.servletContext;
	}

	@Override
	public void setServletConfig(@Nullable ServletConfig servletConfig) {
		this.servletConfig = servletConfig;
		if (servletConfig != null && this.servletContext == null) {
			setServletContext(servletConfig.getServletContext());
		}
	}

	@Override
	@Nullable
	public ServletConfig getServletConfig() {
		return this.servletConfig;
	}


	/**
	 * {@inheritDoc}
	 * <p>Replace {@code Servlet}-related property sources.
	 */
	@Override
	protected void initPropertySources() {
		ConfigurableEnvironment env = getEnvironment();
		if (env instanceof ConfigurableWebEnvironment) {
			((ConfigurableWebEnvironment) env).initPropertySources(this.servletContext, this.servletConfig);
		}
	}
```

由`XmlWebApplicationContext`间接继承的另一个抽象类是`AbstractRefreshableApplicationContext`。它有几种处理上下文刷新的方法。处理事件通知的类是**org.springframework.context.support.AbstractApplicationContext**，由`XmlWebApplicationContext`间接继承。它包含一个将事件(`ApplicationEvent`类的实例)发送到所有侦听对象的`publishEvent`方法。

但是我们的重中之重是处理生命周期，是`AbstractApplicationContext`类的`public void refresh() throws BeansException, IllegalStateException`方法来做到的。

```java
@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```

通过阅读源码，我们可以注意到以下操作:

1. 上下文准备刷新(属性源初始化)
2. bean工厂准备好用来一系列操作(classloader定义，基本bean注册)
3. bean后置处理(postProcessBeanFactory方法)被调用
4. 消息源(消息管理)被初始化
5. event multicaster初始化(event multicaster是将事件分派到合适的侦听对象的对象)
6. 在特定的上下文子类中初始化其他特殊的bean。
7. 监听器的注册
8. 所有剩余的bean的实例化(例如:转换服务)

在非Web环境中，我们可以使用标准应用程序上下文，如`FileSystemXmlApplicationContext`，`ClassPathXmlApplicationContext`或`GenericXmlApplicationContext`。

## 关于Spring的应用程序环境的一些实践

关于此 部分，我们将看到:如何在一个控制器中获得一个上下文，查找得到一些bean配置并来解析一个消息。在进入正式的代码之前，我们需要做一些上下文的配置:

```xml
<!-- activate configuration by annotations, for example enable @Controller annotation -->
<context:annotation-config/>

<!-- indicates where Spring should looking for application services as services, controllers or components, annotated respectively with @Service, @Controller and @Component -->
<context:component-scan base-package="com.mysite.test"/>

<!-- registers RequestMappingHandlerMapping, RequestMappingHandlerAdapter and ExceptionHandlerExceptionResolver; thanks to it, Spring can resolve requests annotated with @RequestMapping and @ExceptionHandler -->
<mvc:annotation-driven/>

<!-- represents a bean which will resolve the messages -->
<bean class="org.springframework.context.support.ReloadableResourceBundleMessageSource" id="messageSource">
  <property name="basenames">
    <list>
      <value>classpath:messages</value>
      <value>classpath:errors</value>
    </list>
  </property>
  <property name="defaultEncoding" value="UTF-8">
  	<property name="fallbackToSystemLocale" value="false">
	</property>
  </property>
  </bean>
```

通过上面的配置，我们可以编写一个测试controller和一个类ApplicationContextProvider，它将保存一个应用程序上下文实例并按需返回:

```java
// controller, TestController.java
@Controller
public class TestController {

  @Autowired
  private ApplicationContext context;

  @RequestMapping(value = "/test", method = RequestMethod.GET)
  public String test() {
    LOGGER.debug("[TestController] Received application context :"+context);
    ApplicationContext providerContext = ApplicationContextProvider.getApplicationContext();
    LOGGER.debug("[TestController] Provider context is :"+providerContext);

    if (this.context == providerContext) {
    LOGGER.debug("[TestController] Both contextes are the same");
    }

    LOGGER.debug("[TestController] Message is :"+this.context.getMessage("testMessage", new Object[] {}, Locale.ENGLISH));
    return "test";
  }
}

// context provider, ApplicationContextProvider.java
@Component
public class ApplicationContextProvider implements ApplicationContextAware {

  private static ApplicationContext context;

  @Override
  public void setApplicationContext(ApplicationContext c) throws BeansException {
    context = c;
  }

  public static ApplicationContext getApplicationContext() {
    return context;
  }

}
```

`ApplicationContext`实例由`Spring`管理。这就是为什么我们可以使用`@Autowired`注解将其注入另一个Spring管理的bean(在我们的例子中是一个controller )。这是通过注入的从一个bean得到上下文第一种方法。

第二种方法是使`ApplicationContextProvider`类实现**org.springframework.context.ApplicationContextAware**接口。这里需要说一下，该接口实现后可以获取当前正在运行的`ApplicationContext`的这个事件的通知。所以实现类必须实现这个方法:**void setApplicationContext(ApplicationContext applicationContext)throws BeansException**。该方法允许设置当前的`ApplicationContext`实例并用来使用。上下文通过**org.springframework.context.support.ApplicationContextAwareProcessor**传递给`ApplicationContextAware`实现，在`AbstractApplicationContext`类中注册(见下面源码)。需要注意的是，`ApplicationcontextAwareProcessor`也用于设置bean工厂或应用程序的上下文环境，见下面此类源码中的`private final StringValueResolver embeddedValueResolver;`的`StringValueResolver`接口的实现。可以知道，要实现这两种功能，这些类必须分别从**org.springframework.context**包中实现`EmbeddedValueResolverAware`和`EnvironmentAware`接口。

```java
/**
	 * Configure the factory's standard context characteristics,
	 * such as the context's ClassLoader and post-processors.
	 * @param beanFactory the BeanFactory to configure
	 */
	protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// Tell the internal bean factory to use the context's class loader etc.
		beanFactory.setBeanClassLoader(getClassLoader());
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// Configure the bean factory with context callbacks.
      	//将applicationContext实例扔进去，见下面对ApplicationContextAwareProcessor的源码注释
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		// BeanFactory interface not registered as resolvable type in a plain factory.
		// MessageSource registered (and found for autowiring) as a bean.
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		// Register early post-processor for detecting inner beans as ApplicationListeners.
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

		// Detect a LoadTimeWeaver and prepare for weaving, if found.
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		// Register default environment beans.
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}
```

```java
/**
 * {@link org.springframework.beans.factory.config.BeanPostProcessor}
 * 看下面这句注释:
 * implementation that passes the ApplicationContext to beans that
 * implement the {@link EnvironmentAware}, {@link EmbeddedValueResolverAware},
 * {@link ResourceLoaderAware}, {@link ApplicationEventPublisherAware},
 * {@link MessageSourceAware} and/or {@link ApplicationContextAware} interfaces.
 *
 * <p>Implemented interfaces are satisfied in order of their mention above.
 *
 * <p>Application contexts will automatically register this with their
 * underlying bean factory. Applications do not use this directly.
 *
 * @author Juergen Hoeller
 * @author Costin Leau
 * @author Chris Beams
 * @since 10.10.2003
 * @see org.springframework.context.EnvironmentAware
 * @see org.springframework.context.EmbeddedValueResolverAware
 * @see org.springframework.context.ResourceLoaderAware
 * @see org.springframework.context.ApplicationEventPublisherAware
 * @see org.springframework.context.MessageSourceAware
 * @see org.springframework.context.ApplicationContextAware
 * @see org.springframework.context.support.AbstractApplicationContext#refresh()
 */
class ApplicationContextAwareProcessor implements BeanPostProcessor {

	private final ConfigurableApplicationContext applicationContext;

	private final StringValueResolver embeddedValueResolver;


	/**
	 * Create a new ApplicationContextAwareProcessor for the given context.
	 * 要创建此实例，必须要有ConfigurableApplicationContext的上下文实例才行
	 */
	public ApplicationContextAwareProcessor(ConfigurableApplicationContext applicationContext) {
		this.applicationContext = applicationContext;
		this.embeddedValueResolver = new EmbeddedValueResolver(applicationContext.getBeanFactory());
	}


	@Override
	public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
		AccessControlContext acc = null;

		if (System.getSecurityManager() != null &&
				(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
						bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
						bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)) {
			acc = this.applicationContext.getBeanFactory().getAccessControlContext();
		}

		if (acc != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareInterfaces(bean);
				return null;
			}, acc);
		}
		else {
			invokeAwareInterfaces(bean);
		}

		return bean;
	}
```

在我们的例子中，对于`ApplicationContextAware`的实现是只是一个简单的上下文提供者。但是在别的地方，我们定义的这个provider可能是用来得到上下文资源的对象。**这就是关于获取应用程序上下文的两种方式**。

最后,我们通过一个方法来使用context(上下文)的getMessage方法来对消息解析。在我们的`message_en.properties`文件中，可以事先声明消息的模板:**testMessage =It’s our test message with content。**然后我们会在日志文件中看到相应的输出。

顺便说一下，从`ApplicationContextProvider`获得的对象和`@Autowired`的对象之间的上下文是相同的:

# 666. 彩蛋

如果你对 Spring 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)