title: Tomcat 7 启动分析（四）各组件 init、start 方法调用
date: 2018-01-06
tag: 
categories: Tomcat
permalink: Tomcat/yuliu/Start-analysis-4-components-int-and-start-method
author: 预流
from_url: https://juejin.im/post/5a6d6f6751882573520da54d
wechat_url: 

-------

摘要: 原创出处 https://juejin.im/post/5a6d6f6751882573520da54d 「预流」欢迎转载，保留摘要，谢谢！


-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

在正常启动 Tomcat 7 的情况下，上篇文章分析到了执行 org.apache.catalina.core.StandardServer 的 init 和 start 方法这儿，那么就来看看这两个方法里面到底干了些什么。

但是在 StandardServer 类里面并没有发现这两个方法：

![img](https://user-gold-cdn.xitu.io/2018/1/28/1613b7bf0965d9af?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 由此推知这两方法必定是在该类的父类中已实现了，在 StandardServer 类的父类 LifecycleMBeanBase 类的父类 LifecycleBase 类里面终于找到了这两个方法的实现，下面先来看下 init 方法：



```Java
     1	    @Override
     2	    public final synchronized void init() throws LifecycleException {
     3	        if (!state.equals(LifecycleState.NEW)) {
     4	            invalidTransition(Lifecycle.BEFORE_INIT_EVENT);
     5	        }
     6	        setStateInternal(LifecycleState.INITIALIZING, null, false);
     7
     8	        try {
     9	            initInternal();
    10	        } catch (Throwable t) {
    11	            ExceptionUtils.handleThrowable(t);
    12	            setStateInternal(LifecycleState.FAILED, null, false);
    13	            throw new LifecycleException(
    14	                    sm.getString("lifecycleBase.initFail",toString()), t);
    15	        }
    16
    17	        setStateInternal(LifecycleState.INITIALIZED, null, false);
    18	    }
    19
    20
    21	    protected abstract void initInternal() throws LifecycleException;

```

先将干扰程序阅读视线的 setStateInternal 方法调用忽略掉（下一篇文章会详细讲解该方法），发现这里面就做了一件事情，调用了一下接下来定义的抽象方法 initInternal()（第 21 行）。实际上看下 LifecycleBase 的实现类就会发现，所有的组件类几乎都继承了 LifecycleBase 类，所以这些组件类一般只会有 initInternal 方法的定义。（这里所说的组件类就是前面我们分析 Digester 解析时发现的 StandardServer、StandardService、StandardEngine、StandardHost、StandardContext 等类）

这里所说的组件可以将其理解为我们最开始分析 server.xml 时 xml 文件里的各个节点，父子关系也即 xml 文件里的父子节点。浏览下 LifecycleBase 的子类就会发现节点的实现类都是这个类的子类（记住这点，后面会提到）。

![img](https://user-gold-cdn.xitu.io/2018/1/28/1613b7e99ebc0a4f?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 同样分析 start 方法：



```Java
     1	    /**
     2	     * {@inheritDoc}
     3	     */
     4	    @Override
     5	    public final synchronized void start() throws LifecycleException {
     6
     7	        if (LifecycleState.STARTING_PREP.equals(state) ||
     8	                LifecycleState.STARTING.equals(state) ||
     9	                LifecycleState.STARTED.equals(state)) {
    10
    11	            if (log.isDebugEnabled()) {
    12	                Exception e = new LifecycleException();
    13	                log.debug(sm.getString("lifecycleBase.alreadyStarted",
    14	                        toString()), e);
    15	            } else if (log.isInfoEnabled()) {
    16	                log.info(sm.getString("lifecycleBase.alreadyStarted",
    17	                        toString()));
    18	            }
    19
    20	            return;
    21	        }
    22
    23	        if (state.equals(LifecycleState.NEW)) {
    24	            init();
    25	        } else if (state.equals(LifecycleState.FAILED)){
    26	            stop();
    27	        } else if (!state.equals(LifecycleState.INITIALIZED) &&
    28	                !state.equals(LifecycleState.STOPPED)) {
    29	            invalidTransition(Lifecycle.BEFORE_START_EVENT);
    30	        }
    31
    32	        setStateInternal(LifecycleState.STARTING_PREP, null, false);
    33
    34	        try {
    35	            startInternal();
    36	        } catch (Throwable t) {
    37	            ExceptionUtils.handleThrowable(t);
    38	            setStateInternal(LifecycleState.FAILED, null, false);
    39	            throw new LifecycleException(
    40	                    sm.getString("lifecycleBase.startFail",toString()), t);
    41	        }
    42
    43	        if (state.equals(LifecycleState.FAILED) ||
    44	                state.equals(LifecycleState.MUST_STOP)) {
    45	            stop();
    46	        } else {
    47	            // Shouldn't be necessary but acts as a check that sub-classes are
    48	            // doing what they are supposed to.
    49	            if (!state.equals(LifecycleState.STARTING)) {
    50	                invalidTransition(Lifecycle.AFTER_START_EVENT);
    51	            }
    52
    53	            setStateInternal(LifecycleState.STARTED, null, false);
    54	        }
    55	    }
    56
    57
    58	    /**
    59	     * Sub-classes must ensure that the state is changed to
    60	     * {@link LifecycleState#STARTING} during the execution of this method.
    61	     * Changing state will trigger the {@link Lifecycle#START_EVENT} event.
    62	     *
    63	     * If a component fails to start it may either throw a
    64	     * {@link LifecycleException} which will cause it's parent to fail to start
    65	     * or it can place itself in the error state in which case {@link #stop()}
    66	     * will be called on the failed component but the parent component will
    67	     * continue to start normally.
    68	     *
    69	     * @throws LifecycleException
    70	     */
    71	    protected abstract void startInternal() throws LifecycleException;

```

第 7 到 21 行是 start 功能的前置校验，这里如果发现 start 方法已经调用过了，将会记录日志并直接返回。第 23 到 30 行如果发现 start 放的需要做的前置方法没有调用完，或者调用出错，将会先调用这些前置方法。第 32 行暂时先不管，不影响程序阅读，第 35 行是该方法的实质，将会调用本类中定义的抽象方法 startInternal()（第 71 行）。下面的代码同上述一样，都是一些 start 方法调用过程中可能出现的错误的错误处理。

从以上 init 和 start 方法的定义可以看到这两个方法最终将会调用子类中定义的 initInternal 和 startInternal 。

接回本文开头，一开始在找 StandardServer 类中 init 和 start 方法的定义，看完了上面的分析发现最终会调用 StandardServer 类的 initInternal 和 startInternal 两个方法。那这两个方法里面干了些什么？

initInternal 方法：

```Java
     1	    /**
     2	     * Invoke a pre-startup initialization. This is used to allow connectors
     3	     * to bind to restricted ports under Unix operating environments.
     4	     */
     5	    @Override
     6	    protected void initInternal() throws LifecycleException {
     7
     8	        super.initInternal();
     9
    10	        // Register global String cache
    11	        // Note although the cache is global, if there are multiple Servers
    12	        // present in the JVM (may happen when embedding) then the same cache
    13	        // will be registered under multiple names
    14	        onameStringCache = register(new StringCache(), "type=StringCache");
    15
    16	        // Register the MBeanFactory
    17	        MBeanFactory factory = new MBeanFactory();
    18	        factory.setContainer(this);
    19	        onameMBeanFactory = register(factory, "type=MBeanFactory");
    20
    21	        // Register the naming resources
    22	        globalNamingResources.init();
    23
    24	        // Populate the extension validator with JARs from common and shared
    25	        // class loaders
    26	        if (getCatalina() != null) {
    27	            ClassLoader cl = getCatalina().getParentClassLoader();
    28	            // Walk the class loader hierarchy. Stop at the system class loader.
    29	            // This will add the shared (if present) and common class loaders
    30	            while (cl != null && cl != ClassLoader.getSystemClassLoader()) {
    31	                if (cl instanceof URLClassLoader) {
    32	                    URL[] urls = ((URLClassLoader) cl).getURLs();
    33	                    for (URL url : urls) {
    34	                        if (url.getProtocol().equals("file")) {
    35	                            try {
    36	                                File f = new File (url.toURI());
    37	                                if (f.isFile() &&
    38	                                        f.getName().endsWith(".jar")) {
    39	                                    ExtensionValidator.addSystemResource(f);
    40	                                }
    41	                            } catch (URISyntaxException e) {
    42	                                // Ignore
    43	                            } catch (IOException e) {
    44	                                // Ignore
    45	                            }
    46	                        }
    47	                    }
    48	                }
    49	                cl = cl.getParent();
    50	            }
    51	        }
    52	        // Initialize our defined Services
    53	        for (int i = 0; i < services.length; i++) {
    54	            services[i].init();
    55	        }
    56	    }

```

init 方法里面做了好几件事情，牵涉的话题比较多，这里重点关注最后第 53 到 55 行的代码，这里将循环调用 Server 类里内置的 Service 数组的 init 方法。

startInternal 方法：

```Java
     1	    /**
     2	     * Start nested components ({@link Service}s) and implement the requirements
     3	     * of {@link org.apache.catalina.util.LifecycleBase#startInternal()}.
     4	     *
     5	     * @exception LifecycleException if this component detects a fatal error
     6	     *  that prevents this component from being used
     7	     */
     8	    @Override
     9	    protected void startInternal() throws LifecycleException {
    10
    11	        fireLifecycleEvent(CONFIGURE_START_EVENT, null);
    12	        setState(LifecycleState.STARTING);
    13
    14	        globalNamingResources.start();
    15
    16	        // Start our defined Services
    17	        synchronized (services) {
    18	            for (int i = 0; i < services.length; i++) {
    19	                services[i].start();
    20	            }
    21	        }
    22	    }

```

重点关注第 17 到 21 行，同上一段所分析的代码类似，将循环调用 Sever 类里内置的 Service 数组的 start 方法。

那么这里提到的 Service 的对象到底是什么？

上篇文章分析 Digester 时提到“**经过对 xml 文件的解析将会产生 org.apache.catalina.core.StandardServer、org.apache.catalina.core.StandardService、org.apache.catalina.connector.Connector、org.apache.catalina.core.StandardEngine、org.apache.catalina.core.StandardHost、org.apache.catalina.core.StandardContext 等等一系列对象，这些对象从前到后前一个包含后一个对象的引用（一对一或一对多的关系）。**”

所以正常情况下这里的 Service 将会是 org.apache.catalina.core.StandardService 的对象（该代码见 org.apache.catalina.startup.Catalina 类的 339 到 341 行）。

所以按上面的分析，接下来将会调用 StandardService 类的 init 和 start 方法，实际上这个类也是 LifecycleBase 类的子类，所以最终的也会调用本类中的 initInternal 和 startInternal 方法。

initInternal 方法：

```Java
     1	    /**
     2	     * Invoke a pre-startup initialization. This is used to allow connectors
     3	     * to bind to restricted ports under Unix operating environments.
     4	     */
     5	    @Override
     6	    protected void initInternal() throws LifecycleException {
     7
     8	        super.initInternal();
     9
    10	        if (container != null) {
    11	            container.init();
    12	        }
    13
    14	        // Initialize any Executors
    15	        for (Executor executor : findExecutors()) {
    16	            if (executor instanceof LifecycleMBeanBase) {
    17	                ((LifecycleMBeanBase) executor).setDomain(getDomain());
    18	            }
    19	            executor.init();
    20	        }
    21
    22	        // Initialize our defined Connectors
    23	        synchronized (connectors) {
    24	            for (Connector connector : connectors) {
    25	                try {
    26	                    connector.init();
    27	                } catch (Exception e) {
    28	                    String message = sm.getString(
    29	                            "standardService.connector.initFailed", connector);
    30	                    log.error(message, e);
    31
    32	                    if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE"))
    33	                        throw new LifecycleException(message);
    34	                }
    35	            }
    36	        }
    37	    }

```

这里将会调用 Service 下的各类子组件中的 init 方法。

startInternal 方法：

```Java
     1	    /**
     2	     * Start nested components ({@link Executor}s, {@link Connector}s and
     3	     * {@link Container}s) and implement the requirements of
     4	     * {@link org.apache.catalina.util.LifecycleBase#startInternal()}.
     5	     *
     6	     * @exception LifecycleException if this component detects a fatal error
     7	     *  that prevents this component from being used
     8	     */
     9	    @Override
    10	    protected void startInternal() throws LifecycleException {
    11
    12	        if(log.isInfoEnabled())
    13	            log.info(sm.getString("standardService.start.name", this.name));
    14	        setState(LifecycleState.STARTING);
    15
    16	        // Start our defined Container first
    17	        if (container != null) {
    18	            synchronized (container) {
    19	                container.start();
    20	            }
    21	        }
    22
    23	        synchronized (executors) {
    24	            for (Executor executor: executors) {
    25	                executor.start();
    26	            }
    27	        }
    28
    29	        // Start our defined Connectors second
    30	        synchronized (connectors) {
    31	            for (Connector connector: connectors) {
    32	                try {
    33	                    // If it has already failed, don't try and start it
    34	                    if (connector.getState() != LifecycleState.FAILED) {
    35	                        connector.start();
    36	                    }
    37	                } catch (Exception e) {
    38	                    log.error(sm.getString(
    39	                            "standardService.connector.startFailed",
    40	                            connector), e);
    41	                }
    42	            }
    43	        }
    44	    }

```

同理，将会调用 service 下各类子组件中的 start 方法。

StandardService 的子容器是 StandardEngine ，看下 StandardEngine 的 startInternal 方法：

```Java
     1	    protected synchronized void startInternal() throws LifecycleException {
     2
     3	        // Log our server identification information
     4	        if(log.isInfoEnabled())
     5	            log.info( "Starting Servlet Engine: " + ServerInfo.getServerInfo());
     6
     7	        // Standard container startup
     8	        super.startInternal();
     9	    }

```

这里不同于上面两级容器的实现，而是直接调用了父类的 startInternal 方法：

```Java
     1	    protected synchronized void startInternal() throws LifecycleException {
     2
     3	        // Start our subordinate components, if any
     4	        if ((loader != null) && (loader instanceof Lifecycle))
     5	            ((Lifecycle) loader).start();
     6	        logger = null;
     7	        getLogger();
     8	        if ((manager != null) && (manager instanceof Lifecycle))
     9	            ((Lifecycle) manager).start();
    10	        if ((cluster != null) && (cluster instanceof Lifecycle))
    11	            ((Lifecycle) cluster).start();
    12	        Realm realm = getRealmInternal();
    13	        if ((realm != null) && (realm instanceof Lifecycle))
    14	            ((Lifecycle) realm).start();
    15	        if ((resources != null) && (resources instanceof Lifecycle))
    16	            ((Lifecycle) resources).start();
    17
    18	        // Start our child containers, if any
    19	        Container children[] = findChildren();
    20	        List> results = new ArrayList>();
    21	        for (int i = 0; i < children.length; i++) {
    22	            results.add(startStopExecutor.submit(new StartChild(children[i])));
    23	        }
    24
    25	        boolean fail = false;
    26	        for (Future result : results) {
    27	            try {
    28	                result.get();
    29	            } catch (Exception e) {
    30	                log.error(sm.getString("containerBase.threadedStartFailed"), e);
    31	                fail = true;
    32	            }
    33
    34	        }
    35	        if (fail) {
    36	            throw new LifecycleException(
    37	                    sm.getString("containerBase.threadedStartFailed"));
    38	        }
    39
    40	        // Start the Valves in our pipeline (including the basic), if any
    41	        if (pipeline instanceof Lifecycle)
    42	            ((Lifecycle) pipeline).start();
    43
    44
    45	        setState(LifecycleState.STARTING);
    46
    47	        // Start our thread
    48	        threadStart();
    49
    50	    }

```

第 19 到 34 行即启动当前容器下的子容器的代码，这里采用了分线程分别启动的方式。核心的调用子容器的 start 方法的代码在 StartChild 类的 call 方法中：

```Java
     1	    private static class StartChild implements Callable {
     2
     3	        private Container child;
     4
     5	        public StartChild(Container child) {
     6	            this.child = child;
     7	        }
     8
     9	        @Override
    10	        public Void call() throws LifecycleException {
    11	            child.start();
    12	            return null;
    13	        }
    14	    }

```

这里使用了JDK 5 的执行线程的方式，不理解的话请参考相关文档说明。

StandardHost 中的 startInternal 与 StandardEngine 类似，这里不再赘述，建议有兴趣的朋友逐一分析 Review 一遍，碰到组件里面嵌套的变量不知道具体实现类的就从上篇文章里面提到的 createStartDigester 那边开始找起，这里不能直接找到的就在里面提到的 new *RuleSet 的 addRuleInstances 方法里面找。通过这种调用将会最终执行完所有在 server.xml 里配置的节点的实现类中 initInternal 和 startInternal 方法。

![img](https://user-gold-cdn.xitu.io/2018/1/28/1613b89c94aa8417?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 最后上面提到的 org.apache.catalina.core.StandardServer、org.apache.catalina.core.StandardService、org.apache.catalina.connector.Connector、org.apache.catalina.core.StandardEngine、org.apache.catalina.core.StandardHost、org.apache.catalina.core.StandardContext 等等组件的这两个方法都会调用到。



就这样，Tomcat 7 在内存中为这一连串组件产生对象，建立对象调用关系，调用它们各自的初始化和启动方法，启动的总体过程就介绍完了，这些对象 start 之后将会响应客户端的请求，为用户服务了。当然，这里还没有涉及到对于具体的发布到 tomcat 里面的没有应用的载入过程，web 应用中配置的 servlet、filter、listener 等的载入、启动服务过程，浏览器发起的一个请求如何经过 Tomcat 内各组件的流转调用到具体应用里去的，这一系列问题都还没谈到。因为 Tomcat 本身庞大繁杂，需要找一个视角切入进去，为了叙述的简单，先从整体上对 Tomcat 内包含的各组件产生机制有一个大体轮廓的了解，这样为后面的介绍提供一个统一的背景。

下一篇文章将分析本文开头遗留的一个问题 —— setStateInternal 方法的作用，以及 Tomcat 中的 Lifecycle 实现原理。