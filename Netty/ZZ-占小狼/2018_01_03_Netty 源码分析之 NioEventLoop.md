title: 【占小狼】Netty 源码分析之 NioEventLoop
date: 2018-01-03
tags:
categories: Netty
permalink: Netty/zhanxiaolang/NioEventLoop
author: 占小狼
from_url: https://www.jianshu.com/p/9acf36f7e025
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484514&idx=1&sn=be55d170466b057c171c7643d048180a&chksm=fa497bd3cd3ef2c5dc5b46b7721d4b078420d130d80b95d2a76ca048786d8071da904ec0af40#rd

-------

摘要: 原创出处 https://www.jianshu.com/p/9acf36f7e025 「占小狼」欢迎转载，保留摘要，谢谢！


-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

上一章节中，我们分析了Netty服务的启动过程，本章节分析Netty的NioEventLoop是如工作的。

![img](https://upload-images.jianshu.io/upload_images/2184951-2e248d85df2a1a86.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/490)

NioEventLoop中维护了一个线程，线程启动时会调用NioEventLoop的run方法，执行I/O任务和非I/O任务：

**I/O任务**
即selectionKey中ready的事件，如accept、connect、read、write等，由processSelectedKeys方法触发。

**非IO任务**
添加到taskQueue中的任务，如register0、bind0等任务，由runAllTasks方法触发。

两种任务的执行时间比由变量ioRatio控制，默认为50，则表示允许非IO任务执行的时间与IO任务的执行时间相等。

**NioEventLoop.run** 方法实现

```Java
protected void run() {
    for (;;) {
        boolean oldWakenUp = wakenUp.getAndSet(false);
        try {
            if (hasTasks()) {
                selectNow();
            } else {
                select(oldWakenUp);
                if (wakenUp.get()) {
                    selector.wakeup();
                }
            }

            cancelledKeys = 0;
            needsToSelectAgain = false;
            final int ioRatio = this.ioRatio;
            if (ioRatio == 100) {
                processSelectedKeys();
                runAllTasks();
            } else {
                final long ioStartTime = System.nanoTime();

                processSelectedKeys();

                final long ioTime = System.nanoTime() - ioStartTime;
                runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
            }

            if (isShuttingDown()) {
                closeAll();
                if (confirmShutdown()) {
                    break;
                }
            }
        } catch (Throwable t) {
            logger.warn("Unexpected exception in the selector loop.", t);

            // Prevent possible consecutive immediate failures that lead to
            // excessive CPU consumption.
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                // Ignore.
            }
        }
    }
}
```

hasTasks()方法判断当前taskQueue是否有元素。
1、 如果taskQueue中有元素，执行 **selectNow()** 方法，最终执行selector.selectNow()，该方法会立即返回。

```Java
void selectNow() throws IOException {
    try {
        selector.selectNow();
    } finally {
        // restore wakup state if needed
        if (wakenUp.get()) {
            selector.wakeup();
        }
    }
}
```

2、 如果taskQueue没有元素，执行 **select(oldWakenUp)** 方法，代码如下：

```Java
private void select(boolean oldWakenUp) throws IOException {
    Selector selector = this.selector;
    try {
        int selectCnt = 0;
        long currentTimeNanos = System.nanoTime();
        long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);
        for (;;) {
            long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
            if (timeoutMillis <= 0) {
                if (selectCnt == 0) {
                    selector.selectNow();
                    selectCnt = 1;
                }
                break;
            }

            int selectedKeys = selector.select(timeoutMillis);
            selectCnt ++;

            if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
                // - Selected something,
                // - waken up by user, or
                // - the task queue has a pending task.
                // - a scheduled task is ready for processing
                break;
            }
            if (Thread.interrupted()) {
                // Thread was interrupted so reset selected keys and break so we not run into a busy loop.
                // As this is most likely a bug in the handler of the user or it's client library we will
                // also log it.
                //
                // See https://github.com/netty/netty/issues/2426
                if (logger.isDebugEnabled()) {
                    logger.debug("Selector.select() returned prematurely because " +
                            "Thread.currentThread().interrupt() was called. Use " +
                            "NioEventLoop.shutdownGracefully() to shutdown the NioEventLoop.");
                }
                selectCnt = 1;
                break;
            }

            long time = System.nanoTime();
            if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
                // timeoutMillis elapsed without anything selected.
                selectCnt = 1;
            } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                    selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
                // The selector returned prematurely many times in a row.
                // Rebuild the selector to work around the problem.
                logger.warn(
                        "Selector.select() returned prematurely {} times in a row; rebuilding selector.",
                        selectCnt);

                rebuildSelector();
                selector = this.selector;

                // Select again to populate selectedKeys.
                selector.selectNow();
                selectCnt = 1;
                break;
            }

            currentTimeNanos = time;
        }

        if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS) {
            if (logger.isDebugEnabled()) {
                logger.debug("Selector.select() returned prematurely {} times in a row.", selectCnt - 1);
            }
        }
    } catch (CancelledKeyException e) {
        if (logger.isDebugEnabled()) {
            logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector - JDK bug?", e);
        }
        // Harmless exception - log anyway
    }
}
```

**这个方法解决了Nio中臭名昭著的bug：selector的select方法导致cpu100%。**
1、delayNanos(currentTimeNanos)：计算延迟任务队列中第一个任务的到期执行时间（即最晚还能延迟多长时间执行），默认返回1s。每个SingleThreadEventExecutor都持有一个延迟执行任务的优先队列PriorityQueue，启动线程时，往队列中加入一个任务。

```Java
protected long delayNanos(long currentTimeNanos) {
    ScheduledFutureTask<?> delayedTask = delayedTaskQueue.peek();
    if (delayedTask == null) {
        return SCHEDULE_PURGE_INTERVAL;
    }
    return delayedTask.delayNanos(currentTimeNanos);
}

//ScheduledFutureTask
public long delayNanos(long currentTimeNanos) {
    return Math.max(0, deadlineNanos() - (currentTimeNanos - START_TIME));
}
public long deadlineNanos() {
    return deadlineNanos;
}
```

2、如果延迟任务队列中第一个任务的最晚还能延迟执行的时间小于500000纳秒，且selectCnt == 0（selectCnt 用来记录selector.select方法的执行次数和标识是否执行过selector.selectNow()），则执行selector.selectNow()方法并立即返回。
3、否则执行selector.select(timeoutMillis)，这个方法已经在[深入浅出NIO Socket](https://www.jianshu.com/p/0d497fe5484a)分析过。
4、如果已经存在ready的selectionKey，或者selector被唤醒，或者taskQueue不为空，或则scheduledTaskQueue不为空，则退出循环。
5、如果 selectCnt 没达到阈值SELECTOR_AUTO_REBUILD_THRESHOLD（默认512），则继续进行for循环。其中 currentTimeNanos 在select操作之后会重新赋值当前时间，如果selector.select(timeoutMillis)行为真的阻塞了timeoutMillis，第二次的timeoutMillis肯定等于0，此时selectCnt 为1，所以会直接退出for循环。
6、如果触发了epool cpu100%的bug，会发生什么？
selector.select(timeoutMillis)操作会立即返回，不会阻塞timeoutMillis，导致 currentTimeNanos 几乎不变，这种情况下，会反复执行selector.select(timeoutMillis)，变量selectCnt 会逐渐变大，当selectCnt 达到阈值，则执行rebuildSelector方法，进行selector重建，解决cpu占用100%的bug。

```Java
public void rebuildSelector() {
        if (!inEventLoop()) {
            execute(new Runnable() {
                @Override
                public void run() {
                    rebuildSelector();
                }
            });
            return;
        }
        final Selector oldSelector = selector;
        final Selector newSelector;
        if (oldSelector == null) {
            return;
        }
        try {
            newSelector = openSelector();
        } catch (Exception e) {
            logger.warn("Failed to create a new Selector.", e);
            return;
        }
        // Register all channels to the new Selector.
        int nChannels = 0;
        for (;;) {
            try {
                for (SelectionKey key: oldSelector.keys()) {
                    Object a = key.attachment();
                    try {
                        if (key.channel().keyFor(newSelector) != null) {
                            continue;
                        }
                        int interestOps = key.interestOps();
                        key.cancel();
                        key.channel().register(newSelector, interestOps, a);
                        nChannels ++;
                    } catch (Exception e) {
                        logger.warn("Failed to re-register a Channel to the new Selector.", e);
                        if (a instanceof AbstractNioChannel) {
                            AbstractNioChannel ch = (AbstractNioChannel) a;
                            ch.unsafe().close(ch.unsafe().voidPromise());
                        } else {
                            @SuppressWarnings("unchecked")
                            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
                            invokeChannelUnregistered(task, key, e);
                        }
                    }
                }
            } catch (ConcurrentModificationException e) {
                // Probably due to concurrent modification of the key set.
                continue;
            }

            break;
        }
        selector = newSelector;
        try {
            // time to close the old selector as everything else is registered to the new one
            oldSelector.close();
        } catch (Throwable t) {
            if (logger.isWarnEnabled()) {
                logger.warn("Failed to close the old Selector.", t);
            }
        }
        logger.info("Migrated " + nChannels + " channel(s) to the new Selector.");
    }
```

rebuildSelector过程：
1、通过方法openSelector创建一个新的selector。
2、将old selector的selectionKey执行cancel。
3、将old selector的channel重新注册到新的selector中。

对selector进行rebuild后，需要重新执行方法selectNow，检查是否有已ready的selectionKey。

方法selectNow()或select(oldWakenUp)返回后，执行方法processSelectedKeys和runAllTasks。
1、**processSelectedKeys** 用来处理有事件发生的selectkey，这里对优化过的方法processSelectedKeysOptimized进行分析：

```Java
private void processSelectedKeysOptimized(SelectionKey[] selectedKeys) {
    for (int i = 0;; i ++) {
        final SelectionKey k = selectedKeys[i];
        if (k == null) {
            break;
        }
        // null out entry in the array to allow to have it GC'ed once the Channel close
        // See https://github.com/netty/netty/issues/2363
        selectedKeys[i] = null;

        final Object a = k.attachment();

        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            @SuppressWarnings("unchecked")
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task);
        }

        if (needsToSelectAgain) {
            // null out entries in the array to allow to have it GC'ed once the Channel close
            // See https://github.com/netty/netty/issues/2363
            for (;;) {
                i++;
                if (selectedKeys[i] == null) {
                    break;
                }
                selectedKeys[i] = null;
            }

            selectAgain();
            // Need to flip the optimized selectedKeys to get the right reference to the array
            // and reset the index to -1 which will then set to 0 on the for loop
            // to start over again.
            //
            // See https://github.com/netty/netty/issues/1523
            selectedKeys = this.selectedKeys.flip();
            i = -1;
        }
    }
}
```

在优化过的方法中，有事件发生的selectkey存放在数组selectedKeys中，通过遍历selectedKeys，处理每一个selectkey，具体处理过程，会在后续进行分析。

2、**runAllTasks** 处理非I/O任务。
如果  ioRatio 不为100时，方法runAllTasks的执行时间只能为ioTime * (100 - ioRatio) / ioRatio，其中ioTime 是方法processSelectedKeys的执行时间。

```Java
protected boolean runAllTasks(long timeoutNanos) {
    fetchFromScheduledTaskQueue();
    Runnable task = pollTask();
    if (task == null) {
        return false;
    }

    final long deadline = ScheduledFutureTask.nanoTime() + timeoutNanos;
    long runTasks = 0;
    long lastExecutionTime;
    for (;;) {
        try {
            task.run();
        } catch (Throwable t) {
            logger.warn("A task raised an exception.", t);
        }
        runTasks ++;
        // Check timeout every 64 tasks because nanoTime() is relatively expensive.
        // XXX: Hard-coded value - will make it configurable if it is really a problem.
        if ((runTasks & 0x3F) == 0) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            if (lastExecutionTime >= deadline) {
                break;
            }
        }
        task = pollTask();
        if (task == null) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            break;
        }
    }
    this.lastExecutionTime = lastExecutionTime;
    return true;
}
```

方法**fetchFromScheduledTaskQueue**把scheduledTaskQueue中已经超过延迟执行时间的任务移到taskQueue中等待被执行。

```Java
private void fetchFromScheduledTaskQueue() {
    if (hasScheduledTasks()) {
        long nanoTime = AbstractScheduledEventExecutor.nanoTime();
        for (;;) {
            Runnable scheduledTask = pollScheduledTask(nanoTime);
            if (scheduledTask == null) {
                break;
            }
            taskQueue.add(scheduledTask);
        }
    }
}
```

依次从taskQueue任务task执行，每执行64个任务，进行耗时检查，如果已执行时间超过预先设定的执行时间，则停止执行非IO任务，避免非IO任务太多，影响IO任务的执行。

# 666. 彩蛋

如果你对 Netty 并发感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)