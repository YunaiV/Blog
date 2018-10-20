title: Spring 5 源码解析 —— Spring 框架中的事件和监听器
date: 2018-01-22
tag: 
categories: Spring
permalink: Spring/ApplicationContextEvent
author: 一叶知秋
from_url: https://muyinchen.github.io/2017/09/27/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— Spring%E6%A1%86%E6%9E%B6%E4%B8%AD%E7%9A%84%E4%BA%8B%E4%BB%B6%E5%92%8C%E7%9B%91%E5%90%AC%E5%99%A8/
wechat_url: 

-------

摘要: 原创出处 https://muyinchen.github.io/2017/09/27/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— Spring%E6%A1%86%E6%9E%B6%E4%B8%AD%E7%9A%84%E4%BA%8B%E4%BB%B6%E5%92%8C%E7%9B%91%E5%90%AC%E5%99%A8/ 「一叶知秋」欢迎转载，保留摘要，谢谢！

- [事件驱动编程](http://www.iocoder.cn/Spring/ApplicationContextEvent/)
- [Spring中的Events](http://www.iocoder.cn/Spring/ApplicationContextEvent/)
- [在Spring中实现一个简单的监听器](http://www.iocoder.cn/Spring/ApplicationContextEvent/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

事件和平时所用的回调思想在与GUI(JavaScript，Swing)相关的技术中非常流行。而在Web应用程序的服务器端，我们很少去直接使用。但这并不意味着我们无法在服务端去实现一个面向事件的体系结构。

在本文中，我们将重点介绍Spring框架中的事件处理。首先，会先介绍下事件驱动编程这个概念。接着，我们会将精力放在专门用于Spring框架中的事件处理之上。然后我们会看到实现事件调度和监听的主要方法。最后，我们将在Spring应用程序中展示如何使用基本的监听器。

## 事件驱动编程

在开始讨论事件驱动编程的编程方面之前，先来说一个场景，用来帮助大家更好地理解`event-driven`这个概念。在一个地方只有两个卖衣服的商店A和B.在A店中，我们的消费者需要一个一个的接受服务，即，同一时间只有一个客户可以购物。在B店里，可以允许几个客户同时进行购物，当有客户需要卖家的帮助时，他需要举起他的右手示意一下。卖家看到后会来找他，帮助其做出更好的选择。关于事件驱动(`event-driven`)编程这个概念通过上述场景描述总结后就是：通过做一些动作来作为对一些行为的回应。

如上所见，事件驱动的编程(也称为基于事件的编程)是**基于对接收到的信号的反应**的**编程形式**。这些信号必须以某种形式来传输信息。举一个简单例子：`点击按钮`。我们将这些信号称为**事件**。这些事件可以通过用户操作(鼠标点击，触摸)或程序条件执行触发(例如：一个元素的加载结束可以启动另一个操作)来产生。

为了更好地了解，请看以下示例，模仿用户在GUI界面中的操作：

```java
public class EventBasedTest {

  @Test
  public void test() {
    Mouse mouse = new Mouse();
    mouse.addListener(new MouseListener() {
      @Override
      public void onClick(Mouse mouse) {
        System.out.println("Listener#1 called");
        mouse.addListenerCallback();
      }
    });
    mouse.addListener(new MouseListener() {
      @Override
      public void onClick(Mouse mouse) {
        System.out.println("Listener#2 called");
        mouse.addListenerCallback();
      }
    });
    mouse.click();
    assertTrue("2 listeners should be invoked but only "+mouse.getListenerCallbacks()+" were", mouse.getListenerCallbacks() == 2);
  }
}


class Mouse {
  private List<mouselistener> listeners = new ArrayList<mouselistener>();
  private int listenerCallbacks = 0;

  public void addListenerCallback() {
    listenerCallbacks++;
  }

  public int getListenerCallbacks() {
    return listenerCallbacks;
  }

  public void addListener(MouseListener listener) {
    listeners.add(listener);
  }

  public void click() {
    System.out.println("Clicked !");
    for (MouseListener listener : listeners) {
      listener.onClick(this);
    }
  }
}

interface MouseListener {
  public void onClick(Mouse source);
}
```

打印输出如下所示：

```
Clicked !
Listener#1 called
Listener#2 called
```

## Spring中的Events

Spring基于实现**org.springframework.context.ApplicationListener**接口的bean来进行事件处理。这个接口中只有一个方法，**onApplicationEvent**用来当一个事件发送过来时这个方法来触发相应的处理。该接口可以通过指定需要接收的事件来实现(不懂看源码咯，源码里方法接收一个`event`作为参数)。由此，Spring会自动过滤筛选可以用来接收给定事件的监听器(`listeners`)。

```java
/**
 * Interface to be implemented by application event listeners.
 * Based on the standard {@code java.util.EventListener} interface
 * for the Observer design pattern.
 *
 * <p>As of Spring 3.0, an ApplicationListener can generically declare the event type
 * that it is interested in. When registered with a Spring ApplicationContext, events
 * will be filtered accordingly, with the listener getting invoked for matching event
 * objects only.
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @param <E> the specific ApplicationEvent subclass to listen to
 * @see org.springframework.context.event.ApplicationEventMulticaster
 */
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {

	/**
	 * Handle an application event.
	 * @param event the event to respond to
	 */
	void onApplicationEvent(E event);

}
```

事件通过**org.springframework.context.ApplicationEvent**实例来表示。这个抽象类继承扩展了**java.util.EventObject**，可以使用**EventObject中的getSource**方法，我们可以很容易地获得所发生的给定事件的对象。这里，事件存在两种类型：

- **与应用程序上下文相关联**：所有这种类型的事件都继承自**org.springframework.context.event.ApplicationContextEvent**类。它们应用于由**org.springframework.context.ApplicationContext**引发的事件(其构造函数传入的是`ApplicationContext`类型的参数)。这样，我们就可以直接通过应用程序上下文的生命周期来得到所发生的事件：`ContextStartedEvent`在上下文启动时被启动，当它停止时启动`ContextStoppedEvent`，当上下文被刷新时产生`ContextRefreshedEvent`，最后在上下文关闭时产生`ContextClosedEvent`。

```java
/**
 * Base class for events raised for an {@code ApplicationContext}.
 *
 * @author Juergen Hoeller
 * @since 2.5
 */
@SuppressWarnings("serial")
public abstract class ApplicationContextEvent extends ApplicationEvent {

	/**
	 * Create a new ContextStartedEvent.
	 * @param source the {@code ApplicationContext} that the event is raised for
	 * (must not be {@code null})
	 */
	public ApplicationContextEvent(ApplicationContext source) {
		super(source);
	}

	/**
	 * Get the {@code ApplicationContext} that the event was raised for.
	 */
	public final ApplicationContext getApplicationContext() {
		return (ApplicationContext) getSource();
	}

}

/**
 * Event raised when an {@code ApplicationContext} gets started.
 *
 * @author Mark Fisher
 * @author Juergen Hoeller
 * @since 2.5
 * @see ContextStoppedEvent
 */
@SuppressWarnings("serial")
public class ContextStartedEvent extends ApplicationContextEvent {

	/**
	 * Create a new ContextStartedEvent.
	 * @param source the {@code ApplicationContext} that has been started
	 * (must not be {@code null})
	 */
	public ContextStartedEvent(ApplicationContext source) {
		super(source);
	}

}

/**
 * Event raised when an {@code ApplicationContext} gets stopped.
 *
 * @author Mark Fisher
 * @author Juergen Hoeller
 * @since 2.5
 * @see ContextStartedEvent
 */
@SuppressWarnings("serial")
public class ContextStoppedEvent extends ApplicationContextEvent {

	/**
	 * Create a new ContextStoppedEvent.
	 * @param source the {@code ApplicationContext} that has been stopped
	 * (must not be {@code null})
	 */
	public ContextStoppedEvent(ApplicationContext source) {
		super(source);
	}

}

/**
 * Event raised when an {@code ApplicationContext} gets initialized or refreshed.
 *
 * @author Juergen Hoeller
 * @since 04.03.2003
 * @see ContextClosedEvent
 */
@SuppressWarnings("serial")
public class ContextRefreshedEvent extends ApplicationContextEvent {

	/**
	 * Create a new ContextRefreshedEvent.
	 * @param source the {@code ApplicationContext} that has been initialized
	 * or refreshed (must not be {@code null})
	 */
	public ContextRefreshedEvent(ApplicationContext source) {
		super(source);
	}

}

/**
 * Event raised when an {@code ApplicationContext} gets closed.
 *
 * @author Juergen Hoeller
 * @since 12.08.2003
 * @see ContextRefreshedEvent
 */
@SuppressWarnings("serial")
public class ContextClosedEvent extends ApplicationContextEvent {

	/**
	 * Creates a new ContextClosedEvent.
	 * @param source the {@code ApplicationContext} that has been closed
	 * (must not be {@code null})
	 */
	public ContextClosedEvent(ApplicationContext source) {
		super(source);
	}

}
```

- **与request 请求相关联**：由**org.springframework.web.context.support.RequestHandledEvent**实例来表示，当在ApplicationContext中处理请求时，它们被引发。

Spring如何将事件分配给专门的监听器？这个过程由事件广播器(`event multicaster`)来实现，由**org.springframework.context.event.ApplicationEventMulticaster**接口的实现表示。此接口定义了3种方法，用于：

- **添加新的监听器**：定义了两种方法来添加新的监听器：**addApplicationListener(ApplicationListener<?> listener)**和**addApplicationListenerBean(String listenerBeanName)**。当监听器对象已知时，可以应用第一个。如果使用第二个，我们需要将bean name 得到listener对象(`依赖查找DL`)，然后再将其添加到`listener`列表中。
- **删除监听器**：添加方法一样，我们可以通过传递对象来删除一个监听器(**removeApplicationListener(ApplicationListener<?> listener)**或通过传递bean名称(**removeApplicationListenerBean(String listenerBeanName))**， 第三种方法，**removeAllListeners()**用来删除所有已注册的监听器
- **将事件发送到已注册的监听器**：由**multicastEvent(ApplicationEvent event)**源码注释可知，它用来向所有注册的监听器发送事件。实现可以从**org.springframework.context.event.SimpleApplicationEventMulticaster中**找到，如下所示：

```java
@Override
public void multicastEvent(ApplicationEvent event) {
	multicastEvent(event, resolveDefaultEventType(event));
}

@Override
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
	ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
	for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
		Executor executor = getTaskExecutor();
		if (executor != null) {
			executor.execute(() -> invokeListener(listener, event));
		}
		else {
			invokeListener(listener, event);
		}
	}
}

private ResolvableType resolveDefaultEventType(ApplicationEvent event) {
	return ResolvableType.forInstance(event);
}

/**
 * Invoke the given listener with the given event.
 * @param listener the ApplicationListener to invoke
 * @param event the current event to propagate
 * @since 4.1
 */
@SuppressWarnings({"unchecked", "rawtypes"})
protected void invokeListener(ApplicationListener listener, ApplicationEvent event) {
	ErrorHandler errorHandler = getErrorHandler();
	if (errorHandler != null) {
		try {
			listener.onApplicationEvent(event);
		}
		catch (Throwable err) {
			errorHandler.handleError(err);
		}
	}
	else {
		try {
			listener.onApplicationEvent(event);
		}
		catch (ClassCastException ex) {
			String msg = ex.getMessage();
			if (msg == null || msg.startsWith(event.getClass().getName())) {
				// Possibly a lambda-defined listener which we could not resolve the generic event type for
				Log logger = LogFactory.getLog(getClass());
				if (logger.isDebugEnabled()) {
					logger.debug("Non-matching event type for listener: " + listener, ex);
				}
			}
			else {
				throw ex;
			}
		}
	}
}
```

我们来看看`event multicaster`在应用程序上下文中所在的位置。在`AbstractApplicationContext`中定义的一些方法可以看到其中包含调用**public void publishEvent**方法。通过这种方法的注释可知，它负责向所有监听器发送给定的事件：

```java
/**
	 * Publish the given event to all listeners.
	 * <p>Note: Listeners get initialized after the MessageSource, to be able
	 * to access it within listener implementations. Thus, MessageSource
	 * implementations cannot publish events.
	 * @param event the event to publish (may be application-specific or a
	 * standard framework event)
	 */
	@Override
	public void publishEvent(ApplicationEvent event) {
		publishEvent(event, null);
	}

	/**
	 * Publish the given event to all listeners.
	 * <p>Note: Listeners get initialized after the MessageSource, to be able
	 * to access it within listener implementations. Thus, MessageSource
	 * implementations cannot publish events.
	 * @param event the event to publish (may be an {@link ApplicationEvent}
	 * or a payload object to be turned into a {@link PayloadApplicationEvent})
	 */
	@Override
	public void publishEvent(Object event) {
		publishEvent(event, null);
	}

	/**
	 * Publish the given event to all listeners.
	 * @param event the event to publish (may be an {@link ApplicationEvent}
	 * or a payload object to be turned into a {@link PayloadApplicationEvent})
	 * @param eventType the resolved event type, if known
	 * @since 4.2
	 */
	protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
		Assert.notNull(event, "Event must not be null");
		if (logger.isTraceEnabled()) {
			logger.trace("Publishing event in " + getDisplayName() + ": " + event);
		}

		// Decorate event as an ApplicationEvent if necessary
		ApplicationEvent applicationEvent;
		if (event instanceof ApplicationEvent) {
			applicationEvent = (ApplicationEvent) event;
		}
		else {
			applicationEvent = new PayloadApplicationEvent<>(this, event);
			if (eventType == null) {
				eventType = ((PayloadApplicationEvent)applicationEvent).getResolvableType();
			}
		}

		// Multicast right now if possible - or lazily once the multicaster is initialized
		if (this.earlyApplicationEvents != null) {
			this.earlyApplicationEvents.add(applicationEvent);
		}
		else {
			getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
		}

		// Publish event via parent context as well...
		if (this.parent != null) {
			if (this.parent instanceof AbstractApplicationContext) {
				((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
			}
			else {
				this.parent.publishEvent(event);
			}
		}
	}
```

该方法由以下方法调用：启动上下文(启动后发布`ContextStartedEvent`)，停止上下文(停止后发布`ContextStoppedEvent`)，刷新上下文(刷新后发布`ContextRefreshedEvent`)并关闭上下文(关闭后发布`ContextClosedEvent`)：

```java
/**
 * Finish the refresh of this context, invoking the LifecycleProcessor's
 * onRefresh() method and publishing the
 * {@link org.springframework.context.event.ContextRefreshedEvent}.
 */
protected void finishRefresh() {
	// Clear context-level resource caches (such as ASM metadata from scanning).
	clearResourceCaches();

	// Initialize lifecycle processor for this context.
	initLifecycleProcessor();

	// Propagate refresh to lifecycle processor first.
	getLifecycleProcessor().onRefresh();

	// Publish the final event.生命周期Refreshed事件
	publishEvent(new ContextRefreshedEvent(this));

	// Participate in LiveBeansView MBean, if active.
	LiveBeansView.registerApedplicationContext(this);
}
/**
 * Actually performs context closing: publishes a ContextClosedEvent and
 * destroys the singletons in the bean factory of this application context.
 * <p>Called by both {@code close()} and a JVM shutdown hook, if any.
 * @see org.springframework.context.event.ContextClosedEvent
 * @see #destroyBeans()
 * @see #close()
 * @see #registerShutdownHook()
 */
protected void doClose() {
	if (this.active.get() && this.closed.compareAndSet(false, true)) {
		if (logger.isInfoEnabled()) {
			logger.info("Closing " + this);
		}

		LiveBeansView.unregisterApplicationContext(this);

		try {
			// Publish shutdown event.  ContextClosed事件
			publishEvent(new ContextClosedEvent(this));
		}
		catch (Throwable ex) {
			logger.warn("Exception thrown from ApplicationListener handling ContextClosedEvent", ex);
		}

		// Stop all Lifecycle beans, to avoid delays during individual destruction.
		try {
			getLifecycleProcessor().onClose();
		}
         ...
       }

   //---------------------------------------------------------------------
   // Implementation of Lifecycle interface
   //---------------------------------------------------------------------

@Override
public void start() {
	getLifecycleProcessor().start();
	publishEvent(new ContextStartedEvent(this));
}

@Override
public void stop() {
	getLifecycleProcessor().stop();
	publishEvent(new ContextStoppedEvent(this));
}
```

使用Spring的Web应用程序也可以处理与请求相关联的另一种类型的事件(之前说到的`RequestHandledEvent`)。它的处理方式和面向上下文的事件类似。首先，我们可以找到**org.springframework.web.servlet.FrameworkServlet**中处理请求的方法**processRequest**。在这个方法结束的时候，调用了`private void publishRequestHandledEvent(HttpServletRequest request, HttpServletResponse response, long startTime, @Nullable Throwable failureCause)`方法。如其名称所表达的，此方法将向所有监听器发布给定的`RequestHandledEvent`。事件在传递给应用程序上下文的`publishEvent`方法后，将由`event multicaster`发送。这里没毛病，因为`RequestHandledEvent`扩展了与`ApplicationContextEvent`相同的类，即`ApplicationEvent`。来看看`publishRequestHandledEvent`方法的源码：

```java
private void publishRequestHandledEvent(HttpServletRequest request, HttpServletResponse response,
			long startTime, @Nullable Throwable failureCause) {
		//很多人问我Spring5和4的代码有什么区别，就在很多细微的地方，Spring一直在做不懈的改进和封装，不多说，没事可自行			//对比，能学到很多东西
		if (this.publishEvents && this.webApplicationContext != null) {
			// Whether or not we succeeded, publish an event.
			long processingTime = System.currentTimeMillis() - startTime;
			this.webApplicationContext.publishEvent(
					new ServletRequestHandledEvent(this,
							request.getRequestURI(), request.getRemoteAddr(),
							request.getMethod(), getServletConfig().getServletName(),
							WebUtils.getSessionId(request), getUsernameForRequest(request),
							processingTime, failureCause, response.getStatus()));
		}
	}
```

需要注意的是，你可以关闭基于请求的事件的调度。**FrameworkServlet的setPublishEvents(boolean publishEvents)**允许禁用事件分派，例如改进应用程序性能(看代码注释，当没有监听器来管理相应事件的时候，干嘛要浪费性能)。默认情况下，事件调度被激活(默认为true)。

```java
/** Should we publish a ServletRequestHandledEvent at the end of each request? */
private boolean publishEvents = true;
/**
 * Set whether this servlet should publish a ServletRequestHandledEvent at the end
 * of each request. Default is "true"; can be turned off for a slight performance
 * improvement, provided that no ApplicationListeners rely on such events.
 * @see org.springframework.web.context.support.ServletRequestHandledEvent
 */
public void setPublishEvents(boolean publishEvents) {
	this.publishEvents = publishEvents;
}
```

假如有思考的话，从上面的代码中可以知道，事件在应用程序响应性上的表现会很差(大都是一个调用另一个)。这是因为默认情况下，它们是同步调用线程(即使用同一线程去处理事务，处理请求，以及准备视图的输出)。因此，如果一个监听器需要几秒钟的时间来响应，整个应用程序可能会受到慢的要死。幸运的是，我们可以指定事件处理的异步执行(参考上面的`multicastEvent`源码)。需要注意的是，所处理的事件将无法与调用者的上下文(类加载器或事务)进行交互。这里参考`multicastEvent`方法源码即可。默认情况下，**org.springframework.core.task.SyncTaskExecutor**用来调用相应监听器。

```java
public class SyncTaskExecutor implements TaskExecutor, Serializable {

	/**
	 * Executes the given {@code task} synchronously, through direct
	 * invocation of it's {@link Runnable#run() run()} method.
	 * @throws IllegalArgumentException if the given {@code task} is {@code null}
	 */
	@Override
	public void execute(Runnable task) {
		Assert.notNull(task, "Runnable must not be null");
		task.run();
	}

}
```

## 在Spring中实现一个简单的监听器

为了更好的理解事件监听器，我们来写一个小的测试用例。通过这个例子，我们要证明默认情况下，监听器`listeners`在其调用者线程中执行了分发的事件。所以，为了不立即得到结果，我们在监听器中休眠5秒(调用Thread.sleep(5000))。测试检查是否达到3个目的：如果controller 的返回结果和所预期的视图名称相匹配，如果事件监听器花了5秒钟的时间才响应(Thread.sleep执行没有任何问题)，并且如果controller 的同样花了5秒钟来生成视图(因为监听器的休眠)。

第二个定义的测试将验证我们的监听器是否在另一个事件中被捕获(和之前的类继承同一个类型)。首先，在配置文件中对bean的定义：

```xml
< -- This bean will catch SampleCustomEvent launched in tested controller -->
<bean class="com.migo.event.SampleCustomEventListener">
< -- Thanks to this bean we'll able to get the execution times of tested controller and listener -->
<bean class="com.migo.event.TimeExecutorHolder" id="timeExecutorHolder">
</bean></bean>
```

事件和监听器的代码：

```java
public class SampleCustomEvent extends ApplicationContextEvent {

  private static final long serialVersionUID = 4236181525834402987L;

  public SampleCustomEvent(ApplicationContext source) {
    super(source);
  }
}

public class OtherCustomEvent extends ApplicationContextEvent {

  private static final long serialVersionUID = 5236181525834402987L;

  public OtherCustomEvent(ApplicationContext source) {
    super(source);
  }
}

public class SampleCustomEventListener implements ApplicationListener<samplecustomevent> {

  @Override
  public void onApplicationEvent(SampleCustomEvent event) {
    long start = System.currentTimeMillis();
    try {
      Thread.sleep(5000);
    } catch (Exception e) {
      e.printStackTrace();
    }
    long end = System.currentTimeMillis();
    int testTime = Math.round((end - start) / 1000);
    ((TimeExecutorHolder) event.getApplicationContext().getBean("timeExecutorHolder")).addNewTime("sampleCustomEventListener", new Integer(testTime));
  }
}
```

没什么复杂的，事件只能被用来初始化。监听器通过获取当前时间(以毫秒为单位)来测试所执行时间，并在转换后保存(以秒为单位)。监听器使用的`TimeExecutorHolder`也不复杂：

```java
public class TimeExecutorHolder {

  private Map<String, Integer> testTimes = new HashMap();

  public void addNewTime(String key, Integer value) {
    testTimes.put(key, value);
  }

  public Integer getTestTime(String key) {
    return testTimes.get(key);
  }
}
```

此对象只保留测试元素的执行时间一个Map。测试的controller实现看起来类似于监听器。唯一的区别是它发布一个事件(接着被已定义的监听器捕获)并返回一个名为“success”的视图：

```java
@Controller
public class TestController {
  @Autowired
  private ApplicationContext context;

  @RequestMapping(value = "/testEvent")
  public String testEvent() {
    long start = System.currentTimeMillis();
    context.publishEvent(new SampleCustomEvent(context));
    long end = System.currentTimeMillis();
    int testTime = (int)((end - start) / 1000);
    ((TimeExecutorHolder) context.getBean("timeExecutorHolder")).addNewTime("testController", new Integer(testTime));
    return "success";
  }

  @RequestMapping(value = "/testOtherEvent")
  public String testOtherEvent() {
    context.publishEvent(new OtherCustomEvent(context));
    return "success";
  }
}
```

最后，写一个测试用例，它调用/testEvent并在`TimeExecutorHolder bean`之后检查以验证两个部分的执行时间：

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations={"file:applicationContext-test.xml"})
@WebAppConfiguration
public class SpringEventsTest {

  @Autowired
  private WebApplicationContext wac;

  private MockMvc mockMvc;

  @Before
  public void setUp() {
    this.mockMvc = MockMvcBuilders.webAppContextSetup(this.wac).build();
  }

  @Test
  public void test() {
    try {
      MvcResult result = mockMvc.perform(get("/testEvent")).andReturn();
      ModelAndView view = result.getModelAndView();
      String expectedView = "success";
      assertTrue("View name from /testEvent should be '"+expectedView+"' but was '"+view.getViewName()+"'", view.getViewName().equals(expectedView));
    } catch (Exception e) {
      e.printStackTrace();
    }
    TimeExecutorHolder timeHolder = (TimeExecutorHolder) this.wac.getBean("timeExecutorHolder");
    int controllerSec = timeHolder.getTestTime("testController").intValue();
    int eventSec = timeHolder.getTestTime("sampleCustomEventListener").intValue();
    assertTrue("Listener for SampleCustomEvent should take 5 seconds before treating the request but it took "+eventSec+" instead",  eventSec == 5);
    assertTrue("Because listener took 5 seconds to response, controller should also take 5 seconds before generating the view, but it took "+controllerSec+ " instead", controllerSec == 5);
  }

  @Test
  public void otherTest() {
    TimeExecutorHolder timeHolder = (TimeExecutorHolder) this.wac.getBean("timeExecutorHolder");
    timeHolder.addNewTime("sampleCustomEventListener", -34);
    try {
      MvcResult result = mockMvc.perform(get("/testOtherEvent")).andReturn();
      ModelAndView view = result.getModelAndView();
      String expectedView = "success";
      assertTrue("View name from /testEvent should be '"+expectedView+"' but was '"+view.getViewName()+"'", view.getViewName().equals(expectedView));
    } catch (Exception e) {
      e.printStackTrace();
    }
    Integer eventSecObject = timeHolder.getTestTime("sampleCustomEventListener");
    assertTrue("SampleCustomEventListener shouldn't be trigerred on OtherEvent but it was", eventSecObject.intValue() == -34);
  }
}
```

测试通过没有任何问题。它证明了我们所设定的许多假设。

首先，我们看到事件编程包括在信号发送到应用程序时触发并执行某些操作。这个信号必须有一个监听器在监听。在Spring中，由于监听器中的泛型定义(`void onApplicationEvent(E event);`)，事件可以很容易地被`listeners`所捕获。通过它，如果所触发的事件对应于监听器所预期的事件，我们无须多余的检查(说的啰嗦了，就是符合所需求的类型即可，省去很多麻烦，我们可以直接根据泛型就可以实现很多不同的处理)。我们还发现，默认情况下，监听器是以同步方式执行的。所以在调用线程同时执行比如视图生成或数据库处理的操作是不行的。

最后，要说的是，算是一个前后端通用的思想吧，所谓的事件，其实想来，不过是一个接口而已，把这个接口派发出去(event multicaster)，由谁来实现，这是他们的事情，这里就有一个装饰类(这么理解就好)，其名字叫listener，拿到这个派发的事件接口，然后调用相应的实现，这里为了程序的更加灵活和高可用，我们会调用相应的adapter适配器，最后调用其相应的Handler实现，然后Handler会调用相应的service，service调用dao。

同样这个思想用在前端就是组件对外派发一个事件，这个事件由其父组件或者实现，或者继续向外派发，最后用一个具体的方法将之实现即可

其实对应于我们的数学来讲就是，我们定义一个数学公式f(x)*p(y)一样，这个派发出去，无论我们先实现了f(x)还是先实现了p(y)，还是一次全实现，还是分几次派发出去，终究我们会在最后去将整个公式完全解答出来，这也就是所谓的事件机制，难么？

# 666. 彩蛋

如果你对 Spring 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)