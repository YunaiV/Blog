title: Spring 5 源码解析 —— Spring 中的处理拦截器
date: 2018-01-10
tag: 
categories: Spring
permalink: Spring/interceptor
author: 一叶知秋
from_url: https://muyinchen.github.io/2017/08/07/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— Spring%E4%B8%AD%E7%9A%84%E5%A4%84%E7%90%86%E6%8B%A6%E6%88%AA%E5%99%A8/
wechat_url: 

-------

摘要: 原创出处 https://muyinchen.github.io/2017/08/07/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— Spring%E4%B8%AD%E7%9A%84%E5%A4%84%E7%90%86%E6%8B%A6%E6%88%AA%E5%99%A8/ 「一叶知秋」欢迎转载，保留摘要，谢谢！

- [什么是Spring中的处理程序拦截器？](http://www.iocoder.cn/Spring/interceptor/)
- [拦截器和过滤器之间的区别](http://www.iocoder.cn/Spring/interceptor/)
- [什么是默认的Spring拦截器？](http://www.iocoder.cn/Spring/interceptor/)
- [在Spring中自定义处理程序拦截器](http://www.iocoder.cn/Spring/interceptor/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

在Java的Web应用程序中通常使用过滤器(即filter)来捕获HTTP请求。但它们仅为webapps保留。Spring引入了一种新的方法来实现，更通用，称为处理程序拦截器。

本文将分3部分。第一部分来讲Spring处理程序拦截器的理论概念。第二部分，说一说默认的Spring拦截器。最后一部分老规矩，应用实战，我们将写我们自己的处理程序拦截器。

## 什么是Spring中的处理程序拦截器？

要了解Spring拦截器的作用，我们需要先解释一下HTTP请求的执行链。DispatcherServlet捕获每个请求。调度员做的第一件事就是将接收到的URL和相应的controller进行映射(controller必须恰到好处地处理当前的请求)。但是，在到达对应的controller之前，请求可以被拦截器处理。这些拦截器就像过滤器。只有当URL找到对应于它们的映射时才调用它们。在通过拦截器(拦截器预处理，其实也可以说前置处理)进行前置处理后，请求最终到达controller。之后，发送请求生成视图。但是在这之前，拦截器还是有可能来再次处理它(拦截器后置处理)。只有在最后一次操作之后，视图解析器才能捕获数据并输出视图。

处理程序映射拦截器基于**org.springframework.web.servlet.HandlerInterceptor**接口。和之前简要描述的那样，它们可以在将其发送到控制器(方法前使用**preHandle**)之前或之后(方法后使用**postHandle**)拦截请求。`preHandle`方法返回一个`布尔值`，如果返回`false`，则可以在执行链中`执行中断请求处理`。此接口中还有一个方法**afterCompletion**，只有在`preHandler`方法发送为true时才会在渲染视图后调用它(完成请求处理后的回调，即渲染视图后)。

拦截器也可以在新线程中启动。在这种情况下，拦截器必须实现**org.springframework.web.servlet.AsyncHandlerInterceptor**接口。它继承`HandlerInterceptor`并提供一个方法**afterConcurrentHandlingStarted**。每次处理程序得到正确执行时，都会调用此方法而不是调用`postHandler()`和`afterCompletion()`。它也可以对发送请求进行异步处理。通过Spring源码此方法注释可以知道，这个方法的典型的应用是可以用来清理本地线程变量。

```Java
/**
 * Extends {@code HandlerInterceptor} with a callback method invoked after the
 * start of asynchronous request handling.
 *
 * <p>When a handler starts an asynchronous request, the {@link DispatcherServlet}
 * exits without invoking {@code postHandle} and {@code afterCompletion} as it
 * normally does for a synchronous request, since the result of request handling
 * (e.g. ModelAndView) is likely not yet ready and will be produced concurrently
 * from another thread. In such scenarios, {@link #afterConcurrentHandlingStarted}
 * is invoked instead, allowing implementations to perform tasks such as cleaning
 * up thread-bound attributes before releasing the thread to the Servlet container.
 *
 * <p>When asynchronous handling completes, the request is dispatched to the
 * container for further processing. At this stage the {@code DispatcherServlet}
 * invokes {@code preHandle}, {@code postHandle}, and {@code afterCompletion}.
 * To distinguish between the initial request and the subsequent dispatch
 * after asynchronous handling completes, interceptors can check whether the
 * {@code javax.servlet.DispatcherType} of {@link javax.servlet.ServletRequest}
 * is {@code "REQUEST"} or {@code "ASYNC"}.
 *
 * <p>Note that {@code HandlerInterceptor} implementations may need to do work
 * when an async request times out or completes with a network error. For such
 * cases the Servlet container does not dispatch and therefore the
 * {@code postHandle} and {@code afterCompletion} methods will not be invoked.
 * Instead, interceptors can register to track an asynchronous request through
 * the {@code registerCallbackInterceptor} and {@code registerDeferredResultInterceptor}
 * methods on {@link org.springframework.web.context.request.async.WebAsyncManager
 * WebAsyncManager}. This can be done proactively on every request from
 * {@code preHandle} regardless of whether async request processing will start.
 *
 * @author Rossen Stoyanchev
 * @since 3.2
 * @see org.springframework.web.context.request.async.WebAsyncManager
 * @see org.springframework.web.context.request.async.CallableProcessingInterceptor
 * @see org.springframework.web.context.request.async.DeferredResultProcessingInterceptor
 */
public interface AsyncHandlerInterceptor extends HandlerInterceptor {

	/**
	 * Called instead of {@code postHandle} and {@code afterCompletion}, when
	 * the a handler is being executed concurrently.
	 * <p>Implementations may use the provided request and response but should
	 * avoid modifying them in ways that would conflict with the concurrent
	 * execution of the handler. A typical use of this method would be to
	 * clean up thread-local variables.
	 * @param request the current request
	 * @param response the current response
	 * @param handler the handler (or {@link HandlerMethod}) that started async
	 * execution, for type and/or instance examination
	 * @throws Exception in case of errors
	 */
	void afterConcurrentHandlingStarted(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception;

}
```

## 拦截器和过滤器之间的区别

拦截器看起来很像servlet过滤器，为什么Spring不采用默认的Java解决方案？这其中主要区别就是`两者的作用域`的问题。过滤器只能在servlet容器下使用。而我们的Spring容器不一定运行在web环境中，在这种情况下过滤器就不好使了，而拦截器依然可以在Spring容器中调用。

Spring通过拦截器为请求提供了一个更细粒度的控制。就像我们之前看到的那样，它们可以在controller对请求处理之前或之后被调用，也可以在将渲染视图呈现给用户之后被调用。如果是过滤器的话，只能在将响应返回给最终用户之前使用它们。

下一个不同之处在于中断链执行的难易程度。拦截器可以通过在`preHandler()`方法内返回`false`来简单实现。而在过滤器的情况下，它就变得复杂了，因为它必须处理请求和响应对象来引发中断，需要一些额外的动作，比如如将用户重定向到错误页面。

## 什么是默认的Spring拦截器？

Spring主要将拦截器用于切换操作。比如我们最常用的功能之一是区域设置更改(也就是本地化更改)。请查看**org.springframework.web.servlet.i18n.LocaleChangeInterceptor**类中源码，可以通过我们所定义的语言环境解析器来对HTTP请求进行分析来实现。所有区域设置解析器都会分析请求元素(headers，Cookie)，以确定向用户提供哪种本地化语言设置。

另一个本地拦截器是**org.springframework.web.servlet.theme.ThemeChangeInterceptor**，它允许更改视图的主题(见此类的注释)。它还使用主题解析器更精确地来知道要使用的主题(参照下面**preHandle**方法)。它的解析器也基于请求分析(cookie，会话或参数)。

```Java
/**
 * Interceptor that allows for changing the current theme on every request,
 * via a configurable request parameter (default parameter name: "theme").
 *
 * @author Juergen Hoeller
 * @since 20.06.2003
 * @see org.springframework.web.servlet.ThemeResolver
 */
public class ThemeChangeInterceptor extends HandlerInterceptorAdapter {

	/**
	 * Default name of the theme specification parameter: "theme".
	 */
	public static final String DEFAULT_PARAM_NAME = "theme";

	private String paramName = DEFAULT_PARAM_NAME;


	/**
	 * Set the name of the parameter that contains a theme specification
	 * in a theme change request. Default is "theme".
	 */
	public void setParamName(String paramName) {
		this.paramName = paramName;
	}

	/**
	 * Return the name of the parameter that contains a theme specification
	 * in a theme change request.
	 */
	public String getParamName() {
		return this.paramName;
	}


	@Override
	public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws ServletException {

		String newTheme = request.getParameter(this.paramName);
		if (newTheme != null) {
			ThemeResolver themeResolver = RequestContextUtils.getThemeResolver(request);
			if (themeResolver == null) {
				throw new IllegalStateException("No ThemeResolver found: not in a DispatcherServlet request?");
			}
			themeResolver.setThemeName(request, response, newTheme);
		}
		// Proceed in any case.
		return true;
	}

}
```

## 在Spring中自定义处理程序拦截器

我们写一个例子来简单实现HandlerInterceptor。一个乐透彩票的场景，这个自定义的拦截器将分析每个请求，并决定是否是彩票的“lottery winner”。为了简化代码逻辑，只有用于生成一个随机数并通过取模判断是否返回0的请求。

```Java
public class LotteryInterceptor implements HandlerInterceptor {

    public static final String ATTR_NAME = "lottery_winner";
    private static final Logger LOGGER = LoggerFactory.getLogger(LotteryInterceptor.class);

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception exception) throws Exception {
        LOGGER.debug("[LotteryInterceptor] afterCompletion");

    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView view) throws Exception {
        LOGGER.debug("[LotteryInterceptor] postHandle");

    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        LOGGER.debug("[LotteryInterceptor] preHandle");
        if (request.getSession().getAttribute(ATTR_NAME) == null) {
            Random random = new Random();
            int i = random.nextInt(10);
            request.getSession().setAttribute(ATTR_NAME, i%2 == 0);
        }
        return true;
    }

}
```

关于相应controller中要展示的信息:

```Java
@Controller
public class TestController {
        private static final Logger LOGGER = LoggerFactory.getLogger(TestController.class);
    @RequestMapping(value = "/test", method = RequestMethod.GET)
    public String test(HttpServletRequest request) {
        LOGGER.debug("Controller asks, are you a lottery winner ? "+request.getSession().getAttribute(LotteryInterceptor.ATTR_NAME));
        return "test";
    }
}
```

如果我们尝试访问`/test`，我们将看不到拦截器的日志，因为它没有在配置中定义。如果我们是使用注解来配置的webapp。我们需要将下面这个配置添加到应用程序的上下文文件中(Springboot配置个相应的bean就可):

```xml
<mvc:interceptors>
    <bean class="com.migo.interceptors.LotteryInterceptor" />
</mvc:interceptors>
```

现在我们可以访问/ test页面并检查日志:

```Java
[LotteryInterceptor] preHandle
Controller asks, are you a lottery winner ? false
[LotteryInterceptor] postHandle
[LotteryInterceptor] afterCompletion
```

总结一下，拦截器是一种可以应用到整个Spring生态系统中的servlet过滤器。它们可以在请求之前或之后启动，也可以在视图呈现之后启动。它们也可以通过AsyncHandlerInterceptor接口的实现达到异步处理的效果。

# 666. 彩蛋

如果你对 Spring 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)