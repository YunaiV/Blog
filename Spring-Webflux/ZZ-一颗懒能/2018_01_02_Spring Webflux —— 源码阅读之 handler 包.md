title: Spring Webflux —— 源码阅读之 handler 包
date: 2018-01-02
tag: 
categories: Spring Webflux
permalink: Spring-Webflux/lanneng/handler
author: 一颗懒能
from_url: https://www.jianshu.com/p/23ee4b77c160
wechat_url: https://mp.weixin.qq.com/s/W7avKGPxBT7kx2s7rQOCoA

-------

摘要: 原创出处 https://www.jianshu.com/p/23ee4b77c160 「一颗懒能」欢迎转载，保留摘要，谢谢！

- [AbstractHandlerMapping](http://www.iocoder.cn/Spring-Webflux/lanneng/handler/)
- [接口](http://www.iocoder.cn/Spring-Webflux/lanneng/handler/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

提供包括抽象基类在内的HandlerMapping实现。

先扔一张整体的diagram类图：

![img](http://upload-images.jianshu.io/upload_images/8565418-70f8c0ce8d7aa434.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

simoleUrlHandlerMapping.jpg

## AbstractHandlerMapping

HandlerMapping实现的抽象基类。

## 接口

- java.lang.Object
  - org.springframework.context.support.ApplicationObjectSupport
    - org.springframework.web.reactive.handler.AbstractHandlerMapping

实现了HandlerMapping, Ordered

```Java
    public abstract class AbstractHandlerMapping extends
    ApplicationObjectSupport implements HandlerMapping, Ordered {



private static final WebHandler REQUEST_HANDLED_HANDLER = exchange -> Mono.empty();


private int order = Integer.MAX_VALUE;  // default: same as non-Ordered

private final PathPatternParser patternParser;

private final UrlBasedCorsConfigurationSource globalCorsConfigSource;

private CorsProcessor corsProcessor = new DefaultCorsProcessor();


public AbstractHandlerMapping() {
      this.patternParser = new PathPatternParser();
      this.globalCorsConfigSource = new UrlBasedCorsConfigurationSource(this.patternParser);
}
```

**查找给定请求的handler，如果找不到特定的请求，则返回一个空的Mono。这个方法被getHandler（org.springframework.web.server.ServerWebExchange）调用。**

在CORS 预先请求中，该方法应该返回一个匹配，而不是预先请求的请求，而是基于URL路径的预期实际请求，从“Access-Control-Request-Method”头，以及“Access-Control-Request-Headers”头的HTTP方法，通过 getcorsconfiguration获得CORS配置来允许通过，

如果匹配到一个handler，就返回Mono

```Java
protected abstract Mono<?> getHandlerInternal(ServerWebExchange exchange);
```

检索给定handle的CORS配置。

```Java
@Nullable
protected CorsConfiguration getCorsConfiguration(Object handler, ServerWebExchange exchange) {
    if (handler instanceof CorsConfigurationSource) {
        return ((CorsConfigurationSource) handler).getCorsConfiguration(exchange);
    }
    return null;
}
```

抽象类实现的主要的具体方法,来获得具体的Handle，实现了HandlerMapping中的getHandler，`Mono<Object> getHandler(ServerWebExchange exchange);`

```Java
@Override
public Mono<Object> getHandler(ServerWebExchange exchange) {
    return getHandlerInternal(exchange).map(handler -> {
        if (CorsUtils.isCorsRequest(exchange.getRequest())) {
            CorsConfiguration configA = this.globalCorsConfigSource.getCorsConfiguration(exchange);
            CorsConfiguration configB = getCorsConfiguration(handler, exchange);
            CorsConfiguration config = (configA != null ? configA.combine(configB) : configB);
            if (!getCorsProcessor().process(config, exchange) ||
                    CorsUtils.isPreFlightRequest(exchange.getRequest())) {
                return REQUEST_HANDLED_HANDLER;
            }
        }
        return handler;
    });
}
```

### AbstractUrlHandlerMapping

**基于URL映射的HandlerMapping实现的抽象基类。**

### 接口

- java.lang.Object
  - org.springframework.context.support.ApplicationObjectSupport
    - org.springframework.web.reactive.handler.AbstractHandlerMapping
      - org.springframework.web.reactive.handler.AbstractUrlHandlerMapping

支持直接匹配，例如注册的“/ test”匹配“/ test”，以及各种ant样式匹配，例如， “/ test *”匹配“/ test”和“/ team”，“/ test / *”匹配“/ test”下的所有路径， 。有关详细信息，请参阅PathPattern javadoc。

将搜索所有路径模式以查找当前请求路径的最具体匹配。最具体的模式定义为使用最少捕获变量和通配符的最长路径模式。

```Java
private final Map<PathPattern, Object> handlerMap = new LinkedHashMap<>();
```

返回注册路径模式和handle的只读视图，这些注册路径模式和handle可能是一个实际的handle实例或延迟初始化handle的bean名称。

```Java
public final Map<PathPattern, Object> getHandlerMap() {
    return Collections.unmodifiableMap(this.handlerMap);
}
```

我们可以再看到下面这两个方法实现了handle的注册，会把所有的路径映射，和handle实例放在handlerMap中

```Java
protected void registerHandler(String[] urlPaths, String beanName)
        throws BeansException, IllegalStateException {

    Assert.notNull(urlPaths, "URL path array must not be null");
    for (String urlPath : urlPaths) {
        registerHandler(urlPath, beanName);
    }
}


protected void registerHandler(String urlPath, Object handler)
        throws BeansException, IllegalStateException {

    Assert.notNull(urlPath, "URL path must not be null");
    Assert.notNull(handler, "Handler object must not be null");
    Object resolvedHandler = handler;

    // Parse path pattern
    urlPath = prependLeadingSlash(urlPath);
    PathPattern pattern = getPathPatternParser().parse(urlPath);
    if (this.handlerMap.containsKey(pattern)) {
        Object existingHandler = this.handlerMap.get(pattern);
        if (existingHandler != null) {
            if (existingHandler != resolvedHandler) {
                throw new IllegalStateException(
                        "Cannot map " + getHandlerDescription(handler) + " to [" + urlPath + "]: " +
                        "there is already " + getHandlerDescription(existingHandler) + " mapped.");
            }
        }
    }

    // Eagerly resolve handler if referencing singleton via name.
    if (!this.lazyInitHandlers && handler instanceof String) {
        String handlerName = (String) handler;
        if (obtainApplicationContext().isSingleton(handlerName)) {
            resolvedHandler = obtainApplicationContext().getBean(handlerName);
        }
    }

    // Register resolved handler
    this.handlerMap.put(pattern, resolvedHandler);
    if (logger.isInfoEnabled()) {
        logger.info("Mapped URL path [" + urlPath + "] onto " + getHandlerDescription(handler));
    }
}
```

预处理映射的路径，如果不以`/`开头就加上`/`

```Java
private static String prependLeadingSlash(String pattern) {
    if (StringUtils.hasLength(pattern) && !pattern.startsWith("/")) {
        return "/" + pattern;
    }
    else {
        return pattern;
    }
}
```

在这一步的时候开始获取内部的handle，查找给定请求的handle，如果找不到特定的请求，则返回一个空的Mono。

```Java
    @Override
public Mono<Object> getHandlerInternal(ServerWebExchange exchange) {
    PathContainer lookupPath = exchange.getRequest().getPath().pathWithinApplication();
    Object handler;
    try {
        handler = lookupHandler(lookupPath, exchange);
    }
    catch (Exception ex) {
        return Mono.error(ex);
    }

    if (handler != null && logger.isDebugEnabled()) {
        logger.debug("Mapping [" + lookupPath + "] to " + handler);
    }
    else if (handler == null && logger.isTraceEnabled()) {
        logger.trace("No handler mapping found for [" + lookupPath + "]");
    }

    return Mono.justOrEmpty(handler);
}
```

获取到handle的类，先获取请求的url地址，调用`lookupHandler(lookupPath, exchange)`去找这个handle。

```Java
@Nullable
protected Object lookupHandler(PathContainer lookupPath, ServerWebExchange exchange)
        throws Exception {

    return this.handlerMap.entrySet().stream()
            .filter(entry -> entry.getKey().matches(lookupPath))
            .sorted((entry1, entry2) ->
                    PathPattern.SPECIFICITY_COMPARATOR.compare(entry1.getKey(), entry2.getKey()))
            .findFirst()
            .map(entry -> {
                PathPattern pattern = entry.getKey();
                if (logger.isDebugEnabled()) {
                    logger.debug("Matching pattern for request [" + lookupPath + "] is " + pattern);
                }
                PathContainer pathWithinMapping = pattern.extractPathWithinPattern(lookupPath);
                return handleMatch(entry.getValue(), pattern, pathWithinMapping, exchange);
            })
            .orElse(null);
}
```

在这里又调用了`handleMatch(entry.getValue(), pattern, pathWithinMapping, exchange)`来匹配handle，验证过后，然后设置到ServerWebExchange中最后返回。

```Java
private Object handleMatch(Object handler, PathPattern bestMatch, PathContainer pathWithinMapping,
        ServerWebExchange exchange) {

    // Bean name or resolved handler?
    if (handler instanceof String) {
        String handlerName = (String) handler;
        handler = obtainApplicationContext().getBean(handlerName);
    }

    validateHandler(handler, exchange);

    exchange.getAttributes().put(BEST_MATCHING_HANDLER_ATTRIBUTE, handler);
    exchange.getAttributes().put(BEST_MATCHING_PATTERN_ATTRIBUTE, bestMatch);
    exchange.getAttributes().put(PATH_WITHIN_HANDLER_MAPPING_ATTRIBUTE, pathWithinMapping);

    return handler;
}
```

### SimpleUrlHandlerMapping

HandlerMapping的实现，来把url请求映射到对应的`request handler`的bean

支持映射到bean实例和映射到bean名称;非单例的handler需要映射到bean名称。

“urlMap”属性适合用bean实例填充处理程序映射。可以通过java.util.Properties类接受的形式，通过“映射”属性设置映射到bean名称，如下所示：

> /welcome.html=ticketController

> /show.html=ticketController

语法是PATH = HANDLER_BEAN_NAME。如果路径不是以斜杠开始的，则给它自动补充一个斜杠。

支持直接匹配，例如注册的“/ test”匹配“/ test”，以及各种ant样式匹配，例如， “/ test *”匹配“/ test”和“/ team”，“/ test / *”匹配“/ test”下的所有路径， 。有关详细信息，请参阅PathPattern javadoc。

### 接口

- java.lang.Object
  - org.springframework.context.support.ApplicationObjectSupport
    - org.springframework.web.reactive.handler.AbstractHandlerMapping
      - org.springframework.web.reactive.handler.AbstractUrlHandlerMapping
        - org.springframework.web.reactive.handler.SimpleUrlHandlerMapping

```Java
private final Map<String, Object> urlMap = new LinkedHashMap<>();


程序启动遍历的时候把加载到的所有映射路径，和handle设置到urlMap
public void setUrlMap(Map<String, ?> urlMap) {
    this.urlMap.putAll(urlMap);
}

获得所有的urlMap，允许urlMap访问URL路径映射，可以添加或覆盖特定条目。
public Map<String, ?> getUrlMap() {
    return this.urlMap;
}


初始化程序上下文，除了父类的初始化，还调用了registerHandler
@Override
public void initApplicationContext() throws BeansException {
    super.initApplicationContext();
    registerHandlers(this.urlMap);
}
```

开始注册handler，注册urlMap中为相应路径指定的所有的handler。
如果handler不能注册，抛出 `BeansException`
如果有注册的handler有冲突，比如两个相同的，抛出`java.lang.IllegalStateException`

```java
protected void registerHandlers(Map<String, Object> urlMap) throws BeansException {
    if (urlMap.isEmpty()) {
        logger.warn("Neither 'urlMap' nor 'mappings' set on SimpleUrlHandlerMapping");
    }
    else {
        for (Map.Entry<String, Object> entry : urlMap.entrySet()) {
            String url = entry.getKey();
            Object handler = entry.getValue();
            // Prepend with slash if not already present.
            if (!url.startsWith("/")) {
                url = "/" + url;
            }
            // Remove whitespace from handler bean name.
            if (handler instanceof String) {
                handler = ((String) handler).trim();
            }
            registerHandler(url, handler);
        }
    }
}
```

这里调用的`registerHandler(url, handler)`就是刚刚抽象类`AbstractUrlHandlerMapping`中的registerHandler方法

# 666. 彩蛋

如果你对 Spring Webflux 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)