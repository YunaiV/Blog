title: Kafka 解惑之 Old Producer（3）—— Async Analysis
date: 2018-01-11
tag:
categories: Kafka
permalink: Kafka/old-producer-async-analysis
author: 朱小厮
from_url: https://blog.csdn.net/u013256816/article/details/79230518
wechat_url:

-------

摘要: 原创出处 https://blog.csdn.net/u013256816/article/details/79230518 「朱小厮」欢迎转载，保留摘要，谢谢！


-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

讲述完了Sync模式下的结构脉络，下面就来聊一聊Async的，Async会将客户端所要发送的消息暂存在LinkedBlockingQueue中，然后通过特制的ProducerSendThread来根据条件发送。这个LinkedBlockingQueue的长度大小是通过queue.buffering.max.messages这个参数设置的，默认值为10000。启用异步模式时，producer缓存队列里最大缓存的消息数量，如果超过这个值，producer就会阻塞或者丢掉消息。

在讲述Old Producer的开篇，我们展示过sync和async的代码，不妨这里在赘述一下：

```java
  config.producerType match {
    case "sync" =>
    case "async" =>
      sync = false
      producerSendThread = new ProducerSendThread[K,V]("ProducerSendThread-" + config.clientId,
                                                       queue,
                                                       eventHandler,
                                                       config.queueBufferingMaxMs,
                                                       config.batchNumMessages,
                                                       config.clientId)
      producerSendThread.start()
  }
```

这里可以看到sync的情况什么都不需要做，而async的情况就需要开启ProducerSendThread, 而在ProducerSendThread构造函数列表中的queue就是指客户端暂存消息的LinkedBlockingQueue。
注意ProducerSendThread构造函数列表中的config.queueBufferingMaxMs和config.batchNumMessages两个参数分表对应Producer端的配置queue.buffering.max.ms和batch.num.messages，默认值分别为5000和200，这两个参数后面会有用途。

> 注意在Scala语言中是没有break语句的，这点与Java不同，Scala的相当于每个case语句末尾都加上了一个break语句，所以读到相关源码的同学不要误以为是Java中语法的那种情况。

在async模式下，消息发送不是直接调用sync模式下的DefaultEventHandler的handle()方法，而是调用kafka.producer.Producer的asyncSend方法如下（只展示主要内容）：

```java
private def asyncSend(messages: Seq[KeyedMessage[K,V]]) {
  for (message <- messages) {
    val added = config.queueEnqueueTimeoutMs match {
      case 0  =>
        queue.offer(message)
      case _  =>
        try {
          if (config.queueEnqueueTimeoutMs < 0) {
            queue.put(message)
            true
          } else {
            queue.offer(message, config.queueEnqueueTimeoutMs, TimeUnit.MILLISECONDS)
          }
        }
        catch {
          case _: InterruptedException =>
            false
        }
    }}}
```

可以看到消息入队（存入LinkedBlockingQueue）中受到queue.enqueue.timeout.ms参数的影响，这个参数表示当LinkedBlockingQueue中消息达到上限queue.buffering.max.messages个数时所需要等待的时间。如果queue.enqueue.timeout.ms参数设置为0，表示不需要等待直接丢弃消息；如果设置为-1（默认值）则队列满时会阻塞等待。

消息存入LinkedBlockingQueue中就需要一个异步的线程ProducerSendThread来执行发送消息的操作，这个操作主要是通过kafka.producer.async.ProducerSendThread类中的processEvents方法来执行。processEvents方法的具体细节如下：

```java
private def processEvents() {
  var lastSend = Time.SYSTEM.milliseconds
  var events = new ArrayBuffer[KeyedMessage[K,V]]
  var full: Boolean = false

  // drain the queue until you get a shutdown command
  Iterator.continually(queue.poll(scala.math.max(0, (lastSend + queueTime) - Time.SYSTEM.milliseconds), TimeUnit.MILLISECONDS))
                    .takeWhile(item => if(item != null) item ne shutdownCommand else true).foreach {
    currentQueueItem =>
      // check if the queue time is reached. This happens when the poll method above returns after a timeout and
      // returns a null object
      val expired = currentQueueItem == null
      if(currentQueueItem != null) {
        events += currentQueueItem
      }

      // check if the batch size is reached
      full = events.size >= batchSize

      if(full || expired) {
        // if either queue time has reached or batch size has reached, dispatch to event handler
        tryToHandle(events)
        lastSend = Time.SYSTEM.milliseconds
        events = new ArrayBuffer[KeyedMessage[K,V]]
      }
  }
  // send the last batch of events
  tryToHandle(events)
  if(queue.size > 0)
    throw new IllegalQueueStateException("Invalid queue state! After queue shutdown, %d remaining items in the queue"
      .format(queue.size))
}
```

一长串Scala源码会不会看的一头雾水？这里来一步一步的分析一下：

1. 首先持续的拉取queue(LinkedBlockingQueue)中的消息，注意这里用的是poll(long timeout, TimeUnit unit)方法，这个表示这里会等待一段时间之后再拉取队列中的消息，这个等待的时间由queueTime也就是queue.buffering.max.ms这个参数设置。比如我们设置成1000时（默认为5000），它会大致缓存1s的数据再一次发送出去，这样可以极大的增加broker吞吐量，但也会造成时效性的降低。
2. 如果拉取到了消息，那么就存储缓存events(ArrayBuffer[KeyedMessage[K,V]])中，等到events中的消息大于batchSize大小，也就是batch.num.messages个数时再调用tryToHandle(events）来处理消息。
3. tryToHandle(events）就是调用DefaultEventHandler类中的handle()方法，接下去的工作就和Sync模式的相同。
4. 如果在等待的时间内没有获取到相应的消息，那么无需等待 events.size >= batchSize条件的满足就可以发送消息。

在讲述Sync模式的时候笔者画过一份结构图，这里也来画一幅Async结构图来收尾，与Sync模式的类似，具体如下：
![这里写图片描述](http://static.iocoder.cn/csdn/20180201164529831?)

# 666. 彩蛋

如果你对 Kafka 并发感兴趣，欢迎加入我的知识一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)