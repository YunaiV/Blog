title: Tomcat 7 启动分析（五）Lifecycle 机制和实现原理
date: 2018-01-07
tag: 
categories: Tomcat
permalink: Tomcat/yuliu/Start-analysis-5-Lifecycle
author: 预流
from_url: https://juejin.im/post/5a6d73a36fb9a01cba42d1d7
wechat_url: 

-------

摘要: 原创出处 https://juejin.im/user/59356fea570c35005b5fc55b/posts 「预流」欢迎转载，保留摘要，谢谢！


-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

在[上篇文章分析 Tomcat 7 的各组件的 init、start 方法](http://www.iocoder.cn/Tomcat/yuliu/Start-analysis-4-components-int-and-start-method)时经常会看到有一个 setStateInternal 方法的调用，在查看 LifecycleBase 类及其它各组件的源码时会在多处看到这个方法的调用，这篇文章就来说说这方法，以及与这个方法相关的 Tomcat 的 Lifecycle 机制和实现原理。

接上文里谈到了 Tomcat 7 的各组件的父类 LifecycleBase 类，该类实现了接口 org.apache.catalina.Lifecycle，下面是这个接口里定义的常量和方法：

![img](https://user-gold-cdn.xitu.io/2018/1/28/1613b8ca65f0ac1e?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 细心的读者会发现，上篇文章里提到的 init 和 start 方法实际上是在这个接口里面定义好的，也正因为有各组件最终都会实现这个接口作为前提条件，所以才能支持组件内部的 initInternal、startInternal 方法内对于子组件（组件里面嵌套的子组件都是以接口的形式定义的，但这些接口都会以 Lifecycle 作为父接口）的 init 和 start 方法的调用。通过这种方式，只要调用了最外层的 Server 组件的 init 和 start 方法，就可以将 Tomcat 内部的各级子组件初始化和启动起来。我叫这种方式为链式调用。实际上关于 Tomcat 的关闭机制也是通过这种方式一步步调用各层组件的 stop 方法的。这里不再展开叙述，留待读者自己研究研究吧。



Lifecycle 接口中的这些字符串常量定义主要用于事件类型的定义，先按下不表，文章后面会提到。

重点看下面三个方法：

```Java
    /**
     * Add a LifecycleEvent listener to this component.
     *
     * @param listener The listener to add
     */
    public void addLifecycleListener(LifecycleListener listener);//给该组将添加一个监听器


    /**
     * Get the life cycle listeners associated with this life cycle. If this
     * component has no listeners registered, a zero-length array is returned.
     */
    public LifecycleListener[] findLifecycleListeners();//获取该组件所有已注册的监听器


    /**
     * Remove a LifecycleEvent listener from this component.
     *
     * @param listener The listener to remove
     */
    public void removeLifecycleListener(LifecycleListener listener);//删除该组件中的一个监听器

```

这三个方法的作用在代码的注释里简要说明了一下。这三个方法涉及 org.apache.catalina.LifecycleListener 接口，那么就看下这个接口的定义：

```Java
public interface LifecycleListener {


    /**
     * Acknowledge the occurrence of the specified event.
     *
     * @param event LifecycleEvent that has occurred
     */
    public void lifecycleEvent(LifecycleEvent event);


}

```

如此简单，只有一个方法，这个方法用作某个事件（ org.apache.catalina.LifecycleEvent ）产生时通知当前监听器的实现类，具体针对该事件如何处理由监听器实现类自己决定。

看下 LifecycleEvent 的实现：

```Java
public final class LifecycleEvent extends EventObject {

    private static final long serialVersionUID = 1L;


    // ----------------------------------------------------------- Constructors

    /**
     * Construct a new LifecycleEvent with the specified parameters.
     *
     * @param lifecycle Component on which this event occurred
     * @param type Event type (required)
     * @param data Event data (if any)
     */
    public LifecycleEvent(Lifecycle lifecycle, String type, Object data) {

        super(lifecycle);
        this.type = type;
        this.data = data;
    }


    // ----------------------------------------------------- Instance Variables


    /**
     * The event data associated with this event.
     */
    private Object data = null;


    /**
     * The event type this instance represents.
     */
    private String type = null;


    // ------------------------------------------------------------- Properties


    /**
     * Return the event data of this event.
     */
    public Object getData() {

        return (this.data);

    }


    /**
     * Return the Lifecycle on which this event occurred.
     */
    public Lifecycle getLifecycle() {

        return (Lifecycle) getSource();

    }


    /**
     * Return the event type of this event.
     */
    public String getType() {

        return (this.type);

    }


}

```

这个类也很简单，data 和 type 作为类的内置实例变量，唯一特别是使用了 jdk 内置的 java.util.EventObject 作为父类来支持事件定义，这里在事件构造函数中将 org.apache.catalina.Lifecycle 类的实例 lifecycle 作为事件源，保存 lifecycle 对象的引用，并提供了 getLifecycle 方法返回这个引用。

那么 Tomcat 中是如何实现关于这些事件的监听以及通知的呢？

在本文开头提到 的LifecycleBase 类中第 47 行定义了一个实例变量 lifecycle ，正是通过该变量来注册组件上定义的各类监听器的。留心一下 lifecycle 这个实例变量，它并不是 org.apache.catalina.Lifecycle 类的实例，而是 org.apache.catalina.util.LifecycleSupport 类的实例。正是这个工具类提供了事件监听和事件通知的功能。

先看下实际代码中是如何给组件发布时间通知的，看下前面文章中曾经提到过的 org.apache.catalina.core.StandardServer 类的 startInternal 方法：

```Java
     1	    protected void startInternal() throws LifecycleException {
     2
     3	        fireLifecycleEvent(CONFIGURE_START_EVENT, null);
     4	        setState(LifecycleState.STARTING);
     5
     6	        globalNamingResources.start();
     7
     8	        // Start our defined Services
     9	        synchronized (services) {
    10	            for (int i = 0; i < services.length; i++) {
    11	                services[i].start();
    12	            }
    13	        }
    14	    }

```

我们前面已经分析过第 9 到 13 行代码，这里看下第 3 行，它调用了父类 org.apache.catalina.util.LifecycleBase 里的 fireLifecycleEvent 方法，这里的`CONFIGURE_START_EVENT`就是本文最开始 Lifecycle 接口中定义的常量，这里表示发布了一个 start 配置事件。

org.apache.catalina.util.LifecycleBase 类中的 fireLifecycleEvent 方法里调用的是 org.apache.catalina.util.LifecycleSupport 类 fireLifecycleEvent 方法，该方法代码如下：

```
    public void fireLifecycleEvent(String type, Object data) {

        LifecycleEvent event = new LifecycleEvent(lifecycle, type, data);
        LifecycleListener interested[] = listeners;
        for (int i = 0; i < interested.length; i++)
            interested[i].lifecycleEvent(event);

    }

```

通过传进来的两个参数构造一个 LifecycleEvent 对象，然后向注册到组件中的所有监听器发布这个新构造的事件对象。

这里有个疑问，到底什么时候向组件里注册监听器的呢？

还是以 StandardServer 举例，在前面讲 Digester 的使用时，org.apache.catalina.startup.Catalina 类的 createStartDigester 方法有这么一段代码：

```Java
     1	        // Configure the actions we will be using
     2	        digester.addObjectCreate("Server",
     3	                                 "org.apache.catalina.core.StandardServer",
     4	                                 "className");
     5	        digester.addSetProperties("Server");
     6	        digester.addSetNext("Server",
     7	                            "setServer",
     8	                            "org.apache.catalina.Server");
     9
    10	        digester.addObjectCreate("Server/GlobalNamingResources",
    11	                                 "org.apache.catalina.deploy.NamingResources");
    12	        digester.addSetProperties("Server/GlobalNamingResources");
    13	        digester.addSetNext("Server/GlobalNamingResources",
    14	                            "setGlobalNamingResources",
    15	                            "org.apache.catalina.deploy.NamingResources");
    16
    17	        digester.addObjectCreate("Server/Listener",
    18	                                 null, // MUST be specified in the element
    19	                                 "className");
    20	        digester.addSetProperties("Server/Listener");
    21	        digester.addSetNext("Server/Listener",
    22	                            "addLifecycleListener",
    23	                            "org.apache.catalina.LifecycleListener");

```

第 17 到 24 行，将调用 org.apache.catalina.core.StandardServer 类的 addLifecycleListener 方法，将根据 server.xml 中配置的 Server 节点下的 Listener 节点所定义的 className 属性构造对象实例，并作为 addLifecycleListener 方法的入参。所有的监听器都会实现上面提到的 org.apache.catalina.LifecycleListener 接口。Server 节点下的 Listener 节点有好几个，这里以 org.apache.catalina.core.JasperListener 举例。

在构造完 org.apache.catalina.core.JasperListener 类的对象之后，调用 addLifecycleListener 方法，这个方法并没有直接在 org.apache.catalina.core.StandardServer 类中定义，而是在它的父类 org.apache.catalina.util.LifecycleBase 中：

```Java
    @Override
    public void addLifecycleListener(LifecycleListener listener) {
        lifecycle.addLifecycleListener(listener);
    }

```

这里调用的是前述的 org.apache.catalina.util.LifecycleSupport 类的 addLifecycleListener 方法：

```Java
    /**
     * Add a lifecycle event listener to this component.
     *
     * @param listener The listener to add
     */
    public void addLifecycleListener(LifecycleListener listener) {

      synchronized (listenersLock) {
          LifecycleListener results[] =
            new LifecycleListener[listeners.length + 1];
          for (int i = 0; i < listeners.length; i++)
              results[i] = listeners[i];
          results[listeners.length] = listener;
          listeners = results;
      }

    }

```

LifecycleSupport 作为一个工具类，内部保存了一个监听器对象实例数组，见该类的第 68 行：

```Java
/**
 * The set of registered LifecycleListeners for event notifications.
 */
private LifecycleListener listeners[] = new LifecycleListener[0];

```

上面的 addLifecycleListener 方法内部实现的是同步给该数组增加一个监听器对象。

看到这里应该大体明白 Tomcat 中的 Lifecycle 是怎么回事了，总的来说就是通过一个工具类 LifecycleSupport ，调用该类的 addLifecycleListener 方法增加监听器，需要发布事件时还是调用该工具类的 fireLifecycleEvent 方法，将事件发布给组件上注册的所有监听器，由监听器内部实现来决定是否处理该事件。

拿前面看到的一个监听器 org.apache.catalina.core.JasperListener 举例：

```Java
     1	public class JasperListener
     2	    implements LifecycleListener {
     3
     4	    private static final Log log = LogFactory.getLog(JasperListener.class);
     5
     6	    /**
     7	     * The string manager for this package.
     8	     */
     9	    protected static final StringManager sm =
    10	        StringManager.getManager(Constants.Package);
    11
    12
    13	    // ---------------------------------------------- LifecycleListener Methods
    14
    15
    16	    /**
    17	     * Primary entry point for startup and shutdown events.
    18	     *
    19	     * @param event The event that has occurred
    20	     */
    21	    @Override
    22	    public void lifecycleEvent(LifecycleEvent event) {
    23
    24	        if (Lifecycle.BEFORE_INIT_EVENT.equals(event.getType())) {
    25	            try {
    26	                // Set JSP factory
    27	                Class.forName("org.apache.jasper.compiler.JspRuntimeContext",
    28	                              true,
    29	                              this.getClass().getClassLoader());
    30	            } catch (Throwable t) {
    31	                ExceptionUtils.handleThrowable(t);
    32	                // Should not occur, obviously
    33	                log.warn("Couldn't initialize Jasper", t);
    34	            }
    35	            // Another possibility is to do directly:
    36	            // JspFactory.setDefaultFactory(new JspFactoryImpl());
    37	        }
    38
    39	    }
    40
    41
    42	}

```

重点关注来自接口的 lifecycleEvent 方法的实现，可以看到这个监听器只关心事件类型为`BEFORE_INIT_EVENT`的事件，如果发布了该事件，才会做后续处理（这里会产生一个 org.apache.jasper.compiler.JspRuntimeContext 对象）。

Lifecycle 相关类 UML 关系图：

![img](https://user-gold-cdn.xitu.io/2018/1/28/1613b9561b7ea1e6?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 如果对设计模式比较熟悉的话会发现 Tomcat 的 Lifecycle 使用的是观察者模式：LifecycleListener 代表的是抽象观察者，它定义一个 lifecycleEvent 方法，而实现该接口的监听器是作为具体的观察者。Lifecycle  接口代表的是抽象主题，它定义了管理观察者的方法和它要所做的其它方法。而各组件代表的是具体主题，它实现了抽象主题的所有方法。通常会由具体主题保存对具体观察者对象有用的内部状态；在这种内部状态改变时给其观察者发出一个通知。Tomcat 对这种模式做了改进，增加了另外两个工具类：LifecycleSupport、LifecycleEvent ，它们作为辅助类扩展了观察者的功能。LifecycleEvent 中定义了事件类别，不同的事件在具体观察者中可区别处理，更加灵活。LifecycleSupport 类代理了所有具体主题对观察者的管理，将这个管理抽出来统一实现，以后如果修改只要修改 LifecycleSupport 类就可以了，不需要去修改所有具体主题，因为所有具体主题的对观察者的操作都被代理给 LifecycleSupport 类了。



事件的发布使用的是推模式，即每发布一个事件都会通知主题的所有具体观察者，由各观察者再来决定是否需要对该事件进行后续处理。

下面再来看看本文一开头所说的 setStateInternal 方法，以 org.apache.catalina.core.StandardServer 类为例，上面看到的 startInternal 方法中第 4 行：`setState(LifecycleState.STARTING)`;

它调用了父类 org.apache.catalina.util.LifecycleBase 中的 setState 方法：

```Java
    /**
     * Provides a mechanism for sub-classes to update the component state.
     * Calling this method will automatically fire any associated
     * {@link Lifecycle} event. It will also check that any attempted state
     * transition is valid for a sub-class.
     *
     * @param state The new state for this component
     */
    protected synchronized void setState(LifecycleState state)
            throws LifecycleException {
        setStateInternal(state, null, true);
    }

```

在这个类里面调用本类的一个同步方法 setStateInternal ：

```Java
     1	    private synchronized void setStateInternal(LifecycleState state,
     2	            Object data, boolean check) throws LifecycleException {
     3
     4	        if (log.isDebugEnabled()) {
     5	            log.debug(sm.getString("lifecycleBase.setState", this, state));
     6	        }
     7
     8	        if (check) {
     9	            // Must have been triggered by one of the abstract methods (assume
    10	            // code in this class is correct)
    11	            // null is never a valid state
    12	            if (state == null) {
    13	                invalidTransition("null");
    14	                // Unreachable code - here to stop eclipse complaining about
    15	                // a possible NPE further down the method
    16	                return;
    17	            }
    18
    19	            // Any method can transition to failed
    20	            // startInternal() permits STARTING_PREP to STARTING
    21	            // stopInternal() permits STOPPING_PREP to STOPPING and FAILED to
    22	            // STOPPING
    23	            if (!(state == LifecycleState.FAILED ||
    24	                    (this.state == LifecycleState.STARTING_PREP &&
    25	                            state == LifecycleState.STARTING) ||
    26	                    (this.state == LifecycleState.STOPPING_PREP &&
    27	                            state == LifecycleState.STOPPING) ||
    28	                    (this.state == LifecycleState.FAILED &&
    29	                            state == LifecycleState.STOPPING))) {
    30	                // No other transition permitted
    31	                invalidTransition(state.name());
    32	            }
    33	        }
    34
    35	        this.state = state;
    36	        String lifecycleEvent = state.getLifecycleEvent();
    37	        if (lifecycleEvent != null) {
    38	            fireLifecycleEvent(lifecycleEvent, data);
    39	        }
    40	    }

```

重点关注第 35 到 39 行，第 35 行将入参 LifecycleState 实例赋值给本类中的实例变量保存起来，第 36 行取出 LifecycleState 实例的 LifecycleEvent 事件，如果该事件非空，则调用 fireLifecycleEvent 方法发布该事件。

既然看到了 LifecycleState 类，就看下 LifecycleState 类的定义：

```Java
     1	public enum LifecycleState {
     2	    NEW(false, null),
     3	    INITIALIZING(false, Lifecycle.BEFORE_INIT_EVENT),
     4	    INITIALIZED(false, Lifecycle.AFTER_INIT_EVENT),
     5	    STARTING_PREP(false, Lifecycle.BEFORE_START_EVENT),
     6	    STARTING(true, Lifecycle.START_EVENT),
     7	    STARTED(true, Lifecycle.AFTER_START_EVENT),
     8	    STOPPING_PREP(true, Lifecycle.BEFORE_STOP_EVENT),
     9	    STOPPING(false, Lifecycle.STOP_EVENT),
    10	    STOPPED(false, Lifecycle.AFTER_STOP_EVENT),
    11	    DESTROYING(false, Lifecycle.BEFORE_DESTROY_EVENT),
    12	    DESTROYED(false, Lifecycle.AFTER_DESTROY_EVENT),
    13	    FAILED(false, null),
    14	    MUST_STOP(true, null),
    15	    MUST_DESTROY(false, null);
    16
    17	    private final boolean available;
    18	    private final String lifecycleEvent;
    19
    20	    private LifecycleState(boolean available, String lifecycleEvent) {
    21	        this.available = available;
    22	        this.lifecycleEvent = lifecycleEvent;
    23	    }
    24
    25	    /**
    26	     * May the public methods other than property getters/setters and lifecycle
    27	     * methods be called for a component in this state? It returns
    28	     * <code>true</code> for any component in any of the following states:
    29	     * <ul>
    30	     * <li>{@link #STARTING}</li>
    31	     * <li>{@link #STARTED}</li>
    32	     * <li>{@link #STOPPING_PREP}</li>
    33	     * <li>{@link #MUST_STOP}</li>
    34	     * </ul>
    35	     */
    36	    public boolean isAvailable() {
    37	        return available;
    38	    }
    39
    40	    /**
    41	     *
    42	     */
    43	    public String getLifecycleEvent() {
    44	        return lifecycleEvent;
    45	    }
    46	}

```

这个类在之前的 Tomcat 4 和 Tomcat 5 中都没有看到，可能是 Tomcat 7 里面新定义的吧，就是一个枚举，内嵌了两个实例变量，一个布尔值表示是否可用，一个字符串表示是事件类型，看已经定义的枚举值里面发现这个字符串要么不设值，要么就是 Lifecycle 类中定义好的字符串常量。这个类实际上就是对 Lifecycle 类中定义好的字符串常量做了另外一层封装。

再说回开头在各组件代码中经常会看到的 setStateInternal 方法的调用，实际上就是向该组件中已注册的监听器发布一个事件。