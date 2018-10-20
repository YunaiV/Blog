title: 【carl.zhao】Spring MVC 与 Servlet
date: 2018-05-04
tag: 
categories: Spring-MVC
permalink: Spring-MVC/carlzhao/Servlet
author: carlzhao
from_url: http://blog.csdn.net/u012410733/article/details/76732339
wechat_url: 

-------

摘要: 原创出处 http://blog.csdn.net/u012410733/article/details/76732339 「carlzhao」欢迎转载，保留摘要，谢谢！

- [1、Servlet容器的加载顺序](http://www.iocoder.cn/Spring-MVC/carlzhao/Servlet/)
- [2、Servlet的生命周期](http://www.iocoder.cn/Spring-MVC/carlzhao/Servlet/)
- [3、Spring MVC父子容器及初始化组件](http://www.iocoder.cn/Spring-MVC/carlzhao/Servlet/)
  - [3.1 通过反射创建root容器对象](http://www.iocoder.cn/Spring-MVC/carlzhao/Servlet/)
  - [3.2 加载相应配置文件，初始化root容器](http://www.iocoder.cn/Spring-MVC/carlzhao/Servlet/)
  - [3.3 把root容器放在ServletContext中](http://www.iocoder.cn/Spring-MVC/carlzhao/Servlet/)
  - [3.4 关联Servlet WebApplicationContext与 Root WebApplicationContext](http://www.iocoder.cn/Spring-MVC/carlzhao/Servlet/)
  - [3.5 初始化Spring MVC默认组件](http://www.iocoder.cn/Spring-MVC/carlzhao/Servlet/)
- [4、Spring MVC的处理HTTP请求](http://www.iocoder.cn/Spring-MVC/carlzhao/Servlet/)
  - [4.1 URL到框架的映射](http://www.iocoder.cn/Spring-MVC/carlzhao/Servlet/)
  - [4.2 http请求参数绑定](http://www.iocoder.cn/Spring-MVC/carlzhao/Servlet/)
  - [4.3 http响应的生成和输出](http://www.iocoder.cn/Spring-MVC/carlzhao/Servlet/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

相信大家都能够在上网上看到Spring MVC的核心类其实就是DispatherServlet，也就是Spring MVC处理请求的核心分发器。其实核心分发器几乎是所有MVC框架设计中的核心概念，像在Struts2也有类似的分发器FilterDispatcher。只不过Spring MVC中的是一个Servlet，而Struts2里面的是一个Filter.既然我们知道了Spring MVC的中核心类DispatcherServlet是一个Servlet。下面我们就来了解一下Servlet相关的概念。以助于我们后面分析Spring MVC的整体框架。

# 1、Servlet容器的加载顺序

相信大多数写Java Web程序的都接触过web.xml这个配置文件。下面我们就来看一下里面最主要的几个元素。

- context-param : 对于所有Servlet都可用的全局变量,被封装于ServletContext时。
- listener : 基于观察者模式设计的，用于对开发 Servlet 应用程序提供了一种快捷的手段，能够方便的从另一个纵向维度控制程序和数据。
- filter : 用于拦截Servlet，可以在调用Servlet之前或者之后修改Servlet中Request或者Response的值。
- servlet : 用于监听并处理所有的Web请求。在service及其相关方法中对Http请求的处理流程。

这里是Servlet容器中我们使用到最多的几个元素。它们在Servlet里面的初始化顺序如下：

> context-param –> listener -> filter -> servlet

具体可以参见：[web.xml 初始化顺序](http://blog.csdn.net/u012410733/article/details/74853601)

# 2、Servlet的生命周期

了解了Servlet里面的主要元素的加载顺序，下面我们就来看一下Servlet里面的生命周期：

```Java
public interface Servlet {

    public void init(ServletConfig config) throws ServletException;

    public ServletConfig getServletConfig();

    public void service(ServletRequest req, ServletResponse res)
    throwsServletException;, IOException;

    public String getServletInfo();

    public void destroy();

}
```

我们可以看到在Servlet接口里面有5个方面。

- init()：Servlet的初始化方法,因为Servlet是单例，所以整个系统启动时，只会运行一次。
- getServletConfig() : 获取Servlet配置信息，也就是之前所说的context-param配置的值。
- service() : 监听并响应客户端的请求
- getServletInfo() : 获取Servlet信息
- destroy() : Servlet容器调用，用于销毁Servlet.

所以，Servlet生命周期分为三个阶段：

1. 初始化阶段，调用init()方法
2. 响应客户请求阶段，调用service()方法
3. 终止阶段，调用destroy()方法

了解了这些Servlet的相关信息，我相信下面再来分析Spring MVC大家就会清楚很多。

# 3、Spring MVC父子容器及初始化组件

在Spring中我们都知道，它的核心概念是bean。而Spring容器又是通过BeanFactory来管理bean的。当我们查看Spring的BeanFactory的体系结构的时候不难发现HierarchicalBeanFactory这个接口。

```Java
public interface HierarchicalBeanFactory extends BeanFactory {

    BeanFactory getParentBeanFactory();

    boolean containsLocalBean(String name);`

}
```

可以看到它有一个getParentBeanFactory()方法。那么Spring框架就可以建立起.器的结构.在使用Spring MVC的时候我们可以看出，Spring MVC的体系是基于Spring容器来的。下面我们就来看一下Spring MVC是如何构建起父子容器的。在此之前我们先来看一段web.xml配置文件。

```XML
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:applicationContext.xml</param-value>
    </context-param>

    <!-- init root webapplicationcontext -->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <!-- init servlet webapplicationcontext -->
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>/WEB-INF/dispatcher-servlet.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
```

相信大家对于以上配置文件不会陌生，以我们之前的Servlet容器的初始化顺序我们来看一下以上配置文件。

- 首先，把contextConfigLocation这个参数以及它的值初始化到ServletContext中。
- 然后，初始化ContextLoaderListener加载contextConfigLocation里面配置的Spring配置文件，构建起root容器
- 最后，初始化DispatcherServlet加载它里面的contextConfigLocation，构造起web容器。再把root容器与web容器建立起父子关系。

下面我们以源代码的形式来说明以上的过程：

首先，我们先来看一下ContextLoaderListener类结构。

![这里写图片描述](http://static.iocoder.cn/csdn/20170805175220529?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQxMDczMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

可以看到ContextLoaderListener实现了ServletContextListener这个接口，而这个接口的里面有2个方法。

- contextInitialized() ： 容器初始化的时候调用。
- contextDestroyed() ： 容器销毁的时候调用。

所以在Servlet容器初始化的时候会调用contextInitialized方法。

![这里写图片描述](http://static.iocoder.cn/csdn/20170805175731105?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQxMDczMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

这里它会调用它的类父ContextLoader的initWebApplicationContext方法。

```Java
    public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
        try {
            if (this.context == null) {
                this.context = createWebApplicationContext(servletContext);
            }
            if (this.context instanceof ConfigurableWebApplicationContext) {
                ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
                if (!cwac.isActive()) {
                    if (cwac.getParent() == null) {
                        ApplicationContext parent = loadParentContext(servletContext);
                        cwac.setParent(parent);
                    }
                    configureAndRefreshWebApplicationContext(cwac, servletContext);
                }
            }
            servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

            ClassLoader ccl = Thread.currentThread().getContextClassLoader();
            if (ccl == ContextLoader.class.getClassLoader()) {
                currentContext = this.context;
            }
            else if (ccl != null) {
                currentContextPerThread.put(ccl, this.context);
            }

            return this.context;
        }
    }
```

## 3.1 通过反射创建root容器对象

在最开始它会通过反射的方式调用createWebApplicationContext()创建默认的配置在ContextLoader.properties里面的WebApplicationContext接口实例XmlWebApplicationContext。

```Java
protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
    // 检测ContextClass
    Class<?> contextClass = determineContextClass(sc);
    if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
        throw new ApplicationContextException("Custom context class [" + contextClass.getName() +
                "] is not of type [" + ConfigurableWebApplicationContext.class.getName() + "]");
    }
    return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

因为没有配置contextClass所以会加载配置在ContextLoader.properties里面的对象。

```Java
protected Class<?> determineContextClass(ServletContext servletContext) {
    String contextClassName = servletContext.getInitParameter(CONTEXT_CLASS_PARAM);
    if (contextClassName != null) {
        try {
            return ClassUtils.forName(contextClassName, ClassUtils.getDefaultClassLoader());
        }
        catch (ClassNotFoundException ex) {
            throw new ApplicationContextException(
                    "Failed to load custom context class [" + contextClassName + "]", ex);
        }
    }
    else {
        contextClassName = defaultStrategies.getProperty(WebApplicationContext.class.getName());
        try {
            return ClassUtils.forName(contextClassName, ContextLoader.class.getClassLoader());
        }
        catch (ClassNotFoundException ex) {
            throw new ApplicationContextException(
                    "Failed to load default context class [" + contextClassName + "]", ex);
        }
    }
}
```

## 3.2 加载相应配置文件，初始化root容器

然后调用configureAndRefreshWebApplicationContext方法获取到配置在web.xml里面的configLocationParam值。把它配置成Spring容器的资源位置，最后通过调用wac.refresh()对于整个Spring容器进行加载。

```Java
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
    if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
        String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
        if (idParam != null) {
            wac.setId(idParam);
        }
        else {
            wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
                    ObjectUtils.getDisplayString(sc.getContextPath()));
        }
    }

    wac.setServletContext(sc);

    // 获取contextConfigLocation值
    String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
    if (configLocationParam != null) {
        wac.setConfigLocation(configLocationParam);
    }

    ConfigurableEnvironment env = wac.getEnvironment();
    if (env instanceof ConfigurableWebEnvironment) {
        ((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
    }

    customizeContext(sc, wac);

    // 加载并刷新Spring容器
    wac.refresh();
}
```

## 3.3 把root容器放在ServletContext中

把初始化好的root 容器以WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE为key，放在Servlet容器当中，以备后面使用。

![这里写图片描述](http://static.iocoder.cn/csdn/20170805181940850?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQxMDczMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## 3.4 关联Servlet WebApplicationContext与 Root WebApplicationContext

我们知道servlet的规范中，servlet装载时会调用其init()方法，那么自然是调用DispatcherServlet的init方法，通过源码一看，竟然没有，但是并不带表真的没有，你会发现在父类的父类中：org.springframework.web.servlet.HttpServletBean有这个方法，如下图所示：

```Java
    public final void init() throws ServletException {
        if (logger.isDebugEnabled()) {
            logger.debug("Initializing servlet '" + getServletName() + "'");
        }

        // 把DispatcherServlet里面的init-param转化成PropertyValues
        PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
        if (!pvs.isEmpty()) {
            try {
                // 把DispatcherServlet转换成BeanWrapper,这样就把DispatcherServlet转换成了Spring中的Bean管理
                BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
                ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
                bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
                initBeanWrapper(bw);
                bw.setPropertyValues(pvs, true);
            }
            catch (BeansException ex) {
                if (logger.isErrorEnabled()) {
                    logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
                }
                throw ex;
            }
        }

        // Let subclasses do whatever initialization they like.
        initServletBean();

        if (logger.isDebugEnabled()) {
            logger.debug("Servlet '" + getServletName() + "' configured successfully");
        }
    }
```

注意代码：initServletBean(); 其余的都和加载bean关系并不是特别大，跟踪进去会发I发现这个方法是在类：org.springframework.web.servlet.FrameworkServlet类中（是DispatcherServlet的父类、HttpServletBean的子类），内部通过调用initWebApplicationContext()来初始化一个WebApplicationContext，源码片段（篇幅所限，不拷贝所有源码，仅仅截取片段）

![这里写图片描述](http://static.iocoder.cn/csdn/20170805182757752?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQxMDczMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

接下来需要知道的是如何初始化这个context的（按照使用习惯，其实只要得到了ApplicationContext，就得到了bean的信息，所以在初始化ApplicationCotext的时候，就已经初始化好了bean的信息，至少至少，它初始化好了bean的路径，以及描述信息），所以我们一旦知道ApplicationCotext是怎么初始化的，就基本知道bean是如何加载的了。

```Java
    // 获取到root 容器
    protected WebApplicationContext initWebApplicationContext() {
        WebApplicationContext rootContext =
                WebApplicationContextUtils.getWebApplicationContext(getServletContext());
        WebApplicationContext wac = null;

        if (this.webApplicationContext != null) {
            wac = this.webApplicationContext;
            if (wac instanceof ConfigurableWebApplicationContext) {
                ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
                if (!cwac.isActive()) {
                    if (cwac.getParent() == null) {
                        // 设置父容器
                        cwac.setParent(rootContext);
                    }
                    configureAndRefreshWebApplicationContext(cwac);
                }
            }
        }
        if (wac == null) {
            wac = findWebApplicationContext();
        }
        if (wac == null) {
            wac = createWebApplicationContext(rootContext);
        }

        if (!this.refreshEventReceived) {
            onRefresh(wac);
        }

        if (this.publishContext) {
            String attrName = getServletContextAttributeName();
            getServletContext().setAttribute(attrName, wac);
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Published WebApplicationContext of servlet '" + getServletName() +
                        "' as ServletContext attribute with name [" + attrName + "]");
            }
        }

        return wac;
    }
```

最后调用configureAndRefreshWebApplicationContext方法获取到DispatcherServlet里面的contextConfigLocation值，创建并这个Servlet WebApplicationContex.

## 3.5 初始化Spring MVC默认组件

DispatcherServlet的初始化主线的执行体系是顺着其继承结构依次进行的，我们在之前曾经讨论过它的执行次序。所以，只有在FrameworkServlet完成了对于WebApplicationContext和组件的初始化之后，执行权才被正式转移到DispatcherServlet中。我们可以来看看DispatcherServlet此时究竟干了哪些事：

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

onRefresh是FrameworkServlet中预留的扩展方法，在DispatcherServlet中做了一个基本实现：initStrategies。我们粗略一看，很容易就能明白DispatcherServlet到底在这里干些什么了：初始化组件。

读者或许会问，组件不是已经在WebApplicationContext初始化的时候已经被初始化过了嘛？这里所谓的组件初始化，指的又是什么呢？让我们来看看其中的一个方法的源码：

```Java
/**
 * Initialize the MultipartResolver used by this class.
 * <p>If no bean is defined with the given name in the BeanFactory for this namespace,
 * no multipart handling is provided.
 */
private void initMultipartResolver(ApplicationContext context) {
    try {
        this.multipartResolver = context.getBean(MULTIPART_RESOLVER_BEAN_NAME, MultipartResolver.class);
        if (logger.isDebugEnabled()) {
            logger.debug("Using MultipartResolver [" + this.multipartResolver + "]");
        }
    } catch (NoSuchBeanDefinitionException ex) {
        // Default is no multipart resolver.
        this.multipartResolver = null;
        if (logger.isDebugEnabled()) {
            logger.debug("Unable to locate MultipartResolver with name '" + MULTIPART_RESOLVER_BEAN_NAME +
                    "': no multipart request handling provided");
        }
    }
}
```

原来，这里的初始化，指的是DispatcherServlet从容器（WebApplicationContext）中读取组件的实现类，并缓存于DispatcherServlet内部的过程。还记得我们之前给出的DispatcherServlet的数据结构吗？这些位于DispatcherServlet内部的组件实际上只是一些来源于容器缓存实例，不过它们同样也是DispatcherServlet进行后续操作的基础。

到了这里，把Spring MVC与Servlet容器的关系，以及Spring MVC通过Servlet容器的初始化顺序创建父子容器，以及根据Servlet的生命周期的init()方法来初始化Spring MVC的默认组件展现了出来。

# 4、Spring MVC的处理HTTP请求

Spring MVC要处理http请求，它首先要解决3个问题。

- URL到框架的映射。
- http请求参数绑定
- http响应的生成和输出

## 4.1 URL到框架的映射

在Spring MVC我们只需要创建一个对象在上面标注@Controller注解。然后创建一个方法标注@RequestMapping然后这个方法就能够处理对应的url了。具体解析可以参看 – [Spring MVC @RequestMapping](http://blog.csdn.net/u012410733/article/details/51912375)

## 4.2 http请求参数绑定

对于http请求参数的绑定，在Spring MVC生成URL到框架的映射关系时会创建一个HandlerMethod对象。这个对象里面包括了这个url对应Spring对应标注了@RequestMapping方法的MethodParameter(方法参数)。通过HttpServletRequest获取到请求参数，然后再通过HandlerMethodArgumentResolver接口把请求参数绑定到方法参数中。

```Java
public interface HandlerMethodArgumentResolver {

    boolean supportsParameter(MethodParameter parameter);

    Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
            NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception;

}
```

具体可以参考 – [Spring MVC DataBinder](http://blog.csdn.net/u012410733/article/details/53368351)

## 4.3 http响应的生成和输出

在整个系统运行的过程中处于Servlet的service()方法都处于侦听模式，侦听并处理所有的Web请求。因此，在service及其相关方法中，我们看到的则是对Http请求的处理流程。

在Spring MVC容器在初始的时候创建了URL到框架的映射，当前端请求的到来的时候。Spring MVC会获取到对应的HandlerMethod对象,HandlerMethod包括里面的属性。

```Java
public class HandlerMethod {

    /** Logger that is available to subclasses */
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

其中最主要的属性是以下三个：

- bean ： @Controller对象实例。
- method ：@Controller对象实例标注了@RequestMapping的方法对象，也就是处理对应url的方法
- parameters ： @RequestMapping里面的方法参数。
  ’
  通过http请求参数绑定，把Request请求的参数绑定解析成对应的方法参数。然后通过反射:

```Java
method.invoke(bean, paramters);
```

完成对http请求的整个响应。 具体可以参见 – [Spring MVC DispatcherServlet](http://blog.csdn.net/u012410733/article/details/51920055)

# 666. 彩蛋

如果你对 Java 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)