title: Spring 5 源码解析 —— 论 Spring DispatcherServlet 的生命周期
date: 2018-01-08
tag: 
categories: Spring
permalink: Spring/DispatcherServlet
author: 一叶知秋
from_url: https://muyinchen.github.io/2017/08/02/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— %E8%AE%BASpring%20DispatcherServlet%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F/
wechat_url: 

-------

摘要: 原创出处 https://muyinchen.github.io/2017/08/02/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— %E8%AE%BASpring%20DispatcherServlet%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F/ 「一叶知秋」欢迎转载，保留摘要，谢谢！

- [什么是前端控制器模式？](http://www.iocoder.cn/Spring/DispatcherServlet)
- [什么是DispatcherServlet的执行链？](http://www.iocoder.cn/Spring/DispatcherServlet)
- [什么是DispatcherServlet？](http://www.iocoder.cn/Spring/DispatcherServlet)
- [Custom DispatcherServlet](http://www.iocoder.cn/Spring/DispatcherServlet)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

Spring Web框架架构的主要部分是`DispatcherServlet`。也就是本文中重点介绍的对象。
在本文的第一部分中，我们将看到基于`Spring的DispatcherServlet`的主要概念：前端控制器模式。第二部分将专门介绍Spring应用程序中的执行链。接下来是`DispatcherServlet类`的解释。在最后一部分，我们将尝试开发一个自定义的`dispatcher servlet`。

请注意，本文分析的DispatcherServlet来自Spring的5.0.0.RC3版本。如果使用不同的版本，则可能需要进行几个调整，其实因为分析的都是比较固定的东西，很少有改的。

## 什么是前端控制器模式？

在进入`DispatcherServlet`之前，我们需要了解一些关于它的概念基础。`DispatcherServlet`所隐含的关键概念其实就是**前端控制器模式**。

此模式为Web应用程序提供了一个中心入口点。该集中入口点将系统组件的共同特征进行重新组合。我们可以在那里找到安全资源，语言切换，会话管理，缓存或输入过滤的处理程序。这样做的一个很大的好处是:这个共同的入口点有助于避免代码重复。

因此，从技术上讲，前端控制器模式由一个捕获所有传入请求的类组成。之后，分析每个请求以知道哪个控制器以及哪个方法应该来处理该请求。

前端控制器模式有助于对以下询问做出最佳响应:

- 如何集中授权和认证？
- 如何处理正确的视图渲染？
- 如何使用URL重写映射将请求发送到适当的控制器？

这个前台控制器模式包含5名参与者：

- 客户端：发送请求。
- 控制器：应用程序的中心点，捕获所有请求。
- 调度员：管理视图的选择，以呈现给客户端。
- 视图：表示呈现给客户端的内容。
- 帮助：帮助查看和/或控制器完成请求处理。

## 什么是DispatcherServlet的执行链？

由标题可以看到，前端控制器模式有自己的执行链。这意味着它有自己的逻辑来处理请求并将视图返回给客户端：

1. 请求由客户端发送。它到达作为Spring的默认前端控制器的`DispatcherServlet`类。

2. `DispatcherServlet`使用请求处理程序映射来发现将分析请求的控制器(controller

   )。接口**org.springframework.web.servlet.HandlerMapping**的实现返回一个包含**org.springframework.web.servlet.HandlerExecutionChain**类的实例。此实例包含可在控制器调用之前或之后调用的处理程序拦截器数组。你可以在Spring中有关于拦截器的文章中了解更多的信息。如果在所有定义的处理程序映射中找不到`HandlerExecutionChain`，这意味着Spring无法将URL与对应的控制器进行匹配。这样的话会抛出一个错误。

3. 现在系统进行拦截器预处理并调用由映射处理器找到的相应的controller(其实就是在找到的controller之前进行一波拦截处理)。在controller处理请求后，`DispatcherServlet`开始拦截器的后置处理。在此步骤结束时，它从controller接收ModelAndView实例(整个过程其实就是 `request请求`->`进入interceptors`->`controller`->`从interceptors出来`->`ModelAndView接收`)。

4. DispatcherServlet现在将使用的该视图的名称发送到视图解析器。这个解析器将决定前台的展现内容。接着，它将此视图返回给DispatcherServlet，其实也就是一个“视图生成后可调用”的拦截器。

5. 最后一个操作是视图的渲染并作为对客户端request请求的响应。

## 什么是DispatcherServlet？

通过上面讲到的前端控制器模式，我们可以很轻易的知道`DispatcherServlet`是基于`Spring`的`Web`应用程序的中心点。它需要传入请求，并在处理程序映射，拦截器，控制器和视图解析器的帮助下，生成对客户端的响应。所以，我们可以分析这个类的细节，并总结出一些核心要点。

下面是处理一个请求时`DispatcherServlet`执行的步骤：

#### 1. 策略初始化

`DispatcherServlet`是一个位于**org.springframework.web.servlet**包中的类，并扩展了同一个包中的抽象类`FrameworkServlet`。它包含一些解析器的私有静态字段(用于本地化，视图，异常或上传文件)，`映射处理器:handlerMapping`和`处理适配器:handlerAdapter`(进入这个类的第一眼就能看到的)。`DispatcherServlet`非常重要的一个核心点就是是初始化策略的方法(**protected void initStrategies（ApplicationContext context）**)。在调用`onRefresh`方法时调用此方法。最后一次调用是在`FrameworkServlet`中通过`initServletBean`和`initWebApplicationContext`方法进行的(`initServletBean`方法中调用`initWebApplicationContext`，后者调用`onRefresh(wac)`)。`initServletBean`通过所提供的这些策略生成我们所需要的应用程序上下文。其中每个策略都会产生一类在`DispatcherServlet`中用来处理传入请求的对象。

基于篇幅，有些代码就不给贴示了，请在相应版本的源码中自行对照查找，此处只给一部分源码:

```Java
/**
 * This implementation calls {@link #initStrategies}.
 */
@Override
protected void onRefresh(ApplicationContext context) {
	initStrategies(context);
}

/**
 * Initialize the strategy objects that this servlet uses.
 * <p>May be overridden in subclasses in order to initialize further strategy objects.
 */
protected void initStrategies(ApplicationContext context) {
	initMultipartResolver(context);
	initLocaleResolver(context);
	initThemeResolver(context);
	initHandlerMappings(context);
	initHandlerAdapters(context);
	initHandlerExceptionResolvers(context);
	initRequestToViewNameTranslator(context);
	initViewResolvers(context);
	initFlashMapManager(context);
}
```

需要注意的是，如果找的结果不存在，则捕获异常`NoSuchBeanDefinitionException`(下面两段代码的第一段)，并采用默认策略。如果在DispatcherServlet.properties文件中初始定义的默认策略不存在，则抛出BeanInitializationException异常(下面两段代码的第二段)。默认策略如下：

```Java
/**
	 * Initialize the LocaleResolver used by this class.
	 * <p>If no bean is defined with the given name in the BeanFactory for this namespace,
	 * we default to AcceptHeaderLocaleResolver.
	 */
	private void initLocaleResolver(ApplicationContext context) {
		try {
			this.localeResolver = context.getBean(LOCALE_RESOLVER_BEAN_NAME, LocaleResolver.class);
			if (logger.isDebugEnabled()) {
				logger.debug("Using LocaleResolver [" + this.localeResolver + "]");
			}
		}
		catch (NoSuchBeanDefinitionException ex) {
			// We need to use the default.
			this.localeResolver = getDefaultStrategy(context, LocaleResolver.class);
			if (logger.isDebugEnabled()) {
				logger.debug("Unable to locate LocaleResolver with name '" + LOCALE_RESOLVER_BEAN_NAME +"': using default [" + this.localeResolver + "]");
			}
		}
	}
```

抛出异常后调用`getDefaultStrategy`(因为容器里都是单例的存在，所以只需要判断基于这个接口的默认实现实例size为1即可，两个以上还能叫默认么，都有选择了):

```Java
/**
 * Return the default strategy object for the given strategy interface.
 * The default implementation delegates to {@link #getDefaultStrategies},
 * expecting a single object in the list.
 * @param context the current WebApplicationContext
 * @param strategyInterface the strategy interface
 * @return the corresponding strategy object
 * @see #getDefaultStrategies
 */
protected <T> T getDefaultStrategy(ApplicationContext context, Class<T> strategyInterface) {
    List<T> strategies = getDefaultStrategies(context, strategyInterface);
    if (strategies.size() != 1) {
        throw new BeanInitializationException(
                "DispatcherServlet needs exactly 1 strategy for interface [" + strategyInterface.getName() + "]");
    }
    return strategies.get(0);
}

	/**
	 * Create a List of default strategy objects for the given strategy interface.
	 * The default implementation uses the "DispatcherServlet.properties" file (in the same
	 * package as the DispatcherServlet class) to determine the class names. It instantiates
	 * the strategy objects through the context's BeanFactory.
	 * @param context the current WebApplicationContext
	 * @param strategyInterface the strategy interface
	 * @return the List of corresponding strategy objects
	 */
	@SuppressWarnings("unchecked")
	protected <T> List<T> getDefaultStrategies(ApplicationContext context, Class<T> strategyInterface) {
		String key = strategyInterface.getName();
		String value = defaultStrategies.getProperty(key);
		if (value != null) {
			String[] classNames = StringUtils.commaDelimitedListToStringArray(value);
			List<T> strategies = new ArrayList<>(classNames.length);
			for (String className : classNames) {
				try {
					Class<?> clazz = ClassUtils.forName(className, DispatcherServlet.class.getClassLoader());
					Object strategy = createDefaultStrategy(context, clazz);
					strategies.add((T) strategy);
				}
				catch (ClassNotFoundException ex) {
					throw new BeanInitializationException("Could not find DispatcherServlet's default strategy class [" + className +"] for interface [" + key + "]", ex);
				}
				catch (LinkageError err) {
					throw new BeanInitializationException(
							"Error loading DispatcherServlet's default strategy class [" + className +"] for interface [" + key + "]: problem with class file or dependent class", err);
				}
			}
			return strategies;
		}
		else {
			return new LinkedList<>();
		}
	}
```

#### 2.请求预处理

`FrameworkServlet`抽象类扩展了同一个包下的`HttpServletBean`，`HttpServletBean`扩展了**javax.servlet.http.HttpServlet**。点开这个类源码可以看到，`HttpServlet`是一个抽象类，其方法定义主要用来处理每种类型的`HTTP`请求：`doGet（GET请求）`，`doPost（POST）`，`doPut（PUT）`，`doDelete（DELETE）`，`doTrace（TRACE）`，`doHead（HEAD）` ，`doOptions（OPTIONS）`。`FrameworkServlet`通过将每个传入的请求调度到**processRequest(HttpServletRequest request，HttpServletResponse response)**来覆盖它们。`processRequest`是一个`protected`和`final`的方法，它构造出`LocaleContext`和`ServletRequestAttributes`对象，两者都可以在`initContextHolders(request, localeContext, requestAttributes)`之后访问。所有这些操作的关键代码 请看：

```Java
@Override
protected final void doGet(HttpServletRequest request, HttpServletResponse response)
        throws ServletException, IOException {

    processRequest(request, response);
}
 ...
/**
 * Process this request, publishing an event regardless of the outcome.
 * The actual event handling is performed by the abstract
 * {@link #doService} template method.
 */
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
        throws ServletException, IOException {

    long startTime = System.currentTimeMillis();
    Throwable failureCause = null;

    LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
    LocaleContext localeContext = buildLocaleContext(request);

    RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
    ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);

    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
    asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), new RequestBindingInterceptor());

    initContextHolders(request, localeContext, requestAttributes);

    try {
        doService(request, response);
    }
    catch (ServletException ex) {
        failureCause = ex;
        throw ex;
    }
    catch (IOException ex) {
        failureCause = ex;
        throw ex;
    }
    catch (Throwable ex) {
        failureCause = ex;
        throw new NestedServletException("Request processing failed", ex);
    }

    finally {
        resetContextHolders(request, previousLocaleContext, previousAttributes);
        if (requestAttributes != null) {
            requestAttributes.requestCompleted();
        }

        if (logger.isDebugEnabled()) {
            if (failureCause != null) {
                this.logger.debug("Could not complete request", failureCause);
            }
            else {
                if (asyncManager.isConcurrentHandlingStarted()) {
                    logger.debug("Leaving response open for concurrent processing");
                }
                else {
                    this.logger.debug("Successfully completed request");
                }
            }
        }

        publishRequestHandledEvent(request, startTime, failureCause);
    }
}

...

  private void initContextHolders(HttpServletRequest request,
			@Nullable LocaleContext localeContext, @Nullable RequestAttributes requestAttributes) {

		if (localeContext != null) {
			LocaleContextHolder.setLocaleContext(localeContext, this.threadContextInheritable);
		}
		if (requestAttributes != null) {
			RequestContextHolder.setRequestAttributes(requestAttributes, this.threadContextInheritable);
		}
		if (logger.isTraceEnabled()) {
			logger.trace("Bound request context to thread: " + request);
		}
	}
```

#### 3.请求处理

由上面所看到的，在`processRequest`的代码中，调用**initContextHolders**方法后，调用**protected void doService(HttpServletRequest request，HttpServletResponse response)**。doService将一些附加参数放入request（如Flash映射:`request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap())`，上下文信息:`request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext())`等）中，并调用**protected void doDispatch(HttpServletRequest request，HttpServletResponse response)**。

`doDispatch`方法最重要的部分是处理(`handler`)的检索。`doDispatch`调用`getHandler()`方法来分析处理后的请求并返回`HandlerExecutionChain`实例。此实例包含`handler mapping` 和``interceptors(拦截器)`。`DispatcherServlet`做的另一件事是应用预处理程序拦截器（*applyPreHandle()*）。如果至少有一个返回`false`，则请求处理停止。否则，`servlet`使用与 `handler adapter`适配(其实理解成这也是个`handler`就对了)相应的`handler mapping`来生成视图对象。

`doDispatch`方法：

```Java
/**
 * Process the actual dispatching to the handler.
 * The handler will be obtained by applying the servlet's HandlerMappings in order.
 * The HandlerAdapter will be obtained by querying the servlet's installed HandlerAdapters
 * to find the first that supports the handler class.
 * All HTTP methods are handled by this method. It's up to HandlerAdapters or handlers
 * themselves to decide which methods are acceptable.
 * @param request current HTTP request
 * @param response current HTTP response
 * @throws Exception in case of any kind of processing failure
 */
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		boolean multipartRequestParsed = false;

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		try {
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
				processedRequest = checkMultipart(request);
				multipartRequestParsed = (processedRequest != request);

				// Determine handler for the current request.
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null) {
					noHandlerFound(processedRequest, response);
					return;
				}

				// Determine handler adapter for the current request.获取可处理request的Handler，适配器其实还				//是调用的相应的Handler,一样的功能，具体请参考本人的Spring设计模式中的适配器模式
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

				// Process last-modified header, if supported by the handler.
				String method = request.getMethod();
				boolean isGet = "GET".equals(method);
				if (isGet || "HEAD".equals(method)) {
					long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
					if (logger.isDebugEnabled()) {
						logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
					}
					if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
						return;
					}
				}

				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}

				// Actually invoke the handler.此处就会调用我们写的controller来执行咯
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}
				//视图解析
				applyDefaultViewName(processedRequest, mv);
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			catch (Exception ex) {
				dispatchException = ex;
			}
			catch (Throwable err) {
				// As of 4.3, we're processing Errors thrown from handler methods as well,
				// making them available for @ExceptionHandler methods and other scenarios.
				dispatchException = new NestedServletException("Handler dispatch failed", err);
			}
          	//此处进行最后一步的视图渲染
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		catch (Exception ex) {
			triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
		}
		catch (Throwable err) {
			triggerAfterCompletion(processedRequest, response, mappedHandler,
					new NestedServletException("Handler processing failed", err));
		}
		finally {
			if (asyncManager.isConcurrentHandlingStarted()) {
				// Instead of postHandle and afterCompletion
				if (mappedHandler != null) {
					mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
				}
			}
			else {
				// Clean up any resources used by a multipart request.
				if (multipartRequestParsed) {
					cleanupMultipart(processedRequest);
				}
			}
		}
	}
