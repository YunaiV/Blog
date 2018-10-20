title: Spring 5 源码解析 —— 使用 Spring AnnotationUtils 处理注解
date: 2018-01-21
tag: 
categories: Spring
permalink: Spring/AnnotationUtils
author: 一叶知秋
from_url: https://muyinchen.github.io/2017/09/22/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— %E4%BD%BF%E7%94%A8Spring%20AnnotationUtils%E5%A4%84%E7%90%86%E6%B3%A8%E8%A7%A3/
wechat_url: 

-------

摘要: 原创出处 https://muyinchen.github.io/2017/09/22/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— %E4%BD%BF%E7%94%A8Spring%20AnnotationUtils%E5%A4%84%E7%90%86%E6%B3%A8%E8%A7%A3/ 「一叶知秋」欢迎转载，保留摘要，谢谢！

- [什么是Spring中的AnnotationUtils类？](http://www.iocoder.cn/Spring/AnnotationUtils/)
- [有哪些地方使用了AnnotationUtils方法？](http://www.iocoder.cn/Spring/AnnotationUtils/)
- [AnnotationUtils in Action](http://www.iocoder.cn/Spring/AnnotationUtils/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

通过Java中的注解，程序员可以将配置文件中的一些配置通过使用Java类来实现。例如，在Spring中，通过`@RequestMapping`注解，我们可以直接在`controller`内配置`URL映射`。一般来说，成功者的背后离不开一帮默默支持他的小伙伴，这里同样是，一旦离开其中一个，就甭指望实现所期望的功能，这里要说的一个就是`AnnotationUtils`。

在本文中，我们将会看到AnnotationUtils类是如何给我们提供极大的便利的。首先，我们将关注下其所有可用的方法。进而，我们来看看这些方法用在了什么地方。最后，老规矩，Demo。

代码截的不少，主要还是为了在平时不一定有IDE环境下清楚的看清弄懂上下文的调用关系，也方面读者可以快速的找到相应的代码所在地。

另外，这一篇融合了前面很多篇的细节，有些不清晰明了的地方，请回头看前面的系列。

## 什么是Spring中的AnnotationUtils类？

`AnnotationUtils`是一个专门用于处理复杂注解问题的类。其主要由公共和静态方法组成，它允许在类，方法或字段上检查注解。另外，`AnnotationUtils`不仅仅来做简单的类分析。它也通过查找在超类和接口上的注解来做更多的事情。基于反射的API，`AnnotationUtils`使用**java.lang.reflect的** 3个元素来处理注解:

- **Annotation**:表示注解。


- **AnnotatedElement**:表示被注解元素。
- **Method**:提供某些类或接口中的方法的信息。

现在，我们来看看`AnnotationUtils`类中的最重要的几个public 方法:

- **getAnnotation**:有3个这样的名字的方法存在。 第一个接收参数`Annotation的对象实例`。第二个是`AnnotatedElement`的实例。第三个`getAnnotation`方法接收参数`Method`对象。它们都从字段，类或方法得到并返回注解。

```java
/**
 * Get a single {@link Annotation} of {@code annotationType} from the supplied
 * annotation: either the given annotation itself or a direct meta-annotation
 * thereof.
 * <p>Note that this method supports only a single level of meta-annotations.
 * For support for arbitrary levels of meta-annotations, use one of the
 * {@code find*()} methods instead.
 * @param ann the Annotation to check
 * @param annotationType the annotation type to look for, both locally and as a meta-annotation
 * @return the first matching annotation, or {@code null} if not found
 * @since 4.0
 */
@SuppressWarnings("unchecked")
@Nullable
public static <A extends Annotation> A getAnnotation(Annotation ann, Class<A> annotationType) {
	if (annotationType.isInstance(ann)) {
		return synthesizeAnnotation((A) ann);
	}
	Class<? extends Annotation> annotatedElement = ann.annotationType();
	try {
		return synthesizeAnnotation(annotatedElement.getAnnotation(annotationType), annotatedElement);
	}
	catch (Throwable ex) {
		handleIntrospectionFailure(annotatedElement, ex);
		return null;
	}
}

/**
 * Get a single {@link Annotation} of {@code annotationType} from the supplied
 * {@link AnnotatedElement}, where the annotation is either <em>present</em> or
 * <em>meta-present</em> on the {@code AnnotatedElement}.
 * <p>Note that this method supports only a single level of meta-annotations.
 * For support for arbitrary levels of meta-annotations, use
 * {@link #findAnnotation(AnnotatedElement, Class)} instead.
 * @param annotatedElement the {@code AnnotatedElement} from which to get the annotation
 * @param annotationType the annotation type to look for, both locally and as a meta-annotation
 * @return the first matching annotation, or {@code null} if not found
 * @since 3.1
 */
@Nullable
public static <A extends Annotation> A getAnnotation(AnnotatedElement annotatedElement, Class<A> annotationType) {
	try {
		A annotation = annotatedElement.getAnnotation(annotationType);
		if (annotation == null) {
			for (Annotation metaAnn : annotatedElement.getAnnotations()) {
				annotation = metaAnn.annotationType().getAnnotation(annotationType);
				if (annotation != null) {
					break;
				}
			}
		}
		return (annotation != null ? synthesizeAnnotation(annotation, annotatedElement) : null);
	}
	catch (Throwable ex) {
		handleIntrospectionFailure(annotatedElement, ex);
		return null;
	}
}

/**
 * Get a single {@link Annotation} of {@code annotationType} from the
 * supplied {@link Method}, where the annotation is either <em>present</em>
 * or <em>meta-present</em> on the method.
 * <p>Correctly handles bridge {@link Method Methods} generated by the compiler.
 * <p>Note that this method supports only a single level of meta-annotations.
 * For support for arbitrary levels of meta-annotations, use
 * {@link #findAnnotation(Method, Class)} instead.
 * @param method the method to look for annotations on
 * @param annotationType the annotation type to look for
 * @return the first matching annotation, or {@code null} if not found
 * @see org.springframework.core.BridgeMethodResolver#findBridgedMethod(Method)
 * @see #getAnnotation(AnnotatedElement, Class)
 */
@Nullable
public static <A extends Annotation> A getAnnotation(Method method, Class<A> annotationType) {
	Method resolvedMethod = BridgeMethodResolver.findBridgedMethod(method);
	return getAnnotation((AnnotatedElement) resolvedMethod, annotationType);
}
```

- **getRepeatableAnnotations**:通过向参数中传递`AnnotatedElement`和`所要查找的注解类型`来访问，有两个使用此名称的方法。 这两个方法从所提供的`AnnotatedElement`上得到可重复的注解(即传入的`annotationType`)对应的元素。例如，它可以返回使用`@RequestMapping`注解所注解的方法(请看下面的源码)。

```java
	/**
	 * Get the <em>repeatable</em> {@linkplain Annotation annotations} of
	 * {@code annotationType} from the supplied {@link AnnotatedElement}, where
	 * such annotations are either <em>present</em>, <em>indirectly present</em>,
	 * or <em>meta-present</em> on the element.
	 * <p>This method mimics the functionality of Java 8's
	 * {@link java.lang.reflect.AnnotatedElement#getAnnotationsByType(Class)}
	 * with support for automatic detection of a <em>container annotation</em>
	 * declared via @{@link java.lang.annotation.Repeatable} (when running on
	 * Java 8 or higher) and with additional support for meta-annotations.
	 * <p>Handles both single annotations and annotations nested within a
	 * <em>container annotation</em>.
	 * <p>Correctly handles <em>bridge methods</em> generated by the
	 * compiler if the supplied element is a {@link Method}.
	 * <p>Meta-annotations will be searched if the annotation is not
	 * <em>present</em> on the supplied element.
	 * @param annotatedElement the element to look for annotations on
	 * @param annotationType the annotation type to look for
	 * @return the annotations found or an empty set (never {@code null})
	 * @since 4.2
	 * @see #getRepeatableAnnotations(AnnotatedElement, Class, Class)
	 * @see #getDeclaredRepeatableAnnotations(AnnotatedElement, Class, Class)
	 * @see AnnotatedElementUtils#getMergedRepeatableAnnotations(AnnotatedElement, Class)
	 * @see org.springframework.core.BridgeMethodResolver#findBridgedMethod
	 * @see java.lang.annotation.Repeatable
	 * @see java.lang.reflect.AnnotatedElement#getAnnotationsByType
	 */
	@Nullable
	public static <A extends Annotation> Set<A> getRepeatableAnnotations(AnnotatedElement annotatedElement,
			Class<A> annotationType) {

		return getRepeatableAnnotations(annotatedElement, annotationType, null);
	}

	/**
	 * Get the <em>repeatable</em> {@linkplain Annotation annotations} of
	 * {@code annotationType} from the supplied {@link AnnotatedElement}, where
	 * such annotations are either <em>present</em>, <em>indirectly present</em>,
	 * or <em>meta-present</em> on the element.
	 * <p>This method mimics the functionality of Java 8's
	 * {@link java.lang.reflect.AnnotatedElement#getAnnotationsByType(Class)}
	 * with additional support for meta-annotations.
	 * <p>Handles both single annotations and annotations nested within a
	 * <em>container annotation</em>.
	 * <p>Correctly handles <em>bridge methods</em> generated by the
	 * compiler if the supplied element is a {@link Method}.
	 * <p>Meta-annotations will be searched if the annotation is not
	 * <em>present</em> on the supplied element.
	 * @param annotatedElement the element to look for annotations on
	 * @param annotationType the annotation type to look for
	 * @param containerAnnotationType the type of the container that holds
	 * the annotations; may be {@code null} if a container is not supported
	 * or if it should be looked up via @{@link java.lang.annotation.Repeatable}
	 * when running on Java 8 or higher
	 * @return the annotations found or an empty set (never {@code null})
	 * @since 4.2
	 * @see #getRepeatableAnnotations(AnnotatedElement, Class)
	 * @see #getDeclaredRepeatableAnnotations(AnnotatedElement, Class)
	 * @see #getDeclaredRepeatableAnnotations(AnnotatedElement, Class, Class)
	 * @see AnnotatedElementUtils#getMergedRepeatableAnnotations(AnnotatedElement, Class, Class)
	 * @see org.springframework.core.BridgeMethodResolver#findBridgedMethod
	 * @see java.lang.annotation.Repeatable
	 * @see java.lang.reflect.AnnotatedElement#getAnnotationsByType
	 */
	public static <A extends Annotation> Set<A> getRepeatableAnnotations(AnnotatedElement annotatedElement,
			Class<A> annotationType, @Nullable Class<? extends Annotation> containerAnnotationType) {

		Set<A> annotations = getDeclaredRepeatableAnnotations(annotatedElement, annotationType, containerAnnotationType);
		if (!annotations.isEmpty()) {
			return annotations;
		}

		if (annotatedElement instanceof Class) {
			Class<?> superclass = ((Class<?>) annotatedElement).getSuperclass();
			if (superclass != null && Object.class != superclass) {
				return getRepeatableAnnotations(superclass, annotationType, containerAnnotationType);
			}
		}

		return getRepeatableAnnotations(annotatedElement, annotationType, containerAnnotationType, false);
	}
/**
	 * Get the declared <em>repeatable</em> {@linkplain Annotation annotations}
	 * of {@code annotationType} from the supplied {@link AnnotatedElement},
	 * where such annotations are either <em>directly present</em>,
	 * <em>indirectly present</em>, or <em>meta-present</em> on the element.
	 * <p>This method mimics the functionality of Java 8's
	 * {@link java.lang.reflect.AnnotatedElement#getDeclaredAnnotationsByType(Class)}
	 * with support for automatic detection of a <em>container annotation</em>
	 * declared via @{@link java.lang.annotation.Repeatable} (when running on
	 * Java 8 or higher) and with additional support for meta-annotations.
	 * <p>Handles both single annotations and annotations nested within a
	 * <em>container annotation</em>.
	 * <p>Correctly handles <em>bridge methods</em> generated by the
	 * compiler if the supplied element is a {@link Method}.
	 * <p>Meta-annotations will be searched if the annotation is not
	 * <em>present</em> on the supplied element.
	 * @param annotatedElement the element to look for annotations on
	 * @param annotationType the annotation type to look for
	 * @return the annotations found or an empty set (never {@code null})
	 * @since 4.2
	 * @see #getRepeatableAnnotations(AnnotatedElement, Class)
	 * @see #getRepeatableAnnotations(AnnotatedElement, Class, Class)
	 * @see #getDeclaredRepeatableAnnotations(AnnotatedElement, Class, Class)
	 * @see AnnotatedElementUtils#getMergedRepeatableAnnotations(AnnotatedElement, Class)
	 * @see org.springframework.core.BridgeMethodResolver#findBridgedMethod
	 * @see java.lang.annotation.Repeatable
	 * @see java.lang.reflect.AnnotatedElement#getDeclaredAnnotationsByType
	 */
	public static <A extends Annotation> Set<A> getDeclaredRepeatableAnnotations(AnnotatedElement annotatedElement,
			Class<A> annotationType) {

		return getDeclaredRepeatableAnnotations(annotatedElement, annotationType, null);
	}

	/**
	 * Get the declared <em>repeatable</em> {@linkplain Annotation annotations}
	 * of {@code annotationType} from the supplied {@link AnnotatedElement},
	 * where such annotations are either <em>directly present</em>,
	 * <em>indirectly present</em>, or <em>meta-present</em> on the element.
	 * <p>This method mimics the functionality of Java 8's
	 * {@link java.lang.reflect.AnnotatedElement#getDeclaredAnnotationsByType(Class)}
	 * with additional support for meta-annotations.
	 * <p>Handles both single annotations and annotations nested within a
	 * <em>container annotation</em>.
	 * <p>Correctly handles <em>bridge methods</em> generated by the
	 * compiler if the supplied element is a {@link Method}.
	 * <p>Meta-annotations will be searched if the annotation is not
	 * <em>present</em> on the supplied element.
	 * @param annotatedElement the element to look for annotations on
	 * @param annotationType the annotation type to look for
	 * @param containerAnnotationType the type of the container that holds
	 * the annotations; may be {@code null} if a container is not supported
	 * or if it should be looked up via @{@link java.lang.annotation.Repeatable}
	 * when running on Java 8 or higher
	 * @return the annotations found or an empty set (never {@code null})
	 * @since 4.2
	 * @see #getRepeatableAnnotations(AnnotatedElement, Class)
	 * @see #getRepeatableAnnotations(AnnotatedElement, Class, Class)
	 * @see #getDeclaredRepeatableAnnotations(AnnotatedElement, Class)
	 * @see AnnotatedElementUtils#getMergedRepeatableAnnotations(AnnotatedElement, Class, Class)
	 * @see org.springframework.core.BridgeMethodResolver#findBridgedMethod
	 * @see java.lang.annotation.Repeatable
	 * @see java.lang.reflect.AnnotatedElement#getDeclaredAnnotationsByType
	 */
	public static <A extends Annotation> Set<A> getDeclaredRepeatableAnnotations(AnnotatedElement annotatedElement,
			Class<A> annotationType, @Nullable Class<? extends Annotation> containerAnnotationType) {
		//得到一个set集合，看下面源码
		return getRepeatableAnnotations(annotatedElement, annotationType, containerAnnotationType, true);
	}

	/**
	 * 最主要的实现是这个方法
	 * Perform the actual work for {@link #getRepeatableAnnotations(AnnotatedElement, Class, Class)}
	 * and {@link #getDeclaredRepeatableAnnotations(AnnotatedElement, Class, Class)}.
	 * <p>Correctly handles <em>bridge methods</em> generated by the
	 * compiler if the supplied element is a {@link Method}.
	 * <p>Meta-annotations will be searched if the annotation is not
	 * <em>present</em> on the supplied element.
	 * @param annotatedElement the element to look for annotations on
	 * @param annotationType the annotation type to look for
	 * @param containerAnnotationType the type of the container that holds
	 * the annotations; may be {@code null} if a container is not supported
	 * or if it should be looked up via @{@link java.lang.annotation.Repeatable}
	 * when running on Java 8 or higher
	 * @param declaredMode {@code true} if only declared annotations (i.e.,
	 * directly or indirectly present) should be considered
	 * @return the annotations found or an empty set (never {@code null})
	 * @since 4.2
	 * @see org.springframework.core.BridgeMethodResolver#findBridgedMethod
	 * @see java.lang.annotation.Repeatable
	 */
	private static <A extends Annotation> Set<A> getRepeatableAnnotations(AnnotatedElement annotatedElement,
			Class<A> annotationType, @Nullable Class<? extends Annotation> containerAnnotationType, boolean declaredMode) {

		Assert.notNull(annotatedElement, "AnnotatedElement must not be null");
		Assert.notNull(annotationType, "Annotation type must not be null");

		try {
			if (annotatedElement instanceof Method) {
				annotatedElement = BridgeMethodResolver.findBridgedMethod((Method) annotatedElement);
			}
          //返回的这个set集合包含的是要查找的注解及其被注解的元素，看下面源码
			return new AnnotationCollector<>(annotationType, containerAnnotationType, declaredMode).getResult(annotatedElement);
		}
		catch (Throwable ex) {
			handleIntrospectionFailure(annotatedElement, ex);
			return Collections.emptySet();
		}
	}

...
  private static class AnnotationCollector<A extends Annotation> {

		private final Class<A> annotationType;

		@Nullable
		private final Class<? extends Annotation> containerAnnotationType;

		private final boolean declaredMode;

		private final Set<AnnotatedElement> visited = new HashSet<>();

		private final Set<A> result = new LinkedHashSet<>();

		AnnotationCollector(Class<A> annotationType,
				@Nullable Class<? extends Annotation> containerAnnotationType, boolean declaredMode) {

			this.annotationType = annotationType;
			this.containerAnnotationType = (containerAnnotationType != null ? containerAnnotationType :
					resolveContainerAnnotationType(annotationType));
			this.declaredMode = declaredMode;
		}

		Set<A> getResult(AnnotatedElement element) {
			process(element);
			return Collections.unmodifiableSet(this.result);
		}

		@SuppressWarnings("unchecked")
		private void process(AnnotatedElement element) {
			if (this.visited.add(element)) {
				try {
					Annotation[] annotations = (this.declaredMode ? element.getDeclaredAnnotations() : element.getAnnotations());//从元素上得到注解
					for (Annotation ann : annotations) {
						Class<? extends Annotation> currentAnnotationType = ann.annotationType();
						if (ObjectUtils.nullSafeEquals(this.annotationType, currentAnnotationType)) {							//这个set集合添加的是要查找的注解及其被注解的元素
							this.result.add(synthesizeAnnotation((A) ann, element));
						}
						else if (ObjectUtils.nullSafeEquals(this.containerAnnotationType, currentAnnotationType)) {
							this.result.addAll(getValue(element, ann));
						}
						else if (!isInJavaLangAnnotationPackage(currentAnnotationType)) {
							process(currentAnnotationType);
						}
					}
				}
				catch (Throwable ex) {
					handleIntrospectionFailure(element, ex);
				}
			}
		}
```

- **findAnnotation**:通过传入`AnnotatedElement`和`注解类型`来查找方法或者类对象上的注解。

  ```java
     /**
   * Find a single {@link Annotation} of {@code annotationType} on the
   * supplied {@link AnnotatedElement}.
   * <p>Meta-annotations will be searched if the annotation is not
   * <em>directly present</em> on the supplied element.
   * <p><strong>Warning</strong>: this method operates generically on
   * annotated elements. In other words, this method does not execute
   * specialized search algorithms for classes or methods. If you require
   * the more specific semantics of {@link #findAnnotation(Class, Class)}
   * or {@link #findAnnotation(Method, Class)}, invoke one of those methods
   * instead.
   * @param annotatedElement the {@code AnnotatedElement} on which to find the annotation
   * @param annotationType the annotation type to look for, both locally and as a meta-annotation
   * @return the first matching annotation, or {@code null} if not found
   * @since 4.2
   */
  @Nullable
  public static <A extends Annotation> A findAnnotation(AnnotatedElement annotatedElement, Class<A> annotationType) {
  	Assert.notNull(annotatedElement, "AnnotatedElement must not be null");

  	// Do NOT store result in the findAnnotationCache since doing so could break
  	// findAnnotation(Class, Class) and findAnnotation(Method, Class).
  	A ann = findAnnotation(annotatedElement, annotationType, new HashSet<>());
  	return (ann != null ? synthesizeAnnotation(ann, annotatedElement) : null);
  }

  /**
   	 * Perform the search algorithm for {@link #findAnnotation(AnnotatedElement, Class)}
   	 * avoiding endless recursion by tracking which annotations have already
   	 * been <em>visited</em>.
   	 * @param annotatedElement the {@code AnnotatedElement} on which to find the annotation
   	 * @param annotationType the annotation type to look for, both locally and as a meta-annotation
   	 * @param visited the set of annotations that have already been visited
   	 * @return the first matching annotation, or {@code null} if not found
   	 * @since 4.2
   	 */
   	@Nullable
   	private static <A extends Annotation> A findAnnotation(
   			AnnotatedElement annotatedElement, Class<A> annotationType, Set<Annotation> visited) {
   		try {
   			A annotation = annotatedElement.getDeclaredAnnotation(annotationType);
   			if (annotation != null) {
   				return annotation;
   			}
   			for (Annotation declaredAnn : annotatedElement.getDeclaredAnnotations()) {
   				Class<? extends Annotation> declaredType = declaredAnn.annotationType();
   				if (!isInJavaLangAnnotationPackage(declaredType) && visited.add(declaredAnn)) {
   					annotation = findAnnotation((AnnotatedElement) declaredType, annotationType, visited);
   					if (annotation != null) {
   						return annotation;
   					}
   				}
   			}
   		}
   		catch (Throwable ex) {
   			handleIntrospectionFailure(annotatedElement, ex);
   		}
   		return null;
   	}
  ```

- **isAnnotationDeclaredLocally**:检查注解是否在类中本地声明，而不是继承。

```java
/**
	 * Determine whether an annotation of the specified {@code annotationType}
	 * is declared locally (i.e., <em>directly present</em>) on the supplied
	 * {@code clazz}.
	 * <p>The supplied {@link Class} may represent any type.
	 * <p>Meta-annotations will <em>not</em> be searched.
	 * <p>Note: This method does <strong>not</strong> determine if the annotation
	 * is {@linkplain java.lang.annotation.Inherited inherited}. For greater
	 * clarity regarding inherited annotations, consider using
	 * {@link #isAnnotationInherited(Class, Class)} instead.
	 * @param annotationType the annotation type to look for
	 * @param clazz the class to check for the annotation on
	 * @return {@code true} if an annotation of the specified {@code annotationType}
	 * is <em>directly present</em>
	 * @see java.lang.Class#getDeclaredAnnotations()
	 * @see java.lang.Class#getDeclaredAnnotation(Class)
	 * @see #isAnnotationInherited(Class, Class)
	 */
	public static boolean isAnnotationDeclaredLocally(Class<? extends Annotation> annotationType, Class<?> clazz) {
		Assert.notNull(annotationType, "Annotation type must not be null");
		Assert.notNull(clazz, "Class must not be null");
		try {
			return (clazz.getDeclaredAnnotation(annotationType) != null);
		}
		catch (Throwable ex) {
			handleIntrospectionFailure(clazz, ex);
			return false;
		}
	}
```

- **isAnnotationInherited**:检查注解是否从另一个类继承(即未在本地声明)。

```java
/**
	 * Determine whether an annotation of the specified {@code annotationType}
	 * is <em>present</em> on the supplied {@code clazz} and is
	 * {@linkplain java.lang.annotation.Inherited inherited} (i.e., not
	 * <em>directly present</em>).
	 * <p>Meta-annotations will <em>not</em> be searched.
	 * <p>If the supplied {@code clazz} is an interface, only the interface
	 * itself will be checked. In accordance with standard meta-annotation
	 * semantics in Java, the inheritance hierarchy for interfaces will not
	 * be traversed. See the {@linkplain java.lang.annotation.Inherited javadoc}
	 * for the {@code @Inherited} meta-annotation for further details regarding
	 * annotation inheritance.
	 * @param annotationType the annotation type to look for
	 * @param clazz the class to check for the annotation on
	 * @return {@code true} if an annotation of the specified {@code annotationType}
	 * is <em>present</em> and <em>inherited</em>
	 * @see Class#isAnnotationPresent(Class)
	 * @see #isAnnotationDeclaredLocally(Class, Class)
	 */
	public static boolean isAnnotationInherited(Class<? extends Annotation> annotationType, Class<?> clazz) {
		Assert.notNull(annotationType, "Annotation type must not be null");
		Assert.notNull(clazz, "Class must not be null");
		return (clazz.isAnnotationPresent(annotationType) && !isAnnotationDeclaredLocally(annotationType, clazz));
	}
```

- **getAnnotationAttributes**:获取给定注解的属性。

```java
/**
	 * Retrieve the given annotation's attributes as a {@link Map}, preserving all
	 * attribute types.
	 * <p>Equivalent to calling {@link #getAnnotationAttributes(Annotation, boolean, boolean)}
	 * with the {@code classValuesAsString} and {@code nestedAnnotationsAsMap} parameters
	 * set to {@code false}.
	 * <p>Note: This method actually returns an {@link AnnotationAttributes} instance.
	 * However, the {@code Map} signature has been preserved for binary compatibility.
	 * @param annotation the annotation to retrieve the attributes for
	 * @return the Map of annotation attributes, with attribute names as keys and
	 * corresponding attribute values as values (never {@code null})
	 * @see #getAnnotationAttributes(AnnotatedElement, Annotation)
	 * @see #getAnnotationAttributes(Annotation, boolean, boolean)
	 * @see #getAnnotationAttributes(AnnotatedElement, Annotation, boolean, boolean)
	 */
	public static Map<String, Object> getAnnotationAttributes(Annotation annotation) {
		return getAnnotationAttributes(null, annotation);
	}

	/**
	 * Retrieve the given annotation's attributes as a {@link Map}.
	 * <p>Equivalent to calling {@link #getAnnotationAttributes(Annotation, boolean, boolean)}
	 * with the {@code nestedAnnotationsAsMap} parameter set to {@code false}.
	 * <p>Note: This method actually returns an {@link AnnotationAttributes} instance.
	 * However, the {@code Map} signature has been preserved for binary compatibility.
	 * @param annotation the annotation to retrieve the attributes for
	 * @param classValuesAsString whether to convert Class references into Strings (for
	 * compatibility with {@link org.springframework.core.type.AnnotationMetadata})
	 * or to preserve them as Class references
	 * @return the Map of annotation attributes, with attribute names as keys and
	 * corresponding attribute values as values (never {@code null})
	 * @see #getAnnotationAttributes(Annotation, boolean, boolean)
	 */
	public static Map<String, Object> getAnnotationAttributes(Annotation annotation, boolean classValuesAsString) {
		return getAnnotationAttributes(annotation, classValuesAsString, false);
	}

	/**
	 * Retrieve the given annotation's attributes as an {@link AnnotationAttributes} map.
	 * <p>This method provides fully recursive annotation reading capabilities on par with
	 * the reflection-based {@link org.springframework.core.type.StandardAnnotationMetadata}.
	 * @param annotation the annotation to retrieve the attributes for
	 * @param classValuesAsString whether to convert Class references into Strings (for
	 * compatibility with {@link org.springframework.core.type.AnnotationMetadata})
	 * or to preserve them as Class references
	 * @param nestedAnnotationsAsMap whether to convert nested annotations into
	 * {@link AnnotationAttributes} maps (for compatibility with
	 * {@link org.springframework.core.type.AnnotationMetadata}) or to preserve them as
	 * {@code Annotation} instances
	 * @return the annotation attributes (a specialized Map) with attribute names as keys
	 * and corresponding attribute values as values (never {@code null})
	 * @since 3.1.1
	 */
	public static AnnotationAttributes getAnnotationAttributes(Annotation annotation, boolean classValuesAsString,
			boolean nestedAnnotationsAsMap) {

		return getAnnotationAttributes(null, annotation, classValuesAsString, nestedAnnotationsAsMap);
	}

	/**
	 * Retrieve the given annotation's attributes as an {@link AnnotationAttributes} map.
	 * <p>Equivalent to calling {@link #getAnnotationAttributes(AnnotatedElement, Annotation, boolean, boolean)}
	 * with the {@code classValuesAsString} and {@code nestedAnnotationsAsMap} parameters
	 * set to {@code false}.
	 * @param annotatedElement the element that is annotated with the supplied annotation;
	 * may be {@code null} if unknown
	 * @param annotation the annotation to retrieve the attributes for
	 * @return the annotation attributes (a specialized Map) with attribute names as keys
	 * and corresponding attribute values as values (never {@code null})
	 * @since 4.2
	 * @see #getAnnotationAttributes(AnnotatedElement, Annotation, boolean, boolean)
	 */
	public static AnnotationAttributes getAnnotationAttributes(@Nullable AnnotatedElement annotatedElement, Annotation annotation) {
		return getAnnotationAttributes(annotatedElement, annotation, false, false);
	}

	/**
	 * Retrieve the given annotation's attributes as an {@link AnnotationAttributes} map.
	 * <p>This method provides fully recursive annotation reading capabilities on par with
	 * the reflection-based {@link org.springframework.core.type.StandardAnnotationMetadata}.
	 * @param annotatedElement the element that is annotated with the supplied annotation;
	 * may be {@code null} if unknown
	 * @param annotation the annotation to retrieve the attributes for
	 * @param classValuesAsString whether to convert Class references into Strings (for
	 * compatibility with {@link org.springframework.core.type.AnnotationMetadata})
	 * or to preserve them as Class references
	 * @param nestedAnnotationsAsMap whether to convert nested annotations into
	 * {@link AnnotationAttributes} maps (for compatibility with
	 * {@link org.springframework.core.type.AnnotationMetadata}) or to preserve them as
	 * {@code Annotation} instances
	 * @return the annotation attributes (a specialized Map) with attribute names as keys
	 * and corresponding attribute values as values (never {@code null})
	 * @since 4.2
	 */
	public static AnnotationAttributes getAnnotationAttributes(@Nullable AnnotatedElement annotatedElement,
			Annotation annotation, boolean classValuesAsString, boolean nestedAnnotationsAsMap) {

		return getAnnotationAttributes(
				(Object) annotatedElement, annotation, classValuesAsString, nestedAnnotationsAsMap);
	}

	private static AnnotationAttributes getAnnotationAttributes(@Nullable Object annotatedElement,
			Annotation annotation, boolean classValuesAsString, boolean nestedAnnotationsAsMap) {

		AnnotationAttributes attributes =
				retrieveAnnotationAttributes(annotatedElement, annotation, classValuesAsString, nestedAnnotationsAsMap);
		postProcessAnnotationAttributes(annotatedElement, attributes, classValuesAsString, nestedAnnotationsAsMap);
		return attributes;
	}

	/**
	 * Retrieve the given annotation's attributes as an {@link AnnotationAttributes} map.
	 * <p>This method provides fully recursive annotation reading capabilities on par with
	 * the reflection-based {@link org.springframework.core.type.StandardAnnotationMetadata}.
	 * <p><strong>NOTE</strong>: This variant of {@code getAnnotationAttributes()} is
	 * only intended for use within the framework. The following special rules apply:
	 * <ol>
	 * <li>Default values will be replaced with default value placeholders.</li>
	 * <li>The resulting, merged annotation attributes should eventually be
	 * {@linkplain #postProcessAnnotationAttributes post-processed} in order to
	 * ensure that placeholders have been replaced by actual default values and
	 * in order to enforce {@code @AliasFor} semantics.</li>
	 * </ol>
	 * @param annotatedElement the element that is annotated with the supplied annotation;
	 * may be {@code null} if unknown
	 * @param annotation the annotation to retrieve the attributes for
	 * @param classValuesAsString whether to convert Class references into Strings (for
	 * compatibility with {@link org.springframework.core.type.AnnotationMetadata})
	 * or to preserve them as Class references
	 * @param nestedAnnotationsAsMap whether to convert nested annotations into
	 * {@link AnnotationAttributes} maps (for compatibility with
	 * {@link org.springframework.core.type.AnnotationMetadata}) or to preserve them as
	 * {@code Annotation} instances
	 * @return the annotation attributes (a specialized Map) with attribute names as keys
	 * and corresponding attribute values as values (never {@code null})
	 * @since 4.2
	 * @see #postProcessAnnotationAttributes
	 */
	static AnnotationAttributes retrieveAnnotationAttributes(@Nullable Object annotatedElement, Annotation annotation,
			boolean classValuesAsString, boolean nestedAnnotationsAsMap) {

		Class<? extends Annotation> annotationType = annotation.annotationType();
		AnnotationAttributes attributes = new AnnotationAttributes(annotationType);

		for (Method method : getAttributeMethods(annotationType)) {
			try {
				Object attributeValue = method.invoke(annotation);
				Object defaultValue = method.getDefaultValue();
				if (defaultValue != null && ObjectUtils.nullSafeEquals(attributeValue, defaultValue)) {
					attributeValue = new DefaultValueHolder(defaultValue);
				}
				attributes.put(method.getName(),
						adaptValue(annotatedElement, attributeValue, classValuesAsString, nestedAnnotationsAsMap));
			}
			catch (Throwable ex) {
				if (ex instanceof InvocationTargetException) {
					Throwable targetException = ((InvocationTargetException) ex).getTargetException();
					rethrowAnnotationConfigurationException(targetException);
				}
				throw new IllegalStateException("Could not obtain annotation attribute value for " + method, ex);
			}
		}

		return attributes;
	}

	/**
	 * Adapt the given value according to the given class and nested annotation settings.
	 * <p>Nested annotations will be
	 * {@linkplain #synthesizeAnnotation(Annotation, AnnotatedElement) synthesized}.
	 * @param annotatedElement the element that is annotated, used for contextual
	 * logging; may be {@code null} if unknown
	 * @param value the annotation attribute value
	 * @param classValuesAsString whether to convert Class references into Strings (for
	 * compatibility with {@link org.springframework.core.type.AnnotationMetadata})
	 * or to preserve them as Class references
	 * @param nestedAnnotationsAsMap whether to convert nested annotations into
	 * {@link AnnotationAttributes} maps (for compatibility with
	 * {@link org.springframework.core.type.AnnotationMetadata}) or to preserve them as
	 * {@code Annotation} instances
	 * @return the adapted value, or the original value if no adaptation is needed
	 */
	@Nullable
	static Object adaptValue(@Nullable Object annotatedElement, @Nullable Object value,
			boolean classValuesAsString, boolean nestedAnnotationsAsMap) {

		if (classValuesAsString) {
			if (value instanceof Class) {
				return ((Class<?>) value).getName();
			}
			else if (value instanceof Class[]) {
				Class<?>[] clazzArray = (Class<?>[]) value;
				String[] classNames = new String[clazzArray.length];
				for (int i = 0; i < clazzArray.length; i++) {
					classNames[i] = clazzArray[i].getName();
				}
				return classNames;
			}
		}

		if (value instanceof Annotation) {
			Annotation annotation = (Annotation) value;
			if (nestedAnnotationsAsMap) {
				return getAnnotationAttributes(annotatedElement, annotation, classValuesAsString, true);
			}
			else {
				return synthesizeAnnotation(annotation, annotatedElement);
			}
		}

		if (value instanceof Annotation[]) {
			Annotation[] annotations = (Annotation[]) value;
			if (nestedAnnotationsAsMap) {
				AnnotationAttributes[] mappedAnnotations = new AnnotationAttributes[annotations.length];
				for (int i = 0; i < annotations.length; i++) {
					mappedAnnotations[i] =
							getAnnotationAttributes(annotatedElement, annotations[i], classValuesAsString, true);
				}
				return mappedAnnotations;
			}
			else {
				return synthesizeAnnotationArray(annotations, annotatedElement);
			}
		}

		// Fallback
		return value;
	}
```

- **getValue**:获取给定注解的值。 有这个名称的存在两种方法。第一个返回注解的全局值。第二个是指定注解参数的值。

```java
/**
 * Retrieve the <em>value</em> of the {@code value} attribute of a
 * single-element Annotation, given an annotation instance.
 * @param annotation the annotation instance from which to retrieve the value
 * @return the attribute value, or {@code null} if not found unless the attribute
 * value cannot be retrieved due to an {@link AnnotationConfigurationException},
 * in which case such an exception will be rethrown
 * @see #getValue(Annotation, String)
 */
@Nullable
public static Object getValue(Annotation annotation) {
	return getValue(annotation, VALUE);
}

/**
 * Retrieve the <em>value</em> of a named attribute, given an annotation instance.
 * @param annotation the annotation instance from which to retrieve the value
 * @param attributeName the name of the attribute value to retrieve
 * @return the attribute value, or {@code null} if not found unless the attribute
 * value cannot be retrieved due to an {@link AnnotationConfigurationException},
 * in which case such an exception will be rethrown
 * @see #getValue(Annotation)
 * @see #rethrowAnnotationConfigurationException(Throwable)
 */
@Nullable
public static Object getValue(@Nullable Annotation annotation, @Nullable String attributeName) {
	if (annotation == null || !StringUtils.hasText(attributeName)) {
		return null;
	}
	try {
		Method method = annotation.annotationType().getDeclaredMethod(attributeName);
		ReflectionUtils.makeAccessible(method);
		return method.invoke(annotation);
	}
	catch (InvocationTargetException ex) {
		rethrowAnnotationConfigurationException(ex.getTargetException());
		throw new IllegalStateException(
				"Could not obtain value for annotation attribute '" + attributeName + "' in " + annotation, ex);
	}
	catch (Throwable ex) {
		handleIntrospectionFailure(annotation.getClass(), ex);
		return null;
	}
}
```

- **getDefaultValue**:获取给定注解或注解属性的默认值(注意`@Nullable`注解就知道为什么这么说了)。

```java
/**
 * Retrieve the <em>default value</em> of the {@code value} attribute
 * of a single-element Annotation, given an annotation instance.
 * @param annotation the annotation instance from which to retrieve the default value
 * @return the default value, or {@code null} if not found
 * @see #getDefaultValue(Annotation, String)
 */
@Nullable
public static Object getDefaultValue(Annotation annotation) {
	return getDefaultValue(annotation, VALUE);
}

/**
 * Retrieve the <em>default value</em> of a named attribute, given an annotation instance.
 * @param annotation the annotation instance from which to retrieve the default value
 * @param attributeName the name of the attribute value to retrieve
 * @return the default value of the named attribute, or {@code null} if not found
 * @see #getDefaultValue(Class, String)
 */
@Nullable
public static Object getDefaultValue(@Nullable Annotation annotation, @Nullable String attributeName) {
	if (annotation == null) {
		return null;
	}
	return getDefaultValue(annotation.annotationType(), attributeName);
}

/**
 * Retrieve the <em>default value</em> of the {@code value} attribute
 * of a single-element Annotation, given the {@link Class annotation type}.
 * @param annotationType the <em>annotation type</em> for which the default value should be retrieved
 * @return the default value, or {@code null} if not found
 * @see #getDefaultValue(Class, String)
 */
@Nullable
public static Object getDefaultValue(Class<? extends Annotation> annotationType) {
	return getDefaultValue(annotationType, VALUE);
}

/**
 * Retrieve the <em>default value</em> of a named attribute, given the
 * {@link Class annotation type}.
 * @param annotationType the <em>annotation type</em> for which the default value should be retrieved
 * @param attributeName the name of the attribute value to retrieve.
 * @return the default value of the named attribute, or {@code null} if not found
 * @see #getDefaultValue(Annotation, String)
 */
@Nullable
public static Object getDefaultValue(
		@Nullable Class<? extends Annotation> annotationType, @Nullable String attributeName) {

	if (annotationType == null || !StringUtils.hasText(attributeName)) {
		return null;
	}
	try {
		return annotationType.getDeclaredMethod(attributeName).getDefaultValue();
	}
	catch (Throwable ex) {
		handleIntrospectionFailure(annotationType, ex);
		return null;
	}
}
```

## 有哪些地方使用了AnnotationUtils方法？

很多Spring项目模块都用了`AnnotationUtils`。这里我们将重点关注与`core`和Web开发相关的项目模块:`Web`，`Web MVC`，`context`和`bean`。这里就不罗嗦太多了，只列出在这些Spring项目中使用的`AnnotationUtils`的地方:

1. web MVC

   - `AnnotationMethodHandlerAdapter`，直到 Spring 3.1的版本都是作为注解方法的主要处理程序，使用`AnnotationUtils`来检查可用于方法级别的不同注解，如:`@RequestMapping`，`@ResponseStatus`，`@ResponseBody`或`@ModelAttribute`。
   - 作为`AnnotationMethodHandlerAdapter`接班人，`RequestMappingHandlerMapping`与`AnnotationUtils`一起解析`@RequestMapping`并构造了封装映射配置(变量，HTTP方法， accepted headers 等)的`RequestMappingInfo`对象。

   ```java
   /**
    * Uses method and type-level @{@link RequestMapping} annotations to create
    * the RequestMappingInfo.
    * @return the created RequestMappingInfo, or {@code null} if the method
    * does not have a {@code @RequestMapping} annotation.
    * @see #getCustomMethodCondition(Method)
    * @see #getCustomTypeCondition(Class)
    */
   @Override
   protected RequestMappingInfo getMappingForMethod(Method method, Class<?> handlerType) {
   	RequestMappingInfo info = createRequestMappingInfo(method);
   	if (info != null) {
   		RequestMappingInfo typeInfo = createRequestMappingInfo(handlerType);
   		if (typeInfo != null) {
   			info = typeInfo.combine(info);
   		}
   	}
   	return info;
   }
   /**
    * Delegates to {@link #createRequestMappingInfo(RequestMapping, RequestCondition)},
    * supplying the appropriate custom {@link RequestCondition} depending on whether
    * the supplied {@code annotatedElement} is a class or method.
    * @see #getCustomTypeCondition(Class)
    * @see #getCustomMethodCondition(Method)
    */
   private RequestMappingInfo createRequestMappingInfo(AnnotatedElement element) {
   	RequestMapping requestMapping = AnnotatedElementUtils.findMergedAnnotation(element, RequestMapping.class);
   	RequestCondition<?> condition = (element instanceof Class ?
   			getCustomTypeCondition((Class<?>) element) : getCustomMethodCondition((Method) element));
   	return (requestMapping != null ? createRequestMappingInfo(requestMapping, condition) : null);
   }
   ```

   - `RequestMappingHandlerAdapter`是`Web MVC`项目中使用`AnnotationUtils`的第三个重要类。我们可以找到2个调用了`findAnnotation()`方法并都返回`MethodFilter`类的实例的方法。一个表示`@InitBinder`注解，第一个表示`@ModelAttribute`。

   ```java
   	/**
   	 * MethodFilter that matches {@link InitBinder @InitBinder} methods.
   	 */
   	public static final MethodFilter INIT_BINDER_METHODS = method ->
   			AnnotationUtils.findAnnotation(method, InitBinder.class) != null;

   	/**
   	 * MethodFilter that matches {@link ModelAttribute @ModelAttribute} methods.
   	 */
   	public static final MethodFilter MODEL_ATTRIBUTE_METHODS = method ->
   			((AnnotationUtils.findAnnotation(method, RequestMapping.class) == null) &&
   			(AnnotationUtils.findAnnotation(method, ModelAttribute.class) != null));

   	//以上两个方法的用法
   private void initControllerAdviceCache() {
   		if (getApplicationContext() == null) {
   			return;
   		}
   		if (logger.isInfoEnabled()) {
   			logger.info("Looking for @ControllerAdvice: " + getApplicationContext());
   		}

   		List<ControllerAdviceBean> beans = ControllerAdviceBean.findAnnotatedBeans(getApplicationContext());
   		AnnotationAwareOrderComparator.sort(beans);

   		List<Object> requestResponseBodyAdviceBeans = new ArrayList<Object>();

   		for (ControllerAdviceBean bean : beans) {
             //传入MODEL_ATTRIBUTE_METHODS这个表达式
   			Set<Method> attrMethods = MethodIntrospector.selectMethods(bean.getBeanType(), MODEL_ATTRIBUTE_METHODS);
   			if (!attrMethods.isEmpty()) {
   				this.modelAttributeAdviceCache.put(bean, attrMethods);
   				if (logger.isInfoEnabled()) {
   					logger.info("Detected @ModelAttribute methods in " + bean);
   				}
   			}
             //传入 INIT_BINDER_METHODS 这个表达式
   			Set<Method> binderMethods = MethodIntrospector.selectMethods(bean.getBeanType(), INIT_BINDER_METHODS);
   			if (!binderMethods.isEmpty()) {
   				this.initBinderAdviceCache.put(bean, binderMethods);
   				if (logger.isInfoEnabled()) {
   					logger.info("Detected @InitBinder methods in " + bean);
   				}
   			}
   			if (RequestBodyAdvice.class.isAssignableFrom(bean.getBeanType())) {
   				requestResponseBodyAdviceBeans.add(bean);
   				if (logger.isInfoEnabled()) {
   					logger.info("Detected RequestBodyAdvice bean in " + bean);
   				}
   			}
   			if (ResponseBodyAdvice.class.isAssignableFrom(bean.getBeanType())) {
   				requestResponseBodyAdviceBeans.add(bean);
   				if (logger.isInfoEnabled()) {
   					logger.info("Detected ResponseBodyAdvice bean in " + bean);
   				}
   			}
   		}

   		if (!requestResponseBodyAdviceBeans.isEmpty()) {
   			this.requestResponseBodyAdvice.addAll(0, requestResponseBodyAdviceBeans);
   		}
   	}
   ```

接上面最后一个方法`selectMethods`的实现，然后，我们平时写代码的时候也可以参考此实现形式:

**org.springframework.core.MethodIntrospector**

```java
	/**
	 * Select methods on the given target type based on a filter.
	 * <p>Callers define methods of interest through the {@code MethodFilter} parameter.
	 * @param targetType the target type to search methods on
	 * @param methodFilter a {@code MethodFilter} to help
	 * recognize handler methods of interest
	 * @return the selected methods, or an empty set in case of no match
	 */
	public static Set<Method> selectMethods(Class<?> targetType, final ReflectionUtils.MethodFilter methodFilter) {
		return selectMethods(targetType, new MetadataLookup<Boolean>() {
			@Override
			public Boolean inspect(Method method) {
				return (methodFilter.matches(method) ? Boolean.TRUE : null);
			}
		}).keySet();
	}
//此段代码可迭代为下面形式,这样更加符合之前定义methodFilter的习惯
	public static Set<Method> selectMethods(Class<?> targetType, final ReflectionUtils.MethodFilter methodFilter) {
		return selectMethods(targetType, (MetadataLookup)(method)-> {
				return (methodFilter.matches(method) ? Boolean.TRUE : null);
		}).keySet();
	}
```

1. web(此处分析俩过时的类，Spring5里没有，但是4里面包含有，之前的文章有写)

   - 如果对象必须使用`@Valid`进行验证，或者`@InitBinder`方法存在，`HandlerMethodInvoker`使用`AnnotationUtils`来知道`@ModelAttribute`注解是什么。
   - 这个项目的另一个关键类，`HandlerMethodResolver`，通过调用`AnnotationUtils`方法来确定方法的类型(handler，binder或model-attribute)。具体点讲就是通过3种方法完成:`isHandlerMethod`，`isInitBinderMethod`和`isModelAttributeMethod`。每个都接受Method的实例为参数。

2. context

   - 用于解析**bean**注解的类`BeanAnnotationHelper`使用`AnnotationUtils`中的`findMergedAnnotation()`方法来处理用`@Bean`注解的类。我们使用它来确定bean的名称。

   ```java
   /**
    * Utilities for processing {@link Bean}-annotated methods.
    *
    * @author Chris Beams
    * @author Juergen Hoeller
    * @since 3.1
    */
   class BeanAnnotationHelper {

   	public static boolean isBeanAnnotated(Method method) {
   		return AnnotatedElementUtils.hasAnnotation(method, Bean.class);
   	}

   	public static String determineBeanNameFor(Method beanMethod) {
   		// By default, the bean name is the name of the @Bean-annotated method
   		String beanName = beanMethod.getName();

   		// Check to see if the user has explicitly set a custom bean name...
   		Bean bean = AnnotatedElementUtils.findMergedAnnotation(beanMethod, Bean.class);
   		if (bean != null && bean.name().length > 0) {
   			beanName = bean.name()[0];
   		}

   		return beanName;
   	}

   }
   ```

   ​

   - 还有一个要说的就是`AnnotationAsyncExecutionInterceptor`类，其内同样使用`AnnotationUtils`来解析注解。它调用`findMergedAnnotation()`方法来解析在运行时所执行的方法的名称。

   ```java
   /**
    * Specialization of {@link AsyncExecutionInterceptor} that delegates method execution to
    * an {@code Executor} based on the {@link Async} annotation. Specifically designed to
    * support use of {@link Async#value()} executor qualification mechanism introduced in
    * Spring 3.1.2. Supports detecting qualifier metadata via {@code @Async} at the method or
    * declaring class level. See {@link #getExecutorQualifier(Method)} for details.
    *
    * @author Chris Beams
    * @author Stephane Nicoll
    * @since 3.1.2
    * @see org.springframework.scheduling.annotation.Async
    * @see org.springframework.scheduling.annotation.AsyncAnnotationAdvisor
    */
   public class AnnotationAsyncExecutionInterceptor extends AsyncExecutionInterceptor {

   	/**
   	 * Create a new {@code AnnotationAsyncExecutionInterceptor} with the given executor
   	 * and a simple {@link AsyncUncaughtExceptionHandler}.
   	 * @param defaultExecutor the executor to be used by default if no more specific
   	 * executor has been qualified at the method level using {@link Async#value()};
   	 * as of 4.2.6, a local executor for this interceptor will be built otherwise
   	 */
   	public AnnotationAsyncExecutionInterceptor(@Nullable Executor defaultExecutor) {
   		super(defaultExecutor);
   	}

   	/**
   	 * Create a new {@code AnnotationAsyncExecutionInterceptor} with the given executor.
   	 * @param defaultExecutor the executor to be used by default if no more specific
   	 * executor has been qualified at the method level using {@link Async#value()};
   	 * as of 4.2.6, a local executor for this interceptor will be built otherwise
   	 * @param exceptionHandler the {@link AsyncUncaughtExceptionHandler} to use to
   	 * handle exceptions thrown by asynchronous method executions with {@code void}
   	 * return type
   	 */
   	public AnnotationAsyncExecutionInterceptor(@Nullable Executor defaultExecutor, AsyncUncaughtExceptionHandler exceptionHandler) {
   		super(defaultExecutor, exceptionHandler);
   	}
     		/**
   	 	 *  Return the qualifier or bean name of the executor to be used when executing the
        	 * given method, specified via {@link Async#value} at the method or declaring
            * class level. If {@code @Async} is specified at both the method and class level, the
            * method's {@code #value} takes precedence (even if empty string, indicating that
            * the default executor should be used preferentially).
            * @param method the method to inspect for executor qualifier metadata
            * @return the qualifier if specified, otherwise empty string indicating that the
            * {@linkplain #setExecutor(Executor) default executor} should be used
            * @see #determineAsyncExecutor(Method)
            */
               @Override
               protected String getExecutorQualifier(Method method) {
               // Maintainer's note: changes made here should also be made in
               // AnnotationAsyncExecutionAspect#getExecutorQualifier
               Async async = AnnotatedElementUtils.findMergedAnnotation(method, Async.class);
               if (async == null) {
               	async = AnnotatedElementUtils.findMergedAnnotation(method.getDeclaringClass(), Async.class);
               }
               return (async != null ? async.value() : null);
               }
   }
   ```

3. bean

   - 我们可以在`StaticListableBeanFactory`或`DefaultListableBeanFactory`类中找到`AnnotationUtils`用来查找`bean`的注解的用法。

   **org.springframework.beans.factory.support.StaticListableBeanFactory**

   ```java
   @Override
   public Map<String, Object> getBeansWithAnnotation(Class<? extends Annotation> annotationType)
   		throws BeansException {

   	Map<String, Object> results = new LinkedHashMap<>();
   	for (String beanName : this.beans.keySet()) {
   		if (findAnnotationOnBean(beanName, annotationType) != null) {
   			results.put(beanName, getBean(beanName));
   		}
   	}
   	return results;
   }

   @Override
   public <A extends Annotation> A findAnnotationOnBean(String beanName, Class<A> annotationType)
   		throws NoSuchBeanDefinitionException{

   	Class<?> beanType = getType(beanName);
   	return (beanType != null ? AnnotationUtils.findAnnotation(beanType, annotationType) : null);
   }
   ```

   **org.springframework.beans.factory.support.DefaultListableBeanFactory**

   ```java
   @Override
   public String[] getBeanNamesForAnnotation(Class<? extends Annotation> annotationType) {
   	List<String> results = new ArrayList<>();
   	for (String beanName : this.beanDefinitionNames) {
   		BeanDefinition beanDefinition = getBeanDefinition(beanName);
   		if (!beanDefinition.isAbstract() && findAnnotationOnBean(beanName, annotationType) != null) {
   			results.add(beanName);
   		}
   	}
   	for (String beanName : this.manualSingletonNames) {
   		if (!results.contains(beanName) && findAnnotationOnBean(beanName, annotationType) != null) {
   			results.add(beanName);
   		}
   	}
   	return results.toArray(new String[results.size()]);
   }

   @Override
   public Map<String, Object> getBeansWithAnnotation(Class<? extends Annotation> annotationType) {
   	String[] beanNames = getBeanNamesForAnnotation(annotationType);
   	Map<String, Object> results = new LinkedHashMap<>(beanNames.length);
   	for (String beanName : beanNames) {
   		results.put(beanName, getBean(beanName));
   	}
   	return results;
   }
   /**
    * Find a {@link Annotation} of {@code annotationType} on the specified
    * bean, traversing its interfaces and super classes if no annotation can be
    * found on the given class itself, as well as checking its raw bean class
    * if not found on the exposed bean reference (e.g. in case of a proxy).
    */
   @Override
   public <A extends Annotation> A findAnnotationOnBean(String beanName, Class<A> annotationType)
   		throws NoSuchBeanDefinitionException{

   	A ann = null;
   	Class<?> beanType = getType(beanName);
   	if (beanType != null) {
   		ann = AnnotationUtils.findAnnotation(beanType, annotationType);
   	}
   	if (ann == null && containsBeanDefinition(beanName)) {
   		BeanDefinition bd = getMergedBeanDefinition(beanName);
   		if (bd instanceof AbstractBeanDefinition) {
   			AbstractBeanDefinition abd = (AbstractBeanDefinition) bd;
   			if (abd.hasBeanClass()) {
   				ann = AnnotationUtils.findAnnotation(abd.getBeanClass(), annotationType);
   			}
   		}
   	}
   	return ann;
   }
   ```

   ​

## AnnotationUtils in Action

为了更好地理解Spring中AnnotationUtils的工作方式，我们来搞两个注解:第一个对应方法，第二个对应类。之后，写两个测试类和一个`playground`类，我们要达到的目的是输出由`AnnotationUtils`完成的注解分析结果。这两个注解定义如下:

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
public @interface ClassNameAnnotation {

  String className() default "Empty class name";
}
```

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
public @interface StaticTextAnnotation {

  String text() default "Default text for static text annotation";
  String value() default "Default value";
}
```

**@Retention注解是必需的(估计这里是大家的知识盲区，所以特地点出来)。否则AnnotationUtils将无法检测到这些注解。**注意在`StaticTextAnnotation`中存在`value()`属性，并且在`ClassNameAnnotation`中不存在此属性。在以下代码中，你可以找到相应的测试类:

```java
@ClassNameAnnotation(className = "TestChildren")
public class TestParent {

  @StaticTextAnnotation(value= "Custom text value", text = "Test text")
  public String test(HttpServletRequest request) {
    return "test";
  }
}
```

```java
public class TestChildren extends TestParent {
}
```

`TestChildren`类没有任何注解。我们使用它来测试继承注解检查。

```java
public class Playground {
  public static void main(String[] args) {
    try {
      Method method = TestParent.class.getMethod("test", new Class[]{HttpServletRequest.class});
      Annotation staticTextAnnot = AnnotationUtils.findAnnotation(method, StaticTextAnnotation.class);
      System.out.println("@StaticTextAnnotation of method is: "+staticTextAnnot);
      System.out.println("@StaticTextAnnotation method value: "+AnnotationUtils.getValue(staticTextAnnot, "text"));
      System.out.println("@StaticTextAnnotation method default value: "+AnnotationUtils.getDefaultValue(staticTextAnnot, "text"));
      System.out.println("@StaticTextAnnotation value: "+AnnotationUtils.getValue(staticTextAnnot));

      // inheriting annotations tests
      Annotation classNameAnnotation = AnnotationUtils.findAnnotation(TestChildren.class, ClassNameAnnotation.class);
      System.out.println("@ClassNameAnnotation of TestChildren.class is: "+classNameAnnotation);
      System.out.println("@ClassNameAnnotation method value: "+AnnotationUtils.getValue(classNameAnnotation, "className"));
      System.out.println("@ClassNameAnnotation method default value: "+AnnotationUtils.getDefaultValue(classNameAnnotation, "className"));
      System.out.println("@ClassNameAnnotation value: "+AnnotationUtils.getValue(classNameAnnotation));
    } catch (Exception e) {
      e.printStackTrace();
    }
  }
}
```

`Playground`的`main`方法结果如下:

```java
@StaticTextAnnotation of method is: @com.migo.annotations.StaticTextAnnotation(text=Test text, value=Custom text value)
@StaticTextAnnotation method value: Test text
@StaticTextAnnotation method default value: Default text for static text annotation
@StaticTextAnnotation value: Custom text value
@ClassNameAnnotation of TestChildren.class is: @com.migo.annotations.ClassNameAnnotation(className=TestChildren)
@ClassNameAnnotation method value: TestChildren
@ClassNameAnnotation method default value: Empty class name
@ClassNameAnnotation value: null
```

如上所示，我们可以很轻易的了解很多注解点。我们可以检查`value()`属性或另一个自定义属性的值。我们还可以检查属性的默认值。除此之外，`AnnotationUtils`还可以在继承体系结构中进行注解操作。

# 666. 彩蛋

如果你对 Spring 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)