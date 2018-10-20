title: Zuul源码分析 —— Filter 注册
date: 2018-01-04
tag: 
categories: Zuul
permalink: Zuul/yuan135/filter-registry
author: 源135
from_url: https://my.oschina.net/u/3300636/blog/851984
wechat_url: 

-------

摘要: 原创出处 https://my.oschina.net/u/3300636/blog/851984 「源135」欢迎转载，保留摘要，谢谢！


-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

最近项目中使用zuul，zuul网上资料较少，顺便把源码看一遍分析一下。网上有几篇相关文章，已转载在博客中，这里再补充一点东西，应该就差不多了。这里讲的有点跳，如果zuul不熟悉，建议先看下[zuul源码分析之Request生命周期管理](http://blog.csdn.net/haha7289/article/details/54312043) 这篇文章。

### zuul开启

首先由@EnableZuulProxy 注解是开启zuul的注解。

```Java
@EnableCircuitBreaker
@EnableDiscoveryClient
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
//引入zuul配置
@Import({ZuulProxyConfiguration.class})
public @interface EnableZuulProxy {
}
```

ZuulProxyConfiguration 继承了ZuulConfiguration

```Java
//该配置等同于xml中的beans
@Configuration
public class ZuulProxyConfiguration extends ZuulConfiguration {
```

```Java
@Configuration
@EnableConfigurationProperties({ZuulProperties.class})
//加载了ZuulServlet后才会加载该配置
@ConditionalOnClass({ZuulServlet.class})
@Import({ServerPropertiesAutoConfiguration.class})
public class ZuulConfiguration {
```

这两个类主要加载了四类东西：

1，注册zuulServlet，ZuulController 这是后面调用一些列zuulFilter的入口。

```Java
//ZuulController实现ServletWrappingController
protected ModelAndView handleRequestInternal(HttpServletRequest request, HttpServletResponse response) throws Exception {
    ModelAndView var3;
    try {
       //此处调用zuulServlet中的方法
        var3 = super.handleRequestInternal(request, response);
    } finally {
        RequestContext.getCurrentContext().unset();
    }

    return var3;
}
```

2，加载zuul各种自带的filter，比如preDecorationFilter、ribbonRoutingFilter等等

  我这个版本中（1.1.2.RELEASE），zuul默认加载的filter有： pre类型的filter5个，route的3个，post的2个，共10个filter。  每个filter的功能这里就不详述了。

一些场景下，我们想要禁用掉部分过滤器，此时该怎么办呢？在网上看到了这个方法：

只需设置`zuul.<SimpleClassName>.<filterType>.disable=true` ，即可禁用SimpleClassName所对应的过滤器。以过滤器org.springframework.cloud.netflix.zuul.filters.post.SendResponseFilter为例，只需设置`zuul.SendResponseFilter.post.disable=true` ，即可禁用该过滤器。

3，加载ZuulFilterConfiguration，这个是定义在ZuulConfiguration中的Configuration。这个配置是将所有filter由ZuulFilterInitializer收集到FilterRegistry中。调用时，filter都是从这当中取的。

```Java
@Configuration
protected static class ZuulFilterConfiguration {
//按类型将所有ZuulFilter注入到map中
    @Autowired
    private Map<String, ZuulFilter> filters;

    protected ZuulFilterConfiguration() {
    }

    @Bean
    public ZuulFilterInitializer zuulFilterInitializer() {
        return new ZuulFilterInitializer(this.filters);
    }
}
```

4，加载ZuulRefreshListener、ZuulDiscoveryRefreshListener用于动态刷新zuul配置

p.s. 当应用启动时，加载顺序是：Configuration配置》ConditionalOnClass注解的bean》Configuration中的Configuration》自定义bean》Configuration中的bean》Configuration中的Configuration中的bean

### zuul之filter调用

基本的调用顺序就不细说了，转载文章中介绍的比较详细。这里说一下zuul一个有点绕的地方。在ZuulServlet中

```Java
public void service(ServletRequest servletRequest, ServletResponse servletResponse) throws ServletException, IOException {
    try {
        this.init((HttpServletRequest)servletRequest, (HttpServletResponse)servletResponse);
        RequestContext e = RequestContext.getCurrentContext();
        e.setZuulEngineRan();

        try {
            this.preRoute();
        } catch (ZuulException var12) {
            this.error(var12);
            this.postRoute();
            return;
        }

        try {
            this.route();
        } catch (ZuulException var13) {
            this.error(var13);
            this.postRoute();
            return;
        }

        try {
            this.postRoute();
        } catch (ZuulException var11) {
            this.error(var11);
        }
    } catch (Throwable var14) {
        this.error(new ZuulException(var14, 500, "UNHANDLED_EXCEPTION_" + var14.getClass().getName()));
    } finally {
       //清空上下文
        RequestContext.getCurrentContext().unset();
    }
}
```

如果执行pre类型的filter抛出ZuulException则执行error类型的filter,再执行post类型的filter

如果执行route类型的filter抛出ZuulException则执行error类型的filter,再执行post类型的filter

如果执行post类型的filter抛出ZuulException则执行error类型的filter

如果在抛出其他异常，或者在catch ZuulException后执行发生异常，则执行error类型的filter

这里有点绕，如果使用的时候需要注意一下。另外zuul许多默认的filter,虽然是zuul开发人员考虑周到，但在我们自己定制功能中，也可能造成麻烦。

# 666. 彩蛋

如果你对 Zuul  感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)