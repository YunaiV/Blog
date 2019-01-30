title: 面试问烂的 Spring MVC 过程
date: 2018-11-10
tags:
categories: 精进
permalink: Fight/Interview-poorly-asked-SpringMVC-process
author: 莫那一鲁道
from_url: https://www.jianshu.com/p/e18fd44964eb
wechat_url:

-------

摘要: 原创出处 https://www.jianshu.com/p/e18fd44964eb 「莫那一鲁道」欢迎转载，保留摘要，谢谢！

- [Spring MVC 过程](http://www.iocoder.cn/Fight/Interview-poorly-asked-SpringMVC-process/)
  - [1. 设置属性](http://www.iocoder.cn/Fight/Interview-poorly-asked-SpringMVC-process/)
  - [2. 根据 Request 请求的 URL 得到对应的 handler 执行链，其实就是拦截器和 Controller 代理对象。](http://www.iocoder.cn/Fight/Interview-poorly-asked-SpringMVC-process/)
  - [3. 得到 handler 的适配器](http://www.iocoder.cn/Fight/Interview-poorly-asked-SpringMVC-process/)
  - [4. 循环执行 handler 的 pre 拦截器](http://www.iocoder.cn/Fight/Interview-poorly-asked-SpringMVC-process/)
  - [5. 执行真正的 handler，并返回  ModelAndView(Handler 是个代理对象，可能会执行 AOP )](http://www.iocoder.cn/Fight/Interview-poorly-asked-SpringMVC-process/)
  - [6. 循环执行 handler 的 post 拦截器](http://www.iocoder.cn/Fight/Interview-poorly-asked-SpringMVC-process/)
  - [7. 根据 ModelAndView 信息得到 View 实例](http://www.iocoder.cn/Fight/Interview-poorly-asked-SpringMVC-process/)
  - [8. 渲染 View 返回](http://www.iocoder.cn/Fight/Interview-poorly-asked-SpringMVC-process/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

SpringMVC ，这个应该是国内面试必问题，网上有很多答案，其实背背就可以。但今天笔者带大家一起深入浅出源码，看看他的原理。以期让印象更加深刻，面试的时候游刃有余。

# Spring MVC 过程

先来张图：



![img](https:////upload-images.jianshu.io/upload_images/4236553-154a7dd426ad864c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/jpeg)



代码位置：`com.interface21.web.servlet.DispatcherServlet#doService`

（没错，就是 Spring 1.0 的代码，大道至简，现在的 Spring 经过 15 年的发展，已经太过臃肿，从学习角度来说，interface 21 是最好的代码，不接受反驳）

代码如下：

## 1. 设置属性

```Java
// 1. 设置属性
// Make web application context available
request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());

// Make locale resolver available
request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);

// Make theme resolver available
request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
```

## 2. 根据 Request 请求的 URL 得到对应的 handler 执行链，其实就是拦截器和 Controller 代理对象。

```Java
// 2. 找 handler 返回执行链
HandlerExecutionChain mappedHandler = getHandler(request);
```

## 3. 得到 handler 的适配器

```Java
// This will throw an exception if no adapter is found
// 3. 返回 handler 的适配器
HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
```

关于这个适配器，作用到底是啥呢？HandlerAdapter 注释写到：This interface is not intended for application developers. It is available to handlers who want to develop their own web workflow.
 译：此接口不适用于应用程序开发人员。它适用于想要开发自己的Web工作流程的处理程序。

也就说说，如果你想要在处理 handler 之前做一些操作的话，可能需要这个，即适配一下这个 handler。例如 Spring 的测试程序做的那样：

```Java
public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object delegate)
            throws IOException, ServletException {
                      // 你可能需要 doSomething.......
            ((MyHandler) delegate).doSomething(request);
            return null;
        }
```

## 4. 循环执行 handler 的 pre 拦截器

```Java
// 4. 循环执行 handler 的 pre 拦截器
for (int i = 0; i < mappedHandler.getInterceptors().length; i++) {
    HandlerInterceptor interceptor = mappedHandler.getInterceptors()[i];
    // pre 拦截器
    if (!interceptor.preHandle(request, response, mappedHandler.getHandler())) {
        return;
    }
}
```

这个没什么好讲的吧？

## 5. 执行真正的 handler，并返回  ModelAndView(Handler 是个代理对象，可能会执行 AOP )

```Java
// 5. 执行真正的 handler，并返回  ModelAndView(Handler 是个代理对象，可能会执行 AOP )
ModelAndView mv = ha.handle(request, response, mappedHandler.getHandler());
```

## 6. 循环执行 handler 的 post 拦截器

```Java
// 6. 循环执行 handler 的 post 拦截器
for (int i = mappedHandler.getInterceptors().length - 1; i >=0 ; i--) {
    HandlerInterceptor interceptor = mappedHandler.getInterceptors()[i];
    // post 拦截器
    interceptor.postHandle(request, response, mappedHandler.getHandler());
}
```

## 7. 根据 ModelAndView 信息得到 View 实例

```
View view = null;
if (mv.isReference()) {
    // We need to resolve this view name
    // 7. 根据 ModelAndView 信息得到 View 实例
    view = this.viewResolver.resolveViewName(mv.getViewName(), locale);
}
```

## 8. 渲染 View 返回

```Java
// 8. 渲染 View 返回
view.render(mv.getModel(), request, response);
```