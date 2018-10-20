title: Netflix Ribbon 介绍
date: 2018-01-08
tag: 
categories: Ribbon
permalink: Ribbon/yiguoqiang/NetflixRibbon
author: 易国强
from_url: http://tech.lede.com/2018/01/12/rd/server/NetflixRibbon/
wechat_url: 

-------

摘要: 原创出处 http://tech.lede.com/2018/01/12/rd/server/NetflixRibbon/ 「易国强」欢迎转载，保留摘要，谢谢！


-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------


在Spring Cloud微服务架构中，我们一般都会用到客户端的负载均衡，ribbon就是其中的主要实现，netflix开源的ribbon主要提供云端的负载均衡，有多种负载均衡策略可以自行选择，一般我们会配合服务发现eureka和断路器hystrix使用，下面主要就ribbon的源码做简要的介绍，同时提供了基于服务发现的简单使用示例。



### 负载均衡的组成

- 一个典型的负载均衡包含三个部分：
  - 基于特定标准的服务器(节点)列表
  - 定义并实现负载均衡的策略的类
  - 定义并实现一种机制来确定列表中节点/服务器的适宜性/可用性的类

### netflix-ribbon

- 最新的ribbon版本maven引入如下所示：

  ```XML
  <dependency>
  	<groupId>com.netflix.ribbon</groupId>
  	<artifactId>ribbon</artifactId>
  	<version>2.2.2</version>
  </dependency>
  ```

- 组件依赖关系示意图：

