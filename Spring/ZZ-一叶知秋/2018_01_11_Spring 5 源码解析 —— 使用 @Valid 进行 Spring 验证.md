title: Spring 5 源码解析 —— 使用 @Valid 进行 Spring 验证
date: 2018-01-11
tag: 
categories: Spring
permalink: Spring/@Valid
author: 一叶知秋
from_url: https://muyinchen.github.io/2017/08/11/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— %E4%BD%BF%E7%94%A8@Valid%E8%BF%9B%E8%A1%8CSpring%E9%AA%8C%E8%AF%81/
wechat_url: 

-------

摘要: 原创出处 https://muyinchen.github.io/2017/08/11/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— %E4%BD%BF%E7%94%A8@Valid%E8%BF%9B%E8%A1%8CSpring%E9%AA%8C%E8%AF%81/ 「一叶知秋」欢迎转载，保留摘要，谢谢！

- [使用@Valid注解在Spring中进行验证流程](http://www.iocoder.cn/Spring/@Valid/)
- [如何在Spring中处理验证？](http://www.iocoder.cn/Spring/@Valid/)
- [controller方法内获取BindingResult](http://www.iocoder.cn/Spring/@Valid/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

> 验证功能在Spring中是很常用的。你可以使用注解或自己的验证器并将其绑定到请求中。本文将重点介绍第一种解决方案。

第一部分将介绍注解验证流程。在第二部分中，将介绍基本实现的组件。最后一部分将包含Spring初学者开发人员常见错误的解释:是否有必要直接在验证对象之后放置`BindingResult`。

## 使用@Valid注解在Spring中进行验证流程

要了解使用`标准Java @Valid`或`特定Spring @Validated`注解的验证过程，我们首先需要了解Spring如何解析使用了`@ModelAttribute`注解的对象。它们在controller的方法签名进行注解。@ModelAttribute注解用于将动态请求参数`转换`为Java注解中指定的对象。例如，观察代码**@ModelAttribute(“article”)Article article** ,Spring会尝试将所有请求参数匹配到Article类的字段中。现在，假设这个类有两个字段:`title`和`content`。如果请求包含`title`和`content`参数，它们将被用作`Article`的`title`和`content`的值(后面会对`@ModelAttribute`方面的源码做进一步的分析)。

当我们有对象需要进行验证时，`@ModelAttribute`注解的处理器(**org.springframework.web.method.annotation.ModelAttributeMethodProcessor**)会检查是否必须应用验证注解。注解验证必须以“Valid”这个字眼开头。接下来，对象通过**org.springframework.validation.DataBinder**类中的**public void validate(Object … validationHints)**进行**验证**。该方法遍历所有可用的验证器，并调用每个验证器的`validate`方法。验证器取自带有`validator`ID的bean。这样，它可以与`annotation-driven`的xml配置相关联:

```xml
<mvc:annotation-driven validator="validator" >
```

如果未指定验证器bean，则将使用默认验证器:**org.springframework.validation.beanvalidation.LocalValidatorFactoryBean**。

## 如何在Spring中处理验证？

我们已经了解了验证流程。现在，我们可以专注于验证过程本身，即验证器是如何知道一个字段不正确的。`LocalValidatorFactoryBean`继承自同一个包下的`SpringValidatorAdapter`，但不会覆盖其的validate()方法。这些方法用于检查验证字段是否正确。更准确地说，`SpringValidatorAdapter`包含一个目标验证器字段(`Validator`类型的`targetValidator`)。它将在`validate()`方法中使用来验证已验证对象的所有字段。

```Java
public class SpringValidatorAdapter implements SmartValidator, javax.validation.Validator {

	private static final Set<String> internalAnnotationAttributes = new HashSet<>(3);

	static {
		internalAnnotationAttributes.add("message");
		internalAnnotationAttributes.add("groups");
		internalAnnotationAttributes.add("payload");
	}

	@Nullable
	private javax.validation.Validator targetValidator;


	/**
	 * Create a new SpringValidatorAdapter for the given JSR-303 Validator.
	 * @param targetValidator the JSR-303 Validator to wrap
	 */
	public SpringValidatorAdapter(javax.validation.Validator targetValidator) {
		Assert.notNull(targetValidator, "Target Validator must not be null");
		this.targetValidator = targetValidator;
	}

	SpringValidatorAdapter() {
	}

	void setTargetValidator(javax.validation.Validator targetValidator) {
		this.targetValidator = targetValidator;
	}
...
	@Override
	public void validate(@Nullable Object target, Errors errors) {
		if (this.targetValidator != null) {
			processConstraintViolations(this.targetValidator.validate(target), errors);
		}
	}

	@Override
	public void validate(@Nullable Object target, Errors errors, @Nullable Object... validationHints) {
		if (this.targetValidator != null) {
			Set<Class<?>> groups = new LinkedHashSet<>();
			if (validationHints != null) {
				for (Object hint : validationHints) {
					if (hint instanceof Class) {
						groups.add((Class<?>) hint);
					}
				}
			}
			processConstraintViolations(
					this.targetValidator.validate(target, groups.toArray(new Class<?>[groups.size()])), errors);
		}
	}
```

此验证的结果是由在`SpringValidatorAdapter`内的`protected void processConstraintViolations(Set<ConstraintViolation<Object>> violations, Errors errors)`方法处理得到。它将错误从JSR-303验证器附加到给定的Spring的错误对象(觉得别扭请看下面方法上的英文注释)。

```Java
/**
	 * Process the given JSR-303 ConstraintViolations, adding corresponding errors to
	 * the provided Spring {@link Errors} object.
	 * @param violations the JSR-303 ConstraintViolation results
	 * @param errors the Spring errors object to register to
	 */
	protected void processConstraintViolations(Set<ConstraintViolation<Object>> violations, Errors errors) {
		for (ConstraintViolation<Object> violation : violations) {
			String field = determineField(violation);
			FieldError fieldError = errors.getFieldError(field);
			if (fieldError == null || !fieldError.isBindingFailure()) {
				try {
					ConstraintDescriptor<?> cd = violation.getConstraintDescriptor();
					String errorCode = determineErrorCode(cd);
					Object[] errorArgs = getArgumentsForConstraint(errors.getObjectName(), field, cd);
					if (errors instanceof BindingResult) {
						// Can do custom FieldError registration with invalid value from ConstraintViolation,
						// as necessary for Hibernate Validator compatibility (non-indexed set path in field)
						BindingResult bindingResult = (BindingResult) errors;
						String nestedField = bindingResult.getNestedPath() + field;
						if ("".equals(nestedField)) {
							String[] errorCodes = bindingResult.resolveMessageCodes(errorCode);
							bindingResult.addError(new ObjectError(
									errors.getObjectName(), errorCodes, errorArgs, violation.getMessage()));
						}
						else {
							Object rejectedValue = getRejectedValue(field, violation, bindingResult);
							String[] errorCodes = bindingResult.resolveMessageCodes(errorCode, field);
							bindingResult.addError(new FieldError(
									errors.getObjectName(), nestedField, rejectedValue, false,
									errorCodes, errorArgs, violation.getMessage()));
						}
					}
					else {
						// got no BindingResult - can only do standard rejectValue call
						// with automatic extraction of the current field value
						errors.rejectValue(field, errorCode, errorArgs, violation.getMessage());
					}
				}
				catch (NotReadablePropertyException ex) {
					throw new IllegalStateException("JSR-303 validated property '" + field +
							"' does not have a corresponding accessor for Spring data binding - " +
							"check your DataBinder's configuration (bean property versus direct field access)", ex);
				}
			}
		}
	}
```

验证错误直接附加到`DataBinder`的**private AbstractPropertyBindingResult bindingResult**字段。

```
public class DataBinder implements PropertyEditorRegistry, TypeConverter {

	/** Default object name used for binding: "target" */
	public static final String DEFAULT_OBJECT_NAME = "target";

	/** Default limit for array and collection growing: 256 */
	public static final int DEFAULT_AUTO_GROW_COLLECTION_LIMIT = 256;


	/**
	 * We'll create a lot of DataBinder instances: Let's use a static logger.
	 */
	protected static final Log logger = LogFactory.getLog(DataBinder.class);

	@Nullable
	private final Object target;

	private final String objectName;

	@Nullable
	private AbstractPropertyBindingResult bindingResult;

	@Nullable
	private SimpleTypeConverter typeConverter;
```

此时它的值会在`ModelAttributeMethodProcessor`中检索:

```Java
if (binder.getBindingResult().hasErrors()) {
    if (isBindExceptionRequired(binder, parameter)) {
        throw new BindException(binder.getBindingResult());
    }
}
```

## controller方法内获取BindingResult

需要注意的是，要在控制器的方法中检索`BindingResult`，必须将`BindingResult`实例直接放在经过验证的对象之后。具体请看**public String addArticle(@ModelAttribute(“article”) @Valid Article article, BindingResult result)**，`BindingResult`的实例将包含所有的验证错误。这时，如果你在`Article`和`BindingResult`实例之间放置另一个对象(例如:`HttpServletRequest request`)，将抛出如下异常:

```shell
An Errors/BindingResult argument is expected to be declared immediately after the  model attribute, the @RequestBody or the @RequestPart arguments to which they apply.
```

此错误消息的内容可以在**org.springframework.web.method.annotation.ErrorsMethodArgumentResolver**类中找到。此类用于从方法签名中解析错误实例。如果问为什么用`ErrorsMethodArgumentResolver`来解析`BindingResults`？简单来说，这是由于`BindingResult`接口扩展了`Errors`接口的缘故。所以，两者都可以用相同的参数解析器解决。

```Java
/**
 * Resolves {@link Errors} method arguments.
 *
 * <p>An {@code Errors} method argument is expected to appear immediately after
 * the model attribute in the method signature. It is resolved by expecting the
 * last two attributes added to the model to be the model attribute and its
 * {@link BindingResult}.
 *
 * @author Rossen Stoyanchev
 * @since 3.1
 */
public class ErrorsMethodArgumentResolver implements HandlerMethodArgumentResolver {

	@Override
	public boolean supportsParameter(MethodParameter parameter) {
		Class<?> paramType = parameter.getParameterType();
		return Errors.class.isAssignableFrom(paramType);
	}

	@Override
	public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

		Assert.state(mavContainer != null, "Errors/BindingResult argument only supported on regular handler methods");

		ModelMap model = mavContainer.getModel();
		if (model.size() > 0) {
			int lastIndex = model.size()-1;
			String lastKey = new ArrayList<>(model.keySet()).get(lastIndex);
			if (lastKey.startsWith(BindingResult.MODEL_KEY_PREFIX)) {
				return model.get(lastKey);
			}
		}

		throw new IllegalStateException(
				"An Errors/BindingResult argument is expected to be declared immediately after the model attribute, " +
				"the @RequestBody or the @RequestPart arguments to which they apply: " + parameter.getMethod());
	}

}
```

从上面代码可以看出，由于BindingResult的放置的位置 不正确，而导致验证过程失败的方法其实很简单:

```Java
ModelMap model = mavContainer.getModel();
if (model.size() > 0) {
    int lastIndex = model.size()-1;
    String lastKey = new ArrayList<String>(model.keySet()).get(lastIndex);
    if (lastKey.startsWith(BindingResult.MODEL_KEY_PREFIX)) {
        return model.get(lastKey);
    }
}
```

可以看到，它获得用于构建视图部分的模型数据的ModelMap。所要验证对象和`BindingResult`如果放置正确，那么所要打印的日志应该如下:

```shell
model equals to {article=Article {text = }, org.springframework.validation.BindingResult.article=org.springframework.validation.BeanPropertyBindingResult: 1 errors
Field error in object 'article' on field 'text': rejected value []; codes [NotEmpty.article.text,NotEmpty.text,NotEmpty.java.lang.String,NotEmpty]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [article.text,text]; arguments []; default message [text]]; default message [Text can't be empty]}
```

之后，将值放在`ArrayList`中，并获取最后一个 entry key。然后，检查此键是否以`org.springframework.validation.BindingResult`开头(BindingResult 接口的常量值)。如果是，该方法返回发现的Errors实例。否则，将抛出一个IllegalStateException异常。

```java
public interface BindingResult extends Errors {

	/**
	 * Prefix for the name of the BindingResult instance in a model,
	 * followed by the object name.
	 */
	String MODEL_KEY_PREFIX = BindingResult.class.getName() + ".";


	/**
	 * Return the wrapped target object, which may be a bean, an object with
	 * public fields, a Map - depending on the concrete binding strategy.
	 */
	@Nullable
	Object getTarget();
```

这篇文章讲了Spring 验证的一些过程细节。它的第一部分介绍了验证流程，从@ModelAttribute开始，并以验证器集合结束。第二部分看了看基本的Spring验证器。在最后，我们看到一个非常常见的bug，基于直接在验证对象之后放置BindingResult实例，并解释了其中的原理所在。

# 666. 彩蛋

如果你对 Spring 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)