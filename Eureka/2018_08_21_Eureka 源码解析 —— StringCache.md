title: Eureka 源码解析 —— StringCache
date: 2018-08-21
tags:
categories: Eureka
permalink: Eureka/string-cache
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484932&idx=2&sn=a8f524d2c6cfa535f5668105a1e78115&chksm=fa4979b5cd3ef0a305248fcda5465e74afb1b9e2735ac919b1f1fd267b93d4d1552bc555ab3a&token=1051140534&lang=zh_CN#rd

---

摘要: 原创出处 http://www.iocoder.cn/Eureka/string-cache/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Eureka/string-cache/)
- [2. StringCache](http://www.iocoder.cn/Eureka/string-cache/)
- [3. 使用场景](http://www.iocoder.cn/Eureka/string-cache/)
- [666. 彩蛋](http://www.iocoder.cn/Eureka/string-cache/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2018_05_18.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

---

# 1. 概述

本文主要分享 **Eureka 自己实现的 StringCache**。

先一起来看下美团点评技术团队对 `String#intern(...)` 的分享：

> FROM [《深入解析String#intern》「 引言 」](https://tech.meituan.com/in_depth_understanding_string_intern.html)  
> 在 JAVA 语言中有8中基本类型和一种比较特殊的类型 `String`。这些类型为了使他们在运行过程中速度更快，更节省内存，都提供了一种常量池的概念。常量池就类似一个 JAVA 系统级别提供的缓存。  
> 8 种基本类型的常量池都是系统协调的，`String` 类型的常量池比较特殊。它的主要使用方法有两种：  
> 
> * 直接使用双引号声明出来的 `String` 对象会直接存储在常量池中
> * 如果不是用双引号声明的 `String` 对象，可以使用String提供的 `intern` 方法。`intern` 方法会从字符串常量池中查询当前字符串是否存在，若不存在就会将当前字符串放入常量池中

* 字符串常量池能带来速度更快，更节省内存的好处
* **非双引号声明**的 String 对象，需要使用 `String#intern()` 方法，将字符串存储到字符串常量池。

看起来一切都非常非常非常美好，那为什么 Eureka 自己实现了 StringCache ？

继续参见美团点评技术团队对 `String#intern(...)` 的分享：

> FROM [《深入解析String#intern》「 native 代码 」](https://tech.meituan.com/in_depth_understanding_string_intern.html)  
> JAVA 使用 JNI 调用 c++ 实现的 StringTable 的 `intern` 方法, StringTable的 `intern` 方法跟 Java 中的 HashMap 的实现是差不多的, 只是不能自动扩容。**默认大小是1009**。  

> 要注意的是，String 的 String Pool 是一个**固定大小**的 Hashtable，默认值大小长度是 1009，如果放进 String Pool 的 String 非常多，就会造成**Hash冲突严重**，从而导致链表会很长，而链表长了后直接会造成的影响就是当调用String.intern时性能会大幅下降（因为要一个一个找）。  

> 在 JDK6 中 StringTable 是固定的，就是 1009 的长度，所以如果常量池中的字符串过多就会导致效率下降很快。在jdk7中，StringTable的长度可以通过一个参数指定：

> * -XX:StringTableSize=99991

* JDK 自带的 String Pool 固定大小( 即使可配 )，不支持自动扩容，大量使用 `String#intern(...)` 后，会导致性能大幅度下降。
* Eureka 的应用实例( InstanceInfo ) 的 `appName`、`appGroupName`、`vipAddress`、`secureVipAddress`、`metadata` 和应用( Application )的 `name` 等属性需要使用到 String Pool ，为了在大量的网络通信序列化反序列的过程中，速度更快，更节省内容。

另外，FastJSON 在 1.124 版本**之前**也使用 `String#intern(...)` 方法，优化 JSON Key 的速度和空间，但是在大量动态 JSON Key 的场景下，反而会导致性能下降。所以 FastJSON 1.124 修复了该问题。参见如下：

> FROM [《深入解析String#intern》「 fastjson 不当使用 」](https://tech.meituan.com/in_depth_understanding_string_intern.html)  
> ![](http://www.iocoder.cn/images/Eureka/2018_08_21/01.png)

* But ，FastJSON 1.124 版本**之前**恰好适合 Eureka ，因为 `appName`、`appGroupName` **相对不那么动态**。考虑到可能还是有大量的字符串存在，因而实现自定义的 StringCache 类，以解决 StringPool 的 HashTable 不支持动态扩容的情况。

OK，下面我们来看看 Eureka 是如何实现自定义的 StringCache 类。

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



# 2. StringCache

`com.netflix.discovery.util.StringCache` ，字符串缓存。代码如下：

```Java
  1: public class StringCache {
  2: 
  3:     public static final int LENGTH_LIMIT = 38;
  4: 
  5:     private static final StringCache INSTANCE = new StringCache();
  6: 
  7:     private final ReadWriteLock lock = new ReentrantReadWriteLock();
  8:     private final Map<String, WeakReference<String>> cache = new WeakHashMap<String, WeakReference<String>>();
  9:     private final int lengthLimit;
 10: 
 11:     public StringCache() {
 12:         this(LENGTH_LIMIT);
 13:     }
 14: 
 15:     public StringCache(int lengthLimit) {
 16:         this.lengthLimit = lengthLimit;
 17:     }
 18: 
 19:     public String cachedValueOf(final String str) {
 20:         if (str != null && (lengthLimit < 0 || str.length() <= lengthLimit)) {
 21:             // Return value from cache if available
 22:             try {
 23:                 lock.readLock().lock();
 24:                 WeakReference<String> ref = cache.get(str);
 25:                 if (ref != null) {
 26:                     return ref.get();
 27:                 }
 28:             } finally {
 29:                 lock.readLock().unlock();
 30:             }
 31: 
 32:             // Update cache with new content
 33:             try {
 34:                 lock.writeLock().lock();
 35:                 WeakReference<String> ref = cache.get(str);
 36:                 if (ref != null) {
 37:                     return ref.get();
 38:                 }
 39:                 cache.put(str, new WeakReference<>(str));
 40:             } finally {
 41:                 lock.writeLock().unlock();
 42:             }
 43:             return str;
 44:         }
 45:         return str;
 46:     }
 47: 
 48:     public int size() {
 49:         try {
 50:             lock.readLock().lock();
 51:             return cache.size();
 52:         } finally {
 53:             lock.readLock().unlock();
 54:         }
 55:     }
 56: 
 57:     public static String intern(String original) {
 58:         return INSTANCE.cachedValueOf(original);
 59:     }
 60: 
 61: }
```

* `INSTANCE` 属性，字符串缓存**单例**。
* `lock` 属性，**读写锁**，保证读写互斥。
* `cache` 属性，缓存哈希表。
    * 使用 WeakHashMap，当 StringCache 被回收时，其对应的值一起被移除。
    * [《WeakHashMap和HashMap的区别》](http://blog.csdn.net/yangzl2008/article/details/6980709)
    * [《Java 集合系列13之 WeakHashMap详细介绍(源码解析)和使用示例》](http://www.cnblogs.com/skywang12345/p/3311092.html)
* `lengthLimit` 属性，缓存字符串最大长度。默认值：38 。
* `#cachedValueOf(...)` 方法，获得字符串缓存。若缓存不存在，则进行缓存。和 `String#intern()` 的逻辑相同，区别在于 `cache` 支持自动扩容。
    * 第 22 至 30 行 ：读锁，读取缓存。
    * 第 32 至 42 行 ：缓存不存在，写锁，写入缓存。 
* `#size()` 方法，缓存大小。
* `#intern()` **静态**方法，使用 `INSTANCE` 获取缓存字符串。

# 3. 使用场景

在 InstanceInfo 下的使用，点击 [链接](https://github.com/YunaiV/eureka/blob/7f868f9ca715a8862c0c10cac04e238bbf371db0/eureka-client/src/main/java/com/netflix/appinfo/InstanceInfo.java#L233) 查看。

在 Application 下的使用，点击 [链接](https://github.com/YunaiV/eureka/blob/7f868f9ca715a8862c0c10cac04e238bbf371db0/eureka-client/src/main/java/com/netflix/discovery/shared/Application.java#L95) 查看。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

又 Get 新姿势了，好开森。

胖友，分享个朋友圈，可好？！


