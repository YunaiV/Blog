title: Spring Webflux —— 源码阅读之 resource 包
date: 2018-01-06
tag: 
categories: Spring Webflux
permalink: Spring-Webflux/lanneng/resource
author: 一颗懒能
from_url: https://www.jianshu.com/p/4ad3c2565650
wechat_url: 

-------

摘要: 原创出处 https://www.jianshu.com/p/4ad3c2565650 「一颗懒能」欢迎转载，保留摘要，谢谢！

- [支持为静态资源提供服务的类。](http://www.iocoder.cn/Spring-Webflux/lanneng/resource/)
  - [HttpResource](http://www.iocoder.cn/Spring-Webflux/lanneng/resource/)
  - [ResourceResolver](http://www.iocoder.cn/Spring-Webflux/lanneng/resource/)
  - [ResourceResolverChain](http://www.iocoder.cn/Spring-Webflux/lanneng/resource/)
  - [ResourceTransformer](http://www.iocoder.cn/Spring-Webflux/lanneng/resource/)
  - [ResourceTransformerChain](http://www.iocoder.cn/Spring-Webflux/lanneng/resource/)
  - [AbstractResourceResolver](http://www.iocoder.cn/Spring-Webflux/lanneng/resource/)
  - [VersionStrategy](http://www.iocoder.cn/Spring-Webflux/lanneng/resource/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# 支持为静态资源提供服务的类。

## HttpResource

HttpResource diagram:

![img](http://upload-images.jianshu.io/upload_images/8565418-40868f8748382f88.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

httpResource.jpg

将资源写入HTTP响应的扩展接口。

HTTP头将被提供给服务当前资源的HTTP响应。返回HttpHeaders

```java
public interface HttpResource extends Resource {

    /**
     * The HTTP headers to be contributed to the HTTP response
     * that serves the current resource.
     * @return the HTTP response headers
     */
    HttpHeaders getResponseHeaders();
}
```

## ResourceResolver

![img](http://upload-images.jianshu.io/upload_images/8565418-59085bf480d9651c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

![resourceTransform.jpg](http://upload-images.jianshu.io/upload_images/8565418-7323e094de2c1538.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

解决对服务器端资源的请求的策略。

提供解决传入请求到实际资源的机制，以及获取客户端在请求资源时应该使用的公共URL路径。

```java
public interface ResourceResolver {
```

将提供的请求和请求路径解析为存在于其中一个给定资源位置下的资源。

```java
Mono<Resource> resolveResource(@Nullable ServerWebExchange exchange, String requestPath,
        List<? extends Resource> locations, ResourceResolverChain chain);
```

解析面向外部的公用URL路径，供客户端用来访问位于给定内部资源路径的资源。在向客户端呈现URL链接时这很有用。

```java
Mono<String> resolveUrlPath(String resourcePath, List<? extends Resource> locations,
        ResourceResolverChain chain);

}
```

## ResourceResolverChain

调用ResourceResolvers链的协定，其中每个解析器都被赋予一个引用链，允许它在必要时进行委托。

```java
public interface ResourceResolverChain {
```

将提供的请求和请求路径解析为存在于其中一个给定资源位置下的资源。

```java
Mono<Resource> resolveResource(@Nullable ServerWebExchange exchange, String requestPath,
        List<? extends Resource> locations);
```

解析面向外部的公用URL路径，供客户端用来访问位于给定内部资源路径的资源。
Mono<String> resolveUrlPath(String resourcePath, List<? extends Resource> locations);

```java
}
```

## ResourceTransformer

![img](http://upload-images.jianshu.io/upload_images/8565418-e18237bc7be58d9e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

resourceTransform.jpg

转换资源内容的抽象。

转换给定的资源。

```java
@FunctionalInterface
public interface ResourceTransformer {

    /**
     * Transform the given resource.
     * @param exchange the current exchange
     * @param resource the resource to transform
     * @param transformerChain the chain of remaining transformers to delegate to
     * @return the transformed resource (never empty)
     */
    Mono<Resource> transform(ServerWebExchange exchange, Resource resource,
            ResourceTransformerChain transformerChain);

}
```

## ResourceTransformerChain

一个调用ResourceTransformers链的协议，每个解析器都被赋予一个链，让它在必要时进行委托。

- **getResolverChain()** 返回用于解析正在转换的资源的ResourceResolverChain。这可能需要解析相关的资源，例如链接到其他资源。
- **transform(ServerWebExchange exchange, Resource resource)** 转换给定的资源。

```java
public interface ResourceTransformerChain {

    /**
     * Return the {@code ResourceResolverChain} that was used to resolve the
     * {@code Resource} being transformed. This may be needed for resolving
     * related resources, e.g. links to other resources.
     */
    ResourceResolverChain getResolverChain();

    /**
     * Transform the given resource.
     * @param exchange the current exchange
     * @param resource the candidate resource to transform
     * @return the transformed or the same resource, never empty
     */
    Mono<Resource> transform(ServerWebExchange exchange, Resource resource);

}
```

```java
@FunctionalInterface
protected static interface CssLinkResourceTransformer.LinkParser
```

提取表示链接的内容块。

```java
@FunctionalInterface
protected interface LinkParser {

    void parse(String cssContent, SortedSet<ContentChunkInfo> result);

}
```

## AbstractResourceResolver

基于 ResourceResolver提供一致的日志记录。

```java
public abstract class AbstractResourceResolver implements ResourceResolver {

protected final Log logger = LogFactory.getLog(getClass());
```

将提供的请求和请求路径解析为存在于其中一个给定资源位置下的资源。

```java
@Override
public Mono<Resource> resolveResource(@Nullable ServerWebExchange exchange, String requestPath,
        List<? extends Resource> locations, ResourceResolverChain chain) {

    if (logger.isTraceEnabled()) {
        logger.trace("Resolving resource for request path \"" + requestPath + "\"");
    }
    return resolveResourceInternal(exchange, requestPath, locations, chain);
}
```

解析面向外部的公用URL路径，供客户端用来访问位于给定内部资源路径的资源。

```java
@Override
public Mono<String> resolveUrlPath(String resourceUrlPath, List<? extends Resource> locations,
        ResourceResolverChain chain) {

    if (logger.isTraceEnabled()) {
        logger.trace("Resolving public URL for resource path \"" + resourceUrlPath + "\"");
    }

    return resolveUrlPathInternal(resourceUrlPath, locations, chain);
}


protected abstract Mono<Resource> resolveResourceInternal(@Nullable ServerWebExchange exchange,
        String requestPath, List<? extends Resource> locations, ResourceResolverChain chain);

protected abstract Mono<String> resolveUrlPathInternal(String resourceUrlPath,
        List<? extends Resource> locations, ResourceResolverChain chain);

}
```

## VersionStrategy

![img](http://upload-images.jianshu.io/upload_images/8565418-89e96fde3d70f0b3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

VersionStrategy.jpg

确定静态资源的版本并应用and/or从URL路径中提取的策略。

```java
public interface VersionStrategy {
```

从请求路径中提取资源版本。

```java
@Nullable
String extractVersion(String requestPath);
```

从请求路径中删除版本。假定给定的版本是通过extractVersion（String）提取的。

```java
String removeVersion(String requestPath, String version);
```

给给定的请求路径添加一个版本。

```java
String addVersion(String requestPath, String version);
```

确定给定资源的版本

```java
Mono<String> getResourceVersion(Resource resource);
```

}

### VersionStrategy 的实现类：

#### AbstractFileNameVersionStrategy

基于文件名后缀的抽象基类，基于VersionStrategy实现，例如“static/ myresource-version.js”

```java
protected final Log logger = LogFactory.getLog(getClass());

private static final Pattern pattern = Pattern.compile("-(\\S*)\\.");
```

从请求路径中提取资源版本。

```java
@Override
public String extractVersion(String requestPath) {
    Matcher matcher = pattern.matcher(requestPath);
    if (matcher.find()) {
        String match = matcher.group(1);
        return (match.contains("-") ? match.substring(match.lastIndexOf('-') + 1) : match);
    }
    else {
        return null;
    }
}
```

从请求路径中删除版本

```java
@Override
public String removeVersion(String requestPath, String version) {
    return StringUtils.delete(requestPath, "-" + version);
}
```

给给定的请求路径添加一个版本。

```java
@Override
public String addVersion(String requestPath, String version) {
    String baseFilename = StringUtils.stripFilenameExtension(requestPath);
    String extension = StringUtils.getFilenameExtension(requestPath);
    return (baseFilename + '-' + version + '.' + extension);
}
```

### AbstractPrefixVersionStrategy

用于在URL路径中插入前缀的版本策略实现的抽象基类。例如：“version/static/myresource.js”。

```java
protected final Log logger = LogFactory.getLog(getClass());


private final String prefix;


protected AbstractPrefixVersionStrategy(String version) {
    Assert.hasText(version, "'version' must not be empty");
    this.prefix = version;
}


@Override
public String extractVersion(String requestPath) {
    return requestPath.startsWith(this.prefix) ? this.prefix : null;
}

@Override
public String removeVersion(String requestPath, String version) {
    return requestPath.substring(this.prefix.length());
}

@Override
public String addVersion(String path, String version) {
    if (path.startsWith(".")) {
        return path;
    }
    else if (this.prefix.endsWith("/") || path.startsWith("/")) {
        return this.prefix + path;
    }
    else {
        return this.prefix + '/' + path;
    }
}
```

### 具体实现类 ContentVersionStrategy

从资源的内容中计算Hex MD5散列的版本策略，并将其附加到文件名。“styles/ main-e36d2e05253c6c7085a91522ce43a0b4.css”。

```java
public class ContentVersionStrategy extends AbstractFileNameVersionStrategy {

private static final DataBufferFactory dataBufferFactory = new DefaultDataBufferFactory();
```

确定给定资源的版本。

```java
@Override
public Mono<String> getResourceVersion(Resource resource) {
    return DataBufferUtils.read(resource, dataBufferFactory, StreamUtils.BUFFER_SIZE)
            .reduce(DataBuffer::write)
            .map(buffer -> {
                byte[] result = new byte[buffer.readableByteCount()];
                buffer.read(result);
                DataBufferUtils.release(buffer);
                return DigestUtils.md5DigestAsHex(result);
            });
}

}
```

### 具体实现类 FixedVersionStrategy

依赖于固定版本作为请求路径前缀的VersionStrategy，例如减少SHA，版本名称，发布日期等

例如当ContentVersionStrategy无法使用时，例如使用负责加载JavaScript资源并需要知道其相对路径的JavaScript模块加载器时，这非常有用。

```java
public class FixedVersionStrategy extends AbstractPrefixVersionStrategy {

    private final Mono<String> versionMono;


    /**
     * Create a new FixedVersionStrategy with the given version string.
     * @param version the fixed version string to use
     */
    public FixedVersionStrategy(String version) {
        super(version);
        this.versionMono = Mono.just(version);
    }


    @Override
    public Mono<String> getResourceVersion(Resource resource) {
        return this.versionMono;
    }

}
```

# 666. 彩蛋

如果你对 Spring Webflux 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)