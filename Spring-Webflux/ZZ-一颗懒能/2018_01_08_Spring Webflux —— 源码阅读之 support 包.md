title: Spring Webflux —— 源码阅读之 support 包
date: 2018-01-08
tag: 
categories: Spring Webflux
permalink: Spring-Webflux/lanneng/support
author: 一颗懒能
from_url: https://www.jianshu.com/p/ad3979266616
wechat_url: 

-------

摘要: 原创出处 https://www.jianshu.com/p/ad3979266616 「一颗懒能」欢迎转载，保留摘要，谢谢！

- [Spring WebFlux安装的支持类。](http://www.iocoder.cn/Spring-Webflux/lanneng/support/)
- [AbstractAnnotationConfigDispatcherHandlerInitializer](http://www.iocoder.cn/Spring-Webflux/lanneng/support/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

## Spring WebFlux安装的支持类。

## AbstractAnnotationConfigDispatcherHandlerInitializer

getConfigClasses（）需要具体实现类。更多的模板和定制方法由AbstractDispatcherHandlerInitializer提供。

```Java
public abstract class AbstractAnnotationConfigDispatcherHandlerInitializer
        extends AbstractDispatcherHandlerInitializer {

    /**
     * 创建要提供给DispatcherHandler的应用程序上下文。
     * 返回的上下文被委托给Spring的DispatcherHandler.DispatcherHandler（ApplicationContext）。因此，它通常包含控制
         * 器，视图解析器和其他Web相关的bean。
     * 该实现创建一个AnnotationConfigApplicationContext，为其提供由getConfigClasses（）返回的带注释的类。
     */
    @Override
    protected ApplicationContext createApplicationContext() {
        AnnotationConfigApplicationContext servletAppContext = new AnnotationConfigApplicationContext();
        Class<?>[] configClasses = getConfigClasses();
        if (!ObjectUtils.isEmpty(configClasses)) {
            servletAppContext.register(configClasses);
        }
        return servletAppContext;
    }

    /**
     * 为应用程序上下文指定@Configuration或者@Component类。
     * 返回dispatcher servlet应用程序上下文的配置类
     */
    protected abstract Class<?>[] getConfigClasses();

}
```

### AbstractDispatcherHandlerInitializer

WebApplicationInitializer实现的基类，在Servlet上下文中注册一个DispatcherHandler，并将其包装在一个ServletHttpHandlerAdapter中。

大多数应用程序应该考虑扩展Spring Java配置，AbstractAnnotationConfigDispatcherHandlerInitializer子类。

```Java
public abstract class AbstractDispatcherHandlerInitializer implements WebApplicationInitializer {

    /**
     * 默认的servlet名字. 可以通过覆盖{@link #getServletName}来自定义.
     */
    public static final String DEFAULT_SERVLET_NAME = "dispatcher-handler";

    /**
     * 默认的servlet映射。. 可以通过覆盖{@link #getServletMapping()}.来自定义
     */
    public static final String DEFAULT_SERVLET_MAPPING = "/";

        //配置给定的ServletContext（servlets, filters, listeners context-params and attributes ）来初始化此Web应用程序
    @Override
    public void onStartup(ServletContext servletContext) throws ServletException {
        registerDispatcherHandler(servletContext);
    }

    /**
     * 针对给定的servlet上下文注册一个DispatcherHandler。
     *此方法将创建一个DispatcherHandler，并使用从createApplicationContext（）返回的应用程序上下文对其进行初始化。创
         * 建的处理程序将被包装在一个ServletHttpHandlerAdapter servlet中，其名称由getServletName（）返回，将其映射到从
         *getServletMapping（）返回的模式。
     *进一步的自定义可以通过覆盖customizeRegistration（ServletRegistration.Dynamic）或
         *  createDispatcherHandler（ApplicationContext）来实现。
     */
    protected void registerDispatcherHandler(ServletContext servletContext) {
        String servletName = getServletName();
        Assert.hasLength(servletName, "getServletName() must not return empty or null");

        ApplicationContext applicationContext = createApplicationContext();
        Assert.notNull(applicationContext,
                "createApplicationContext() did not return an application " +
                "context for servlet [" + servletName + "]");

        refreshApplicationContext(applicationContext);
        registerCloseListener(servletContext, applicationContext);

        WebHandler dispatcherHandler = createDispatcherHandler(applicationContext);
        Assert.notNull(dispatcherHandler,
                "createDispatcherHandler() did not return a WebHandler for servlet [" + servletName + "]");

        ServletHttpHandlerAdapter handlerAdapter = createHandlerAdapter(dispatcherHandler);
        Assert.notNull(handlerAdapter,
                "createHttpHandler() did not return a ServletHttpHandlerAdapter for servlet [" + servletName + "]");

        ServletRegistration.Dynamic registration = servletContext.addServlet(servletName, handlerAdapter);
        Assert.notNull(registration,
                "Failed to register servlet with name '" + servletName + "'." +
                "Check if there is another servlet registered under the same name.");

        registration.setLoadOnStartup(1);
        registration.addMapping(getServletMapping());
        registration.setAsyncSupported(true);

        customizeRegistration(registration);
    }

    /**
     * Return the name under which the {@link ServletHttpHandlerAdapter} will be registered.
     * Defaults to {@link #DEFAULT_SERVLET_NAME}.
     * @see #registerDispatcherHandler(ServletContext)
     */
    protected String getServletName() {
        return DEFAULT_SERVLET_NAME;
    }

    /**
     * 创建要提供给DispatcherHandler的应用程序上下文。
     * 返回的上下文被委托给Spring的DispatcherHandler.DispatcherHandler（ApplicationContext）。因此，它通常包含控制
         * 器，视图解析器和其他Web相关的bean。
     */
    protected abstract ApplicationContext createApplicationContext();

    /**
     * 如有必要，刷新给定的应用程序上下文。
     */
    protected void refreshApplicationContext(ApplicationContext context) {
        if (context instanceof ConfigurableApplicationContext) {
            ConfigurableApplicationContext cac = (ConfigurableApplicationContext) context;
            if (!cac.isActive()) {
                cac.refresh();
            }
        }
    }

    /**
     * 使用指定的ApplicationContext创建DispatcherHandler（或其他类型的WebHandler派生调度程序）。
     */
    protected WebHandler createDispatcherHandler(ApplicationContext applicationContext) {
        return new DispatcherHandler(applicationContext);
    }

    /**
     *创建一个ServletHttpHandlerAdapter。 默认实现返回一个ServletHttpHandlerAdapter和提供的webHandler。
     */
    protected ServletHttpHandlerAdapter createHandlerAdapter(WebHandler webHandler) {
        HttpHandler httpHandler = new HttpWebHandlerAdapter(webHandler);
        return new ServletHttpHandlerAdapter(httpHandler);
    }

    /**
     * Specify the servlet mapping for the {@code ServletHttpHandlerAdapter}.
     * <p>Default implementation returns {@code /}.
     * @see #registerDispatcherHandler(ServletContext)
     */
    protected String getServletMapping() {
        return DEFAULT_SERVLET_MAPPING;
    }

    /**
     *一旦registerDispatcherHandler（ServletContext）完成，可以选择执行进一步的注册自定义配置定制。
     *
     */
    protected void customizeRegistration(ServletRegistration.Dynamic registration) {
    }

    /**
     * Register a {@link ServletContextListener} that closes the given application context
     */
    protected void registerCloseListener(ServletContext servletContext, ApplicationContext applicationContext) {
        if (applicationContext instanceof ConfigurableApplicationContext) {
            ConfigurableApplicationContext context = (ConfigurableApplicationContext) applicationContext;
            ServletContextDestroyedListener listener = new ServletContextDestroyedListener(context);
            servletContext.addListener(listener);
        }
    }


    private static class ServletContextDestroyedListener implements ServletContextListener {

        private final ConfigurableApplicationContext applicationContext;

        public ServletContextDestroyedListener(ConfigurableApplicationContext applicationContext) {
            this.applicationContext = applicationContext;
        }

        @Override
        public void contextInitialized(ServletContextEvent sce) {
        }

        @Override
        public void contextDestroyed(ServletContextEvent sce) {
            this.applicationContext.close();
        }
    }
```

### AbstractServletHttpHandlerAdapterInitializer

在Servlet上下文中注册ServletHttpHandlerAdapter的WebApplicationInitializer实现的基类。

# 666. 彩蛋

如果你对 Spring Webflux 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)