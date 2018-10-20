title: Spring 5 源码解析 —— Spring 中的 Bean post processors
date: 2018-01-20
tag: 
categories: Spring
permalink: Spring/Bean-post-processors
author: 一叶知秋
from_url: https://muyinchen.github.io/2017/09/18/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— Spring%E4%B8%AD%E7%9A%84Bean%20post%20processors/
wechat_url: 

-------

摘要: 原创出处 https://muyinchen.github.io/2017/09/18/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— Spring%E4%B8%AD%E7%9A%84Bean%20post%20processors/ 「一叶知秋」欢迎转载，保留摘要，谢谢！

- [什么是bean post processor？](http://www.iocoder.cn/Spring/Bean-post-processors/)
- [bean后置处理器Demo](http://www.iocoder.cn/Spring/Bean-post-processors/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

我们之前已经对[Spring中的bean工厂后置处理器](https://muyinchen.github.io/2017/09/16/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-Spring%E4%B8%AD%E7%9A%84bean%E5%B7%A5%E5%8E%82%E5%90%8E%E7%BD%AE%E5%A4%84%E7%90%86%E5%99%A8/)说道过了。但是，依然留下了一个类似的概念的小尾巴需要来解释，这就是bean后置处理器(bean post processors)。

本文将分为两部分。在第一部分，将了解下Spring的单个后处理器bean。第二部分将涉及一些后置处理器(post processors)实际使用的例子。

## 什么是bean post processor？

bean生命周期始于加载bean的定义。通过拿到的这个定义，Spring可以构造出(`construct`嘛)bean并注入组件(因为我们常用的就是在controller里 service里使用)。之后，所有的bean都可以进行**后置处理**。这意味着我们可以实现一些自定义逻辑并调用它。并在调用bean的初始化方法(xml配置所定义的init-method 属性)之前和/或之后进行调用(当然默认的上下文环境是Spring容器)。

你不能为给定的bean类型明确指定一个bean后置处理器。每个定义的后处理器可以应用于`application context`中的所有定义的bean。后置处理器bean必须实现**org.springframework.beans.factory.config.BeanPostProcessor**接口并定义`postProcessBeforeInitialization`和`postProcessAfterInitialization`方法。第一个在调用初始化方法(init-method所指定的方法)之前被调用，第二个在调用初始化方法之后被调用。这两个方法都有两个参数：

- Object：表示已处理的bean的实例。
- 字符串：包含已处理的bean的名称。

```java
/**
 * Factory hook that allows for custom modification of new bean instances,
 * e.g. checking for marker interfaces or wrapping them with proxies.
 *
 * <p>ApplicationContexts can autodetect BeanPostProcessor beans in their
 * bean definitions and apply them to any beans subsequently created.
 * Plain bean factories allow for programmatic registration of post-processors,
 * applying to all beans created through this factory.
 *
 * <p>Typically, post-processors that populate beans via marker interfaces
 * or the like will implement {@link #postProcessBeforeInitialization},
 * while post-processors that wrap beans with proxies will normally
 * implement {@link #postProcessAfterInitialization}.
 *
 * @author Juergen Hoeller
 * @since 10.10.2003
 * @see InstantiationAwareBeanPostProcessor
 * @see DestructionAwareBeanPostProcessor
 * @see ConfigurableBeanFactory#addBeanPostProcessor
 * @see BeanFactoryPostProcessor
 */
public interface BeanPostProcessor {

	/**
	 * Apply this BeanPostProcessor to the given new bean instance <i>before</i> any bean
	 * initialization callbacks (like InitializingBean's {@code afterPropertiesSet}
	 * or a custom init-method). The bean will already be populated with property values.
	 * The returned bean instance may be a wrapper around the original.
	 * <p>The default implementation returns the given {@code bean} as-is.
	 * @param bean the new bean instance
	 * @param beanName the name of the bean
	 * @return the bean instance to use, either the original or a wrapped one;
	 * if {@code null}, no subsequent BeanPostProcessors will be invoked
	 * @throws org.springframework.beans.BeansException in case of errors
	 * @see org.springframework.beans.factory.InitializingBean#afterPropertiesSet
	 */
	@Nullable
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

	/**
	 * Apply this BeanPostProcessor to the given new bean instance <i>after</i> any bean
	 * initialization callbacks (like InitializingBean's {@code afterPropertiesSet}
	 * or a custom init-method). The bean will already be populated with property values.
	 * The returned bean instance may be a wrapper around the original.
	 * <p>In case of a FactoryBean, this callback will be invoked for both the FactoryBean
	 * instance and the objects created by the FactoryBean (as of Spring 2.0). The
	 * post-processor can decide whether to apply to either the FactoryBean or created
	 * objects or both through corresponding {@code bean instanceof FactoryBean} checks.
	 * <p>This callback will also be invoked after a short-circuiting triggered by a
	 * {@link InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation} method,
	 * in contrast to all other BeanPostProcessor callbacks.
	 * <p>The default implementation returns the given {@code bean} as-is.
	 * @param bean the new bean instance
	 * @param beanName the name of the bean
	 * @return the bean instance to use, either the original or a wrapped one;
	 * if {@code null}, no subsequent BeanPostProcessors will be invoked
	 * @throws org.springframework.beans.BeansException in case of errors
	 * @see org.springframework.beans.factory.InitializingBean#afterPropertiesSet
	 * @see org.springframework.beans.factory.FactoryBean
	 */
	@Nullable
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

}
```

一起来思考下，当我们需要检测一个bean是否可以被后置处理(其实就是构造函数执行完毕，`init-method`所指定的方法执行前后所要调用的处理逻辑)。为了避免写很多的if-else判断，我们可以创建一个支持后置处理的抽象出来的接口，然后由所有bean来实现。这样，我们的代码将更加具有可读性，这也就是这个接口的抽取思想。

假如没有看过我的[Spring工厂后置处理器](https://muyinchen.github.io/2017/09/16/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-Spring%E4%B8%AD%E7%9A%84bean%E5%B7%A5%E5%8E%82%E5%90%8E%E7%BD%AE%E5%A4%84%E7%90%86%E5%99%A8/)这篇文章，请先阅读完，因为本章是对其做的一个补充来讲的。接着，要讲大家关心的东西:他们之间的一些区别。`Bean Factory`后置处理器只适用于bean定义这块。它们在对象创建之前被调用，这就是为什么他们只能更改bean元数据的原因。不像`BeanPostProcessors bean` 可以更改对象的属性。你再思考该问题，如果bean工厂后置处理器和bean后置处理器重写覆盖同一对象的属性，则最后保留的值将由bean后置处理器设置的这个，这是因为它是在bean factory后置处理器之后才被调用的。

### init-method 执行释疑

关于`init-method`的执行的位置，有些人可能有疑问，这里拿个之前存的一个例子:

在配置文件中添加如下Bean定义：

```xml
<bean class="InitSequenceBean" init-method="initMethod"></bean>
```

```Java
public class InitSequenceBean implements InitializingBean {

    public InitSequenceBean() {
       System.out.println("InitSequenceBean: constructor");
    }

    @PostConstruct
    public void postConstruct() {
       System.out.println("InitSequenceBean: postConstruct");
    }

    public void initMethod() {
       System.out.println("InitSequenceBean: init-method");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
       System.out.println("InitSequenceBean: afterPropertiesSet");
    }
}
```

执行结果:

```
InitSequenceBean: constructor
InitSequenceBean: postConstruct
InitSequenceBean: afterPropertiesSet
InitSequenceBean: init-method
```

通过上述输出结果，三者的先后顺序也就一目了然了：

**Constructor > @PostConstruct > InitializingBean > init-method**

**@PostConstruct**通过`Debug`追源码可以找到这个类:**org.springframework.context.annotation.CommonAnnotationBeanPostProcessor**，从命名上，我们就可以得到某些信息—>这是一个`BeanPostProcessor`。

## bean后置处理器Demo

在我们的例子中，我们要使在程序部署时将无法使用的bean无效化。假如我们有一个VOD流媒体网站，所有的电影都可以在一个月的第一个星期免费观看(这个仅在部署时实现此效果，因为举例嘛，完善的话可以做个定时任务啥的，后面会拿一篇来讲)。验证的代码如下：

```Java
@Controller
public MovieController {
  @Autowired
  private ViewChecker viewChecker;

  // some of request mapped methods

  // check method
  private boolean movieCanBeWatched(Movie movie) {
    if (viewChecker == null) {
      return true;
    }
    return viewChecker.canBeWatched(movie);
  }
}
```

我们对一个bean进行A&B测试，以获取并格式化网店中的产品列表。第一个bean用来获取访问量最多的商品。第二个是基于用户的喜好。也就是说通过这个A&B就可以得到最受欢迎的商品(本来想举个复杂的例子的，还是算了，搞简单点吧，要不篇幅太长了)。首先，我们来定义一个bean配置：

```xml
<bean class="com.migo.bean.BeanPostProcessorSample">
<bean id="viewChecker" class="com.migo.movie.ViewChecker"/>
</bean>
```

第一个bean代表后置处理器bean。第二个，`viewChecker`是一个用来检查用户是否可以查看电影的类。我们先来看看这第二个class的：

```java
public class ViewChecker implements ProcessedBean {

  @Override
  public boolean isValid() {
    // visitors can watch movies freely between the 1st and 7th day of every month
    Calendar calendar = Calendar.getInstance();
    return calendar.get(Calendar.DAY_OF_MONTH) > 8;
  }
}
```

可以看到，代码量很少。`ProcessedBean`接口如下所示：

```java
public interface ProcessedBean {
  public boolean isValid();
}
```

实现此接口的所有bean必须实现`isValid()`方法，这样就可以用来判断该应用程序上下文是否可以使用该bean。在`BeanPostProcessorSample`中调用`ifValid`方法：

```java
public class BeanPostProcessorSample implements BeanPostProcessor  {
  @Override
  public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
    if (bean instanceof ProcessedBean) {
      if (!((ProcessedBean)bean).isValid()) {
        return null;
      }
    }
    return bean;
  }

  @Override
  public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    return bean;
  }

}
```

由上可知，我们仅实现`afterInitialization`这个后置处理器方法。通过它，我们就可以确定所分析的bean可能在`init-method(如果指定)`中所设置的数据。如果分析的bean的`isValid()`是`false`，我们返回null。但请注意返回值null(再强调一遍)。如果无效bean还存在另一个依赖关系，可以看到类似于下面这样的异常(这个异常我们经常见，空指针异常)：

```java
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'adminController': Injection of resource dependencies failed; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'adminService': Injection of resource dependencies failed; nested exception is
//也就是容器里找不到DataSource 这个bean的实例
java.lang.IllegalArgumentException: DataSource must not be null
  at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.postProcessPropertyValues(CommonAnnotationBeanPostProcessor.java:307)
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
Caused by: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'adminService': Injection of resource dependencies failed; nested exception is java.lang.IllegalArgumentException: DataSource must not be null
  at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.postProcessPropertyValues(CommonAnnotationBeanPostProcessor.java:307)
  at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.populateBean(AbstractAutowireCapableBeanFactory.java:1185)
  at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:537)
  at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:475)
  at org.springframework.beans.factory.support.AbstractBeanFactory$1.getObject(AbstractBeanFactory.java:304)
  at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:228)
  at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:300)
  at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:200)
  at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.autowireResource(CommonAnnotationBeanPostProcessor.java:445)
  at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.getResource(CommonAnnotationBeanPostProcessor.java:419)
  at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor$ResourceElement.getResourceToInject(CommonAnnotationBeanPostProcessor.java:544)
  at org.springframework.beans.factory.annotation.InjectionMetadata$InjectedElement.inject(InjectionMetadata.java:150)
  at org.springframework.beans.factory.annotation.InjectionMetadata.inject(InjectionMetadata.java:87)
  at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.postProcessPropertyValues(CommonAnnotationBeanPostProcessor.java:304)
  ... 21 more
Caused by: java.lang.IllegalArgumentException: DataSource must not be null
  at org.springframework.util.Assert.notNull(Assert.java:112)
  at org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate.(NamedParameterJdbcTemplate.java:89)
  at org.springframework.jdbc.core.simple.SimpleJdbcTemplate.(SimpleJdbcTemplate.java:70)
  at com.migo.service.AdminService.setDataSource(AdminService.java:38)
  at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
  at sun.reflect.NativeMethodAccessorImpl.invoke(Unknown Source)
  at sun.reflect.DelegatingMethodAccessorImpl.invoke(Unknown Source)
  at java.lang.reflect.Method.invoke(Unknown Source)
  at org.springframework.beans.factory.annotation.InjectionMetadata$InjectedElement.inject(InjectionMetadata.java:159)
  at org.springframework.beans.factory.annotation.InjectionMetadata.inject(InjectionMetadata.java:87)
  at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.postProcessPropertyValues(CommonAnnotationBeanPostProcessor.java:304)
  ... 34 more
```

这问题发生在`adminService`创建的时候：

```java
@Service("adminService")
public class AdminService implements GenericService {

  private SimpleJdbcTemplate jdbcTemplate;

  @Resource(name="dataSource")
  public void setDataSource(DataSource dataSource) {
    this.jdbcTemplate = new SimpleJdbcTemplate(dataSource);
  }
}
```

通过这篇文章，我们在bean生命周期几乎每个阶段都可以对它进行操作。我们可以使用`BeanFactoryPostProcessors`来更改`Bean`的定义，也可以使用`bean post processors(后置处理器)`来更改bean对象。但在更改任何内容之前，你需要分析其依赖关系。因为无效的bean(通过在后置处理器方法中返回null)可能会导致所依赖bean初始化(即空指针)的问题。

# 666. 彩蛋

如果你对 Spring 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)