```

#### 4.视图解析

获取`ModelAndView`实例以查看呈现后，`doDispatch`方法调用**private void applyDefaultViewName(HttpServletRequest request，ModelAndView mv)**。默认视图名称根据定义的bean名称，即`viewNameTranslator`。默认情况下，它的实现是**org.springframework.web.servlet.RequestToViewNameTranslator**。这个默认实现只是简单的将URL转换为视图名称，例如(直接从`RequestToViewNameTranslator`获取):http:// localhost:8080/admin/index.html将生成视图admin / index。

代码如下:

下一步是调用后置拦截器(其实就是出拦截器)做的一些处理。

```Java
/** RequestToViewNameTranslator used by this servlet */
	@Nullable
	private RequestToViewNameTranslator viewNameTranslator;
...
  protected void initStrategies(ApplicationContext context) {
		initMultipartResolver(context);
		initLocaleResolver(context);
		initThemeResolver(context);
		initHandlerMappings(context);
		initHandlerAdapters(context);
		initHandlerExceptionResolvers(context);
		initRequestToViewNameTranslator(context); //看下面注释
		initViewResolvers(context);
		initFlashMapManager(context);
	}
...

  /**
	 * Initialize the RequestToViewNameTranslator used by this servlet instance.
	 * <p>If no implementation is configured then we default to DefaultRequestToViewNameTranslator.
	 */
	private void initRequestToViewNameTranslator(ApplicationContext context) {
		try {
			this.viewNameTranslator =
					context.getBean(REQUEST_TO_VIEW_NAME_TRANSLATOR_BEAN_NAME, RequestToViewNameTranslator.class);
			if (logger.isDebugEnabled()) {
				logger.debug("Using RequestToViewNameTranslator [" + this.viewNameTranslator + "]");
			}
		}
		catch (NoSuchBeanDefinitionException ex) {
			// We need to use the default.
			this.viewNameTranslator = getDefaultStrategy(context, RequestToViewNameTranslator.class);
			if (logger.isDebugEnabled()) {
				logger.debug("Unable to locate RequestToViewNameTranslator with name '" +
						REQUEST_TO_VIEW_NAME_TRANSLATOR_BEAN_NAME + "': using default [" + this.viewNameTranslator +"]");
			}
		}
	}

