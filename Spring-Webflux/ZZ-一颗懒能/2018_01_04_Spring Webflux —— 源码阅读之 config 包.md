title: Spring Webflux —— 源码阅读之 config 包
date: 2018-01-04
tag: 
categories: Spring Webflux
permalink: Spring-Webflux/lanneng/config
author: 一颗懒能
from_url: https://www.jianshu.com/p/a266a676bf9d
wechat_url: 

-------

摘要: 原创出处 https://www.jianshu.com/p/a266a676bf9d 「一颗懒能」欢迎转载，保留摘要，谢谢！

- [包名 `org.springframework.web.reactive.config`](http://www.iocoder.cn/Spring-Webflux/lanneng/config/)
- [介绍](http://www.iocoder.cn/Spring-Webflux/lanneng/config/)
- [接口 `WebFluxConfigurer`](http://www.iocoder.cn/Spring-Webflux/lanneng/config/)
- [类](http://www.iocoder.cn/Spring-Webflux/lanneng/config/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

## 包名 `org.springframework.web.reactive.config`

看一张diagram：

![img](http://upload-images.jianshu.io/upload_images/8565418-fde4139fe6ee6d28.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/568)

config.jpg

## 介绍

`Spring WebFlux`配置基础架构。
将此注释添加到`@Configuration`类中，从`WebFluxConfigurationSupport`导入`Spring Web Reactive`配置，例如：

```Java
 @Configuration
 @EnableWebFlux
 @ComponentScan(basePackageClasses = MyConfiguration.class)
 public class MyConfiguration {
 }

```

## 接口 `WebFluxConfigurer`

定义了回调方法来自定义通过@EnableWebFlux来启用WebFlux应用程序的配置。

@Enablewebflux注释的配置类可以实现这个接口的调用，并提供一个定制默认配置的机会。考虑实现这个接口，并根据需要覆盖相关的方法。

> default void configureContentTypeResolver(RequestedContentTypeResolverBuilder builder)

配置如何解决响应请求的内容类型。

> default void addCorsMappings(CorsRegistry registry)

配置交叉原点请求处理。

> default void configurePathMatching(PathMatchConfigurer configurer)

配置路径匹配选项。 HandlerMappings与路径匹配选项。

> default void addResourceHandlers(ResourceHandlerRegistry registry)

为服务静态资源添加资源处理程序。

> default void configureArgumentResolvers(ArgumentResolverConfigurer configurer)

配置自定义控制器方法参数的解析器。

> default void configureHttpMessageCodecs(ServerCodecConfigurer configurer)

配置自定义HTTP消息读取器和编写器或重写内置的。

> default void addFormatters(FormatterRegistry registry)

添加自定义转换器和格式化程序，用于执行控制器方法参数的类型转换和格式化。

> @Nullable
> default MessageCodesResolver getMessageCodesResolver()

供一个自定义MessageCodesResolver用于数据绑定，而不是DataBinder中默认创建的那个。

> default void configureViewResolvers(ViewResolverRegistry registry)

配置视图解析以处理控制器方法的返回值，这些方法依赖于解析视图来呈现响应。默认情况下，所有的控制器方法都依赖于视图解析，除非使用@ responsebody或显式返回ResponseEntity。视图可以显式地指定为字符串返回值或隐式，例如void返回值。

## 类

### org.springframework.web.reactive.config.CorsRegistration

协助创建映射到路径模式的CorsConfiguration实例。默认情况下，当最大时间设置为30分钟时，允许GET，HEAD和POST请求的所有来源，标题和凭证。

#### 构造方法：

> CorsRegistration(java.lang.String pathPattern)
> 创建一个新的CorsRegistration，允许指定路径的最大时间设置为1800秒（30分钟）的GET，HEAD和POST请求的所有来源，标题和凭证。
> CORS配置应该适用的路径;支持精确的路径映射URI（如“/ admin”）以及Ant样式的路径模式（如“/ admin / **”）。

#### 方法：

- allowCredentials(boolean allowCredentials) 是否支持用户凭据。
- allowedHeaders(java.lang.String... headers) 设置飞行前请求可以在实际请求中使用的标题列表。如果它是一个:cache - control、content - language、Expires、last - modified或Pragma，根据CORS规范，则不需要列出header名称。
  默认情况下，所有标题都是允许的。
- allowedMethods(java.lang.String... methods) 设置允许的HTTP方法
- allowedOrigins(java.lang.String... origins) 设置允许的来源 特殊值“*”允许所有域，默认情况下所有的来源都是允许的。
- exposedHeaders(java.lang.String... headers) 设置“简单”标题以外的响应标题列表
- getCorsConfiguration()
- getPathPattern()
- maxAge(long maxAge) 配置客户端可以缓存飞行前请求响应的时间，以秒为单位。

### org.springframework.web.reactive.config.CorsRegistry

CorsRegistry协助注册CorsConfiguration映射到路径模式。
为指定的路径模式启用跨域请求处理

#### 方法：

- addMapping(java.lang.String pathPattern) 为指定的路径模式启用跨域请求处理。支持精确的路径映射URI（如“/ admin”）以及Ant样式的路径模式（如“/ admin / **”）。默认情况下，允许所有来源，所有标题，凭据和GET，HEAD和POST方法，并且最大时间设置为30分钟。
- protected java.util.Map<java.lang.String,CorsConfiguration> getCorsConfigurations()

```Java
    private final List<CorsRegistration> registrations = new ArrayList<>();

    public CorsRegistration addMapping(String pathPattern) {
        CorsRegistration registration = new CorsRegistration(pathPattern);
        this.registrations.add(registration);
        return registration;
    }



    protected Map<String, CorsConfiguration> getCorsConfigurations() {
        Map<String, CorsConfiguration> configs = new LinkedHashMap<>(this.registrations.size());
        for (CorsRegistration registration : this.registrations) {
            configs.put(registration.getPathPattern(), registration.getCorsConfiguration());
        }
        return configs;
    }

```

### org.springframework.web.reactive.config.PathMatchConfigurer

协助配置HandlerMapping的路径匹配选项。

#### 方法

- **isUseCaseSensitiveMatch()**
- **isUseTrailingSlashMatch()**  是否与URL匹配，而不管是否存在尾部斜线。如果启用，映射到“/ users”的方法也匹配“/ users /”。 默认值是true。
- setUseCaseSensitiveMatch(java.lang.Boolean caseSensitiveMatch) 是否与网址匹配，而不考虑其情况。
- setUseTrailingSlashMatch(java.lang.Boolean trailingSlashMatch) 是否与URL匹配，而不管是否以斜杠结尾。

```Java
    @Nullable
    private Boolean trailingSlashMatch;


    @Nullable
    private Boolean caseSensitiveMatch;



    public PathMatchConfigurer setUseCaseSensitiveMatch(Boolean caseSensitiveMatch) {
        this.caseSensitiveMatch = caseSensitiveMatch;
        return this;
    }

    public PathMatchConfigurer setUseTrailingSlashMatch(Boolean trailingSlashMatch) {
        this.trailingSlashMatch = trailingSlashMatch;
        return this;
    }

    @Nullable
    protected Boolean isUseTrailingSlashMatch() {
        return this.trailingSlashMatch;
    }

    @Nullable
    protected Boolean isUseCaseSensitiveMatch() {
        return this.caseSensitiveMatch;
    }

```

### org.springframework.web.reactive.config.ResourceChainRegistration

协助完成资源和转换器的注册

#### 构造器

- ResourceChainRegistration(boolean cacheResources)
- ResourceChainRegistration(boolean cacheResources)

#### 方法

- addResolver(ResourceResolver resolver) 将一个资源解析器添加到链中。
- addTransformer(ResourceTransformer transformer) 向链中添加资源转换器器。
- getResourceResolvers()
- getResourceTransformers()

```
    private static final String DEFAULT_CACHE_NAME = "spring-resource-chain-cache";

    private static final boolean isWebJarsAssetLocatorPresent = ClassUtils.isPresent(
            "org.webjars.WebJarAssetLocator", ResourceChainRegistration.class.getClassLoader());


    private final List<ResourceResolver> resolvers = new ArrayList<>(4);

    private final List<ResourceTransformer> transformers = new ArrayList<>(4);

    private boolean hasVersionResolver;

    private boolean hasPathResolver;

    private boolean hasCssLinkTransformer;

    private boolean hasWebjarsResolver;

        public ResourceChainRegistration(boolean cacheResources) {
        this(cacheResources, cacheResources ? new ConcurrentMapCache(DEFAULT_CACHE_NAME) : null);
    }

    public ResourceChainRegistration(boolean cacheResources, @Nullable Cache cache) {
        Assert.isTrue(!cacheResources || cache != null, "'cache' is required when cacheResources=true");
        if (cacheResources) {
            this.resolvers.add(new CachingResourceResolver(cache));
            this.transformers.add(new CachingResourceTransformer(cache));
        }
    }



    public ResourceChainRegistration addResolver(ResourceResolver resolver) {
        Assert.notNull(resolver, "The provided ResourceResolver should not be null");
        this.resolvers.add(resolver);
        if (resolver instanceof VersionResourceResolver) {
            this.hasVersionResolver = true;
        }
        else if (resolver instanceof PathResourceResolver) {
            this.hasPathResolver = true;
        }
        else if (resolver instanceof WebJarsResourceResolver) {
            this.hasWebjarsResolver = true;
        }
        return this;
    }



    public ResourceChainRegistration addTransformer(ResourceTransformer transformer) {
        Assert.notNull(transformer, "The provided ResourceTransformer should not be null");
        this.transformers.add(transformer);
        if (transformer instanceof CssLinkResourceTransformer) {
            this.hasCssLinkTransformer = true;
        }
        return this;
    }



    protected List<ResourceResolver> getResourceResolvers() {
        if (!this.hasPathResolver) {
            List<ResourceResolver> result = new ArrayList<>(this.resolvers);
            if (isWebJarsAssetLocatorPresent && !this.hasWebjarsResolver) {
                result.add(new WebJarsResourceResolver());
            }
            result.add(new PathResourceResolver());
            return result;
        }
        return this.resolvers;
    }

    protected List<ResourceTransformer> getResourceTransformers() {
        if (this.hasVersionResolver && !this.hasCssLinkTransformer) {
            List<ResourceTransformer> result = new ArrayList<>(this.transformers);
            boolean hasTransformers = !this.transformers.isEmpty();
            boolean hasCaching = hasTransformers && this.transformers.get(0) instanceof CachingResourceTransformer;
            result.add(hasCaching ? 1 : 0, new CssLinkResourceTransformer());
            return result;
        }
        return this.transformers;
    }

```

### org.springframework.web.reactive.config.ResourceHandlerRegistry

通过Spring WebFlux存储资源处理程序的注册，以提供静态资源（如图像，css文件和其他），包括设置优化的高速缓存头，以便在Web浏览器中进行高效加载。资源可以从Web应用程序根目录下的位置，类路径和其他位置提供。

要创建资源处理程序，请使用addResourceHandler（String ...）提供应为其调用处理程序以提供静态资源（例如“/ resources / **”）的URL路径模式。

然后在返回的ResourceHandlerRegistration上使用其他方法来添加一个或多个从（例如{“/”，“classpath：/ META-INF / public-web-resources /”}）静态内容的位置，或者指定一个缓存期间服务的资源。

#### 构造器

- ResourceHandlerRegistry(ResourceLoader resourceLoader) 为给定资源加载器（通常是应用程序上下文）创建新的资源处理程序注册表。
- ​

#### 方法

- addResourceHandler(java.lang.String... patterns)           添加一个资源处理程序，用于根据指定的URL路径模式提供静态资源。 处理程序将针对每个与指定路径模式匹配的传入请求进行调用。像“/static/ **”或“/css/{filename:\w+\.css}”的模式是允许的。有关语法的更多详细信息，请参阅PathPattern。
- getHandlerMapping() 返回映射资源处理程序的处理程序映射;或者在没有注册的情况下为空。
- hasMappingForPattern(java.lang.String pathPattern) 资源处理程序是否已经注册了给定的路径模式。
- setOrder(int order) 指定相对于Spring配置中配置的其他HandlerMappings进行资源处理的顺序。 使用的默认值是Integer.MAX_VALUE-1。

```Java
    private final ResourceLoader resourceLoader;

    private final List<ResourceHandlerRegistration> registrations = new ArrayList<>();

    private int order = Integer.MAX_VALUE -1;

    public ResourceHandlerRegistry(ResourceLoader resourceLoader) {
        this.resourceLoader = resourceLoader;
    }

    public ResourceHandlerRegistration addResourceHandler(String... patterns) {
        ResourceHandlerRegistration registration = new ResourceHandlerRegistration(this.resourceLoader, patterns);
        this.registrations.add(registration);
        return registration;
    }

    public boolean hasMappingForPattern(String pathPattern) {
        for (ResourceHandlerRegistration registration : this.registrations) {
            if (Arrays.asList(registration.getPathPatterns()).contains(pathPattern)) {
                return true;
            }
        }
        return false;
    }


    public ResourceHandlerRegistry setOrder(int order) {
        this.order = order;
        return this;
    }

    @Nullable
    protected AbstractUrlHandlerMapping getHandlerMapping() {
        if (this.registrations.isEmpty()) {
            return null;
        }
        Map<String, WebHandler> urlMap = new LinkedHashMap<>();
        for (ResourceHandlerRegistration registration : this.registrations) {
            for (String pathPattern : registration.getPathPatterns()) {
                ResourceWebHandler handler = registration.getRequestHandler();
                try {
                    handler.afterPropertiesSet();
                }
                catch (Throwable ex) {
                    throw new BeanInitializationException("Failed to init ResourceHttpRequestHandler", ex);
                }
                urlMap.put(pathPattern, handler);
            }
        }
        SimpleUrlHandlerMapping handlerMapping = new SimpleUrlHandlerMapping();
        handlerMapping.setOrder(this.order);
        handlerMapping.setUrlMap(urlMap);
        return handlerMapping;
    }
```

### org.springframework.web.reactive.config.ResourceHandlerRegistration

协助创建和配置静态资源处理程序。

#### 构造器

- ResourceHandlerRegistration(ResourceLoader resourceLoader, java.lang.String... pathPatterns)
  创建一个ResourceHandlerRegistration实例。
  - resourceLoader ：用于将字符串位置转换为资源的资源加载器
  - pathPatterns - 一个或多个资源URL路径模式

#### 方法

- addResourceLocations(java.lang.String... resourceLocations)
  - 添加一个或多个资源位置来服务静态内容。每个位置必须指向一个有效的目录。多个位置可以指定为逗号分隔的列表，并且位置将按照指定的顺序对给定的资源进行检查。
  - 例如，{“/”，“classpath：/ META-INF / public-web-resources /”}允许资源从Web应用程序的根目录和任何包含/ META-INF / public-web-resources /的目录，Web应用程序根目录中的资源优先。
- getPathPatterns()
- getRequestHandler()
- resourceChain(boolean cacheResources)
- resourceChain(boolean cacheResources, Cache cache)
- setCacheControl(CacheControl cacheControl) 指定应该由资源处理程序使用的CacheControl。

```Java
    private final ResourceLoader resourceLoader;

    private final String[] pathPatterns;

    private final List<Resource> locations = new ArrayList<>();

    @Nullable
    private CacheControl cacheControl;

    @Nullable
    private ResourceChainRegistration resourceChainRegistration;

    public ResourceHandlerRegistration(ResourceLoader resourceLoader, String... pathPatterns) {
        Assert.notNull(resourceLoader, "ResourceLoader is required");
        Assert.notEmpty(pathPatterns, "At least one path pattern is required for resource handling");
        this.resourceLoader = resourceLoader;
        this.pathPatterns = pathPatterns;
    }

    public ResourceHandlerRegistration addResourceLocations(String... resourceLocations) {
        for (String location : resourceLocations) {
            this.locations.add(this.resourceLoader.getResource(location));
        }
        return this;
    }



    public ResourceHandlerRegistration setCacheControl(CacheControl cacheControl) {
        this.cacheControl = cacheControl;
        return this;
    }

    public ResourceChainRegistration resourceChain(boolean cacheResources) {
        this.resourceChainRegistration = new ResourceChainRegistration(cacheResources);
        return this.resourceChainRegistration;
    }

    public ResourceChainRegistration resourceChain(boolean cacheResources, Cache cache) {
        this.resourceChainRegistration = new ResourceChainRegistration(cacheResources, cache);
        return this.resourceChainRegistration;
    }


    protected String[] getPathPatterns() {
        return this.pathPatterns;
    }


    protected ResourceWebHandler getRequestHandler() {
        ResourceWebHandler handler = new ResourceWebHandler();
        if (this.resourceChainRegistration != null) {
            handler.setResourceResolvers(this.resourceChainRegistration.getResourceResolvers());
            handler.setResourceTransformers(this.resourceChainRegistration.getResourceTransformers());
        }
        handler.setLocations(this.locations);
        if (this.cacheControl != null) {
            handler.setCacheControl(this.cacheControl);
        }
        return handler;
    }

```

### org.springframework.web.reactive.config.UrlBasedViewResolverRegistration

协助配置UrlBasedViewResolver的属性。

#### 构造器

- UrlBasedViewResolverRegistration(UrlBasedViewResolver viewResolver)

#### 方法

- getViewResolver()
- prefix(java.lang.String prefix) 设置在构建URL时附加到视图名称的前缀。
- suffix(java.lang.String suffix) 设置在构建URL时附加到视图名称的后缀。
- viewClass(java.lang.Class<?> viewClass) 设置应该用于创建视图的视图类。
- viewNames(java.lang.String... viewNames) 设置可由该视图解析器处理的视图名称（或名称模式）。视图名称可以包含简单的通配符，这样'my *'，'* Report'和'* Repo *'将全部匹配视图名称'myReport'。
- ​

```Java
    private final UrlBasedViewResolver viewResolver;


    public UrlBasedViewResolverRegistration(UrlBasedViewResolver viewResolver) {
        Assert.notNull(viewResolver, "ViewResolver must not be null");
        this.viewResolver = viewResolver;
    }

    public UrlBasedViewResolverRegistration prefix(String prefix) {
        this.viewResolver.setPrefix(prefix);
        return this;
    }

    public UrlBasedViewResolverRegistration suffix(String suffix) {
        this.viewResolver.setSuffix(suffix);
        return this;
    }


    public UrlBasedViewResolverRegistration viewClass(Class<?> viewClass) {
        this.viewResolver.setViewClass(viewClass);
        return this;
    }


    public UrlBasedViewResolverRegistration viewNames(String... viewNames) {
        this.viewResolver.setViewNames(viewNames);
        return this;
    }

    protected UrlBasedViewResolver getViewResolver() {
        return this.viewResolver;
    }


```

### org.springframework.web.reactive.config.ViewResolverRegistry

协助配置一个ViewResolver的链，支持不同的模板机制。另外，还可以根据所请求的内容类型配置defaultView以进行渲染，例如， JSON，XML等

#### 构造器

- ViewResolverRegistry(ApplicationContext applicationContext)
- ​

#### 方法

- defaultViews(View... defaultViews)                  设置与任何视图名称关联的默认视图，并根据请求的内容类型的最佳匹配进行选择。使用HttpMessageWriterView来调整和使用任何现有的HttpMessageWriter（例如JSON，XML）作为一个视图。
- freeMarker() 注册一个带有“.ftl”后缀的FreeMarkerViewResolver。
- getDefaultViews()
- getOrder()
- getViewResolvers()
- hasRegistrations() 是否有任何视图解析器已被注册。
- order(int order) 设置ViewResolutionResultHandler的顺序。 默认情况下，此属性未设置，这意味着结果处理程序将按顺序排列。
- viewResolver(ViewResolver viewResolver) 注册一个ViewResolver bean实例。这可能有助于配置第三方解析器实现，或者在这个类中不公开一些需要设置的高级属性时，可以替代其他注册方法。
- ​

```Java
    @Nullable
    private final ApplicationContext applicationContext;

    private final List<ViewResolver> viewResolvers = new ArrayList<>(4);

    private final List<View> defaultViews = new ArrayList<>(4);


    @Nullable
    private Integer order;

    public ViewResolverRegistry(@Nullable ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }

    public UrlBasedViewResolverRegistration freeMarker() {
        if (!checkBeanOfType(FreeMarkerConfigurer.class)) {
            throw new BeanInitializationException("In addition to a FreeMarker view resolver " +
                    "there must also be a single FreeMarkerConfig bean in this web application context " +
                    "(or its parent): FreeMarkerConfigurer is the usual implementation. " +
                    "This bean may be given any name.");
        }
        FreeMarkerRegistration registration = new FreeMarkerRegistration();
        UrlBasedViewResolver resolver = registration.getViewResolver();
        if (this.applicationContext != null) {
            resolver.setApplicationContext(this.applicationContext);
        }
        this.viewResolvers.add(resolver);
        return registration;
    }

    public void viewResolver(ViewResolver viewResolver) {
        this.viewResolvers.add(viewResolver);
    }

    public void defaultViews(View... defaultViews) {
        this.defaultViews.addAll(Arrays.asList(defaultViews));
    }


    public boolean hasRegistrations() {
        return (!this.viewResolvers.isEmpty());
    }

    public void order(int order) {
        this.order = order;
    }

    private boolean checkBeanOfType(Class<?> beanType) {
        return (this.applicationContext == null ||
                !ObjectUtils.isEmpty(BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
                        this.applicationContext, beanType, false, false)));
    }

    protected int getOrder() {
        return (this.order != null ? this.order : Ordered.LOWEST_PRECEDENCE);
    }

    protected List<ViewResolver> getViewResolvers() {
        return this.viewResolvers;
    }

    protected List<View> getDefaultViews() {
        return this.defaultViews;
    }


    private static class FreeMarkerRegistration extends UrlBasedViewResolverRegistration {

        public FreeMarkerRegistration() {
            super(new FreeMarkerViewResolver());
            getViewResolver().setSuffix(".ftl");
        }
    }


```

### org.springframework.web.reactive.config.WebFluxConfigurationSupport

#### 所有实现的接口

`Aware`, `ApplicationContextAware`

Spring WebFlux配置的主要类。直接导入或扩展和重写受保护的方法来自定义。

```
@Nullable
    private Map<String, CorsConfiguration> corsConfigurations;

    @Nullable
    private PathMatchConfigurer pathMatchConfigurer;

    @Nullable
    private ViewResolverRegistry viewResolverRegistry;

    @Nullable
    private ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(@Nullable ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }

    @Nullable
    public final ApplicationContext getApplicationContext() {
        return this.applicationContext;
    }


    @Bean
    public DispatcherHandler webHandler() {
        return new DispatcherHandler();
    }

    @Bean
    @Order(0)
    public WebExceptionHandler responseStatusExceptionHandler() {
        return new ResponseStatusExceptionHandler();
    }

    @Bean
    public RequestMappingHandlerMapping requestMappingHandlerMapping() {
        RequestMappingHandlerMapping mapping = createRequestMappingHandlerMapping();
        mapping.setOrder(0);
        mapping.setContentTypeResolver(webFluxContentTypeResolver());
        mapping.setCorsConfigurations(getCorsConfigurations());

        PathMatchConfigurer configurer = getPathMatchConfigurer();
        Boolean useTrailingSlashMatch = configurer.isUseTrailingSlashMatch();
        Boolean useCaseSensitiveMatch = configurer.isUseCaseSensitiveMatch();
        if (useTrailingSlashMatch != null) {
            mapping.setUseTrailingSlashMatch(useTrailingSlashMatch);
        }
        if (useCaseSensitiveMatch != null) {
            mapping.setUseCaseSensitiveMatch(useCaseSensitiveMatch);
        }
        return mapping;
    }

    protected RequestMappingHandlerMapping createRequestMappingHandlerMapping() {
        return new RequestMappingHandlerMapping();
    }

    @Bean
    public RequestedContentTypeResolver webFluxContentTypeResolver() {
        RequestedContentTypeResolverBuilder builder = new RequestedContentTypeResolverBuilder();
        configureContentTypeResolver(builder);
        return builder.build();
    }
    protected void configureContentTypeResolver(RequestedContentTypeResolverBuilder builder) {
    }



    protected final Map<String, CorsConfiguration> getCorsConfigurations() {
        if (this.corsConfigurations == null) {
            CorsRegistry registry = new CorsRegistry();
            addCorsMappings(registry);
            this.corsConfigurations = registry.getCorsConfigurations();
        }
        return this.corsConfigurations;
    }

    protected void addCorsMappings(CorsRegistry registry) {
    }

    protected final PathMatchConfigurer getPathMatchConfigurer() {
        if (this.pathMatchConfigurer == null) {
            this.pathMatchConfigurer = new PathMatchConfigurer();
            configurePathMatching(this.pathMatchConfigurer);
        }
        return this.pathMatchConfigurer;
    }

    public void configurePathMatching(PathMatchConfigurer configurer) {
    }

    @Bean
    public RouterFunctionMapping routerFunctionMapping() {
        RouterFunctionMapping mapping = createRouterFunctionMapping();
        mapping.setOrder(-1); // go before RequestMappingHandlerMapping
        mapping.setMessageReaders(serverCodecConfigurer().getReaders());
        mapping.setCorsConfigurations(getCorsConfigurations());

        return mapping;
    }

    @Bean
    public HandlerMapping resourceHandlerMapping() {
        ResourceLoader resourceLoader = this.applicationContext;
        if (resourceLoader == null) {
            resourceLoader = new DefaultResourceLoader();
        }
        ResourceHandlerRegistry registry = new ResourceHandlerRegistry(resourceLoader);
        addResourceHandlers(registry);

        AbstractHandlerMapping handlerMapping = registry.getHandlerMapping();
        if (handlerMapping != null) {
            PathMatchConfigurer configurer = getPathMatchConfigurer();
            Boolean useTrailingSlashMatch = configurer.isUseTrailingSlashMatch();
            Boolean useCaseSensitiveMatch = configurer.isUseCaseSensitiveMatch();
            if (useTrailingSlashMatch != null) {
                handlerMapping.setUseTrailingSlashMatch(useTrailingSlashMatch);
            }
            if (useCaseSensitiveMatch != null) {
                handlerMapping.setUseCaseSensitiveMatch(useCaseSensitiveMatch);
            }
        }
        else {
            handlerMapping = new EmptyHandlerMapping();
        }
        return handlerMapping;
    }

    ...

```

### org.springframework.web.reactive.config.WebFluxConfigurerComposite

#### 所有实现的接口

`WebFluxConfigurer`

将WebFluxConfigurer委托给一个或多个。

```Java
    public class WebFluxConfigurerComposite implements WebFluxConfigurer {

    private final List<WebFluxConfigurer> delegates = new ArrayList<>();


    public void addWebFluxConfigurers(List<WebFluxConfigurer> configurers) {
        if (!CollectionUtils.isEmpty(configurers)) {
            this.delegates.addAll(configurers);
        }
    }


    @Override
    public void configureContentTypeResolver(RequestedContentTypeResolverBuilder builder) {
        this.delegates.forEach(delegate -> delegate.configureContentTypeResolver(builder));
    }

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        this.delegates.forEach(delegate -> delegate.addCorsMappings(registry));
    }

    @Override
    public void configurePathMatching(PathMatchConfigurer configurer) {
        this.delegates.forEach(delegate -> delegate.configurePathMatching(configurer));
    }

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        this.delegates.forEach(delegate -> delegate.addResourceHandlers(registry));
    }

    @Override
    public void configureArgumentResolvers(ArgumentResolverConfigurer configurer) {
        this.delegates.forEach(delegate -> delegate.configureArgumentResolvers(configurer));
    }

    @Override
    public void configureHttpMessageCodecs(ServerCodecConfigurer configurer) {
        this.delegates.forEach(delegate -> delegate.configureHttpMessageCodecs(configurer));
    }

    @Override
    public void addFormatters(FormatterRegistry registry) {
        this.delegates.forEach(delegate -> delegate.addFormatters(registry));
    }

    @Override
    public Validator getValidator() {
        return createSingleBean(WebFluxConfigurer::getValidator, Validator.class);
    }

    @Override
    public MessageCodesResolver getMessageCodesResolver() {
        return createSingleBean(WebFluxConfigurer::getMessageCodesResolver, MessageCodesResolver.class);
    }

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        this.delegates.forEach(delegate -> delegate.configureViewResolvers(registry));
    }

    @Nullable
    private <T> T createSingleBean(Function<WebFluxConfigurer, T> factory, Class<T> beanType) {
        List<T> result = this.delegates.stream().map(factory).filter(t -> t != null).collect(Collectors.toList());
        if (result.isEmpty()) {
            return null;
        }
        else if (result.size() == 1) {
            return result.get(0);
        }
        else {
            throw new IllegalStateException("More than one WebFluxConfigurer implements " +
                    beanType.getSimpleName() + " factory method.");
        }
    }

}
```

### org.springframework.web.reactive.config.DelegatingWebFluxConfiguration

- java.lang.Object
  - org.springframework.web.reactive.config.WebFluxConfigurationSupport
    - org.springframework.web.reactive.config.DelegatingWebFluxConfiguration

WebFluxConfigurationSupport的一个子类，用于检测并委托所有类型为WebFluxConfigurer的bean，允许它们自定义由WebFluxConfigurationSupport提供的配置。这是由@EnableWebFlux实际导入的类。

```Java
@Configuration
public class DelegatingWebFluxConfiguration extends WebFluxConfigurationSupport {

    private final WebFluxConfigurerComposite configurers = new WebFluxConfigurerComposite();

    @Autowired(required = false)
    public void setConfigurers(List<WebFluxConfigurer> configurers) {
        if (!CollectionUtils.isEmpty(configurers)) {
            this.configurers.addWebFluxConfigurers(configurers);
        }
    }

    @Override
    protected void configureContentTypeResolver(RequestedContentTypeResolverBuilder builder) {
        this.configurers.configureContentTypeResolver(builder);
    }

    @Override
    protected void addCorsMappings(CorsRegistry registry) {
        this.configurers.addCorsMappings(registry);
    }

    @Override
    public void configurePathMatching(PathMatchConfigurer configurer) {
        this.configurers.configurePathMatching(configurer);
    }

    @Override
    protected void addResourceHandlers(ResourceHandlerRegistry registry) {
        this.configurers.addResourceHandlers(registry);
    }

    @Override
    protected void configureArgumentResolvers(ArgumentResolverConfigurer configurer) {
        this.configurers.configureArgumentResolvers(configurer);
    }

    @Override
    protected void configureHttpMessageCodecs(ServerCodecConfigurer configurer) {
        this.configurers.configureHttpMessageCodecs(configurer);
    }

    @Override
    protected void addFormatters(FormatterRegistry registry) {
        this.configurers.addFormatters(registry);
    }

    @Override
    protected Validator getValidator() {
        Validator validator = this.configurers.getValidator();
        return (validator != null ? validator : super.getValidator());
    }

    @Override
    protected MessageCodesResolver getMessageCodesResolver() {
        MessageCodesResolver messageCodesResolver = this.configurers.getMessageCodesResolver();
        return (messageCodesResolver != null ? messageCodesResolver : super.getMessageCodesResolver());
    }

    @Override
    protected void configureViewResolvers(ViewResolverRegistry registry) {
        this.configurers.configureViewResolvers(registry);
    }
}
```

# 666. 彩蛋

如果你对 Spring Webflux 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)