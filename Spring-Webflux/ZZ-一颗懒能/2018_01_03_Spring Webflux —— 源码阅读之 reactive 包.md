title: Spring Webflux —— 源码阅读之 reactive 包
date: 2018-01-03
tag: 
categories: Spring Webflux
permalink: Spring-Webflux/lanneng/reactive
author: 一颗懒能
from_url: https://www.jianshu.com/p/869dec448bdf
wechat_url: 

-------

摘要: 原创出处 https://www.jianshu.com/p/869dec448bdf 「一颗懒能」欢迎转载，保留摘要，谢谢！

- [（Interface）](http://www.iocoder.cn/Spring-Webflux/lanneng/reactive/)
- [HandlerAdapter](http://www.iocoder.cn/Spring-Webflux/lanneng/reactive/)
- [All Known Implementing Classes 所有已知实施类：](http://www.iocoder.cn/Spring-Webflux/lanneng/reactive/)
- [HandlerMapping](http://www.iocoder.cn/Spring-Webflux/lanneng/reactive/)
- [All Known Implementing Classes 所有已知实施类：](http://www.iocoder.cn/Spring-Webflux/lanneng/reactive/)
- [HandlerResultHandler](http://www.iocoder.cn/Spring-Webflux/lanneng/reactive/)
- [All Known Implementing Classes 所有已知实施类：](http://www.iocoder.cn/Spring-Webflux/lanneng/reactive/)
- [Class](http://www.iocoder.cn/Spring-Webflux/lanneng/reactive/)
- [HandlerResult](http://www.iocoder.cn/Spring-Webflux/lanneng/reactive/)
- [DispatcherHandler（程序入口 ， 核心）](http://www.iocoder.cn/Spring-Webflux/lanneng/reactive/)
- [All Implemented Interfaces 所有实现的接口：](http://www.iocoder.cn/Spring-Webflux/lanneng/reactive/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

### 介绍

`Spring-webflux`模块的顶级软件包包含`DispatcherHandler`，它是WebFlux服务器端点处理的主要入口点，包括用于将请求映射到处理程序的关键协议，调用它们并处理结果。

该模块为反应式服务器端点提供两种编程模型。一个基于注释`@Controller` ，另一个基于功能路由和处理。该模块还包含一个功能性，反应性`WebClient`以及客户端和服务器，反应式`WebSocket`支持。

几张diagram：

![img](http://upload-images.jianshu.io/upload_images/8565418-b2593ce0e3d52fa5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/591)

dispachServlet.jpg

## （Interface）

## HandlerAdapter

将`DispatcherHandler`与调用处理程序的细节相分离的协议，使得它可以支持任何处理程序类型。

### supports

> boolean supports(Object handler);

这个`HandlerAdapter`是否支持给定的`handler`。

> Mono<HandlerResult> handle(ServerWebExchange exchange, Object handler);

用给定的handler 来处理请求

鼓励实现处理因调用处理程序而导致的异常，并且必要时返回表示错误响应的替代结果。

此外，由于异步`HandlerResult`可能会在结果处理过程中产生错误，因此还鼓励在`HandlerResult`中设置异常处理程序，以便稍后在结果处理后也可以应用异常处理程序。

`exchange` 当前的服务器交换

返回值：`reactor.core.publisher.Mono<HandlerResult>` ， 这将返回`Mono`,一个单独的`HandlerResult`或没有，
如果请求已完全处理，不需要进一步处理。

## All Known Implementing Classes 所有已知实施类：

`HandlerFunctionAdapter`, `RequestMappingHandlerAdapter`, `SimpleHandlerAdapter`, `WebSocketHandlerAdapter`

## HandlerMapping

## All Known Implementing Classes 所有已知实施类：

`AbstractHandlerMapping`,`AbstractHandlerMethodMapping`, `AbstractUrlHandlerMapping`,`RequestMappingHandlerMapping`, `RequestMappingInfoHandlerMapping`, `RouterFunctionMapping`, `SimpleUrlHandlerMapping`

由定义请求和处理程序对象之间的映射的对象实现的接口。

> BEST_MATCHING_HANDLER_ATTRIBUTE
> 包含最佳匹配模式的映射处理程序的属性名称。

> BEST_MATCHING_PATTERN_ATTRIBUTE
> 在处理程序映射中包含最佳匹配模式的属性的名称。

> MATRIX_VARIABLES_ATTRIBUTE
> 包含具有URI变量名称的映射的属性的名称以及每个URI变量的相应MultiValueMap。

> PATH_WITHIN_HANDLER_MAPPING_ATTRIBUTE
> 包含处理程序映射中的路径的属性的名称，在匹配的情况下，例如“/ static / **”或完全相关的URI。

> PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE

包含适用于映射处理程序的可生产的MediaType集合的属性的名称。

> URI_TEMPLATE_VARIABLES_ATTRIBUTE
> 包含URI模板的属性的名称将变量名称映射到值。

`Mono<Object> getHandler(ServerWebExchange exchange);`

返回此请求的处理程序(handler)。

返回`Mono<Object>`：一个发出一个值的Mono，如果请求无法解析到一个处理程序，那么它就没有

## HandlerResultHandler

## All Known Implementing Classes 所有已知实施类：

`ResponseBodyResultHandler`, `ResponseEntityResultHandler`, `ServerResponseResultHandler`, `ViewResolutionResultHandler`

处理`HandlerResult`，通常由`HandlerAdapter`返回。

### supports

> boolean supports(HandlerResult result);;

该对象是否可以使用给定的`HandlerResult`

> Mono<Void> handleResult(ServerWebExchange exchange, HandlerResult result);

处理给定的结果修改响应标头and/or将数据写入响应。

`result` - 处理的结果

`Mono <Void>`来指示请求处理完成。

用给定的handler 来处理请求

鼓励实现处理因调用处理程序而导致的异常，并且必要时返回表示错误响应的替代结果。

## Class

### BindingContext

帮助将请求数据绑定到对象上并提供对具有控制器特定属性的共享模型的访问的上下文。

提供为特定目标创建`WebExchangeDataBinder`的方法，命令对象将数据绑定和验证应用，或者没有目标对象用于从请求值进行简单类型转换。用于请求的默认模型的容器。

### Constructors 构造器

> BindingContext()

创建一个新的BindingContext。

> BindingContext(WebBindingInitializer initializer)

使用给定的初始化程序创建一个新的BindingContext。
初始化程序 - 要应用的绑定初始化程序（可能为null）

### Method

> public Model getModel()

```
返回一个对象
```

> public WebExchangeDataBinder createDataBinder(ServerWebExchange exchange,
> @Nullable
> java.lang.Object target,
> java.lang.String name)

```
                                          创建一个WebExchangeDataBinder，以在目标命令对象上应用数据绑定和验证。
```

> protected WebExchangeDataBinder initDataBinder(WebExchangeDataBinder binder,
> ServerWebExchange exchange)

```
                                           始化给定交换的数据绑定实例。
```

> public WebExchangeDataBinder createDataBinder(ServerWebExchange exchange,
> java.lang.String name)

```
                                          创建一个没有目标对象的WebExchangeDataBinder，将请求值的类型转换为简单类型。
```

## HandlerResult

表示调用处理程序或处理程序方法的结果。

> public HandlerResult(java.lang.Object handler,
> @Nullable
> java.lang.Object returnValue,
> MethodParameter returnType)

```
                 创建一个新的HandlerResult
```

- handler - the handler that handled the request
- returnValue - the return value from the handler possibly null
- returnType - the return value type
- ​

> public java.lang.Object getHandler()

```
返回处理请求的处理程序。
```

> @Nullable
> public java.lang.Object getReturnValue()

```
返回从处理程序返回的值（如果有）。
```

> public ResolvableType getReturnType()

```
返回从处理程序返回的值的类型 - 例如在控制器方法的签名上声明的返回类型。另请参见getReturnTypeSource（）来获取返回类型的底层MethodParameter。
```

> public BindingContext getBindingContext()

返回用于请求处理的BindingContext。

## DispatcherHandler（程序入口 ， 核心）

## All Implemented Interfaces 所有实现的接口：

`Aware`, `ApplicationContextAware`, `WebHandler`

HTTP请求处理程序/控制器的中央调度程序

分发已经注册的处理程序来处理请求，提供方便的映射方式

DispatcherHandler从Spring配置中发现需要的代理组件。它在应用程序上下文中检测到以下内容：

- HandlerMapping -- map requests to handler objects（将请求映射到处理对象）
- HandlerAdapter -- for using any handler interface（使用相应的处理程序接口）
- HandlerResultHandler -- process handler return values（流程处理程序返回值）

`DispatcherHandler`也设计为一个`Spring bean`本身，并实现了`ApplicationContextAware`以访问其运行的上下文。如果`DispatcherHandler`被声明为bean名称“webHandler”，那么它将被`WebHttpHandlerBuilder.applicationContext`（`org.springframework.context.ApplicationContext`）发现，与`WebFilter`，`WebExceptionHandler`等一起创建一个处理链。

`@EnableWebFlux`配置中包含`DispatcherHandler` beande声明。

### 构造器

> DispatcherHandler()

**创建一个新的DispatcherHandler，它需要通过setApplicationContext（org.springframework.context.ApplicationContext）配置一个ApplicationContext**

> DispatcherHandler(ApplicationContext applicationContext)

**为给定的ApplicationContext创建一个新的DispatcherHandler。**

### 方法摘要

> @Nullable
> public final java.util.List<HandlerMapping> getHandlerMappings()

**返回在注入的上下文中类型检测到的所有HandlerMapping bean，并排序。**

> 注意:如果在setApplicationContext（ApplicationContext）之前调用，此方法可能返回null。返回的是具有配置的映射的不可变列表或空值

> public void setApplicationContext(ApplicationContext applicationContext)

设置此对象运行的ApplicationContext。通常，此调用将用于初始化对象。

在普通bean属性的采集之后但是在初始化回调之前调用，例如InitializingBean.afterPropertiesSet（）或自定义init方法。在ResourceLoaderAware.setResourceLoader（org.springframework.core.io.ResourceLoader），ApplicationEventPublisherAware.setApplicationEventPublisher（org.springframework.context.ApplicationEventPublisher）和MessageSourceAware（如果适用）后调用。

覆盖了接口ApplicationContextAware中的setApplicationContext

applicationContext - 该对象要使用的ApplicationContext对象

> protected void initStrategies(ApplicationContext context)

** 初始化环境 **

> public reactor.core.publisher.Mono<java.lang.Void> handle(ServerWebExchange exchange)

处理Web服务器交换。

在接口WebHandler中定义了handle方法，在这里来实现

`Mono <Void>`来指示请求处理完成

下面让我们来看看这个调度器是怎么实现的。

```Java
    @Nullable
    private List<HandlerMapping> handlerMappings;

    @Nullable
    private List<HandlerAdapter> handlerAdapters;

    @Nullable
    private List<HandlerResultHandler> resultHandlers;

```

首先定义了可以为空的三个List列表，分别是`handlerMappings`，`handlerAdapters`，`resultHandlers`

```Java
public DispatcherHandler(ApplicationContext applicationContext) {
        initStrategies(applicationContext);
    }

@Override
public void setApplicationContext(ApplicationContext applicationContext) {
        initStrategies(applicationContext);
    }


protected void initStrategies(ApplicationContext context) {
        Map<String, HandlerMapping> mappingBeans = BeanFactoryUtils.beansOfTypeIncludingAncestors(
                context, HandlerMapping.class, true, false);

        ArrayList<HandlerMapping> mappings = new ArrayList<>(mappingBeans.values());
        AnnotationAwareOrderComparator.sort(mappings);
        this.handlerMappings = Collections.unmodifiableList(mappings);

        Map<String, HandlerAdapter> adapterBeans = BeanFactoryUtils.beansOfTypeIncludingAncestors(
                context, HandlerAdapter.class, true, false);

        this.handlerAdapters = new ArrayList<>(adapterBeans.values());
        AnnotationAwareOrderComparator.sort(this.handlerAdapters);

        Map<String, HandlerResultHandler> beans = BeanFactoryUtils.beansOfTypeIncludingAncestors(
                context, HandlerResultHandler.class, true, false);

        this.resultHandlers = new ArrayList<>(beans.values());
        AnnotationAwareOrderComparator.sort(this.resultHandlers);
    }
```

在实例化DispatcherHandler之后，开始调用`initStrategies（applicationContext）`，这个方法，`beansOfTypeIncludingAncestors`这个方法来通过解析和反射等方法来收集`context`中所有的bean（可以通过注解或者xml配置Spring Bean，Springboot都是注解一般），来创建ioc容器， 上面代码中的`mappingBeans` `adapterBeans` `beans` 都是如此，创建好之后来让之前最初创建的三个List变量集合引用，并且对他们进行排序。

到现在为止，初始化已经完成了，接下来我们来看这个入口类的核心方法：

```Java
    @Override
    public Mono<Void> handle(ServerWebExchange exchange) {
        if (logger.isDebugEnabled()) {
            ServerHttpRequest request = exchange.getRequest();
            logger.debug("Processing " + request.getMethodValue() + " request for [" + request.getURI() + "]");
        }
        if (this.handlerMappings == null) {
            return Mono.error(HANDLER_NOT_FOUND_EXCEPTION);
        }
        return Flux.fromIterable(this.handlerMappings)
                .concatMap(mapping -> mapping.getHandler(exchange))
                .next()
                .switchIfEmpty(Mono.error(HANDLER_NOT_FOUND_EXCEPTION))
                .flatMap(handler -> invokeHandler(exchange, handler))
                .flatMap(result -> handleResult(exchange, result));
    }


package org.springframework.web.server;

public interface WebHandler {

    /**
     * Handle the web server exchange.
     * @param exchange the current server exchange
     * @return {@code Mono<Void>} to indicate when request handling is complete
     */
    Mono<Void> handle(ServerWebExchange exchange);

}


```

其实这个handle方法是实现了`package org.springframework.web.server`中的`WebHandler`中的接口，这里可以看到先拿到了请求中的`request`，然后接下来就是`Flux`开始遍历我们的`handlerMappings`,从`ServerWebExchange`中获得相应的`Handle`,会调用`invokeHandler(exchange, handler)`来匹配真正要使用的那个`handlerAdapter`，来执行handle方法处理请求逻辑，`return handlerAdapter.handle(exchange, handler);`,这个过程中，别的类已经处理完了，并且返回了结果(还没看到哪些类)，返回一个`Mono<HandlerResult>`,代码如下：

```Java
    private Mono<HandlerResult> invokeHandler(ServerWebExchange exchange, Object handler) {
        if (this.handlerAdapters != null) {
            for (HandlerAdapter handlerAdapter : this.handlerAdapters) {
                if (handlerAdapter.supports(handler)) {
                    return handlerAdapter.handle(exchange, handler);
                }
            }
        }
        return Mono.error(new IllegalStateException("No HandlerAdapter: " + handler));
    }
```

最后就是调用`handleResult(exchange, result)`这个方法，代码如下：

```Java
    private HandlerResultHandler getResultHandler(HandlerResult handlerResult) {
        if (this.resultHandlers != null) {
            for (HandlerResultHandler resultHandler : this.resultHandlers) {
                if (resultHandler.supports(handlerResult)) {
                    return resultHandler;
                }
            }
        }
        throw new IllegalStateException("No HandlerResultHandler for " + handlerResult.getReturnValue());
    }

```

同理，也是要遍历来匹配真正的`resultHandler`,然后获得这个，调用`handlerResult.getReturnValue());`来返回结果。今天先看到这，还有点懵。不过可以感受到这个异步非阻塞IO框架的强大之处。

# 666. 彩蛋

如果你对 Spring Webflux 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)