....
  /**
	 * Translate the supplied request into a default view name.
	 * @param request current HTTP servlet request
	 * @return the view name (or {@code null} if no default found)
	 * @throws Exception if view name translation failed
	 */
	@Nullable
	protected String getDefaultViewName(HttpServletRequest request) throws Exception {
		return (this.viewNameTranslator != null ? this.viewNameTranslator.getViewName(request) : null);
	}
```

`org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator`实现的`org.springframework.web.servlet.RequestToViewNameTranslator`接口，其内对上段代码中`getDefaultViewName`的实现为:

```Java
public class DefaultRequestToViewNameTranslator implements RequestToViewNameTranslator {

	private static final String SLASH = "/";


	private String prefix = "";

	private String suffix = "";

	private String separator = SLASH;

	private boolean stripLeadingSlash = true;

	private boolean stripTrailingSlash = true;

	private boolean stripExtension = true;

	private UrlPathHelper urlPathHelper = new UrlPathHelper();


	/**
	 * Set the prefix to prepend to generated view names.
	 * @param prefix the prefix to prepend to generated view names
	 */
	public void setPrefix(String prefix) {
		this.prefix = (prefix != null ? prefix : "");
	}

	/**
	 * Set the suffix to append to generated view names.
	 * @param suffix the suffix to append to generated view names
	 */
	public void setSuffix(String suffix) {
		this.suffix = (suffix != null ? suffix : "");
	}

