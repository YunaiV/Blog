title: 【zhisheng】看透 Spring MVC 源代码分析与实践 —— 俯视 Spring MVC
date: 2018-01-02
tag: 
categories: Spring-MVC
permalink: Spring-MVC/zhisheng/Spring-MVC02/
author: zhisheng
from_url: http://www.54tianzhisheng.cn/2017/07/14/Spring-MVC02/
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484187&idx=1&sn=4ddf541e7f84aeca1934403957eab61b&chksm=fa497caacd3ef5bc8a70be11220069e92a7131476382fa4316834a6c7a6f6b16b87902bdaf0c&scene=21#wechat_redirect

-------

摘要: 原创出处 http://www.54tianzhisheng.cn/2017/07/14/Spring-MVC02/ 「zhisheng」欢迎转载，保留摘要，谢谢！

- [Spring MVC 之初体验](http://www.iocoder.cn/Spring-MVC/laotian/)
  - [环境搭建](http://www.iocoder.cn/Spring-MVC/laotian/)
  - [SPRING MVC 简单配置](http://www.iocoder.cn/Spring-MVC/laotian/)
- [创建 Spring MVC 之器](http://www.iocoder.cn/Spring-MVC/laotian/)
  - [小结](http://www.iocoder.cn/Spring-MVC/laotian/)
- [Spring MVC 之用](http://www.iocoder.cn/Spring-MVC/laotian/)
  - [小结](http://www.iocoder.cn/Spring-MVC/laotian/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

![spring-mvc](http://zhisheng.iocoder.cn/spring-mvc.png)

### Spring MVC 之初体验

#### 环境搭建

在 IDEA 中新建一个 web 项目，用 Maven 管理项目的话，在 pom.xml 中加入 Spring MVC 和 Servlet 依赖即可。

```XML
<!-- https://mvnrepository.com/artifact/org.springframework/spring-webmvc -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webmvc</artifactId>
    <version>4.3.9.RELEASE</version>
</dependency>
<!-- https://mvnrepository.com/artifact/javax.servlet/javax.servlet-api -->
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>3.1.0</version>
    <scope>provided</scope>
</dependency>
```

#### SPRING MVC 简单配置

- 在 web.xml 中配置 Servlet
- 创建 Spring MVC 的 xml 配置文件
- 创建 Controller 和 View

1、web.xml

```XML
<!-- Spring MVC配置 -->
<!-- ====================================== -->
<servlet>
    <servlet-name>spring</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <!-- 可以自定义servlet.xml配置文件的位置和名称，默认为WEB-INF目录下，名称为[<servlet-name>]-servlet.xml，如spring-servlet.xml
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/spring-servlet.xml</param-value>&nbsp; 默认
    </init-param>
    -->
    <load-on-startup>1</load-on-startup>
</servlet>

<servlet-mapping>
    <servlet-name>spring</servlet-name>
    <url-pattern>*.do</url-pattern>
</servlet-mapping>


<!-- Spring配置 -->
<!-- ====================================== -->
<listener>
   <listenerclass>
     org.springframework.web.context.ContextLoaderListener
   </listener-class>
</listener>


<!-- 指定Spring Bean的配置文件所在目录。默认配置在WEB-INF目录下 -->
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:config/applicationContext.xml</param-value>
</context-param>
```

2、spring-servlet.xml

```XML
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
        xmlns:context="http://www.springframework.org/schema/context"
   xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
       http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-3.0.xsd
       http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-3.0.xsd
       http://www.springframework.org/schema/context <a href="http://www.springframework.org/schema/context/spring-context-3.0.xsd">http://www.springframework.org/schema/context/spring-context-3.0.xsd</a>">

    <!-- 启用spring mvc 注解 -->
    <context:annotation-config />

    <!-- 设置使用注解的类所在的jar包 -->
    <context:component-scan base-package="controller"></context:component-scan>

    <!-- 完成请求和注解POJO的映射 -->
    <bean class="org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter" />
　　
    <!-- 对转向页面的路径解析。prefix：前缀， suffix：后缀 -->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver" p:prefix="/jsp/" p:suffix=".jsp" />
</beans>
```

3、Controller

```Java
package controller;

import javax.servlet.http.HttpServletRequest;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;

import entity.User;

@Controller  //类似Struts的Action
public class TestController {

    @RequestMapping("/test/login.do")  // 请求url地址映射，类似Struts的action-mapping
    public String testLogin(@RequestParam(value="username")String username, String password, HttpServletRequest request) {
        // @RequestParam是指请求url地址映射中必须含有的参数(除非属性 required=false, 默认为 true)
        // @RequestParam可简写为：@RequestParam("username")

        if (!"admin".equals(username) || !"admin".equals(password)) {
            return "loginError"; // 跳转页面路径（默认为转发），该路径不需要包含spring-servlet配置文件中配置的前缀和后缀
        }
        return "loginSuccess";
    }

    @RequestMapping("/test/login2.do")
    public ModelAndView testLogin2(String username, String password, int age){
        // request和response不必非要出现在方法中，如果用不上的话可以去掉
        // 参数的名称是与页面控件的name相匹配，参数类型会自动被转换

        if (!"admin".equals(username) || !"admin".equals(password) || age < 5) {
            return new ModelAndView("loginError"); // 手动实例化ModelAndView完成跳转页面（转发），效果等同于上面的方法返回字符串
        }
        return new ModelAndView(new RedirectView("../index.jsp"));  // 采用重定向方式跳转页面
        // 重定向还有一种简单写法
        // return new ModelAndView("redirect:../index.jsp");
    }

    @RequestMapping("/test/login3.do")
    public ModelAndView testLogin3(User user) {
        // 同样支持参数为表单对象，类似于Struts的ActionForm，User不需要任何配置，直接写即可
        String username = user.getUsername();
        String password = user.getPassword();
        int age = user.getAge();

        if (!"admin".equals(username) || !"admin".equals(password) || age < 5) {
            return new ModelAndView("loginError");
        }
        return new ModelAndView("loginSuccess");
    }

    @Resource(name = "loginService")  // 获取applicationContext.xml中bean的id为loginService的，并注入
    private LoginService loginService;  //等价于spring传统注入方式写get和set方法，这样的好处是简洁工整，省去了不必要得代码

    @RequestMapping("/test/login4.do")
    public String testLogin4(User user) {
        if (loginService.login(user) == false) {
            return "loginError";
        }
        return "loginSuccess";
    }
}
```

@RequestMapping 可以写在方法上，也可以写在类上，上面代码方法上的 RequestMapping 都含有 `/test` ， 那么我们就可以将其抽出直接写在类上，那么方法里面就不需要写 `/test` 了。

如下即可：

```Java
@Controller
@RequestMapping("/test")
public class TestController {

    @RequestMapping("/login.do")  // 请求url地址映射，类似Struts的action-mapping
    public String testLogin(@RequestParam(value="username")String username, String password, HttpServletRequest request) {
        // @RequestParam是指请求url地址映射中必须含有的参数(除非属性 required=false, 默认为 true)
        // @RequestParam可简写为：@RequestParam("username")

        if (!"admin".equals(username) || !"admin".equals(password)) {
            return "loginError"; // 跳转页面路径（默认为转发），该路径不需要包含spring-servlet配置文件中配置的前缀和后缀
        }
        return "loginSuccess";
    }
    //省略其他的
}
```

上面的代码方法的参数中可以看到有一个 `@RequestParam` 注解，其实还有 `@PathVariable` 。这两个的区别是啥呢？

- `@PathVariable` 标记在方法的参数上，利用它标记的参数可以利用请求路径传值。
- @RequestParam是指请求url地址映射中必须含有的参数(除非属性 required=false, 默认为 true)

看如下例子：

```Java
@RequestMapping("/user/{userId}")  // 请求url地址映射
public String userinfo(Model model, @PathVariable("userId") int userId,  HttpSession session) {
         System.out.println("进入  userinfo  页面");
        //判断是否有用户登录
        User user1 = (User) session.getAttribute("user");
        if (user1 == null) {
            return "login";
        }
        User user = userService.selectUserById(userId);
        model.addAttribute("user", user);
        return "userinfo";
    }
```

上面例子中如果浏览器请求的是 `/user/1` 的时候，就表示此时的用户 id 为 1，此时就会先从 session 中查找是否有 “user” 属性，如果有的话，就代表用户此时处于登录的状态，如果没有的话，就会让用户返回到登录页面，这种机制在各种网站经常会使用的，然后根据这个 id = 1 ，去查找用户的信息，然后把查找的 “user” 放在 model 中，然后返回用户详情页面，最后在页面中用 `$!{user.name}` 获取用户的名字，同样的方式可以获取用户的其他信息，把所有的用户详情信息展示出来。

### 创建 Spring MVC 之器

Spring MVC 核心 Servlet 架构图如下：

![spring-mvc1](http://zhisheng.iocoder.cn/spring-mvc1.jpg)

Java 中常用的 Servlet 我在另外一篇文章写的很清楚了，有兴趣的请看：[通过源码详解 Servlet](http://www.54tianzhisheng.cn/2017/07/09/servlet/) ，这里我就不再解释了。

这里主要讲 Spring 中的 HttpServletBean、FrameworkServlet、DispatcherServlet 这三个类的创建过程。

通过上面的图，可以看到这三个类直接实现三个接口：EnvironmentCapable、EnvironmentAware、ApplicationContextAware。下面我们直接看下这三个接口的内部是怎样写的。

EnvironmentCapable.java

```Java
public interface EnvironmentCapable {
    //返回组件的环境，可能返回 null 或者默认环境
    @Nullable
    Environment getEnvironment();
}
```

EnvironmentAware.java

```Java
public interface EnvironmentAware extends Aware {
    //设置组件的运行环境
    void setEnvironment(Environment environment);
}
```

ApplicationContextAware.java

```Java
public interface ApplicationContextAware extends Aware {
    //设置运行对象的应用上下文
    //当类实现这个接口后，这个类可以获取ApplicationContext中所有的bean，也就是说这个类可以直接获取Spring配置文件中所有有引用到的bean对象
    void setApplicationContext(ApplicationContext applicationContext) throws BeansException;
}
```

怎么使用这个这个接口呢？

参考文章：[org.springframework.context.ApplicationContextAware使用理解](http://blog.csdn.net/kaiwii/article/details/6872642)

**HttpServletBean**

这里就直接看其中最重要的 init() 方法的代码了：

```Java
/**
 * 将配置参数映射到此servlet的bean属性，并调用子类初始化。
 *  如果 bean 配置不合法（或者需要的参数丢失）或者子类初始化发生错误，那么就会抛出 ServletException 异常
 */
@Override
public final void init() throws ServletException {
   //日志代码删除了

   // 从init参数设置bean属性。
   //获得web.xml中的contextConfigLocation配置属性，就是spring MVC的配置文件
   PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
   if (!pvs.isEmpty()) {
      try {
         BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
         //获取服务器的各种信息
         ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
         bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
         //模板方法，可以在子类中调用，做一些初始化工作，bw代表DispatcherServelt
         initBeanWrapper(bw);
         //将配置的初始化值设置到DispatcherServlet中
         bw.setPropertyValues(pvs, true);
      }
      catch (BeansException ex) {
         //日志代码
         throw ex;
      }
   }

   // Let subclasses do whatever initialization they like.
   //模板方法，子类初始化的入口方法
   initServletBean();

   //日志代码删除了
}
```

**FrameworkServlet**

其中重要方法如下：里面也就两句关键代码，日志代码我直接删掉了

```Java
protected final void initServletBean() throws ServletException {
        //日志代码删除了
        long startTime = System.currentTimeMillis();

        //就是 try 语句里面有两句关键代码
        try {
            //初始化 webApplicationContext
            this.webApplicationContext = initWebApplicationContext();
            //模板方法，
            initFrameworkServlet();
        }
        catch (ServletException ex) {
            this.logger.error("Context initialization failed", ex);
            throw ex;
        }
        catch (RuntimeException ex) {
            this.logger.error("Context initialization failed", ex);
            throw ex;
        }

        //日志代码删除了
    }
```

再来看看上面代码中调用的 initWebApplicationContext() 方法

```Java
protected WebApplicationContext initWebApplicationContext() {
        //获取 rootContext
        WebApplicationContext rootContext =
                WebApplicationContextUtils.getWebApplicationContext(getServletContext());
        WebApplicationContext wac = null;

        if (this.webApplicationContext != null) {
            // 上下文实例在构造时注入 - >使用它
            wac = this.webApplicationContext;
            if (wac instanceof ConfigurableWebApplicationContext) {
                ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
                if (!cwac.isActive()) {
                // 如果上下文尚未刷新 -> 提供诸如设置父上下文，设置应用程序上下文ID等服务
                    if (cwac.getParent() == null) {
            // 上下文实例被注入没有显式的父类 -> 将根应用程序上下文（如果有的话可能为null）设置为父级
                        cwac.setParent(rootContext);
                    }
                    configureAndRefreshWebApplicationContext(cwac);
                }
            }
        }
        if (wac == null) {
    // 当 WebApplicationContext 已经存在 ServletContext 中时，通过配置在 servlet 中的 ContextAttribute 参数获取
            wac = findWebApplicationContext();
        }
        if (wac == null) {
            // 如果 WebApplicationContext 还没有创建，则创建一个
            wac = createWebApplicationContext(rootContext);
        }
        if (!this.refreshEventReceived) {
            // 当 ContextRefreshedEvent 事件没有触发时调用此方法，模板方法，可以在子类重写
            onRefresh(wac);
        }
        if (this.publishContext) {
            // 将 ApplicationContext 保存到 ServletContext 中去
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

initWebApplicationContext 方法做了三件事：

- 获取 Spring 的根容器 rootContext
- 设置 webApplicationContext 并根据情况调用 onRefresh 方法
- 将 webApplicationContext 设置到 ServletContext 中

这里在讲讲上面代码中的 wac == null 的几种情况：

1）、当 WebApplicationContext 已经存在 ServletContext 中时，通过配置在 servlet 中的 ContextAttribute 参数获取，调用的是 findWebApplicationContext() 方法

```Java
protected WebApplicationContext findWebApplicationContext() {
        String attrName = getContextAttribute();
        if (attrName == null) {
            return null;
        }
        WebApplicationContext wac =
                WebApplicationContextUtils.getWebApplicationContext(getServletContext(), attrName);
        if (wac == null) {
            throw new IllegalStateException("No WebApplicationContext found: initializer not registered?");
        }
        return wac;
    }
```

2)、如果 WebApplicationContext 还没有创建，调用的是 createWebApplicationContext 方法

```Java
protected WebApplicationContext createWebApplicationContext(@Nullable ApplicationContext parent) {
        //获取创建类型
        Class<?> contextClass = getContextClass();
        //删除了打印日志代码

        //检查创建类型
        if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
            throw new ApplicationContextException(
                    "Fatal initialization error in servlet with name '" + getServletName() +
                    "': custom WebApplicationContext class [" + contextClass.getName() +
                    "] is not of type ConfigurableWebApplicationContext");
        }
        //具体创建
        ConfigurableWebApplicationContext wac =
                (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);

        wac.setEnvironment(getEnvironment());
        wac.setParent(parent);
  //并设置的 contextConfigLocation 参数传给 wac，默认是 WEB-INFO/[ServletName]-Servlet.xml
        wac.setConfigLocation(getContextConfigLocation());

        //调用的是下面的方法
        configureAndRefreshWebApplicationContext(wac);

        return wac;
    }

protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac) {
        if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
            // The application context id is still set to its original default value
            // -> assign a more useful id based on available information
            if (this.contextId != null) {
                wac.setId(this.contextId);
            }
            else {
                // Generate default id...
                wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
                        ObjectUtils.getDisplayString(getServletContext().getContextPath()) + '/' + getServletName());
            }
        }

        wac.setServletContext(getServletContext());
        wac.setServletConfig(getServletConfig());
        wac.setNamespace(getNamespace());
        wac.addApplicationListener(new SourceFilteringListener(wac, new ContextRefreshListener()));

        // The wac environment's #initPropertySources will be called in any case when the context
        // is refreshed; do it eagerly here to ensure servlet property sources are in place for
        // use in any post-processing or initialization that occurs below prior to #refresh
        ConfigurableEnvironment env = wac.getEnvironment();
        if (env instanceof ConfigurableWebEnvironment) {
            ((ConfigurableWebEnvironment) env).initPropertySources(getServletContext(), getServletConfig());
        }

        postProcessWebApplicationContext(wac);
        applyInitializers(wac);
        wac.refresh();
    }
```

里面还有 doXXX() 方法，大家感兴趣的可以去看看。

**DispatcherServlet**

DispatcherServlet 继承自 FrameworkServlet，onRefresh 方法是 DispatcherServlet 的入口方法，在 initStrategies 方法中调用了 9 个初始化的方法。

![onrefresh](http://zhisheng.iocoder.cn/onrefresh.jpg)

这里分析其中一个初始化方法：initLocaleResolver() 方法

```Java
private void initLocaleResolver(ApplicationContext context) {
        try {
            //在 context 中获取
            this.localeResolver = context.getBean(LOCALE_RESOLVER_BEAN_NAME, LocaleResolver.class);
            //删除了打印日志的代码
        }
        catch (NoSuchBeanDefinitionException ex) {
            //使用默认的策略
            this.localeResolver = getDefaultStrategy(context, LocaleResolver.class);
            //删除了打印日志的代码
        }
    }
```

查看默认策略代码：

```Java
protected <T> T getDefaultStrategy(ApplicationContext context, Class<T> strategyInterface) {
        //调用 getDefaultStrategies 方法
        List<T> strategies = getDefaultStrategies(context, strategyInterface);
        if (strategies.size() != 1) {
            throw new BeanInitializationException(
                    "DispatcherServlet needs exactly 1 strategy for interface [" + strategyInterface.getName() + "]");
        }
        return strategies.get(0);
    }

    /**
     * Create a List of default strategy objects for the given strategy interface.
     * <p>The default implementation uses the "DispatcherServlet.properties" file (in the same
     * package as the DispatcherServlet class) to determine the class names. It instantiates
     * the strategy objects through the context's BeanFactory.
     */
    @SuppressWarnings("unchecked")
    protected <T> List<T> getDefaultStrategies(ApplicationContext context, Class<T> strategyInterface) {
        String key = strategyInterface.getName();
        //根据策略接口的名字从 defaultStrategies 获取所需策略的类型
        String value = defaultStrategies.getProperty(key);
        if (value != null) {
            //如果有多个默认值的话，就以逗号分隔为数组
            String[] classNames = StringUtils.commaDelimitedListToStringArray(value);
            List<T> strategies = new ArrayList<>(classNames.length);
            //按获取到的类型初始化策略
            for (String className : classNames) {
                try {
                    Class<?> clazz = ClassUtils.forName(className, DispatcherServlet.class.getClassLoader());
                    Object strategy = createDefaultStrategy(context, clazz);
                    strategies.add((T) strategy);
                }
                catch (ClassNotFoundException ex) {
                    throw new BeanInitializationException(
                            "Could not find DispatcherServlet's default strategy class [" + className + "] for interface [" + key + "]", ex);
                }
                catch (LinkageError err) {
                    throw new BeanInitializationException(
                            "Error loading DispatcherServlet's default strategy class [" + className + "] for interface [" + key + "]: problem with class file or dependent class", err);
                }
            }
            return strategies;
        }
        else {
            return new LinkedList<>();
        }
    }
```

其他几个方法大概也类似，我就不再写了。

#### 小结

主要讲了 Spring MVC 自身创建过程，分析了 Spring MVC 中 Servlet 的三个层次：HttpServletBean、FrameworkServlet 和 DispatcherServlet。HttpServletBean 继承自 Java 的 HttpServlet，其作用是将配置的参数设置到相应的属性上；FrameworkServlet 初始化了 WebApplicationContext；DispatcherServlet 初始化了自身的 9 个组件。

### Spring MVC 之用

分析 Spring MVC 是怎么处理请求的。首先分析 HttpServletBean、FrameworkServlet 和 DispatcherServlet 这三个 Servlet 的处理过程，最后分析 doDispatcher 的结构。

**HttpServletBean**

参与了创建工作，并没有涉及请求的处理。

**FrameworkServlet**

在类中的 service() 、doGet()、doPost()、doPut()、doDelete()、doOptions()、doTrace() 这些方法中可以看到都调用了一个共同的方法 processRequest() ，它是类在处理请求中最核心的方法。

```Java
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {

        long startTime = System.currentTimeMillis();
        Throwable failureCause = null;
        //获取 LocaleContextHolder 中原来保存的 LocaleContext
        LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
        //获取当前请求的 LocaleContext
        LocaleContext localeContext = buildLocaleContext(request);
        //获取 RequestContextHolder 中原来保存的 RequestAttributes
        RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
        //获取当前请求的 ServletRequestAttributes
        ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);

        WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
        asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), new RequestBindingInterceptor());
        //将当前请求的 LocaleContext 和 ServletRequestAttributes 设置到 LocaleContextHolder 和 RequestContextHolder
        initContextHolders(request, localeContext, requestAttributes);

        try {
            //实际处理请求的入口，这是一个模板方法，在 Dispatcher 类中才有具体实现
            doService(request, response);
        }catch (ServletException ex) {
            failureCause = ex;
            throw ex;
        }catch (IOException ex) {
            failureCause = ex;
            throw ex;
        }catch (Throwable ex) {
            failureCause = ex;
            throw new NestedServletException("Request processing failed", ex);
        }finally {
            //将 previousLocaleContext，previousAttributes 恢复到 LocaleContextHolder 和 RequestContextHolder 中
            resetContextHolders(request, previousLocaleContext, previousAttributes);
            if (requestAttributes != null) {
                requestAttributes.requestCompleted();
            }
            //删除了日志打印代码
            //发布了一个 ServletRequestHandledEvent 类型的消息
            publishRequestHandledEvent(request, response, startTime, failureCause);
        }
    }
```

**DispatcherServlet**

上一章中其实还没把该类讲清楚，在这个类中，里面的智行处理的入口方法应该是 doService 方法，方法里面调用了 doDispatch 进行具体的处理，在调用 doDispatch 方法之前 doService 做了一些事情：首先判断是不是 include 请求，如果是则对 request 的 Attribute 做个快照备份，等 doDispatcher 处理完之后（如果不是异步调用且未完成）进行还原 ，在做完快照后又对 request 设置了一些属性。

```Java
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
        // Keep a snapshot of the request attributes in case of an include,
        // to be able to restore the original attributes after the include.
        Map<String, Object> attributesSnapshot = null;
        if (WebUtils.isIncludeRequest(request)) {
            attributesSnapshot = new HashMap<>();
            Enumeration<?> attrNames = request.getAttributeNames();
            while (attrNames.hasMoreElements()) {
                String attrName = (String) attrNames.nextElement();
                if (this.cleanupAfterInclude || attrName.startsWith(DEFAULT_STRATEGIES_PREFIX)){
                    attributesSnapshot.put(attrName, request.getAttribute(attrName));
                }
            }
        }
        // Make framework objects available to handlers and view objects.
        request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
        request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
        request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
        request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());

        FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
        if (inputFlashMap != null) {
            request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
        }
        request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
        request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);

        try {
            //调用 doDispatch 方法
            doDispatch(request, response);
        }finally {
            if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
                // Restore the original attribute snapshot, in case of an include.
                if (attributesSnapshot != null) {
                    restoreAttributesAfterInclude(request, attributesSnapshot);
                }
            }
        }
    }
