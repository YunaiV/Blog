title: Spring Cloud Ribbon 源码分析
date: 2018-01-09
tag: 
categories: Ribbon
permalink: Ribbon/saleson/spring-cloud-ribbon
author: saleson
from_url: https://blog.csdn.net/mr_rain/article/details/80067092
wechat_url: 

-------

摘要: 原创出处 https://blog.csdn.net/mr_rain/article/details/80067092 「saleson」欢迎转载，保留摘要，谢谢！


-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

Ribbon是Netflix公司开源的一个负载均衡的项目，它属于上述的第二种，是一个客户端负载均衡器，运行在客户端上。我们先来看三张序列图，分别是RestTemplate、Feign、Zuul在使用Ribbon的调用链。
![rest template used ribbon](http://static.iocoder.cn/csdn/20180424162000283?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L01yX3JhaW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

![feign used ribbon](http://static.iocoder.cn/csdn/20180424162013301?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L01yX3JhaW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

![zuul used ribbon](http://static.iocoder.cn/csdn/20180424162104431?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L01yX3JhaW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

从上面三张图中可以看出， 最终都是调用ILoadBalancer#chooseServer()方法筛选服务实例，并且在spring cloud中， 默认使用的是ZoneAwareLoadBalancer。

#### ILoadBalancer

咱们首先来看看ILoadBalancer接口，在这个接口中定义了客户端负载均衡需要使用到的一系列抽象操作

```java
public interface ILoadBalancer {
    public void addServers(List<Server> newServers);
    public Server chooseServer(Object key);
    public void markServerDown(Server server);
    public List<Server> getReachableServers();
    public List<Server> getAllServers();
}
```

- addServers(List)
  向负载均衡器中维护的实例列表增加服务实例
- chooseServer(Object)
  通过某种策略，从负载均衡器中挑选出一个具体的服务实例
- markServerDown(Server)
  用来通知和标识负载均衡器中某个具体实例已经停止服务，不然负载均衡器在下一次获取服务实例清单前都会认为服务实例均是正常服务的
- getReachableServers()
  返回当前可正常服务的实例列表
- getAllServers()
  返回所有已知的服务实例列表，包括正常服务和停止服务的实例
  在该接口中涉及的Server对象定义是一个传统的服务端节点，在其中存储了服务端节点的一些基础信息，如host\port等。

对于该接口的实现类ZoneAwareLoadBalancer，整理出如下图所示的结构
![ZoneAwareLoadBalancer Diagrams](http://static.iocoder.cn/csdn/201804241627573?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L01yX3JhaW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

可以看出，BaseLoadBalancer类实现了基础的负载均衡，而DynamicServerListLoadBalancer和ZoneAwareLoadBalancer在其基础上做了一些扩展。

#### AbstractLoadBalancer

AbstractLoadBalancer是ILoadBalancer接口的抽象实现。

```java
public abstract class AbstractLoadBalancer implements ILoadBalancer {
    public enum ServerGroup{
        ALL,
        STATUS_UP,
        STATUS_NOT_UP
    }
    public Server chooseServer() {
        return chooseServer(null);
    }
    public abstract List<Server> getServerList(ServerGroup serverGroup);
    public abstract LoadBalancerStats getLoadBalancerStats();
}
```

在该抽象类中定义了一个关于服务实例的分组枚举类ServerGroup,它包含三种不同的类型。
ALL: 所有服务实例
STATUS_UP: 正服务的实例
STATUS_NOT_UP: 停止服务的实例

还实现了一个chooseServer()方法，该方法调用接口中的chooseServer(Object key)实现，其中参数key为null，表示在选择服务实例时忽略key的条件判断。
然后定义了两个抽象方法

- getServerList(ServerGroup):
  根据分组类型来返回不同的服务实例列表。
- getLoadBalancerStats():
  返回LoadBalancerStatus对象。LoadBalancerStats对象是用来存储负载均衡器中各个服务实例当前的属性和统计信息的，我们可以根据这些信息来观察负载均衡器的运行情况，同是它们也是用来制定负载均衡策略的重要依据，比如在AvailabilityPredicate和ZoneAvoiadancePredicate都有用到。

#### BaseLoadBalancer

BaseLoadBalancer是Ribbon负载均衡器的基础实现类，在该类中定义了很多关于负载均衡器相关的基础内容。

- 定义并维护了两个服务实例Server对象的列表
  一个是存储所有的服务实例清单
  一个是存储正常服务的实例清单

- 定义了存储负载系统器各服务实例属性和统计信息的LoadBalanceStats对象

- 实现了ILoadBalancer接口定义的负载均衡器中应具备的一系列基础操作方法
  addServers(List newServers)
  chooseServer(Object key)
  markServerDown(Server server)
  getReachableServers()
  getAllServers()

- 定义了检查服务实例是否正常服务的IPing对象
  IPing是用来向server发起”ping”，来判断该server是否有响应，从而判断该server是否可用。它有一个boolean isAlive(Server)方法，IPing的实现类有以下几种
  **PingUrl**
  真实的去ping 某个url，判断其是否alive

  **PingConstant**
  固定返回某服务是否可用，默认返回true，即可用

  **NoOpPing**
  不去ping,直接返回true,即可用

  **DummyPing**
  直接返回true

  **NIWSDiscoveryPing**
  根据DiscoveryEnabledServer的InstanceInfo的InstanceStatus去判断，如果为InstanceStatus.UP，则为可用，否则不可用

- 定义了检查服务实例操作的执行策略对象IPingStrategy
  ping的执行策略，在BaseLoadBalancer中默认是线型轮询，可以自己去扩展为并行。

- 定义了负载均衡的处理规则IRule对象
  查看该类的chooseServer(Object key)方法，我们可以知道，负载均衡器实际将服务实例选择的任务逻辑委托给了IRule实例的choose()方法来实现，并默认初始化RoundRobinRule(线性轮询)为IRule的实现对象。IRule的实现类有
  **BestAvailableRule**
  选择最小请求数的Server

  **RandomRule**
  随机选择

  **RoundRobinRule**
  线性轮询

  **RetryRule**
  根据轮询的方式重试

  **WeightedResponseTimeRule**
  根据响应时间去分配Weight，Weight越高，被选择的可能性就越大

  ** ZoneAvoidanceRule\**
  根据Server的zone区域和可用性来轮询选择

- 启动ping任务
  在BaseLoadBalancer中构造方法中会启动一个定时检查Server是否正常服务的任务，默认执行间隔是10秒。也可以通过IClientConfig设置执行间隔时间。

  ```java
  DefaultClientConfigImpl config = new DefaultClientConfigImpl();
  …
  config.setProperty(IClientConfigKey.Keys.NFLoadBalancerPingInterval, 15); // 注意单位s
  …
  ILoadBalancer lb = new BaseLoadBalancer(config);
  ```

#### DynamicServerListLoadBalancer

DynamicServerListLoadBalancer继承于BaseLoadBalancer，它对基础负载均衡器做了扩展。在该负载均衡器，实现了服务实例清单在运行期间的动态更新；同时还具备了对服务实例清单的过滤功能，也就是说可以通过过滤器来选择获取一批服务实例清单。

- ServerList
  它是DynamicServerListLoadBalancer新增的服务列表操作对象，它定义了两个方法
  **getInitialListOfServers()**
  用于获取初始化的服务实例清单

  **getUpdatedListOfServers()**
  用于获取更新的服务实例清单

  在DynamicServerListLoadBalancer中使用的是DomainExtractingServerList，该ServerList在内部还定义了一个ServerList list，DomainExtractingServerList对上述两个方法的具体实现是委托给了内部定义的ServerList对象。
  而这个内部定义的ServerList对象实际上是DiscoveryEnabledNIWSServerList，实现逻辑主要依靠EurekaClient从注册中心获取服务实例(instanceInfo)列表，然后遍历这些服务实例，将状态为UP(正常服务)的实例转换成DiscoverEnabledServer对象，最后组织成列表返回。
  返回List到了DomainExtractingServerList类，将DiscoveryEnabledNIWSServerList返回的DiscoverEnabledServer对象，转换成其子类对象DomainExtractingServer，在该类的构造方法中初始化了一些属性，比如id、zone、isAliveFlag、readyToServer等。

- ServerListUpdater
  这个接口实现的是对ServerList的更新，可以称之为”服务更新器”。它定义了一系列控制它的方法

  ```java
  public interface ServerListUpdater {
      //内部接口
      public interface UpdateAction {
      //实现对ServerList的更新操作
          void doUpdate();
      }
      //启动服务更新器
      void start(UpdateAction updateAction);
      //停止服务更新器
      void stop();
      //返回最近的更新时间戳
      String getLastUpdate();
      //返回上一次更新到现在的时间间隔，单位为ms
      long getDurationSinceLastUpdateMs();
      //返回错过的更新周期数
      int getNumberMissedCycles();
      //返回核心线程数
      int getCoreThreads();
  }
  ```

  它的实现类有两个
  **PollingServerListUpdater**
  DynamicServerListLoadBalancer默认的策略，它是通过定时任务的方式进行服务列表的更新

  **EurekaNotificationServerListUpdater**
  它需要利用Eureka的事件监听器来驱动服务列表的更新操作

- ServerListFilter
  该接口很简单，只有一个方法List getFilteredListOfServers(List)，它主要用于实现服务实例列表的过滤，通过传入服务实例清单，根据一些规则返回过滤后的服务实例清单。
  ![ServerListFilter Hierarhy](http://static.iocoder.cn/csdn/20180424165148983?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L01yX3JhaW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

  **ZoneAffinityServerListFilter**
  该过滤器基于”区域感知”(Zone Affinity)的方式实现服务实例的过滤，它会根据提供服务的实例所在的区域(Zone)与消费者自身所在的区域(Zone)进行比较，过滤掉那些不是在同一区域的实例。
  需要注意的是，该过滤器还会通过shouldEnableZoneAffinity(List)方法来判断是否要启用”区域感知”的功能，它使用了LoadBalancerStats的getZoneSnapshot方法来获取这些过滤后的同区域实例的基础指标(包含实例数、断路器断开数、清动请求数、实例平均负载等)，根据一系列的算法得出下面同筱评价值并与阈值进行对比，若有一个条件符合，就不启用”区域感知”过滤后的服务实例清单。目的是集群出现区域故障时，依然可以依靠其它区域的正常实例提供服务，保障高可用。
  1､故障实例百分比(断路器断开数/实例数)>=0.8
  2､实例平均负载>=0.6
  3､可用实例数(实例数-数路器断开数)<2

  **DefaultNIWSServerListFilter**
  该过滤器完全继承ZoneAffinityServerListFilter

  **ServerListSubsetFilter**
  该过滤器适用于大规模服务器集群(上百或更多)的系统，因为它可以产生一个”区域感知”结果的子集列表，同时它还能够通过比较服务实例的通信失败数和并发连接数来判定该服务是否健康来选择性的从服务实例列表中剔除那些相对不够健康的实例

  **ZonePreferenceServerListFilter**
  使用spring cloud整合eureka和ribbon时默认使用的该过滤器。它实现了通过配置或eureka实例无数据的所属区域(Zone)来过滤出同区域的服务实例。
  首先通过父类获取到通过”区域感知”过滤后的服务实例列表，然后遍历这个列表，取出根据消费者配置预设的zone来进行过滤，如果过滤后的结果为空，就直接返回父类返回的结果，否则返回过滤后的结果。
  消费者预设的zone可以通过eureka.instance.metadataMap.zone进行配置，初始化的代码在org.springframework.cloud.netflix.ribbon.eureka.EurekaRibbonClientConfiguration#preprocess()方法中， 如果eureka.instance.metadataMap.zone没有设置， 会读取默认的region和zone。

#### ZoneAwareLoadBalancer

ZoneAwareLoadBalancer是对DynamicServerListLoadBalancer的扩展。在DynamicServerListLoadBalancer中没有区分zone区域，而ZoneAwareLoadBalancer主要扩展的功能就是增加了zone区域过滤。
1､重写setServerListForZones(Map)方法，按zone区域创建BaseLoadBalancer，如果ZoneAwareLoadBalancer#IRule为空，默认使用AvailabilityFilteringRule，否则就使用ZoneAwareLoadBalancer#IRule。
2､重写chooseServer(Object)方法，当负载均衡器中维护的实例zone区域个数大于1时，会执行以下策略，否则执行父类的方法。
3､根据LoadBalancerStats创建zone区域快照用于后续的算法中。
4､根据zone区域快照中的统计数据来实现可用区的挑选。
5､当返回的可用zone区域集合不空，并且个数小于zone区域总数，就随机选择一个zone区域。
6､在确定了zone区域后，获取对应zone区域的负载均衡器，并调用chooseServer(Object)方法来选择具体的服务实例。

# 666. 彩蛋

如果你对 Ribbon   感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)