title: Tomcat 7 的一次请求分析（四）Tomcat 7 阀机制原理
date: 2018-01-13
tag: 
categories: Tomcat
permalink: Tomcat/yuliu/A-request-analysis-4-Tomcat-7-valve-mechanism-principle
author: 预流
from_url: https://juejin.im/post/5a72c2886fb9a01ca915c732
wechat_url: 

-------


摘要: 原创出处 https://juejin.im/post/5a72c2886fb9a01ca915c732 「预流」欢迎转载，保留摘要，谢谢！


-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

通过这一系列的前三部分看到了一次客户端连接在 Tomcat 内部被转换成了请求对象（`org.apache.catalina.connector.Request`类的实例），并在该请求对象内部将与本次请求相关的 Host、Context、Wrapper 对象的引用。本文主要分析该请求对象在容器内部流转的经过。

再来看一下 Tomcat 7 内部的组件结构图：

![img](https://user-gold-cdn.xitu.io/2018/1/31/1614b97971ecddc5?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 其实这张图已经给出了答案，在 Connector 接收到一次连接并转化成请求（ Request ）后，会将请求传递到 Engine 的管道（ Pipeline ）的阀（ ValveA ）中。请求在 Engine 的管道中最终会传递到 Engine Valve 这个阀中。接着请求会从 Engine Valve 传递到一个 Host 的管道中，在该管道中最后传递到 Host Valve 这个阀里。接着从 Host Valve 传递到一个 Context 的管道中，在该管道中最后传递到 Context Valve 中。接下来请求会传递到 Wrapper C 内的管道所包含的阀 Wrapper Valve 中，在这里会经过一个过滤器链（ Filter Chain ），最终送到一个 Servlet 中。



如果你不了解上面这段文字描述中所谓的管道（ Pipeline ）和阀（ Valve ）的概念，别急，下面会讲到这个。先从源码层面看下这段文字描述的经过。上面提到的`org.apache.catalina.connector.CoyoteAdapter`类的 service 方法：

```Java
     1	    public void service(org.apache.coyote.Request req,
     2	                        org.apache.coyote.Response res)
     3	        throws Exception {
     4
     5	        Request request = (Request) req.getNote(ADAPTER_NOTES);
     6	        Response response = (Response) res.getNote(ADAPTER_NOTES);
     7
     8	        if (request == null) {
     9
    10	            // Create objects
    11	            request = connector.createRequest();
    12	            request.setCoyoteRequest(req);
    13	            response = connector.createResponse();
    14	            response.setCoyoteResponse(res);
    15
    16	            // Link objects
    17	            request.setResponse(response);
    18	            response.setRequest(request);
    19
    20	            // Set as notes
    21	            req.setNote(ADAPTER_NOTES, request);
    22	            res.setNote(ADAPTER_NOTES, response);
    23
    24	            // Set query string encoding
    25	            req.getParameters().setQueryStringEncoding
    26	                (connector.getURIEncoding());
    27
    28	        }
    29
    30	        if (connector.getXpoweredBy()) {
    31	            response.addHeader("X-Powered-By", POWERED_BY);
    32	        }
    33
    34	        boolean comet = false;
    35	        boolean async = false;
    36
    37	        try {
    38
    39	            // Parse and set Catalina and configuration specific
    40	            // request parameters
    41	            req.getRequestProcessor().setWorkerThreadName(Thread.currentThread().getName());
    42	            boolean postParseSuccess = postParseRequest(req, request, res, response);
    43	            if (postParseSuccess) {
    44	                //check valves if we support async
    45	                request.setAsyncSupported(connector.getService().getContainer().getPipeline().isAsyncSupported());
    46	                // Calling the container
    47	                connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);
    48
    49	                if (request.isComet()) {
    50	                    if (!response.isClosed() && !response.isError()) {
    51	                        if (request.getAvailable() || (request.getContentLength() > 0 && (!request.isParametersParsed()))) {
    52	                            // Invoke a read event right away if there are available bytes
    53	                            if (event(req, res, SocketStatus.OPEN)) {
    54	                                comet = true;
    55	                                res.action(ActionCode.COMET_BEGIN, null);
    56	                            }
    57	                        } else {
    58	                            comet = true;
    59	                            res.action(ActionCode.COMET_BEGIN, null);
    60	                        }
    61	                    } else {
    62	                        // Clear the filter chain, as otherwise it will not be reset elsewhere
    63	                        // since this is a Comet request
    64	                        request.setFilterChain(null);
    65	                    }
    66	                }
    67
    68	            }
    69	            AsyncContextImpl asyncConImpl = (AsyncContextImpl)request.getAsyncContext();
    70	            if (asyncConImpl != null) {
    71	                async = true;
    72	            } else if (!comet) {
    73	                request.finishRequest();
    74	                response.finishResponse();
    75	                if (postParseSuccess &&
    76	                        request.getMappingData().context != null) {
    77	                    // Log only if processing was invoked.
    78	                    // If postParseRequest() failed, it has already logged it.
    79	                    // If context is null this was the start of a comet request
    80	                    // that failed and has already been logged.
    81	                    ((Context) request.getMappingData().context).logAccess(
    82	                            request, response,
    83	                            System.currentTimeMillis() - req.getStartTime(),
    84	                            false);
    85	                }
    86	                req.action(ActionCode.POST_REQUEST , null);
    87	            }
    88
    89	        } catch (IOException e) {
    90	            // Ignore
    91	        } finally {
    92	            req.getRequestProcessor().setWorkerThreadName(null);
    93	            // Recycle the wrapper request and response
    94	            if (!comet && !async) {
    95	                request.recycle();
    96	                response.recycle();
    97	            } else {
    98	                // Clear converters so that the minimum amount of memory
    99	                // is used by this processor
   100	                request.clearEncoders();
   101	                response.clearEncoders();
   102	            }
   103	        }
   104
   105	    }

```

之前主要分析了第 42 行的代码，通过 postParseRequest 方法的调用请求对象内保存了关于本次请求的具体要执行的 Host、Context、Wrapper 组件的引用。

看下第 47 行：

```Java
                connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);

```

虽然只有一行，但调用了一堆方法，这里对这些方法逐个分析一下：

connector.getService() 获取的是当前 connector 关联的 Service 组件，默认情况下获得的就是`org.apache.catalina.core.StandardService`的对象。其 getContainer 方法获得的是`org.apache.catalina.core.StandardEngine`的对象，这段的由来在[前面讲 Digester 的解析文章](https://link.juejin.im?target=https%3A%2F%2Fjuejin.im%2Fpost%2F5a6d1ff6f265da3e243bc1de)时，createStartDigester 方法中的这段代码：

```Java
        digester.addRuleSet(new EngineRuleSet("Server/Service/"));

```

在 EngineRuleSet 类的 addRuleInstances 方法中的这一段代码：

```Java
    public void addRuleInstances(Digester digester) {

        digester.addObjectCreate(prefix + "Engine",
                                 "org.apache.catalina.core.StandardEngine",
                                 "className");
        digester.addSetProperties(prefix + "Engine");
        digester.addRule(prefix + "Engine",
                         new LifecycleListenerRule
                         ("org.apache.catalina.startup.EngineConfig",
                          "engineConfigClass"));
        digester.addSetNext(prefix + "Engine",
                            "setContainer",
                            "org.apache.catalina.Container");

```

结合上一段代码可以看出 Tomcat 启动时，如果碰到 server.xml 里的 Server/Service/Engine 节点，先实例化一个`org.apache.catalina.core.StandardEngine`对象，在第 11 到 13 行，会以 StandardEngine 对象为入参调用`org.apache.catalina.core.StandardService`的 setContainer 方法。

所以上面 connector.getService().getContainer() 方法得到的实际上是 StandardEngine 对象。紧接着的 getPipeline 方法返回的是 StandardEngine 类的父类`org.apache.catalina.core.ContainerBase`类的成员变量 pipeline ，看下该类中这个变量的声明代码：

```Java
    /**
     * The Pipeline object with which this Container is associated.
     */
    protected Pipeline pipeline = new StandardPipeline(this);

```

所以 connector.getService().getContainer().getPipeline() 方法返回的是`org.apache.catalina.core.StandardPipeline`类的对象，该对象就是本部分开头部分提到的管道（ Pipeline ）。

下面讲一下 Tomcat 7 中的管道和阀的概念和实现：

所有的管道类都会实现`org.apache.catalina.Pipeline`这个接口，看下这个接口中定义的方法：

![img](https://user-gold-cdn.xitu.io/2018/1/31/1614b9dfa8e689e4?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 Tomat 7 中一个管道包含多个阀（ Valve ），这些阀共分为两类，一类叫基础阀（通过 getBasic、setBasic 方法调用），一类是普通阀（通过 addValve、removeValve 调用）。管道都是包含在一个容器当中，所以 API 里还有 getContainer 和 setContainer 方法。一个管道一般有一个基础阀（通过 setBasic 添加），可以有 0 到多个普通阀（通过 addValve 添加）。



所有的阀类都会实现`org.apache.catalina.Valve`这个接口，看下这个接口中定义的方法：

![img](https://user-gold-cdn.xitu.io/2018/1/31/1614b9eee6cec248?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 重点关注 setNext、getNext、invoke 这三个方法，通过setNext设置该阀的下一阀，通过 getNext 返回该阀的下一个阀的引用，invoke 方法则执行该阀内部自定义的请求处理代码。



Tomcat 7 里 Pipeline 的默认实现类是`org.apache.catalina.core.StandardPipeline`，其内部有三个成员变量：basic、first、container 。

```Java
    /**
     * The basic Valve (if any) associated with this Pipeline.
     */
    protected Valve basic = null;

    /**
     * The Container with which this Pipeline is associated.
     */
    protected Container container = null;

    /**
     * The first valve associated with this Pipeline.
     */
    protected Valve first = null;

```

看下该类的 addValve 方法：

```Java
     1	    public void addValve(Valve valve) {
     2
     3	        // Validate that we can add this Valve
     4	        if (valve instanceof Contained)
     5	            ((Contained) valve).setContainer(this.container);
     6
     7	        // Start the new component if necessary
     8	        if (getState().isAvailable()) {
     9	            if (valve instanceof Lifecycle) {
    10	                try {
    11	                    ((Lifecycle) valve).start();
    12	                } catch (LifecycleException e) {
    13	                    log.error("StandardPipeline.addValve: start: ", e);
    14	                }
    15	            }
    16	        }
    17
    18	        // Add this Valve to the set associated with this Pipeline
    19	        if (first == null) {
    20	            first = valve;
    21	            valve.setNext(basic);
    22	        } else {
    23	            Valve current = first;
    24	            while (current != null) {
    25	                if (current.getNext() == basic) {
    26	                    current.setNext(valve);
    27	                    valve.setNext(basic);
    28	                    break;
    29	                }
    30	                current = current.getNext();
    31	            }
    32	        }
    33
    34	        container.fireContainerEvent(Container.ADD_VALVE_EVENT, valve);
    35	    }

```

在第 18 到 32 行，每次给管道添加一个普通阀的时候如果管道内原来没有普通阀则将新添加的阀作为该管道的成员变量 first 的引用，如果管道内已有普通阀，则把新加的阀加到所有普通阀链条末端，并且将该阀的下一个阀的引用设置为管道的基础阀。这样管道内的阀结构如下图所示：

![img](https://user-gold-cdn.xitu.io/2018/1/31/1614ba071f88ac56?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 即 Pipeline 内部维护 first 和 basic 两个阀，其它相关阀通过 getNext 来获取。



看下 getFirst 方法的实现：

```Java
    public Valve getFirst() {
        if (first != null) {
            return first;
        }

        return basic;
    }

```

如果管道中有普通阀则返回普通阀链条最开始的那个，否则就返回基础阀。

在 Tomcat 7 中所有作为普通阀的类的 invoke 方法实现中都会有这段代码：

getNext().invoke(request, response);

```

通过这种机制来保证调用管道最开头一端的阀的 invoke 方法，最终会执行完该管道相关的所有阀的 invoke 方法，并且最后执行的必定是该管道基础阀的 invoke 方法。

再回到`connector.getService().getContainer().getPipeline().getFirst().invoke(request, response)`这段代码的解释，这里将会执行 StandardEngine 类的管道中的所有阀（包括普通阀和基础阀）的 invoke 方法，并且最后会执行基础阀的 invoke 方法。

Tomcat 7 在默认情况下 Engine 节点没有普通阀，如果想要添加普通阀的话，可以通过在 server.xml 文件的 engine 节点下添加 Valve 节点，参加该文件中的普通阀配置的示例：

```XML
<Valve className="org.apache.catalina.authenticator.SingleSignOn" />

```

那么就来看看 StandardEngine 类的管道中的基础阀的代码实现。先看下该基础阀设置的代码，在`org.apache.catalina.core.StandardEngine`对象的构造函数中：

```Java
     1	    public StandardEngine() {
     2
     3	        super();
     4	        pipeline.setBasic(new StandardEngineValve());
     5	        /* Set the jmvRoute using the system property jvmRoute */
     6	        try {
     7	            setJvmRoute(System.getProperty("jvmRoute"));
     8	        } catch(Exception ex) {
     9	            log.warn(sm.getString("standardEngine.jvmRouteFail"));
    10	        }
    11	        // By default, the engine will hold the reloading thread
    12	        backgroundProcessorDelay = 10;
    13
    14	    }

```

第 4 行即设置基础阀。所以`connector.getService().getContainer().getPipeline().getFirst().invoke(request, response)`会执行到 org.apache.catalina.core.StandardEngineValve 类的 invoke 方法：

```Java
     1	    public final void invoke(Request request, Response response)
     2	        throws IOException, ServletException {
     3
     4	        // Select the Host to be used for this Request
     5	        Host host = request.getHost();
     6	        if (host == null) {
     7	            response.sendError
     8	                (HttpServletResponse.SC_BAD_REQUEST,
     9	                 sm.getString("standardEngine.noHost",
    10	                              request.getServerName()));
    11	            return;
    12	        }
    13	        if (request.isAsyncSupported()) {
    14	            request.setAsyncSupported(host.getPipeline().isAsyncSupported());
    15	        }
    16
    17	        // Ask this Host to process this request
    18	        host.getPipeline().getFirst().invoke(request, response);
    19
    20	    }

```

第 5 行，从请求对象中取出该请求关联的 Host（默认情况下是`org.apache.catalina.core.StandardHost`对象），请求是如何找到关联的 Host 的请本文之前的部分。经过上述代码分析应该可以看出第 18 行会执行 StandardHost 对象的管道内所有的阀的 invoke 方法。

看下 StandardHost 的构造方法的实现：

    public StandardHost() {

        super();
        pipeline.setBasic(new StandardHostValve());

    }

```

所以看下`org.apache.catalina.core.StandardHostValve`类的 invoke 方法：

```Java
     1	    public final void invoke(Request request, Response response)
     2	        throws IOException, ServletException {
     3
     4	        // Select the Context to be used for this Request
     5	        Context context = request.getContext();
     6	        if (context == null) {
     7	            response.sendError
     8	                (HttpServletResponse.SC_INTERNAL_SERVER_ERROR,
     9	                 sm.getString("standardHost.noContext"));
    10	            return;
    11	        }
    12
    13	        // Bind the context CL to the current thread
    14	        if( context.getLoader() != null ) {
    15	            // Not started - it should check for availability first
    16	            // This should eventually move to Engine, it's generic.
    17	            if (Globals.IS_SECURITY_ENABLED) {
    18	                PrivilegedAction pa = new PrivilegedSetTccl(
    19	                        context.getLoader().getClassLoader());
    20	                AccessController.doPrivileged(pa);
    21	            } else {
    22	                Thread.currentThread().setContextClassLoader
    23	                        (context.getLoader().getClassLoader());
    24	            }
    25	        }
    26	        if (request.isAsyncSupported()) {
    27	            request.setAsyncSupported(context.getPipeline().isAsyncSupported());
    28	        }
    29
    30	        // Don't fire listeners during async processing
    31	        // If a request init listener throws an exception, the request is
    32	        // aborted
    33	        boolean asyncAtStart = request.isAsync();
    34	        // An async error page may dispatch to another resource. This flag helps
    35	        // ensure an infinite error handling loop is not entered
    36	        boolean errorAtStart = response.isError();
    37	        if (asyncAtStart || context.fireRequestInitEvent(request)) {
    38
    39	            // Ask this Context to process this request
    40	            try {
    41	                context.getPipeline().getFirst().invoke(request, response);
    42	            } catch (Throwable t) {
    43	                ExceptionUtils.handleThrowable(t);
    44	                if (errorAtStart) {
    45	                    container.getLogger().error("Exception Processing " +
    46	                            request.getRequestURI(), t);
    47	                } else {
    48	                    request.setAttribute(RequestDispatcher.ERROR_EXCEPTION, t);
    49	                    throwable(request, response, t);
    50	                }
    51	            }
    52
    53	            // If the request was async at the start and an error occurred then
    54	            // the async error handling will kick-in and that will fire the
    55	            // request destroyed event *after* the error handling has taken
    56	            // place
    57	            if (!(request.isAsync() || (asyncAtStart &&
    58	                    request.getAttribute(
    59	                            RequestDispatcher.ERROR_EXCEPTION) != null))) {
    60	                // Protect against NPEs if context was destroyed during a
    61	                // long running request.
    62	                if (context.getState().isAvailable()) {
    63	                    if (!errorAtStart) {
    64	                        // Error page processing
    65	                        response.setSuspended(false);
    66
    67	                        Throwable t = (Throwable) request.getAttribute(
    68	                                RequestDispatcher.ERROR_EXCEPTION);
    69
    70	                        if (t != null) {
    71	                            throwable(request, response, t);
    72	                        } else {
    73	                            status(request, response);
    74	                        }
    75	                    }
    76
    77	                    context.fireRequestDestroyEvent(request);
    78	                }
    79	            }
    80	        }
    81
    82	        // Access a session (if present) to update last accessed time, based on a
    83	        // strict interpretation of the specification
    84	        if (ACCESS_SESSION) {
    85	            request.getSession(false);
    86	        }
    87
    88	        // Restore the context classloader
    89	        if (Globals.IS_SECURITY_ENABLED) {
    90	            PrivilegedAction pa = new PrivilegedSetTccl(
    91	                    StandardHostValve.class.getClassLoader());
    92	            AccessController.doPrivileged(pa);
    93	        } else {
    94	            Thread.currentThread().setContextClassLoader
    95	                    (StandardHostValve.class.getClassLoader());
    96	        }
    97	    }

```

第 41 行，会调用该请求相关的 Context 的管道内所有的阀的 invoke 方法，默认情况下 Context 是`org.apache.catalina.core.StandardContext`类的对象，其构造方法中设置了管道的基础阀：

```Java
    public StandardContext() {

        super();
        pipeline.setBasic(new StandardContextValve());
        broadcaster = new NotificationBroadcasterSupport();
        // Set defaults
        if (!Globals.STRICT_SERVLET_COMPLIANCE) {
            // Strict servlet compliance requires all extension mapped servlets
            // to be checked against welcome files
            resourceOnlyServlets.add("jsp");
        }
    }

```

看下其基础阀的 invoke 方法代码：

```Java
     1	    public final void invoke(Request request, Response response)
     2	        throws IOException, ServletException {
     3
     4	        // Disallow any direct access to resources under WEB-INF or META-INF
     5	        MessageBytes requestPathMB = request.getRequestPathMB();
     6	        if ((requestPathMB.startsWithIgnoreCase("/META-INF/", 0))
     7	                || (requestPathMB.equalsIgnoreCase("/META-INF"))
     8	                || (requestPathMB.startsWithIgnoreCase("/WEB-INF/", 0))
     9	                || (requestPathMB.equalsIgnoreCase("/WEB-INF"))) {
    10	            response.sendError(HttpServletResponse.SC_NOT_FOUND);
    11	            return;
    12	        }
    13
    14	        // Select the Wrapper to be used for this Request
    15	        Wrapper wrapper = request.getWrapper();
    16	        if (wrapper == null || wrapper.isUnavailable()) {
    17	            response.sendError(HttpServletResponse.SC_NOT_FOUND);
    18	            return;
    19	        }
    20
    21	        // Acknowledge the request
    22	        try {
    23	            response.sendAcknowledgement();
    24	        } catch (IOException ioe) {
    25	            container.getLogger().error(sm.getString(
    26	                    "standardContextValve.acknowledgeException"), ioe);
    27	            request.setAttribute(RequestDispatcher.ERROR_EXCEPTION, ioe);
    28	            response.sendError(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
    29	            return;
    30	        }
    31
    32	        if (request.isAsyncSupported()) {
    33	            request.setAsyncSupported(wrapper.getPipeline().isAsyncSupported());
    34	        }
    35	        wrapper.getPipeline().getFirst().invoke(request, response);
    36	    }

```

最后的第 35 行，从请求中取出关联的 wrapper 对象后调用其管道内所有阀的 invoke 方法。wrapper 对象默认是`org.apache.catalina.core.StandardWrapper`类的实例，同样是在该类的构造方法中设置的基础阀：

```Java
    public StandardWrapper() {

        super();
        swValve=new StandardWrapperValve();
        pipeline.setBasic(swValve);
        broadcaster = new NotificationBroadcasterSupport();

    }

```

有兴趣可以看下基础阀`org.apache.catalina.core.StandardWrapperValve`的 invoke 方法，在这里最终会调用请求的 url 所匹配的 Servlet 相关过滤器（ filter ）的 doFilter 方法及该 Servlet 的 service 方法（这段实现都是在过滤器链 ApplicationFilterChain 类的 doFilter 方法中），这里不再贴出代码分析。

这里可以看出容器内的 Engine、Host、Context、Wrapper 容器组件的实现的共通点：

1. 这些组件内部都有一个成员变量 pipeline ，因为它们都是从`org.apache.catalina.core.ContainerBase`类继承来的，pipeline 就定义在这个类中。所以每一个容器内部都关联了一个管道。
2. 都是在类的构造方法中设置管道内的基础阀。
3. 所有的基础阀的实现最后都会调用其下一级容器（直接从请求中获取下一级容器对象的引用，在上面的分析中已经设置了与该请求相关的各级具体组件的引用）的 getPipeline().getFirst().invoke() 方法，直到 Wrapper 组件。因为 Wrapper 是对一个 Servlet 的包装，所以它的基础阀内部调用的过滤器链的 doFilter 方法和 Servlet 的 service 方法。

正是通过这种管道和阀的机制及上述的 3 点前提，使得请求可以从连接器内一步一步流转到具体 Servlet 的 service 方法中。这样，关于Tomcat 7 中一次请求的分析介绍完毕，从中可以看出在浏览器发出一次 Socket 连接请求之后 Tomcat 容器内运转处理的大致流程。