	/**
	 * Set the value that will replace '{@code /}' as the separator
	 * in the view name. The default behavior simply leaves '{@code /}'
	 * as the separator.
	 */
	public void setSeparator(String separator) {
		this.separator = separator;
	}
	...
	/**
	 * Translates the request URI of the incoming {@link HttpServletRequest}
	 * into the view name based on the configured parameters.
	 * @see org.springframework.web.util.UrlPathHelper#getLookupPathForRequest
	 * @see #transformPath
	 */
	@Override
	public String getViewName(HttpServletRequest request) {
		String lookupPath = this.urlPathHelper.getLookupPathForRequest(request);
		return (this.prefix + transformPath(lookupPath) + this.suffix);
	}

	/**
	 * Transform the request URI (in the context of the webapp) stripping
	 * slashes and extensions, and replacing the separator as required.
	 * @param lookupPath the lookup path for the current request,
	 * as determined by the UrlPathHelper
	 * @return the transformed path, with slashes and extensions stripped
	 * if desired
	 */
	protected String transformPath(String lookupPath) {
		String path = lookupPath;
		if (this.stripLeadingSlash && path.startsWith(SLASH)) {
			path = path.substring(1);
		}
		if (this.stripTrailingSlash && path.endsWith(SLASH)) {
			path = path.substring(0, path.length() - 1);
		}
		if (this.stripExtension) {
			path = StringUtils.stripFilenameExtension(path);
		}
		if (!SLASH.equals(this.separator)) {
			path = StringUtils.replace(path, SLASH, this.separator);
		}
		return path;
	}

}
```

#### 5.处理调度请求 - 视图渲染

现在，`servlet`知道应该是哪个视图被渲染。它通过**private void processDispatchResult(HttpServletRequest request，HttpServletResponse response，@Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,@Nullable Exception exception)**方法来进行最后一步操作 - 视图渲染。

首先，`processDispatchResult`检查它们是否有参数传递异常。有一些异常的话，它定义了一个新的视图，专门用来定位错误页面。如果没有任何异常，该方法将检查`ModelAndView实例`，如果它不为`null`，则调用`render`方法。

渲染方法`protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception`。跳进此方法内部，根据定义的视图策略，它会查找得到一个`View类`实例。它将负责显示响应。如果没有找到`View`，则会抛出一个`ServletException异常`。有的话，`DispatcherServlet`会调用其`render`方法来显示结果。

其实可以说成是后置拦截器(进入拦截器前置拦截处理->controller处理->出拦截器之前的此拦截器的后置处理)，也就是在请求处理的最后一个步骤中被调用。

下面是`processDispatchResult`和`render(渲染)`的相关代码:

```Java
/**
	 * Handle the result of handler selection and handler invocation, which is
	 * either a ModelAndView or an Exception to be resolved to a ModelAndView.
	 */
	private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
			@Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
			@Nullable Exception exception) throws Exception {

		boolean errorView = false;

		if (exception != null) {
			if (exception instanceof ModelAndViewDefiningException) {
				logger.debug("ModelAndViewDefiningException encountered", exception);
				mv = ((ModelAndViewDefiningException) exception).getModelAndView();
			}
			else {
				Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
				mv = processHandlerException(request, response, handler, exception);
				errorView = (mv != null);
			}
		}

		// Did the handler return a view to render?
		if (mv != null && !mv.wasCleared()) {
          //开始渲染
			render(mv, request, response);
			if (errorView) {
				WebUtils.clearErrorRequestAttributes(request);
			}
		}
		else {
			if (logger.isDebugEnabled()) {
				logger.debug("Null ModelAndView returned to DispatcherServlet with name '" + getServletName() +
						"': assuming HandlerAdapter completed request handling");
			}
		}

		if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
			// Concurrent handling started during a forward
			return;
		}

		if (mappedHandler != null) {
			mappedHandler.triggerAfterCompletion(request, response, null);
		}
	}


