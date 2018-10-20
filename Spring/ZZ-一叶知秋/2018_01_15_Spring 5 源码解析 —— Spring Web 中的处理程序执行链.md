title: Spring 5 源码解析 —— Spring Web 中的处理程序执行链
date: 2018-01-15
tag: 
categories: Spring
permalink: Spring/HandlerExecutionChain
author: 一叶知秋
from_url: https://muyinchen.github.io/2017/08/28/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— Spring%20Web%E4%B8%AD%E7%9A%84%E5%A4%84%E7%90%86%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C%E9%93%BE/
wechat_url: 

-------

摘要: 原创出处 https://muyinchen.github.io/2017/08/28/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— Spring%20Web%E4%B8%AD%E7%9A%84%E5%A4%84%E7%90%86%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C%E9%93%BE/ 「一叶知秋」欢迎转载，保留摘要，谢谢！

- [什么是Spring中的处理程序执行链？](http://www.iocoder.cn/Spring/HandlerExecutionChain/)
- [HandlerExecutionChain类](http://www.iocoder.cn/Spring/HandlerExecutionChain/)
- [自定义处理程序执行链](http://www.iocoder.cn/Spring/HandlerExecutionChain/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

> Spring的DispatcherServlet假如缺少几个关键元素将无法分派请求。其中最重要的一个是处理程序执行链。
>
> 在这篇文章中，我们把注意力放在**处理程序执行链**之上。老规矩，第一部分将介绍这个概念。第二部分把目光引入到Spring执行链的世界中。在最后一部分，我们将分析如何在Spring中利用之前自定义DispatcherServlet中实现一个自定义的处理程序执行链。

## 什么是Spring中的处理程序执行链？

Spring中的处理程序执行链是一种由处理程序映射和处理程序拦截器(简单点说就是由谁来处理，处理之前和之后应该干点啥)组成的责任链设计模式。处理器映射器用于将当前请求与其专用的controller进行匹配。拦截器是用来在一些调度动作(如controller解析，视图渲染等)之前和之后所调用的对象。

我们所说的一个处理程序执行链是`dispatcher servlet`用来处理接收到的请求的一组元素。需要说的是，所有执行链调用都由`dispatcher servlet`类来进行。其实执行链只是一种容器(见源码):

- 定义处理程序映射和拦截器
- 定义在某些时刻应用所应该调度的方法(如处理程序适配器适配之后，`controller`的方法调用之后等等)

## HandlerExecutionChain类

处理程序执行链由**org.springframework.web.servlet.HandlerExecutionChain**类表示。它的主要包含两个私有字段:**Object handler** 和 **HandlerInterceptor[] interceptors**,它们被用在请求的调度过程中。第一个包含用于查找处理程序适配器实例的处理程序对象。第二个是包含拦截器的数组，用来应用于处理过的请求(这里这么说是因为这是一条执行链，一个接一个来对这个请求进行处理)。

在`DispatcherServlet`类中，`HandlerExecutionChain`的查找通过**protected HandlerExecutionChain getHandler(HttpServletRequest request)**完成。它遍历所有可用的处理程序映射，并返回能够处理请求的第一个处理程序。

在`DispatcherServlet`与`HandlerExecutionChain`实例中要完成的第二件事是应用拦截器的前后调用。这是由DispatcherServlet的方法，如`applyPreHandle`，`applyPostHandle`，`applyAfterConcurrentHandlingStarted`和`triggerAfterCompletion`(关于后两者我会在后面专门写由Java并发编程到线程池到forkjoin到nio到netty这个系列来具体讲述的)。

```java
/**
 * Handler execution chain, consisting of handler object and any handler interceptors.
 * Returned by HandlerMapping's {@link HandlerMapping#getHandler} method.
 *
 * @author Juergen Hoeller
 * @since 20.06.2003
 * @see HandlerInterceptor
 */
public class HandlerExecutionChain {

	private static final Log logger = LogFactory.getLog(HandlerExecutionChain.class);

	private final Object handler;

	@Nullable
	private HandlerInterceptor[] interceptors;

	@Nullable
	private List<HandlerInterceptor> interceptorList;

	private int interceptorIndex = -1;

  ...
    /**
	 * Trigger afterCompletion callbacks on the mapped HandlerInterceptors.
	 * Will just invoke afterCompletion for all interceptors whose preHandle invocation
	 * has successfully completed and returned true.
	 */
	void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response, @Nullable Exception ex)
			throws Exception {

		HandlerInterceptor[] interceptors = getInterceptors();
		if (!ObjectUtils.isEmpty(interceptors)) {
			for (int i = this.interceptorIndex; i >= 0; i--) {
				HandlerInterceptor interceptor = interceptors[i];
				try {
					interceptor.afterCompletion(request, response, this.handler, ex);
				}
				catch (Throwable ex2) {
					logger.error("HandlerInterceptor.afterCompletion threw exception", ex2);
				}
			}
		}
	}

    /**
	 * Apply afterConcurrentHandlerStarted callback on mapped AsyncHandlerInterceptors.
	 */
	void applyAfterConcurrentHandlingStarted(HttpServletRequest request, HttpServletResponse response) {
		HandlerInterceptor[] interceptors = getInterceptors();
		if (!ObjectUtils.isEmpty(interceptors)) {
			for (int i = interceptors.length - 1; i >= 0; i--) {
				if (interceptors[i] instanceof AsyncHandlerInterceptor) {
					try {
						AsyncHandlerInterceptor asyncInterceptor = (AsyncHandlerInterceptor) interceptors[i];
						asyncInterceptor.afterConcurrentHandlingStarted(request, response, this.handler);
					}
					catch (Throwable ex) {
						logger.error("Interceptor [" + interceptors[i] + "] failed in afterConcurrentHandlingStarted", ex);
					}
				}
			}
		}
	}
```

**org.springframework.web.servlet.DispatcherServlet**

```java
/**
	 * Return the HandlerExecutionChain for this request.
	 * <p>Tries all handler mappings in order.
	 * @param request current HTTP request
	 * @return the HandlerExecutionChain, or {@code null} if no handler could be found
	 */
	@Nullable
	protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		if (this.handlerMappings != null) {
			for (HandlerMapping hm : this.handlerMappings) {
				if (logger.isTraceEnabled()) {
					logger.trace(
							"Testing handler map [" + hm + "] in DispatcherServlet with name '" + getServletName() + "'");
				}
				HandlerExecutionChain handler = hm.getHandler(request);
				if (handler != null) {
					return handler;
				}
			}
		}
		return null;
	}

...

  protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
  		//HandlerExecutionChain定义出来，做成函数内局部变量可以做到逃逸管理，和request请求做到用完即毁，提		//高性能，防止内存泄漏
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
              	//拿到执行链
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null) {
					noHandlerFound(processedRequest, response);
					return;
				}

				// Determine handler adapter for the current request.
              	//通过处理器找到相应的适配器，其实就是个拓展代理，参考之前我写的Spring设计模式
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
				//提前批预处理
				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}

				// Actually invoke the handler.
              	//执行核心处理
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}
				//默认视图渲染
				applyDefaultViewName(processedRequest, mv);
              	//秋后算账
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
              	//如果是异步的话，不需要解释了吧，看上面英文
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

## 自定义处理程序执行链

为了说明处理程序执行链的使用，我们将从关于[Spring DispatcherServlet生命周期](https://muyinchen.github.io/2017/08/02/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-%E8%AE%BASpring%20DispatcherServlet%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F/)的文章中拿到我们自定义的`dispatcher servlet`类，并向其添加一个自定义的处理程序执行链。但是，无须深挖并使用`HandlerExecutionChain`中的所有`Spring`的类，我们来创建一个新的对象(`DumberHandlerExecutionChain`)，添加两个方法来调用拦截器，并在`DispatcherServlet`的类中使用它。请看下面编写的代码:

```java
// we start directly by doService method which handles incoming request
// retrieve execution chain and handler adapters adapted to received request
DumberHandlerExecutionChain executionChain = new DumberHandlerExecutionChain(getHandlerExecutionChain(request));
System.out.println("Working with following handler execution chain: "+executionChain);
HandlerAdapter adapter = getHandlerAdapter(executionChain.getHandler());

if (!executionChain.preHandle(request, response)) {
    throw new IllegalStateException("Some of defined interceptors weren't ivoked correctly.");
}
// handle the request and try to generate a ModelAndView instance
ModelAndView modelView = adapter.handle(request, response, executionChain.getHandler());
if (modelView == null) {
    throw new IllegalStateException("Handled ModelAndView can't be null (handled with adapter: "+adapter+")");
}
if (!modelView.isReference()) {
    throw new UnsupportedOperationException("Only view models defined as references can be used in this servlet");
}
executionChain.postHandle(request, response, modelView);
```

只需要改变3行代码。第一个是`DumberHandlerExecutionChain`实例的定义，替换掉`HandlerExecutionChain`。第二个更改是`applyPreHandler`和`applyPostHandler`方法。这段代码这样来看的话好理解吧。我们来看看DumberHandlerExecutionChain类的定义:

```java
public class DumberHandlerExecutionChain extends HandlerExecutionChain {

    public DumberHandlerExecutionChain(HandlerExecutionChain chain) {
        super(chain);
        System.out.println("Overriden constructor DumberHandlerExecutionChain invoked");
    }

    public boolean preHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
        for (HandlerInterceptor interceptor : getInterceptors()) {
            System.out.println("Running pre handler for :"+interceptor);
            if (!interceptor.preHandle(request, response, this.getHandler())) {
                System.out.println("An error occured on calling handler for "+interceptor);
                return false;
            }
        }
        return true;
    }

    public void postHandle(HttpServletRequest request, HttpServletResponse response, ModelAndView viewModel) throws Exception {
        for (HandlerInterceptor interceptor : getInterceptors()) {
            interceptor.postHandle(request, response, this.getHandler(), viewModel);
            System.out.println("Running post handler for :"+interceptor);
        }
    }

    @Override
    public String toString() {
        return "DumberHandlerExecutionChain {interceptors :"+Arrays.asList(this.getInterceptors())+", handler: "+this.getHandler()+"}";
    }
}
```

它们是前面提到的两种方法:`preHandle`和`postHandle`。两者很相似。他们首先迭代所有可用的拦截器。区别在于第一个调用拦截器的`preHandle`方法和第二个拦截器的`postHandle`方法。第二个区别是结果。如果所有拦截器正确完成了操作，`preHandle`将返回true。`postHandle`不返回任何东西(这里和`HandlerExecutionChain`源码内相应的方法实现大致一样，但是做了逻辑上的简单处理达到咱们想要的效果即可)。

但这两种方法并不是这个类的核心。它最重要的地方是调用这个父类的构造函数:

```java
// 1. Invoked directly by super(chain) call
public HandlerExecutionChain(Object handler) {
        this(handler, null);
}

// 2. Called directly by previous constructor
public HandlerExecutionChain(Object handler, @Nullable HandlerInterceptor... interceptors) {
		if (handler instanceof HandlerExecutionChain) {
			HandlerExecutionChain originalChain = (HandlerExecutionChain) handler;
			this.handler = originalChain.getHandler();
			this.interceptorList = new ArrayList<>();
			CollectionUtils.mergeArrayIntoCollection(originalChain.getInterceptors(), this.interceptorList);
			CollectionUtils.mergeArrayIntoCollection(interceptors, this.interceptorList);
		}
		else {
			this.handler = handler;
			this.interceptors = interceptors;
		}
	}
```

由上可以看到，通过`handler instanceof`检查，我们可以在`dispatcher servlet`中本地生成`HandlerExecutionChain`。我们不需要查找产生`HandlerExecutionChain`实例的处理程序映射(例如:`org.springframework.web.servlet.handler.AbstractHandlerMapping`或`org.springframework.web.servlet.handler.AbstractUrlHandlerMapping`的实现类)并覆盖重写现有代码。而不是使用这些复杂的步骤，我们只需简单地将`HandlerExecutionChain`的实例传递给我们自定义的执行链类的构造函数即可。

完成上面的工作，接下来，你可以在日志中看到以下信息:

```shell
Overriden constructor DumberHandlerExecutionChain invoked
//tag1
Working with following handler execution chain: DumberHandlerExecutionChain {interceptors :[org.springframework.web.servlet.handler.ConversionServiceExposingInterceptor@77f6d2e3, com.migo.interceptors.LotteryInterceptor@6d8f729c], handler: public java.lang.String com.migo.controller.TestController.test(javax.servlet.http.HttpServletRequest)}
//pre
Running pre handler for :org.springframework.web.servlet.handler.ConversionServiceExposingInterceptor@77f6d2e3
Running pre handler for :com.migo.interceptors.LotteryInterceptor@6d8f729c
[LotteryInterceptor] preHandle
//excute Controller
Controller asks, are you a lottery winner ? true
Current locale is :org.springframework.web.servlet.DispatcherServlet$1@5cf346dc
Request attributes are :org.apache.catalina.connector.RequestFacade@7d9ccb73
//post
Running post handler for :org.springframework.web.servlet.handler.ConversionServiceExposingInterceptor@77f6d2e3
[LotteryInterceptor] postHandle
Running post handler for :com.migo.interceptors.LotteryInterceptor@6d8f729c
```

在本文中，我们分析了在Spring 中dispatcher servlet中处理程序执行链的概念。而且已经知道，它不仅包含处理程序映射(在查找处理程序适配器之后用，源码有标注注释)，而且在不同的时间点会调用拦截器。接着，我们详细分析了`HandlerExecutionChain`类。里面有两个主要的私有字段，一个是一个处理程序，另一个是拦截器数组。在最后一部分，我们通过写一个我们自定义的处理程序执行链，以能够在我们自定义处理程序适配器的操作之前和之后调用拦截器。

# 666. 彩蛋

如果你对 Spring 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)