title: Kafka 日志清理之 Log Compaction
date: 2018-01-25
tag: 
categories: Kafka
permalink: Kafka/log-compaction
author: 朱小厮
from_url: https://blog.csdn.net/u013256816/article/details/80487758
wechat_url: 

-------

摘要: 原创出处 https://blog.csdn.net/u013256816/article/details/80487758 「朱小厮」欢迎转载，保留摘要，谢谢！

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

> 在上一篇文章《Kafka日志清理之Log Deletion》中介绍了日志清理的方式之一——日志删除，本文承接上篇，主要来介绍Log Compaction。

Kafka中的Log Compaction是指在默认的日志删除（Log Deletion）规则之外提供的一种清理过时数据的方式。如下图所示，Log Compaction对于有相同key的的不同value值，只保留最后一个版本。如果应用只关心key对应的最新value值，可以开启Kafka的日志清理功能，Kafka会定期将相同key的消息进行合并，只保留最新的value值。
![img](http://static.iocoder.cn/csdn/20180528195613195?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMyNTY4MTY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
（图1）

有很多中文资料会把Log Compaction翻译为“日志压缩”，笔者认为不够妥帖，压缩应该是指Compression，在Kafka中消息可以采用GZip、Snappy、LZ4等压缩方式进行压缩，如果把Log Compaction翻译为日志压缩，容易让人和消息压缩（Message Compression）产生关联，而实则是两个不同的概念。英文“Compaction”可以直译为“压紧、压实”，如果这里将Log Compaction直译为“日志压紧”或者“日志压实”又未免太过生硬。考虑到“日志压缩”的说法已经广为接受，笔者这里**勉强**接受此种说法，不过文中尽量直接使用英文Log Compaction来表示日志压缩。读者在遇到类似“压缩”的字眼之时需格外注意这个压缩是具体指日志压缩（Log Compaction）还是指消息压缩（Message Compression）。

Log Compaction执行前后，日志分段中的每条消息的偏移量和写入时的保持一致。Log Compaction会生成新的日志分段文件，日志分段中每条消息的物理位置会重新按照新文件来组织。Log Compaction执行过后的偏移量不再是连续的，不过这并不影响日志的查询。

Kafka的Log Compaction可以类比于Redis的SNAPSHOTTING的持久化模式。试想一下，如果一个系统使用Kafka来保存状态，每次有状态变更都会将其写入Kafka中。在某一时刻此系统异常崩溃，进而在恢复时通过读取Kafka中的消息来恢复其应有的状态，那么此系统关心的是它原本的最新状态而不是历史时刻中的每一个状态。如果Kafka的日志保存策略是日志删除（Log Deletion），那么系统势必要一股脑的读取Kafka中的所有数据来恢复，而如果日志保存策略是Log Compaction，那么可以减少数据的加载量进而加快系统的恢复速度。Log Compaction在某些应用场景下可以简化技术栈，提高系统整体的质量。

我们知道可以通过配置log.dir或者log.dirs参数来设置Kafka日志的存放目录，而对于每一个日志目录下都有一个名为“cleaner-offset-checkpoint”的文件，这个文件就是清理检查点文件，用来记录每个主题的每个分区中已清理的偏移量。通过清理检查点文件可以将日志文件（Log）分成两个部分，参考下图，通过检查点cleaner checkpoint来划分出一个已经清理过的clean部分和一个还未清理过的dirty部分。在日志清理的同时，客户端也会读取日志。dirty部分的消息偏移量是逐一递增的，而clean部分的消息偏移量是断续的，如果客户端总能赶上dirty部分，它就能读取到日志的所有消息，反之，就不可能读到全部的消息。

![img](http://static.iocoder.cn/csdn/20180528195621648?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMyNTY4MTY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
（图2）

上图中firstDirtyOffset（与cleaner checkpoint相等）表示dirty部分的起始偏移量，而firstUncleanableOffset为dirty部分的截止偏移量，整个dirty部分的偏移量范围为[firstDirtyOffset, firstUncleanableOffset)，注意这里是左闭右开区间。为了避免当前活跃的日志分段activeSegment成为热点文件，activeSegment不会参与Log Compaction的操作。同时Kafka支持通过参数log.cleaner.min.compaction.lag.ms（默认值为0）来配置消息在被清理前的最小保留时间，默认情况下firstUncleanableOffset等于activeSegment的baseOffset。

注意Log Compaction是针对key的，所以在使用时应注意每个消息的key值不为null。每个broker会启动log.cleaner.thread（默认值为1）个日志清理线程负责执行清理任务，这些线程会选择“污浊率”最高的日志文件进行清理。用cleanBytes表示clean部分的日志占用大小，dirtyBytes表示dirty部分的日志占用大小，那么这个日志的污浊率（dirtyRatio）为：

```java
dirtyRatio = dirtyBytes / (cleanBytes + dirtyBytes)
```

为了防止日志不必要的频繁清理操作，Kafka还使用了参数log.cleaner.min.cleanable.ratio（默认值为0.5）来限定可进行清理操作的最小污浊率。

> Kafka中用于保存消费者消费位移的主题“__consumer_offsets”使用的就是Log Compaction策略。

这里我们已经知道了怎样选择合适的日志文件做清理操作，然而我们怎么对日志文件中消息的key进行筛选操作呢？Kafka中的每个日志清理线程会使用一个名为“SkimpyOffsetMap”的对象来构建key与offset的映射关系的哈希表。日志清理需要遍历两次日志文件，第一次遍历把每个key的哈希值和最后出现的offset都保存在SkimpyOffsetMap中，映射模型如下图所示。第二次遍历检查每个消息是否符合保留条件，如果符合就保留下来，否则就会被清理掉。假设一条消息的offset为O1，这条消息的key在SkimpyOffsetMap中所对应的offset为O2，如果O1>=O2即为满足保留条件。
![img](http://static.iocoder.cn/csdn/20180528195630761?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMyNTY4MTY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
（图3）

默认情况下SkimpyOffsetMap使用MD5来计算key的哈希值，占用空间大小为16B，根据这个哈希值来从SkimpyOffsetMap中找到对应的槽位，如果发生冲突则用线性探测法处理。为了防止哈希冲突过于频繁，我们也可以通过broker端参数log.cleaner.io.buffer.load.factor（默认值为0.9）来调整负载因子。偏移量占用空间大小为8B，故一个映射项占用大小为24B。每个日志清理线程的SkimpyOffsetMap的内存占用大小为log.cleaner.dedupe.buffer.size / log.cleaner.thread，默认值为 = 128MB/1 = 128MB。所以默认情况下SkimpyOffsetMap可以保存128MB * 0.9 /24B ≈ 5033164个key的记录。假设每条消息的大小为1KB，那么这个SkimpyOffsetMap可以用来映射4.8GB的日志文件，而如果有重复的key，那么这个数值还会增大，整体上来说SkimpyOffsetMap极大的节省了内存空间且非常高效。

> “SkimpyOffsetMap”这个取名也很有意思，“Skimpy”可以直译为“不足的”，可以看出它最初的设计者也认为这种实现不够严谨。如果遇到两个不同的key但哈希值相同的情况，那么其中一个key所对应的消息就会丢失。虽然说MD5这类摘要算法的冲突概率非常小，但根据墨菲定律，任何一个事件，只要具有大于0的几率，就不能假设它不会发生，所以在使用Log Compaction策略时要注意这一点。

Log Compaction会为我们保留key相应的最新value值，那么当我们需要删除一个key怎么办？Kafka中提供了一个墓碑消息（tombstone）的概念，如果一条消息的key不为null，但是其value为null，那么此消息就是墓碑消息。日志清理线程发现墓碑消息时会先进行常规的清理，并保留墓碑消息一段时间。墓碑消息的保留条件是当前墓碑消息所在的日志分段的最近修改时间lastModifiedTime大于deleteHorizonMs，参考图2，这个deleteHorizonMs的计算方式为clean部分中最后一个日志分段的最近修改时间减去保留阈值deleteRetionMs（通过broker端参数log.cleaner.delete.retention.ms配置，默认值为86400000，即24小时）的大小，即：

```java
deleteHorizonMs = clean部分中最后一个LogSegment的lastModifiedTime - deleteRetionMs
```

所以墓碑消息的保留条件为：

```
所在LogSegment的lastModifiedTime > deleteHorizonMs
=> 所在LogSegment的lastModifiedTime > clean部分中最后一个LogSegment的lastModifiedTime - deleteRetionMs
=> 所在LogSegment的lastModifiedTime + deleteRetionMs > clean部分中最后一个LogSegment的lastModifiedTime
(可以对照图2中的deleteRetionMs所标记的位置去理解)
```

Log Compaction执行过后的日志分段的大小会比原先的日志分段的要小，为了防止出现太多的小文件，Kafka在实际清理过程中并不对单个的日志分段进行单独清理，而是会将日志文件中offset从0至firstUncleanableOffset的所有日志分段进行分组，每个日志分段只属于一组，分组策略为：按照日志分段的顺序遍历，每组中日志分段的占用空间大小之和不超过segmentSize（可以通过broker端参数log.segments.bytes设置，默认值为1GB），且对应的索引文件占用大小之和不超过maxIndexSize（可以通过broker端参数log.index.interval.bytes设置，默认值为10MB）。同一个组的多个日志分段清理过后，只会生成一个新的日志分段。

![img](http://static.iocoder.cn/csdn/20180528195641722?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMyNTY4MTY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
（图4）

参考上图，假设所有的参数配置都为默认值，在Log Compaction之前checkpoint的初始值为0。执行第一次Log Compaction之后，每个非活跃的日志分段的大小都有所缩减，checkpoint的值也有所变化。执行第二次Log Compaction时会将组队成[0.4GB, 0.4GB]、[0.3GB, 0.7GB]、[0.3GB]、[1GB]这4个分组，并且从第二次Log Compaction开始还会涉及墓碑消息的清除。同理，第三次Log Compaction过后的情形可参考上图尾部。Log Compaction过程中会将对每个日志分组中需要保留的消息拷贝到一个以“.clean”为后缀的临时文件中，此临时文件以当前日志分组中第一个日志分段的文件名命名，例如：00000000000000000000.log.clean。Log Compaction过后将“.clean”的文件修改为以“.swap”后缀的文件，例如：00000000000000000000.log.swap，然后删除掉原本的日志文件，最后才把文件的“.swap”后缀去掉，整个过程中的索引文件的变换也是如此，至此一个完整Log Compaction操作才算完成。

以上是整个日志压缩（Log Compaction）过程的详解，读者需要注意将日志压缩和日志删除区分开，日志删除是指清除整个日志分段，而日志压缩是针对相同key的消息的合并清理。

# 666. 彩蛋

如果你对 Kafka 并发感兴趣，欢迎加入我的知识一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)