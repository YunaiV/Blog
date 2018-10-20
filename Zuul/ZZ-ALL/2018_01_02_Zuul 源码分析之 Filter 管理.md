title: Zuul 源码分析之 Filter 管理
date: 2018-01-02
tag: 
categories: Zuul
permalink: Zuul/haha7289/filter-manager
author: haha7289
from_url: https://blog.csdn.net/haha7289/article/details/54344150
wechat_url: 

-------

摘要: 原创出处 https://blog.csdn.net/haha7289/article/details/54344150 「haha7289」欢迎转载，保留摘要，谢谢！

- [`FilterFileManager`](http://www.iocoder.cn/Zuul/haha7289/filter-manager/)
- [FilterLoader](http://www.iocoder.cn/Zuul/haha7289/filter-manager/)
- [FilterRegistry](http://www.iocoder.cn/Zuul/haha7289/filter-manager/)
- [DynamicCodeCompiler](http://www.iocoder.cn/Zuul/haha7289/filter-manager/)
- [总结](http://www.iocoder.cn/Zuul/haha7289/filter-manager/)
- [参考](http://www.iocoder.cn/Zuul/haha7289/filter-manager/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

zuul支持动加载Filter类文件。实现原理是监控存放Filter文件的目录，定期扫描这些目录，如果发现有新Filter源码文件或者Filter源码文件有改动，则对文件进行编译加载。目前zuul支持使用Groovy编写的Filter。

# `FilterFileManager`

`FilterFileManager`用于管理Filter存放目录，并定期扫描目录的变化。
![FilterFileManager的类图](http://static.iocoder.cn/csdn/20170113181141977?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaGFoYTcyODk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
他的功能如下：

- 开启一个线程，开始轮询

```Java
void startPoller() {
        poller = new Thread("GroovyFilterFileManagerPoller") {
            public void run() {
                while (bRunning) {
                    try {
                        sleep(pollingIntervalSeconds * 1000);
                        manageFiles();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }
        };
        poller.setDaemon(true);
        poller.start();
    }
```

- 每次轮询，处理目录内的所有*.groovy文件，即调用`FilterLoader.getInstance().putFilter(file);`

```Java
void processGroovyFiles(List<File> aFiles)  {
        for (File file : aFiles) {
            FilterLoader.getInstance().putFilter(file);
        }
    }

    void manageFiles() throws Exception, IllegalAccessException, InstantiationException {
        List<File> aFiles = getFiles();
        processGroovyFiles(aFiles);
    }
```

# FilterLoader

编译、加载filter文件，并且检查源文件是否有变更，除此之外，它还按照`filterType`组织并维护`List<ZuulFilter>`

![FilterLoader类图](http://static.iocoder.cn/csdn/20170110112228013?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaGFoYTcyODk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

主要逻辑

```Java
 public boolean putFilter(File file) throws Exception {
    String sName = file.getAbsolutePath() + file.getName();
    // 如果文件在上次加载后发生了变化，重新编译加载
    if (filterClassLastModified.get(sName) != null && (file.lastModified() != filterClassLastModified.get(sName))) {
        filterRegistry.remove(sName);
    }
    ZuulFilter filter = filterRegistry.get(sName);
    if (filter == null) {
        // 编译、加载文件
        Class clazz = COMPILER.compile(file);
        if (!Modifier.isAbstract(clazz.getModifiers())) {
            filter = (ZuulFilter) FILTER_FACTORY.newInstance(clazz);
            // 为了下次Request使用filter，清空filter.filterType()类型的List<Filter>缓存，下次Request重新构建
            List<ZuulFilter> list = hashFiltersByType.get(filter.filterType());
            if (list != null) {
                hashFiltersByType.remove(filter.filterType()); //rebuild this list
            }
            // filterRegistry 管理所有的filter，
            filterRegistry.put(file.getAbsolutePath() + file.getName(), filter);
            // 记录filter文件更新时间。
            filterClassLastModified.put(sName, file.lastModified());
            return true;
            }
        }
        return false;
    }
```

这里主要的逻辑是把Groovy源码进行编译并加载进jvm里。

# FilterRegistry

用于管理加载的filter，数据结构比较简单，使用
`ConcurrentHashMap<String, ZuulFilter> filters`，启动key为filter的name：`file.getAbsolutePath() + file.getName();`

# DynamicCodeCompiler

是一个接口，定义两种加载编译源码的方法：

```Java
    /**
        方法最后返回的是Class，即源码编译成字节码后，还要加载。
        需要支持热加载，文件变化更新等。
     */
    Class compile(String sCode, String sName) throws Exception;

    Class compile(File file) throws Exception;
```

# 总结

zuul整体框架比较简单，如果要实现API-Gateway，或者client-api adapter code，可以参考下。框架背后的思想也许更值得我们参考。

# 参考

- [Announcing Zuul: Edge Service in the Cloud](http://techblog.netflix.com/2013/06/announcing-zuul-edge-service-in-cloud.html)
- [Building Microservices: Using an API Gateway](https://www.nginx.com/blog/building-microservices-using-an-api-gateway/?utm_source=introduction-to-microservices&utm_medium=blog)
- [Embracing the Differences : Inside the Netflix API Redesign](http://techblog.netflix.com/2012/07/embracing-differences-inside-netflix.html)
- [Pattern: API Gateway](http://microservices.io/patterns/apigateway.html)
- [Optimizing the Netflix API](http://techblog.netflix.com/2013/01/optimizing-netflix-api.html)

# 666. 彩蛋

如果你对 Zuul  感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)