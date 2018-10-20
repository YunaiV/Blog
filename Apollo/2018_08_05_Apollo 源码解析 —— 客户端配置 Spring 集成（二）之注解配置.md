title: Apollo 源码解析 —— 客户端配置 Spring 集成（二）之注解配置
date: 2018-08-05
tags:
categories: Apollo
permalink: Apollo/client-config-spring-2

-------

摘要: 原创出处 http://www.iocoder.cn/Apollo/client-config-spring-2/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Apollo/client-config-spring-2/)
- [2. @EnableApolloConfig](http://www.iocoder.cn/Apollo/client-config-spring-2/)
- [3. ApolloConfigRegistrar](http://www.iocoder.cn/Apollo/client-config-spring-2/)
- [4. 注解](http://www.iocoder.cn/Apollo/client-config-spring-2/)
  - [4.1 @ApolloJsonValue](http://www.iocoder.cn/Apollo/client-config-spring-2/)
  - [4.2 @ApolloConfig](http://www.iocoder.cn/Apollo/client-config-spring-2/)
  - [4.3 @ApolloConfigChangeListener](http://www.iocoder.cn/Apollo/client-config-spring-2/)
- [5. 处理器](http://www.iocoder.cn/Apollo/client-config-spring-2/)
  - [5.1 ApolloJsonValueProcessor](http://www.iocoder.cn/Apollo/client-config-spring-2/)
  - [5.2 ApolloAnnotationProcessor](http://www.iocoder.cn/Apollo/client-config-spring-2/)
- [666. 彩蛋](http://www.iocoder.cn/Apollo/client-config-spring-2/)

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

**本文分享 Spring 注解 + Java Config 配置的集成**，包括两方面：

* Apollo Config 集成到 Spring PropertySource 体系中。
* **自动更新** Spring Placeholder Values ，参见 [PR #972](https://github.com/ctripcorp/apollo/pull/972) 。

# 2. @EnableApolloConfig

`com.ctrip.framework.apollo.spring.annotation.@EnableApolloConfig` **注解**，可以使用它声明使用的 Apollo Namespace ，和 Apollo XML 配置的 `<apollo:config />` 等价。

**1. 代码如下**：

```Java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(ApolloConfigRegistrar.class)
public @interface EnableApolloConfig {

    /**
     * Apollo namespaces to inject configuration into Spring Property Sources.
     *
     * @return Namespace 名字的集合
     */
    String[] value() default {ConfigConsts.NAMESPACE_APPLICATION};

    /**
     * The order of the apollo config, default is {@link Ordered#LOWEST_PRECEDENCE}, which is Integer.MAX_VALUE.
     * If there are properties with the same name in different apollo configs, the apollo config with smaller order wins.
     *
     * @return 优先级
     */
    int order() default Ordered.LOWEST_PRECEDENCE;

}
```

* `value` 属性，Namespace 名字的集合，默认为 `"application"` 。
* `order` 属性，优先级，值越小，优先级越高。
* `@Import(Class<?>[])` 注解，引用 ApolloConfigRegistrar 类。详细解析，见 [「3. ApolloConfigRegistrar」](#) 。

**2. 例子如下**：

```Java
@Configuration
@EnableApolloConfig({"someNamespace","anotherNamespace"})
public class AppConfig {
}
```

* 可声明**多个**。

# 3. ApolloConfigRegistrar

`com.ctrip.framework.apollo.spring.annotation.ApolloConfigRegistrar` ，实现 ImportBeanDefinitionRegistrar 接口，Apollo Spring Java Config 注册器。代码如下：

> ImportBeanDefinitionRegistrar 的详解解析，推荐阅读文章 [《动态注册bean，Spring官方套路：使用ImportBeanDefinitionRegistrar》](https://zhuanlan.zhihu.com/p/30123517) 。

```Java
  1: public class ApolloConfigRegistrar implements ImportBeanDefinitionRegistrar {
  2: 
  3:     @Override
  4:     public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
  5:         // 解析 @EnableApolloConfig 注解
  6:         AnnotationAttributes attributes = AnnotationAttributes.fromMap(importingClassMetadata.getAnnotationAttributes(EnableApolloConfig.class.getName()));
  7:         String[] namespaces = attributes.getStringArray("value");
  8:         int order = attributes.getNumber("order");
  9:         // 添加到 PropertySourcesProcessor 中
 10:         PropertySourcesProcessor.addNamespaces(Lists.newArrayList(namespaces), order);
 11: 
 12:         // 注册 PropertySourcesPlaceholderConfigurer 到 BeanDefinitionRegistry 中，替换 PlaceHolder 为对应的属性值，参考文章 https://leokongwq.github.io/2016/12/28/spring-PropertyPlaceholderConfigurer.html
 13:         BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, PropertySourcesPlaceholderConfigurer.class.getName(), PropertySourcesPlaceholderConfigurer.class);
 14:         //【差异】注册 PropertySourcesProcessor 到 BeanDefinitionRegistry 中，因为可能存在 XML 配置的 Bean ，用于 PlaceHolder 自动更新机制
 15:         BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, PropertySourcesProcessor.class.getName(), PropertySourcesProcessor.class);
 16:         // 注册 ApolloAnnotationProcessor 到 BeanDefinitionRegistry 中，解析 @ApolloConfig 和 @ApolloConfigChangeListener 注解。
 17:         BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, ApolloAnnotationProcessor.class.getName(), ApolloAnnotationProcessor.class);
 18:         // 注册 SpringValueProcessor 到 BeanDefinitionRegistry 中，用于 PlaceHolder 自动更新机制
 19:         BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, SpringValueProcessor.class.getName(), SpringValueProcessor.class);
 20:         //【差异】注册 SpringValueDefinitionProcessor 到 BeanDefinitionRegistry 中，因为可能存在 XML 配置的 Bean ，用于 PlaceHolder 自动更新机制
 21:         BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, SpringValueDefinitionProcessor.class.getName(), SpringValueDefinitionProcessor.class);
 22:         // 注册 ApolloJsonValueProcessor 到 BeanDefinitionRegistry 中，解析 @ApolloJsonValue 注解。
 23:         BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, ApolloJsonValueProcessor.class.getName(), ApolloJsonValueProcessor.class);
 24:     }
 25: 
 26: }
```

* 第 5 至 10 行：解析 `@EnableApolloConfig` 注解，并调用 `PropertySourcesProcessor#addNamespaces(namespace, order)` 方法，添加到 PropertySourcesProcessor 中。通过这样的方式，`@EnableApolloConfig` 声明的 Namespace 们，就已经**集成**到 Spring ConfigurableEnvironment 中。
* 第 12 至 23 行：注册各种 **Processor** 到  BeanDefinitionRegistry 中。和 ConfigPropertySourcesProcessor **类似**，**差异**在于**多**注册了 PropertySourcesProcessor 和 SpringValueDefinitionProcessor 两个，因为可能存在 **XML 配置的 Bean** 。
* 总的来说，这个类起到的作用，和 NamespaceHandler **基本一致**。

# 4. 注解

## 4.1 @ApolloJsonValue

`com.ctrip.framework.apollo.spring.annotation.@ApolloJsonValue` 注解，将 Apollo 的**一个 JSON 格式的属性**进行注入，例如：

```Java
// Inject the json property value for type SomeObject.
// Suppose SomeObject has 2 properties, someString and someInt, then the possible config
// in Apollo is someJsonPropertyKey={"someString":"someValue", "someInt":10}.
@ApolloJsonValue("${someJsonPropertyKey:someDefaultValue}")
private SomeObject someObject;
```

* **错误**的理解：笔者一开理解错了，认为是从 Apollo 中，格式为 **JSON** Namespace 取其中一个 **KEY** 。 
* **正确**的理解：将 Apollo **任意格式**的 Namespace 的**一个 Item** 配置项，解析成对应类型的对象，注入到 `@ApolloJsonValue` 的对象中。

-------

代码如下：

```Java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.METHOD})
@Documented
public @interface ApolloJsonValue {

    /**
     * The actual value expression: e.g. "${someJsonPropertyKey:someDefaultValue}".
     */
    String value();

}
```

* 对应的处理器，见 [「5.1 ApolloJsonValueProcessor」](#) 。

## 4.2 @ApolloConfig

`com.ctrip.framework.apollo.spring.annotation.@ApolloConfig` 注解，将 Apollo Config 对象注入，例如：

```Java
// Inject the config for "someNamespace"
@ApolloConfig("someNamespace")
private Config config;
```

-------

代码如下：

```Java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@Documented
public @interface ApolloConfig {

    /**
     * Apollo namespace for the config, if not specified then default to application
     */
    String value() default ConfigConsts.NAMESPACE_APPLICATION;

}
```

* 对应的处理器，见 [「5.2 ApolloAnnotationProcessor」](#) 。

## 4.3 @ApolloConfigChangeListener

`com.ctrip.framework.apollo.spring.annotation.@ApolloConfigChangeListener` 注解，将被注解的方法，向指定的 Apollo Config 发起配置变更**监听**，例如：

```Java
// Listener on namespaces of "someNamespace" and "anotherNamespace"
@ApolloConfigChangeListener({"someNamespace","anotherNamespace"})
private void onChange(ConfigChangeEvent changeEvent) {
   // handle change event
}
```

* 虽然已经有**自动更新**机制，但是不可避免会有需要更新和初始化**关联的**属性。例如：
    * 例子一：[《Apollo应用之动态调整线上数据源(DataSource)》](http://www.kailing.pub/article/index/arcid/198.html)
    * 例子二：[ApolloRefreshConfig.java](https://github.com/ctripcorp/apollo/blob/v0.9.1/apollo-demo/src/main/java/com/ctrip/framework/apollo/demo/spring/common/refresh/ApolloRefreshConfig.java)
    * 例子三：[SpringBootApolloRefreshConfig.java](https://github.com/ctripcorp/apollo/blob/master/apollo-demo/src/main/java/com/ctrip/framework/apollo/demo/spring/springBootDemo/refresh/SpringBootApolloRefreshConfig.java)

-------

代码如下：

```Java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Documented
public @interface ApolloConfigChangeListener {

    /**
     * Apollo namespace for the config, if not specified then default to application
     */
    String[] value() default {ConfigConsts.NAMESPACE_APPLICATION};

}
```

* 对应的处理器，见 [「5.2 ApolloAnnotationProcessor」](#) 。

# 5. 处理器

## 5.1 ApolloJsonValueProcessor

`com.ctrip.framework.apollo.spring.annotation.ApolloJsonValueProcessor` ，实现 BeanFactoryAware 接口，继承 ApolloProcessor 抽象类，`@ApolloJsonValue` 注解处理器，有两个作用：

1. **注入** `@ApolloJsonValue` 注解的属性或方法，对应的值。
2. 自动更新 Spring Placeholder Values 。

### 5.1.1 构造方法

```Java
private static final Logger logger = LoggerFactory.getLogger(ApolloJsonValueProcessor.class);
private static final Gson gson = new Gson();

private final ConfigUtil configUtil;
private final PlaceholderHelper placeholderHelper;
private final SpringValueRegistry springValueRegistry;
private ConfigurableBeanFactory beanFactory;

public ApolloJsonValueProcessor() {
    configUtil = ApolloInjector.getInstance(ConfigUtil.class);
    placeholderHelper = SpringInjector.getInstance(PlaceholderHelper.class);
    springValueRegistry = SpringInjector.getInstance(SpringValueRegistry.class);
}
```

* `beanFactory` 字段，通过 `#setBeanFactory(BeanFactory)` 方法，注入该字段。代码如下：

    ```Java
    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = (ConfigurableBeanFactory) beanFactory;
    }
    ```
    * 这就是为什么实现 **BeanFactoryAware** 接口的原因。

### 5.1.2 processField

```Java
  1: @Override
  2: protected void processField(Object bean, String beanName, Field field) {
  3:     ApolloJsonValue apolloJsonValue = AnnotationUtils.getAnnotation(field, ApolloJsonValue.class);
  4:     if (apolloJsonValue == null) {
  5:         return;
  6:     }
  7:     // 获得 Placeholder 表达式
  8:     String placeholder = apolloJsonValue.value();
  9:     // 解析对应的值
 10:     Object propertyValue = placeholderHelper.resolvePropertyValue(beanFactory, beanName, placeholder);
 11:     // 忽略，非 String 值
 12:     // propertyValue will never be null, as @ApolloJsonValue will not allow that
 13:     if (!(propertyValue instanceof String)) {
 14:         return;
 15:     }
 16: 
 17:     // 设置到 Field 中
 18:     boolean accessible = field.isAccessible();
 19:     field.setAccessible(true);
 20:     ReflectionUtils.setField(field, bean, parseJsonValue((String) propertyValue, field.getGenericType()));
 21:     field.setAccessible(accessible);
 22: 
 23:     // 是否开启自动更新机制
 24:     if (configUtil.isAutoUpdateInjectedSpringPropertiesEnabled()) {
 25:         // 提取 `keys` 属性们。
 26:         Set<String> keys = placeholderHelper.extractPlaceholderKeys(placeholder);
 27:         // 循环 `keys` ，创建对应的 SpringValue 对象，并添加到 `springValueRegistry` 中。
 28:         for (String key : keys) {
 29:             SpringValue springValue = new SpringValue(key, placeholder, bean, beanName, field, true);
 30:             springValueRegistry.register(key, springValue);
 31:             logger.debug("Monitoring {}", springValue);
 32:         }
 33:     }
 34: }
```

* 第 3 至 21 行：注入 `@ApolloJsonValue` 注解的属性，对应的值。
    * 第 8 行：获得 Placeholder 表达式。
    * 第 10 行：调用 `PlaceholderHelper#resolvePropertyValue(beanFactory, beanName, placeholder` 方法，**解析**属性值。
    * 第 17 至 21 行：**设置值**，到注解的 Field 中。`#parseJsonValue(value, targetType)` 方法，解析成对应值的类型( Type ) 。代码如下：

        ```Java
        private Object parseJsonValue(String json, Type targetType) {
            try {
                return gson.fromJson(json, targetType);
            } catch (Throwable ex) {
                logger.error("Parsing json '{}' to type {} failed!", json, targetType, ex);
                throw ex;
            }
        }
        ```
        * x
    
* 第 23 至 33 行：用于**自动更新** Spring Placeholder Values 机制。
    * 第 26 行：调用 `PlaceholderHelper#extractPlaceholderKeys(placeholder)` 方法，提取 `keys` 属性们。
    * 第 27 至 32 行：**循环** `keys` ，创建对应的 SpringValue 对象，并添加到 `springValueRegistry` 中。
    
### 5.1.3 processMethod
    
```Java
@Override
protected void processMethod(Object bean, String beanName, Method method) {
    ApolloJsonValue apolloJsonValue = AnnotationUtils.getAnnotation(method, ApolloJsonValue.class);
    if (apolloJsonValue == null) {
        return;
    }
    // 获得 Placeholder 表达式
    String placeHolder = apolloJsonValue.value();
    // 解析对应的值
    Object propertyValue = placeholderHelper.resolvePropertyValue(beanFactory, beanName, placeHolder);
    // 忽略，非 String 值
    // propertyValue will never be null, as @ApolloJsonValue will not allow that
    if (!(propertyValue instanceof String)) {
        return;
    }

    Type[] types = method.getGenericParameterTypes();
    Preconditions.checkArgument(types.length == 1, "Ignore @Value setter {}.{}, expecting 1 parameter, actual {} parameters", bean.getClass().getName(), method.getName(), method.getParameterTypes().length);

    // 调用 Method ，设置值
    boolean accessible = method.isAccessible();
    method.setAccessible(true);
    ReflectionUtils.invokeMethod(method, bean, parseJsonValue((String) propertyValue, types[0]));
    method.setAccessible(accessible);

    // 是否开启自动更新机制
    if (configUtil.isAutoUpdateInjectedSpringPropertiesEnabled()) {
        // 提取 `keys` 属性们。
        Set<String> keys = placeholderHelper.extractPlaceholderKeys(placeHolder);
        // 循环 `keys` ，创建对应的 SpringValue 对象，并添加到 `springValueRegistry` 中。
        for (String key : keys) {
            SpringValue springValue = new SpringValue(key, apolloJsonValue.value(), bean, beanName, method, true);
            springValueRegistry.register(key, springValue);
            logger.debug("Monitoring {}", springValue);
        }
    }
}
```
* 和 `#processField(bean, beanName, field)` 方法，类似。胖友自己看代码注释。

## 5.2 ApolloAnnotationProcessor

`com.ctrip.framework.apollo.spring.annotation.ApolloAnnotationProcessor` ，实现 BeanFactoryAware 接口，继承 ApolloProcessor 抽象类，处理 `@ApolloConfig` 和 `@ApolloConfigChangeListener` 注解处理器的初始化。


### 5.2.1 processField

```Java
@Override
protected void processField(Object bean, String beanName, Field field) {
    ApolloConfig annotation = AnnotationUtils.getAnnotation(field, ApolloConfig.class);
    if (annotation == null) {
        return;
    }

    Preconditions.checkArgument(Config.class.isAssignableFrom(field.getType()), "Invalid type: %s for field: %s, should be Config", field.getType(), field);

    // 创建 Config 对象
    String namespace = annotation.value();
    Config config = ConfigService.getConfig(namespace);

    // 设置 Config 对象，到对应的 Field
    ReflectionUtils.makeAccessible(field);
    ReflectionUtils.setField(field, bean, config);
}
```

* 处理 `@ApolloConfig` 注解，创建( *获得* )对应的 Config 对象，设置到**注解**的 Field 中。

### 5.2.2 processMethod

```Java
@Override
protected void processMethod(final Object bean, String beanName, final Method method) {
    ApolloConfigChangeListener annotation = AnnotationUtils.findAnnotation(method, ApolloConfigChangeListener.class);
    if (annotation == null) {
        return;
    }
    Class<?>[] parameterTypes = method.getParameterTypes();
    Preconditions.checkArgument(parameterTypes.length == 1, "Invalid number of parameters: %s for method: %s, should be 1", parameterTypes.length, method);
    Preconditions.checkArgument(ConfigChangeEvent.class.isAssignableFrom(parameterTypes[0]), "Invalid parameter type: %s for method: %s, should be ConfigChangeEvent", parameterTypes[0], method);

    // 创建 ConfigChangeListener 监听器。该监听器会调用被注解的方法。
    ReflectionUtils.makeAccessible(method);
    ConfigChangeListener configChangeListener = new ConfigChangeListener() {
        @Override
        public void onChange(ConfigChangeEvent changeEvent) {
            ReflectionUtils.invokeMethod(method, bean, changeEvent);
        }
    };

    // 向指定 Namespace 的 Config 对象们，注册该监听器
    String[] namespaces = annotation.value();
    for (String namespace : namespaces) {
        Config config = ConfigService.getConfig(namespace);
        config.addChangeListener(configChangeListener);
    }
}
```

* 处理 `@ApolloConfigChangeListener` 注解，创建**回调注解方法的** ConfigChangeListener 对象，并向指定 Namespace **们**的 Config 对象**们**，注册该监听器。

# 666. 彩蛋

😈 即将完结，不晓得为啥有点惆怅。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

