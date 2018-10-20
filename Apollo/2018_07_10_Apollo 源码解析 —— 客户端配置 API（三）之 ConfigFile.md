title: Apollo 源码解析 —— 客户端 API 配置（三）之 ConfigFile
date: 2018-07-05
tags:
categories: Apollo
permalink: Apollo/client-config-api-3

-------

摘要: 原创出处 http://www.iocoder.cn/Apollo/client-config-api-3/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Apollo/client-config-api-3/)
- [2. ConfigFile](http://www.iocoder.cn/Apollo/client-config-api-3/)
- [3. AbstractConfigFile](http://www.iocoder.cn/Apollo/client-config-api-3/)
  - [3.1 构造方法](http://www.iocoder.cn/Apollo/client-config-api-3/)
  - [3.2 获得内容](http://www.iocoder.cn/Apollo/client-config-api-3/)
  - [3.3 获得 Namespace 名字](http://www.iocoder.cn/Apollo/client-config-api-3/)
  - [3.4 添加配置变更监听器](http://www.iocoder.cn/Apollo/client-config-api-3/)
  - [3.5 触发配置变更监听器们](http://www.iocoder.cn/Apollo/client-config-api-3/)
  - [3.6 onRepositoryChange](http://www.iocoder.cn/Apollo/client-config-api-3/)
- [4. PropertiesConfigFile](http://www.iocoder.cn/Apollo/client-config-api-3/)
  - [4.1 构造方法](http://www.iocoder.cn/Apollo/client-config-api-3/)
  - [4.2 更新内容](http://www.iocoder.cn/Apollo/client-config-api-3/)
  - [4.3 获得内容](http://www.iocoder.cn/Apollo/client-config-api-3/)
  - [4.4 获得 Namespace 名字](http://www.iocoder.cn/Apollo/client-config-api-3/)
- [5. PlainTextConfigFile](http://www.iocoder.cn/Apollo/client-config-api-3/)
  - [5.1 XmlConfigFile](http://www.iocoder.cn/Apollo/client-config-api-3/)
  - [5.2 JsonConfigFile](http://www.iocoder.cn/Apollo/client-config-api-3/)
  - [5.3 YamlConfigFile](http://www.iocoder.cn/Apollo/client-config-api-3/)
  - [5.4 YmlConfigFile](http://www.iocoder.cn/Apollo/client-config-api-3/)
- [666. 彩蛋](http://www.iocoder.cn/Apollo/client-config-api-3/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2018_05_18.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

# 1. 概述

> 老艿艿：本系列假定胖友已经阅读过 [《Apollo 官方 wiki 文档》](https://github.com/ctripcorp/apollo/wiki/) ，特别是 [《Java 客户端使用指南》](https://github.com/ctripcorp/apollo/wiki/Apollo%E5%BC%80%E6%94%BE%E5%B9%B3%E5%8F%B0) 。

本文接 [《Apollo 源码解析 —— 客户端 API 配置（二）之一览》](http://www.iocoder.cn/Apollo/client-config-api-2/?self) 一文，分享 ConfigFile 接口，及其子类，如下图：

![ConfigFile 类图](http://www.iocoder.cn/images/Apollo/2018_07_01/03.png)

从实现上，ConfigFile 和 Config 超级类似，所以本文会写的比较简洁。

* Config 基于 **KV** 数据结构。
* ConfigFile 基于 **String** 数据结构。

# 2. ConfigFile

在  [《Apollo 源码解析 —— 客户端 API 配置（一）之一览》](http://www.iocoder.cn/Apollo/client-config-api-1/?self) 的 [「3.2 ConfigFile」](#) 中，有详细分享。

# 3. AbstractConfigFile

`com.ctrip.framework.apollo.internals.AbstractConfigFile` ，实现 ConfigFile、RepositoryChangeListener 接口，ConfigFile 抽象类，实现了 1）异步通知监听器、2）计算属性变化等等**特性**，是 AbstractConfig + DefaultConfig 的功能**子集**。

## 3.1 构造方法

```Java
private static final Logger logger = LoggerFactory.getLogger(AbstractConfigFile.class);

/**
 * ExecutorService 对象，用于配置变化时，异步通知 ConfigChangeListener 监听器们
 *
 * 静态属性，所有 Config 共享该线程池。
 */
private static ExecutorService m_executorService;

/**
 * Namespace 的名字
 */
protected String m_namespace;
/**
 * ConfigChangeListener 集合
 */
private List<ConfigFileChangeListener> m_listeners = Lists.newCopyOnWriteArrayList();

protected ConfigRepository m_configRepository;
/**
 * 配置 Properties 的缓存引用
 */
protected AtomicReference<Properties> m_configProperties;

static {
    m_executorService = Executors.newCachedThreadPool(ApolloThreadFactory.create("ConfigFile", true));
}

public AbstractConfigFile(String namespace, ConfigRepository configRepository) {
    m_configRepository = configRepository;
    m_namespace = namespace;
    m_configProperties = new AtomicReference<>();
    // 初始化
    initialize();
}

private void initialize() {
    try {
        // 初始化 m_configProperties
        m_configProperties.set(m_configRepository.getConfig());
    } catch (Throwable ex) {
        Tracer.logError(ex);
        logger.warn("Init Apollo Config File failed - namespace: {}, reason: {}.", m_namespace, ExceptionUtil.getDetailMessage(ex));
    } finally {
        //register the change listener no matter config repository is working or not
        //so that whenever config repository is recovered, config could get changed
        // 注册到 ConfigRepository 中，从而实现每次配置发生变更时，更新配置缓存 `m_configProperties` 。
        m_configRepository.addChangeListener(this);
    }
}
```

## 3.2 获得内容

交给**子类**自己实现。

## 3.3 获得 Namespace 名字

```Java
@Override
public String getNamespace() {
    return m_namespace;
}
```

## 3.4 添加配置变更监听器

```Java
@Override
public void addChangeListener(ConfigFileChangeListener listener) {
    if (!m_listeners.contains(listener)) {
        m_listeners.add(listener);
    }
}
```

## 3.5 触发配置变更监听器们

```Java
private void fireConfigChange(final ConfigFileChangeEvent changeEvent) {
    // 缓存 ConfigChangeListener 数组
    for (final ConfigFileChangeListener listener : m_listeners) {
        m_executorService.submit(new Runnable() {
            @Override
            public void run() {
                String listenerName = listener.getClass().getName();
                Transaction transaction = Tracer.newTransaction("Apollo.ConfigFileChangeListener", listenerName);
                try {
                    // 通知监听器
                    listener.onChange(changeEvent);
                    transaction.setStatus(Transaction.SUCCESS);
                } catch (Throwable ex) {
                    transaction.setStatus(ex);
                    Tracer.logError(ex);
                    logger.error("Failed to invoke config file change listener {}", listenerName, ex);
                } finally {
                    transaction.complete();
                }
            }
        });
    }
}
```

## 3.6 onRepositoryChange

`#onRepositoryChange(namespace, newProperties)` 方法，当 ConfigRepository 读取到配置发生变更时，计算配置变更集合，并通知监听器们。代码如下：

```Java
@Override
public synchronized void onRepositoryChange(String namespace, Properties newProperties) {
    // 忽略，若未变更
    if (newProperties.equals(m_configProperties.get())) {
        return;
    }
    // 读取新的 Properties 对象
    Properties newConfigProperties = new Properties();
    newConfigProperties.putAll(newProperties);

    // 获得【旧】值
    String oldValue = getContent();
    // 更新为【新】值
    update(newProperties);
    // 获得新值
    String newValue = getContent();

    // 计算变化类型
    PropertyChangeType changeType = PropertyChangeType.MODIFIED;
    if (oldValue == null) {
        changeType = PropertyChangeType.ADDED;
    } else if (newValue == null) {
        changeType = PropertyChangeType.DELETED;
    }

    // 通知监听器们
    this.fireConfigChange(new ConfigFileChangeEvent(m_namespace, oldValue, newValue, changeType));

    Tracer.logEvent("Apollo.Client.ConfigChanges", m_namespace);
}
```

* 调用 `#update(newProperties)` **抽象**方法，更新为【新】值。该方法需要子类自己去实现。抽象方法如下：

    ```Java
    protected abstract void update(Properties newProperties);
    ```

# 4. PropertiesConfigFile

`com.ctrip.framework.apollo.internals.PropertiesConfigFile` ，实现 AbstractConfigFile 抽象类，类型为 `.properties` 的 ConfigFile 实现类。

## 4.1 构造方法

```Java
private static final Logger logger = LoggerFactory.getLogger(PropertiesConfigFile.class);

/**
 * 配置字符串缓存
 */
protected AtomicReference<String> m_contentCache;

public PropertiesConfigFile(String namespace, ConfigRepository configRepository) {
    super(namespace, configRepository);
    m_contentCache = new AtomicReference<>();
}
```

* 因为 Properties 是 **KV** 数据结构，需要将**多条** KV  拼接成一个字符串，进行缓存到 `m_contentCache` 中。

## 4.2 更新内容

```Java
@Override
protected void update(Properties newProperties) {
    // 设置【新】Properties
    m_configProperties.set(newProperties);
    // 清空缓存
    m_contentCache.set(null);
}
```

## 4.3 获得内容

```Java
@Override
public String getContent() {
    // 更新到缓存
    if (m_contentCache.get() == null) {
        m_contentCache.set(doGetContent());
    }
    // 从缓存中，获得配置字符串
    return m_contentCache.get();
}

String doGetContent() {
    if (!this.hasContent()) {
        return null;
    }
    try {
        return PropertiesUtil.toString(m_configProperties.get()); // 拼接 KV 属性，成字符串
    } catch (Throwable ex) {
        ApolloConfigException exception =  new ApolloConfigException(String.format("Parse properties file content failed for namespace: %s, cause: %s", m_namespace, ExceptionUtil.getDetailMessage(ex)));
        Tracer.logError(exception);
        throw exception;
    }
}

@Override
public boolean hasContent() {
    return m_configProperties.get() != null && !m_configProperties.get().isEmpty();
}
```

* 调用 `PropertiesUtil#toString(Properties)` 方法，将 Properties 拼接成字符串。代码如下：

    ```Java
    /**
     * Transform the properties to string format
     *
     * @param properties the properties object
     * @return the string containing the properties
     * @throws IOException
     */
    public static String toString(Properties properties) throws IOException {
        StringWriter writer = new StringWriter();
        properties.store(writer, null);
        StringBuffer stringBuffer = writer.getBuffer();
        // 去除头部自动添加的注释
        filterPropertiesComment(stringBuffer);
        return stringBuffer.toString();
    }
    
    /**
     * filter out the first comment line
     *
     * @param stringBuffer the string buffer
     * @return true if filtered successfully, false otherwise
     */
    static boolean filterPropertiesComment(StringBuffer stringBuffer) {
        //check whether has comment in the first line
        if (stringBuffer.charAt(0) != '#') {
            return false;
        }
        int commentLineIndex = stringBuffer.indexOf("\n");
        if (commentLineIndex == -1) {
            return false;
        }
        stringBuffer.delete(0, commentLineIndex + 1);
        return true;
    }
    ```
    * 因为 `Properties#store(writer, null)` 方法，会自动在**首行**，添加**注释时间**。代码如下：

        ```Java
        private void store0(BufferedWriter bw, String comments, boolean escUnicode)
            throws IOException
        {
            if (comments != null) {
                writeComments(bw, comments);
            }
            bw.write("#" + new Date().toString()); // 自动在**首行**，添加**注释时间**。
            bw.newLine();
            synchronized (this) {
                for (Enumeration<?> e = keys(); e.hasMoreElements();) {
                    String key = (String)e.nextElement();
                    String val = (String)get(key);
                    key = saveConvert(key, true, escUnicode);
                    /* No need to escape embedded and trailing spaces for value, hence
                     * pass false to flag.
                     */
                    val = saveConvert(val, false, escUnicode);
                    bw.write(key + "=" + val);
                    bw.newLine();
                }
            }
            bw.flush();
        }
        ```
        * 从实现代码，我们可以看出，拼接的字符串，每一行一个 **KV** 属性。例子如下：

            ```Java
            key2=value2
            key1=value1
            ```
            * x

## 4.4 获得 Namespace 名字

```Java
@Override
public ConfigFileFormat getConfigFileFormat() {
    return ConfigFileFormat.Properties;
}
```

# 5. PlainTextConfigFile

`com.ctrip.framework.apollo.internals.PlainTextConfigFile` ，实现 AbstractConfigFile 抽象类，**纯文本** ConfigFile 抽象类，例如 `xml` `yaml` 等等。

**更新内容**

```Java
@Override
protected void update(Properties newProperties) {
    m_configProperties.set(newProperties);
}
```

**获得内容**

```Java
@Override
public String getContent() {
    if (!this.hasContent()) {
        return null;
    }
    return m_configProperties.get().getProperty(ConfigConsts.CONFIG_FILE_CONTENT_KEY);
}

@Override
public boolean hasContent() {
    if (m_configProperties.get() == null) {
        return false;
    }
    return m_configProperties.get().containsKey(ConfigConsts.CONFIG_FILE_CONTENT_KEY);
}
```

* 直接从 `"content"` 配置项，获得配置文本。这也是为什么类名以 **PlainText** 开头的原因。

-------

🙂 PlainTextConfigFile 的子类，代码基本一致，差别在于 `#getConfigFileFormat()` **实现**方法，返回不同的 ConfigFileFormat 。

## 5.1 XmlConfigFile

`com.ctrip.framework.apollo.internals.XmlConfigFile` ，实现 PlainTextConfigFile 抽象类，类型为 `.xml` 的 ConfigFile 实现类。代码如下：

```Java
public class XmlConfigFile extends PlainTextConfigFile {

    public XmlConfigFile(String namespace, ConfigRepository configRepository) {
        super(namespace, configRepository);
    }

    @Override
    public ConfigFileFormat getConfigFileFormat() {
        return ConfigFileFormat.XML;
    }

}
```

## 5.2 JsonConfigFile

`com.ctrip.framework.apollo.internals.JsonConfigFile` ，实现 PlainTextConfigFile 抽象类，类型为 `.json` 的 ConfigFile 实现类。代码如下：

```Java
public class JsonConfigFile extends PlainTextConfigFile {

    public JsonConfigFile(String namespace,
                          ConfigRepository configRepository) {
        super(namespace, configRepository);
    }

    @Override
    public ConfigFileFormat getConfigFileFormat() {
        return ConfigFileFormat.JSON;
    }

}
```

## 5.3 YamlConfigFile

`com.ctrip.framework.apollo.internals.YamlConfigFile` ，实现 PlainTextConfigFile 抽象类，类型为 `.yaml` 的 ConfigFile 实现类。代码如下：

```Java
public class YamlConfigFile extends PlainTextConfigFile {

    public YamlConfigFile(String namespace, ConfigRepository configRepository) {
        super(namespace, configRepository);
    }

    @Override
    public ConfigFileFormat getConfigFileFormat() {
        return ConfigFileFormat.YAML;
    }

}
```

## 5.4 YmlConfigFile

`com.ctrip.framework.apollo.internals.YmlConfigFile` ，实现 PlainTextConfigFile 抽象类，类型为 `.yaml` 的 ConfigFile 实现类。代码如下：

```Java
public class YmlConfigFile extends PlainTextConfigFile {

    public YmlConfigFile(String namespace, ConfigRepository configRepository) {
        super(namespace, configRepository);
    }

    @Override
    public ConfigFileFormat getConfigFileFormat() {
        return ConfigFileFormat.YML;
    }
    
}
```

# 666. 彩蛋

恩。  
嗯嗯。  
嗯嗯嗯。  
水更，哈哈哈。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)


