title: Spring 5 源码解析 —— Spring 中的 bean工厂后置处理器
date: 2018-01-19
tag: 
categories: Spring
permalink: Spring/bean-factory-post-processor
author: 一叶知秋
from_url: https://muyinchen.github.io/2017/09/16/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— Spring%E4%B8%AD%E7%9A%84bean%E5%B7%A5%E5%8E%82%E5%90%8E%E7%BD%AE%E5%A4%84%E7%90%86%E5%99%A8/
wechat_url: 

-------

摘要: 原创出处 https://muyinchen.github.io/2017/09/16/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— Spring%E4%B8%AD%E7%9A%84bean%E5%B7%A5%E5%8E%82%E5%90%8E%E7%BD%AE%E5%A4%84%E7%90%86%E5%99%A8/ 「一叶知秋」欢迎转载，保留摘要，谢谢！

- [什么是Spring工厂的后置处理器？](http://www.iocoder.cn/Spring/bean-factory-post-processor)
- [一个简单的Spring bean厂后置处理器Demo](http://www.iocoder.cn/Spring/bean-factory-post-processor)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------


Spring允许我们使用bean来进行大量的操作(这也是我们面向对象思想最常用的表达方式)。我们可以决定它们在容器中是否是(单例或者是原型)唯一实例。通过bean工厂后置处理器我们还可以在`初始化`时进行一些额外的操作。

在这篇文章中，来介绍下**bean factory post processor**。在第一部分，我们来发现下这个概念背后玄机。之后我们会写一些有趣代码来让大伙更好地理解这个概念。

## 什么是Spring工厂的后置处理器？

有时我们可能需要在Spring应用程序中实现一些动态行为。举个简单的例子，假设在你的网站中，你要显示按时间来显示两个文本内容。上午的时候，你会显示“早安”。下午，显示的文字将是“下午好”。另外，你有两个日常部署，上午12点，另一个在晚上12点。需要强调的是，这个文本内容必须由一个bean来处理。我们现在有两个选择:每次部署时更改应用程序上下文文件(太麻烦了)，或者定义一个实现**org.springframework.beans.factory.config.BeanFactoryPostProcessor**接口的bean 。第二个解决方案更优雅，因为我们只需要编写一次代码，然后就可以忽视它的存在了(无须次次去修改了)。

那么，这个优雅的`BeanFactoryPostProcessor`是何方神圣？它是由bean实现的接口，它们可以修改其他bean的定义。请注意，只能修改定义，即构造函数参数，属性值。`BeanFactoryPostProcessor`bean在初始化“正常”bean之前被调用，这就是为什么它能修改元数据的原因(meta data)。调用是通过**org.springframework.context.support.AbstractApplicationContext的** **protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory)**来**实现的**:

```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
  PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
}
```

在`PostProcessorRegistrationDelegate`里面，负责bean factory后置处理器执行的方法是:

```java
/**
 * Invoke the given BeanFactoryPostProcessor beans.
 */
private static void invokeBeanFactoryPostProcessors(
		Collection<? extends BeanFactoryPostProcessor> postProcessors, ConfigurableListableBeanFactory beanFactory) {
	for (BeanFactoryPostProcessor postProcessor : postProcessors) {
		postProcessor.postProcessBeanFactory(beanFactory);
	}
}
```

如你所见，由`BeanFactoryPostProcessor`实现覆盖的主要方法是`postProcessBeanFactory`。这也是我们来自己定制bean定义的地方。我们通过在**org.springframework.beans.factory.config.BeanDefinition**对象上进行定制。关于这点我已经[在Spring5源码解析-Spring框架中的单例和原型bean](https://muyinchen.github.io/2017/09/15/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-Spring%E6%A1%86%E6%9E%B6%E4%B8%AD%E7%9A%84%E5%8D%95%E4%BE%8B%E5%92%8C%E5%8E%9F%E5%9E%8Bbean/)的文章中已经写过，它们(`BeanDefinition对象`)包含大量关于bean元数据的信息:构造函数参数，属性值或作用域。

## 一个简单的Spring bean厂后置处理器Demo

关于理论的重要部分已经都在前面进行了描述。在这部分中，我们将重点放在一个简单实用的案例上。你还记得第一部分的“早安”和“下午好”的例子？如果忘了，请回去再瞅眼的。接着，让我们尝试在代码中实现这种案例。首先，我们将在配置文件中定义一些bean:

```xml
<bean class="com.migo.bean.BeanModifier">

<bean name="welcomerBean" class="com.migo.bean.Welcomer" init-method="initWelcomer">
  <property name="welcomeText" value="Good morning"></property>
 </bean>
</bean>
```

第一个bean代表将实现`BeanFactoryPostProcessor`接口的bean。第二个bean是注入的类，它会在页面中显示欢迎的文本内容。他们是两个bean的代码:

```java
// Welcomer.java
public class Welcomer {

  private String welcomeText;

  public void initWelcomer() {
    LOGGER.debug("Welcomer is initialized");
  }

  public void setWelcomeText(String welcomeText) {
    LOGGER.debug("Setting welcomeText to: "+welcomeText);
    this.welcomeText = welcomeText;
  }

  public String getWelcomeText() {
    return this.welcomeText;
  }

  @Override
  public String toString() {
    return "Welcomer {text: "+this.welcomeText+"}";
  }

}

// BeanModifier.java
public class BeanModifier implements BeanFactoryPostProcessor {

  @Override
  public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
    try {
      Calendar calendar = Calendar.getInstance();
      if (calendar.get(Calendar.AM_PM) == Calendar.PM) {
        BeanDefinition welcomerDef = beanFactory.getBeanDefinition("welcomerBean");
        welcomerDef.getPropertyValues().add("welcomeText", "Good afternoon");
      }
    } catch (Exception e) {
        LOGGER.error("An error occurred on setting welcomeText", e);
    }
  }
}

// test method
ApplicationContext context = new FileSystemXmlApplicationContext("/home/bartosz/webapp/src/main/resources/META-INF/applicationContext.xml");

Welcomer welcomer = (Welcomer) context.getBean("welcomer");
System.out.println("Text: "+welcomer.getWelcomeText());
```

如果现在是下午，输出应该是:

```shell
Setting welcomeText to: Good afternoon
Welcomer is initialized
Text: Good afternoon
```

我们可以看到，`BeanModifier`在`Welcomer`的真正初始化之前被调用。由于重写覆盖了`postProcessBeanFactory`方法，这样我们可以通过检查日期，并为属性`welcomeText`设置正确的值。

这篇文章虽短，但它描述了一些在一些“动态”场景中我们如何以一个更效率的方式来实现的实用操作。比如，你会碰到这种，我们常常见到一个游戏会有例行维护，那么我们会发现排行榜会在例行维护后刷新，你每次登录游戏也会对你的一些属性或者积分进行刷新，其实你每次登录就是又初始化了一遍你这个bean，这样，我们就可以做很多事情了，比如为最佳用户添加一些奖励积分。通过`BeanFactoryPostProcessor`这个 bean，这种处理就可以在Java方法内自动完成，无须我们在每次部署时通过手动来完成。

# 666. 彩蛋

如果你对 Spring 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)