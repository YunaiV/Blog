title: Spring 5 源码解析 —— Spring 中的异步事件
date: 2018-01-23
tag: 
categories: Spring
permalink: Spring/asyncTaskExecutor
author: 一叶知秋
from_url: https://muyinchen.github.io/2017/09/28/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— Spring%E4%B8%AD%E7%9A%84%E5%BC%82%E6%AD%A5%E4%BA%8B%E4%BB%B6/
wechat_url: 

-------

摘要: 原创出处 https://muyinchen.github.io/2017/09/28/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— Spring%E4%B8%AD%E7%9A%84%E5%BC%82%E6%AD%A5%E4%BA%8B%E4%BB%B6/ 「一叶知秋」欢迎转载，保留摘要，谢谢！

- [Spring中的异步事件](http://www.iocoder.cn/Spring/asyncTaskExecutor/)
- [写一个Spring中异步事件的例子](http://www.iocoder.cn/Spring/asyncTaskExecutor/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

上一篇 [Spring框架中的事件和监听器](https://muyinchen.github.io/2017/09/27/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-Spring%E6%A1%86%E6%9E%B6%E4%B8%AD%E7%9A%84%E4%BA%8B%E4%BB%B6%E5%92%8C%E7%9B%91%E5%90%AC%E5%99%A8/)并未对Spring框架中的异步事件涉及太多，所以本篇是对其一个补充。

同步事件有一个主要缺点：它们在所调用线程的本地执行(也就是将所调用线程看成主线程的话，就是在主线程里依次执行)。如果监听器处理同步事件需要5秒钟的响应，则最终结果是用户将在至少5秒内无法看到响应(可以通过[Spring框架中的事件和监听器](https://muyinchen.github.io/2017/09/27/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-Spring%E6%A1%86%E6%9E%B6%E4%B8%AD%E7%9A%84%E4%BA%8B%E4%BB%B6%E5%92%8C%E7%9B%91%E5%90%AC%E5%99%A8/)中的例子了解具体)。所以，我们可以通过一个替代方案来解决这个问题 - 异步事件。

接下来也就是介绍Spring框架中的异步事件。老规矩，第一部分深入框架源码，将描述主要组成部分以及它们如何一起协作的。在第二部分，我们将编写一些测试用例来检查异步事件的执行情况。

## Spring中的异步事件

在Spring中处理异步事件是基于本地的Java并发解决方案—任务执行器(可以了解下Java Executor框架的内容)。事件由**multicastEvent** 方法调度。它通过使用**java.util.concurrent.Executor**接口的实现将事件发送到专用的监听器。Multicaster会调用同步执行器，因为它是默认实现，这点在[Spring框架中的事件和监听器](https://muyinchen.github.io/2017/09/27/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-Spring%E6%A1%86%E6%9E%B6%E4%B8%AD%E7%9A%84%E4%BA%8B%E4%BB%B6%E5%92%8C%E7%9B%91%E5%90%AC%E5%99%A8/)有明确的例子，从源码的角度也就是是否设置有`SyncTaskExecutor`实例。从`public void setTaskExecutor(@Nullable Executor taskExecutor)`其中，@Nullable 可看出Executor参数可为null，默认不设置的话，multicastEvent也就直接 跳过异步执行了

**org.springframework.context.event.SimpleApplicationEventMulticaster**

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
	 * Set a custom executor (typically a {@link 	org.springframework.core.task.TaskExecutor})
	 * to invoke each listener with.
	 * <p>Default is equivalent to {@link org.springframework.core.task.SyncTaskExecutor},
	 * executing all listeners synchronously in the calling thread.
	 * <p>Consider specifying an asynchronous task executor here to not block the
	 * caller until all listeners have been executed. However, note that asynchronous
	 * execution will not participate in the caller's thread context (class loader,
	 * transaction association) unless the TaskExecutor explicitly supports this.
	 * @see org.springframework.core.task.SyncTaskExecutor
	 * @see org.springframework.core.task.SimpleAsyncTaskExecutor
	 * @Nullable 可看出Executor参数可为null，默认不设置的话，上面multicastEvent也就直接	  * 跳过异步执行了
	 */
	public void setTaskExecutor(@Nullable Executor taskExecutor) {
		this.taskExecutor = taskExecutor;
	}

	/**
	 * Return the current task executor for this multicaster.
	 */
	@Nullable
	protected Executor getTaskExecutor() {
		return this.taskExecutor;
	}
```

异步执行器的实现可以参考**org.springframework.core.task.SimpleAsyncTaskExecutor**。这个类为每个提交的任务创建新的线程。然而，它不会重用线程，所以如果我们有很多长执行时间的异步任务需要来处理的时候，线程创建的风险就会变得太大了，会占用大量的资源，不光是cpu还包括jvm。具体源码如下:

```java
/**
 * Executes the given task, within a concurrency throttle
 * if configured (through the superclass's settings).
 * @see #doExecute(Runnable)
 */
@Override
public void execute(Runnable task) {
	execute(task, TIMEOUT_INDEFINITE);
}

/**
 * Executes the given task, within a concurrency throttle
 * if configured (through the superclass's settings).
 * <p>Executes urgent tasks (with 'immediate' timeout) directly,
 * bypassing the concurrency throttle (if active). All other
 * tasks are subject to throttling.
 * @see #TIMEOUT_IMMEDIATE
 * @see #doExecute(Runnable)
 */
@Override
public void execute(Runnable task, long startTimeout) {
	Assert.notNull(task, "Runnable must not be null");
	Runnable taskToUse = (this.taskDecorator != null ? this.taskDecorator.decorate(task) : task);
	if (isThrottleActive() && startTimeout > TIMEOUT_IMMEDIATE) {
		this.concurrencyThrottle.beforeAccess();
		doExecute(new ConcurrencyThrottlingRunnable(taskToUse));
	}
	else {
		doExecute(taskToUse);
	}
}

@Override
public Future<?> submit(Runnable task) {
     	//创建
	FutureTask<Object> future = new FutureTask<>(task, null);
     	//执行
	execute(future, TIMEOUT_INDEFINITE);
	return future;
}

@Override
public <T> Future<T> submit(Callable<T> task) {
	FutureTask<T> future = new FutureTask<>(task);
	execute(future, TIMEOUT_INDEFINITE);
	return future;
}
/**
 * Template method for the actual execution of a task.
 * <p>The default implementation creates a new Thread and starts it.
 * @param task the Runnable to execute
 * @see #setThreadFactory
 * @see #createThread
 * @see java.lang.Thread#start()
 */
protected void doExecute(Runnable task) {
	Thread thread = (this.threadFactory != null ? this.threadFactory.newThread(task) : createThread(task));
     //可以看出，执行也只是简单的将创建的线程start执行下，别提什么重用了
	thread.start();
}
```

为了从线程池功能中受益，我们可以使用另一个Spring的Executor实现，**org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor**。类如其名，这个`Executor`允许我们使用线程池。关于线程池的源码，请期待我的Java9的书籍，里面会涉及到这里面的细节分析，也可以参考其他博客的博文(哈哈，我就是打个小广告而已)。

**org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor**

```java
	/**
	 * Return the underlying ThreadPoolExecutor for native access.
	 * @return the underlying ThreadPoolExecutor (never {@code null})
	 * @throws IllegalStateException if the ThreadPoolTaskExecutor hasn't been initialized yet
	 */
	public ThreadPoolExecutor getThreadPoolExecutor() throws IllegalStateException {
		Assert.state(this.threadPoolExecutor != null, "ThreadPoolTaskExecutor not initialized");
		return this.threadPoolExecutor;
	}

@Override
	public void execute(Runnable task) {
		Executor executor = getThreadPoolExecutor();
		try {
			executor.execute(task);
		}
		catch (RejectedExecutionException ex) {
			throw new TaskRejectedException("Executor [" + executor + "] did not accept task: " + task, ex);
		}
	}

	@Override
	public void execute(Runnable task, long startTimeout) {
		execute(task);
	}

	@Override
	public Future<?> submit(Runnable task) {
		ExecutorService executor = getThreadPoolExecutor();
		try {
			return executor.submit(task);
		}
		catch (RejectedExecutionException ex) {
			throw new TaskRejectedException("Executor [" + executor + "] did not accept task: " + task, ex);
		}
	}
```

## 写一个Spring中异步事件的例子

我们来编写一个能够同时处理同步和异步事件的multicaster。同步事件将使用本地同步调度程序进行调度(SyncTaskExecutor)，异步使用Spring的ThreadPoolTaskExecutor实现。

```java
/**
 * 下面的注释意思很明显了，不多说了
 * {@link TaskExecutor} implementation that executes each task <i>synchronously</i>
 * in the calling thread.
 *
 * <p>Mainly intended for testing scenarios.
 *
 * <p>Execution in the calling thread does have the advantage of participating
 * in it's thread context, for example the thread context class loader or the
 * thread's current transaction association. That said, in many cases,
 * asynchronous execution will be preferable: choose an asynchronous
 * {@code TaskExecutor} instead for such scenarios.
 *
 * @author Juergen Hoeller
 * @since 2.0
 * @see SimpleAsyncTaskExecutor
 */
@SuppressWarnings("serial")
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

首先，我们需要为我们的测试用例添加一些bean：

```xml
<bean id="syncTaskExecutor" class="org.springframework.core.task.SyncTaskExecutor" />
<bean id="asyncTaskExecutor" class="org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor">
  <!-- 10 task will be submitted immediately -->
  <property name="corePoolSize" value="10" />
  <!-- If 10 task are already submitted and treated, we allow to enlarge pool capacity to 15 (10 from core pool size + 5 from max pool size) -->
  <property name="maxPoolSize" value="15" />
  <!-- Number of tasks that can be placed into waiting queue -->
  <property name="queueCapacity" value="10" />
</bean>

<bean id="applicationEventMulticaster" class="com.migo.event.SimpleEventMulticaster">
  <property name="taskExecutor" ref="syncTaskExecutor" />
  <property name="asyncTaskExecutor" ref="asyncTaskExecutor" />
</bean>
<bean id="taskStatsHolder" class="com.migo.event.TaskStatsHolder" />
```

用于测试任务执行结果的两个类：

```java
// TaskStatsHolder.java
/****
 ** Holder bean for all executed tasks.
 **/
public class TaskStatsHolder {

  private Map<String, TaskStatData> tasks = new HashMap<String, TaskStatData>();

  public void addNewTaskStatHolder(String key, TaskStatData value) {
    tasks.put(key, value);
  }

  public TaskStatData getTaskStatHolder(String key) {
    return tasks.get(key);
  }
}

// TaskStatData.java
/****
 ** Holder class for all statistic data about already executed tasks.
 **/
public class TaskStatData {

    private String threadName;
    private int executionTime;
    private long startTime;
    private long endTime;

    public TaskStatData(String threadName, long startTime, long endTime) {
      this.threadName = threadName;
      this.startTime = startTime;
      this.endTime = endTime;
      this.executionTime = Math.round((endTime - startTime) / 1000);
    }

    public String getThreadName() {
      return threadName;
    }
    public int getExecutionTime() {
      return this.executionTime;
    }
    public long getStartTime() {
      return this.startTime;
    }
    public long getEndTime() {
      return this.endTime;
    }

    @Override
    public String toString() {
      StringBuilder result = new StringBuilder();
      result.append("TaskStatData {thread name: ").append(this.threadName).append(", start time: ").append(new Date(this.startTime));
      result.append(", end time: ").append(new Date(this.endTime)).append(", execution time: ").append(this.executionTime).append(" seconds}");
      return result.toString();
    }

}
```

如上代码所示，这些都是简单对象。我们会使用这些对象来检查我们的假设和执行结果是否相匹配。两个要分发的事件也很简单：

```java
// ProductChangeFailureEvent.java
/**
 * This is synchronous event dispatched when one product is modified in the backoffice.
 * When product's modification fails (database, validation problem), this event is dispatched to
 * all listeners. It's synchronous because we want to inform the user that some actions were done
 * after the failure. Otherwise (asynchronous character of event) we shouldn't be able to
 * know if something was done or not after the dispatch.
 **/
public class ProductChangeFailureEvent extends ApplicationContextEvent {

  private static final long serialVersionUID = -1681426286796814792L;
  public static final String TASK_KEY = "ProductChangeFailureEvent";

  public ProductChangeFailureEvent(ApplicationContext source) {
    super(source);
  }
}

// NotifMailDispatchEvent.java
/**
 * Event dispatched asynchronously every time when we want to send a notification mail.
 * Notification mails to send should be stored somewhere (filesystem, database...) but in
 * our case, we'll handle only one notification mail: when one product out-of-stock becomes available again.
 **/
public class NotifMailDispatchEvent extends ApplicationContextEvent implements AsyncApplicationEvent {

  private static final long serialVersionUID = 9202282810553100778L;
  public static final String TASK_KEY = "NotifMailDispatchEvent";

  public NotifMailDispatchEvent(ApplicationContext source) {
    super(source);
  }
}
```

而用于处理相应调度事件的监听器也只需要将数据放入`TaskStatsHolder实例类`中即可：

```java
// ProductChangeFailureListener.java
@Component
public class ProductChangeFailureListener
    implements ApplicationListener<ProductChangeFailureEvent>{

  @Override
  public void onApplicationEvent(ProductChangeFailureEvent event) {
    long start = System.currentTimeMillis();
    long end = System.currentTimeMillis();
    ((TaskStatsHolder) event.getApplicationContext().getBean("taskStatsHolder")).addNewTaskStatHolder(ProductChangeFailureEvent.TASK_KEY, new TaskStatData(Thread.currentThread().getName(), start, end));
  }

}

// NotifMailDispatchListener.java
@Component
public class NotifMailDispatchListener
    implements ApplicationListener<NotifMailDispatchEvent>{

  @Override
  public void onApplicationEvent(NotifMailDispatchEvent event) throws InterruptedException {
    long start = System.currentTimeMillis();
    // sleep 5 seconds to avoid that two listeners execute at the same moment
    Thread.sleep(5000);
    long end = System.currentTimeMillis();
    ((TaskStatsHolder) event.getApplicationContext().getBean("taskStatsHolder")).addNewTaskStatHolder(NotifMailDispatchEvent.TASK_KEY, new TaskStatData(Thread.currentThread().getName(), start, end));
  }
}
```

用于测试的controller如下所示：

```java
@Controller
public class ProductController {

  @Autowired
  private ApplicationContext context;

  @RequestMapping(value = "/products/change-failure")
  public String changeFailure() {
    try {
      System.out.println("I'm modifying the product but a NullPointerException will be thrown");
      String name = null;
      if (name.isEmpty()) {
        // show error message here
        throw new RuntimeException("NullPointerException");
      }
    } catch (Exception e) {
            context.publishEvent(new ProductChangeFailureEvent(context));
    }
    return "success";
  }


  @RequestMapping(value = "/products/change-success")
  public String changeSuccess() {
    System.out.println("Product was correctly changed");
    context.publishEvent(new NotifMailDispatchEvent(context));
    return "success";
  }
}
```

最后，测试用例：

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations={"classpath:applicationContext-test.xml"})
@WebAppConfiguration
public class SpringSyncAsyncEventsTest {

  @Autowired
  private WebApplicationContext wac;

  @Test
  public void test() {
    MockMvc mockMvc = MockMvcBuilders.webAppContextSetup(this.wac).build();
    // execute both urls simultaneously
    mockMvc.perform(get("/products/change-success"));
    mockMvc.perform(get("/products/change-failure"));

    // get stats holder and check if both stats are available:
    // - mail dispatching shouldn't be available because it's executed after a sleep of 5 seconds
    // - product failure should be available because it's executed synchronously, almost immediately (no operations in listeners)
    TaskStatsHolder statsHolder = (TaskStatsHolder) this.wac.getBean("taskStatsHolder");
    TaskStatData mailStatData = statsHolder.getTaskStatHolder(NotifMailDispatchEvent.TASK_KEY);
    TaskStatData productFailureData = statsHolder.getTaskStatHolder(ProductChangeFailureEvent.TASK_KEY);
    assertTrue("Task for mail dispatching is executed after 5 seconds, so at this moment, it taskStatsHolder shouldn't contain it",
        mailStatData == null);
    assertTrue("productFailureHolder shouldn't be null but it is",
        productFailureData != null);
    assertTrue("Product failure listener should be executed within 0 seconds but took "+productFailureData.getExecutionTime()+" seconds",
        productFailureData.getExecutionTime() == 0);
    while (mailStatData == null) {
        mailStatData = statsHolder.getTaskStatHolder(NotifMailDispatchEvent.TASK_KEY);
    }

    // check mail dispatching stats again, when available
    assertTrue("Now task for mail dispatching should be at completed state",
        mailStatData != null);
    assertTrue("Task for mail dispatching should take 5 seconds but it took "+mailStatData.getExecutionTime()+" seconds",
        mailStatData.getExecutionTime() == 5);
    assertTrue("productFailureHolder shouldn't be null but it is",
        productFailureData != null);
    assertTrue("Product failure listener should be executed within 0 seconds but took "+productFailureData.getExecutionTime()+" seconds",
        productFailureData.getExecutionTime() == 0);
    assertTrue("Thread executing mail dispatch and product failure listeners shouldn't be the same",
        !productFailureData.getThreadName().equals(mailStatData.getThreadName()));
    assertTrue("Thread executing product failure listener ("+productFailureData.getThreadName()+") should be the same as current thread ("+Thread.currentThread().getName()+") but it wasn't",
        Thread.currentThread().getName().equals(productFailureData.getThreadName()));
    assertTrue("Thread executing mail dispatch listener ("+mailStatData.getThreadName()+") shouldn't be the same as current thread ("+Thread.currentThread().getName()+") but it was",
        !Thread.currentThread().getName().equals(mailStatData.getThreadName()));
    // make some output to see the informations about tasks
    System.out.println("Data about mail notif dispatching event: "+mailStatData);
    System.out.println("Data about product failure dispatching event: "+productFailureData);
  }
}
```

因之前整理的笔记此处SimpleEventMulticaster忘了放进去，也懒得去找了，可以通过xml定义去查看下，这个测试用例可以看出两个listener不是由同一个executor启动的，Product failure 监听器由同步执行器执行。因为他们没有做任何操作，几乎立即返回结果。关于邮件调度事件，通过休眠5秒可以得到其执行时间超过Product failure 监听器的执行时间。通过分析输出可以知道，两者在不同的线程中执行，所以由不同的执行器执行(关于这俩执行器的例子可以再搜下相关博文，其实主要还是想表达`SyncTaskExecutor`是在主线程里执行，而`asyncTaskExecutor`由线程池里管理的线程执行)。

```shell
Product was correctly changed
I'm modifying the product but a NullPointerException will be thrown
Data about mail notif dispatching event: TaskStatData {thread name: asyncTaskExecutor-1(异步线程), start time: Thu Jun 19 21:14:18 CEST 2016, end time: Thu Jun 19 21:14:23 CEST 2016, execution time: 5 seconds}
Data about product failure dispatching event: TaskStatData {thread name: main(主线程), start time: Thu Jun 19 21:14:21 CEST 2016, end time: Thu Jun 19 21:14:21 CEST 2016, execution time: 0 seconds}
```

本文简单介绍了如何在Spring中处理异步事件。当监听器需要执行很长时间，而我们又不想阻塞应用程序执行，就可以使用异步执行。异步执行可以通过异步执行器(如ThreadPoolTaskExecutor或SimpleAsyncTaskExecutor)实现。

# 666. 彩蛋

如果你对 Spring 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)