...
  /**
	 * Render the given ModelAndView.
	 * <p>This is the last stage in handling a request. It may involve resolving the view by name.
	 * @param mv the ModelAndView to render
	 * @param request current HTTP servlet request
	 * @param response current HTTP servlet response
	 * @throws ServletException if view is missing or cannot be resolved
	 * @throws Exception if there's a problem rendering the view
	 */
	protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
		// Determine locale for request and apply it to the response.
		Locale locale =
				(this.localeResolver != null ? this.localeResolver.resolveLocale(request) : request.getLocale());
		response.setLocale(locale);

		View view;
		String viewName = mv.getViewName();
		if (viewName != null) {
			// We need to resolve the view name.
			view = resolveViewName(viewName, mv.getModelInternal(), locale, request);
			if (view == null) {
				throw new ServletException("Could not resolve view with name '" + mv.getViewName() +
						"' in servlet with name '" + getServletName() + "'");
			}
		}
		else {
			// No need to lookup: the ModelAndView object contains the actual View object.
			view = mv.getView();
			if (view == null) {
				throw new ServletException("ModelAndView [" + mv + "] neither contains a view name nor a " +"View object in servlet with name '" + getServletName() + "'");
			}
		}

		// Delegate to the View object for rendering.
		if (logger.isDebugEnabled()) {
			logger.debug("Rendering view [" + view + "] in DispatcherServlet with name '" + getServletName() + "'");
		}
		try {
			if (mv.getStatus() != null) {
				response.setStatus(mv.getStatus().value());
			}
			view.render(mv.getModelInternal(), request, response);
		}
		catch (Exception ex) {
			if (logger.isDebugEnabled()) {
				logger.debug("Error rendering view [" + view + "] in DispatcherServlet with name '" +
						getServletName() + "'", ex);
			}
			throw ex;
		}
	}