[![image](https://github.com/siyuyifang/image/blob/master/ribbon/netflix-ribbon.png?raw=true)](https://github.com/siyuyifang/image/blob/master/ribbon/netflix-ribbon.png?raw=true)

- 由上图可以看见，ribbon包主要依赖了ribbon-transport 和 hystrix-core两个组件。

[![image](https://github.com/siyuyifang/image/blob/master/ribbon/1-4.png?raw=true)](https://github.com/siyuyifang/image/blob/master/ribbon/1-4.png?raw=true)

- ribbon中实现的关键组件类主要包括如下：
  [![image](https://github.com/siyuyifang/image/blob/master/ribbon/Netflix-Ribbon-galance.png?raw=true)](https://github.com/siyuyifang/image/blob/master/ribbon/Netflix-Ribbon-galance.png?raw=true)

### ribbon-core

- IclientConfig

  - ribbon-core
  - 客户端的参数配置默认实现类： DefaultClientConfigImpl.java

- ribbon参数配置格式

  ```yaml
  - <clientName>.<nameSpace>.<propertyName>=<value>
  - <clientName>：这是调用ribbon的客户端名称（一般可理解为服务提供者的spring.application.name值），如果此值为没有配置，则此条属性会作用到所有的客户端。
  - <nameSpace>：默认值为 “ribbon”
  - <propertyName>：所有的可用的属性都在com.netflix.client.conf.CommonClientConfigKey类中定义
  - 若属性没有配置，则会使用上述提到的DefaultClientConfigImpl类中的默认配置。
  ```

- 主要参数配置格式

```yaml
# 负载均衡类，默认为om.netflix.loadbalancer.ZoneAwareLoadBalance
<clientName>.<nameSpace>.NFLoadBalancerClassName=xx
# 负载均衡规则类，默认为com.netflix.loadbalancer.AvailabilityFilteringRule
<clientName>.<nameSpace>.NFLoadBalancerRuleClassName=xx
# 心跳检测类，默认为com.netflix.loadbalancer.DummyPing
<clientName>.<nameSpace>.NFLoadBalancerPingClassName=xx
# 服务列表类，默认为com.netflix.loadbalancer.ConfigurationBasedServerList,结合eureka使用时默认为com.netflix.niws.loadbalancer.DiscoveryEnabledNIWSServerList
<clientName>.<nameSpace>.NIWSServerListClassName=xx
# 服务过滤类，默认为com.netflix.loadbalancer.ZoneAffinityServerListFilter。结合spring cloud eureka使用时默认为org.springframework.cloud.netflix.ribbon.ZonePreferenceServerListFilter
<clientName>.<nameSpace>.NIWSServerListFilterClassName=xx
```

- 默认参数示例
  - 关闭eureka的服务发现功能
    - ribbon服务提供端的名称.ribbon.eureka.enabled=false
  - 指定ribbon客户端的服务列表
    - ribbon服务提供端的名称.ribbon.listOfServers=ip1:port1,ip2:port2
  - 开启区域关联性检查
    - ribbon服务提供端的名称.ribbon.EnableZoneAffinity=true
  - ……

### ribbon-loadbalancer

- 包内类依赖关系示意图：

  ​

  ![image](https://github.com/siyuyifang/image/blob/master/ribbon/ribbon-loadbalancer.png?raw=true)

  #### 负载均衡器

- 依赖关示意图：
  [![image](https://github.com/siyuyifang/image/blob/master/ribbon/1-1.png?raw=true)](https://github.com/siyuyifang/image/blob/master/ribbon/1-1.png?raw=true)

- NoOpLoadBalancer.java

  - 啥都不做

- BaseLoadBalancer.java

  - 负载均衡器的基本实现，可以将任意服务器列表设置为服务器池。可以设置ping来确定服务器的活跃程度。在内部，这个类维护一个“所有”服务器列表和一个“up”服务器列表，并根据调用者的要求使用它们
  - TimerTask定时任务进行ping
  - SerialPingStrategy
    - 如果IPing接口实现效率较慢，或者有大量服务器，则表现不是很好。 逐个进行ping，耗时较长，改善方式可以放入线程池中执行，但这个策略设计之初是运行在内存中eureka之间的信息调用
  - 默认规则：最著名和最基本的负载平衡策略 Round Robin Rule

- DynamicServerListLoadBalancer.java

  - 通过动态源获取服务器的候选列表的负载平衡器
  - 可以通过筛选标准来传递服务器列表，以过滤不符合所需条件的服务器

- ZoneAwareLoadBalancer.java

  - 统计每个zone的平均请求的情况，保证从所有zone选取对当前客户端服务最好的服务组列表
  - 这是一个区域内的未完成请求的总数除以可用的目标实例数量(不包括断路器跳闸实例)。当超时在一个坏的区域中缓慢发生时，这个度量是非常有效的
  - 负载均衡器将计算和检查所有可用区域的区域数据。如果任何区域的平均主动请求达到了配置的阈值，则该区域将从活动服务器列表中删除。如果超过一个区域已经达到了阈值，那么每个服务器上最活跃请求的区域将被删除。一旦最坏的区域被放弃，一个区域将被选择在剩下的区域中，与它的实例数量成正比。对于每个请求，将从所选择的区域返回一个服务器(规则是负载均衡策略，例如{可用性过滤规则})，上面的步骤将被重复。也就是说，每个区域的相关负载均衡决策都是在实时统计数据的帮助下做出的
  - 默认规则: AvailabilityFilteringRule

#### 服务列表过滤器

- 依赖关示意图：
  [![image](https://github.com/siyuyifang/image/blob/master/ribbon/1-2.png?raw=true)](https://github.com/siyuyifang/image/blob/master/ribbon/1-2.png?raw=true)

- ServerListFilter

  - 接口
  - 过滤已配置的或动态获取的候选服务器列表

- AbstractServerListFilter

  - LoadBalancerStats
  - 负责从负载均衡器中过滤出当前可用的服务器列表的类

- ZoneAffinityServerListFilter

  - EnableZoneAffinity 或 EnableZoneExclusivity 开启状态使用，默认关闭
  - 处理基于区域感知的过滤服务器
  - 过滤掉不和客户端在相同zone的服务，若不存在相同zone，则不进行过滤。

- ServerListSubsetFilter

  - 服务器列表过滤器，它限制负载均衡器使用的服务器的数量为所有服务器的子集。如果服务器场很大，这很有用。在成百上千的情况下，使用它们中的每一个，并保持http客户端连接池中的连接是不必要的。它还可以通过比较总的网络故障和并发连接来驱逐相对不健康的服务器。

  - 若启用该过滤器，配置如下:(配置eureka使用的场景)

    ```yaml
    <clientName>.ribbon.NIWSServerListClassName=com.netflix.niws.loadbalancer.DiscoveryEnabledNIWSServerList
    # the server must register itself with Eureka server with VipAddress "myservice"
    <clientName>.ribbon.DeploymentContextBasedVipAddresses=myservice
    <clientName>.ribbon.NIWSServerListFilterClassName=com.netflix.loadbalancer.ServerListSubsetFilter
    # only show client 5 servers. default is 20.
    <clientName>.ribbon.ServerListSubsetFilter.size=5
    ```

#### 负载均衡规则

- 依赖关示意图：
  [![image](https://github.com/siyuyifang/image/blob/master/ribbon/1-3.png?raw=true)](https://github.com/siyuyifang/image/blob/master/ribbon/1-3.png?raw=true)
- IRule
  - choose(..)
  - setLoadBalancer(..)
  - getLoadBalancer()
- ClientConfigEnabledRoundRobinRule
- BestAvailableRule
  - 一般结合ServerListSubsetFilter使用。
  - 过滤掉多次访问失败处于断路状态的服务。
  - 确保只需要在少量服务器中找到最小的并发请求。此外，每个客户机将获得一个随机的服务器列表，该列表避免了一个服务器的并发请求最少的服务器被大量客户机选择，并立即被淹没。
- PredicateBasedRule （抽象类）
  - 代理AbstractServerPredicate类的实例
  - chooseRoundRobinAfterFiltering 后获取Server实例
- ZoneAvoidanceRule
  - 继承PredicateBasedRule类
  - 基于区域（zone）和服务有效性原则，使用CompositePredicate类来过滤服务列表
  - CompositePredicate = ZoneAvoidancePredicate + AvailabilityPredicate
  - FallbackPredicate –> AvailabilityPredicate
  - 使用ZoneAvoidancePredicate和AvailabilityPredicate来判断是否选择某个server，前一个，以一个区域为单位考察可用性，对于不可用的区域整个丢弃，从剩下区域中选可用的server。判断出最差的区域，排除掉最差区域。在剩下的区域中，将按照服务器实例数的概率抽样法选择，从而判断判定一个zone的运行性能是否可用，剔除不可用的zone（的所有server），AvailabilityPredicate用于过滤掉连接数过多的Server
- AvailabilityFilteringRule
  - 继承PredicateBasedRule类
    - 基于服务有效性原则，使用AvailabilityPredicate类来过滤服务列表
    - 过滤掉多次访问失败而处于断路状态的服务实例
    - 过滤掉并发的连接数量超过阈值的服务实例
    - 过滤完成按照RoundRobinRule策略访问
- RandomRule
  - 在现有服务器之间随机分配流量的一种负载平衡策略
- RoundRobinRule
  - 轮询，默认规则
  - 更高level规则的回退策略
- WeightedResponseTimeRule
  - 基于响应时间的权重计算方法： weight = total responseTime - responseTime
  - 响应时间越快，服务权重越大
  - 当服务器没有足够的统计信息时（如刚启动时）使用轮询策略
- RetryRule
  - 重试机制
  - 默认重试毫秒数 500
  - 考虑到定义的规则可以串行多个，这个重试规则允许在现有规则中添加重试逻辑
  - 先按照轮询策略获取服务，获取失败，则在设定时间内重试，获取可用服务

#### 心跳检测

- 依赖关系示意如下：
  [![image](https://github.com/siyuyifang/image/blob/master/ribbon/1-5.png?raw=true)](https://github.com/siyuyifang/image/blob/master/ribbon/1-5.png?raw=true)
- IPing
  - isAlive()
- DummyPing
  - 继承抽象类AbstractLoadBalancerPing
  - 默认的简单实现，认为所以服务都是存活状态
- NoOpPing
  - 啥都不干
- PingConstant
  - 可以通过方法设置指定服务器存活状态
- NIWSDiscoveryPing
  - ribbon-eureka包中提供的类，结合eureka使用时，如果Discovery Client在线，则认为心跳检测通过。
- PingUrl
  - ribbon-httpclient包中提供的类
  - 采用此方式，会使用httpclient调用服务的一个url，如果调用成功则认为本次心跳检测通过，即服务存活可用。

#### ribbon-transport

- 这个模块中提供的功能实际上是没有用于生产环境中使用的
- 支持http、tcp、udp协议
- 采用rxnetty实现
- Reactive 响应式(反应式)编程 是一种新的编程风格，其特点是异步或并发、事件驱动、推送PUSH机制以及观察者模式的衍生。
- 异步： 一个实现异步操作的库，扩展的观察者模式来实现的
- 简洁：随着程序逻辑变得越来越复杂，它依然能够保持简洁

#### ribbon-eureka

- 当Ribbon与Eureka联合使用时，ribbonServerList会被DiscoveryEnabledNIWSServerList重写，扩展成从Eureka注册中心中获取服务端列表。同时它也会用NIWSDiscoveryPing来取代IPing，它将职责委托给Eureka来确定服务端是否已经启动。

### spring-cloud-starter-ribbon

- spring-cloud-starter-ribbon 主要是对spring-cloud-starter-netflix-ribbon包的封装

- spring-cloud-starter-netflix-ribbon中主要包含的组件有：ribbon、ribbon-httpclient、spring-cloud-strater、spring-cloud-starter-netflix-archaius、spring-cloud-netflix-core

- 使用示例

  - 配合eureka的服务发现与注册功能使用。
  - 配置一个eureka-server实例
  - 配置两个ribbon服务提供端实例
  - 配置一个ribbon服务消费端实例

- 主要配置信息如下：

  - eureka-server

    - 在入口类开启服务注册功能 @EnableEurekaServer
    - 在application.properties文件中配置端口号应用名称等信息

    ```
    spring.application.name=eureka-server
    server.port=1111
    eureka.server.enable-self-preservation=false
    eureka.instance.prefer-ip-address=true
    ```

    - pom文件中主要需要加入eureka-server的依赖

    ```
    <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka-server</artifactId>
    </dependency>
    ```

  - ribbon-provider

    - 在入口类开启服务发现功能 @EnableDiscoveryClient
    - 在application.properties文件中配置端口号应用名称等信息

    ```
    spring.application.name=ribbon-provider
    server.port=2222
    eureka.client.service-url.defaultZone=http://localhost:1111/eureka/
    ```

    - pom文件中主要需要加入eureka-client的依赖

    ```
       <dependency>
    		<groupId>org.springframework.cloud</groupId>
    		<artifactId>spring-cloud-starter-eureka</artifactId>
    </dependency>
    ```

    - 提供一个服务接口，示例如下：

    ```
    @RestController
    public class DemoController {

        @GetMapping("/hello")
        public String hello(String name){
            System.out.println("invoke my service");
            return "hello, " + name;
        }
    }
    ```

    - 启动两个服务提供者实例，端口为2222，2223

  - ribbon-consumer

    - 服务消费者的依赖和配置基本和ribbon-provider一致，因为他们对于eureka-server来说都是client的身份，注意在配置文件中端口号和名称的区别。

    - 在application.properties文件中配置端口号应用名称等信息

      ```
      spring.application.name=ribbon-consumer
      server.port=3333
      eureka.client.service-url.defaultZone=http://localhost:1111/eureka/
      ```

    - 我们主要在此使用ribbon的负载均衡功能，在pom文件中主要引入的依赖如下：

      ```
      <dependency>
          <groupId>org.springframework.cloud</groupId>
          <artifactId>spring-cloud-starter-eureka</artifactId>
      </dependency>
      <dependency>
          <groupId>org.springframework.cloud</groupId>
          <artifactId>spring-cloud-starter-ribbon</artifactId>
      </dependency>
      ```

    - ribbon消费者使用的接口示例如下，消费ribbon-provider中提供的接口。

      ```java
      /**
       * 开启客户端负载均衡
       */
      @LoadBalanced
      @Bean
      RestTemplate restTemplate() {
          return new RestTemplate();
      }

      @Autowired
      RestTemplate restTemplate;

      @GetMapping("/test")
      public String test() {
          System.out.println("enter test");
          String username = "zhangsan";
          String result =
          //ribbon-provider为eureka中注册节点名称
          restTemplate.getForEntity("http://ribbon-provider/hello?name=" + username, String.class).getBody();
          return result;
      }
      ```

  - 启动上述配置的各环境实例，访问服务消费端的接口，我们会发现在两个服务提供者的实例控制台轮流会输出“invoke my service”的信息。因为ribbon默认的负载均衡策略为轮询（Round Robin）机制。所以会交替访问两个实例。

### spring-cloud-starter-feign

- 主要集成feign、ribbon、hystrix组件
- 比spring-cloud-starter-ribbon 多包含feign-hystrix

### spring-cloud-starter-eureka

- 主要是对spring-cloud-starter-netflix-eureka-client的包装
- ribbon + eureka + starter web 系列组件包的集合

### 参考资料

- <https://github.com/Netflix/ribbon>
- <http://blog.csdn.net/hry2015/article/details/78357990>
- <http://blog.csdn.net/zzzzzz55300411/article/details/54352534>
- <http://blog.didispace.com/springcloud-sourcecode-ribbon/>

# 666. 彩蛋

如果你对 Ribbon   感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)