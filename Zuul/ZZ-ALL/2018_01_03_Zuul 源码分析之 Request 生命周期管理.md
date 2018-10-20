title: Zuul 源码分析之 Request 生命周期管理
date: 2018-01-03
tag: 
categories: Zuul
permalink: Zuul/haha7289/request-lifecycle
author: haha7289
from_url: https://blog.csdn.net/haha7289/article/details/54312043
wechat_url:

-------

摘要: 原创出处 https://blog.csdn.net/haha7289/article/details/54312043 「haha7289」欢迎转载，保留摘要，谢谢！

- [zuul核心框架](http://www.iocoder.cn/Zuul/haha7289/request-lifecycle/)
  - [ZuulFilter](http://www.iocoder.cn/Zuul/haha7289/request-lifecycle/)
- [Request生命周期](http://www.iocoder.cn/Zuul/haha7289/request-lifecycle/)
  - [ZuulServlet](http://www.iocoder.cn/Zuul/haha7289/request-lifecycle/)
  - [ZuulRunner](http://www.iocoder.cn/Zuul/haha7289/request-lifecycle/)
  - [`FilterProcessor`](http://www.iocoder.cn/Zuul/haha7289/request-lifecycle/)
  - [总结](http://www.iocoder.cn/Zuul/haha7289/request-lifecycle/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# zuul核心框架

zuul是可以认为是一种API-Gateway。zuul的核心是一系列的**filters**, 其作用可以类比Servlet框架的Filter，或者AOP。其原理就是在zuul把Request route到源web-service的时候，处理一些逻辑，比如Authentication，Load Shedding等。 下图是zuul的核心框架。对于框架中的核心类将一一分析。

![zuul核心框架](http://static.iocoder.cn/csdn/20170110200310378?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaGFoYTcyODk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## ZuulFilter

![ZuulFilter类图](http://static.iocoder.cn/csdn/20170110203340908?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaGFoYTcyODk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

`ZuulFilter`主要特征如下：

- **Type**：定义filter的类别，用字符串代表，有四种标准类别，代表了Request的生命周期。`filterType()`返回值代表该filter的Type。
  1. **PRE**: 该类型的filters在Request routing到源web-service之前执行。用来实现Authentication、选择源服务地址等
  2. **ROUTING**：该类型的filters用于把Request routing到源web-service，源web-service是实现业务逻辑的服务。这里使用HttpClient请求web-service。
  3. **POST**：该类型的filters在**ROUTING**返回Response后执行。用来实现对Response结果进行修改，收集统计数据以及把Response传输会客户端。
  4. **ERROR**：上面三个过程中任何一个出现错误都交由ERROR类型的filters进行处理。
- **Execution Order**: 同一个**Type**的filters组成Pipeline，**Execution Order**决定他们执行的顺序。`filterOrder()`返回值是该filter的**Execution Order**。
- **Criteria**：定义了filter执行需要满足的条件。对应的方法是`shouldFilter()`
- **Action**： 定了filter处理逻辑。对应的方法是`run()`。

下面是一个 简单的filter示例，该filter用于延迟一个出故障设备的请求。

```Java
class DeviceDelayFilter extends ZuulFilter {

    def static Random rand = new Random()

    @Override
     String filterType() {
       return 'pre'
     }

    @Override
    int filterOrder() {
       return 5
    }

    @Override
    boolean shouldFilter() {
 return  RequestContext.getRequest().
     getParameter("deviceType")?equals("BrokenDevice"):false
    }

    @Override
    Object run() {
 sleep(rand.nextInt(20000)) //Sleep for a random number of seconds
                                   //between [0-20]
    }
}
```

filter的功能并不具有太多特色，它和Servlet框架的Filter以及AOP功能及角色都很像，应该是zuul的开发者借鉴了这些优秀的设计。
zuul框架主要的功能就是动态的读取，编译，运行这些filter。filter之间不直接*communicate* ，他们之间通过`RequestContext`来共享状态信息，既然filter都是对特定Request的处理，那么`RequestContext`就是Request的Context，`RequestContext`用来管理 Request的Context，不受其它Request的影响。
Filter源码文件放在zuul 服务特定的目录， zuul server会定期扫描目录下的文件的变化。如果有Filter文件更新，源文件会被动态的读取，编译加载进入服务，接下来的Request处理就由这些新加入的filter处理。

# Request生命周期

![这里写图片描述](http://static.iocoder.cn/csdn/20170111153538629?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaGFoYTcyODk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## ZuulServlet

Zuul基于Servlet框架，`ZuulServlet`用于处理所有的Request。其可以认Http Request的入口。
其类图如下：
![ZuulServlet类图](http://static.iocoder.cn/csdn/20170110192746643?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaGFoYTcyODk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

如果对SpringMVC比较熟悉，可以把`ZuulServlet`类比为`DispatcherServlet`，所有的Request都要经过`ZuulServlet`的处理。因此`ZuulServlet`是zuul框架源码分析的入口点。
zuul本身不实现Web容器，因此zuul本身其实也就没有太过复杂的线程模型和执行逻辑。不过在此回顾下Servlet框架以及典型Web 容器的线程模型，这也是理解zuul线程模型的关键。如果对`SpringMVC`比较熟悉的话，那么zuul的在整个web 容器所处的位置基本上和`SpringMVC`一致。

### Servlet的生命周期

Servlet 通过一个定义良好的生命周期来进行管理,该生命周期规定了 Servlet 如何被加载、实例化、初始化、 处理客户端请求,以及何时结束服务。该生命周期可以通过 `javax.servlet.Servlet` 接口中的 `init`、`service` 和 `destroy` 这些 API 来表示,所有 Servlet 必须直接或间接的实现 `GenericServlet` 或 `HttpServlet` 抽象类。
Servlet的生命周期有四个阶段：**加载并实例化**、**初始化**、**请求处理**、**销毁**。主要涉及到的方法有`init`、`service`、`doGet`、`doPost`、`destory`等。

![Servlet的生命周期](http://static.iocoder.cn/csdn/20170110190902092?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaGFoYTcyODk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### Web容器线程模型

Servlet只是基于Java技术的web组件，该组件由容器托管，用于生成动态内容。Servlet容器是web Server或application server 的一部分，供基于Request/Response发送模型的网络服务，解码基于MIME的请求，并格式化基于MIME的响应。Servlet容器包含并管理Servlet生命周期。典型的Servlet容器有Tomcat、Jetty。

![Tomcat的线程模型](http://static.iocoder.cn/csdn/20170110191446224?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaGFoYTcyODk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

上图是Tomcat基于NIO的线程模型，其基于典型的Acceptor/Reactor线程模型。

![Acceptor/Reactor线程模型](http://static.iocoder.cn/csdn/20170110191827045?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaGFoYTcyODk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

此处不详细分析Tomcat对该线程模型的实现，仅关心与Servlet生命周期相关的部分。在Tomcat的线程模型中，Worker线程用来处理Request。当容器收到一个Request后，调度线程从Worker线程池中选出一个Worker线程，将请求传递给该线程，然后由该线程来执行Servlet的`service()`方法。且该worker线程只能同时处理一个Request请求，如果过程中发生了阻塞，那么该线程就会被阻塞，而不能去处理其他任务。
Servlet默认情况下一个**单例多线程**。

回到zuul，由上面的回顾可知，zuul逻辑的入口必定是`ZuulServlet.service(ServletRequest servletRequest, ServletResponse servletResponse)`。下面看下源代码的简化版 ，只保留逻辑部分，。

```Java
    public void service(ServletRequest servletRequest, ServletResponse servletResponse)  {
            // 用于初始化RequestContext
            init((HttpServletRequest) servletRequest, (HttpServletResponse) servletResponse);
            /* RequestContext 用于记录Request的context。前面也分析了，由于Servlet是单例多线程的，而Request由唯一 worker线程处理,这里的RequestContext使用`ThreadLocal`实现，其本身简单wrap了`ConcurrentHashMap`,*/
            RequestContext context = RequestContext.getCurrentContext();
            // 执行Pre filters逻辑
            preRoute();
            // 执行route逻辑
             route();
             // 执行postRoute逻辑
             postRoute();
    }
```

`RequestContext`提供了执行filter Pipeline所需要的Context，因为Servlet是单例多线程，这就要求RequestContext即要**线程安全**又要**Request安全**。context使用`ThreadLocal`保存，这样每个worker线程都有一个与其绑定的`RequestContext`，因为worker仅能同时处理一个Request，这就保证了Request Context 即是线程安全的由是Request安全的。所谓Request 安全，即该Request的Context不会与其他同时处理Request冲突。
`RequestContext`简单wrap 了`ConcurrentHashMap`吗，没有太过复杂的逻辑，此处不再分析。

三个核心的方法`preRoute()`,`route()`, `postRoute()`，zuul对request处理逻辑都在这三个方法里，`ZuulServlet`交给`ZuulRunner`去执行。由于`ZuulServlet`是单例，因此`ZuulRunner`也仅有一个实例。

## ZuulRunner

`ZuulRunner`直接将执行逻辑交由`FilterProcessor`处理。
`FilterProcessor`也是单例。
![ZuulRunner类图](http://static.iocoder.cn/csdn/20170110202730091?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaGFoYTcyODk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## `FilterProcessor`

其功能就是依据filterType执行filter的处理逻辑，其类图如下。

![FilterProcessor类图](http://static.iocoder.cn/csdn/20170110111228755?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaGFoYTcyODk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

核心方法`runFilters()`

```Java
    public Object runFilters(String sType) throws Throwable {
        if (RequestContext.getCurrentContext().debugRouting()) {
            Debug.addRoutingDebug("Invoking {" + sType + "} type filters");
        }
        boolean bResult = false;
        List<ZuulFilter> list = FilterLoader.getInstance().getFiltersByType(sType);
        if (list != null) {
            for (int i = 0; i < list.size(); i++) {
                ZuulFilter zuulFilter = list.get(i);
                Object result = processZuulFilter(zuulFilter);
                if (result != null && result instanceof Boolean) {
                    bResult |= ((Boolean) result);
                }
            }
        }
        return bResult;
    }
```

每个filter的处理逻辑，仅保留逻辑部分。

```Java
    public Object processZuulFilter(ZuulFilter filter) throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        long execTime = 0;
        String filterName = "";
        try {
            long ltime = System.currentTimeMillis();
            // 运行filter的处理逻辑
            ZuulFilterResult result = filter.runFilter();
            ExecutionStatus s = result.getStatus();
            execTime = System.currentTimeMillis() - ltime;
            // 记录filter的处理状态，如果filter处理失败，把异常抛出
            switch (s) {
                case FAILED:
                    t = result.getException();
                    ctx.addFilterExecutionSummary(filterName, ExecutionStatus.FAILED.name(), execTime);
                    break;
                case SUCCESS:
                    o = result.getResult();
                    ctx.addFilterExecutionSummary(filterName, ExecutionStatus.SUCCESS.name(), execTime);
                    break;
                default:
                    break;
            }
            if (t != null) throw t;
            usageNotifier.notify(filter, s);
            return o;
        } catch (Throwable e) {
          usageNotifier.notify(filter, ExecutionStatus.FAILED);
        }
    }
```

上面两段代码是`FilterProcessor`对filter的处理逻辑。

- 首先根据**Type**获取所有输入该Type的filter，`List<ZuulFilter> list`。
- 遍历该list，执行每个filter的处理逻辑，`processZuulFilter(ZuulFilter filter)`
- `RequestContext`对每个filter的执行状况进行记录，应该留意，此处的执行状态主要包括其执行时间、以及执行成功或者失败，如果执行失败则对异常封装后抛出。
  到目前为止，zuul框架对每个filter的执行结果都没有太多的处理，它没有把上一filter的执行结果交由下一个将要执行的filter，仅仅是记录执行状态，如果执行失败抛出异常并终止执行。

最后看下方法`ZuulFilter.runFilter()`

```Java
public ZuulFilterResult runFilter() {
        ZuulFilterResult zr = new ZuulFilterResult();
        if (!isFilterDisabled()) {
            if (shouldFilter()) {
                Tracer t = TracerFactory.instance().startMicroTracer("ZUUL::" + this.getClass().getSimpleName());
                try {
                    Object res = run();
                    zr = new ZuulFilterResult(res, ExecutionStatus.SUCCESS);
                } catch (Throwable e) {
                    t.setName("ZUUL::" + this.getClass().getSimpleName() + " failed");
                    zr = new ZuulFilterResult(ExecutionStatus.FAILED);
                    zr.setException(e);
                } finally {
                    t.stopAndLog();
                }
            } else {
                zr = new ZuulFilterResult(ExecutionStatus.SKIPPED);
            }
        }
        return zr;
    }
```

zuul框架对filter的处理到此就结束了。从上面代码可以看出，`ZuulFilterResult` 记录了该filter的执行状态，`run()` 中返回的Object其实在zuul框架中没有用到过。
分析到这里，可以看出除去 zuul框架对filter的管理，zuul框架作用有限。而且filter也十分简单，没有Servlet框架中Filter丰富。

## 总结

- `ZuulServlet`是zuul框架的入口，其采用`Servlet`框架，是单例多线程，可以把它类比为SpringMVC的`DispatcherServlet`。`ZuulServlet`定义Request的生命周期内的处理逻辑，每个阶段的具体处理逻辑交由`ZuulRunner`和`FilterProcessor`。
- 以上是Request生命周期内的主要逻辑。下面分析Filter生命周期管理。

# 666. 彩蛋

如果你对 Zuul  感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)