```

```Java
/**
 * Handle the result of handler selection and handler invocation, which is
 * either a ModelAndView or an Exception to be resolved to a ModelAndView.
 */
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
        HandlerExecutionChain mappedHandler, ModelAndView mv, Exception exception) throws Exception {

    boolean errorView = false;

    if (exception != null) {
        if (exception instanceof ModelAndViewDefiningException) {
            logger.debug("ModelAndViewDefiningException encountered", exception);
            mv = ((ModelAndViewDefiningException) exception).getModelAndView();
        }
        else {
            Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
            mv = processHandlerException(request, response, handler, exception);
            errorView = (mv != null);
        }
    }

    // Did the handler return a view to render?
    if (mv != null && !mv.wasCleared()) {
        render(mv, request, response);
        if (errorView) {
            WebUtils.clearErrorRequestAttributes(request);
        }
    }
    else {
        if (logger.isDebugEnabled()) {
            logger.debug("Null ModelAndView returned to DispatcherServlet with name '" + getServletName() +
                    "': assuming HandlerAdapter completed request handling");
        }
    }

    if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
        // Concurrent handling started during a forward
        return;
    }

    if (mappedHandler != null) {
        mappedHandler.triggerAfterCompletion(request, response, null);
    }
}

/**
 * Render the given ModelAndView.
 * This is the last stage in handling a request. It may involve resolving the view by name.
 * @param mv the ModelAndView to render
 * @param request current HTTP servlet request
 * @param response current HTTP servlet response
 * @throws ServletException if view is missing or cannot be resolved
 * @throws Exception if there's a problem rendering the view
 */
protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
    // Determine locale for request and apply it to the response.
    Locale locale = this.localeResolver.resolveLocale(request);
    response.setLocale(locale);

    View view;
    if (mv.isReference()) {
        // We need to resolve the view name.
        view = resolveViewName(mv.getViewName(), mv.getModelInternal(), locale, request);
        if (view == null) {
            throw new ServletException(
                    "Could not resolve view with name '" + mv.getViewName() + "' in servlet with name '" + getServletName() + "'");
        }
    }
    else {
        // No need to lookup: the ModelAndView object contains the actual View object.
        view = mv.getView();
        if (view == null) {
            throw new ServletException("ModelAndView [" + mv + "] neither contains a view name nor a " + "View object in servlet with name '" + getServletName() + "'");
        }
    }

    // Delegate to the View object for rendering.
    if (logger.isDebugEnabled()) {
        logger.debug("Rendering view [" + view + "] in DispatcherServlet with name '" + getServletName() + "'");
    }
    try {
        view.render(mv.getModelInternal(), request, response);
    }
    catch (Exception ex) {
        if (logger.isDebugEnabled()) {
            logger.debug("Error rendering view [" + view + "] in DispatcherServlet with name '" + getServletName() + "'", ex);
        }
        throw ex;
    }
}
```

在这部分中，你需要记住的是我们定义了两个上下文：一个用于应用程序，另一个用于Web应用程序。他们有什么区别？应用程序上下文包含所有通用配置，比如service定义，数据库配置。Web应用程序上下文定义所有与Web相关的组件，比如`controllers`或视图解析器。

## Custom DispatcherServlet

我们已经了解了`DispatcherServlet`的理论知识。通过文中的这些实用要点，我们可以编写自己的servlet来分派处理请求。同样的，我们也将按步进行，从捕获请求开始，以视图渲染结束。

通过上面的描述，为了捕获请求，我们需要覆盖`doService`方法：

```Java
public class CustomDispatcherServlet extends FrameworkServlet {
    private static final Logger LOGGER = LoggerFactory.getLogger(CustomDispatcherServlet.class);

    @Override
    protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
        LOGGER.debug("[CustomDispatcherServlet] I got the request !");
    }
}
```

这样，在我们的日志文件中，我们应该可以找到一条“[CustomDispatcherServlet]I got the request！”。接着，我们继续添加在`DispatcherServlet`中`doDispatch方法`所应该做的一些工作：

```Java
@Override
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
    LOGGER.debug("[CustomDispatcherServlet] I got the request !");
    try {
        LOGGER.debug("[CustomDispatcherServlet] doService");
        LocaleContext localeContext = buildLocaleContext(request);

        RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
        ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);

        initContextHolders(request, localeContext, requestAttributes);
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

