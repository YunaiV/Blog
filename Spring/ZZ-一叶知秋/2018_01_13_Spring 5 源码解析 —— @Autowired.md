title: Spring 5 源码解析 —— @Autowired
date: 2018-01-13
tag: 
categories: Spring
permalink: Spring/@Autowired
author: 一叶知秋
from_url: https://muyinchen.github.io/2017/08/23/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— @Autowired/
wechat_url: 

-------

摘要: 原创出处 https://muyinchen.github.io/2017/08/23/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— @Autowired/ 「一叶知秋」欢迎转载，保留摘要，谢谢！

- [@Autowired所具有的功能](http://www.iocoder.cn/Spring/@Autowired/)
- [在Spring中如何使用@Autowired](http://www.iocoder.cn/Spring/@Autowired/)
- [@Autowired注解背后的工作原理？](http://www.iocoder.cn/Spring/@Autowired/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

> 你有没有思考过Spring中的@Autowired注解？通常用于方便依赖注入，而隐藏在这个过程之后的机制到底是怎样，将在本篇中进行讲述。

## @Autowired所具有的功能

`@Autowired`是一个用来执行依赖注入的注解。每当一个`Spring`管理的`bean`发现有这个注解时候，它会直接注入相应的另一个`Spring`管理的`bean`。

**该注解可以在不同的层次上应用:**

- **类字段:**Spring将通过扫描自定义的`packages`(例如在我们所注解的`controllers`)或通过在配置文件中直接查找bean。
- **方法:**使用`@Autowired`注解的每个方法都要用到依赖注入。但要注意的是，方法签名中呈现的所有对象都必须是Spring所管理的bean。如果你有一个方法，比如`setTest(Article article, NoSpringArticle noSpringArt)` ，其中只有一个参数 (*Article article*)是由Spring管理的，那么就将抛出一个**org.springframework.beans.factory.BeanCreationException**异常。这是由于Spring容器里并没有指定的一个或多个参数所指向的bean，所以也就无法解析它们。完整的异常跟踪如下:

```SHELL
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'testController': Injection of autowired dependencies failed; nested exception is org.springframework.beans.factory.BeanCreationException: Could not autowire method: public void org.krams.tutorial.controller.TestController.ix(com.mysite.controller.IndexController,com.mysite.nospring.data.Article); nested exception is org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type [com.mysite.nospring.data.Article] found for dependency: expected at least 1 bean which qualifies as autowire candidate for this dependency. Dependency annotations: {}
```

- **构造函数:**`@Autowired`的工作方式和方法相同。

**对象注入需要遵循一些规则。一个bean可以按照下面的方式注入:**

- **名称:**bean解析是通过bean名称(看后面的例子)。
- **类型:**解析过程基于bean的类型。

在某些情况下，`@Autowired`应该通过`@Qualifier`注解协作注入。例如下面几个是相同类型的bean:

```XML
<bean name="comment1" class="com.migo.Comment">
    <property name="text" value="Content of the 1st comment" />
</bean>

<bean name="comment2" class="com.migo.Comment">
    <property name="text" value="Content of the 2nd comment" />
</bean>
```

上面这种情况，假如只是一个简单的`@Autowired`，`Spring`根本不知道你要注入哪个`bean`。这就是为什么我们要使用`@Qualifier(value =“beanName”)`这个注解。在我们的例子中，要从 `com.migo.Comment`这个类型的bean中区分`comment1`,`comment2`，我们可以写下面的代码:

```Java
@Qualifier(value="comment1")
@Autowired
private Comment firstComment;

@Qualifier(value="comment2")
@Autowired
private Comment secondComment;
```

## 在Spring中如何使用@Autowired

正如前面部分所看到的，我们知道了在Spring中实现`@Autowired`的不同方法。在这一部分中，我们将使用`XML`配置的方式激活`@Autowired`注解来自动注入。然后，我们将编写一个简单的类并配置一些bean。最后，我们将分别在另外两个类中使用它们:由@Controller注解的控件和不由Spring所管理的类。(为什么用`XML`配置来做例子，我觉得这样更直观，其实XML和使用注解没多少区别，都是往容器里添加一些bean和组织下彼此之间的依赖而已，不必要非要拘泥于一种形式，哪种顺手用哪种，不过`Springboot`自定义的这些还是推荐使用注解了)

我们从启动注解的自动注入开始:

```XML
<context:annotation-config />
```

你必须将上面这个放在应用程序上下文配置中。它可以使在遇到`@Autowired注解`时启用依赖注入。

现在，我们来编写和配置我们的bean:

```Java
// beans first
public class Comment {

    private String content;

    public void setContent(String content) {
        this.content = content;
    }

    public String getContent() {
        return this.content;
    }

}

// sample controller
@Controller
public class TestController {

    @Qualifier(value="comment1")
    @Autowired
    private Comment firstComment;

    @Qualifier(value="comment2")
    @Autowired
    private Comment secondComment;

    @RequestMapping(value = "/test", method = RequestMethod.GET)
    public String test() {
        System.out.println("1st comment text: "+firstComment.getText());
        System.out.println("2nd comment text: "+secondComment.getText());
        return "test";
    }

}

// no-Spring managed class
public class TestNoSpring {

    @Autowired
    private Comment comment;


    public void testComment(String content) {
        if (comment == null) {
            System.out.println("Comment's instance wasn't autowired because this class is not Spring-managed bean");
        } else {
            comment.setContent(content);
            System.out.println("Comment's content: "+comment.getContent());
        }
    }

}
```

XML配置(在前面部分已经看到过):

```XML
<bean name="comment1" class="com.specimen.exchanger.Comment">
    <property name="content" value="Content of the 1st comment" />
</bean>

<bean name="comment2" class="com.specimen.exchanger.Comment">
    <property name="content" value="Content of the 2nd comment" />
</bean>
```

现在，我们打开`http://localhost:8080/test`来运行`TestController`。如预期的那样，`TestController`的注解字段正确地自动注入，而`TestNoSpring`的注解字段并没有注入进去:

```SHELL
1st comment text: Content of the 1st comment
2nd comment text: Content of the 2nd comment
Comment's instance wasn't autowired because this class is not Spring-managed bean
```

哪里不对 ？TestNoSpring类不由Spring所管理。这就是为什么Spring不能注入Comment实例的依赖。我们将在下一部分中解释这个概念。

## @Autowired注解背后的工作原理？

在讨论代码细节之前，我们再来了解下基础知识。Spring管理可用于整个应用程序的Java对象bean。他们所在的Spring容器，被称为应用程序上下文。这意味着我们不需要处理他们的生命周期(初始化，销毁)。该任务由此容器来完成。另外，该上下文具有入口点，在Web应用程序中，是dispatcher servlet。容器(也就是该上下文)会在它那里被启动并且所有的bean都会被注入。

说的再清楚点，请看`<context:annotation-config />`的定义:

```xml
<xsd:element name="annotation-config">
		<xsd:annotation>
			<xsd:documentation><![CDATA[
	Activates various annotations to be detected in bean classes: Spring's @Required and
	@Autowired, as well as JSR 250's @PostConstruct, @PreDestroy and @Resource (if available),
	JAX-WS's @WebServiceRef (if available), EJB 3's @EJB (if available), and JPA's
	@PersistenceContext and @PersistenceUnit (if available). Alternatively, you may
	choose to activate the individual BeanPostProcessors for those annotations.

	Note: This tag does not activate processing of Spring's @Transactional or EJB 3's
	@TransactionAttribute annotation. Consider the use of the <tx:annotation-driven>
	tag for that purpose.

	See javadoc for org.springframework.context.annotation.AnnotationConfigApplicationContext
	for information on code-based alternatives to bootstrapping annotation-driven support.
			]]></xsd:documentation>
		</xsd:annotation>
	</xsd:element>
```

**可以看出 :** 类内部的注解，如：`@Autowired`、`@Value`、`@Required`、`@Resource`以及`EJB`和`WebSerivce`相关的注解，是容器对Bean对象实例化和依赖注入时，通过容器中注册的Bean后置处理器处理这些注解的。

所以配置了上面这个配置(`<context:component-scan>`假如有配置这个，那么就可以省略`<context:annotation-config />`)后，将隐式地向Spring容器注册`AutowiredAnnotationBeanPostProcessor`、`CommonAnnotationBeanPostProcessor`、`RequiredAnnotationBeanPostProcessor`、`PersistenceAnnotationBeanPostProcessor`以及这4个专门用于处理注解的Bean后置处理器。

当 Spring 容器**启动**时，`AutowiredAnnotationBeanPostProcessor` 将**扫描 Spring 容器中所有 Bean，当发现 Bean 中拥有@Autowired 注解时就找到和其匹配（默认按类型匹配）的 Bean**，**并注入**到对应的地方中去。 源码分析如下:

通过**org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor**可以**实现依赖自动注入**。通过这个类来处理`@Autowired`和`@Value`这俩`Spring注解`。它也可以管理`JSR-303`的`@Inject`注解(如果可用的话)。在`AutowiredAnnotationBeanPostProcessor`构造函数中定义要处理的注解:

```Java
public class AutowiredAnnotationBeanPostProcessor extends InstantiationAwareBeanPostProcessorAdapter
		implements MergedBeanDefinitionPostProcessor, PriorityOrdered, BeanFactoryAware {
		...
	/**
	 * Create a new AutowiredAnnotationBeanPostProcessor
	 * for Spring's standard {@link Autowired} annotation.
	 * <p>Also supports JSR-330's {@link javax.inject.Inject} annotation, if available.
	 */
	@SuppressWarnings("unchecked")
	public AutowiredAnnotationBeanPostProcessor() {
		this.autowiredAnnotationTypes.add(Autowired.class);
		this.autowiredAnnotationTypes.add(Value.class);
		try {
			this.autowiredAnnotationTypes.add((Class<? extends Annotation>)
					ClassUtils.forName("javax.inject.Inject", AutowiredAnnotationBeanPostProcessor.class.getClassLoader()));
			logger.info("JSR-330 'javax.inject.Inject' annotation found and supported for autowiring");
		}
		catch (ClassNotFoundException ex) {
			// JSR-330 API not available - simply skip.
		}
	}
	...
	}
```

之后，有几种方法来对`@Autowired注解`进行处理。

第一个，`private InjectionMetadata buildAutowiringMetadata(final Class<?> clazz)`解析等待**自动注入**类的所有属性。它通过分析所有字段和方法并初始化**org.springframework.beans.factory.annotation.InjectionMetadata**类的实例来实现。

```Java
private InjectionMetadata buildAutowiringMetadata(final Class<?> clazz) {
		LinkedList<InjectionMetadata.InjectedElement> elements = new LinkedList<>();
		Class<?> targetClass = clazz;

		do {
			final LinkedList<InjectionMetadata.InjectedElement> currElements = new LinkedList<>();
			//分析所有字段
			ReflectionUtils.doWithLocalFields(targetClass, field -> {
              //findAutowiredAnnotation(field)此方法后面会解释
				AnnotationAttributes ann = findAutowiredAnnotation(field);
				if (ann != null) {
					if (Modifier.isStatic(field.getModifiers())) {
						if (logger.isWarnEnabled()) {
							logger.warn("Autowired annotation is not supported on static fields: " + field);
						}
						return;
					}
					boolean required = determineRequiredStatus(ann);
					currElements.add(new AutowiredFieldElement(field, required));
				}
			});
			//分析所有方法
			ReflectionUtils.doWithLocalMethods(targetClass, method -> {
				Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
				if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
					return;
				}
				AnnotationAttributes ann = findAutowiredAnnotation(bridgedMethod);
				if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
					if (Modifier.isStatic(method.getModifiers())) {
						if (logger.isWarnEnabled()) {
							logger.warn("Autowired annotation is not supported on static methods: " + method);
						}
						return;
					}
					if (method.getParameterCount() == 0) {
						if (logger.isWarnEnabled()) {
							logger.warn("Autowired annotation should only be used on methods with parameters: " +
									method);
						}
					}
					boolean required = determineRequiredStatus(ann);
					PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
					currElements.add(new AutowiredMethodElement(method, required, pd));
				}
			});

			elements.addAll(0, currElements);
			targetClass = targetClass.getSuperclass();
		}
		while (targetClass != null && targetClass != Object.class);
		//返回一个InjectionMetadata初始化的对象实例
		return new InjectionMetadata(clazz, elements);
	}
...

  /**
	 * 'Native' processing method for direct calls with an arbitrary target instance,
	 * resolving all of its fields and methods which are annotated with {@code @Autowired}.
	 * @param bean the target instance to process
	 * @throws BeanCreationException if autowiring failed
	 */
	public void processInjection(Object bean) throws BeanCreationException {
		Class<?> clazz = bean.getClass();
		InjectionMetadata metadata = findAutowiringMetadata(clazz.getName(), clazz, null);
		try {
			metadata.inject(bean, null, null);
		}
		catch (BeanCreationException ex) {
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					"Injection of autowired dependencies failed for class [" + clazz + "]", ex);
		}
	}
```

`InjectionMetadata`类包含要注入的元素的列表。注入是通过Java的API `Reflection (Field set(Object obj, Object value)` 或`Method invoke(Object obj，Object ... args)`方法完成的。此过程直接在`AutowiredAnnotationBeanPostProcessor`的方法中调用`public void processInjection(Object bean) throws BeanCreationException`。它将所有可注入的bean检索为`InjectionMetadata`实例，并调用它们的`inject()`方法。

```Java
public class InjectionMetadata {
  ...
public void inject(Object target, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
		Collection<InjectedElement> checkedElements = this.checkedElements;
		Collection<InjectedElement> elementsToIterate =
				(checkedElements != null ? checkedElements : this.injectedElements);
		if (!elementsToIterate.isEmpty()) {
			boolean debug = logger.isDebugEnabled();
			for (InjectedElement element : elementsToIterate) {
				if (debug) {
					logger.debug("Processing injected element of bean '" + beanName + "': " + element);
				}
              	//看下面静态内部类的方法
				element.inject(target, beanName, pvs);
			}
		}
	}
  ...
    public static abstract class InjectedElement {

		protected final Member member;

		protected final boolean isField;
      ...
        /**
		 * Either this or {@link #getResourceToInject} needs to be overridden.
		 */
		protected void inject(Object target, @Nullable String requestingBeanName, @Nullable PropertyValues pvs)
				throws Throwable {

			if (this.isField) {
				Field field = (Field) this.member;
				ReflectionUtils.makeAccessible(field);
				field.set(target, getResourceToInject(target, requestingBeanName));
			}
			else {
				if (checkPropertySkipping(pvs)) {
					return;
				}
				try {
                  	//具体的注入看此处咯
					Method method = (Method) this.member;
					ReflectionUtils.makeAccessible(method);
					method.invoke(target, getResourceToInject(target, requestingBeanName));
				}
				catch (InvocationTargetException ex) {
					throw ex.getTargetException();
				}
			}
		}
      ...
    }
}
```

`AutowiredAnnotationBeanPostProcessor`类中的另一个重要方法是**private AnnotationAttributes findAutowiredAnnotation(AccessibleObject ao)**。它通过分析属于一个字段或一个方法的所有注解来查找`@Autowired`注解。如果未找到`@Autowired`注解，则返回`null`，字段或方法也就视为不可注入。

```Java
@Nullable
	private AnnotationAttributes findAutowiredAnnotation(AccessibleObject ao) {
		if (ao.getAnnotations().length > 0) {
			for (Class<? extends Annotation> type : this.autowiredAnnotationTypes) {
				AnnotationAttributes attributes = AnnotatedElementUtils.getMergedAnnotationAttributes(ao, type);
				if (attributes != null) {
					return attributes;
				}
			}
		}
		return null;
	}
```

在上面的文章中，我们看到了Spring中自动注入过程。通过整篇文章可以看到，这种依赖注入是一种便捷易操作方式(可以在字段以及方法上完成)，也促使我们逐渐在抛弃XML配置文件。还增强了代码的可读性。

# 666. 彩蛋

如果你对 Spring 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)