```

doDispatch() 方法：

```Java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
        HttpServletRequest processedRequest = request;
        HandlerExecutionChain mappedHandler = null;
        boolean multipartRequestParsed = false;

        WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
        try {
            ModelAndView mv = null;
            Exception dispatchException = null;
            try {
                //检查是不是上传请求
                processedRequest = checkMultipart(request);
                multipartRequestParsed = (processedRequest != request);

                // Determine handler for the current request.  根据 request 找到 Handler
                mappedHandler = getHandler(processedRequest);
                if (mappedHandler == null || mappedHandler.getHandler() == null) {
                    noHandlerFound(processedRequest, response);
                    return;
                }

    // Determine handler adapter for the current request.根据 Handler 找到对应的 HandlerAdapter
                HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
                // Process last-modified header, if supported by the handler.
                //处理 GET 、 HEAD 请求的 LastModified
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
                //执行相应的 Interceptor 的 preHandle
                if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                    return;
                }
                // Actually invoke the handler. HandlerAdapter 使用 Handler 处理请求
                mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
                //如果需要异步处理，直接返回
                if (asyncManager.isConcurrentHandlingStarted()) {
                    return;
                }
                //当 view 为空时，根据 request 设置默认 view
                applyDefaultViewName(processedRequest, mv);
                //执行相应 Interceptor 的 postHandler
                mappedHandler.applyPostHandle(processedRequest, response, mv);
            }catch (Exception ex) {
                dispatchException = ex;
            }catch (Throwable err) {
                // As of 4.3, we're processing Errors thrown from handler methods as well,
                // making them available for @ExceptionHandler methods and other scenarios.
                dispatchException = new NestedServletException("Handler dispatch failed", err);
            }
            //调用 processDispatchResult 方法处理上面处理之后的结果（包括处理异常，渲染页面，发出完成通知触发 Interceptor 的 afterCompletion）
            processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
        }catch (Exception ex) {
            triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
        }catch (Throwable err) {
            triggerAfterCompletion(processedRequest, response, mappedHandler,
                    new NestedServletException("Handler processing failed", err));
        }finally {
          //判断是否执行异步请求
            if (asyncManager.isConcurrentHandlingStarted()) {
                // Instead of postHandle and afterCompletion
                if (mappedHandler != null) {
                    mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
                }
            }else {
                // Clean up any resources used by a multipart request. 删除上传请求的资源
                if (multipartRequestParsed) {
                    cleanupMultipart(processedRequest);
                }
            }
        }
    }
```

Handler，HandlerMapping，HandlerAdapter 三个区别：

- Handler：处理器，对应 MVC 的 C层，也就是 Controller 层，具体表现形式有很多种，可以是类，方法，它的类型是 Object，只要可以处理实际请求就可以是 Handler。
- HandlerMapping：用来查找 Handler 的。
- HandlerAdapter ：Handler 适配器，

另外 View 和 ViewResolver 的原理与 Handler 和 HandlerMapping 的原理类似。

![doDispatcher](http://zhisheng.iocoder.cn/doDispatcher.png)

#### 小结

本章分析了 Spring MVC 的请求处理的过程。

# 666. 彩蛋

如果你对 Java 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)