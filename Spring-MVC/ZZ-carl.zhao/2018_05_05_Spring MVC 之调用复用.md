title: 【carl.zhao】Spring MVC 之调用复用
date: 2018-05-05
tag: 
categories: Spring-MVC
permalink: Spring-MVC/carlzhao/Invoke
author: carlzhao
from_url: http://blog.csdn.net/u012410733/article/details/76862212
wechat_url: 

-------

摘要: 原创出处 http://blog.csdn.net/u012410733/article/details/76862212 「carlzhao」欢迎转载，保留摘要，谢谢！

- [1、Spring MVC Init](http://www.iocoder.cn/Spring-MVC/carlzhao/Invoke/)
- [3、Method Argument Resolver](http://www.iocoder.cn/Spring-MVC/carlzhao/Invoke/)
- [4、Spring MVC Invoke](http://www.iocoder.cn/Spring-MVC/carlzhao/Invoke/)
  - [4.1 @InitBinder](http://www.iocoder.cn/Spring-MVC/carlzhao/Invoke/)
  - [4.2 @ModelAttribute](http://www.iocoder.cn/Spring-MVC/carlzhao/Invoke/)
  - [4.3 @ExceptionHandler](http://www.iocoder.cn/Spring-MVC/carlzhao/Invoke/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

如果大家看过Spring MVC的源代码都会知道.Spring MVC框架在Spring容器初始化的时候,通过@RequestMapping建立起请求路径与调用方法的映射(没有看过源码的同学应该也能够想明白)。

# 1、Spring MVC Init

下面我们来看一下Spring MVC创建映射的代码时序图。

![这里写图片描述](http://static.iocoder.cn/csdn/20170807230145159?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQxMDczMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

我们先来看一下RequestMappingInfo里面的属性，然后再来说一下整个时序图干了哪些事。

```Java
public final class RequestMappingInfo implements RequestCondition<RequestMappingInfo> {

    // 请求映射名称,对应@RequestMapping注解中name
    private final String name;

    // 请求映射资源路径条件(uri),对应@RequestMapping注解中path
    private final PatternsRequestCondition patternsCondition;

    // 请求映射资源方法(RequestMethod -- GET/POST/PUT/DELETE等),对应@RequestMapping注解中method
    private final RequestMethodsRequestCondition methodsCondition;

    // 请求映射参数,用于缩小映射,对应@RequestMapping注解中params
    private final ParamsRequestCondition paramsCondition;

    // 请求映射对应header参数,对应@RequestMapping注解中headers
    private final HeadersRequestCondition headersCondition;

    // 请求映射对应Content-Type,对应@RequestMapping注解中consumes
    private final ConsumesRequestCondition consumesCondition;

    // 请求映射响应Content-Type,对应@RequestMapping注解中produces
    private final ProducesRequestCondition producesCondition;

    // 请求映射自定义条件,对应RequestMappingHandlerMapping#getCustomMethodCondition
    private final RequestConditionHolder customConditionHolder;

}

```

我们可以看到RequestMappingInfo以及它的属性都实现了RequestCondition这个接口。里面有三个方法：

- combine ： 用于合并类上面与方法上面的@RequestMapping注解信息。
- getMatchingCondition ： 获取到对应匹配的Condition;
- compareTo ： 当有多个满足时，选择最佳匹配。

这样就可以RequestMappingInfo就是Spring MVC映射是http请求映射的信息。然后我们再来分析上面的时序图。

1. 当Spring容器实例化AbstractHandlerMethodMapping对象的继承类的时候,首先调用Spring生命周期实现接口InitializingBean的回调方法afterPropertiesSet。
2. 然后调用ApplicationContext中的getBeanNamesForType，拿到Spring容器中所有注解的bean对象名称。
3. 循环遍历beanname拿到bean对象，从这个对象中通过反射拿到类上标注了@Controller或者@RequestMapping注解的类
4. 然后通过反射从这个类上面拿到所有方法标注了@RequestMapping的方法，并且产生标注@RequestMapping与RequestMappingInfo的映射Map

```Java
public class HandlerMethod {

    protected final Log logger = LogFactory.getLog(getClass());

    private final Object bean;

    private final BeanFactory beanFactory;

    private final Class<?> beanType;

    private final Method method;

    private final Method bridgedMethod;

    private final MethodParameter[] parameters;

    private HttpStatus responseStatus;

    private String responseStatusReason;

    private HandlerMethod resolvedFromHandlerMethod;

}
```

其中最重要有三个属性:

- bean : 当前handler处理类,标注@Controller的实例对象。
- method : 处理方法,也就是标注了@RequestMapping的方法对象。
- parameters : 处理方法的方法参数,也就是Spring MVC需要作参数绑定的对象。当Spring MVC接收到HTTP请求的时候,Spring MVC从HttpServletRequest对象里面获取到前端传递过来的请求参数。然后通过HandlerMethodArgumentResolver接口，通过策略模式把HttpServletRequest里面的请求参数绑定到对应方法中的方法中。

有了这三个参数，Spring MVC就可以使用反射.

```Java
Object returnValue = method.invoke(bean, parameters);1
```

 2、Spring MVC handler invoke

要明白Spring MVC对于HTTP请求的调用，首先我们要看一下HandlerMethod的类继承关系图了。

![这里写图片描述](http://static.iocoder.cn/csdn/20170807235630372?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQxMDczMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

其实它还有一个继承类，只是与我们的分析无关就没有把它画出来。在这个类图中，把关键属性于关键方法都表示了出来。下面我们再来看一下RequestMappingHandlerAdapter这个对象。这个对象是

```Java
public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter
        implements BeanFactoryAware, InitializingBean {

    // 请求参数处理类,把HttpServletRequest里面的请求参数与HandlerMethod的MethodParameter转化成@RequestMapping的方法参数
    private HandlerMethodArgumentResolverComposite argumentResolvers;

    // 返回参数处理类
    private HandlerMethodReturnValueHandlerComposite returnValueHandlers;

    // @Controller类中@InitBinder注解处理方法缓存
    private final Map<Class<?>, Set<Method>> initBinderCache = new ConcurrentHashMap<Class<?>, Set<Method>>(64);

    // @ControllerAdvice中@InitBinder注解处理方法缓存
    private final Map<ControllerAdviceBean, Set<Method>> initBinderAdviceCache =
            new LinkedHashMap<ControllerAdviceBean, Set<Method>>();

    // @ControllerAdvice中@ModelAttribute注解处理方法缓存
    private final Map<ControllerAdviceBean, Set<Method>> modelAttributeAdviceCache =
            new LinkedHashMap<ControllerAdviceBean, Set<Method>>();

    // HTTP请求处理方法
    protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
            HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

        ServletWebRequest webRequest = new ServletWebRequest(request, response);
        try {
            // 数据绑定工厂类
            WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
            ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

            // HTTP调用处理类
            ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
            invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
            invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
            invocableMethod.setDataBinderFactory(binderFactory);
            invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

            ModelAndViewContainer mavContainer = new ModelAndViewContainer();
            mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
            // 处理@ModelAttribute
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

            // 最终会调用到InvocableHandlerMethod#invokeForRequest
            // 这里就会处理HttpServletRequest与HandlerMethod的MethodParameter转化成@RequestMapping的方法参数
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
}

```

这个类里面的HandlerMethodArgumentResolverComposite里面有Spring MVC对于请求参数解析的默认处理集合。具体可以看RequestMappingHandlerAdapter#afterPropertiesSet方法，当这个对象实例化的时候就会把这些处理类添加进去。HandlerMethodReturnValueHandlerComposite里面是Spring MVC对于返回参数的默认处理集合，同样的也是由上面的方法添加进去的。其它属性是Spring MVC对于注解@ControllerAdvice、@InitBinder、@ModelAttribute、@ExceptionHandler的方法的处理。其实Spring MVC最终都会把这些注解中的方法调用封装成InvocableHandlerMethod这个对象或其子类。最后达到代码复用的目的。

# 3、Method Argument Resolver

这里我们要讲的是的是Spring MVC调用代码复用，那么我们首先看一下Spring MVC对于请求方法参数的解析。也就是InvocableHandlerMethod#getMethodArgumentValues。下面我们来看一下这个方法的具体代码：

```Java
    private Object[] getMethodArgumentValues(NativeWebRequest request, ModelAndViewContainer mavContainer,
            Object... providedArgs) throws Exception {

        MethodParameter[] parameters = getMethodParameters();
        Object[] args = new Object[parameters.length];
        for (int i = 0; i < parameters.length; i++) {

            // 通过提供的参数(providedArgs)来获取参数
            MethodParameter parameter = parameters[i];
            parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
            args[i] = resolveProvidedArgument(parameter, providedArgs);
            if (args[i] != null) {
                continue;
            }

            // 通过Spring中的HandlerMethodArgumentResolver来获取参数
            if (this.argumentResolvers.supportsParameter(parameter)) {
                try {
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

            // 如果上面都获取不到，报错
            if (args[i] == null) {
                throw new IllegalStateException("Could not resolve method parameter at index " +
                        parameter.getParameterIndex() + " in " + parameter.getMethod().toGenericString() +
                        ": " + getArgumentResolutionErrorMessage("No suitable resolver for", i));
            }
        }
        return args;
    }
```

通过上面的代码我们可以看到，这个方法有3个参数。

- request : 这个对象包含HttpServletRequst与HttpServletResponse对象。
- mavContainer ： 这个对象包括DispatcherServlet#INPUT_FLASH_MAP_ATTRIBUTE与@ModelAttribute里面的值。
- providedArgs : 传入对象，主要是Spring MVC异常处理传入Exception信息。

通过传入的这3个参数，就可以首先遍历providedArgs看是否有值的对象属性MethodParameter的类型的实例。如果是就获取到这个请求参数对象，否则就通过HandlerMethodArgumentResolver接口会解析请求参数。如果都拿不到这个请求参数对象，那么就会报错。

> **第一种方式调用主要针对二种情况：**

- @InitBinder注解。进行这种方式调用的时候它会传入一个WebDataBinder对象。这样我们就可以修改这个对象的值了。
  具体入口在InitBinderDataBinderFactory#initBinder

```Java
    public void initBinder(WebDataBinder binder, NativeWebRequest request) throws Exception {
        for (InvocableHandlerMethod binderMethod : this.binderMethods) {
            if (isBinderMethodApplicable(binderMethod, binder)) {
                Object returnValue = binderMethod.invokeForRequest(request, null, binder);
                if (returnValue != null) {
                    throw new IllegalStateException("@InitBinder methods should return void: " + binderMethod);
                }
            }
        }
    }
```

典型用法为：

```Java
    @InitBinder
    public void init(DataBinder binder){
        binder.addCustomFormatter(new DateFormatter("yyyy-MM-dd mm:HH:ss"));
    }
```

- @ExceptionHandler注解：它会把Exception，Throwable，HandlerMethod对象传过来。具体的入口在ExceptionHandlerExceptionResolver#doResolveHandlerMethodException：

```Java
    protected ModelAndView doResolveHandlerMethodException(HttpServletRequest request,
            HttpServletResponse response, HandlerMethod handlerMethod, Exception exception) {

        ServletInvocableHandlerMethod exceptionHandlerMethod = getExceptionHandlerMethod(handlerMethod, exception);
        if (exceptionHandlerMethod == null) {
            return null;
        }

        exceptionHandlerMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
        exceptionHandlerMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);

        ServletWebRequest webRequest = new ServletWebRequest(request, response);
        ModelAndViewContainer mavContainer = new ModelAndViewContainer();

        try {
            if (logger.isDebugEnabled()) {
                logger.debug("Invoking @ExceptionHandler method: " + exceptionHandlerMethod);
            }
            Throwable cause = exception.getCause();
            if (cause != null) {
                // Expose cause as provided argument as well
                exceptionHandlerMethod.invokeAndHandle(webRequest, mavContainer, exception, cause, handlerMethod);
            }
            else {
                // Otherwise, just the given exception as-is
                exceptionHandlerMethod.invokeAndHandle(webRequest, mavContainer, exception, handlerMethod);
            }
        }
        catch (Throwable invocationEx) {
            ...
        }
    }
```

典型用法为：

```Java
    @ExceptionHandler(value = {Exception.class})
    @ResponseBody
    public CommonResponse handlerException(HttpServletRequest request, HttpServletResponse response, Exception e){
        if(e instanceof MethodArgumentNotValidException) {
            MethodArgumentNotValidException exception = (MethodArgumentNotValidException) e;
            List<ObjectError> allErrors = exception.getBindingResult().getAllErrors();
            if(CollectionUtils.isEmpty(allErrors)){
                return new CommonResponse(-1, "系统异常");
            }
            StringBuilder sb = new StringBuilder();
            for (ObjectError allError : allErrors) {
                sb.append(allError.getDefaultMessage()).append("\n");
            }
            return new CommonResponse(-1, sb.toString());
        }
        if(e instanceof BindException) {
            BindException exception = (BindException) e;
            List<ObjectError> allErrors = exception.getBindingResult().getAllErrors();
            if(CollectionUtils.isEmpty(allErrors)){
                return new CommonResponse(-1, "系统异常");
            }
            StringBuilder sb = new StringBuilder();
            for (ObjectError allError : allErrors) {
                sb.append(allError.getDefaultMessage()).append("\n");
            }
            return new CommonResponse(-1, sb.toString());
        }
        return new CommonResponse(-1, "系统异常");
    }
```

当然它们都可以放在@Controller里面，也可以放在@ControllerAdvice里面。这样可以方便的控制它们的作用范围。

# 4、Spring MVC Invoke

上面我们解析了Spring MVC中的HTTP请求的具体调用过程。下面我们就来分析一下Spring MVC使用这一种方式的其它调用。来领略一下Spring代码的复用艺术。

## 4.1 @InitBinder

注解@InitBinder可以应用到@Controller类中的方法上也可以应用@ControllerAdvice的方法上。只不是一个局部的，是作用于当前类的HTTP请求;而另一个是全局的，作用为所有的HTTP请求。在调用HTTP请求的时候都会@Controller类与@ControllerAdvice类中的@InitBinder方法都检索出来，只不过在具体HTTP调用的时候是否应用就要看是不是匹配了。下面分析一下作用于@Controller里面的@InitBinder.

![这里写图片描述](http://static.iocoder.cn/csdn/20170808221212536?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQxMDczMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

我们可以看到在容器初始化的时候并不会初始@InitBinder这个注解标注了的方法。而是在首次HTTP调用的时候会初始化当前处理方法类中的@InitBinder方法。以及@ControllerAdvice中的@InitBinder方法。这样的好处就是每个Controller中即可以使用全局的InitBinder方法也可以和其它Controller相互隔绝。它的调用时候是先进行HTTP的处理调用，在这个处理调用的参数解析的时候会进行InitBinder方法处理。这里是和HTTP处理方法都是调用的InvocableHandlerMethod#invokeForRequest。

## 4.2 @ModelAttribute

注解@ModelAttribute可以应用到@Controller类中的方法上也可以应用@ControllerAdvice的方法上。只不是一个局部的，是作用于当前类的HTTP请求;而另一个是全局的，作用为所有的HTTP请求。

![这里写图片描述](http://static.iocoder.cn/csdn/20170808223352968?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQxMDczMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

这里只分析了一下@ModelAttribute注解在当前类的时候，并没有分析在@ControllerAdvice。大家如果感兴趣可以自行分析。可以看到这个过程发生在HTTP正式处理之前。把获取到的值塞到ModelAndViewContainer中，用于后面使用。这里是和HTTP处理方法都是调用的InvocableHandlerMethod#invokeForRequest。

## 4.3 @ExceptionHandler

![这里写图片描述](http://static.iocoder.cn/csdn/20170808230538874?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQxMDczMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

注解@ExceptionHandler可以应用到@Controller类中的方法上也可以应用@ControllerAdvice的方法上。只不是一个局部的，是作用于当前类的HTTP请求;而另一个是全局的，作用为所有的HTTP请求。

它的效果与@InitBinder类似，即可以有全局的配置又可以有局部的配置。当HTTP请求处理发生异常的时候，它会首先在当前类找是否有对应异常的@ExceptionHandler处理方法如果有就用它进行处理。如果没有，就找一下@ControllerAdvice注解类里面是否有对应异常的处理，有就处理。如果没有就用Spring MVC的默认异常处理方式。

# 666. 彩蛋

如果你对 Java 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)