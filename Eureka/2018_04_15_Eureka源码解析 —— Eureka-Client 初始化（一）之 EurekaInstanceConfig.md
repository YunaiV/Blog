title: Eureka 源码解析 —— Eureka-Client 初始化（一）之 EurekaInstanceConfig
date: 2018-04-15
tags:
categories: Eureka
permalink: Eureka/eureka-client-init-first

---

摘要: 原创出处 http://www.iocoder.cn/Eureka/eureka-client-init-first/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Eureka 1.8.X 版本** 

- [1. 概述](http://www.iocoder.cn/Eureka/eureka-client-init-first/)
- [2. EurekaInstanceConfig](http://www.iocoder.cn/Eureka/eureka-client-init-first/)
  - [2.1 类关系图](http://www.iocoder.cn/Eureka/eureka-client-init-first/)
  - [2.2 配置属性](http://www.iocoder.cn/Eureka/eureka-client-init-first/)
  - [2.3 AbstractInstanceConfig](http://www.iocoder.cn/Eureka/eureka-client-init-first/)
  - [2.4 PropertiesInstanceConfig](http://www.iocoder.cn/Eureka/eureka-client-init-first/)
  - [2.5 MyDataCenterInstanceConfig](http://www.iocoder.cn/Eureka/eureka-client-init-first/)
  - [2.6 小结](http://www.iocoder.cn/Eureka/eureka-client-init-first/)
- [3. InstanceInfo](http://www.iocoder.cn/Eureka/eureka-client-init-first/)
- [4. ApplicationInfoManager](http://www.iocoder.cn/Eureka/eureka-client-init-first/)
- [666. 彩蛋](http://www.iocoder.cn/Eureka/eureka-client-init-first/)

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

本文主要分享 **Eureka-Client 自身初始化的过程**，不包含 Eureka-Client 向 Eureka-Server 的注册过程( 🙂后面会另外文章分享 )。

Eureka-Client 自身初始化过程中，涉及到主要对象如下图：

![](http://www.iocoder.cn/images/Eureka/2018_04_15/01.png)

1. **创建** EurekaInstanceConfig对象
1. 使用 EurekaInstanceConfig对象 **创建** InstanceInfo对象
1. 使用 EurekaInstanceConfig对象 + InstanceInfo对象 **创建** ApplicationInfoManager对象
1. **创建** EurekaClientConfig对象
1. 使用 ApplicationInfoManager对象 + EurekaClientConfig对象 **创建** EurekaClient对象

考虑到整个初始化的过程中涉及的配置特别多，拆分成三篇文章：

1. **【本文】**（一）EurekaInstanceConfig
2. （二）[EurekaClientConfig](http://www.iocoder.cn/Eureka/eureka-client-init-second/)
3. （三）[EurekaClient](http://www.iocoder.cn/Eureka/eureka-client-init-third/)

下面我们来看看每个**类**的实现。

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



# 2. EurekaInstanceConfig

`com.netflix.appinfo.EurekaInstanceConfig`，Eureka **应用实例**配置**接口**。在下文你会看到 EurekaClientConfig **接口**，两者的区别如下：

* EurekaInstanceConfig，重在**应用实例**，例如，应用名、应用的端口等等。此处应用指的是，Application Consumer 和 Application Provider。
* EurekaClientConfig，重在 **Eureka-Client**，例如， 连接的 Eureka-Server 的地址、获取服务提供者列表的频率、注册自身为服务提供者的频率等等。

![](http://www.iocoder.cn/images/Eureka/2018_04_15/02.jpeg)

## 2.1 类关系图

EurekaInstanceConfig 整体类关系如下图：

![](http://www.iocoder.cn/images/Eureka/2018_04_15/03.png)

* 本文只解析**红圈**部分类。
* EurekaArchaius2ClientConfig 基于 [Netflix Archaius 2.x](https://github.com/Netflix/archaius) 实现，目前还在开发中，因此暂不解析。
* CloudInstanceConfig、Ec2EurekaArchaius2InstanceConfig 基于亚马逊 AWS，大多数读者和我对 AWS 都不了解，因此暂不解析。

## 2.2 配置属性

点击 [EurekaInstanceConfig](https://github.com/YunaiV/eureka/blob/8b0f67ac33116ee05faad1ff5125034cfcf573bf/eureka-client/src/main/java/com/netflix/appinfo/EurekaInstanceConfig.java) 查看配置属性简介，已经添加中文注释，可以对照着英文注释一起理解。这里笔者摘出部分较为重要的属性：

* `#getLeaseRenewalIntervalInSeconds()` ：租约续约频率，单位：秒。应用不断通过按照该频率发送心跳给 Eureka-Server 以达到续约的作用。当 Eureka-Server 超过最大频率未收到续约（心跳），契约失效，进行应用移除。应用移除后，其他应用无法从 Eureka-Server 获取该应用。
* `#getLeaseExpirationDurationInSeconds()` ：契约过期时间，单位：秒。
* `#getDataCenterInfo()` ：数据中心信息。`com.netflix.appinfo.DataCenterInfo`，数据中心信息**接口**，目前较为简单，标记所属数据中心名。一般情况下，我们使用 `Name.MyOwn`。接口实现代码如下：

    ```Java
    public interface DataCenterInfo {
    
        /**
         * 数据中心名枚举
         */
        enum Name {
            Netflix,
            Amazon,
            MyOwn
        }
    
        /**
         * @return 归属的数据中心名
         */
        Name getName();
    }
    ```

* `#getNamespace()` ：配置命名空间，默认使用 `eureka`。以 `eureka-client.properties` 举个例子：

    ```Java
    eureka.name=eureka
    eureka.port=8080
    eureka.vipAddress=eureka.mydomain.net
    ```
    * 每个属性**最前面**的 `eureka` 即是配置命名空间，一般情况无需修改。

* TODO[0004]：健康检查
* `#isInstanceEnabledOnit()` ：应用初始化后是否开启。在[「3. InstanceInfo」](#)详细解析。

## 2.3 AbstractInstanceConfig

`com.netflix.appinfo.AbstractInstanceConfig`，Eureka **应用实例**配置**抽象基类**，主要实现一些相对**通用**的配置，实现代码如下：

```Java
public abstract class AbstractInstanceConfig implements EurekaInstanceConfig {

    /**
     * 契约过期时间，单位：秒
     */
    private static final int LEASE_EXPIRATION_DURATION_SECONDS = 90;
    /**
     * 租约续约频率，单位：秒。
     */
    private static final int LEASE_RENEWAL_INTERVAL_SECONDS = 30;
    /**
     * 应用 https 端口关闭
     */
    private static final boolean SECURE_PORT_ENABLED = false;
    /**
     * 应用 http 端口开启
     */
    private static final boolean NON_SECURE_PORT_ENABLED = true;
    /**
     * 应用 http 端口
     */
    private static final int NON_SECURE_PORT = 80;
    /**
     * 应用 https 端口
     */
    private static final int SECURE_PORT = 443;
    /**
     * 应用初始化后开启
     */
    private static final boolean INSTANCE_ENABLED_ON_INIT = false;
    /**
     * 主机信息
     * key：主机 IP 地址
     * value：主机名
     */
    private static final Pair<String, String> hostInfo = getHostInfo();
    /**
     * 数据中心信息
     */
    private DataCenterInfo info = new DataCenterInfo() {
        @Override
        public Name getName() {
            return Name.MyOwn;
        }
    };
    
    private static Pair<String, String> getHostInfo() {
        Pair<String, String> pair;
        try {
            InetAddress localHost = InetAddress.getLocalHost();
            pair = new Pair<String, String>(localHost.getHostAddress(), localHost.getHostName());
        } catch (UnknownHostException e) {
            logger.error("Cannot get host info", e);
            pair = new Pair<String, String>("", "");
        }
        return pair;
    }
    
    // .... 省略 setting / getting 方法
}
```

* `#getHostInfo()` 方法，获取本地服务器的主机名和主机 IP 地址。**如果主机有多网卡或者虚拟机网卡**，这块要小心，解决方式如下：
    * 手动配置本机的 `hostname` + `etc/hosts` 文件，从而映射主机名和 IP 地址。
    * 使用 Spring-Cloud-Eureka-Client 的话，参考[周立 —— 《Eureka服务注册过程详解之IpAddress》](http://www.itmuch.com/spring-cloud-code-read/spring-cloud-code-read-eureka-registry-ip/?from=www.iocoder.cn)解决。

## 2.4 PropertiesInstanceConfig

`com.netflix.appinfo.PropertiesInstanceConfig`，基于**配置文件**的 Eureka **应用实例**配置**抽象基类**，实现代码如下：

```Java
public abstract class PropertiesInstanceConfig extends AbstractInstanceConfig implements EurekaInstanceConfig {

    /**
     * 命名空间
     */
    protected final String namespace;
    /**
     * 配置文件对象
     */
    protected final DynamicPropertyFactory configInstance;
    /**
     * 应用分组
     * 从 环境变量 获取
     */
    private String appGrpNameFromEnv;

    public PropertiesInstanceConfig() {
        this(CommonConstants.DEFAULT_CONFIG_NAMESPACE);
    }

    public PropertiesInstanceConfig(String namespace) {
        this(namespace, new DataCenterInfo() {
            @Override
            public Name getName() {
                return Name.MyOwn;
            }
        });
    }

    public PropertiesInstanceConfig(String namespace, DataCenterInfo info) {
        super(info);
        // 设置 namespace，为 "." 结尾
        this.namespace = namespace.endsWith(".")
                ? namespace
                : namespace + ".";
        // 从 环境变量 获取 应用分组
        appGrpNameFromEnv = ConfigurationManager.getConfigInstance()
                .getString(FALLBACK_APP_GROUP_KEY, Values.UNKNOWN_APPLICATION);
        // 初始化 配置文件对象
        this.configInstance = Archaius1Utils.initConfig(CommonConstants.CONFIG_FILE_NAME);
    }

    @Override
    public String getAppGroupName() {
        return configInstance.getStringProperty(namespace + APP_GROUP_KEY, appGrpNameFromEnv).get().trim();
    }
}
```

* `configInstance` 属性，配置文件对象，基于 [Netflix Archaius 1.x](https://github.com/Netflix/archaius) 实现配置文件的读取。在 [`com.netflix.appinfo.PropertyBasedInstanceConfigConstants`](https://github.com/YunaiV/eureka/blob/671d7fc20bd6353040431d6e298eac5f82293497/eureka-client/src/main/java/com/netflix/appinfo/PropertyBasedInstanceConfigConstants.java) 可以看到配置文件的每个属性 KEY 。
* `appGrpNameFromEnv` 属性，应用分组，从**环境变量**中获取。从 `#getAppGroupName()` 方法中，可以看到优先还是从配置文件读取。设置方法如下：

    ```Java
    System.setProperty(FALLBACK_APP_GROUP_KEY, "app_gropu_name");
    ```
    * `FALLBACK_APP_GROUP_KEY`，私有静态变量，实际得使用 `NETFLIX_APP_GROUP`。
    * `com.netflix.config.ConfigurationManager` 可以从**环境变量**获取到值。

* 调用 `Archaius1Utils#initConfig(...)` 方法，初始化读取的配置文件对象，实现代码如下：

    ```Java
    public final class Archaius1Utils {
    
        private static final Logger logger = LoggerFactory.getLogger(Archaius1Utils.class);
    
        private static final String ARCHAIUS_DEPLOYMENT_ENVIRONMENT = "archaius.deployment.environment";
        private static final String EUREKA_ENVIRONMENT = "eureka.environment";
    
        public static DynamicPropertyFactory initConfig(String configName) {
            // 配置文件对象
            DynamicPropertyFactory configInstance = DynamicPropertyFactory.getInstance();
            // 配置文件名
            DynamicStringProperty EUREKA_PROPS_FILE = configInstance.getStringProperty("eureka.client.props", configName);
            // 配置文件环境
            String env = ConfigurationManager.getConfigInstance().getString(EUREKA_ENVIRONMENT, "test");
            ConfigurationManager.getConfigInstance().setProperty(ARCHAIUS_DEPLOYMENT_ENVIRONMENT, env);
            // 将配置文件加载到环境变量
            String eurekaPropsFile = EUREKA_PROPS_FILE.get();
            try {
                ConfigurationManager.loadCascadedPropertiesFromResources(eurekaPropsFile);
            } catch (IOException e) {
                logger.warn(
                        "Cannot find the properties specified : {}. This may be okay if there are other environment "
                                + "specific properties or the configuration is installed with a different mechanism.",
                        eurekaPropsFile);
    
            }
            return configInstance;
        }
        
    }
    ```
    * 从环境变量 `eureka.client.props`，获取配置文件名。如果未配置，使用参数 `configName`，即 `CommonConstants.CONFIG_FILE_NAME` ( `"eureka-client"` )。
    * 从环境变量 `eureka.environment` ( EUREKA_ENVIRONMENT )，获取配置文件环境。
    * 调用 `ConfigurationManager#loadCascadedPropertiesFromResources(...)` 方法，读取配置文件到环境变量，首先读取 `${eureka.client.props}` 对应的配置文件；然后读取 `${eureka.client.props}-${eureka.environment}` 对应的配置文件。若有相同属性，进行覆盖。

## 2.5 MyDataCenterInstanceConfig

`com.netflix.appinfo.MyDataCenterInstanceConfig`，非 AWS 数据中心的 Eureka **应用实例**配置**实现类**，实现代码如下：

```Java
public class MyDataCenterInstanceConfig extends PropertiesInstanceConfig implements EurekaInstanceConfig {

    public MyDataCenterInstanceConfig() {
    }

    public MyDataCenterInstanceConfig(String namespace) {
        super(namespace);
    }

    public MyDataCenterInstanceConfig(String namespace, DataCenterInfo dataCenterInfo) {
        super(namespace, dataCenterInfo);
    }

}
```

## 2.6 小结

一般情况下，使用 MyDataCenterInstanceConfig 配置 Eureka 应用实例。

在 Spring-Cloud-Eureka 里，**直接**基于 EurekaInstanceConfig 接口重新实现了配置类，实际逻辑差别不大，在[TODO[0007] ：《Spring-Cloud-Eureka-Client》](#)详细解析。

# 3. InstanceInfo

`com.netflix.appinfo.InstanceInfo`，**应用实例**信息。Eureka-Client 向 Eureka-Server **注册**该对象信息。注册成功后，可以被其他 Eureka-Client **发现**。

**本文仅分享 InstanceInfo 的初始化**。InstanceInfo 里和注册发现相关的属性和方法，暂时跳过。

`com.netflix.appinfo.providers.EurekaConfigBasedInstanceInfoProvider`，基于 EurekaInstanceConfig 创建 InstanceInfo 的工厂，实现代码如下：

```Java
  1: @Singleton
  2: public class EurekaConfigBasedInstanceInfoProvider implements Provider<InstanceInfo> {
  3:     private static final Logger LOG = LoggerFactory.getLogger(EurekaConfigBasedInstanceInfoProvider.class);
  4: 
  5:     private final EurekaInstanceConfig config;
  6: 
  7:     private InstanceInfo instanceInfo;
  8: 
  9:     @Inject(optional = true)
 10:     private VipAddressResolver vipAddressResolver = null;
 11: 
 12:     @Inject
 13:     public EurekaConfigBasedInstanceInfoProvider(EurekaInstanceConfig config) {
 14:         this.config = config;
 15:     }
 16: 
 17:     @Override
 18:     public synchronized InstanceInfo get() {
 19:         if (instanceInfo == null) {
 20:             // Build the lease information to be passed to the server based on config
 21:             // 创建 租约信息构建器，并设置属性
 22:             LeaseInfo.Builder leaseInfoBuilder = LeaseInfo.Builder.newBuilder()
 23:                     .setRenewalIntervalInSecs(config.getLeaseRenewalIntervalInSeconds())
 24:                     .setDurationInSecs(config.getLeaseExpirationDurationInSeconds());
 25: 
 26:             // 创建 VIP地址解析器
 27:             if (vipAddressResolver == null) {
 28:                 vipAddressResolver = new Archaius1VipAddressResolver();
 29:             }
 30: 
 31:             // Builder the instance information to be registered with eureka server
 32:             // 创建 应用实例信息构建器
 33:             InstanceInfo.Builder builder = InstanceInfo.Builder.newBuilder(vipAddressResolver);
 34: 
 35:             // 应用实例编号
 36:             // set the appropriate id for the InstanceInfo, falling back to datacenter Id if applicable, else hostname
 37:             String instanceId = config.getInstanceId();
 38:             DataCenterInfo dataCenterInfo = config.getDataCenterInfo();
 39:             if (instanceId == null || instanceId.isEmpty()) {
 40:                 if (dataCenterInfo instanceof UniqueIdentifier) {
 41:                     instanceId = ((UniqueIdentifier) dataCenterInfo).getId();
 42:                 } else {
 43:                     instanceId = config.getHostName(false);
 44:                 }
 45:             }
 46: 
 47:             // 获得 主机名
 48:             String defaultAddress;
 49:             if (config instanceof RefreshableInstanceConfig) {
 50:                 // Refresh AWS data center info, and return up to date address
 51:                 defaultAddress = ((RefreshableInstanceConfig) config).resolveDefaultAddress(false);
 52:             } else {
 53:                 defaultAddress = config.getHostName(false);
 54:             }
 55:             // fail safe
 56:             if (defaultAddress == null || defaultAddress.isEmpty()) {
 57:                 defaultAddress = config.getIpAddress();
 58:             }
 59: 
 60:             // 设置 应用实例信息构建器 的 属性
 61:             builder.setNamespace(config.getNamespace())
 62:                     .setInstanceId(instanceId)
 63:                     .setAppName(config.getAppname())
 64:                     .setAppGroupName(config.getAppGroupName())
 65:                     .setDataCenterInfo(config.getDataCenterInfo())
 66:                     .setIPAddr(config.getIpAddress())
 67:                     .setHostName(defaultAddress) // 主机名
 68:                     .setPort(config.getNonSecurePort())
 69:                     .enablePort(PortType.UNSECURE, config.isNonSecurePortEnabled())
 70:                     .setSecurePort(config.getSecurePort())
 71:                     .enablePort(PortType.SECURE, config.getSecurePortEnabled())
 72:                     .setVIPAddress(config.getVirtualHostName()) // VIP 地址
 73:                     .setSecureVIPAddress(config.getSecureVirtualHostName())
 74:                     .setHomePageUrl(config.getHomePageUrlPath(), config.getHomePageUrl())
 75:                     .setStatusPageUrl(config.getStatusPageUrlPath(), config.getStatusPageUrl())
 76:                     .setASGName(config.getASGName())
 77:                     .setHealthCheckUrls(config.getHealthCheckUrlPath(),
 78:                             config.getHealthCheckUrl(), config.getSecureHealthCheckUrl());
 79: 
 80:             // 应用初始化后是否开启
 81:             // Start off with the STARTING state to avoid traffic
 82:             if (!config.isInstanceEnabledOnit()) {
 83:                 InstanceStatus initialStatus = InstanceStatus.STARTING;
 84:                 LOG.info("Setting initial instance status as: " + initialStatus);
 85:                 builder.setStatus(initialStatus);
 86:             } else {
 87:                 LOG.info("Setting initial instance status as: {}. This may be too early for the instance to advertise "
 88:                          + "itself as available. You would instead want to control this via a healthcheck handler.",
 89:                          InstanceStatus.UP);
 90:             }
 91: 
 92:             // 设置 应用实例信息构建器 的 元数据( Metadata )集合
 93:             // Add any user-specific metadata information
 94:             for (Map.Entry<String, String> mapEntry : config.getMetadataMap().entrySet()) {
 95:                 String key = mapEntry.getKey();
 96:                 String value = mapEntry.getValue();
 97:                 builder.add(key, value);
 98:             }
 99: 
100:             // 创建 应用实例信息
101:             instanceInfo = builder.build();
102: 
103:             // 设置 应用实例信息 的 租约信息
104:             instanceInfo.setLeaseInfo(leaseInfoBuilder.build());
105:         }
106:         return instanceInfo;
107:     }
108: 
109: }
```

* 该类实现 `javax.inject.Provider` 接口，设置 InstanceInfo 的生成工厂。感兴趣的同学，可以点击[《Google-Guice入门介绍》](http://blog.csdn.net/derekjiang/article/details/7231490)搜索 **Provider** 关键字。目前处于**试验**阶段，未完成。
* `EurekaConfigBasedInstanceInfoProvider(config)` 构造方法，设置生成 InstanceInfo 的 EurekaInstanceConfig 配置。
* 调用 `#get()` 方法，根据 EurekaInstanceConfig 创建 InstanceInfo。InstanceInfo 的绝大数属性和 EurekaInstanceConfig 是一致的 。实现代码如下：

    * 第 21 至 24 行 ：创建租约信息构建器( [`com.netflix.appinfo.LeaseInfo.Builder`](https://github.com/YunaiV/eureka/blob/671d7fc20bd6353040431d6e298eac5f82293497/eureka-client/src/main/java/com/netflix/appinfo/LeaseInfo.java) )，并设置 `renewalIntervalInSecs` / `durationInSecs` 属性。
    * 第 26 至 29 行 ：创建 VIP地址解析器( `com.netflix.appinfo.providers.VipAddressResolver` )。实现代码如下：

        ```Java
        // VipAddressResolver.java
        public interface VipAddressResolver {
        
            String resolveDeploymentContextBasedVipAddresses(String vipAddressMacro);
        
        }
        
        public class Archaius1VipAddressResolver implements VipAddressResolver {
        
            private static final Pattern VIP_ATTRIBUTES_PATTERN = Pattern.compile("\\$\\{(.*?)\\}");
        
            @Override
            public String resolveDeploymentContextBasedVipAddresses(String vipAddressMacro) {
                if (vipAddressMacro == null) {
                    return null;
                }
                String result = vipAddressMacro;
                // 替换表达式
                Matcher matcher = VIP_ATTRIBUTES_PATTERN.matcher(result);
                while (matcher.find()) {
                    String key = matcher.group(1);
                    String value = DynamicPropertyFactory.getInstance().getStringProperty(key, "").get();
        
                    logger.debug("att:" + matcher.group());
                    logger.debug(", att key:" + key);
                    logger.debug(", att value:" + value);
                    logger.debug("");
        
                    result = result.replaceAll("\\$\\{" + key + "\\}", value);
                    matcher = VIP_ATTRIBUTES_PATTERN.matcher(result);
                }
        
                return result;
            }
        }
        ```
        * 使用 `#resolveDeploymentContextBasedVipAddresses()` 方法，将 **VIP地址** 里的 `${(.*?)}` 查找配置文件里的键值进行替换。例如，`${eureka.env}.domain.com`，查找配置文件里的键 `${eureka.env}` 对应值进行替换。TODO[0005]：调试下来，发现 Archaius 已经替换，等到找到答案修改此处。

    * 第 32 至 33 行 ：创建应用实例信息构建器( [`com.netflix.appinfo.InstanceInfo.Builder`](https://github.com/YunaiV/eureka/blob/671d7fc20bd6353040431d6e298eac5f82293497/eureka-client/src/main/java/com/netflix/appinfo/InstanceInfo.java) )。
    * 第 35 至 45 行 ：获得应用实例编号( `instanceId` )。
    * 第 47 至 58 行 ：获得主机名。
    * 第 60 至 78 行 ：设置应用实例信息构建器的属性。
    * 第 80 至 90 行 ：应用初始化后是否开启。
        * 第 82 至 85 行 ：应用**不开启**，应用实例处于 STARTING 状态。
        * 第 86 至 90 行 ：应用**开启**，应用实例处于 UP 状态。
        * **使用应用初始化后不开启**，可以通过调用 `ApplicationInfoManager#setInstanceStatus(...)` 方法改变应用实例状态，在[《Eureka 源码解析 —— 应用实例注册发现 （一）之注册》「2.1 应用实例信息复制器」](http://www.iocoder.cn/Eureka/instance-registry-register/?self)有详细解析。
    * 第 92 至 98 行 ：设置应用实例信息构建器的元数据( Metadata )集合。
    * 第 100 至 101 行 ：创建应用实例信息( [`com.netflix.appinfo.InstanceInfo`](https://github.com/YunaiV/eureka/blob/671d7fc20bd6353040431d6e298eac5f82293497/eureka-client/src/main/java/com/netflix/appinfo/InstanceInfo.java) )。
    * 第 103 至 104 行 ：设置应用实例信息的租约信息( [`com.netflix.appinfo.InstanceInfo`](https://github.com/YunaiV/eureka/blob/671d7fc20bd6353040431d6e298eac5f82293497/eureka-client/src/main/java/com/netflix/appinfo/InstanceInfo.java) )。

# 4. ApplicationInfoManager

`com.netflix.appinfo.ApplicationInfoManager`，应用信息管理器。实现代码如下：

```Java
public class ApplicationInfoManager {

    /**
     * 单例
     */
    private static ApplicationInfoManager instance = new ApplicationInfoManager(null, null, null);

    /**
     * 状态变更监听器
     */
    protected final Map<String, StatusChangeListener> listeners;
    /**
     * 应用实例状态匹配
     */
    private final InstanceStatusMapper instanceStatusMapper;
    /**
     * 应用实例信息
     */
    private InstanceInfo instanceInfo;
    /**
     * 应用实例配置
     */
    private EurekaInstanceConfig config;
    
    // ... 省略其它构造方法

    public ApplicationInfoManager(EurekaInstanceConfig config, InstanceInfo instanceInfo, OptionalArgs optionalArgs) {
        this.config = config;
        this.instanceInfo = instanceInfo;
        this.listeners = new ConcurrentHashMap<String, StatusChangeListener>();
        if (optionalArgs != null) {
            this.instanceStatusMapper = optionalArgs.getInstanceStatusMapper();
        } else {
            this.instanceStatusMapper = NO_OP_MAPPER;
        }

        // Hack to allow for getInstance() to use the DI'd ApplicationInfoManager
        instance = this;
    }
    
    // ... 省略其它方法
}
```

* `listeners` 属性，状态变更监听器集合。在[《Eureka 源码解析 —— 应用实例注册发现 （一）之注册》「2.1 应用实例信息复制器」](http://www.iocoder.cn/Eureka/instance-registry-register/?self)有详细解析。
* `instanceStatusMapper` 属性，应用实例状态匹配。实现代码如下：

    ```Java
    // ApplicationInfoManager.java
    
    public static interface InstanceStatusMapper {
    }
    
    private static final InstanceStatusMapper NO_OP_MAPPER = new InstanceStatusMapper() {
       @Override
       public InstanceStatus map(InstanceStatus prev) {
           return prev;
       }
    };
    ```
    * `#map` 方法，根据传入 `pre` 参数，转换成对应的应用实例状态。
    * 默认情况下，使用 NO_OP_MAPPER 。一般情况下，不需要关注该类。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

涉及到配置，内容初看起来会比较多，慢慢理解后，就会变得很“啰嗦”，请保持耐心。

胖友，分享一个朋友圈可好。

