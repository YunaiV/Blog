title: Apollo 源码解析 —— 客户端配置 Spring 集成（三）之外部化配置
date: 2018-08-10
tags:
categories: Apollo
permalink: Apollo/client-config-spring-3

-------

摘要: 原创出处 http://www.iocoder.cn/Apollo/client-config-spring-3/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Apollo/client-config-spring-3/)
- [2. spring.factories](http://www.iocoder.cn/Apollo/client-config-spring-3/)
- [3. ApolloAutoConfiguration](http://www.iocoder.cn/Apollo/client-config-spring-3/)
- [4. ApolloApplicationContextInitializer](http://www.iocoder.cn/Apollo/client-config-spring-3/)
- [666. 彩蛋](http://www.iocoder.cn/Apollo/client-config-spring-3/)

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

> 老艿艿：本系列假定胖友已经阅读过 [《Apollo 官方 wiki 文档》](https://github.com/ctripcorp/apollo/wiki/) ，特别是 [《Spring 整合方式》](https://github.com/ctripcorp/apollo/wiki/Java%E5%AE%A2%E6%88%B7%E7%AB%AF%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97#32-spring%E6%95%B4%E5%90%88%E6%96%B9%E5%BC%8F) 。
> 
> 😁 因为 Spring 仅仅处于入门水平，所以可能一些地方，表述的灰常业余。


**本文分享 Spring 外部化配置的集成**。我们先看看官方文档的说明：

> 使用上述两种方式的配置形式( *基于 XML 的配置和基于Java的配置* )后，Apollo 会在 Spring 的 **postProcessBeanFactory** 阶段注入配置到 Spring 的 Environment中，早于 bean 的初始化阶段，所以对于普通的 bean 注入配置场景已经能很好的满足。
> 
> 不过 Spring Boot 有一些场景需要配置在更早的阶段注入，比如使用 `@ConditionalOnProperty` 的场景或者是有一些 `spring-boot-starter` 在启动阶段就需要读取配置做一些事情（ 如 [`spring-boot-starter-dubbo`](https://github.com/teaey/spring-boot-starter-dubbo)  )，所以对于 Spring Boot 环境建议通过以下方式来接入 Apollo ( 需要0.10.0及以上版本 ）。
> 使用方式很简单，只需要在 `application.properties/bootstrap.properties` 中按照如下样例配置即可。
> 
> 1、在 bootstrap 阶段注入默认 `"application"` namespace 的配置示例：
> 
> ```
> # will inject 'application' namespace in bootstrap phase
> apollo.bootstrap.enabled = true
> ```
> 
> 2、在 bootstrap 阶段注入**非默认** `"application"` namespace 或多个 namespace 的配置示例
> 
> ```
> apollo.bootstrap.enabled = true
> # will inject 'application' and 'FX.apollo' namespaces in bootstrap phase
> apollo.bootstrap.namespaces = application,FX.apollo
> ```   

下面，让我们来看看具体的代码实现。

# 2. spring.factories

Apollo 在 `apollo-client` 的 [`META-INF/spring.factories`](https://github.com/YunaiV/apollo/blob/2907eebd618825f32b8e27586cb521bcd0221a7e/apollo-client/src/main/resources/META-INF/spring.factories) 定义如下：

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.ctrip.framework.apollo.spring.boot.ApolloAutoConfiguration
org.springframework.context.ApplicationContextInitializer=\
com.ctrip.framework.apollo.spring.boot.ApolloApplicationContextInitializer
```

* 这个 `spring.factories` 里面配置的那些类，主要作用是告诉 Spring Boot 这个 **starter** 所需要加载的那些 xxxAutoConfiguration 和 xxxContextInitializer 类，也就是你真正的要自动注册的那些 bean 或功能。然后，我们实现一个 `spring.factories` 指定的类即可。
* 此处配置了 **ApolloAutoConfiguration** 和 **ApolloApplicationContextInitializer** 类。

# 3. ApolloAutoConfiguration

`com.ctrip.framework.apollo.spring.boot.ApolloAutoConfiguration` ，自动注入 **ConfigPropertySourcesProcessor** bean 对象，当**不存在** **PropertySourcesProcessor** 时，以实现 Apollo 配置的自动加载。代码如下：

```Java
@Configuration
@ConditionalOnProperty(PropertySourcesConstants.APOLLO_BOOTSTRAP_ENABLED)
@ConditionalOnMissingBean(PropertySourcesProcessor.class) // 缺失 PropertySourcesProcessor 时
public class ApolloAutoConfiguration {

    @Bean
    public ConfigPropertySourcesProcessor configPropertySourcesProcessor() {
        return new ConfigPropertySourcesProcessor(); // 注入 ConfigPropertySourcesProcessor bean 对象
    }

}
```

# 4. ApolloApplicationContextInitializer

`com.ctrip.framework.apollo.spring.boot.ApolloApplicationContextInitializer` ，实现 ApplicationContextInitializer 接口，在 Spring Boot 启动阶段( **bootstrap phase** )，注入**配置**的 Apollo Config 对象们。

> 实现代码上，和 PropertySourcesProcessor 一样实现了注入**配置**的 Apollo Config 对象们，差别在于处于 Spring 的不同阶段。

代码如下：

```Java
  1: public class ApolloApplicationContextInitializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {
  2: 
  3:     private static final Logger logger = LoggerFactory.getLogger(ApolloApplicationContextInitializer.class);
  4:     private static final Splitter NAMESPACE_SPLITTER = Splitter.on(",").omitEmptyStrings().trimResults();
  5: 
  6:     private final ConfigPropertySourceFactory configPropertySourceFactory = SpringInjector.getInstance(ConfigPropertySourceFactory.class);
  7: 
  8:     @Override
  9:     public void initialize(ConfigurableApplicationContext context) {
 10:         ConfigurableEnvironment environment = context.getEnvironment();
 11:         // 获得 "apollo.bootstrap.enabled" 配置项
 12:         String enabled = environment.getProperty(PropertySourcesConstants.APOLLO_BOOTSTRAP_ENABLED, "false");
 13:         // 忽略，若未开启
 14:         if (!Boolean.valueOf(enabled)) {
 15:             logger.debug("Apollo bootstrap config is not enabled for context {}, see property: ${{}}", context, PropertySourcesConstants.APOLLO_BOOTSTRAP_ENABLED);
 16:             return;
 17:         }
 18:         logger.debug("Apollo bootstrap config is enabled for context {}", context);
 19: 
 20:         // 忽略，若已经有 APOLLO_BOOTSTRAP_PROPERTY_SOURCE_NAME 的 PropertySource
 21:         if (environment.getPropertySources().contains(PropertySourcesConstants.APOLLO_BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
 22:             // already initialized
 23:             return;
 24:         }
 25: 
 26:         // 获得 "apollo.bootstrap.namespaces" 配置项
 27:         String namespaces = environment.getProperty(PropertySourcesConstants.APOLLO_BOOTSTRAP_NAMESPACES, ConfigConsts.NAMESPACE_APPLICATION);
 28:         logger.debug("Apollo bootstrap namespaces: {}", namespaces);
 29:         List<String> namespaceList = NAMESPACE_SPLITTER.splitToList(namespaces);
 30: 
 31:         // 按照优先级，顺序遍历 Namespace
 32:         CompositePropertySource composite = new CompositePropertySource(PropertySourcesConstants.APOLLO_BOOTSTRAP_PROPERTY_SOURCE_NAME);
 33:         for (String namespace : namespaceList) {
 34:             // 创建 Apollo Config 对象
 35:             Config config = ConfigService.getConfig(namespace);
 36:             // 创建 Namespace 对应的 ConfigPropertySource 对象
 37:             // 添加到 `composite` 中。
 38:             composite.addPropertySource(configPropertySourceFactory.getConfigPropertySource(namespace, config));
 39:         }
 40: 
 41:         // 添加到 `environment` 中，且优先级最高
 42:         environment.getPropertySources().addFirst(composite);
 43:     }
 44: 
 45: }
```

* 第 12 行：获得 `"apollo.bootstrap.enabled"` 配置项。
* 第 13 至 18 行：**忽略**，若未配置开启。
* 第 20 至 24 行：**忽略**，若已经有 *APOLLO_BOOTSTRAP_PROPERTY_SOURCE_NAME* 的 PropertySource 对象。
* 第 26 至 29 行：获得 `"apollo.bootstrap.namespaces"` 配置项。
* 第 31 至 33 行：按照顺序遍历 Namespace 。
    * 第 35 行：调用 `ConfigService#getConfig(namespace)` 方法，获得( *创建* ) Apollo Config 对象。这个地方，非常关键。
    * 第 38 行：调用 `ConfigPropertySourceFactory#getConfigPropertySource(namespace, config)` 方法，创建 Namespace **对应的** ConfigPropertySource 对象。
    * 第 38 行：调用 `CompositePropertySource#addPropertySource(PropertySource)` 方法，添加到 `composite` 中。通过这样的方式，形成**顺序的优先级**。
    * 第 42 行：添加 `composite` 到 `environment` 中。这样，我们从 `environment` 里，**且优先级最高**。

# 666. 彩蛋

完结，撒花。但是，好惆怅！！！

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

