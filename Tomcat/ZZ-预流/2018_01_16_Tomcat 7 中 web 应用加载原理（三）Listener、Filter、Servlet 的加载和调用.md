title: Tomcat 7 中 web 应用加载原理（三）Listener、Filter、Servlet 的加载和调用
date: 2018-01-16
tag: 
categories: Tomcat
permalink: Tomcat/yuliu/Web-application-loading-principle-3-Listener-Filter-Servlet
author: 预流
from_url: https://juejin.im/post/5a7ceeabf265da4e9449a802
wechat_url: 

-------

摘要: 原创出处 https://juejin.im/post/5a7ceeabf265da4e9449a802 「预流」欢迎转载，保留摘要，谢谢！


-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

[前一篇文章](http://www.iocoder.cn/Tomcat/yuliu/Web-application-loading-principle-2-web-xml-parsing)分析到了`org.apache.catalina.deploy.WebXml`类的 configureContext 方法，可以看到在这个方法中通过各种 setXXX、addXXX 方法的调用，使得每个应用中的 web.xml 文件的解析后将应用内部的表示 Servlet、Listener、Filter 的配置信息与表示一个 web 应用的 Context 对象关联起来。

这里列出 configureContext 方法中与 Servlet、Listener、Filter 的配置信息设置相关的调用代码：

```Java
for (FilterDef filter : filters.values()) {
    if (filter.getAsyncSupported() == null) {
        filter.setAsyncSupported("false");
    }
    context.addFilterDef(filter);
}
for (FilterMap filterMap : filterMaps) {
    context.addFilterMap(filterMap);
}

```

这是设置 Filter 相关配置信息的。

```Java
for (String listener : listeners) {
    context.addApplicationListener(
            new ApplicationListener(listener, false));
}

```

这是给应用添加 Listener 的。

```Java
for (ServletDef servlet : servlets.values()) {
    Wrapper wrapper = context.createWrapper();
    // Description is ignored
    // Display name is ignored
    // Icons are ignored

    // jsp-file gets passed to the JSP Servlet as an init-param

    if (servlet.getLoadOnStartup() != null) {
        wrapper.setLoadOnStartup(servlet.getLoadOnStartup().intValue());
    }
    if (servlet.getEnabled() != null) {
        wrapper.setEnabled(servlet.getEnabled().booleanValue());
    }
    wrapper.setName(servlet.getServletName());
    Map<String,String> params = servlet.getParameterMap();
    for (Entry<String, String> entry : params.entrySet()) {
        wrapper.addInitParameter(entry.getKey(), entry.getValue());
    }
    wrapper.setRunAs(servlet.getRunAs());
    Set<SecurityRoleRef> roleRefs = servlet.getSecurityRoleRefs();
    for (SecurityRoleRef roleRef : roleRefs) {
        wrapper.addSecurityReference(
                roleRef.getName(), roleRef.getLink());
    }
    wrapper.setServletClass(servlet.getServletClass());
    MultipartDef multipartdef = servlet.getMultipartDef();
    if (multipartdef != null) {
        if (multipartdef.getMaxFileSize() != null &&
                multipartdef.getMaxRequestSize()!= null &&
                multipartdef.getFileSizeThreshold() != null) {
            wrapper.setMultipartConfigElement(new MultipartConfigElement(
                    multipartdef.getLocation(),
                    Long.parseLong(multipartdef.getMaxFileSize()),
                    Long.parseLong(multipartdef.getMaxRequestSize()),
                    Integer.parseInt(
                            multipartdef.getFileSizeThreshold())));
        } else {
            wrapper.setMultipartConfigElement(new MultipartConfigElement(
                    multipartdef.getLocation()));
        }
    }
    if (servlet.getAsyncSupported() != null) {
        wrapper.setAsyncSupported(
                servlet.getAsyncSupported().booleanValue());
    }
    wrapper.setOverridable(servlet.isOverridable());
    context.addChild(wrapper);
}
for (Entry<String, String> entry : servletMappings.entrySet()) {
    context.addServletMapping(entry.getKey(), entry.getValue());
}

```

这段代码是设置 Servlet 的相关配置信息的。

以上是在各个 web 应用的 web.xml 文件中（如果是 servlet 3，还会包括将这些配置信息放在类的注解中，所以解析 web.xml 文件之前可能会存在各个 web.xml 文件信息的合并步骤，这些动作的代码在[前一篇文章](https://link.juejin.im?target=https%3A%2F%2Fjuejin.im%2Fpost%2F5a7aa6f4f265da4e7e10a0aa)中讲 ContextConfig 类的 webConfig 方法中）的相关配置信息的设置，但需要注意的是，这里仅仅是将这些配置信息保存到了 StandardContext 的相应实例变量中，真正在一次请求访问中用到的 Servlet、Listener、Filter 的实例并没有构造出来，以上方法调用仅构造了代表这些实例的封装类的实例，如 StandardWrapper、ApplicationListener、FilterDef、FilterMap。

那么一个 web 应用中的 Servlet、Listener、Filter 的实例究竟在什么时候构造出来的呢？答案在`org.apache.catalina.core.StandardContext`类的 startInternal 方法中：

![img](https://user-gold-cdn.xitu.io/2018/2/9/161780960e8d4a70?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/9/1617809eefaed912?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/9/161780aa6341aafa?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/9/161780b4b90d3ace?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/9/161780bb9c84f457?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/9/16178175e0a58b8e?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/9/1617817de64e52b4?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/9/16178184bac41c5d?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 这 303 行可以讲的东西有很多，为了不偏离本文主题只抽出与现在要讨论的问题相关的代码来分析。



第 125 行会发布一个`CONFIGURE_START_EVENT`事件，按前一篇博文所述，这里即会触发对 web.xml 的解析。第 205、206 行设置实例管理器为 DefaultInstanceManager（这个类在后面谈实例构造时会用到）。第 237 行会调用 listenerStart 方法，第 255 行调用了 filterStart 方法，第 263 行调用了 loadOnStartup 方法，这三处调用即触发 Listener、Filter、Servlet 真正对象的构造，下面逐个分析这些方法。

listenerStart 方法的完整代码较长，这里仅列出与 Listenner 对象构造相关的代码：

![img](https://user-gold-cdn.xitu.io/2018/2/9/1617819e591a2ed8?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 先从 Context 对象中取出实例变量 applicationListeners（该变量的值在 web.xml 解析时设置），第 12 行通过调用

```
instanceManager.newInstance(listener.getClassName())
```

，前面在看 StandardContext 的 startInternal 方法第 205 行时看到 instanceManager 被设置为 DefaultInstanceManager 对象，所以这里实际会执行 DefaultInstanceManager 类的 newInstance 方法：



```Java
public Object newInstance(String className) throws IllegalAccessException, InvocationTargetException, NamingException, InstantiationException, ClassNotFoundException {
    Class<?> clazz = loadClassMaybePrivileged(className, classLoader);
    return newInstance(clazz.newInstance(), clazz);
}

```

所以`instanceManager.newInstance(listener.getClassName())`这段代码的作用是取出 web.xml 中配置的 Listener 的 class 配置信息，从而构造实际配置的 Listener 对象。

看下 filterStart 方法：

![img](https://user-gold-cdn.xitu.io/2018/2/9/161781bd0c101334?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 这段代码看起来很简单，取出 web.xml 解析时读到的 filter 配置信息，在第 17 行调用 ApplicationFilterConfig 的构造方法：

![img](https://user-gold-cdn.xitu.io/2018/2/9/161781c8ea0dbf3d?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 默认情况下 filterDef 中是没有 Filter 对象的，所以会调用第 12 行 getFilter 方法：

![img](https://user-gold-cdn.xitu.io/2018/2/9/161781d1c694978c?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 与 Listener 的对象构造类似，都是通过调用

```
getInstanceManager().newInstance
```

方法。当然，按照 Servlet 规范，第 13 行还会调用 Filter 的 init 方法。



看下 loadOnStartup 方法：

![img](https://user-gold-cdn.xitu.io/2018/2/9/161781e00e7c7bc6?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 在 web 应用启动时将会加载配置了 load-on-startup 属性的 Servlet。第 24 行，调用了 StandardWrapper 类的 load 方法：

![img](https://user-gold-cdn.xitu.io/2018/2/9/161781ee508ff841?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 在第 2 行 loadServlet 方法中与上面的 Listener 和 Filter 对象构造一样调用

```
instanceManager.newInstance
```

来构造 Servlet 对象，与 Filter 类似在第 5 行调用 Servlet 的 init 方法。



当然这种加载只是针对配置了 load-on-startup 属性的 Servlet 而言，其它一般 Servlet 的加载和初始化会推迟到真正请求访问 web 应用而第一次调用该 Servlet 时，下面会看到这种情况下代码分析。

以上分析的 web 应用启动后这些对象的加载情况，接下来分析一下一次请求访问时，相关的 Filter、Servlet 对象的调用。

在之前的《Tomcat 7 的一次请求分析》系列文章中曾经分析了一次请求如何与容器中的 Engine、Host、Context、Wrapper 各级组件匹配，并在这些容器组件内部的管道中流转的。在[该系列第四篇文章](https://link.juejin.im?target=https%3A%2F%2Fjuejin.im%2Fpost%2F5a72c2886fb9a01ca915c732)最后提到，一次请求最终会执行与它最适配的一个 StandardWrapper 的基础阀`org.apache.catalina.core.StandardWrapperValve`的 invoke 方法。当时限于篇幅没继续往下分析，这里接着这段来看看请求的流转。看下 invoke 方法的代码：

![img](https://user-gold-cdn.xitu.io/2018/2/9/16178241f604aa50?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/9/1617824507e7f621?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/9/1617825126069002?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/9/1617825c3c920d05?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/9/1617826a3fedb2d1?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/9/161782713379072f?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 因为要支持 Servlet 3 的新特性及各种异常处理，这段代码显得比较长。关注重点第 42 行，这里会调用 StandardWrapper 的 allocate 方法，不再贴出这个方法的代码，需要提醒的是在 allocate 方法中可能会调用 loadServlet() 方法，这就是上一段提到的请求访问 web 应用而第一次调用该 Servlet 时再加载并初始化 Servlet 。



第 87 到 91 行会构造一个过滤器链（ filterChain ）用于执行这一次请求所经过的相应 Filter ，第 111 和第 128 行会调用该 filterChain 的 doFilter 方法：

![img](https://user-gold-cdn.xitu.io/2018/2/9/16178289b3794198?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 在该方法最后调用了 internalDoFilter 方法：

![img](https://user-gold-cdn.xitu.io/2018/2/9/16178297765e88e4?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/9/1617829f80ac390d?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/9/161782aae64d4027?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/2/9/161782b233c330fa?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 概述一下这段代码，第 6 到 60 行是执行过滤器链中的各个过滤器的 doFilter 方法，实例变量

```
n
```

表示过滤器链中所有的过滤器，

```
pos
```

表示当前要执行的过滤器。其中第 7 行取出当前要执行的 Filter，之后将

```
pos
```

加 1，接着第 30 行执行 Filter 的 doFilter 方法。一般的过滤器实现中在最后都会有这一句：



```Java
FilterChain.doFilter(request, response);

```

这样就又回到了 filterChain 的 doFilter 方法，形成了一个递归调用。要注意的是，filterChain 对象内部的 pos 是不断加的，所以假如过滤器链中的各个 Filter 的 doFilter 方法都执行完之后将会执行到第 63 行，在接下来的第 92 行、第 95 行即调用 Servlet 的 service 方法。