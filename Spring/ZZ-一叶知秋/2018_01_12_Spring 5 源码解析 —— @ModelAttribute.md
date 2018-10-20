title: Spring 5 源码解析 —— @ModelAttribute
date: 2018-01-12
tag: 
categories: Spring
permalink: Spring/@ModelAttribute
author: 一叶知秋
from_url: https://muyinchen.github.io/2017/08/17/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— @ModelAttribute/
wechat_url: 

-------

摘要: 原创出处 https://muyinchen.github.io/2017/08/17/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— @ModelAttribute/ 「一叶知秋」欢迎转载，保留摘要，谢谢！

- [什么是@ModelAttribute注解？](http://www.iocoder.cn/Spring/@ModelAttribute/)
- [@ModelAttribute注解相关代码详解](http://www.iocoder.cn/Spring/@ModelAttribute/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

> 之前我们讨论了Spring中如何通过验证器来达到校验目的。其中有几行我们提到了@ModelAttribute注解。但是，单单理解这个概念还不够，总感觉飘如浮萍。

本文将对`@ModelAttribute`进行解析。将分为两部分。首先将介绍此注解的用法。第二部分将通过具体的代码来分析这个注解和其相应的解析器的细节。

## 什么是@ModelAttribute注解？

`@ModelAttribute`注解主要用来将请求转换为使用此注解指定的对象。例如，如果在`@ModelAttribute`旁边指定了一个`Article`实例，则与`Article`的字段对应的所有请求参数将被用作`Article`的字段值。什么意思呢，例如，`POST提交`后参数`title`的值将被设置为`Article`的`title` 字段。这里推荐一篇文章解释的很清晰:<http://blog.csdn.net/hejingyuan6/article/details/49995987>

因此，此注解允许开发人员通过请求来持久化一个对象。没有它，Spring认为必须创建一个新对象。另外，它直接显示一个对象模型来查看。你不需要在方法中再调用model.setAttribute()。在视图部分，可以通过注解中的指定值查找指定对象(例如，@ModelAttribute(“articleView”)可以在jsp中通过`${articleView}`获取相应的值)或对象的类名称(例如`@ModelAttribute()Article article`将在视图层获取方式就是`${article}`)。

## @ModelAttribute注解相关代码详解

还是分两波来说吧，也参考了不少其他解析的文章，看了很多相关评论，大都觉得各种迷迷糊糊所以就旧版新版都说说咯，反正都是源码学习，4.3版本之前和之后(4.2之后机制已经改了，下面讲新版本的时候会看到源码相关注释)，4.3版本之后被废弃掉了，先谈老版本的。

### 老版本

总体来看，有三个关键类协助`@ModelAttribute`来转换得到我们所需要的预期对象。第一个是**org.springframework.web.bind.annotation.support.HandlerMethodResolver**。它包含一个Set类型的私有字段，称为`modelAttributeMethods`。此字段包含被`@ModelAttribute`注解了的方法。在init()方法中，解析器将所有相关方法放在此集合中。

```Java
  private final Set<Method> modelAttributeMethods = new LinkedHashSet<Method>();
...
  /**
	 * Initialize a new HandlerMethodResolver for the specified handler type.
	 * @param handlerType the handler class to introspect
	 */
	public void init(final Class<?> handlerType) {
		Set<Class<?>> handlerTypes = new LinkedHashSet<Class<?>>();
		Class<?> specificHandlerType = null;
		if (!Proxy.isProxyClass(handlerType)) {
			handlerTypes.add(handlerType);
			specificHandlerType = handlerType;
		}
		handlerTypes.addAll(Arrays.asList(handlerType.getInterfaces()));
		for (Class<?> currentHandlerType : handlerTypes) {
			final Class<?> targetClass = (specificHandlerType != null ? specificHandlerType : currentHandlerType);
			ReflectionUtils.doWithMethods(currentHandlerType, new ReflectionUtils.MethodCallback() {
				@Override
				public void doWith(Method method) {
					Method specificMethod = ClassUtils.getMostSpecificMethod(method, targetClass);
					Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(specificMethod);
					if (isHandlerMethod(specificMethod) &&
							(bridgedMethod == specificMethod || !isHandlerMethod(bridgedMethod))) {
						handlerMethods.add(specificMethod);
					}
					else if (isInitBinderMethod(specificMethod) &&
							(bridgedMethod == specificMethod || !isInitBinderMethod(bridgedMethod))) {
						initBinderMethods.add(specificMethod);
					}
                  //此处代码可知
					else if (isModelAttributeMethod(specificMethod) &&
							(bridgedMethod == specificMethod || !isModelAttributeMethod(bridgedMethod))) {
						modelAttributeMethods.add(specificMethod);
					}
				}
			}, ReflectionUtils.USER_DECLARED_METHODS);
		}
		this.typeLevelMapping = AnnotationUtils.findAnnotation(handlerType, RequestMapping.class);
		SessionAttributes sessionAttributes = AnnotationUtils.findAnnotation(handlerType, SessionAttributes.class);
		this.sessionAttributesFound = (sessionAttributes != null);
		if (this.sessionAttributesFound) {
			this.sessionAttributeNames.addAll(Arrays.asList(sessionAttributes.names()));
			this.sessionAttributeTypes.addAll(Arrays.asList(sessionAttributes.types()));
		}
	}
```

之后，**org.springframework.web.bind.annotation.support.HandlerMethodInvoker**就可以开始干活了。在其方法`invokeHandlerMethod()`中，它从`modelAttributeMethods Set`遍历所有方法。如果之前model属性没有解析，它将通过创建对象来将请求参数绑定到对象的相应字段。

```Java
for (Method attributeMethod : this.methodResolver.getModelAttributeMethods()) {
				Method attributeMethodToInvoke = BridgeMethodResolver.findBridgedMethod(attributeMethod);
				Object[] args = resolveHandlerArguments(attributeMethodToInvoke, handler, webRequest, implicitModel);
				if (debug) {
					logger.debug("Invoking model attribute method: " + attributeMethodToInvoke);
				}
				String attrName = AnnotationUtils.findAnnotation(attributeMethod, ModelAttribute.class).value();
				if (!"".equals(attrName) && implicitModel.containsAttribute(attrName)) {
					continue;
				}
				ReflectionUtils.makeAccessible(attributeMethodToInvoke);
				Object attrValue = attributeMethodToInvoke.invoke(handler, args);
				if ("".equals(attrName)) {
					Class<?> resolvedType = GenericTypeResolver.resolveReturnType(attributeMethodToInvoke, handler.getClass());
					attrName = Conventions.getVariableNameForReturnType(attributeMethodToInvoke, resolvedType, attrValue);
				}
				if (!implicitModel.containsAttribute(attrName)) {
					implicitModel.addAttribute(attrName, attrValue);
				}
			}
```

通过**org.springframework.web.method.annotation.ModelAttributeMethodProcessor**来做绑定。更确切地说，它是通过方法**protected void bindRequestParameters(WebDataBinder binder，NativeWebRequest request)**来将请求绑定到目标对象。而更准确地说，它使用`WebRequestDataBinder`的`bind()`方法来做到这一点。

```Java
/**
	 * Extension point to bind the request to the target object.
	 * @param binder the data binder instance to use for the binding
	 * @param request the current request
	 */
	protected void bindRequestParameters(WebDataBinder binder, NativeWebRequest request) {
		((WebRequestDataBinder) binder).bind(request);
	}
	...

	/**
	 * 此处会在新版本的最后提到，到时可从此开始看的
	 * Resolve the argument from the model or if not found instantiate it with
	 * its default if it is available. The model attribute is then populated
	 * with request values via data binding and optionally validated
	 * if {@code @java.validation.Valid} is present on the argument.
	 * @throws BindException if data binding and validation result in an error
	 * and the next method parameter is not of type {@link Errors}.
	 * @throws Exception if WebDataBinder initialization fails.
	 */
	@Override
	public final Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

		String name = ModelFactory.getNameForParameter(parameter);
		Object attribute = (mavContainer.containsAttribute(name) ? mavContainer.getModel().get(name) :
				createAttribute(name, parameter, binderFactory, webRequest));

		if (!mavContainer.isBindingDisabled(name)) {
			ModelAttribute ann = parameter.getParameterAnnotation(ModelAttribute.class);
			if (ann != null && !ann.binding()) {
				mavContainer.setBindingDisabled(name);
			}
		}
		//此处来做绑定
		WebDataBinder binder = binderFactory.createBinder(webRequest, attribute, name);
		if (binder.getTarget() != null) {
			if (!mavContainer.isBindingDisabled(name)) {
				bindRequestParameters(binder, webRequest);
			}
			validateIfApplicable(binder, parameter);
			if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
				throw new BindException(binder.getBindingResult());
			}
		}

		// Add resolved attribute and BindingResult at the end of the model
		Map<String, Object> bindingResultModel = binder.getBindingResult().getModel();
		mavContainer.removeAttributes(bindingResultModel);
		mavContainer.addAllAttributes(bindingResultModel);

		return binder.convertIfNecessary(binder.getTarget(), parameter.getParameterType(), parameter);
	}
```

`WebRequestDataBinder`的`bind()`

```Java
/**
	 * Bind the parameters of the given request to this binder's target,
	 * also binding multipart files in case of a multipart request.
	 * <p>This call can create field errors, representing basic binding
	 * errors like a required field (code "required"), or type mismatch
	 * between value and bean property (code "typeMismatch").
	 * <p>Multipart files are bound via their parameter name, just like normal
	 * HTTP parameters: i.e. "uploadedFile" to an "uploadedFile" bean property,
	 * invoking a "setUploadedFile" setter method.
	 * <p>The type of the target property for a multipart file can be Part, MultipartFile,
	 * byte[], or String. The latter two receive the contents of the uploaded file;
	 * all metadata like original file name, content type, etc are lost in those cases.
	 * @param request request with parameters to bind (can be multipart)
	 * @see org.springframework.web.multipart.MultipartRequest
	 * @see org.springframework.web.multipart.MultipartFile
	 * @see javax.servlet.http.Part
	 * @see #bind(org.springframework.beans.PropertyValues)
	 */
	public void bind(WebRequest request) {
		MutablePropertyValues mpvs = new MutablePropertyValues(request.getParameterMap());
		if (isMultipartRequest(request) && request instanceof NativeWebRequest) {
			MultipartRequest multipartRequest = ((NativeWebRequest) request).getNativeRequest(MultipartRequest.class);
			if (multipartRequest != null) {
				bindMultipart(multipartRequest.getMultiFileMap(), mpvs);
			}
			else if (servlet3Parts) {
				HttpServletRequest serlvetRequest = ((NativeWebRequest) request).getNativeRequest(HttpServletRequest.class);
				new Servlet3MultipartHelper(isBindEmptyMultipartFiles()).bindParts(serlvetRequest, mpvs);
			}
		}
		doBind(mpvs);
	}
```

跟着源码 再追下去的话，会发现在其父类`DataBinder`中:

```Java
/**
	 * Actual implementation of the binding process, working with the
	 * passed-in MutablePropertyValues instance.
	 * @param mpvs the property values to bind,
	 * as MutablePropertyValues instance
	 * @see #checkAllowedFields
	 * @see #checkRequiredFields
	 * @see #applyPropertyValues
	 */
	protected void doBind(MutablePropertyValues mpvs) {
		checkAllowedFields(mpvs);
		checkRequiredFields(mpvs);
		applyPropertyValues(mpvs);
	}
```

`DataBinder`的`applyPropertyValues`方法中来对字段值进行设置:

```Java
protected void applyPropertyValues(MutablePropertyValues mpvs) {
    try {
        // Bind request parameters onto target object.
        getPropertyAccessor().setPropertyValues(mpvs, isIgnoreUnknownFields(), isIgnoreInvalidFields());
    }
    catch (PropertyBatchUpdateException ex) {
        // Use bind error processor to create FieldErrors.
        for (PropertyAccessException pae : ex.getPropertyAccessExceptions()) {
            getBindingErrorProcessor().processPropertyAccessException(pae, getInternalBindingResult());
        }
    }
}
```

首先，它得到一个**org.springframework.beans.AbstractPropertyAccessor**类(`getPropertyAccessor`)的实现。之后，通过具体实现这个抽象方法**public void setPropertyValue(String propertyName，Object value)**将`HTTP`请求中找到的值放入解析对象中。此方法由**org.springframework.beans**包中的`BeanWrapperImpl`和`DirectFieldAccessor`类*实现*。默认情况下，`ModelAttributeMethodProcessor`使用的类是**org.springframework.beans.BeanWrapperImpl，**这是`BeanWrapper`的默认实现。此默认实现可以设置和获取`bean`的属性(类字段)。它以这种方式实现一个`setPropertyValue`方法:

```Java
public abstract class AbstractNestablePropertyAccessor extends AbstractPropertyAccessor {
  ...
@Override
	public void setPropertyValue(String propertyName, Object value) throws BeansException {
		AbstractNestablePropertyAccessor nestedPa; //此处看下一段代码一眼便知
		try {
			nestedPa = getPropertyAccessorForPropertyPath(propertyName);
		}
		catch (NotReadablePropertyException ex) {
			throw new NotWritablePropertyException(getRootClass(), this.nestedPath + propertyName,
					"Nested property in path '" + propertyName + "' does not exist", ex);
		}
		PropertyTokenHolder tokens = getPropertyNameTokens(getFinalPath(nestedPa, propertyName));
		nestedPa.setPropertyValue(tokens, new PropertyValue(propertyName, value));
	}
  ...
}
```

```Java
/**
 * Default {@link BeanWrapper} implementation that should be sufficient
 * for all typical use cases. Caches introspection results for efficiency.
 *
 * <p>Note: Auto-registers default property editors from the
 * {@code org.springframework.beans.propertyeditors} package, which apply
 * in addition to the JDK's standard PropertyEditors. Applications can call
 * the {@link #registerCustomEditor(Class, java.beans.PropertyEditor)} method
 * to register an editor for a particular instance (i.e. they are not shared
 * across the application). See the base class
 * {@link PropertyEditorRegistrySupport} for details.
 *
 * <p><b>NOTE: As of Spring 2.5, this is - for almost all purposes - an
 * internal class.</b> It is just public in order to allow for access from
 * other framework packages. For standard application access purposes, use the
 * {@link PropertyAccessorFactory#forBeanPropertyAccess} factory method instead.
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @author Rob Harrop
 * @author Stephane Nicoll
 * @since 15 April 2001
 * @see #registerCustomEditor
 * @see #setPropertyValues
 * @see #setPropertyValue
 * @see #getPropertyValue
 * @see #getPropertyType
 * @see BeanWrapper
 * @see PropertyEditorRegistrySupport
 */
public class BeanWrapperImpl extends AbstractNestablePropertyAccessor implements BeanWrapper {
```

结果被转移到**private void setPropertyValue(PropertyTokenHolder tokens, PropertyValue pv)**，这里就不详细介绍了。你只需要知道它是一个设置一个字段的值的方法。set字段可以是一个简单的类型(String，int等)，也可以是一个集合(List，Map)。

以上介绍了在老版本中关于@ModelAttribute如何在Spring Web应用程序解析的。如上所看到的，代码执行的基本流程以`HandlerMethodResolver`对象开头，并以`ModelAttributeMethodProcessor`实例解析的可选对象结束。整个过程基于数据绑定，在DataBinder子类中实现。他们通过属性访问器(默认BeanWrapperImpl)从请求中获取键值对并将其放在目标对象中。

### 新版本

通过上面可以看出，老版本的代码其实穿梭的蛮复杂的，这里就通过新版的代码再来梳理下:

`@ModelAttribute`注解的方法是作用于整个`Controller`的，实际上在执行`Controller`的每个请求时都会执行`@ModelAttribute`注解的方法。

执行过程在`org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter`中查看，每次执行Controller时都会执行`@ModelAttribute`注解的方法:

```Java
/**
	 * Invoke the {@link RequestMapping} handler method preparing a {@link ModelAndView}
	 * if view resolution is required.
	 * @since 4.2 可以看到4.2开始启用了
	 * @see #createInvocableHandlerMethod(HandlerMethod)
	 */
	@Nullable
	protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

		ServletWebRequest webRequest = new ServletWebRequest(request, response);
		try {
			WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
			ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

			ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
			if (this.argumentResolvers != null) {
				invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
			}
			if (this.returnValueHandlers != null) {
				invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
			}
			invocableMethod.setDataBinderFactory(binderFactory);
			invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

			ModelAndViewContainer mavContainer = new ModelAndViewContainer();
			mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
           //执行@ModelAttribute注解的方法
			modelFactory.initModel(webRequest, mavContainer, invocableMethod);
			mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

			AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
			asyncWebRequest.setTimeout(this.asyncRequestTimeout);

			WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
			asyncManager.setTaskExecutor(this.taskExecutor);
			asyncManager.setAsyncWebRequest(asyncWebRequest);
			asyncManager.registerCallableInterceptors(this.callableInterceptors);
			asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

			if (asyncManager.hasConcurrentResult()) {
				Object result = asyncManager.getConcurrentResult();
				mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
				asyncManager.clearConcurrentResult();
				if (logger.isDebugEnabled()) {
					logger.debug("Found concurrent result value [" + result + "]");
				}
				invocableMethod = invocableMethod.wrapConcurrentResult(result);
			}
			//执行Controller中的方法
			invocableMethod.invokeAndHandle(webRequest, mavContainer);
			if (asyncManager.isConcurrentHandlingStarted()) {
				return null;
			}

			return getModelAndView(mavContainer, modelFactory, webRequest);
		}
		finally {
			webRequest.requestCompleted();
		}
	}
```

`modelFactory.initModel(webRequest, mavContainer, invocableMethod)`中会执行`@ModelAttribute`注解的方法(**org.springframework.web.method.annotation.ModelFactory**中可查看):

```Java
/**
	 * Populate the model in the following order:
	 * <ol>
	 * <li>Retrieve "known" session attributes listed as {@code @SessionAttributes}.
	 * <li>Invoke {@code @ModelAttribute} methods
	 * <li>Find {@code @ModelAttribute} method arguments also listed as
	 * {@code @SessionAttributes} and ensure they're present in the model raising
	 * an exception if necessary.
	 * </ol>
	 * @param request the current request
	 * @param container a container with the model to be initialized
	 * @param handlerMethod the method for which the model is initialized
	 * @throws Exception may arise from {@code @ModelAttribute} methods
	 */
	public void initModel(NativeWebRequest request, ModelAndViewContainer container,
			HandlerMethod handlerMethod) throws Exception {

		Map<String, ?> sessionAttributes = this.sessionAttributesHandler.retrieveAttributes(request);
		container.mergeAttributes(sessionAttributes);
      //执行@ModelAttribute注解的方法
		invokeModelAttributeMethods(request, container);
		////方法执行结果的值放到container
		for (String name : findSessionAttributeArguments(handlerMethod)) {
			if (!container.containsAttribute(name)) {
				Object value = this.sessionAttributesHandler.retrieveAttribute(request, name);
				if (value == null) {
					throw new HttpSessionRequiredException("Expected session attribute '" + name + "'", name);
				}
				container.addAttribute(name, value);
			}
		}
	}
```

在`private void invokeModelAttributeMethods(NativeWebRequest request, ModelAndViewContainer container)`中会判断方法上是否被`@ModelAttribute`注解，如果是则会执行这个方法，并将返回值放到`container`中:

```Java
/**
 * Invoke model attribute methods to populate the model.
 * Attributes are added only if not already present in the model.
 */
private void invokeModelAttributeMethods(NativeWebRequest request, ModelAndViewContainer container)
		throws Exception {

	while (!this.modelMethods.isEmpty()) {
		InvocableHandlerMethod modelMethod = getNextModelMethod(container).getHandlerMethod();
         //判断方法是否被@ModelAttribute注解
		ModelAttribute ann = modelMethod.getMethodAnnotation(ModelAttribute.class);
		Assert.state(ann != null, "No ModelAttribute annotation");
		if (container.containsAttribute(ann.name())) {
			if (!ann.binding()) {
				container.setBindingDisabled(ann.name());
			}
			continue;
		}
	 //执行被@ModelAttribute注解的方法
		Object returnValue = modelMethod.invokeForRequest(request, container);
		if (!modelMethod.isVoid()){
			String returnValueName = getNameForReturnValue(returnValue, modelMethod.getReturnType());
			if (!ann.binding()) {
				container.setBindingDisabled(returnValueName);
			}
			if (!container.containsAttribute(returnValueName)) {
				container.addAttribute(returnValueName, returnValue);
			}
		}
	}
}
```

我们进入**org.springframework.web.method.support.InvocableHandlerMethod** 的`invokeForRequest`方法，在给定`request`请求的上下文中解析其参数值后调用该方法，参数值通常通过 `HandlerMethodArgumentResolver`来解析。

```Java
/**
 * Invoke the method after resolving its argument values in the context of the given request.
 * <p>Argument values are commonly resolved through {@link HandlerMethodArgumentResolver}s.
 * The {@code providedArgs} parameter however may supply argument values to be used directly,
 * i.e. without argument resolution. Examples of provided argument values include a
 * {@link WebDataBinder}, a {@link SessionStatus}, or a thrown exception instance.
 * Provided argument values are checked before argument resolvers.
 * @param request the current request
 * @param mavContainer the ModelAndViewContainer for this request
 * @param providedArgs "given" arguments matched by type, not resolved
 * @return the raw value returned by the invoked method
 * @exception Exception raised if no suitable argument resolver can be found,
 * or if the method raised an exception
 */
@Nullable
public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
		Object... providedArgs) throws Exception {
	//看下面的方法
	Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
	if (logger.isTraceEnabled()) {
		logger.trace("Invoking '" + ClassUtils.getQualifiedMethodName(getMethod(), getBeanType()) +
				"' with arguments " + Arrays.toString(args));
	}
	Object returnValue = doInvoke(args);
	if (logger.isTraceEnabled()) {
		logger.trace("Method [" + ClassUtils.getQualifiedMethodName(getMethod(), getBeanType()) +
				"] returned [" + returnValue + "]");
	}
	return returnValue;
}

/**
 * Get the method argument values for the current request.
 */
private Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
		Object... providedArgs) throws Exception {

	MethodParameter[] parameters = getMethodParameters();
	Object[] args = new Object[parameters.length];
	for (int i = 0; i < parameters.length; i++) {
		MethodParameter parameter = parameters[i];
		parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
		args[i] = resolveProvidedArgument(parameter, providedArgs);
		if (args[i] != null) {
			continue;
		}
		if (this.argumentResolvers.supportsParameter(parameter)) {
			try {
                 //又回归到解析参数的老路上了，就不多解析了
				args[i] = this.argumentResolvers.resolveArgument(
						parameter, mavContainer, request, this.dataBinderFactory);
				continue;
			}
			catch (Exception ex) {
				if (logger.isDebugEnabled()) {
					logger.debug(getArgumentResolutionErrorMessage("Failed to resolve", i), ex);
				}
				throw ex;
			}
		}
		if (args[i] == null) {
			throw new IllegalStateException("Could not resolve method parameter at index " +
					parameter.getParameterIndex() + " in " + parameter.getExecutable().toGenericString() +
					": " + getArgumentResolutionErrorMessage("No suitable resolver for", i));
		}
	}
	return args;
}
```

**org.springframework.web.method.support.HandlerMethodArgumentResolverComposite**

```Java
/**
 * Iterate over registered {@link HandlerMethodArgumentResolver}s and invoke the one that supports it.
 * @throws IllegalStateException if no suitable {@link HandlerMethodArgumentResolver} is found.
 */
@Override
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
		NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

	HandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
	if (resolver == null) {
		throw new IllegalArgumentException("Unknown parameter type [" + parameter.getParameterType().getName() + "]");
	}
     //又回到老版本的resolveArgument路上了
	return resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
}
/**
 * Find a registered {@link HandlerMethodArgumentResolver} that supports the given method parameter.
 */
@Nullable
private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
	HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
	if (result == null) {
		for (HandlerMethodArgumentResolver methodArgumentResolver : this.argumentResolvers) {
			if (logger.isTraceEnabled()) {
				logger.trace("Testing if argument resolver [" + methodArgumentResolver + "] supports [" +
						parameter.getGenericParameterType() + "]");
			}
			if (methodArgumentResolver.supportsParameter(parameter)) {
				result = methodArgumentResolver;
				this.argumentResolverCache.put(parameter, result);
				break;
			}
		}
	}
	return result;
}
```

回到**org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter**,可以看到:

```Java
public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter
		implements BeanFactoryAware, InitializingBean {

	@Nullable
	private List<HandlerMethodArgumentResolver> customArgumentResolvers;

	@Nullable
	private HandlerMethodArgumentResolverComposite argumentResolvers;

	@Nullable
	private HandlerMethodArgumentResolverComposite initBinderArgumentResolvers;

	@Nullable
	private List<HandlerMethodReturnValueHandler> customReturnValueHandlers;
```

又回到老版本的`resolveArgument`这里了,就不往下解释了

关于`@ModelAttribute`的例子请看这篇博客文章的，自己就不整例子了<http://blog.csdn.net/hejingyuan6/article/details/49995987>

总之,通过源码可以看出,当`@ModelAttribute`注解方法时,这个方法在每次访问`Controller`时都会被执行,其执行到的原理就是在每次执行`Controller`时都会判断一次,并执行`@ModelAttribute`的方法,将执行后的结果值放到`container`中,其实我们只需要知道这么多就成了，背后的机制无论新老版本都是解析绑定这4个字。

# 666. 彩蛋

如果你对 Spring 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)