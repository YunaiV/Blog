title: Spring 5 源码解析 —— Spring 框架中的单例和原型 bean
date: 2018-01-18
tag: 
categories: Spring
permalink: Spring/singleton-prototype
author: 一叶知秋
from_url: https://muyinchen.github.io/2017/09/15/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— Spring%E6%A1%86%E6%9E%B6%E4%B8%AD%E7%9A%84%E5%8D%95%E4%BE%8B%E5%92%8C%E5%8E%9F%E5%9E%8Bbean/
wechat_url: 

-------

摘要: 原创出处 https://muyinchen.github.io/2017/09/15/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— Spring%E6%A1%86%E6%9E%B6%E4%B8%AD%E7%9A%84%E5%8D%95%E4%BE%8B%E5%92%8C%E5%8E%9F%E5%9E%8Bbean/ 「一叶知秋」欢迎转载，保留摘要，谢谢！

- [Spring中的bean默认都是单身贵族](http://www.iocoder.cn/Spring/singleton-prototype/)
- [将原型放在单例中，反之亦然](http://www.iocoder.cn/Spring/singleton-prototype/)
- [方法注入](http://www.iocoder.cn/Spring/singleton-prototype/)
- [Spring中的Bean类](http://www.iocoder.cn/Spring/singleton-prototype/)
- [总结](http://www.iocoder.cn/Spring/singleton-prototype/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

最近一直有问我单例和原型bean的一些原理性问题，这里就开一篇来说说的

通过Spring中的依赖注入极大方便了我们的开发。在`xml`通过`<bean>`定义(或者通过`@Bean`在配置类里定义)对象之后，然后只需简单地使用@Autowired注解，就可以使用由Spring上下文管理的每个对象。需要注意的是，所有这些对象在Spring中默认都是单例。

这一次我们会去讨论Spring如何来管理这些定义的bean。在第一部分中，我们将讲解单例和原型作用域的概念。第二部分中，我们将分析单例和原型作用域之间的依赖关系。其后说一下方法注入。最后专门对相关Spring的代码来做下分析，具体看看bean是如何构建出来的。

## Spring中的bean默认都是单身贵族

Spring使用单例设计模式来管理bean？不完全是。Singleton设计模式假定它们是由Java的类加载器管理的jvm中给定类的唯一一个实例。在Spring中，还是有点不一样。默认情况下，它们为每个给定的*org.springframework.context.ApplicationContext*实例存在唯一的一个bean (有点别扭，也就是可以有多个Spring容器，每一个容器内存在唯一bean实例，之前的文章中有涉及例子的)。这意味着如果你有两个或更多上下文，所有这些上下文都由同一Java的类加载器管理(因为在同一个jvm环境中)，则可能会有多个给定bean的实例。唯一需要做到的是必须在每个上下文中定义此bean。讲那么多不如代码更有说服力:

```java
public class MultipleContextes {

  public static void main(String[] args) {
    try {
      // retreive two different contexts
      ApplicationContext firstContext = new FileSystemXmlApplicationContext("/home/bartosz/webapp/src/main/resources/META-INF/applicationContext.xml");
      ApplicationContext secondContext = new FileSystemXmlApplicationContext("/home/bartosz/webapp/src/main/resources/META-INF/applicationContext.xml");

      // compare the objects from different contexts
      ShoppingCart firstShoppingCart = (ShoppingCart) firstContext.getBean("shoppingCart");
      ShoppingCart secondShoppingCart = (ShoppingCart) secondContext.getBean("shoppingCart");
      System.out.println("1. Are they the same ? " + (firstShoppingCart == secondShoppingCart));

      // compare the objects from the same context
      ShoppingCart firstShoppingCartBis = (ShoppingCart) firstContext.getBean("shoppingCart");
      System.out.println("2. Are they the same ? "+ (firstShoppingCart == firstShoppingCartBis));
    } catch (Exception e) {
      e.printStackTrace();
    }
  }
}
```

通过执行此代码，你应该得到:

```
1. Are they the same ? false
2. Are they the same ? true
```

所以你可以看到，bean只是一个上下文的单例。这就是为什么你不应该将Spring的单例概念与设计模式中的的单例混合在一起。

但是，如果要为一个定义的bean在一个上下文内可以使用不同的实例，应该怎么做？很简单，你应该将此Bean配置为原型作用域:

```xml
<bean id="shoppingCart" class="com.migo.data.ShoppingCart" scope="prototype">
</bean>
```

现在，在运行以前的代码之后，你可以看到如下输出:

```
1. Are they the same ? false
2. Are they the same ? false
```

我们已经知道两个作用域之间的区别。但在哪种情况下我们应该选择使用单例还是原型？Singleton适用于**无状态的**bean，即没有状态的bean。比如一个`service`，`DAO`或者`controller`。他们都没有自己的状态(**举个简单的例子，一个函数sin(x)，这个函数本身就是无状态的，所以我们现在喜欢的函数式编程也遵循这个理念**)。而是根据传输的参数执行一些操作(作为HTTP请求参数)。另一方面，我们可以通过**状态**bean管理一些状态。比如购物车bean，假如它是一个单例，那么两个不同消费者购买的产品将被放置在同一个对象上。而如果其中一个消费者想要删除一个产品，另一个消费者就铁定不高兴。这也就是**状态类对象应该是原型**。

这里说点题外话，不能确定时间的保证，未来会出一个用Java的代码习惯去解析vue的一些东西，内容已经总结完毕，也应用到自己的项目中了，然后得出的一些方法论，为什么在这里去说，就是因为vue也是遵循这个无状态和状态专门管理的原则的，扯远了，接着进行下一部分。

## 将原型放在单例中，反之亦然

通过上面的描述，很多概念都很清楚了吧，但有时候会发生一些更复杂的情况。第一个是在原型bean中放置单例。显然，如果注入的单例对象真的是一个单例的bean(没有状态)，这个真的没一点问题。想象一下，对于我们的购物车，我们需要注入产品服务。此服务只会检查添加到购物车的产品是否库存。由于服务没有状态，并且会基于在方法签名中所传递的对象进行验证，因此不存在风险。

另一方面，将原型bean放在单例中需要做更多的工作。我们不能在单例bean中通过使用自动注入(比如`@Autowired`注解)注入原型bean。当Spring初始化所有具有依赖关系的单例bean时，这些注入只会执行一次。这也就意味着在以下代码，`ShoppingCart`的实例将始终是相同的:

```java
@Controller
public class TestController {
  @Autowired
  private ShoppingCart shoppingCart;

  @RequestMapping(value = "/addProduct/{productName}")
  public String testAdd(@PathVariable(value="productName") String productName) {
    Product product = new Product();
    product.setName(productName);
    this.shoppingCart.addProduct(product);
    LOGGER.debug("ShoppingCart is "+this.shoppingCart);
    return "test";
  }
}
```

编译此类并进行一些URL调用:[http://localhost:8080/addProduct/ice%20tea，http://localhost:8080/addProduct/milk。你将看到如下输出的顺序](http://localhost:8080/addProduct/ice%20tea%EF%BC%8Chttp://localhost:8080/addProduct/milk%E3%80%82%E4%BD%A0%E5%B0%86%E7%9C%8B%E5%88%B0%E5%A6%82%E4%B8%8B%E8%BE%93%E5%87%BA%E7%9A%84%E9%A1%BA%E5%BA%8F):

```shell
// after http://localhost:8080/addProduct/ice%20tea
ShoppingCart is ShoppingCart {products: [Product {ice tea}]}
// after http://localhost:8080/addProduct/milk
ShoppingCart is ShoppingCart {products: [Product {ice tea}, Product {milk}]}
```

为了在按照我们预想情况下工作(要求不一样的`ShoppingCart`)，我们可以通过bean工厂手动获取`ShoppingCart`实例(这样就可以再一次生成一个不一样的`ShoppingCart`实例了):

```java
@Controller
public class TestController {
  @Autowired
  private ApplicationContext context;

  @RequestMapping(value = "/addProduct/{productName}")
  public String testAdd(@PathVariable(value="productName") String productName) {
    Product product = new Product();
    product.setName(productName);

    ShoppingCart shoppingCart = (ShoppingCart) context.getBean("shoppingCart");
    shoppingCart.addProduct(product);
    LOGGER.debug("ShoppingCart is "+shoppingCart);
    return "test";
  }
}
```

这样，你就可以日志中看到，每次调用都会有新的`ShoppingCart`实例的生成:

```shell
// after http://localhost:8080/addProduct/ice%20tea
ShoppingCart is ShoppingCart {products: [Product {ice tea}]}
// after http://localhost:8080/addProduct/milk
ShoppingCart is ShoppingCart {products: [Product {milk}]}
```

## 方法注入

有没有别的方法在每次调用都会产生一个新实例？这就是接下来要说的**方法注入**的技术。它看起来有点像我们的手动去进行bean的查找，但更优雅。一个可以被上下文所感知(访问应用程序上下文可以得到)的bean将负责在单例bean中生成原型bean实例:

```java
@Service("shoppingCartProvider")
public class ShoppingCartProvider implements ApplicationContextAware {

  private ApplicationContext context;

  @Override
  public void setApplicationContext(ApplicationContext context) throws BeansException {
    this.context = context;
  }

  public ShoppingCart getInstance() {
    return (ShoppingCart) context.getBean("shoppingCart");
  }

}
```

经过上面的修改，controller这里相应修改:

```java
@Controller
public class TestController {
  @Autowired
  private ShoppingCartProvider shoppingCartProvider;

  @RequestMapping(value = "/addProduct/{productName}")
  public String testAdd(@PathVariable(value="productName") String productName) {
    Product product = new Product();
    product.setName(productName);

    ShoppingCart shoppingCart = shoppingCartProvider.getInstance();
    shoppingCart.addProduct(product);
    System.out.println("ShoppingCart is "+shoppingCart);
    return "test";
  }
}
```

也可以在XML配置文件中定义。里面会有一个属性引用原型bean，并允许在每次调用时创建新的实例。它可以很轻松地在一个bean中混合更多东西:

```xml
 <bean id="shoppingCartProvider" class="com.migo.data.ShoppingCartProvider">
        <lookup-method name="getInstance" bean="shoppingCart">
    	</lookup-method>
</bean>

    <bean id="shoppingCart" class="com.migo.data.ShoppingCart" scope="prototype">
</bean>
```

```java
public abstract class ShoppingCartProvider   {
  public abstract ShoppingCart getInstance();
}
```

`Controller`的代码与实现`ApplicationContextAware`接口的provider的那个例子是一样的。而区别也仅在于provider的bean定义和实现。该定义包含一个标签查找方法。它指定必须使用哪个方法来获取bean属性中指定的bean的新实例。在我们的这个例子中，我们通过调用`ShoppingCartProvider`类的`getInstance`方法来寻找新的`ShoppingCart`的实例。需要注意的一点，类和方法都可以是抽象的。通过这样做，你可以让Spring生成将实现该方法并返回所需bean的子类。如果这个方法不是抽象的，Spring会重写覆盖它。

## Spring中的Bean类

单例的源码实现主要存在于**org.springframework.beans**和**org.springframework.context**包中。首先，从Bean包中查看**BeanFactory**接口。它包含两个我们绝对感兴趣的方法，可用来确定bean是单例还是原型:

- *boolean isSingleton(String name)throws NoSuchBeanDefinitionException*
- *boolean isPrototype(String name)throws NoSuchBeanDefinitionException*

接下来，我们来深入一下`AbstractFactoryBean`,从这个类的注释可以知道它是作为“`FactoryBean实现的简单模板超类(还是直白翻译下比较好，说默认实现也觉得不靠谱)`”。它包含一个用来返回单例或创建原型bean的`getObject`方法的实现。原型和单例是通过`createInstance`方法在不同的时间段进行的。

```java
/**
 * Simple template superclass for {@link FactoryBean} implementations that
 * creates a singleton or a prototype object, depending on a flag.
 *
 * <p>If the "singleton" flag is {@code true} (the default),
 * this class will create the object that it creates exactly once
 * on initialization and subsequently return said singleton instance
 * on all calls to the {@link #getObject()} method.
 *
 * <p>Else, this class will create a new instance every time the
 * {@link #getObject()} method is invoked. Subclasses are responsible
 * for implementing the abstract {@link #createInstance()} template
 * method to actually create the object(s) to expose.
 *
 * @author Juergen Hoeller
 * @author Keith Donald
 * @since 1.0.2
 * @see #setSingleton
 * @see #createInstance()
 */
public abstract class AbstractFactoryBean<T>
		implements FactoryBean<T>, BeanClassLoaderAware, BeanFactoryAware, InitializingBean, DisposableBean {
	/**
	 * Expose the singleton instance or create a new prototype instance.
	 * @see #createInstance()
	 * @see #getEarlySingletonInterfaces()
	 */
	@Override
	public final T getObject() throws Exception {
		if (isSingleton()) {
			return (this.initialized ? this.singletonInstance : getEarlySingletonInstance());
		}
		else {
			return createInstance();
		}
	}
		...
	/**
	 * Template method that subclasses must override to construct
	 * the object returned by this factory.
	 * <p>Invoked on initialization of this FactoryBean in case of
	 * a singleton; else, on each {@link #getObject()} call.
	 * @return the object returned by this factory
	 * @throws Exception if an exception occurred during object creation
	 * @see #getObject()
	 */
	protected abstract T createInstance() throws Exception;
	...
	}
```

另一个我们会感兴趣的一个点是`BeanDefinition接口`。`bean如其名`，它定义了一个bean属性，例如:scope，class name，factory method name，properties或constructor arguments。

**org.springframework.beans.factory.config.BeanDefinition**

```java
/**
 * A BeanDefinition describes a bean instance, which has property values,
 * constructor argument values, and further information supplied by
 * concrete implementations.
 *
 * <p>This is just a minimal interface: The main intention is to allow a
 * {@link BeanFactoryPostProcessor} such as {@link PropertyPlaceholderConfigurer}
 * to introspect and modify property values and other bean metadata.
 *
 * @author Juergen Hoeller
 * @author Rob Harrop
 * @since 19.03.2004
 * @see ConfigurableListableBeanFactory#getBeanDefinition
 * @see org.springframework.beans.factory.support.RootBeanDefinition
 * @see org.springframework.beans.factory.support.ChildBeanDefinition
 */
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {
  //限于篇幅，请自行查看源码，能发现很多有用的东西
}
```

想要看到bean被初始化的位置，我们需要跳转到context包中，更准确地说就是在`AbstractApplicationContext`类(这个类我们已经接触过好多次了)中。在它的**public void refresh()throws BeansException，IllegalStateException**我们可以找到一些关于bean创建的片段，特别是:

- **protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory)**:实现**org.springframework.beans.factory.config .BeanFactoryPostProcessor**接口的所有bean 都被初始化和调用。这种类型bean允许修改另一个bean的属性或构造函数参数(请看**PostProcessorRegistrationDelegate**的相应代码可以知道，使用**BeanFactoryPostProcessor**来处理我们所要用**beanFactory**生成的bean，这里可以直接把**beanFactory**看成是我们需要的bean即可)。但是请注意，在此阶段只能修改bean定义。**“正常”**bean实例尚未创建。关于这块会请参考文章[Spring中的bean工厂后置处理器](https://muyinchen.github.io/2017/09/16/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-Spring%E4%B8%AD%E7%9A%84bean%E5%B7%A5%E5%8E%82%E5%90%8E%E7%BD%AE%E5%A4%84%E7%90%86%E5%99%A8/)。

```java
/**
	 * Instantiate and invoke all registered BeanFactoryPostProcessor beans,
	 * respecting explicit order if given.
	 * <p>Must be called before singleton instantiation.
	 */
	protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

		// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
		// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
		if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}
	}
```

**org.springframework.context.support.PostProcessorRegistrationDelegate**

```java
/**
 * Delegate for AbstractApplicationContext's post-processor handling.
 *
 * @author Juergen Hoeller
 * @since 4.0
 */
class PostProcessorRegistrationDelegate {

	public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

		// Invoke BeanDefinitionRegistryPostProcessors first, if any.
		Set<String> processedBeans = new HashSet<>();

		if (beanFactory instanceof BeanDefinitionRegistry) {
			BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
			List<BeanFactoryPostProcessor> regularPostProcessors = new LinkedList<>();
			List<BeanDefinitionRegistryPostProcessor> registryPostProcessors =
					new LinkedList<>();

			for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
				if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
					BeanDefinitionRegistryPostProcessor registryPostProcessor =
							(BeanDefinitionRegistryPostProcessor) postProcessor;
					registryPostProcessor.postProcessBeanDefinitionRegistry(registry);
					registryPostProcessors.add(registryPostProcessor);
				}
				else {
					regularPostProcessors.add(postProcessor);
				}
			}
...
        }
```

- **protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory)**:这里上下文实例化并调用实现了**org.springframework.beans.factory.config.BeanPostProcessor**接口的所有bean 。实现此接口的bean包含可以在其他bean初始化之前或之后调用的回调。因为内容比较多，关于这块会请参考文章[Spring中的bean工厂后置处理器](https://muyinchen.github.io/2017/09/16/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-Spring%E4%B8%AD%E7%9A%84bean%E5%B7%A5%E5%8E%82%E5%90%8E%E7%BD%AE%E5%A4%84%E7%90%86%E5%99%A8/)。

```java
/**
	 * Instantiate and invoke all registered BeanPostProcessor beans,
	 * respecting explicit order if given.
	 * <p>Must be called before any instantiation of application beans.
	 */
	protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
	}
```

- **protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory)**:主要调用定义在**org.springframework.beans.factory.config.ConfigurableListableBeanFactory**接口内的`preInstantiateSingletons`方法。该方法的目的是实例化所有被定义为非延迟加载的bean。如果在应用程序上下文加载时遇到BeansException异常，则可能来自此方法。当bean无法创建时，它会抛出BeansException异常。

```java
/**
	 * Finish the initialization of this context's bean factory,
	 * initializing all remaining singleton beans.
	 */
	protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no bean post-processor
		// (such as a PropertyPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		beanFactory.freezeConfiguration();

		// Instantiate all remaining (non-lazy-init) singletons.
		beanFactory.preInstantiateSingletons();
	}
```

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

## 总结

首先，我们讲了单例和原型作用域之间的区别。第一个为每个容器创建一个对象，而第二个在每个请求时创建一个新的bean对象。单例和原型都可以一起交叉使用，但原型不能通过`@Autowired`或其他注入方式来解决。它们应该使用`getBean()方法`或`方法查找`来生成新实例。最后随意说了一说关于bean及其初始化的内容。

# 666. 彩蛋

如果你对 Spring 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)