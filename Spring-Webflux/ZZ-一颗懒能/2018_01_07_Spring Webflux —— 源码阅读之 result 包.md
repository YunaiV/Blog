title: Spring Webflux —— 源码阅读之 result 包
date: 2018-01-07
tag: 
categories: Spring Webflux
permalink: Spring-Webflux/lanneng/result
author: 一颗懒能
from_url: https://www.jianshu.com/p/2aee25ff6657
wechat_url: 

-------

摘要: 原创出处 https://www.jianshu.com/p/2aee25ff6657 「一颗懒能」欢迎转载，保留摘要，谢谢！

- [Package org.springframework.web.reactive.result](http://www.iocoder.cn/Spring-Webflux/lanneng/result/)
  - [HandlerResultHandlerSupport](http://www.iocoder.cn/Spring-Webflux/lanneng/result/)
  - [接口 View的Diagram](http://www.iocoder.cn/Spring-Webflux/lanneng/result/)
  - [ViewResolver](http://www.iocoder.cn/Spring-Webflux/lanneng/result/)
  - [Rendering](http://www.iocoder.cn/Spring-Webflux/lanneng/result/)
  - [SimpleHandlerAdapter](http://www.iocoder.cn/Spring-Webflux/lanneng/result/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# Package org.springframework.web.reactive.result

支持各种编程模型样式，包括调用不同类型的handles

![img](http://upload-images.jianshu.io/upload_images/8565418-aa24fd399b55b109.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

handlerResultSupport.png

## HandlerResultHandlerSupport

HandlerResultHandler的基类，支持内容协商和访问ReactiveAdapter注册表。

```Java
    public abstract class HandlerResultHandlerSupport implements Ordered {

private static final MediaType MEDIA_TYPE_APPLICATION_ALL = new MediaType("application");


private final RequestedContentTypeResolver contentTypeResolver;

private final ReactiveAdapterRegistry adapterRegistry;

private int order = LOWEST_PRECEDENCE;


protected HandlerResultHandlerSupport(RequestedContentTypeResolver contentTypeResolver,
        ReactiveAdapterRegistry adapterRegistry) {

    Assert.notNull(contentTypeResolver, "RequestedContentTypeResolver is required");
    Assert.notNull(adapterRegistry, "ReactiveAdapterRegistry is required");
    this.contentTypeResolver = contentTypeResolver;
    this.adapterRegistry = adapterRegistry;
}


/**
 * Return the configured {@link ReactiveAdapterRegistry}.
 */
public ReactiveAdapterRegistry getAdapterRegistry() {
    return this.adapterRegistry;
}

/**
 * Return the configured {@link RequestedContentTypeResolver}.
 */
public RequestedContentTypeResolver getContentTypeResolver() {
    return this.contentTypeResolver;
}

/**
 * Set the order for this result handler relative to others.
 * <p>By default set to {@link Ordered#LOWEST_PRECEDENCE}, however see
 * Javadoc of sub-classes which may change this default.
 * @param order the order
 */
public void setOrder(int order) {
    this.order = order;
}

@Override
public int getOrder() {
    return this.order;
}


/**
 * Get a {@code ReactiveAdapter} for the top-level return value type.
 * @return the matching adapter or {@code null}
 */
@Nullable
protected ReactiveAdapter getAdapter(HandlerResult result) {
    Class<?> returnType = result.getReturnType().getRawClass();
    return getAdapterRegistry().getAdapter(returnType, result.getReturnValue());
}

/**
 * Select the best media type for the current request through a content
 * negotiation algorithm.
 * @param exchange the current request
 * @param producibleTypesSupplier the media types that can be produced for the current request
 * @return the selected media type or {@code null}
 */
@Nullable
protected MediaType selectMediaType(ServerWebExchange exchange,
        Supplier<List<MediaType>> producibleTypesSupplier) {

    List<MediaType> acceptableTypes = getAcceptableTypes(exchange);
    List<MediaType> producibleTypes = getProducibleTypes(exchange, producibleTypesSupplier);

    Set<MediaType> compatibleMediaTypes = new LinkedHashSet<>();
    for (MediaType acceptable : acceptableTypes) {
        for (MediaType producible : producibleTypes) {
            if (acceptable.isCompatibleWith(producible)) {
                compatibleMediaTypes.add(selectMoreSpecificMediaType(acceptable, producible));
            }
        }
    }

    List<MediaType> result = new ArrayList<>(compatibleMediaTypes);
    MediaType.sortBySpecificityAndQuality(result);

    for (MediaType mediaType : result) {
        if (mediaType.isConcrete()) {
            return mediaType;
        }
        else if (mediaType.equals(MediaType.ALL) || mediaType.equals(MEDIA_TYPE_APPLICATION_ALL)) {
            return MediaType.APPLICATION_OCTET_STREAM;
        }
    }

    return null;
}

private List<MediaType> getAcceptableTypes(ServerWebExchange exchange) {
    List<MediaType> mediaTypes = getContentTypeResolver().resolveMediaTypes(exchange);
    return (mediaTypes.isEmpty() ? Collections.singletonList(MediaType.ALL) : mediaTypes);
}

@SuppressWarnings("unchecked")
private List<MediaType> getProducibleTypes(ServerWebExchange exchange,
        Supplier<List<MediaType>> producibleTypesSupplier) {

    Set<MediaType> mediaTypes = exchange.getAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);
    return (mediaTypes != null ? new ArrayList<>(mediaTypes) : producibleTypesSupplier.get());
}

private MediaType selectMoreSpecificMediaType(MediaType acceptable, MediaType producible) {
    producible = producible.copyQualityValue(acceptable);
    Comparator<MediaType> comparator = MediaType.SPECIFICITY_COMPARATOR;
    return (comparator.compare(acceptable, producible) <= 0 ? acceptable : producible);
}
```

}

## 接口 View的Diagram

![img](http://upload-images.jianshu.io/upload_images/8565418-c8a10e477af378f3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

view.png

### View

通过视图解析支持结果处理。

将HandlerResult呈现给HTTP响应的约定。

与Encoder相比，Encoder是一个单实例对象，并对给定类型的任何对象进行编码，因此，视图通常是通过名称来选择的，并使用ViewResolver来解析，例如将其与HTML模板匹配。此外，视图可以基于模型中包含的多个属性呈现。

视图还可以选择从模型中选择一个属性，使用任何现有的编码器来呈现替代媒体类型。

```Java
返回此视图支持的媒体类型列表，或空列表。
List<MediaType> getSupportedMediaTypes();

这个视图是否通过执行重定向来呈现。
default boolean isRedirectView() {
    return false;
}

根据给定的HandlerResult呈现视图。实现可以访问和使用模型，或者仅在其中使用一个特定的属性。
Mono<Void> render(@Nullable Map<String, ?> model, @Nullable MediaType contentType, ServerWebExchange exchange);
```

### AbstractView

View实现的基类。

```Java
public abstract class AbstractView implements View, ApplicationContextAware {

    /**  在 bean factory 有名的RequestDataValueProcessor */
    public static final String REQUEST_DATA_VALUE_PROCESSOR_BEAN_NAME = "requestDataValueProcessor";


    /** 可用于子类的日志记录器 */
    protected final Log logger = LogFactory.getLog(getClass());

    private static final Object NO_VALUE = new Object();


    private final List<MediaType> mediaTypes = new ArrayList<>(4);

    private final ReactiveAdapterRegistry adapterRegistry;

    private Charset defaultCharset = StandardCharsets.UTF_8;

    @Nullable
    private String requestContextAttribute;

    @Nullable
    private ApplicationContext applicationContext;


    public AbstractView() {
        this(new ReactiveAdapterRegistry());
    }

    public AbstractView(ReactiveAdapterRegistry registry) {
        this.mediaTypes.add(ViewResolverSupport.DEFAULT_CONTENT_TYPE);
        this.adapterRegistry = registry;
    }


    /**
     * Set the supported media types for this view.
     * Default is "text/html;charset=UTF-8".
     */
    public void setSupportedMediaTypes(@Nullable List<MediaType> supportedMediaTypes) {
        Assert.notEmpty(supportedMediaTypes, "MediaType List must not be empty");
        this.mediaTypes.clear();
        if (supportedMediaTypes != null) {
            this.mediaTypes.addAll(supportedMediaTypes);
        }
    }

    /**
     * Return the configured media types supported by this view.
     */
    @Override
    public List<MediaType> getSupportedMediaTypes() {
        return this.mediaTypes;
    }

    /**
     * Set the default charset for this view, used when the
     * {@linkplain #setSupportedMediaTypes(List) content type} does not contain one.
     * Default is {@linkplain StandardCharsets#UTF_8 UTF 8}.
     */
    public void setDefaultCharset(Charset defaultCharset) {
        Assert.notNull(defaultCharset, "'defaultCharset' must not be null");
        this.defaultCharset = defaultCharset;
    }

    /**
     * Return the default charset, used when the
     * {@linkplain #setSupportedMediaTypes(List) content type} does not contain one.
     */
    public Charset getDefaultCharset() {
        return this.defaultCharset;
    }

    /**
     * Set the name of the RequestContext attribute for this view.
     * Default is none.
     */
    public void setRequestContextAttribute(@Nullable String requestContextAttribute) {
        this.requestContextAttribute = requestContextAttribute;
    }

    /**
     * Return the name of the RequestContext attribute, if any.
     */
    @Nullable
    public String getRequestContextAttribute() {
        return this.requestContextAttribute;
    }

    @Override
    public void setApplicationContext(@Nullable ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }

    @Nullable
    public ApplicationContext getApplicationContext() {
        return this.applicationContext;
    }

    /**
     * Obtain the ApplicationContext for actual use.
     * @return the ApplicationContext (never {@code null})
     * @throws IllegalStateException in case of no ApplicationContext set
     */
    protected final ApplicationContext obtainApplicationContext() {
        ApplicationContext applicationContext = getApplicationContext();
        Assert.state(applicationContext != null, "No ApplicationContext");
        return applicationContext;
    }


    /**
     * Prepare the model to render.
     * @param model Map with name Strings as keys and corresponding model
     * objects as values (Map can also be {@code null} in case of empty model)
     * @param contentType the content type selected to render with which should
     * match one of the {@link #getSupportedMediaTypes() supported media types}.
     * @param exchange the current exchange
     * @return {@code Mono} to represent when and if rendering succeeds
     */
    @Override
    public Mono<Void> render(@Nullable Map<String, ?> model, @Nullable MediaType contentType,
            ServerWebExchange exchange) {

        if (logger.isTraceEnabled()) {
            logger.trace("Rendering view with model " + model);
        }

        if (contentType != null) {
            exchange.getResponse().getHeaders().setContentType(contentType);
        }

        return getModelAttributes(model, exchange).flatMap(mergedModel -> {
            // Expose RequestContext?
            if (this.requestContextAttribute != null) {
                mergedModel.put(this.requestContextAttribute, createRequestContext(exchange, mergedModel));
            }
            return renderInternal(mergedModel, contentType, exchange);
        });
    }

    /**
     * Prepare the model to use for rendering.
     * <p>The default implementation creates a combined output Map that includes
     * model as well as static attributes with the former taking precedence.
     */
    protected Mono<Map<String, Object>> getModelAttributes(@Nullable Map<String, ?> model,
            ServerWebExchange exchange) {

        int size = (model != null ? model.size() : 0);

        Map<String, Object> attributes = new LinkedHashMap<>(size);
        if (model != null) {
            attributes.putAll(model);
        }

        return resolveAsyncAttributes(attributes).then(Mono.just(attributes));
    }

    /**
     * By default, resolve async attributes supported by the
     * {@link ReactiveAdapterRegistry} to their blocking counterparts.
     * <p>View implementations capable of taking advantage of reactive types
     * can override this method if needed.
     * @return {@code Mono} for the completion of async attributes resolution
     */
    protected Mono<Void> resolveAsyncAttributes(Map<String, Object> model) {

        List<String> names = new ArrayList<>();
        List<Mono<?>> valueMonos = new ArrayList<>();

        for (Map.Entry<String, ?> entry : model.entrySet()) {
            Object value =  entry.getValue();
            if (value == null) {
                continue;
            }
            ReactiveAdapter adapter = this.adapterRegistry.getAdapter(null, value);
            if (adapter != null) {
                names.add(entry.getKey());
                if (adapter.isMultiValue()) {
                    Flux<Object> fluxValue = Flux.from(adapter.toPublisher(value));
                    valueMonos.add(fluxValue.collectList().defaultIfEmpty(Collections.emptyList()));
                }
                else {
                    Mono<Object> monoValue = Mono.from(adapter.toPublisher(value));
                    valueMonos.add(monoValue.defaultIfEmpty(NO_VALUE));
                }
            }
        }

        if (names.isEmpty()) {
            return Mono.empty();
        }

        return Mono.zip(valueMonos,
                values -> {
                    for (int i=0; i < values.length; i++) {
                        if (values[i] != NO_VALUE) {
                            model.put(names.get(i), values[i]);
                        }
                        else {
                            model.remove(names.get(i));
                        }
                    }
                    return NO_VALUE;
                })
                .then();
    }

    /**
     * Create a RequestContext to expose under the specified attribute name.
     * <p>The default implementation creates a standard RequestContext instance
     * for the given request and model. Can be overridden in subclasses for
     * custom instances.
     * @param exchange current exchange
     * @param model combined output Map (never {@code null}),
     * with dynamic values taking precedence over static attributes
     * @return the RequestContext instance
     * @see #setRequestContextAttribute
     */
    protected RequestContext createRequestContext(ServerWebExchange exchange, Map<String, Object> model) {
        return new RequestContext(exchange, model, obtainApplicationContext(), getRequestDataValueProcessor());
    }

    /**
     * Return the {@link RequestDataValueProcessor} to use.
     * <p>The default implementation looks in the {@link #getApplicationContext()
     * Spring configuration} for a {@code RequestDataValueProcessor} bean with
     * the name {@link #REQUEST_DATA_VALUE_PROCESSOR_BEAN_NAME}.
     * @return the RequestDataValueProcessor, or null if there is none at the
     * application context.
     */
    @Nullable
    protected RequestDataValueProcessor getRequestDataValueProcessor() {
        ApplicationContext context = getApplicationContext();
        if (context != null && context.containsBean(REQUEST_DATA_VALUE_PROCESSOR_BEAN_NAME)) {
            return context.getBean(REQUEST_DATA_VALUE_PROCESSOR_BEAN_NAME, RequestDataValueProcessor.class);
        }
        return null;
    }

    /**
     * Subclasses must implement this method to actually render the view.
     * @param renderAttributes combined output Map (never {@code null}),
     * with dynamic values taking precedence over static attributes
     * @param contentType the content type selected to render with which should
     * match one of the {@link #getSupportedMediaTypes() supported media types}.
     *@param exchange current exchange  @return {@code Mono} to represent when
     * and if rendering succeeds
     */
    protected abstract Mono<Void> renderInternal(Map<String, Object> renderAttributes,
            @Nullable MediaType contentType, ServerWebExchange exchange);


    @Override
    public String toString() {
        return getClass().getName();
    }

}
```

## ViewResolver

![img](http://upload-images.jianshu.io/upload_images/8565418-0f88e5b28fd198b6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

ViewResolver.png

## Rendering

![img](http://upload-images.jianshu.io/upload_images/8565418-b1c57a2127225431.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

reanding.jpg

## SimpleHandlerAdapter

HandlerAdapter，它允许使用普通的WebHandler与一般的DispatcherHandler一起使用。

```Java
public class SimpleHandlerAdapter implements HandlerAdapter {

    private static final MethodParameter RETURN_TYPE;

    static {
        try {
            Method method = WebHandler.class.getMethod("handle", ServerWebExchange.class);
            RETURN_TYPE = new MethodParameter(method, -1);
        }
        catch (NoSuchMethodException ex) {
            throw new IllegalStateException(
                    "Failed to initialize the return type for WebHandler: " + ex.getMessage());
        }
    }

        //这个HandlerAdapter是否支持给定的handler。
    @Override
    public boolean supports(Object handler) {
        return WebHandler.class.isAssignableFrom(handler.getClass());
    }

      //鼓励实现处理由调用handler所产生的异常，并在必要时返回代表错误响应的替代结果。
    @Override
    public Mono<HandlerResult> handle(ServerWebExchange exchange, Object handler) {
        WebHandler webHandler = (WebHandler) handler;
        Mono<Void> mono = webHandler.handle(exchange);
        return mono.then(Mono.empty());
    }

}
```

# 666. 彩蛋

如果你对 Spring Webflux 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)