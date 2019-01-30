title: 【zhisheng】看透 Spring MVC 源代码分析与实践 —— Spring MVC 组件分析
date: 2018-01-03
tag: 
categories: Spring-MVC
permalink: Spring-MVC/zhisheng/Spring-MVC03/
author: zhisheng
from_url: http://www.54tianzhisheng.cn/2017/07/21/Spring-MVC03/
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484220&idx=1&sn=3d471da8b26388b0340d606fb151d08b&chksm=fa497c8dcd3ef59b077de75ead5d9f0985fc2927fa68d93f3a850fcb05a6d78177c39560e54c&scene=21#wechat_redirect

-------

摘要: 原创出处 http://www.54tianzhisheng.cn/2017/07/21/Spring-MVC03/ 「zhisheng」欢迎转载，保留摘要，谢谢！

- [第 11 章 —— 组件概览](http://www.iocoder.cn/Spring-MVC/zhisheng/)
- [小结](http://www.iocoder.cn/Spring-MVC/zhisheng/)
- [总结](http://www.iocoder.cn/Spring-MVC/zhisheng/)
- [Spring MVC 原理总结](http://www.iocoder.cn/Spring-MVC/zhisheng/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

由于星期一接到面试通知，和面试官约好了星期四面试，所以这几天没更新完这系列的文章，面完试后立马就把这个解决掉。通过这次面试，也让我懂得了很多，知道了自己的一些不足之处，后面还要继续下功夫好好的深入复习下去。这几篇文章写的我觉得还是不够仔细，感兴趣的还是建议自己去看看源码。

### 第 11 章 —— 组件概览

**HandlerMapping**

根据 request 找到对应的处理器 Handler 和 Interceptors。内部只有一个方法

```Java
HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;
```

**HandlerAdapter**

Handler 适配器，内部方法如下：

```Java
boolean supports(Object handler);//判断是否可以使用某个 Handler
ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception; //具体使用
long getLastModified(HttpServletRequest request, Object handler);//获取资源上一次修改的时间
```

**HandlerExceptionResolver**

根据异常设置 ModelAndView ，再交给 render 方法进行渲染。

```Java
ModelAndView resolveException(
			HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex)
```

**ViewResolver**

用来将 String 类型的视图名和 Locale 解析为 View 类型的视图。

```Java
View resolveViewName(String viewName, Locale locale) throws Exception;
```

它的一个实现类 BeanNameViewResolver，它重写 resolveViewName 方法如下:

```Java
public View resolveViewName(String viewName, Locale locale) throws BeansException {
		ApplicationContext context = getApplicationContext();
		//如果应用上下文没有找到视图，返回 null
		if (!context.containsBean(viewName)) {
			if (logger.isDebugEnabled()) {
				logger.debug("No matching bean found for view name '" + viewName + "'");
			}
			// Allow for ViewResolver chaining...
			return null;
		}
		//如果找到的视图类型不匹配，也返回 null
		if (!context.isTypeMatch(viewName, View.class)) {
			if (logger.isDebugEnabled()) {
				logger.debug("Found matching bean for view name '" + viewName +
						"' - to be ignored since it does not implement View");
			}
			// Since we're looking into the general ApplicationContext here,
			// let's accept this as a non-match and allow for chaining as well...
			return null;
		}
		//根据视图名称从 Spring 容器中查找 Bean，返回找到的 bean
		return context.getBean(viewName, View.class);
	}
```

**RequestToViewNameTranslator**

获取 request 中的视图名。接口里面也是只有一个方法：

```Java
String getViewName(HttpServletRequest request) throws Exception; //根据 request 查找视图名
```

**LocaleResolver**

用于从 request 解析出 Locale。

```Java
public interface LocaleResolver {
  	//从 request 解析出 Locale
	Locale resolveLocale(HttpServletRequest request);
  	//根据 request 设置  locale
	void setLocale(HttpServletRequest request, HttpServletResponse response, @Nullable Locale locale);
}
```

**ThemeResolver**

解析主题

```Java
public interface ThemeResolver {
	//通过给定的 request 查找主题名
	String resolveThemeName(HttpServletRequest request);
	//根据给定的 request 设置主题名
	void setThemeName(HttpServletRequest request, HttpServletResponse response, String themeName);
}
```

在 RequestContext.java 文件中可以获取主题：

```Java
public String getThemeMessage(String code, String defaultMessage) {
		//获取主题的信息
		return getTheme().getMessageSource().getMessage(code, null, defaultMessage, this.locale);
	}

public Theme getTheme() {
		//判断主题是否为空
		if (this.theme == null) {
			// 通过 RequestContextUtils 获取 request 中的主题名
			this.theme = RequestContextUtils.getTheme(this.request);
			if (this.theme == null) {	//如果还是为空的话
				//那就是没有有效的主题解析器和主题
				this.theme = getFallbackTheme();
			}
		}
		return this.theme;
	}
```

RequestContextUtils.getTheme() 方法：

```Java
public static Theme getTheme(HttpServletRequest request) {
		ThemeResolver themeResolver = getThemeResolver(request);
		ThemeSource themeSource = getThemeSource(request);
		if (themeResolver != null && themeSource != null) {
			String themeName = themeResolver.resolveThemeName(request);
			return themeSource.getTheme(themeName);
		}
		else {
			return null;
		}
	}
```

**MultipartResolver**

用于处理上传请求，处理方法：将普通的 request 包装成 MultipartHttpServletRequest

```Java
public interface MultipartResolver {
	//根据 request 判断是否是上传请求
	boolean isMultipart(HttpServletRequest request);
	//将 request 包装成 MultipartHttpServletRequest
	MultipartHttpServletRequest resolveMultipart(HttpServletRequest request) throws MultipartException;
	//清理上传过程中产生的临时资源
	void cleanupMultipart(MultipartHttpServletRequest request);
}
```

**FlashMapManager**

FlashMap 主要在 redirect 中传递参数，FlashMapManager 用来管理 FlashMap 的。

```Java
public interface FlashMapManager {
	//恢复参数，并将恢复过的和超时的参数从保存介质中删除
	@Nullable
	FlashMap retrieveAndUpdate(HttpServletRequest request, HttpServletResponse response);
	//将参数保存起来
	void saveOutputFlashMap(FlashMap flashMap, HttpServletRequest request, HttpServletResponse response);
}
```

### 小结

介绍 Spring MVC 中九大组件的接口、作用、内部方法实现及作用进行了简单的介绍，详细的还需大家自己去看源码。

### 总结

### Spring MVC 原理总结

本质是一个 Servlet，这个 Servlet 继承自 HttpServlet。Spring MVC 中提供了三个层次的 Servlet：HttpServletBean、FrameworkServlet 和 DispatcherServlet。他们相互继承， HttpServletBean 直接继承自 Java 的 HttpServlet。HttpServletBean 用于将 Servlet 中的 Servlet 中配置的参数设置到相应的属性中，FrameworkServlet 初始化了 Spring MVC 中所使用的 WebApplicationContext，具体处理请求的 9 大组件是在 DispatcherServlet 中初始化的，整个继承图如下：

![spring-mvc1](http://zhisheng.iocoder.cn/spring-mvc1.jpg)

# 666. 彩蛋

如果你对 Java 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)