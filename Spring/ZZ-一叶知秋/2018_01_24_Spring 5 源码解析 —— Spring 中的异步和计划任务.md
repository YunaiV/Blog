title: Spring 5 源码解析 —— Spring 中的异步和计划任务
date: 2018-01-24
tag: 
categories: Spring
permalink: Spring/scheduler
author: 一叶知秋
from_url: https://muyinchen.github.io/2017/10/17/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— Spring%E4%B8%AD%E7%9A%84%E5%BC%82%E6%AD%A5%E5%92%8C%E8%AE%A1%E5%88%92%E4%BB%BB%E5%8A%A1/
wechat_url: 

-------

摘要: 原创出处 https://muyinchen.github.io/2017/10/17/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— Spring%E4%B8%AD%E7%9A%84%E5%BC%82%E6%AD%A5%E5%92%8C%E8%AE%A1%E5%88%92%E4%BB%BB%E5%8A%A1/ 「一叶知秋」欢迎转载，保留摘要，谢谢！

- [什么是Spring中的异步任务？](http://www.iocoder.cn/Spring/scheduler/)
- [Spring的异步任务类](http://www.iocoder.cn/Spring/scheduler/)
- [在Spring中配置异步和计划任务](http://www.iocoder.cn/Spring/scheduler/)
- [写一个在Spring中执行异步任务的Demo](http://www.iocoder.cn/Spring/scheduler/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

Java提供了许多创建线程池的方式，并得到一个Future实例来作为任务结果。对于Spring同样小菜一碟，通过其`scheduling`包就可以做到将任务线程中后台执行。

在本文的第一部分中，我们将讨论下Spring中执行计划任务的一些基础知识。之后，我们将解释这些类是如何一起协作来启动并执行计划任务的。下一部分将介绍计划和异步任务的配置。最后，我们来写个Demo，看看如何通过单元测试来编排计划任务。

## 什么是Spring中的异步任务？

在我们正式的进入话题之前，对于Spring，我们需要理解下它实现的两个不同的概念：异步任务和调度任务。显然，两者有一个很大的共同点：都在后台工作。但是，它们之间存在了很大差异。调度任务与异步不同，其作用与Linux中的`CRON job`完全相同(windows里面也有计划任务)。举个栗子，有一个任务必须每40分钟执行一次，那么，可以通过XML文件或者注解来进行此配置。简单的异步任务在后台执行就好，无需配置执行频率。

因为它们是两种不同的任务类型，它们两个的执行者自然也就不同。第一个看起来有点像Java的并发执行器(`concurrency executor`)，这里会有专门去写一篇关于Java中的执行器来具体了解。根据[Spring文档](http://docs.spring.io/spring/docs/5.0.0.M5/spring-framework-reference/htmlsingle/#scheduling)**TaskExecutor**所述，它提供了基于Spring的抽象来处理线程池，这点，也可以通过其类的注释去了解。另一个抽象接口是**TaskScheduler**，它用于在将来给定的时间点来安排任务，并执行一次或定期执行。

在分析源码的过程中，发现另一个比较有趣的点是触发器。它存在两种类型：**CronTrigger**或**PeriodTrigger**。第一个模拟CRON任务的行为。所以我们可以在将来确切时间点提交一个任务的执行。另一个触发器可用于定期执行任务。

## Spring的异步任务类

让我们从**org.springframework.core.task.TaskExecutor**类的分析开始。你会发现，其简单的不行，它是一个扩展Java的Executor接口的接口。它的唯一方法也就是是**执行**，在参数中使用Runnable类型的任务。

```Java
package org.springframework.core.task;

import java.util.concurrent.Executor;

/**
 * Simple task executor interface that abstracts the execution
 * of a {@link Runnable}.
 *
 * <p>Implementations can use all sorts of different execution strategies,
 * such as: synchronous, asynchronous, using a thread pool, and more.
 *
 * <p>Equivalent to JDK 1.5's {@link java.util.concurrent.Executor}
 * interface; extending it now in Spring 3.0, so that clients may declare
 * a dependency on an Executor and receive any TaskExecutor implementation.
 * This interface remains separate from the standard Executor interface
 * mainly for backwards compatibility with JDK 1.4 in Spring 2.x.
 *
 * @author Juergen Hoeller
 * @since 2.0
 * @see java.util.concurrent.Executor
 */
@FunctionalInterface
public interface TaskExecutor extends Executor {

	/**
	 * Execute the given {@code task}.
	 * <p>The call might return immediately if the implementation uses
	 * an asynchronous execution strategy, or might block in the case
	 * of synchronous execution.
	 * @param task the {@code Runnable} to execute (never {@code null})
	 * @throws TaskRejectedException if the given task was not accepted
	 */
	@Override
	void execute(Runnable task);

}
```

相对来说，**org.springframework.scheduling.TaskScheduler**接口就有点复杂了。它定义了一组以schedule开头的名称的方法允许我们定义将来要执行的任务。所有 **schedule\*** 方法返回**java.util.concurrent.ScheduledFuture**实例。Spring5中对`scheduleAtFixedRate`方法做了进一步的充实，其实最终调用的还是**ScheduledFuture<?> scheduleAtFixedRate(Runnable task, long period);**

```Java
public interface TaskScheduler {
	/**
	 * Schedule the given {@link Runnable}, invoking it whenever the trigger
	 * indicates a next execution time.
	 * <p>Execution will end once the scheduler shuts down or the returned
	 * {@link ScheduledFuture} gets cancelled.
	 * @param task the Runnable to execute whenever the trigger fires
	 * @param trigger an implementation of the {@link Trigger} interface,
	 * e.g. a {@link org.springframework.scheduling.support.CronTrigger} object
	 * wrapping a cron expression
	 * @return a {@link ScheduledFuture} representing pending completion of the task,
	 * or {@code null} if the given Trigger object never fires (i.e. returns
	 * {@code null} from {@link Trigger#nextExecutionTime})
	 * @throws org.springframework.core.task.TaskRejectedException if the given task was not accepted
	 * for internal reasons (e.g. a pool overload handling policy or a pool shutdown in progress)
	 * @see org.springframework.scheduling.support.CronTrigger
	 */
	@Nullable
	ScheduledFuture<?> schedule(Runnable task, Trigger trigger);

	/**
	 * Schedule the given {@link Runnable}, invoking it at the specified execution time.
	 * <p>Execution will end once the scheduler shuts down or the returned
	 * {@link ScheduledFuture} gets cancelled.
	 * @param task the Runnable to execute whenever the trigger fires
	 * @param startTime the desired execution time for the task
	 * (if this is in the past, the task will be executed immediately, i.e. as soon as possible)
	 * @return a {@link ScheduledFuture} representing pending completion of the task
	 * @throws org.springframework.core.task.TaskRejectedException if the given task was not accepted
	 * for internal reasons (e.g. a pool overload handling policy or a pool shutdown in progress)
	 * 使用了默认实现，值得我们学习使用的，Java9中同样可以有私有实现的，从这里我们可以做到我只通过		 * 一个接口你来实现，我把其他相应的功能默认实现下，最后调用你自定义实现的接口即可，使接口功能更	  * 加一目了然
	 * @since 5.0
	 * @see #schedule(Runnable, Date)
	 */
	default ScheduledFuture<?> schedule(Runnable task, Instant startTime) {
		return schedule(task, Date.from(startTime));
	}
	/**
	 * Schedule the given {@link Runnable}, invoking it at the specified execution time.
	 * <p>Execution will end once the scheduler shuts down or the returned
	 * {@link ScheduledFuture} gets cancelled.
	 * @param task the Runnable to execute whenever the trigger fires
	 * @param startTime the desired execution time for the task
	 * (if this is in the past, the task will be executed immediately, i.e. as soon as possible)
	 * @return a {@link ScheduledFuture} representing pending completion of the task
	 * @throws org.springframework.core.task.TaskRejectedException if the given task was not accepted
	 * for internal reasons (e.g. a pool overload handling policy or a pool shutdown in progress)
	 */
	ScheduledFuture<?> schedule(Runnable task, Date startTime);

...
/**
	 * Schedule the given {@link Runnable}, invoking it at the specified execution time
	 * and subsequently with the given period.
	 * <p>Execution will end once the scheduler shuts down or the returned
	 * {@link ScheduledFuture} gets cancelled.
	 * @param task the Runnable to execute whenever the trigger fires
	 * @param startTime the desired first execution time for the task
	 * (if this is in the past, the task will be executed immediately, i.e. as soon as possible)
	 * @param period the interval between successive executions of the task
	 * @return a {@link ScheduledFuture} representing pending completion of the task
	 * @throws org.springframework.core.task.TaskRejectedException if  the given task was not accepted
	 * for internal reasons (e.g. a pool overload handling policy or a pool shutdown in progress)
	 * @since 5.0
	 * @see #scheduleAtFixedRate(Runnable, Date, long)
	 */
	default ScheduledFuture<?> scheduleAtFixedRate(Runnable task, Instant startTime, Duration period) {
		return scheduleAtFixedRate(task, Date.from(startTime), period.toMillis());
	}

	/**
	 * Schedule the given {@link Runnable}, invoking it at the specified execution time
	 * and subsequently with the given period.
	 * <p>Execution will end once the scheduler shuts down or the returned
	 * {@link ScheduledFuture} gets cancelled.
	 * @param task the Runnable to execute whenever the trigger fires
	 * @param startTime the desired first execution time for the task
	 * (if this is in the past, the task will be executed immediately, i.e. as soon as possible)
	 * @param period the interval between successive executions of the task (in milliseconds)
	 * @return a {@link ScheduledFuture} representing pending completion of the task
	 * @throws org.springframework.core.task.TaskRejectedException if  the given task was not accepted
	 * for internal reasons (e.g. a pool overload handling policy or a pool shutdown in progress)
	 */
	ScheduledFuture<?> scheduleAtFixedRate(Runnable task, Date startTime, long period);

...
}
```

之前提到两个触发器组件，都实现了**org.springframework.scheduling.Trigger**接口。这里，我们只需关注一个的方法**nextExecutionTime** ，其定义下一个触发任务的执行时间。它的两个实现，CronTrigger和PeriodicTrigger，由**org.springframework.scheduling.TriggerContext**来实现信息的存储，由此，我们可以很轻松获得一个任务的最后一个执行时间(**lastScheduledExecutionTime**)，给定任务的最后完成时间(**lastCompletionTime**)或最后一个实际执行时间(**lastActualExecutionTime**)。接下来，我们通过阅读源代码来简单的了解下这些东西。**org.springframework.scheduling.concurrent.ConcurrentTaskScheduler**包含一个私有类`EnterpriseConcurrentTriggerScheduler`。在这个`class`里面，我们可以找到schedule方法：

```Java
public ScheduledFuture<?> schedule(Runnable task, final Trigger trigger) {
  ManagedScheduledExecutorService executor = (ManagedScheduledExecutorService) scheduledExecutor;
  return executor.schedule(task, new javax.enterprise.concurrent.Trigger() {
    @Override
    public Date getNextRunTime(LastExecution le, Date taskScheduledTime) {
      return trigger.nextExecutionTime(le != null ?
        new SimpleTriggerContext(le.getScheduledStart(), le.getRunStart(), le.getRunEnd()) :
        new SimpleTriggerContext());
    }
    @Override
    public boolean skipRun(LastExecution lastExecution, Date scheduledRunTime) {
      return false;
    }
  });
}
```

`SimpleTriggerContext`从其名字就可以看到很多东西了，因为它实现了`TriggerContext`接口。

```Java
/**
 * Simple data holder implementation of the {@link TriggerContext} interface.
 *
 * @author Juergen Hoeller
 * @since 3.0
 */
public class SimpleTriggerContext implements TriggerContext {

	@Nullable
	private volatile Date lastScheduledExecutionTime;

	@Nullable
	private volatile Date lastActualExecutionTime;

	@Nullable
	private volatile Date lastCompletionTime;
...
  /**
	 * Create a SimpleTriggerContext with the given time values.
	 * @param lastScheduledExecutionTime last <i>scheduled</i> execution time
	 * @param lastActualExecutionTime last <i>actual</i> execution time
	 * @param lastCompletionTime last completion time
	 */
	public SimpleTriggerContext(Date lastScheduledExecutionTime, Date lastActualExecutionTime, Date lastCompletionTime) {
		this.lastScheduledExecutionTime = lastScheduledExecutionTime;
		this.lastActualExecutionTime = lastActualExecutionTime;
		this.lastCompletionTime = lastCompletionTime;
	}
  ...
}
```

也正如你看到的，在构造函数中设置的时间值来自**javax.enterprise.concurrent.LastExecution的**实现，其中：

- getScheduledStart：返回上次开始执行任务的时间。它对应于TriggerContext的lastScheduledExecutionTime属性。
- getRunStart：返回给定任务开始运行的时间。在TriggerContext中，它对应于lastActualExecutionTime。
- getRunEnd：任务终止时返回。它用于在TriggerContext中设置lastCompletionTime。

Spring调度和异步执行中的另一个重要类是**org.springframework.core.task.support.TaskExecutorAdapter**。它是一个将**java.util.concurrent.Executor**作为Spring基本的执行器的适配器(描述的有点拗口,看下面代码就明了了)，之前已经描述了`TaskExecutor`。实际上，它引用了Java的ExecutorService，它也是继承了`Executor`接口。此引用用于完成所有提交的任务。

```Java
/**
 * Adapter that takes a JDK {@code java.util.concurrent.Executor} and
 * exposes a Spring {@link org.springframework.core.task.TaskExecutor} for it.
 * Also detects an extended {@code java.util.concurrent.ExecutorService 从此解释上面的说明}, adapting
 * the {@link org.springframework.core.task.AsyncTaskExecutor} interface accordingly.
 *
 * @author Juergen Hoeller
 * @since 3.0
 * @see java.util.concurrent.Executor
 * @see java.util.concurrent.ExecutorService
 * @see java.util.concurrent.Executors
 */
public class TaskExecutorAdapter implements AsyncListenableTaskExecutor {

	private final Executor concurrentExecutor;

	@Nullable
	private TaskDecorator taskDecorator;
  ...
    /**
	 * Create a new TaskExecutorAdapter,
	 * using the given JDK concurrent executor.
	 * @param concurrentExecutor the JDK concurrent executor to delegate to
	 */
	public TaskExecutorAdapter(Executor concurrentExecutor) {
		Assert.notNull(concurrentExecutor, "Executor must not be null");
		this.concurrentExecutor = concurrentExecutor;
	}
 	 /**
	 * Delegates to the specified JDK concurrent executor.
	 * @see java.util.concurrent.Executor#execute(Runnable)
	 */
	@Override
	public void execute(Runnable task) {
		try {
			doExecute(this.concurrentExecutor, this.taskDecorator, task);
		}
		catch (RejectedExecutionException ex) {
			throw new TaskRejectedException(
					"Executor [" + this.concurrentExecutor + "] did not accept task: " + task, ex);
		}
	}

	@Override
	public void execute(Runnable task, long startTimeout) {
		execute(task);
	}

	@Override
	public Future<?> submit(Runnable task) {
		try {
			if (this.taskDecorator == null && this.concurrentExecutor instanceof ExecutorService) {
				return ((ExecutorService) this.concurrentExecutor).submit(task);
			}
			else {
				FutureTask<Object> future = new FutureTask<>(task, null);
				doExecute(this.concurrentExecutor, this.taskDecorator, future);
				return future;
			}
		}
		catch (RejectedExecutionException ex) {
			throw new TaskRejectedException(
					"Executor [" + this.concurrentExecutor + "] did not accept task: " + task, ex);
		}
	}
  ...
}
```

## 在Spring中配置异步和计划任务

下面我们通过代码的方式来实现异步任务。首先，我们需要通过注解来启用配置。它的XML配置如下：

```xml
<task:scheduler id="taskScheduler"/>
<task:executor id="taskExecutor" pool-size="2" />
<task:annotation-driven executor="taskExecutor" scheduler="taskScheduler"/>
<context:component-scan base-package="com.migo.async"/>
```

可以通过将`@EnableScheduling`和`@EnableAsync`注解添加到配置类(用@Configuration注解)来激活两者。完事，我们就可以开始着手实现调度和异步任务。为了实现调度任务，我们可以使用`@Scheduled`注解。我们可以从**org.springframework.scheduling.annotation**包中找到这个注解。它包含了以下几个属性：

- `cron`：使用`CRON`风格(Linux配置定时任务的风格)的配置来配置需要启动的带注解的任务。

- `zone`：要解析`CRON`表达式的时区。

- `fixedDelay`或`fixedDelayString`：在固定延迟时间后执行任务。即任务将在最后一次调用结束和下一次调用的开始之间的这个固定时间段后执行。

- `fixedRate`或`fixedRateString`：使用`fixedRate`注解的方法的调用将以固定的时间段(例如：每10秒钟)进行，与执行生命周期(开始，结束)无关。

- `initialDelay`或`initialDelayString`：延迟首次执行调度方法的时间。请注意，所有值(**fixedDelay** ，**fixedRate** ，**initialDelay** **)必须以毫秒表示。** **需要特别记住的是** ，用@Scheduled注解的方法不能接受任何参数，并且不返回任何内容(void)，如果有返回值，返回值也会被忽略掉的，没什么卵用。定时任务方法由容器管理，而不是由调用者在运行时调用。它们由 **org.springframework.scheduling.annotation.ScheduledAnnotationBeanPostProcessor**来解析，其中包含以下方法来拒绝执行所有不正确定义的函数：

  ```
  protected void processScheduled(Scheduled scheduled, Method method, Object bean) {
    try {
      Assert.isTrue(method.getParameterCount() == 0,
  					"Only no-arg methods may be annotated with @Scheduled");
  	/**
  	 *   之前的版本中直接把返回值非空的给拒掉了，在Spring 4.3 Spring5 的版本中就没那么严格了
     	 *	 Assert.isTrue(void.class.equals(method.getReturnType()),
       *               "Only void-returning methods may be annotated with @Scheduled");
       **/
  // ...
  ```

  ​

  ```Java
  /**
   * 注释很重要
   * An annotation that marks a method to be scheduled. Exactly one of
   * the {@link #cron()}, {@link #fixedDelay()}, or {@link #fixedRate()}
   * attributes must be specified.
   *
   * <p>The annotated method must expect no arguments. It will typically have
   * a {@code void} return type; if not, the returned value will be ignored
   * when called through the scheduler.
   *
   * <p>Processing of {@code @Scheduled} annotations is performed by
   * registering a {@link ScheduledAnnotationBeanPostProcessor}. This can be
   * done manually or, more conveniently, through the {@code <task:annotation-driven/>}
   * element or @{@link EnableScheduling} annotation.
   *
   * <p>This annotation may be used as a <em>meta-annotation</em> to create custom
   * <em>composed annotations</em> with attribute overrides.
   *
   * @author Mark Fisher
   * @author Dave Syer
   * @author Chris Beams
   * @since 3.0
   * @see EnableScheduling
   * @see ScheduledAnnotationBeanPostProcessor
   * @see Schedules
   */
  @Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})
  @Retention(RetentionPolicy.RUNTIME)
  @Documented
  @Repeatable(Schedules.class)
  public @interface Scheduled {
  ...
  }
  ```

  ​

使用`@Async`注解标记一个方法或一个类(通过标记一个类，我们自动将其所有方法标记为异步)。与`@Scheduled`不同，异步任务可以接受参数，并可能返回某些东西。

## 写一个在Spring中执行异步任务的Demo

有了上面这些知识，我们可以来编写异步和计划任务。我们将通过测试用例来展示。我们从不同的任务执行器(task executors)的测试开始：

```Java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations={"classpath:applicationContext-test.xml"})
@WebAppConfiguration
public class TaskExecutorsTest {

  @Test
  public void simpeAsync() throws InterruptedException {
    /**
      * SimpleAsyncTaskExecutor creates new Thread for every task and executes it asynchronously. The threads aren't reused as in
      * native Java's thread pools.
      *
      * The number of concurrently executed threads can be specified through concurrencyLimit bean property
      * (concurrencyLimit XML attribute). Here it's more simple to invoke setConcurrencyLimit method.
      * Here the tasks will be executed by 2 simultaneous threads. Without specifying this value,
      * the number of executed threads will be indefinite.
      *
      * You can observe that only 2 tasks are executed at a given time - even if 3 are submitted to execution (lines 40-42).
      **/
    SimpleAsyncTaskExecutor executor = new SimpleAsyncTaskExecutor("thread_name_prefix_____");
    executor.setConcurrencyLimit(2);
    executor.execute(new SimpleTask("SimpleAsyncTask-1", Counters.simpleAsyncTask, 1000));
    executor.execute(new SimpleTask("SimpleAsyncTask-2", Counters.simpleAsyncTask, 1000));

    Thread.sleep(1050);
    assertTrue("2 threads should be terminated, but "+Counters.simpleAsyncTask.getNb()+" were instead", Counters.simpleAsyncTask.getNb() == 2);

    executor.execute(new SimpleTask("SimpleAsyncTask-3", Counters.simpleAsyncTask, 1000));
    executor.execute(new SimpleTask("SimpleAsyncTask-4", Counters.simpleAsyncTask, 1000));
    executor.execute(new SimpleTask("SimpleAsyncTask-5", Counters.simpleAsyncTask, 2000));

    Thread.sleep(1050);
    assertTrue("4 threads should be terminated, but "+Counters.simpleAsyncTask.getNb()+" were instead", Counters.simpleAsyncTask.getNb() == 4);
    executor.execute(new SimpleTask("SimpleAsyncTask-6", Counters.simpleAsyncTask, 1000));

    Thread.sleep(1050);
    assertTrue("6 threads should be terminated, but "+Counters.simpleAsyncTask.getNb()+" were instead",
      Counters.simpleAsyncTask.getNb() == 6);
  }

  @Test
  public void syncTaskTest() {
    /**
      * SyncTask works almost as Java's CountDownLatch. In fact, this executor is synchronous with the calling thread. In our case,
      * SyncTaskExecutor tasks will be synchronous with JUnit thread. It means that the testing thread will sleep 5
      * seconds after executing the third task ('SyncTask-3'). To prove that, we check if the total execution time is ~5 seconds.
      **/
    long start = System.currentTimeMillis();
    SyncTaskExecutor executor = new SyncTaskExecutor();
    executor.execute(new SimpleTask("SyncTask-1", Counters.syncTask, 0));
    executor.execute(new SimpleTask("SyncTask-2", Counters.syncTask, 0));
    executor.execute(new SimpleTask("SyncTask-3", Counters.syncTask, 0));
    executor.execute(new SimpleTask("SyncTask-4", Counters.syncTask, 5000));
    executor.execute(new SimpleTask("SyncTask-5", Counters.syncTask, 0));
    long end = System.currentTimeMillis();
    int execTime = Math.round((end-start)/1000);
    assertTrue("Execution time should be 5 seconds but was "+execTime+" seconds", execTime == 5);
  }

  @Test
  public void threadPoolTest() throws InterruptedException {
    /**
      * This executor can be used to expose Java's native ThreadPoolExecutor as Spring bean, with the
      * possibility to set core pool size, max pool size and queue capacity through bean properties.
      *
      * It works exactly as ThreadPoolExecutor from java.util.concurrent package. It means that our pool starts
      * with 2 threads (core pool size) and can be growth until 3 (max pool size).
      * In additionally, 1 task can be stored in the queue. This task will be treated
      * as soon as one from 3 threads ends to execute provided task. In our case, we try to execute 5 tasks
      * in 3 places pool and 1 place queue. So the 5th task should be rejected and TaskRejectedException should be thrown.
      **/
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(2);
    executor.setMaxPoolSize(3);
    executor.setQueueCapacity(1);
    executor.initialize();

    executor.execute(new SimpleTask("ThreadPoolTask-1", Counters.threadPool, 1000));
    executor.execute(new SimpleTask("ThreadPoolTask-2", Counters.threadPool, 1000));
    executor.execute(new SimpleTask("ThreadPoolTask-3", Counters.threadPool, 1000));
    executor.execute(new SimpleTask("ThreadPoolTask-4", Counters.threadPool, 1000));
    boolean wasTre = false;
    try {
      executor.execute(new SimpleTask("ThreadPoolTask-5", Counters.threadPool, 1000));
    } catch (TaskRejectedException tre) {
      wasTre = true;
    }
    assertTrue("The last task should throw a TaskRejectedException but it wasn't", wasTre);

    Thread.sleep(3000);

    assertTrue("4 tasks should be terminated, but "+Counters.threadPool.getNb()+" were instead",
      Counters.threadPool.getNb()==4);
  }

}

class SimpleTask implements Runnable {
  private String name;
  private Counters counter;
  private int sleepTime;

  public SimpleTask(String name, Counters counter, int sleepTime) {
    this.name = name;
    this.counter = counter;
    this.sleepTime = sleepTime;
  }

  @Override
  public void run() {
    try {
      Thread.sleep(this.sleepTime);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    this.counter.increment();
    System.out.println("Running task '"+this.name+"' in Thread "+Thread.currentThread().getName());
  }

  @Override
  public String toString() {
          return "Task {"+this.name+"}";
  }
}

enum Counters {

  simpleAsyncTask(0),
  syncTask(0),
  threadPool(0);

  private int nb;

  public int getNb() {
    return this.nb;
  }

  public synchronized void increment() {
    this.nb++;
  }

  private Counters(int n) {
    this.nb = n;
  }
}
```

在过去，我们可以有更多的执行器可以使用(SimpleThreadPoolTaskExecutor，TimerTaskExecutor 这些都2.x 3.x的老古董了)。但都被弃用并由本地Java的执行器取代成为Spring的首选。看看输出的结果：

```Java
Running task 'SimpleAsyncTask-1' in Thread thread_name_prefix_____1
Running task 'SimpleAsyncTask-2' in Thread thread_name_prefix_____2
Running task 'SimpleAsyncTask-3' in Thread thread_name_prefix_____3
Running task 'SimpleAsyncTask-4' in Thread thread_name_prefix_____4
Running task 'SimpleAsyncTask-5' in Thread thread_name_prefix_____5
Running task 'SimpleAsyncTask-6' in Thread thread_name_prefix_____6
Running task 'SyncTask-1' in Thread main
Running task 'SyncTask-2' in Thread main
Running task 'SyncTask-3' in Thread main
Running task 'SyncTask-4' in Thread main
Running task 'SyncTask-5' in Thread main
Running task 'ThreadPoolTask-2' in Thread ThreadPoolTaskExecutor-2
Running task 'ThreadPoolTask-1' in Thread ThreadPoolTaskExecutor-1
Running task 'ThreadPoolTask-4' in Thread ThreadPoolTaskExecutor-3
Running task 'ThreadPoolTask-3' in Thread ThreadPoolTaskExecutor-2
```

以此我们可以推断出，第一个测试为每个任务创建新的线程。通过使用不同的线程名称，我们可以看到相应区别。第二个，同步执行器，应该执行所调用线程中的任务。这里可以看到’main’是主线程的名称，它的主线程调用执行同步所有任务。最后一种例子涉及最大可创建3个线程的线程池。从结果可以看到，他们也确实只有3个创建线程。

现在，我们将编写一些单元测试来看看@Async和@Scheduled实现。

```Java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations={"classpath:applicationContext-test.xml"})
@WebAppConfiguration
public class AnnotationTest {

  @Autowired
  private GenericApplicationContext context;

  @Test
  public void testScheduled() throws InterruptedException {

      System.out.println("Start sleeping");
      Thread.sleep(6000);
      System.out.println("Wake up !");

      TestScheduledTask scheduledTask = (TestScheduledTask) context.getBean("testScheduledTask");
       /**
        * Test fixed delay. It's executed every 6 seconds. The first execution is registered after application's context start.
        **/
      assertTrue("Scheduled task should be executed 2 times (1 before sleep in this method, 1 after the sleep), but was "+scheduledTask.getFixedDelayCounter(),
        scheduledTask.getFixedDelayCounter() == 2);

       /**
        * Test fixed rate. It's executed every 6 seconds. The first execution is registered after application's context start.
        * Unlike fixed delay, a fixed rate configuration executes one task with specified time. For example, it will execute on
        * 6 seconds delayed task at 10:30:30, 10:30:36, 10:30:42 and so on - even if the task 10:30:30 taken 30 seconds to
        * be terminated. In teh case of fixed delay, if the first task takes 30 seconds, the next will be executed 6 seconds
        * after the first one, so the execution flow will be: 10:30:30, 10:31:06, 10:31:12.
        **/
      assertTrue("Scheduled task should be executed 2 times (1 before sleep in this method, 1 after the sleep), but was "+scheduledTask.getFixedRateCounter(),
        scheduledTask.getFixedRateCounter() == 2);
       /**
        * Test fixed rate with initial delay attribute. The initialDelay attribute is set to 6 seconds. It causes that
        * scheduled method is executed 6 seconds after application's context start. In our case, it should be executed
        * only once because of previous Thread.sleep(6000) invocation.
        **/
      assertTrue("Scheduled task should be executed 1 time (0 before sleep in this method, 1 after the sleep), but was "+scheduledTask.getInitialDelayCounter(), scheduledTask.getInitialDelayCounter() == 1);
       /**
        * Test cron scheduled task. Cron is scheduled to be executed every 6 seconds. It's executed only once,
        * so we can deduce that it's not invoked directly before applications
        * context start, but only after configured time (6 seconds in our case).
        **/
      assertTrue("Scheduled task should be executed 1 time (0 before sleep in this method, 1 after the sleep), but was "+scheduledTask.getCronCounter(), scheduledTask.getCronCounter() == 1);
  }

  @Test
  public void testAsyc() throws InterruptedException {
       /**
        * To test @Async annotation, we can create a bean in-the-fly. AsyncCounter bean is a
        * simple counter which value should be equals to 2 at the end of the test. A supplementary test
        * concerns threads which execute both of AsyncCounter methods: one which
        * isn't annotated with @Async and another one which is annotated with it. For the first one, invoking
        * thread should have the same name as the main thread. For annotated method, it can't be executed in
        * the main thread. It must be executed asynchrounously in a new thread.
        **/
      context.registerBeanDefinition("asyncCounter", new RootBeanDefinition(AsyncCounter.class));

      String currentName = Thread.currentThread().getName();
      AsyncCounter asyncCounter = context.getBean("asyncCounter", AsyncCounter.class);
      asyncCounter.incrementNormal();
      assertTrue("Thread executing normal increment should be the same as JUnit thread but it wasn't (expected '"+currentName+"', got '"+asyncCounter.getNormalThreadName()+"')",
                      asyncCounter.getNormalThreadName().equals(currentName));
      asyncCounter.incrementAsync();
      // sleep 50ms and give some time to AsyncCounter to update asyncThreadName value
      Thread.sleep(50);

      assertFalse("Thread executing @Async increment shouldn't be the same as JUnit thread but it wasn (JUnit thread '"+currentName+"', @Async thread '"+asyncCounter.getAsyncThreadName()+"')",
                      asyncCounter.getAsyncThreadName().equals(currentName));
      System.out.println("Main thread execution's name: "+currentName);
      System.out.println("AsyncCounter normal increment thread execution's name: "+asyncCounter.getNormalThreadName());
      System.out.println("AsyncCounter @Async increment thread execution's name: "+asyncCounter.getAsyncThreadName());
      assertTrue("Counter should be 2, but was "+asyncCounter.getCounter(), asyncCounter.getCounter()==2);
  }

}

class AsyncCounter {

  private int counter = 0;
  private String normalThreadName;
  private String asyncThreadName;

  public void incrementNormal() {
    normalThreadName = Thread.currentThread().getName();
    this.counter++;
  }

  @Async
  public void incrementAsync() {
    asyncThreadName = Thread.currentThread().getName();
    this.counter++;
  }

  public String getAsyncThreadName() {
    return asyncThreadName;
  }

  public String getNormalThreadName() {
    return normalThreadName;
  }

  public int getCounter() {
    return this.counter;
  }

}
```

另外，我们需要创建新的配置文件和一个包含定时任务方法的类：

```xml
<!-- imported configuration file first -->
<!-- Activates various annotations to be detected in bean classes -->
<context:annotation-config />

<!-- Scans the classpath for annotated components that will be auto-registered as Spring beans.
 For example @Controller and @Service. Make sure to set the correct base-package-->
<context:component-scan base-package="com.migo.test.schedulers" />

<task:scheduler id="taskScheduler"/>
<task:executor id="taskExecutor" pool-size="40" />
<task:annotation-driven executor="taskExecutor" scheduler="taskScheduler"/>
```

```Java
// scheduled methods after, all are executed every 6 seconds (scheduledAtFixedRate and scheduledAtFixedDelay start to execute at
// application context start, two other methods begin 6 seconds after application's context start)
@Component
public class TestScheduledTask {

  private int fixedRateCounter = 0;
  private int fixedDelayCounter = 0;
  private int initialDelayCounter = 0;
  private int cronCounter = 0;

  @Scheduled(fixedRate = 6000)
  public void scheduledAtFixedRate() {
    System.out.println("<R> Increment at fixed rate");
    fixedRateCounter++;
  }

  @Scheduled(fixedDelay = 6000)
  public void scheduledAtFixedDelay() {
    System.out.println("<D> Incrementing at fixed delay");
    fixedDelayCounter++;
  }

  @Scheduled(fixedDelay = 6000, initialDelay = 6000)
  public void scheduledWithInitialDelay() {
    System.out.println("<DI> Incrementing with initial delay");
    initialDelayCounter++;
  }

  @Scheduled(cron = "**/6 ** ** ** ** **")
  public void scheduledWithCron() {
    System.out.println("<C> Incrementing with cron");
    cronCounter++;

  }

  public int getFixedRateCounter() {
    return this.fixedRateCounter;
  }

  public int getFixedDelayCounter() {
    return this.fixedDelayCounter;
  }

  public int getInitialDelayCounter() {
    return this.initialDelayCounter;
  }

  public int getCronCounter() {
    return this.cronCounter;
  }

}
```

该测试的输出：

```Java
<R> Increment at fixed rate
<D> Incrementing at fixed delay
Start sleeping
<C> Incrementing with cron
<DI> Incrementing with initial delay
<R> Increment at fixed rate
<D> Incrementing at fixed delay
Wake up !
Main thread execution's name: main
AsyncCounter normal increment thread execution's name: main
AsyncCounter @Async increment thread execution's name: taskExecutor-1
```

本文向我们介绍了关于Spring框架另一个大家比较感兴趣的功能–定时任务。我们可以看到，与Linux CRON风格配置类似，这些任务同样可以按照固定的频率进行定时任务的设置。我们还通过例子证明了使用@Async注解的方法会在不同线程中执行。

# 666. 彩蛋

如果你对 Spring 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)