title: 深入浅出高性能服务发现、配置框架 Nacos 系列 3 ：Nacos客户端初始化流程
date: 2010-01-11
tag:
categories: Nacos
permalink: Nacos/wyp57801314/client-init
author: wyp578013140
from_url: http://blog.51cto.com/2662087/2182525
wechat_url:

-------

摘要: 原创出处 http://blog.51cto.com/2662087/2182525 「wyp578013140」欢迎转载，保留摘要，谢谢！

- [总结](http://www.iocoder.cn/Nacos/wyp57801314/client-init/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

上一章节，我们从全局了解了一下[Nacos](https://github.com/alibaba/nacos)项目的模块架构，做到了心中有数，现在，我们去逐步去挖掘里面的代码细节，很多人在学习开源的时候，无从下手，代码那么多，从哪个地方开始看呢？我们可以从一个接口开始入手，这个接口是你使用过的，知道它大概做什么事，有体感的，大家还记得第一章时，我们写的HelloWorld吗，对，就从里面的接口开始剥洋葱。

https://github.com/alibaba/nacos，这个是[Nacos](https://github.com/alibaba/nacos)的github代码地址，开始之前先start关注一下，加上watch，后续[Nacos](https://github.com/alibaba/nacos)的邮件列表也会通知到你，可以关注到[Nacos](https://github.com/alibaba/nacos)的最新实时消息，及各大牛之间的精彩讨论。

下面这段代码，是第一章节发布一个服务的代码：

```Java
public static void main(String[] args) throws NacosException, InterruptedException {
    //发布的服务名
    String serviceName = "helloworld.services";
    //构造一个Nacos实例，入参是Nacos server的ip和服务端口
    NamingService naming = NacosFactory.createNamingService("100.81.0.34:8080");
    //发布一个服务，该服务对外提供的ip为127.0.0.1，端口为8888
    naming.registerInstance(serviceName, "100.81.0.35", 8080);
    Thread.sleep(Integer.MAX_VALUE);}
```

其中，第一步，是构造一个[Nacos](https://github.com/alibaba/nacos)服务实例，构造实例的入参，是一个String,值的规范为ip:port，这个ip，就是我们任意一台Nacos server的地址，我们点进去看这个方法：

```Java
public static NamingService createNamingService(String serverAddr) throws NacosException {
return NamingFactory.createNamingService(serverAddr);
}
```

同时我们看下创建配置服务实例的代码：

```Java
public static ConfigService createConfigService(String serverAddr) throws NacosException {
   return ConfigFactory.createConfigService(serverAddr);}
```

我们可以看到，NacosFactory实际上是一个服务发现和配置管理接口的统一入口，再由它不通的方法，创建不同服务的实例，我们可以直接使用NamingFactory，或者ConfigFactory直接创建[Nacos](https://github.com/alibaba/nacos)的服务实例，也能work

![img](https://cdn.nlark.com/lark/0/2018/png/4232/1537627839251-445ff4a4-d8ec-4cb0-abf5-ffd86d771bec.png)

接下来，看一下，是如何构造出这个[Nacos](https://github.com/alibaba/nacos) naming实例的：

```Java
public static NamingService createNamingService(String serverList) throws NacosException {
   try {
      Class<?> driverImplClass = Class.forName("com.alibaba.nacos.client.naming.NacosNamingService");
      Constructor constructor = driverImplClass.getConstructor(String.class);
      NamingService vendorImpl = (NamingService) constructor.newInstance(serverList);
      return vendorImpl;
   } catch (Throwable e) {
      throw new NacosException(-400, e.getMessage());
   }}
```

通过反射实例化出了一个NamingService的实例NacosNamingService，构造器是一个带String入参的，我们顺着往下看，构造函数里面做了哪些事情：

```Java
public NacosNamingService(String serverList) {

    this.serverList = serverList;
    init();
    eventDispatcher = new EventDispatcher();
    serverProxy = new NamingProxy(namespace, endpoint, serverList);
    beatReactor = new BeatReactor(serverProxy);
    hostReactor = new HostReactor(eventDispatcher, serverProxy, cacheDir);}
```

入参serverList就是我们刚才传入的服务端地址，值赋给了实例的serverList字段，接下来调用了一个init方法，这个方法里面如下：

```Java
private void init() {
    namespace = System.getProperty(PropertyKeyConst.NAMESPACE);
    if (StringUtils.isEmpty(namespace)) {
        namespace = UtilAndComs.DEFAULT_NAMESPACE_ID;
    }
    logName = System.getProperty(UtilAndComs.NACOS_NAMING_LOG_NAME);
    if (StringUtils.isEmpty(logName)) {
        logName = "naming.log";
    }
    cacheDir = System.getProperty("com.alibaba.nacos.naming.cache.dir");
    if (StringUtils.isEmpty(cacheDir)) {
        cacheDir = System.getProperty("user.home") + "/nacos/naming/" + namespace;
    }}
```

这面做了3件事，给namespace，logName，cacheDir赋值，namespace我们么有传入，默认是default，namespace在[Nacos](https://github.com/alibaba/nacos)里面的作用，是用来进行本地缓存隔离的，一台机器上，启动一个[Nacos](https://github.com/alibaba/nacos)的客户端进程，默认的本地缓存路径是default，如果再启动一个，需要重新设置一个namespace，否则就会复用之前的缓存，造成冲突；logName和cacheDir，这2个字段就不解释了，字面理解。这里多说一句，这些值的设置，可以在java启动时，通过系统参数的形式传入，并且是第一优先级的。

init方法执行完之后，接下来是实例化一些框架组件，EventDispatcher，这个是一个经典的事件分发组件，它的工作模式如下：

![img](https://cdn.nlark.com/lark/0/2018/png/4232/1537630615968-2e8b4eeb-af29-47ac-aef2-34d1783343a9.png)

会有一个单独线程从blockQueue中获取事件，这个事件在[Nacos](https://github.com/alibaba/nacos)这里， 就是服务端推送下来的数据，listener在我们订阅一条数据时，会从生成一个listener实例，在事件到了队列中，找到对应的listener，去执行里面listener的回调函数onEvent。如果对这个模式不熟悉的同学，可以再翻看下EventDispatcher的代码，这个属于基础知识了，和业务没有关系，这里就不过多详细讲解，篇幅太长。

接下来，实例化了一个NameProxy的组件，这个东西是干嘛的呢？我们看下里面代码：

```Java
public NamingProxy(String namespace, String endpoint, String serverList) {
    this.namespace = namespace;
    this.endpoint = endpoint;
    if (StringUtils.isNotEmpty(serverList)) {
        this.serverList = Arrays.asList(serverList.split(","));
        if (this.serverList.size() == 1) {
            this.nacosDomain = serverList;
        }
    }
    executorService = new ScheduledThreadPoolExecutor(1, new ThreadFactory() {
        @Override
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r);
            t.setName("com.taobao.vipserver.serverlist.updater");
            t.setDaemon(true);
            return t;
        }
    });
    executorService.scheduleWithFixedDelay(new Runnable() {
        @Override
        public void run() {
            refreshSrvIfNeed();
        }
    }, 0, vipSrvRefInterMillis, TimeUnit.MILLISECONDS);
    refreshSrvIfNeed();}
```

这里面逻辑有些多，我总结下，主要是启动了一个线程，每隔30s，去执行refreshSrvIfNeed()这个方法，

refreshSrvIfNeed()这个方法里面，做的事情，是通过一个http请求，去Nacos server获取一串Nacos server集群的地址列表，具体代码如下：

```Java
  private void refreshSrvIfNeed() {
        try {
            if (!CollectionUtils.isEmpty(serverList)) {
                LogUtils.LOG.info("server list provided by user: " + serverList);
                return;
            }
            if (System.currentTimeMillis() - lastSrvRefTime < vipSrvRefInterMillis) {
                return;
            }
            List<String> list = getServerListFromEndpoint();
            if (list.isEmpty()) {
                throw new Exception("Can not acquire vipserver list");
            }
            if (!CollectionUtils.isEqualCollection(list, serversFromEndpoint)) {
                LogUtils.LOG.info("SERVER-LIST", "server list is updated: " + list);
            }
            serversFromEndpoint = list;
            lastSrvRefTime = System.currentTimeMillis();
        } catch (Throwable e) {
            LogUtils.LOG.warn("failed to update server list", e);
        }
    }
```

获取完地址列表后，赋值给serversFromEndpoint，并且记录当前更新时间，在下一次更新时，小于30s，就不更新，避免频繁更新，总的来说，NameProxy的目的就是定时在客户端维护[Nacos](https://github.com/alibaba/nacos)服务端的最新地址列表。

我们继续往下看，接下来初始化了BeatReactor这个组件，从名字可以猜测，应该是和心跳相关的事情，它初始化的代码如下：

```Java
public BeatReactor(NamingProxy serverProxy) {
    this.serverProxy = serverProxy;
    executorService.scheduleAtFixedRate(new BeatProcessor(), 0, clientBeatInterval, TimeUnit.MILLISECONDS);}
```

起了一个定时间隔为10s的任务，去执行BeatProcessor里面的逻辑，BeatProcessor的代码里面，是循环的去取当前客户端注册好的实例，然后向服务端发送一个http的心跳通知请求，告诉客户端，这个服务的健康状态，具体代码如下：

```Java
class BeatTask implements Runnable {
    BeatInfo beatInfo;
    public BeatTask(BeatInfo beatInfo) {
        this.beatInfo = beatInfo;
    }
    @Override
    public void run() {
        Map<String, String> params = new HashMap<String, String>(2);
        params.put("beat", JSON.toJSONString(beatInfo));
        params.put("dom", beatInfo.getDom());
        try {
            String result = serverProxy.callAllServers(UtilAndComs.NACOS_URL_BASE + "/api/clientBeat", params);
            JSONObject jsonObject = JSON.parseObject(result);

            if (jsonObject != null) {
                clientBeatInterval = jsonObject.getLong("clientBeatInterval");
            }
        } catch (Exception e) {
            LogUtils.LOG.error("CLIENT-BEAT", "failed to send beat: " + JSON.toJSONString(beatInfo), e);
        }
    }}
```

这里就是naocs的客户端主动上报服务健康状况的逻辑了，是服务发现功能，比较重要的一个概念，服务健康检查机制，常用的还有服务端主动去探测客户端的接口返回。

最后一步，就是初始化了一个叫HostReactor的实例，我们来看下，它干了些啥：

```Java
public HostReactor(EventDispatcher eventDispatcher, NamingProxy serverProxy, String cacheDir) {
    this.eventDispatcher = eventDispatcher;
    this.serverProxy = serverProxy;
    this.cacheDir = cacheDir;
    this.serviceInfoMap = new ConcurrentHashMap<>(DiskCache.read(this.cacheDir));
    this.failoverReactor = new FailoverReactor(this, cacheDir);
    this.pushRecver = new PushRecver(this);}
```

第五行，是从缓存文件中加载数据到serviceInfoMap的内存map中，接下来，初始化了一个FailoverReactor的组件，这个是[Nacos](https://github.com/alibaba/nacos)客户端缓存容灾相关的，它里面的初始化代码如下：

```Java
public void init() {
    executorService.scheduleWithFixedDelay(new SwitchRefresher(), 0L, 5000L, TimeUnit.MILLISECONDS);
    executorService.scheduleWithFixedDelay(new DiskFileWriter(), 30, DAY_PERIOD_MINUTES, TimeUnit.MINUTES);
    // backup file on startup if failover directory is empty.
    executorService.schedule(new Runnable() {
        @Override
        public void run() {
            try {
                File cacheDir = new File(failoverDir);

                if (!cacheDir.exists() && !cacheDir.mkdirs()) {
                    throw new IllegalStateException("failed to create cache dir: " + failoverDir);
                }

                File[] files = cacheDir.listFiles();
                if (files == null || files.length <= 0) {
                    new DiskFileWriter().run();
                }
            } catch (Throwable e) {
                LogUtils.LOG.error("NA", "failed to backup file on startup.", e);
            }

        }
    }, 10000L, TimeUnit.MILLISECONDS);}
```

初始化了3个定时任务，第一个任务的代码如下：

```Java
class SwitchRefresher implements Runnable {
    long lastModifiedMillis = 0L;
    @Override
    public void run() {
        try {
            File switchFile = new File(failoverDir + UtilAndComs.FAILOVER_SWITCH);
            if (!switchFile.exists()) {
                switchParams.put("failover-mode", "false");
                LogUtils.LOG.debug("failover switch is not found, " + switchFile.getName());
                return;
            }
            long modified = switchFile.lastModified();
            if (lastModifiedMillis < modified) {
                lastModifiedMillis = modified;
                String failover = ConcurrentDiskUtil.getFileContent(failoverDir + UtilAndComs.FAILOVER_SWITCH, Charset.defaultCharset().toString());
                if (!StringUtils.isEmpty(failover)) {
                    List<String> lines = Arrays.asList(failover.split(DiskCache.getLineSeperator()));
                    for (String line : lines) {
                        String line1 = line.trim();
                        if ("1".equals(line1)) {
                            switchParams.put("failover-mode", "true");
                            LogUtils.LOG.info("failover-mode is on");
                            new FailoverFileReader().run();
                        } else if ("0".equals(line1)) {
                            switchParams.put("failover-mode", "false");
                            LogUtils.LOG.info("failover-mode is off");
                        }
                    }
                } else {
                    switchParams.put("failover-mode", "false");
                }
            }
        } catch (Throwable e) {
            LogUtils.LOG.error("NA", "failed to read failover switch.", e);
        }
    }}
```

首先判定下容灾开关是否有，容灾开关是一个磁盘文件的形式存在，通过容灾开关文件名字，判定容灾开关是否打开，1表示打开，0为关闭，读取到容灾开关后，将值更新到内存中，后续解析地址列表时，首先会判定一下容灾开关是否打开，如果打开了，就读缓存的数据，否则从服务端获取最新数据。

第二个定时任务，做的事情如下：

```Java
class DiskFileWriter extends TimerTask {
    public void run() {
        Map<String, ServiceInfo> map = hostReactor.getServiceInfoMap();
        for (Map.Entry<String, ServiceInfo> entry : map.entrySet()) {
            ServiceInfo serviceInfo = entry.getValue();
            if (StringUtils.equals(serviceInfo.getKey(), UtilAndComs.ALL_IPS) || StringUtils.equals(serviceInfo.getName(), UtilAndComs.ENV_LIST_KEY)
                    || StringUtils.equals(serviceInfo.getName(), "00-00---000-ENV_CONFIGS-000---00-00")
                    || StringUtils.equals(serviceInfo.getName(), "vipclient.properties")
                    || StringUtils.equals(serviceInfo.getName(), "00-00---000-ALL_HOSTS-000---00-00")) {
                continue;
            }
            DiskCache.write(serviceInfo, failoverDir);
        }
    }}
```

每隔24小时，把内存中所有的服务数据，写一遍到磁盘中，其中需要过滤掉一些非域名数据的特殊数据，具体可看代码中的描述。最后一个定时任务，是每隔10s，是检查缓存目录是否存在，同时如果缓存里面值没有的话，主动触发一次缓存写磁盘的操作。

以上就是客户端构造一个[Nacos](https://github.com/alibaba/nacos)实例的初始化全部流程，大部分都是在初始化多个线程池或者定时任务，各司其职，这个也是我们写后端程序的一些基本套路，提高系统的并发能力，同时在对任务的分发和执行，引入一些常用的异步编程模型如队列模型的事件分发，这些都是异步和并发的很好学习素材，这2点也是写高性能程序的基本要求。

# 总结

这一章节，我们通过[Nacos](https://github.com/alibaba/nacos)的NacosFactory构造一个nacos服务实例作为切入点，把客户端的初始化流程给串了一遍，概述下客户端初始化流程做的几件事：

- 初始化事件分发组件，用于处理服务端主动通知下来的变更数据
- 初始化[Nacos](https://github.com/alibaba/nacos)服务集群地址列表更新组件，用于客户端维护[Nacos](https://github.com/alibaba/nacos)服务端的最新地址列表
- 初始化服务健康检查模块，主动给服务端上报服务的健康情况
- 初始化客户端的缓存，10s检查一次，如果没有，则创建
- 24小时备份一次客户端的缓存文件
- 5s检查一次容灾开关，更新到内存中，容灾模式情况下，服务地址读的都是缓存

以上就是[Nacos](https://github.com/alibaba/nacos)客户端实例初始化的整体流程，是不是感觉做的事情挺多的，还有一些代码的细节点，大家自己多精读一下，如果有什么不明白的，可以留言，或者在社区找@超哥帮你解答，如果能发现bug或者其他建议，可以在社区提issue。