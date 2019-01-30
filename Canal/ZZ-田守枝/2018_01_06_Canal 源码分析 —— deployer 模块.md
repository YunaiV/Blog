title: Canal 源码分析 —— deployer 模块
date: 2018-01-06
tag:
categories: Canal
permalink: Canal/tianshouzhi/deployer
author: 田守枝
from_url: http://www.tianshouzhi.com/api/tutorials/canal/381
wechat_url:

-------

摘要: 原创出处 http://www.tianshouzhi.com/api/tutorials/canal/381 「田守枝」欢迎转载，保留摘要，谢谢！

- [1、启动和停止脚本](http://www.iocoder.cn/Canal/tianshouzhi/deployer/)
- [2、CanalLauncher](http://www.iocoder.cn/Canal/tianshouzhi/deployer/)
- [3、CanalController](http://www.iocoder.cn/Canal/tianshouzhi/deployer/)
  - [3.1 配置解析相关代码](http://www.iocoder.cn/Canal/tianshouzhi/deployer/)
  - [3.2 准备canal server相关代码](http://www.iocoder.cn/Canal/tianshouzhi/deployer/)
  - [3.3 初始化zk相关代码](http://www.iocoder.cn/Canal/tianshouzhi/deployer/)
  - [3.4 CanalInstance运行状态监控相关代码](http://www.iocoder.cn/Canal/tianshouzhi/deployer/)
  - [3.5 autoScan机制相关代码](http://www.iocoder.cn/Canal/tianshouzhi/deployer/)
- [4 CanalController的start方法](http://www.iocoder.cn/Canal/tianshouzhi/deployer/)
- [5 总结](http://www.iocoder.cn/Canal/tianshouzhi/deployer/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

canal有两种使用方式：1、独立部署 2、内嵌到应用中。 deployer模块主要用于独立部署canal server。关于这两种方式的区别，请参见server模块源码分析。deployer模块源码目录结构如下所示：

![Image.png](http://static.tianshouzhi.com/ueditor/upload/image/20171212/1513090764865073917.png)

在独立部署canal时，需要首先对canal的源码进行打包

```bash
mvn clean install -Dmaven.test.skip -Denv=release
```

   以本教程使用1.0.24版本为例，打包后会在target目录生成一个以下两个文件：

![Image.png](http://static.tianshouzhi.com/ueditor/upload/image/20171212/1513091166982060800.png)

其中canal.deployer-1.0.24.tar.gz就是canal的独立部署包。解压缩后，目录如下所示。其中bin目录和conf目录(包括子目录spring)中的所有文件，都来自于deployer模块。

```bash
canal├── bin│   ├── startup.bat│   ├── startup.sh│   └── stop.sh├── conf│   ├── canal.properties│   ├── example│   │   └── instance.properties│   ├── logback.xml│   └── spring│       ├── default-instance.xml│       ├── file-instance.xml│       ├── group-instance.xml│       ├── local-instance.xml│       └── memory-instance.xml├── lib│   └── ....依赖的各种jar└── logs
```



deployer模块主要完成以下功能：

1、读取canal,properties配置文件

2、启动canal server，监听canal client的请求

3、启动canal instance，连接mysql数据库，伪装成slave，解析binlog

4、在canal的运行过程中，监听配置文件的变化

# 1、启动和停止脚本

bin目录中包含了canal的启动和停止脚本`startup.sh`和`stop.sh`，当我们要启动canal时，只需要输入以下命令即可

```bash
sh bin/startup.sh
```

​       在windows环境下，可以直接双击startup.bat。

在startup.sh脚本内，会调用com.alibaba.otter.canal.deployer.CanalLauncher类来进行启动，这是分析Canal源码的入口类，如下图所示：

![Image.png](http://static.tianshouzhi.com/ueditor/upload/image/20171212/1513091208060094491.png)

同时，startup.sh还会在bin目录下生成一个`canal.pid`文件，用于存储canal的进程id。当停止canal的时候

```bash
sh bin/stop.sh
```

会根据canal.pid文件中记录的进程id，kill掉canal进程，并且删除这个文件。

# 2、CannalLauncher

`CanalLauncher`是整个源码分析的入口类，代码相当简单。步骤是：

1、读取canal.properties文件中的配置

2、利用读取的配置构造一个CanalController实例，将所有的启动操作都委派给CanalController进行处理。

3、最后注册一个钩子函数，在JVM停止时同时也停止canal server。

com.alibaba.otter.canal.deployer.CanalLauncher

```java
public class CanalLauncher {     private static final String CLASSPATH_URL_PREFIX = "classpath:";    private static final Logger logger               = LoggerFactory.getLogger(CanalLauncher.class);     public static void main(String[] args) throws Throwable {        try {            //1、读取canal.properties文件中配置，默认读取classpath下的canal.properties            String conf = System.getProperty("canal.conf", "classpath:canal.properties");            Properties properties = new Properties();            if (conf.startsWith(CLASSPATH_URL_PREFIX)) {                conf = StringUtils.substringAfter(conf, CLASSPATH_URL_PREFIX);                properties.load(CanalLauncher.class.getClassLoader().getResourceAsStream(conf));            } else {                properties.load(new FileInputStream(conf));            }            //2、启动canal，首先将properties对象传递给CanalController，然后调用其start方法启动            logger.info("## start the canal server.");            final CanalController controller = new CanalController(properties);            controller.start();            logger.info("## the canal server is running now ......");            //3、关闭canal，通过添加JVM的钩子，JVM停止前会回调run方法，其内部调用controller.stop()方法进行停止            Runtime.getRuntime().addShutdownHook(new Thread() {                 public void run() {                    try {                        logger.info("## stop the canal server");                        controller.stop();                    } catch (Throwable e) {                        logger.warn("##something goes wrong when stopping canal Server:\n{}",                            ExceptionUtils.getFullStackTrace(e));                    } finally {                        logger.info("## canal server is down.");                    }                }             });        } catch (Throwable e) {            logger.error("## Something goes wrong when starting up the canal Server:\n{}",                ExceptionUtils.getFullStackTrace(e));            System.exit(0);        }    }}
```

可以看到，CanalLauncher实际上只是负责读取canal.properties配置文件，然后构造CanalController对象，并通过其start和stop方法来开启和停止canal。因此，如果说CanalLauncher是canal源码分析的入口类，那么CanalController就是canal源码分析的核心类。

# 3、CanalLauncher

在CanalController的构造方法中，会对配置文件内容解析，初始化相关成员变量，做好canal server的启动前的准备工作，之后在CanalLauncher中调用CanalController.start方法来启动。

CanalController中定义的相关字段和构造方法，如下所示：

```java
public class CanalController {     private static final Logger  logger   = LoggerFactory.getLogger(CanalController.class);    private Long                                     cid;    private String                                   ip;     private int                                  port;    // 默认使用spring的方式载入        private Map<String, InstanceConfig>              instanceConfigs;    private InstanceConfig                           globalInstanceConfig;    private Map<String, CanalConfigClient>           managerClients;    // 监听instance config的变化    private boolean                             autoScan = true;    private InstanceAction                           defaultAction;    private Map<InstanceMode, InstanceConfigMonitor> instanceConfigMonitors;    private CanalServerWithEmbedded                  embededCanalServer;    private CanalServerWithNetty                     canalServer;     private CanalInstanceGenerator                   instanceGenerator;    private ZkClientx                                zkclientx;     public CanalController(){        this(System.getProperties());    }     public CanalController(final Properties properties){        managerClients = MigrateMap.makeComputingMap(new Function<String, CanalConfigClient>() {             public CanalConfigClient apply(String managerAddress) {                return getManagerClient(managerAddress);            }        });         //1、配置解析           globalInstanceConfig = initGlobalConfig(properties);        instanceConfigs = new MapMaker().makeMap();             initInstanceConfig(properties);         // 2、准备canal server        cid = Long.valueOf(getProperty(properties, CanalConstants.CANAL_ID));        ip = getProperty(properties, CanalConstants.CANAL_IP);        port = Integer.valueOf(getProperty(properties, CanalConstants.CANAL_PORT));        embededCanalServer = CanalServerWithEmbedded.instance();        embededCanalServer.setCanalInstanceGenerator(instanceGenerator);// 设置自定义的instanceGenerator              canalServer = CanalServerWithNetty.instance();        canalServer.setIp(ip);        canalServer.setPort(port);                 //3、初始化zk相关代码         // 处理下ip为空，默认使用hostIp暴露到zk中              if (StringUtils.isEmpty(ip)) {            ip = AddressUtils.getHostIp();        }        final String zkServers = getProperty(properties, CanalConstants.CANAL_ZKSERVERS);        if (StringUtils.isNotEmpty(zkServers)) {            zkclientx = ZkClientx.getZkClient(zkServers);            // 初始化系统目录                     zkclientx.createPersistent(ZookeeperPathUtils.DESTINATION_ROOT_NODE, true);            zkclientx.createPersistent(ZookeeperPathUtils.CANAL_CLUSTER_ROOT_NODE, true);        }        //4 CanalInstance运行状态监控        final ServerRunningData serverData = new ServerRunningData(cid, ip + ":" + port);        ServerRunningMonitors.setServerData(serverData);        ServerRunningMonitors.setRunningMonitors(//...);         //5、autoScan机制相关代码           autoScan = BooleanUtils.toBoolean(getProperty(properties, CanalConstants.CANAL_AUTO_SCAN));        if (autoScan) {            defaultAction = new InstanceAction() {//....};             instanceConfigMonitors = //....        }    }....}
```

为了读者能够尽量容易的看出CanalController的构造方法中都做了什么，上面代码片段中省略了部分代码。这样，我们可以很明显的看出来， ，在CanalController构造方法中的代码分划分为了固定的几个处理步骤，下面按照几个步骤的划分，逐一进行讲解，并详细的介绍CanalController中定义的各个字段的作用。

## 3.1 配置解析相关代码

```java
// 初始化全局参数设置       globalInstanceConfig = initGlobalConfig(properties);instanceConfigs = new MapMaker().makeMap();// 初始化instance config       initInstanceConfig(properties);
```

### 3.1.1 globalInstanceConfig字段

表示canal instance的全局配置，类型为InstanceConfig，通过initGlobalConfig方法进行初始化。主要用于解析`canal.properties`以下几个配置项：

- **canal.instance.global.mode：**确定canal instance配置加载方式，取值有manager|spring两种方式
- **canal.instance.global.lazy：**确定canal instance是否延迟初始化
- **canal.instance.global.manager.address：**配置中心地址。如果canal.instance.global.mode=manager，需要提供此配置项
- **canal.instance.global.spring.xml：**spring配置文件路径。如果canal.instance.global.mode=spring，需要提供此配置项



initGlobalConfig源码如下所示：

```java
private InstanceConfig initGlobalConfig(Properties properties) {    InstanceConfig globalConfig = new InstanceConfig();    //读取canal.instance.global.mode    String modeStr = getProperty(properties, CanalConstants.getInstanceModeKey(CanalConstants.GLOBAL_NAME));    if (StringUtils.isNotEmpty(modeStr)) {        //将modelStr转成枚举InstanceMode，这是一个枚举类，只有2个取值，SPRING\MANAGER，对应两种配置方式        globalConfig.setMode(InstanceMode.valueOf(StringUtils.upperCase(modeStr)));    }    //读取canal.instance.global.lazy    String lazyStr = getProperty(properties, CanalConstants.getInstancLazyKey(CanalConstants.GLOBAL_NAME));    if (StringUtils.isNotEmpty(lazyStr)) {        globalConfig.setLazy(Boolean.valueOf(lazyStr));    }   //读取canal.instance.global.manager.address    String managerAddress = getProperty(properties,        CanalConstants.getInstanceManagerAddressKey(CanalConstants.GLOBAL_NAME));    if (StringUtils.isNotEmpty(managerAddress)) {        globalConfig.setManagerAddress(managerAddress);    }    //读取canal.instance.global.spring.xml    String springXml = getProperty(properties, CanalConstants.getInstancSpringXmlKey(CanalConstants.GLOBAL_NAME));    if (StringUtils.isNotEmpty(springXml)) {        globalConfig.setSpringXml(springXml);    }     instanceGenerator = //...初始化instanceGenerator      return globalConfig;}
```

其中`canal.instance.global.mode`用于确定canal instance的全局配置加载方式，其取值范围有2个：`spring`、`manager`。我们知道一个canal server中可以启动多个canal instance，每个instance都有各自的配置。instance的配置也可以放在本地，也可以放在远程配置中心里。我们可以自定义每个canal instance配置文件存储的位置，如果所有canal instance的配置都在本地或者远程，此时我们就可以通过canal.instance.global.mode这个配置项，来统一的指定配置文件的位置，避免为每个canal instance单独指定。

其中：

**spring方式：**

表示所有的canal instance的配置文件位于本地。此时，我们必须提供配置项canal.instance.global.spring.xml指定spring配置文件的路径。canal提供了多个spring配置文件：file-instance.xml、default-instance.xml、memory-instance.xml、local-instance.xml、group-instance.xml。这么多配置文件主要是为了支持canal instance不同的工作方式。我们在稍后将会讲解各个配置文件的区别。而在这些配置文件的开头，我们无一例外的可以看到以下配置：

```xml
<bean class="com.alibaba.otter.canal.instance.spring.support.PropertyPlaceholderConfigurer" lazy-init="false">        <property name="ignoreResourceNotFound" value="true" />        <property name="systemPropertiesModeName" value="SYSTEM_PROPERTIES_MODE_OVERRIDE"/><!-- 允许system覆盖 -->        <property name="locationNames">            <list>                <value>classpath:canal.properties</value>                <value>classpath:${canal.instance.destination:}/instance.properties</value>            </list>        </property>    </bean>
```

这里我们可以看到，所谓通过spring方式加载canal instance配置，无非就是通过spring提供的PropertyPlaceholderConfigurer来加载canal instance的配置文件instance.properties。

这里instance.properties的文件完整路径是$，其中{canal.instance.destination:}/instance.properties，其中${canal.instance.destination}是一个变量。这是因为我们可以在一个canal server中配置多个canal instance，每个canal instance配置文件的名称都是instance.properties，因此我们需要通过目录进行区分。例如我们通过配置项canal.destinations指定多个canal instance的名字

```bash
canal.destinations= example1,example2
```

此时我们就要conf目录下，新建两个子目录example1和example2，每个目录下各自放置一个instance.properties。

canal在初始化时就会分别使用example1和example2来替换${canal.instance.destination:}，从而分别根据example1/instance.properties和example2/instance.properties创建2个canal instance。

**manager方式：**

表示所有的canal instance的配置文件位于远程配置中心，此时我们必须提供配置项 canal.instance.global.manager.address来指定远程配置中心的地址。目前alibaba内部配置使用这种方式。开发者可以自己实现CanalConfigClient，连接各自的管理系统，完成接入。

### 3.1.2 instanceGenerator字段

类型为`CanalInstanceGenerator`。在initGlobalConfig方法中，除了创建了globalInstanceConfig实例，同时还为字段instanceGenerator字段进行了赋值。

顾名思义，这个字段用于创建`CanalInstance`实例。这是instance模块中的类，其作用就是为canal.properties文件中`canal.destinations`配置项列出的每个destination，创建一个CanalInstance实例。CanalInstanceGenerator是一个接口，定义如下所示：

```java
public interface CanalInstanceGenerator {     /**     * 通过 destination 产生特定的 {@link CanalInstance}     */    CanalInstance generate(String destination);}
```

针对spring和manager两种instance配置的加载方式，CanalInstanceGenerator提供了两个对应的实现类，如下所示：

![Image.png](http://static.tianshouzhi.com/ueditor/upload/image/20171212/1513090905643089186.png)

instanceGenerator字段通过一个匿名内部类进行初始化。其内部会判断配置的各个destination的配置加载方式，spring 或者manager。

```java
instanceGenerator = new CanalInstanceGenerator() {         public CanalInstance generate(String destination) {           //1、根据destination从instanceConfigs获取对应的InstanceConfig对象            InstanceConfig config = instanceConfigs.get(destination);            if (config == null) {                throw new CanalServerException("can't find destination:{}");            }          //2、如果destination对应的InstanceConfig的mode是manager方式，使用ManagerCanalInstanceGenerator            if (config.getMode().isManager()) {                ManagerCanalInstanceGenerator instanceGenerator = new ManagerCanalInstanceGenerator();                instanceGenerator.setCanalConfigClient(managerClients.get(config.getManagerAddress()));                return instanceGenerator.generate(destination);            } else if (config.getMode().isSpring()) {          //3、如果destination对应的InstanceConfig的mode是spring方式，使用SpringCanalInstanceGenerator                SpringCanalInstanceGenerator instanceGenerator = new SpringCanalInstanceGenerator();                synchronized (this) {                    try {                        // 设置当前正在加载的通道，加载spring查找文件时会用到该变量                                                System.setProperty(CanalConstants.CANAL_DESTINATION_PROPERTY, destination);                        instanceGenerator.setBeanFactory(getBeanFactory(config.getSpringXml()));                        return instanceGenerator.generate(destination);                    } catch (Throwable e) {                        logger.error("generator instance failed.", e);                        throw new CanalException(e);                    } finally {                        System.setProperty(CanalConstants.CANAL_DESTINATION_PROPERTY, "");                    }                }            } else {                throw new UnsupportedOperationException("unknow mode :" + config.getMode());            }         }     };
```

上述代码中的第1步比较变态，从instanceConfigs中根据destination作为参数，获得对应的InstanceConfig。而instanceConfigs目前还没有被初始化，这个字段是在稍后将后将要讲解的initInstanceConfig方法初始化的，不过由于这是一个引用类型，当initInstanceConfig方法被执行后，instanceConfigs字段中也就有值了。目前，我们姑且认为， instanceConfigs这个Map<String, InstanceConfig>类型的字段已经被初始化好了。

2、3两步用于确定是instance的配置加载方式是spring还是manager，如果是spring，就使用SpringCanalInstanceGenerator创建CanalInstance实例，如果是manager，就使用ManagerCanalInstanceGenerator创建CanalInstance实例。

由于目前manager方式的源码并未开源，因此，我们只分析SpringCanalInstanceGenerator相关代码。

上述代码中，首先创建了一个SpringCanalInstanceGenerator实例，然后往里面设置了一个BeanFactory。

```java
instanceGenerator.setBeanFactory(getBeanFactory(config.getSpringXml()));
```

其中config.getSpringXml()返回的就是我们在canal.properties中通过canal.instance.global.spring.xml配置项指定了spring配置文件路径。getBeanFactory方法源码如下所示：

```java
private BeanFactory getBeanFactory(String springXml) {        ApplicationContext applicationContext = new ClassPathXmlApplicationContext(springXml);        return applicationContext;    }
```



往`SpringCanalInstanceGenerator`设置了BeanFactory之后，就可以通过其的generate方法获得CanalInstance实例。

SpringCanalInstanceGenerator的源码如下所示：

```java
public class SpringCanalInstanceGenerator implements CanalInstanceGenerator, BeanFactoryAware {     private String      defaultName = "instance";    private BeanFactory beanFactory;     public CanalInstance generate(String destination) {        String beanName = destination;        //首先判断beanFactory是否包含以destination为id的bean        if (!beanFactory.containsBean(beanName)) {            beanName = defaultName;//如果没有，设置要获取的bean的id为instance。        }        //以默认的bean的id值"instance"来获取CanalInstance实例        return (CanalInstance) beanFactory.getBean(beanName);    }     public void setBeanFactory(BeanFactory beanFactory) throws BeansException {        this.beanFactory = beanFactory;    } }
```

首先尝试以传入的参数destination来获取CanalInstance实例，如果没有，就以默认的bean的id值"instance"来获取CanalInstance实例。事实上，如果你没有修改spring配置文件，那么默认的名字就是instance。事实上，在canal提供的各个spring配置文件xxx-instance.xml中，都有类似以下配置：

```xml
 <bean id="instance" class="com.alibaba.otter.canal.instance.spring.CanalInstanceWithSpring">      <property name="destination" value="${canal.instance.destination}" />      <property name="eventParser">         <ref local="eventParser" />      </property>      <property name="eventSink">         <ref local="eventSink" />      </property>      <property name="eventStore">         <ref local="eventStore" />      </property>      <property name="metaManager">         <ref local="metaManager" />      </property>      <property name="alarmHandler">         <ref local="alarmHandler" />      </property>   </bean>
```

上面的代码片段中，我们看到的确有一个bean的名字是instance，其类型是`CanalInstanceWithSpring`，这是CanalInstance接口的实现类。类似的，我们可以想到在manager配置方式下，获取的CanalInstance实现类是`CanalInstanceWithManager`。事实上，你想的没错，CanalInstance的类图继承关系如下所示：

![Image.png](http://static.tianshouzhi.com/ueditor/upload/image/20171212/1513090935159096844.png)

需要注意的是，到目前为止，我们只是创建好了CanalInstanceGenerator，而CanalInstance尚未创建。在CanalController的start方法被调用时，CanalInstance才会被真正的创建，相关源码将在稍后分析。

### 3.1.3 instanceConfigs字段

类型为Map<String, InstanceConfig>。前面提到初始化instanceGenerator后，当其generate方法被调用时，会尝试从instanceConfigs根据一个destination获取对应的`InstanceConfig`，现在分析instanceConfigs的相关初始化代码。

我们知道globalInstanceConfig定义全局的配置加载方式。如果需要把部分CanalInstance配置放于本地，另外一部分CanalIntance配置放于远程配置中心，则只通过全局方式配置，无法达到这个要求。虽然这种情况很少见，但是为了提供最大的灵活性，canal支持每个CanalIntance自己来定义自己的加载方式，来覆盖默认的全局配置加载方式。而每个destination对应的InstanceConfig配置就存放于instanceConfigs字段中。

举例来说：

```bash
//当前server上部署的instance列表canal.destinations=instance1,instance2  //instance配置全局加载方式canal.instance.global.mode = springcanal.instance.global.lazy = falsecanal.instance.global.spring.xml = classpath:spring/file-instance.xml //instance1覆盖全局加载方式canal.instance.instance1.mode = managercanal.instance.instance1.manager.address = 127.0.0.1:1099canal.instance.instance1.lazy = tue
```

这段配置中，设置了instance的全局加载方式为spring，instance1覆盖了全局配置，使用manager方式加载配置。而instance2没有覆盖配置，因此默认使用spring加载方式。

instanceConfigs字段通过initInstanceConfig方法进行初始化

```java
instanceConfigs = new MapMaker().makeMap();//这里利用Google Guava框架的MapMaker创建Map实例并赋值给instanceConfigs// 初始化instance configinitInstanceConfig(properties);
```

initInstanceConfig方法源码如下：

```java
private void initInstanceConfig(Properties properties) {    //读取配置项canal.destinations    String destinationStr = getProperty(properties, CanalConstants.CANAL_DESTINATIONS);    //以","分割canal.destinations，得到一个数组形式的destination    String[] destinations = StringUtils.split(destinationStr, CanalConstants.CANAL_DESTINATION_SPLIT);    for (String destination : destinations) {        //为每一个destination生成一个InstanceConfig实例        InstanceConfig config = parseInstanceConfig(properties, destination);        //将destination对应的InstanceConfig放入instanceConfigs中        InstanceConfig oldConfig = instanceConfigs.put(destination, config);         if (oldConfig != null) {            logger.warn("destination:{} old config:{} has replace by new config:{}", new Object[] { destination,                    oldConfig, config });        }    }}
```

上面代码片段中，首先解析canal.destinations配置项，可以理解一个destination就对应要初始化一个canal instance。针对每个destination会创建各自的InstanceConfig，最终都会放到instanceConfigs这个Map中。

各个destination对应的InstanceConfig都是通过parseInstanceConfig方法来解析

```java
private InstanceConfig parseInstanceConfig(Properties properties, String destination) {    //每个destination对应的InstanceConfig都引用了全局的globalInstanceConfig    InstanceConfig config = new InstanceConfig(globalInstanceConfig);    //...其他几个配置项与获取globalInstanceConfig类似，不再赘述，唯一注意的的是配置项的key部分中的global变成传递进来的destination    return config;}
```

此时我们可以看一下InstanceConfig类的源码：

```java
public class InstanceConfig {     private InstanceConfig globalConfig;    private InstanceMode   mode;    private Boolean        lazy;    private String         managerAddress;    private String         springXml;     public InstanceConfig(){     }     public InstanceConfig(InstanceConfig globalConfig){        this.globalConfig = globalConfig;    }     public static enum InstanceMode {        SPRING, MANAGER;         public boolean isSpring() {            return this == InstanceMode.SPRING;        }         public boolean isManager() {            return this == InstanceMode.MANAGER;        }    }     public Boolean getLazy() {        if (lazy == null && globalConfig != null) {            return globalConfig.getLazy();        } else {            return lazy;        }    }     public void setLazy(Boolean lazy) {        this.lazy = lazy;    }     public InstanceMode getMode() {        if (mode == null && globalConfig != null) {            return globalConfig.getMode();        } else {            return mode;        }    }     public void setMode(InstanceMode mode) {        this.mode = mode;    }     public String getManagerAddress() {        if (managerAddress == null && globalConfig != null) {            return globalConfig.getManagerAddress();        } else {            return managerAddress;        }    }     public void setManagerAddress(String managerAddress) {        this.managerAddress = managerAddress;    }     public String getSpringXml() {        if (springXml == null && globalConfig != null) {            return globalConfig.getSpringXml();        } else {            return springXml;        }    }     public void setSpringXml(String springXml) {        this.springXml = springXml;    }     public String toString() {        return ToStringBuilder.reflectionToString(this, CanalToStringStyle.DEFAULT_STYLE);    } }
```

  可以看到，InstanceConfig类中维护了一个globalConfig字段，其类型也是InstanceConfig。而其相关get方法在执行时，会按照以下逻辑进行判断：如果没有自身没有这个配置，则返回全局配置，如果有，则返回自身的配置。通过这种方式实现对全局配置的覆盖。

##  3.2 准备canal server相关代码

```java
cid = Long.valueOf(getProperty(properties, CanalConstants.CANAL_ID));ip = getProperty(properties, CanalConstants.CANAL_IP);port = Integer.valueOf(getProperty(properties, CanalConstants.CANAL_PORT)); embededCanalServer = CanalServerWithEmbedded.instance();embededCanalServer.setCanalInstanceGenerator(instanceGenerator);// 设置自定义的instanceGeneratorcanalServer = CanalServerWithNetty.instance();canalServer.setIp(ip);canalServer.setPort(port);
```

上述代码中，首先解析了cid、ip、port字段，其中：

**cid：**Long，对应canal.properties文件中的canal.id，目前无实际用途

**ip：**String，对应canal.properties文件中的canal.ip，canal server监听的ip。

**port：**int，对应canal.properties文件中的canal.port，canal server监听的端口



之后分别为以下两个字段赋值：

**embededCanalServer：**类型为CanalServerWithEmbedded

**canalServer：**类型为CanalServerWithNetty

`CanalServerWithEmbedded `和 `CanalServerWithNetty`都实现了CanalServer接口，且都实现了单例模式，通过静态方法instance获取实例。

关于这两种类型的实现，canal官方文档有以下描述：

![Image.png](http://static.tianshouzhi.com/ueditor/upload/image/20171212/1513090984158026921.png)

说白了，就是我们可以不必独立部署canal server。在应用直接使用CanalServerWithEmbedded直连mysql数据库。如果觉得自己的技术hold不住相关代码，就独立部署一个canal server，使用canal提供的客户端，连接canal server获取binlog解析后数据。而CanalServerWithNetty是在CanalServerWithEmbedded的基础上做的一层封装，用于与客户端通信。

在独立部署canal server时，Canal客户端发送的所有请求都交给CanalServerWithNetty处理解析，解析完成之后委派给了交给CanalServerWithEmbedded进行处理。因此CanalServerWithNetty就是一个马甲而已。CanalServerWithEmbedded才是核心。

因此，在上述代码中，我们看到，用于生成CanalInstance实例的instanceGenerator被设置到了CanalServerWithEmbedded中，而ip和port被设置到CanalServerWithNetty中。

关于CanalServerWithNetty如何将客户端的请求委派给CanalServerWithEmbedded进行处理，我们将在server模块源码分析中进行讲解。

## 3.3 初始化zk相关代码

```java
   //读取canal.properties中的配置项canal.zkServers，如果没有这个配置，则表示项目不使用zkfinal String zkServers = getProperty(properties, CanalConstants.CANAL_ZKSERVERS);if (StringUtils.isNotEmpty(zkServers)) {    //创建zk实例    zkclientx = ZkClientx.getZkClient(zkServers);    // 初始化系统目录    //destination列表，路径为/otter/canal/destinations    zkclientx.createPersistent(ZookeeperPathUtils.DESTINATION_ROOT_NODE, true);    //整个canal server的集群列表，路径为/otter/canal/cluster    zkclientx.createPersistent(ZookeeperPathUtils.CANAL_CLUSTER_ROOT_NODE, true);}
```

canal支持利用了zk来完成HA机制、以及将当前消费到到的mysql的binlog位置记录到zk中。ZkClientx是canal对ZkClient进行了一层简单的封装。

显然，当我们没有配置canal.zkServers，那么zkclientx不会被初始化。

关于Canal如何利用ZK做HA，我们将在稍后的代码中进行分。而利用zk记录binlog的消费进度，将在之后的章节进行分析。

## 3.4 CanalInstance运行状态监控相关代码

由于这段代码比较长且恶心，这里笔者暂时对部分代码进行省略，以便读者看清楚整各脉络

```java
final ServerRunningData serverData = new ServerRunningData(cid, ip + ":" + port);        ServerRunningMonitors.setServerData(serverData);        ServerRunningMonitors.setRunningMonitors(MigrateMap.makeComputingMap(new Function<String, ServerRunningMonitor>() {            public ServerRunningMonitor apply(final String destination) {                ServerRunningMonitor runningMonitor = new ServerRunningMonitor(serverData);                runningMonitor.setDestination(destination);                runningMonitor.setListener(new ServerRunningListener() {....});//省略ServerRunningListener的具体实现                if (zkclientx != null) {                    runningMonitor.setZkClient(zkclientx);                }                // 触发创建一下cid节点                runningMonitor.init();                return runningMonitor;            }        }));
```

上述代码中，`ServerRunningMonitors`是ServerRunningMonitor对象的容器，而`ServerRunningMonitor`用于监控CanalInstance。

​    canal会为每一个destination创建一个CanalInstance，每个CanalInstance都会由一个ServerRunningMonitor来进行监控。而ServerRunningMonitor统一由ServerRunningMonitors进行管理。

​    除了CanalInstance需要监控，CanalServer本身也需要监控。因此我们在代码一开始，就看到往ServerRunningMonitors设置了一个ServerRunningData对象，封装了canal server监听的ip和端口等信息。

ServerRunningMonitors源码如下所示：

```java
public class ServerRunningMonitors {    private static ServerRunningData serverData;    private static Map               runningMonitors; // <String,ServerRunningMonitor>    public static ServerRunningData getServerData() {        return serverData;    }    public static Map<String, ServerRunningMonitor> getRunningMonitors() {        return runningMonitors;    }    public static ServerRunningMonitor getRunningMonitor(String destination) {        return (ServerRunningMonitor) runningMonitors.get(destination);    }    public static void setServerData(ServerRunningData serverData) {        ServerRunningMonitors.serverData = serverData;    }    public static void setRunningMonitors(Map runningMonitors) {        ServerRunningMonitors.runningMonitors = runningMonitors;    }}
```

ServerRunningMonitors的setRunningMonitors方法接收的参数是一个Map，其中Map的key是destination，value是ServerRunningMonitor，也就是说针对每一个destination都有一个ServerRunningMonitor来监控。

上述代码中，在往ServerRunningMonitors设置Map时，是通过MigrateMap.makeComputingMap方法来创建的，其接受一个Function类型的参数，这是guava中定义的接口，其声明了apply抽象方法。其工作原理可以通过下面代码片段进行介绍：

```java
Map<String, User> map = MigrateMap.makeComputingMap(new Function<String, User>() {            @Override            public User apply(String name) {                return new User(name);            }        });User user = map.get("tianshouzhi");//第一次获取时会创建assert user != null;assert user == map.get("tianshouzhi");//之后获取，总是返回之前已经创建的对象
```

这段代码中，我们利用MigrateMap.makeComputingMap创建了一个Map，其中key为String类型，value为User类型。当我们调用map.get("tianshouzhi")方法，最开始这个Map中并没有任何key/value的，于是其就会回调Function的apply方法，利用参数"tianshouzhi"创建一个User对象并返回。之后当我们再以"tianshouzhi"为key从Map中获取User对象时，会直接将前面创建的对象返回。不会回调apply方法，也就是说，只有在第一次尝试获取时，才会回调apply方法。

​    而在上述代码中，实际上就利用了这个特性，只不过是根据destination获取ServerRunningMonitor对象，如果不存在就创建。

在创建ServerRunningMonitor对象时，首先根据ServerRunningData创建ServerRunningMonitor实例，之后设置了destination和`ServerRunningListener`对象，接着，判断如果zkClientx字段如果不为空，也设置到ServerRunningMonitor中，最后调用init方法进行初始化。

```java
ServerRunningMonitor runningMonitor = new ServerRunningMonitor(serverData);runningMonitor.setDestination(destination);runningMonitor.setListener(new ServerRunningListener(){...})//省略ServerRunningListener具体代码if (zkclientx != null) {runningMonitor.setZkClient(zkclientx);}// 触发创建一下cid节点runningMonitor.init();return runningMonitor;
```

ServerRunningListener的实现如下：

```java
new ServerRunningListener() {    /*内部调用了embededCanalServer的start(destination)方法。    此处需要划重点，说明每个destination对应的CanalInstance是通过embededCanalServer的start方法启动的，    这与我们之前分析将instanceGenerator设置到embededCanalServer中可以对应上。    embededCanalServer负责调用instanceGenerator生成CanalInstance实例，并负责其启动。*/     public void processActiveEnter() {         try {             MDC.put(CanalConstants.MDC_DESTINATION, String.valueOf(destination));             embededCanalServer.start(destination);         } finally {             MDC.remove(CanalConstants.MDC_DESTINATION);         }     }  //内部调用embededCanalServer的stop(destination)方法。与上start方法类似，只不过是停止CanalInstance。     public void processActiveExit() {         try {             MDC.put(CanalConstants.MDC_DESTINATION, String.valueOf(destination));             embededCanalServer.stop(destination);         } finally {             MDC.remove(CanalConstants.MDC_DESTINATION);         }     }     /*处理存在zk的情况下，在Canalinstance启动之前，在zk中创建节点。     路径为：/otter/canal/destinations/{0}/cluster/{1}，其0会被destination替换，1会被ip:port替换。     此方法会在processActiveEnter()之前被调用*/     public void processStart() {         try {             if (zkclientx != null) {                 final String path = ZookeeperPathUtils.getDestinationClusterNode(destination, ip + ":" + port);                 initCid(path);                 zkclientx.subscribeStateChanges(new IZkStateListener() {                     public void handleStateChanged(KeeperState state) throws Exception {                     }                     public void handleNewSession() throws Exception {                         initCid(path);                     }                 });             }         } finally {             MDC.remove(CanalConstants.MDC_DESTINATION);         }     }//处理存在zk的情况下，在Canalinstance停止前，释放zk节点，路径为/otter/canal/destinations/{0}/cluster/{1}，//其0会被destination替换，1会被ip:port替换。此方法会在processActiveExit()之前被调用     public void processStop() {         try {             MDC.put(CanalConstants.MDC_DESTINATION, String.valueOf(destination));             if (zkclientx != null) {                 final String path = ZookeeperPathUtils.getDestinationClusterNode(destination, ip + ":" + port);                 releaseCid(path);             }         } finally {             MDC.remove(CanalConstants.MDC_DESTINATION);         }     }}
```

上述代码中，我们可以看到启动一个CanalInstance实际上是在ServerRunningListener的processActiveEnter方法中，通过调用embededCanalServer的start(destination)方法进行的，对于停止也是类似。

那么ServerRunningListener中的相关方法到底是在哪里回调的呢？我们可以在ServerRunningMonitor的start和stop方法中找到答案，这里只列出start方法。

```java
public class ServerRunningMonitor extends AbstractCanalLifeCycle { ...public void start() {    super.start();    processStart();//其内部会调用ServerRunningListener的processStart()方法    if (zkClient != null) {//存在zk，以HA方式启动        // 如果需要尽可能释放instance资源，不需要监听running节点，不然即使stop了这台机器，另一台机器立马会start        String path = ZookeeperPathUtils.getDestinationServerRunning(destination);        zkClient.subscribeDataChanges(path, dataListener);         initRunning();    } else {//没有zk，直接启动        processActiveEnter();    }} //...stop方法逻辑类似，相关代码省略}
```

当ServerRunningMonitor的start方法被调用时，其首先会直接调用processStart方法，这个方法内部直接调了ServerRunningListener的processStart()方法，源码如下所示。通过前面的分析，我们已经知道在存在zkClient!=null的情况，会往zk中创建一个节点。

```java
private void processStart() {    if (listener != null) {        try {            listener.processStart();        } catch (Exception e) {            logger.error("processStart failed", e);        }    }}
```

之后会判断是否存在zkClient，如果不存在，则以本地方式启动，如果存在，则以HA方式启动。我们知道，canal server可以部署成两种方式：集群方式或者独立部署。其中集群方式是利用zk来做HA，独立部署则可以直接进行启动。我们先来看比较简单的直接启动。

**直接启动：**

不存在zk的情况下，会进入else代码块，调用processActiveEnter方法，其内部调用了listener的processActiveEnter，启动相应destination对应的CanalInstance。

```java
private void processActiveEnter() {    if (listener != null) {        try {            listener.processActiveEnter();        } catch (Exception e) {            logger.error("processActiveEnter failed", e);        }    }}
```



**HA方式启动：**

存在zk，说明canal server可能做了集群，因为canal就是利用zk来做HA的。首先根据destination构造一个zk的节点路径，然后进行监听。

```java
/*构建临时节点的路径：/otter/canal/destinations/{0}/running，其中占位符{0}会被destination替换。在集群模式下，可能会有多个canal server共同处理同一个destination，在某一时刻，只能由一个canal server进行处理，处理这个destination的canal server进入running状态，其他canal server进入standby状态。*/String path = ZookeeperPathUtils.getDestinationServerRunning(destination); /*对destination对应的running节点进行监听，一旦发生了变化，则说明可能其他处理相同destination的canal server可能出现了异常，此时需要尝试自己进入running状态。*/zkClient.subscribeDataChanges(path, dataListener);
```

上述只是监听代码，之后尝试调用initRunning方法通过HA的方式来启动CanalInstance。

```java
private void initRunning() {    if (!isStart()) {        return;    }    //构建临时节点的路径：/otter/canal/destinations/{0}/running，其中占位符{0}会被destination替换    String path = ZookeeperPathUtils.getDestinationServerRunning(destination);    // 序列化    //构建临时节点的数据，标记当前destination由哪一个canal server处理    byte[] bytes = JsonUtils.marshalToByte(serverData);    try {        mutex.set(false);        //尝试创建临时节点。如果节点已经存在，说明是其他的canal server已经启动了这个canal instance。        //此时会抛出ZkNodeExistsException，进入catch代码块。        zkClient.create(path, bytes, CreateMode.EPHEMERAL);        activeData = serverData;        processActiveEnter();//如果创建成功，触发一下事件，内部调用ServerRunningListener的processActiveEnter方法        mutex.set(true);    } catch (ZkNodeExistsException e) {      //创建节点失败，则根据path从zk中获取当前是哪一个canal server创建了当前canal instance的相关信息。      //第二个参数true，表示的是，如果这个path不存在，则返回null。        bytes = zkClient.readData(path, true);        if (bytes == null) {// 如果不存在节点，立即尝试一次                        initRunning();        } else {        //如果的确存在，则将创建该canal instance实例信息存入activeData中。            activeData = JsonUtils.unmarshalFromByte(bytes, ServerRunningData.class);        }    } catch (ZkNoNodeException e) {//如果/otter/canal/destinations/{0}/节点不存在，进行创建其中占位符{0}会被destination替换        zkClient.createPersistent(ZookeeperPathUtils.getDestinationPath(destination), true);        // 尝试创建父节点                initRunning();    }}
```

可以看到，initRunning方法内部只有在尝试在zk中创建节点成功后，才会去调用listener的processActiveEnter方法来真正启动destination对应的canal instance，这是canal HA方式启动的核心。canal官方文档中介绍了`CanalServer `HA机制启动的流程，如下：

![Image.png](http://static.tianshouzhi.com/ueditor/upload/image/20171212/1513091097416067990.png)

事实上，这个说明的前两步，都是在initRunning方法中实现的。从上面的代码中，我们可以看出，在HA机启动的情况下，initRunning方法不一定能走到processActiveEnter()方法，因为创建临时节点可能会出错。

此外，根据官方文档说明，如果出错，那么当前canal instance则进入standBy状态。也就是另外一个canal instance出现异常时，当前canal instance顶上去。那么相关源码在什么地方呢？在HA方式启动最开始的2行代码的监听逻辑中：

```java
String path = ZookeeperPathUtils.getDestinationServerRunning(destination);zkClient.subscribeDataChanges(path, dataListener);
```

其中dataListener类型是`IZkDataListener`，这是zkclient客户端提供的接口，定义如下：

```java
public interface IZkDataListener {    public void handleDataChange(String dataPath, Object data) throws Exception;    public void handleDataDeleted(String dataPath) throws Exception;}
```

当zk节点中的数据发生变更时，会自动回调这两个方法，很明显，一个是用于处理节点数据发生变化，一个是用于处理节点数据被删除。

而dataListener是在ServerRunningMonitor的构造方法中初始化的，如下：

```java
public ServerRunningMonitor(){    // 创建父节点    dataListener = new IZkDataListener() {        //！！！目前看来，好像并没有存在修改running节点数据的代码，为什么这个方法不是空实现？        public void handleDataChange(String dataPath, Object data) throws Exception {            MDC.put("destination", destination);            ServerRunningData runningData = JsonUtils.unmarshalFromByte((byte[]) data, ServerRunningData.class);            if (!isMine(runningData.getAddress())) {                mutex.set(false);            }             if (!runningData.isActive() && isMine(runningData.getAddress())) { // 说明出现了主动释放的操作，并且本机之前是active                                                 release = true;                releaseRunning();// 彻底释放mainstem            }             activeData = (ServerRunningData) runningData;        }        //当其他canal instance出现异常，临时节点数据被删除时，会自动回调这个方法，此时当前canal instance要顶上去        public void handleDataDeleted(String dataPath) throws Exception {            MDC.put("destination", destination);            mutex.set(false);            if (!release && activeData != null && isMine(activeData.getAddress())) {                // 如果上一次active的状态就是本机，则即时触发一下active抢占                                initRunning();            } else {                // 否则就是等待delayTime，避免因网络瞬端或者zk异常，导致出现频繁的切换操作                                delayExector.schedule(new Runnable() {                     public void run() {                        initRunning();//尝试自己进入running状态                    }                }, delayTime, TimeUnit.SECONDS);            }        }     }; }
```

那么现在问题来了？ServerRunningMonitor的start方法又是在哪里被调用的， 这个方法被调用了，才能真正的启动canal instance。这部分代码我们放到后面的CanalController中的start方法进行讲解。

下面分析最后一部分代码，autoScan机制相关代码。

## 3.5 autoScan机制相关代码

关于autoscan，官方文档有以下介绍：

![Image.png](http://static.tianshouzhi.com/ueditor/upload/image/20171212/1513091028721078148.png)

结合autoscan机制的相关源码：

```java
//   autoScan = BooleanUtils.toBoolean(getProperty(properties, CanalConstants.CANAL_AUTO_SCAN));        if (autoScan) {            defaultAction = new InstanceAction() {//....};             instanceConfigMonitors = //....        }
```

可以看到，autoScan是否需要自动扫描的开关，只有当autoScan为true时，才会初始化defaultAction字段和instanceConfigMonitors字段。其中：

其中：

​    **defaultAction：**其作用是如果配置发生了变更，默认应该采取什么样的操作。其实现了`InstanceAction`接口定义的三个抽象方法：start、stop和reload。当新增一个destination配置时，需要调用start方法来启动；当移除一个destination配置时，需要调用stop方法来停止；当某个destination配置发生变更时，需要调用reload方法来进行重启。

​    **instanceConfigMonitors：**类型为Map<InstanceMode, InstanceConfigMonitor>。defaultAction字段只是定义了配置发生变化默认应该采取的操作，那么总该有一个类来监听配置是否发生了变化，这就是InstanceConfigMonitor的作用。官方文档中，只提到了对canal.conf.dir配置项指定的目录的监听，这指的是通过spring方式加载配置。显然的，通过manager方式加载配置，配置中心的内容也是可能发生变化的，也需要进行监听。此时可以理解为什么instanceConfigMonitors的类型是一个Map，key为InstanceMode，就是为了对这两种方式的配置加载方式都进行监听。

defaultAction字段初始化源码如下所示：

```java
defaultAction = new InstanceAction() {     public void start(String destination) {        InstanceConfig config = instanceConfigs.get(destination);        if (config == null) {            // 重新读取一下instance config            config = parseInstanceConfig(properties, destination);            instanceConfigs.put(destination, config);        }         if (!embededCanalServer.isStart(destination)) {            // HA机制启动            ServerRunningMonitor runningMonitor = ServerRunningMonitors.getRunningMonitor(destination);            if (!config.getLazy() && !runningMonitor.isStart()) {                runningMonitor.start();            }        }    }     public void stop(String destination) {        // 此处的stop，代表强制退出，非HA机制，所以需要退出HA的monitor和配置信息        InstanceConfig config = instanceConfigs.remove(destination);        if (config != null) {            embededCanalServer.stop(destination);            ServerRunningMonitor runningMonitor = ServerRunningMonitors.getRunningMonitor(destination);            if (runningMonitor.isStart()) {                runningMonitor.stop();            }        }    }     public void reload(String destination) {        // 目前任何配置变化，直接重启，简单处理        stop(destination);        start(destination);    }};
```

instanceConfigMonitors字段初始化源码如下所示：

```java
instanceConfigMonitors = MigrateMap.makeComputingMap(new Function<InstanceMode, InstanceConfigMonitor>() {       public InstanceConfigMonitor apply(InstanceMode mode) {           int scanInterval = Integer.valueOf(getProperty(properties, CanalConstants.CANAL_AUTO_SCAN_INTERVAL));           if (mode.isSpring()) {//如果加载方式是spring，返回SpringInstanceConfigMonitor               SpringInstanceConfigMonitor monitor = new SpringInstanceConfigMonitor();               monitor.setScanIntervalInSecond(scanInterval);               monitor.setDefaultAction(defaultAction);               // 设置conf目录，默认是user.dir + conf目录组成               String rootDir = getProperty(properties, CanalConstants.CANAL_CONF_DIR);               if (StringUtils.isEmpty(rootDir)) {                   rootDir = "../conf";               }               if (StringUtils.equals("otter-canal", System.getProperty("appName"))) {                   monitor.setRootConf(rootDir);               } else {                   // eclipse debug模式                   monitor.setRootConf("src/main/resources/");               }               return monitor;           } else if (mode.isManager()) {//如果加载方式是manager，返回ManagerInstanceConfigMonitor               return new ManagerInstanceConfigMonitor();           } else {               throw new UnsupportedOperationException("unknow mode :" + mode + " for monitor");           }       }   });
```

可以看到instanceConfigMonitors也是根据mode属性，来采取不同的监控实现类`SpringInstanceConfigMonitor `或者`ManagerInstanceConfigMonitor`，二者都实现了`InstanceConfigMonitor`接口。

```java
public interface InstanceConfigMonitor extends CanalLifeCycle {    void register(String destination, InstanceAction action);    void unregister(String destination);}
```

当需要对一个destination进行监听时，调用register方法

当取消对一个destination监听时，调用unregister方法。

事实上，unregister方法在canal 内部并没有有任何地方被调用，也就是说，某个destination如果开启了autoScan=true，那么你是无法在运行时停止对其进行监控的。如果要停止，你可以选择将对应的目录删除。

InstanceConfigMonitor本身并不知道哪些canal instance需要进行监控，因为不同的canal instance，有的可能设置autoScan为true，另外一些可能设置为false。

在CanalConroller的start方法中，对于autoScan为true的destination，会调用InstanceConfigMonitor的register方法进行注册，此时InstanceConfigMonitor才会真正的对这个destination配置进行扫描监听。对于那些autoScan为false的destination，则不会进行监听。

目前SpringInstanceConfigMonitor对这两个方法都进行了实现，而ManagerInstanceConfigMonitor目前对这两个方法实现的都是空，需要开发者自己来实现。

在实现ManagerInstanceConfigMonitor时，可以参考SpringInstanceConfigMonitor。

此处不打算再继续进行分析SpringInstanceConfigMonitor的源码，因为逻辑很简单，感兴趣的读者可以自行查看SpringInstanceConfigMonitor 的scan方法，内部在什么情况下会回调defualtAction的start、stop、reload方法 。

# 4 CanalController的start方法

而ServerRunningMonitor的start方法，是在CanalController中的start方法中被调用的，CanalController中的start方法是在CanalLauncher中被调用的。

com.alibaba.otter.canal.deployer.CanalController#start

```java
public void start() throws Throwable {        logger.info("## start the canal server[{}:{}]", ip, port);        // 创建整个canal的工作节点 :/otter/canal/cluster/{0}        final String path = ZookeeperPathUtils.getCanalClusterNode(ip + ":" + port);        initCid(path);        if (zkclientx != null) {            this.zkclientx.subscribeStateChanges(new IZkStateListener() {                public void handleStateChanged(KeeperState state) throws Exception {                }                public void handleNewSession() throws Exception {                    initCid(path);                }            });        }        // 优先启动embeded服务        embededCanalServer.start();        //启动不是lazy模式的CanalInstance，通过迭代instanceConfigs，根据destination获取对应的ServerRunningMonitor，然后逐一启动        for (Map.Entry<String, InstanceConfig> entry : instanceConfigs.entrySet()) {            final String destination = entry.getKey();            InstanceConfig config = entry.getValue();            // 如果destination对应的CanalInstance没有启动，则进行启动            if (!embededCanalServer.isStart(destination)) {                ServerRunningMonitor runningMonitor = ServerRunningMonitors.getRunningMonitor(destination);                //如果不是lazy，lazy模式需要等到第一次有客户端请求才会启动                if (!config.getLazy() && !runningMonitor.isStart()) {                    runningMonitor.start();                }            }            if (autoScan) {                instanceConfigMonitors.get(config.getMode()).register(destination, defaultAction);            }        }        if (autoScan) {//启动配置文件自动检测机制            instanceConfigMonitors.get(globalInstanceConfig.getMode()).start();            for (InstanceConfigMonitor monitor : instanceConfigMonitors.values()) {                if (!monitor.isStart()) {                    monitor.start();//启动monitor                }            }        }        // 启动网络接口，监听客户端请求        canalServer.start();    }
```

# 5 总结

deployer模块的主要作用：

1、读取canal.properties，确定canal instance的配置加载方式

2、确定canal instance的启动方式：独立启动或者集群方式启动

3、监听canal instance的配置的变化，动态停止、启动或新增

4、启动canal server，监听客户端请求