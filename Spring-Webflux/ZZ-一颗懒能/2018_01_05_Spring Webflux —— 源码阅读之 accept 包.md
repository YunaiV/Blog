title: Spring Webflux —— 源码阅读之 accept 包
date: 2018-01-05
tag: 
categories: Spring Webflux
permalink: Spring-Webflux/lanneng/accept
author: 一颗懒能
from_url: https://www.jianshu.com/p/9cf4563ee1e1
wechat_url: 

-------

摘要: 原创出处 https://www.jianshu.com/p/9cf4563ee1e1 「一颗懒能」欢迎转载，保留摘要，谢谢！


-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

RequestedContentTypeResolver策略和实现来解析给定请求的请求内容类型。

![img](http://upload-images.jianshu.io/upload_images/8565418-313a250704fa33f1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

typeResovle.jpg

### 接口

### RequestedContentTypeResolver

为ServerWebExchange解决请求的媒体类型的策略。

```java
public interface RequestedContentTypeResolver {

    /**
     * Resolve the given request to a list of requested media types. The returned
     * list is ordered by specificity first and by quality parameter second.
     * @param exchange the current exchange
     * @return the requested media types or an empty list
     * @throws NotAcceptableStatusException if the requested media type is invalid
     */
    List<MediaType> resolveMediaTypes(ServerWebExchange exchange);

}
```

将给定的请求解析为请求的媒体类型列表。返回的列表首先按特异性排序，然后按质量参数排序。

### 类

### org.springframework.web.reactive.accept.FixedContentTypeResolver

解析器始终解析为媒体类型的固定列表。这可以用作“最后一行”策略，当客户端没有请求任何媒体类型时提供回调

```java
public class FixedContentTypeResolver implements RequestedContentTypeResolver {

    private static final Log logger = LogFactory.getLog(FixedContentTypeResolver.class);


    private final List<MediaType> mediaTypes;


    /**
     * 具有单个默认MediaType的构造函数。.
     */
    public FixedContentTypeResolver(MediaType mediaType) {
        this(Collections.singletonList(mediaType));
    }

    /**
     * 使用默认的MediaType排序列表的构造函数返回用于支持各种内容类型的应用程序。
     如果目标不存在，并且不支持任何其他默认媒体类型，请考虑在最后附加MediaType.ALL。
     */
    public FixedContentTypeResolver(List<MediaType> mediaTypes) {
        this.mediaTypes = Collections.unmodifiableList(mediaTypes);
    }


    /**
     * 返回配置的媒体类型列表。
     */
    public List<MediaType> getContentTypes() {
        return this.mediaTypes;
    }


    @Override
    public List<MediaType> resolveMediaTypes(ServerWebExchange exchange) {
        if (logger.isDebugEnabled()) {
            logger.debug("Requested media types: " + this.mediaTypes);
        }
        return this.mediaTypes;
    }

}
```

### org.springframework.web.reactive.accept.HeaderContentTypeResolver

解析器查看请求的“Accept”标头。

```java
public class HeaderContentTypeResolver implements RequestedContentTypeResolver {

    @Override
    public List<MediaType> resolveMediaTypes(ServerWebExchange exchange) throws NotAcceptableStatusException {
        try {
            List<MediaType> mediaTypes = exchange.getRequest().getHeaders().getAccept();
            MediaType.sortBySpecificityAndQuality(mediaTypes);
            return mediaTypes;
        }
        catch (InvalidMediaTypeException ex) {
            String value = exchange.getRequest().getHeaders().getFirst("Accept");
            throw new NotAcceptableStatusException(
                    "Could not parse 'Accept' header [" + value + "]: " + ex.getMessage());
        }
    }

}
```

### org.springframework.web.reactive.accept.ParameterContentTypeResolver

解析查询参数并使用它查找匹配的MediaType的解析器。查找键可以注册，也可以作为一个后备的MediaTypeFactory执行查找。

```java
public class ParameterContentTypeResolver implements RequestedContentTypeResolver {

    /** Primary lookup for media types by key (e.g. "json" -> "application/json") */
    private final Map<String, MediaType> mediaTypes = new ConcurrentHashMap<>(64);

    private String parameterName = "format";


    public ParameterContentTypeResolver(Map<String, MediaType> mediaTypes) {
        mediaTypes.forEach((key, value) -> this.mediaTypes.put(formatKey(key), value));
    }

    private static String formatKey(String key) {
        return key.toLowerCase(Locale.ENGLISH);
    }


    /**
     * Set the name of the parameter to use to determine requested media types.
     * <p>By default this is set to {@literal "format"}.
     */
    public void setParameterName(String parameterName) {
        Assert.notNull(parameterName, "'parameterName' is required");
        this.parameterName = parameterName;
    }

    public String getParameterName() {
        return this.parameterName;
    }


    @Override
    public List<MediaType> resolveMediaTypes(ServerWebExchange exchange) throws NotAcceptableStatusException {
        String key = exchange.getRequest().getQueryParams().getFirst(getParameterName());
        if (!StringUtils.hasText(key)) {
            return Collections.emptyList();
        }
        key = formatKey(key);
        MediaType match = this.mediaTypes.get(key);
        if (match == null) {
            match = MediaTypeFactory.getMediaType("filename." + key)
                    .orElseThrow(() -> {
                        List<MediaType> supported = new ArrayList<>(this.mediaTypes.values());
                        return new NotAcceptableStatusException(supported);
                    });
        }
        this.mediaTypes.putIfAbsent(key, match);
        return Collections.singletonList(match);
    }
```

### org.springframework.web.reactive.accept.RequestedContentTypeResolverBuilder

Builder的复合RequestedContentTypeResolver代表其他解析器实现一个不同的策略来确定请求的内容类型,例如Accept标头,查询参数,或其他。
使用生成器方法在所需的顺序中添加解析器。对于给定的请求，他首先解析返回一个非空的列表，并且不包含只是MediaType。将使用。
默认情况下,如果没有显式解析器配置,构建器将添加HeaderContentTypeResolver。

```java
public class RequestedContentTypeResolverBuilder {

    private final List<Supplier<RequestedContentTypeResolver>> candidates = new ArrayList<>();


    /**
     * Add a resolver to get the requested content type from a query parameter.
     * By default the query parameter name is {@code "format"}.
     */
    public ParameterResolverConfigurer parameterResolver() {
        ParameterResolverConfigurer parameterBuilder = new ParameterResolverConfigurer();
        this.candidates.add(parameterBuilder::createResolver);
        return parameterBuilder;
    }

    /**
     * Add resolver to get the requested content type from the
     * {@literal "Accept"} header.
     */
    public void headerResolver() {
        this.candidates.add(HeaderContentTypeResolver::new);
    }

    /**
     * Add resolver that returns a fixed set of media types.
     * @param mediaTypes the media types to use
     */
    public void fixedResolver(MediaType... mediaTypes) {
        this.candidates.add(() -> new FixedContentTypeResolver(Arrays.asList(mediaTypes)));
    }

    /**
     * Add a custom resolver.
     * @param resolver the resolver to add
     */
    public void resolver(RequestedContentTypeResolver resolver) {
        this.candidates.add(() -> resolver);
    }

    /**
     * Build a {@link RequestedContentTypeResolver} that delegates to the list
     * of resolvers configured through this builder.
     */
    public RequestedContentTypeResolver build() {

        List<RequestedContentTypeResolver> resolvers =
                this.candidates.isEmpty() ?
                        Collections.singletonList(new HeaderContentTypeResolver()) :
                        this.candidates.stream().map(Supplier::get).collect(Collectors.toList());

        return exchange -> {
            for (RequestedContentTypeResolver resolver : resolvers) {
                List<MediaType> type = resolver.resolveMediaTypes(exchange);
                if (type.isEmpty() || (type.size() == 1 && type.contains(MediaType.ALL))) {
                    continue;
                }
                return type;
            }
            return Collections.emptyList();
        };
    }


    /**
     * Helper to create and configure {@link ParameterContentTypeResolver}.
     */
    public static class ParameterResolverConfigurer {

        private final Map<String, MediaType> mediaTypes = new HashMap<>();

        @Nullable
        private String parameterName;

        /**
         * Configure a mapping between a lookup key (extracted from a query
         * parameter value) and a corresponding {@code MediaType}.
         * @param key the lookup key
         * @param mediaType the MediaType for that key
         */
        public ParameterResolverConfigurer mediaType(String key, MediaType mediaType) {
            this.mediaTypes.put(key, mediaType);
            return this;
        }

        /**
         * Map-based variant of {@link #mediaType(String, MediaType)}.
         * @param mediaTypes the mappings to copy
         */
        public ParameterResolverConfigurer mediaType(Map<String, MediaType> mediaTypes) {
            this.mediaTypes.putAll(mediaTypes);
            return this;
        }

        /**
         * Set the name of the parameter to use to determine requested media types.
         * <p>By default this is set to {@literal "format"}.
         */
        public ParameterResolverConfigurer parameterName(String parameterName) {
            this.parameterName = parameterName;
            return this;
        }

        /**
         * Private factory method to create the resolver.
         */
        private RequestedContentTypeResolver createResolver() {
            ParameterContentTypeResolver resolver = new ParameterContentTypeResolver(this.mediaTypes);
            if (this.parameterName != null) {
                resolver.setParameterName(this.parameterName);
            }
            return resolver;
        }
    }

}
```

# 666. 彩蛋

如果你对 Spring Webflux 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)