这个方法是做什么的？首先，它为构建一个`Locale实例`用来接收请求。第二步是初始化**org.springframework.web.context.request.ServletRequestAttributes**实例。它是`RequestAttributes`接口的实现，和本地化在同一级别。通过这个，我们可以访问`servlet`请求的对象和会话对象，而不必区分会话和全局会话。最后，我们调用**初始化context holders**的`initContextHolders()`方法，即从应用程序通过`LocaleContextHolder`和`RequestContextHolder`静态方法(分别为:`getLocaleContext和getRequestAttributes`)访问请求属性和区域设置的对象。

当请求被拦截，一些基本的设置就绪的时候。我们发现我们还没有执行链和处理器适配器。我们可以通过以下代码进行:

```Java
private HandlerExecutionChain getHandlerExecutionChain(HttpServletRequest request) throws Exception {
    for (HandlerMapping mapper : this.handlerMappings) {
        HandlerExecutionChain executionChain = mapper.getHandler(request);
        if (executionChain != null) {
            return executionChain;
        }
    }
    throw new Exception("Execution chain wasn't be found in provided handler mappings: "+this.handlerMappings);
}
```

通过执行链，我们可以通过 handler adapter将处理当前请求。看以下代码:

```Java
@Override
protected HandlerAdapter getHandlerAdapter(Object executionChain) throws ServletException {
    for (HandlerAdapter adapter : this.handlerAdapters) {
        LOGGER.debug("[CustomDispatcherServlet] "+adapter + " is instanceof HandlerMethod ? "+(adapter instanceof HandlerMethod));
        if (adapter.supports(executionChain)) {
            return adapter;
        }
    }
    throw new  ServletException("Handler adapter was not found from adapters list :"+this.handlerAdapters);
}
```

只有应用程序上下文中定义的适配器（`this.handlerAdapter`）支持适配所生成的执行链（`adapter.supports`）才可以返回我们想要的适配器。最后，我们可以返回到我们的`doService`方法并操作它们来渲染视图：

```Java
ModelAndView modelView = adapter.handle(request, response, executionChain.getHandler());
Locale locale = this.localeResolver.resolveLocale(request);
response.setLocale(locale);

View view = null;
if (!modelView.isReference()) {
    throw new UnsupportedOperationException("Only view models defined as references can be used in this servlet");
}
for (ViewResolver viewResolver : this.viewResolvers) {
    view = viewResolver.resolveViewName(modelView.getViewName(), locale);
    if (view != null) {
        break;
    }
}
if (view == null) {
    throw new ServletException("Could not resolve view with name '" + modelView.getViewName() + "' in servlet with name '" + getServletName() + "'");
}
view.render(modelView.getModelMap(), request, response);
```

我们的servlet中简化了渲染。实际上，我们仅处理`ModelAndView`的引用对象。这意味着`ModelAndView`是一个`String`的实例，用来表示要解析的视图模型，例如：我们定义好几个模板解析器(比如`freemaker`，`Thymeleaf`)，然后查看其配置。在这个检查之后，我们迭代当前视图解析器。能够生成View实例的第一个解析器被视为处理过的请求中使用的解析器。最后，我们检查视图是否正确生成。拿到view实例后，我们调用其render()方法来在屏幕中显示请求处理结果。

在这部分中，我们将描述和代码部分限制在最低限度。只是为了把Spring的整个过程给集中呈现以下，达到更好的理解，其实就是在Servlet中的service方法内做些对request和response的文章而已了。

> 本文介绍了Spring Web应用程序的中心点，一个调度器servlet。请记住，它是一个处理所有传入请求并将视图呈现给用户的类。在重写之前，你应该熟悉执行链，handler mapping 或handler adapter等概念。请记住，第一步要做的是定义在调度过程中我们要调用的所有元素。handler mapping 是将传入请求(也就是它的URL)映射到适当的controller。最后提到的元素，一个handler适配器，就是一个对象，它将通过其内包装的handler mapping将请求发送到controller。此调度产生的结果是ModelAndView类的一个实例，后面被用于生成和渲染视图。

# 666. 彩蛋

如果你对 Spring 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)