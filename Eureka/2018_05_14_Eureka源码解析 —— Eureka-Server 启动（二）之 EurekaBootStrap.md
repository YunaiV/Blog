title: Eureka 源码解析 —— Eureka-Server 启动（二）之 EurekaBootStrap
date: 2018-05-14
tags:
categories: Eureka
permalink: Eureka/eureka-server-init-second

---

摘要: 原创出处 http://www.iocoder.cn/Eureka/eureka-server-init-second/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Eureka 1.8.X 版本** 

- [1. 概述](http://www.iocoder.cn/Eureka/eureka-server-init-second/)
- [2. EurekaBootStrap](http://www.iocoder.cn/Eureka/eureka-server-init-second/)
  - [2.1 初始化 Eureka-Server 配置环境](http://www.iocoder.cn/Eureka/eureka-server-init-second/)
  - [2.2 初始化 Eureka-Server 上下文](http://www.iocoder.cn/Eureka/eureka-server-init-second/)
- [3. Filter](http://www.iocoder.cn/Eureka/eureka-server-init-second/)
  - [3.1 StatusFilter](http://www.iocoder.cn/Eureka/eureka-server-init-second/)
  - [3.2 ServerRequestAuthFilter](http://www.iocoder.cn/Eureka/eureka-server-init-second/)
  - [3.3 RateLimitingFilter](http://www.iocoder.cn/Eureka/eureka-server-init-second/)
  - [3.4 GzipEncodingEnforcingFilter](http://www.iocoder.cn/Eureka/eureka-server-init-second/)
  - [3.5 ServletContainer](http://www.iocoder.cn/Eureka/eureka-server-init-second/)
- [666. 彩蛋](http://www.iocoder.cn/Eureka/eureka-server-init-second/)

---

![](http://www.iocoder.cn/images/common/wechat_mp_2018_05_18.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

---

# 1. 概述

本文接[《Eureka 源码解析 —— Eureka-Server 启动（一）之 EurekaServerConfig》](http://www.iocoder.cn/Eureka/eureka-server-init-first/?self)，主要分享 **Eureka-Server 启动的过程**的第二部分 —— **EurekaBootStrap**。

考虑到整个初始化的过程中涉及的代码特别多，拆分成两两篇文章：

* [ServerConfig](http://www.iocoder.cn/Eureka/eureka-server-init-first/?self)
* 【本文】EurekaBootStrap

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



# 2. EurekaBootStrap

`com.netflix.eureka.EurekaBootStrap`，Eureka-Server **启动入口**。

![](http://www.iocoder.cn/images/Eureka/2018_05_14/01.png)

EurekaBootStrap 实现了 `javax.servlet.ServletContextListener` **接口**，在 Servlet 容器( 例如 Tomcat、Jetty )启动时，调用 `#contextInitialized()` 方法，初始化 Eureka-Server，实现代码如下：

```Java
public class EurekaBootStrap implements ServletContextListener {

    // 省略无关变量和方法

    @Override
    public void contextInitialized(ServletContextEvent event) {
        try {
            // 初始化 Eureka-Server 配置环境
            initEurekaEnvironment();

            // 初始化 Eureka-Server 上下文
            initEurekaServerContext();

            ServletContext sc = event.getServletContext();
            sc.setAttribute(EurekaServerContext.class.getName(), serverContext);
        } catch (Throwable e) {
            logger.error("Cannot bootstrap eureka server :", e);
            throw new RuntimeException("Cannot bootstrap eureka server :", e);
        }
    }

}
```

* 调用 `#initEurekaEnvironment()` 方法，初始化 Eureka-Server **配置**环境。
* 调用 `#initEurekaServerContext()` 方法，初始化 Eureka-Server 上下文。

## 2.1 初始化 Eureka-Server 配置环境

```Java
// EurekaBootStrap.java

/**
* 部署环境 - 测服
*/
private static final String TEST = "test";

private static final String ARCHAIUS_DEPLOYMENT_ENVIRONMENT = "archaius.deployment.environment";

private static final String EUREKA_ENVIRONMENT = "eureka.environment";

/**
* 部署数据中心 - CLOUD
*/
private static final String CLOUD = "cloud";
/**
* 部署数据中心 - 默认
*/
private static final String DEFAULT = "default";

private static final String ARCHAIUS_DEPLOYMENT_DATACENTER = "archaius.deployment.datacenter";

private static final String EUREKA_DATACENTER = "eureka.datacenter";

protected void initEurekaEnvironment() throws Exception {
    logger.info("Setting the eureka configuration..");

   // 设置配置文件的数据中心
   String dataCenter = ConfigurationManager.getConfigInstance().getString(EUREKA_DATACENTER);
   if (dataCenter == null) {
       logger.info("Eureka data center value eureka.datacenter is not set, defaulting to default");
       ConfigurationManager.getConfigInstance().setProperty(ARCHAIUS_DEPLOYMENT_DATACENTER, DEFAULT);
   } else {
       ConfigurationManager.getConfigInstance().setProperty(ARCHAIUS_DEPLOYMENT_DATACENTER, dataCenter);
   }

   // 设置配置文件的环境
   String environment = ConfigurationManager.getConfigInstance().getString(EUREKA_ENVIRONMENT);
   if (environment == null) {
       ConfigurationManager.getConfigInstance().setProperty(ARCHAIUS_DEPLOYMENT_ENVIRONMENT, TEST);
       logger.info("Eureka environment value eureka.environment is not set, defaulting to test");
   }
}
```

* 设置基于 [Netflix Archaius](https://github.com/Netflix/archaius) 实现的配置文件的上下文，从而读取**合适**的配置文件。大多数情况下，只需要设置 `EUREKA_ENVIRONMENT` 即可，不同的服务器环境( 例如，`PROD` / `TEST` 等) 读取不同的配置文件。实现原理，在[《Eureka 源码解析 —— Eureka-Client 初始化（一）之 EurekaInstanceConfig》「2.4 PropertiesInstanceConfig」](http://www.iocoder.cn/Eureka/eureka-client-init-first/?self)有详细解析。
* 感兴趣的也可以阅读：[《Netflix Archaius 官方文档 —— Deployment context》](https://github.com/Netflix/archaius/wiki/Deployment-context)。

## 2.2 初始化 Eureka-Server 上下文

EurekaBootStrap 的 `#initEurekaServerContext()` 方法的实现代码相对较多，已经将代码**切块** + **中文注册**，点击 [EurekaBootStrap](https://github.com/YunaiV/eureka/blob/a1c6074fd038f1182132a43b1ebc4cc08166f0be/eureka-core/src/main/java/com/netflix/eureka/EurekaBootStrap.java#L137) 链接，对照下面每个小结阅读理解。


### 2.2.1 创建 Eureka-Server 配置

```Java
EurekaServerConfig eurekaServerConfig = new DefaultEurekaServerConfig();
```

* 在 [《Eureka 源码解析 —— Eureka-Server 启动（一）之 ServerConfig》「2.3 DefaultEurekaServerConfig」](http://www.iocoder.cn/Eureka/eureka-server-init-first/?self) 有详细解析。

### 2.2.2 Eureka-Server 请求和响应的数据兼容

```Java
// For backward compatibility
JsonXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(), XStream.PRIORITY_VERY_HIGH);
XmlXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(), XStream.PRIORITY_VERY_HIGH);
```

* 目前 Eureka-Server 提供 V2 版本 API ，如上代码主要对 V1 版本 API 做兼容。可以选择跳过。

### 2.2.3 创建 Eureka-Server 请求和响应编解码器

```Java
logger.info("Initializing the eureka client...");
logger.info(eurekaServerConfig.getJsonCodecName());
ServerCodecs serverCodecs = new DefaultServerCodecs(eurekaServerConfig);
```

### 2.2.4 创建 Eureka-Client

```Java
ApplicationInfoManager applicationInfoManager;
if (eurekaClient == null) {
  EurekaInstanceConfig instanceConfig = isCloud(ConfigurationManager.getDeploymentContext())
          ? new CloudInstanceConfig()
          : new MyDataCenterInstanceConfig();
  
  applicationInfoManager = new ApplicationInfoManager(
          instanceConfig, new EurekaConfigBasedInstanceInfoProvider(instanceConfig).get());
  
  EurekaClientConfig eurekaClientConfig = new DefaultEurekaClientConfig();
  eurekaClient = new DiscoveryClient(applicationInfoManager, eurekaClientConfig);
} else {
  applicationInfoManager = eurekaClient.getApplicationInfoManager();
}
```

* Eureka-Server 内嵌 Eureka-Client，用于和 Eureka-Server 集群里其他节点通信交互。
* Eureka-Client 的初始化过程，在[《Eureka 源码解析 —— Eureka-Client 初始化（三）之 EurekaClient》「3. DiscoveryClient」](http://www.iocoder.cn/Eureka/eureka-client-init-third/?self)有详细解析。
* Eureka-Client 也可以通过 EurekaBootStrap 构造方法传递，实现代码如下：

    ```Java
    public class EurekaBootStrap implements ServletContextListener {
    
        private EurekaClient eurekaClient;
        
        public EurekaBootStrap(EurekaClient eurekaClient) {
            this.eurekaClient = eurekaClient;
        }
    
    }
    ```
    * **大多数情况下用不到**。

### 2.2.5 创建应用实例信息的注册表

```Java
PeerAwareInstanceRegistry registry;
if (isAws(applicationInfoManager.getInfo())) { // AWS 相关，跳过
  registry = new AwsInstanceRegistry(
          eurekaServerConfig,
          eurekaClient.getEurekaClientConfig(),
          serverCodecs,
          eurekaClient
  );
  awsBinder = new AwsBinderDelegate(eurekaServerConfig, eurekaClient.getEurekaClientConfig(), registry, applicationInfoManager);
  awsBinder.start();
} else {
  registry = new PeerAwareInstanceRegistryImpl(
          eurekaServerConfig,
          eurekaClient.getEurekaClientConfig(),
          serverCodecs,
          eurekaClient
  );
}
```

* 应用实例信息的注册表**类关系图**如下：

    ![](http://www.iocoder.cn/images/Eureka/2018_05_14/02.png)

* 本文不展开分享，在[《Eureka 源码解析 —— 注册表 InstanceRegistry 类关系》](http://www.iocoder.cn/Eureka/instance-registry-class-diagram/?self)详细解析。

### 2.2.6 创建 Eureka-Server 集群节点集合


```Java
PeerEurekaNodes peerEurekaNodes = getPeerEurekaNodes(
      registry,
      eurekaServerConfig,
      eurekaClient.getEurekaClientConfig(),
      serverCodecs,
      applicationInfoManager
);
```

* `com.netflix.eureka.cluster.PeerEurekaNodes`，Eureka-Server 集群节点集合，在[《Eureka 源码解析 —— Eureka-Server 集群同步》](http://www.iocoder.cn/Eureka/server-cluster/?self)详细解析。

### 2.2.7 创建 Eureka-Server 上下文

```Java
serverContext = new DefaultEurekaServerContext(
      eurekaServerConfig,
      serverCodecs,
      registry,
      peerEurekaNodes,
      applicationInfoManager
);
```
* `com.netflix.eureka.EurekaServerContext`，Eureka-Server 上下文**接口**，提供Eureka-Server 内部各组件对象的**初始化**、**关闭**、**获取**等方法。
* `com.netflix.eureka.EurekaServerContext.DefaultEurekaServerContext`，Eureka-Server 上下文**实现类**，实现代码如下：

    ```Java
    public class DefaultEurekaServerContext implements EurekaServerContext {
    
        /**
         * Eureka-Server 配置
         */
        private final EurekaServerConfig serverConfig;
        /**
         * Eureka-Server 请求和响应编解码器
         */
        private final ServerCodecs serverCodecs;
        /**
         * 应用实例信息的注册表
         */
        private final PeerAwareInstanceRegistry registry;
        /**
         * Eureka-Server 集群节点集合
         */
        private final PeerEurekaNodes peerEurekaNodes;
        /**
         * 应用实例信息管理器
         */
        private final ApplicationInfoManager applicationInfoManager;
        
        // .... 省略方法
    }
    ```

### 2.2.8 初始化 EurekaServerContextHolder

```Java
EurekaServerContextHolder.initialize(serverContext);
```

* `com.netflix.eureka.EurekaServerContextHolder`，Eureka-Server 上下文持有者。通过它，可以很方便的获取到 Eureka-Server 上下文，实现代码如下：

    ```Java
    public class EurekaServerContextHolder {
    
        /**
         * 持有者
         */
        private static EurekaServerContextHolder holder;
        /**
         * Eureka-Server 上下文
         */
        private final EurekaServerContext serverContext;
    
        private EurekaServerContextHolder(EurekaServerContext serverContext) {
            this.serverContext = serverContext;
        }
    
        public EurekaServerContext getServerContext() {
            return this.serverContext;
        }
    
        /**
         * 初始化
         *
         * @param serverContext Eureka-Server 上下文
         */
        public static synchronized void initialize(EurekaServerContext serverContext) {
            holder = new EurekaServerContextHolder(serverContext);
        }
    
        public static EurekaServerContextHolder getInstance() {
            return holder;
        }
    }
    ```

### 2.2.9 初始化 Eureka-Server 上下文

```Java
serverContext.initialize();
logger.info("Initialized server context");
```

* 调用 `ServerContext#initialize()` 方法，初始化 Eureka-Server 上下文，实现代码如下：

    ```Java
    // DefaultEurekaServerContext.java
    @Override
    public void initialize() throws Exception {
       logger.info("Initializing ...");
    
       // 启动 Eureka-Server 集群节点集合（复制）
       peerEurekaNodes.start();
       // 初始化 应用实例信息的注册表
       registry.init(peerEurekaNodes);
    
       logger.info("Initialized");
    }
    ```

### 2.2.10 从其他 Eureka-Server 拉取注册信息

```Java
// Copy registry from neighboring eureka node
int registryCount = registry.syncUp();
registry.openForTraffic(applicationInfoManager, registryCount);
```

* 本文不展开分享，在 [《Eureka 源码解析 —— Eureka-Server 集群同步》](http://www.iocoder.cn/Eureka/server-cluster/?self)详细解析。

### 2.2.11 注册监控

```Java
// Register all monitoring statistics.
EurekaMonitors.registerAllStats();
```

* 配合 [Netflix Servo](https://github.com/Netflix/servo) 实现监控信息采集。

# 3. Filter

Eureka-Server 过滤器( `javax.servlet.Filter` ) **顺序**如下：

* StatusFilter
* ServerRequestAuthFilter
* RateLimitingFilter
* GzipEncodingEnforcingFilter
* ServletContainer

## 3.1 StatusFilter

`com.netflix.eureka.StatusFilter`，Eureka-Server 状态过滤器。当 Eureka-Server 未处于开启( `InstanceStatus.UP` )状态，返回 HTTP 状态码 307 重定向，实现代码如下：

```Java
// StatusFilter.java
private static final int SC_TEMPORARY_REDIRECT = 307;
    
public void doFilter(ServletRequest request, ServletResponse response,
                   FilterChain chain) throws IOException, ServletException {
  InstanceInfo myInfo = ApplicationInfoManager.getInstance().getInfo();
  InstanceStatus status = myInfo.getStatus();
  if (status != InstanceStatus.UP && response instanceof HttpServletResponse) {
      HttpServletResponse httpRespone = (HttpServletResponse) response;
      httpRespone.sendError(SC_TEMPORARY_REDIRECT,
              "Current node is currently not ready to serve requests -- current status: "
                      + status + " - try another DS node: ");
  }
  chain.doFilter(request, response);
}
```

## 3.2 ServerRequestAuthFilter

`com.netflix.eureka.ServerRequestAuthFilter`，Eureka-Server 请求认证过滤器。Eureka-Server 未实现认证。目前打印访问的客户端名和版本号，配合 [Netflix Servo](https://github.com/Netflix/servo) 实现监控信息采集。实现代码如下：

    ```Java
    // ServerRequestAuthFilter.java
    protected void logAuth(ServletRequest request) {
       if (serverConfig.shouldLogIdentityHeaders()) {
           if (request instanceof HttpServletRequest) {
               HttpServletRequest httpRequest = (HttpServletRequest) request;
    
               String clientName = getHeader(httpRequest, AbstractEurekaIdentity.AUTH_NAME_HEADER_KEY);
               String clientVersion = getHeader(httpRequest, AbstractEurekaIdentity.AUTH_VERSION_HEADER_KEY);
    
               DynamicCounter.increment(MonitorConfig.builder(NAME_PREFIX + clientName + "-" + clientVersion).build());
           }
       }
    }
    ```

## 3.3 RateLimitingFilter

`com.netflix.eureka.RateLimitingFilter`，请求限流过滤器。在[《Eureka 源码解析 —— 基于令牌桶算法的 RateLimiter》](http://www.iocoder.cn/Eureka/rate-limiter/?self)详细解析。

## 3.4 GzipEncodingEnforcingFilter

`com.netflix.eureka.GzipEncodingEnforcingFilter`，GZIP 编码过滤器。

## 3.5 ServletContainer

`com.sun.jersey.spi.container.servlet.ServletContainer`，Jersey MVC 请求过滤器。

* Jersey MVC 模式如下图：

   > FROM [《Jersey框架的MVC》](http://blog.csdn.net/wangqyoho/article/details/51981916)
   > ![](http://www.iocoder.cn/images/Eureka/2018_05_14/03.png)
   
* 在 `com.netflix.eureka.resources` 包里，有所有的 Eureka-Server Jersey Resource ( Controller )。
* 过滤器在 `web.xml` 配置如下：

   ```XML
   <filter>
       <filter-name>jersey</filter-name>
       <filter-class>com.sun.jersey.spi.container.servlet.ServletContainer</filter-class>
       <init-param>
         <param-name>com.sun.jersey.config.property.WebPageContentRegex</param-name>
         <param-value>/(flex|images|js|css|jsp)/.*</param-value>
       </init-param>
       <init-param>
         <param-name>com.sun.jersey.config.property.packages</param-name>
         <param-value>com.sun.jersey;com.netflix</param-value>
       </init-param>
   
       <!-- GZIP content encoding/decoding -->
       <init-param>
         <param-name>com.sun.jersey.spi.container.ContainerRequestFilters</param-name>
         <param-value>com.sun.jersey.api.container.filter.GZIPContentEncodingFilter</param-value>
       </init-param>
       <init-param>
         <param-name>com.sun.jersey.spi.container.ContainerResponseFilters</param-name>
         <param-value>com.sun.jersey.api.container.filter.GZIPContentEncodingFilter</param-value>
       </init-param>
   </filter>
      
   <filter-mapping>
       <filter-name>jersey</filter-name>
       <url-pattern>/*</url-pattern>
   </filter-mapping>
   ```

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

啦啦啦，Eureka-Server 启动完成！

准备工作已经完成，可以开始更加有趣的注册、续约、取消注册、过期等等 Eureka-Client 与 Eureka-Server 的交互。

搞起！

胖友，分享一波朋友圈可好！？

