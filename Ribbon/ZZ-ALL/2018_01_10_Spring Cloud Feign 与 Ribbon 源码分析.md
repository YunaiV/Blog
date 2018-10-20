title: Spring Cloud Feign 与 Ribbon 源码分析
date: 2018-01-10
tag: 
categories: Ribbon
permalink: Ribbon/laoyuan/spring-cloud-feign-and-ribbon
author: 老袁
from_url: https://laoyuan.me/posts/spring-cloud-feign-and-ribbon.html
wechat_url: 

-------

摘要: 原创出处 https://laoyuan.me/posts/spring-cloud-feign-and-ribbon.html 「老袁」欢迎转载，保留摘要，谢谢！

- [Feign与Ribbon介绍](http://www.iocoder.cn/Ribbon/laoyuan/spring-cloud-feign-and-ribbon/)
- [使用Feign与Ribbon](http://www.iocoder.cn/Ribbon/laoyuan/spring-cloud-feign-and-ribbon/)
- [源码解析](http://www.iocoder.cn/Ribbon/laoyuan/spring-cloud-feign-and-ribbon/)
- [配置与优化](http://www.iocoder.cn/Ribbon/laoyuan/spring-cloud-feign-and-ribbon/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

## Feign与Ribbon介绍

Feign是一款客户端HTTP调用组件，用于简化目前Rest接口调用操作，可以很方便的使调用HTTP接口像方法调用一样简单。
Rbbion是一款客户端负载均衡组件，提供了容易扩展的负载均衡策略。
Spring基于Netflix开源的以上组件做了大量的封装，可以很方便的和Spirng应用结合用于微服务之间的相互调用。

## 使用Feign与Ribbon

在Spring Cloud应用中使用Feign组件，首先需要在依赖包中加入以下依赖：

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-feign</artifactId>
</dependency>
```

再定义一个可供调用的Feign组件示例如下：

```java
@FeignClient("servicename")
public interface IService {
	/**
	 * 远程调用，直接生成Token
	 *
	 * @param param
	 *            Token生成需要的参数
	 *
	 * @return 加密后的Token
	 */
	@RequestMapping(value = "remoting/security/token", method = RequestMethod.POST)
	public ApiResult<Object> generateToken(
			@RequestBody @Validated(GenerateTokenParam.GenerateToken.class) GenerateTokenParam param);

	/**
	 * 远程调用，验证Token，验证通过后会返回TOKEN中携带的数据
	 *
	 * @param param
	 *            TOEKN字符串
	 *
	 * @return Token中携带的数据
	 */
	@RequestMapping(value = "remoting/security/verify", method = RequestMethod.POST)
	public ApiResult<Object> verifyToken(@RequestBody @Validated VerifyTokenParam param);
}
```

上述代码中，每一个方法都代表通过Feign请求的一个接口，`@RequestMapping`指定请求地址和请求方法。`@FeignClient`则用于指定调用的微服务，结合Rbbion在注册中心注册的服务列表中选择一个合适的服务地址。
目前对FeignClient的应用主要有2种方式：

- 服务提供方定义好Feign组件，自己的Controller实现定义好的Feign组件接口，然后把Feign组件打包成SDK提供给调用方，这样的好处是便于后期服务提供方统一升级组件，比如更换调用路径和参数等；
- 服务调用方自己封装Feign组件，这样的好处是调用方可以灵活的自定义Feign组件；

> 注意：目前Feign调用对基本类型参数传值支持不是很好，所以参数最好封装成一个DTO对象进行传输。如果是JSON参数，最好再定义Feign组件时加上`@RequestBody`注解。

启用Feign组件是使用Feign的第一步，使用如下注解开启：

```java
//启用Feign组件并配置扫描包路径
@EnableFeignClients(basePackages = { "com.xxx.service.api", "com.ooo.sdk.service" })
```

接下来就是在需要调用接口的地方用Spring Bean一样调用其他服务了，示例如下：

```
@Autowired
private IService service;
```

## 源码解析

### Feign Bean创建

Feign组件初始化是从`@EnableFeignClients`注解开始的，注解源码如下：

```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {

	/**
	 * Alias for the {@link #basePackages()} attribute. Allows for more concise annotation
	 * declarations e.g.: {@code @ComponentScan("org.my.pkg")} instead of
	 * {@code @ComponentScan(basePackages="org.my.pkg")}.
	 * @return the array of 'basePackages'.
	 */
	String[] value() default {};

	/**
	 * Base packages to scan for annotated components.
	 * <p>
	 * {@link #value()} is an alias for (and mutually exclusive with) this attribute.
	 * <p>
	 * Use {@link #basePackageClasses()} for a type-safe alternative to String-based
	 * package names.
	 *
	 * @return the array of 'basePackages'.
	 */
	String[] basePackages() default {};

	/**
	 * Type-safe alternative to {@link #basePackages()} for specifying the packages to
	 * scan for annotated components. The package of each class specified will be scanned.
	 * <p>
	 * Consider creating a special no-op marker class or interface in each package that
	 * serves no purpose other than being referenced by this attribute.
	 *
	 * @return the array of 'basePackageClasses'.
	 */
	Class<?>[] basePackageClasses() default {};

	/**
	 * A custom <code>@Configuration</code> for all feign clients. Can contain override
	 * <code>@Bean</code> definition for the pieces that make up the client, for instance
	 * {@link feign.codec.Decoder}, {@link feign.codec.Encoder}, {@link feign.Contract}.
	 *
	 * @see FeignClientsConfiguration for the defaults
	 */
	Class<?>[] defaultConfiguration() default {};

	/**
	 * List of classes annotated with @FeignClient. If not empty, disables classpath scanning.
	 * @return
	 */
	Class<?>[] clients() default {};
}
```

核心有2个方法，`basePackages`与`defaultConfiguration`，前者用于定义扫描包路径，后者用于定义`@FeignClient`组件的配置类，在配置类中可以自己定义Feign请求的`Decoder`解码器、`Encoder`编码器、`Contract`组件扫描构造器。
在注解上有一个关键注解`@Import(FeignClientsRegistrar.class)`，导入了Feign组件的注册器，用于扫描Feign组件与初始化Feign组件的Bean定义信息，各阶段作用建下述源码注释。

```java
class FeignClientsRegistrar implements ImportBeanDefinitionRegistrar,
		ResourceLoaderAware, BeanClassLoaderAware, EnvironmentAware {

	// patterned after Spring Integration IntegrationComponentScanRegistrar
	// and RibbonClientsConfigurationRegistgrar

	private ResourceLoader resourceLoader;

	private ClassLoader classLoader;

	private Environment environment;

	public FeignClientsRegistrar() {
	}

	@Override
	public void setResourceLoader(ResourceLoader resourceLoader) {
		this.resourceLoader = resourceLoader;
	}

	@Override
	public void setBeanClassLoader(ClassLoader classLoader) {
		this.classLoader = classLoader;
	}

	@Override
	public void registerBeanDefinitions(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
		//定义配置类
		registerDefaultConfiguration(metadata, registry);
		registerFeignClients(metadata, registry);
	}

	private void registerDefaultConfiguration(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
		Map<String, Object> defaultAttrs = metadata
				.getAnnotationAttributes(EnableFeignClients.class.getName(), true);
		//如果存在自定义配置则定义配置Bean
		if (defaultAttrs != null && defaultAttrs.containsKey("defaultConfiguration")) {
			String name;
			if (metadata.hasEnclosingClass()) {
				name = "default." + metadata.getEnclosingClassName();
			}
			else {
				name = "default." + metadata.getClassName();
			}
			registerClientConfiguration(registry, name,
					defaultAttrs.get("defaultConfiguration"));
		}
	}

	public void registerFeignClients(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
		ClassPathScanningCandidateComponentProvider scanner = getScanner();
		scanner.setResourceLoader(this.resourceLoader);

		Set<String> basePackages;

		Map<String, Object> attrs = metadata
				.getAnnotationAttributes(EnableFeignClients.class.getName());
		//指定扫描类注解类型为@FeignClient的类
		AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(
				FeignClient.class);
		final Class<?>[] clients = attrs == null ? null
				: (Class<?>[]) attrs.get("clients");
		if (clients == null || clients.length == 0) {
			scanner.addIncludeFilter(annotationTypeFilter);
			basePackages = getBasePackages(metadata);
		}
		else {
			final Set<String> clientClasses = new HashSet<>();
			basePackages = new HashSet<>();
			for (Class<?> clazz : clients) {
				basePackages.add(ClassUtils.getPackageName(clazz));
				clientClasses.add(clazz.getCanonicalName());
			}
			AbstractClassTestingTypeFilter filter = new AbstractClassTestingTypeFilter() {
				@Override
				protected boolean match(ClassMetadata metadata) {
					String cleaned = metadata.getClassName().replaceAll("\\$", ".");
					return clientClasses.contains(cleaned);
				}
			};
			scanner.addIncludeFilter(
					new AllTypeFilter(Arrays.asList(filter, annotationTypeFilter)));
		}
		//扫描定义的Feign组件包路径
		for (String basePackage : basePackages) {
			Set<BeanDefinition> candidateComponents = scanner
					.findCandidateComponents(basePackage);
			for (BeanDefinition candidateComponent : candidateComponents) {
				if (candidateComponent instanceof AnnotatedBeanDefinition) {
					// verify annotated class is an interface
					AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
					AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
					Assert.isTrue(annotationMetadata.isInterface(),
							"@FeignClient can only be specified on an interface");

					Map<String, Object> attributes = annotationMetadata
							.getAnnotationAttributes(
									FeignClient.class.getCanonicalName());

					String name = getClientName(attributes);
					registerClientConfiguration(registry, name,
							attributes.get("configuration"));

					registerFeignClient(registry, annotationMetadata, attributes);
				}
			}
		}
	}

	private void registerFeignClient(BeanDefinitionRegistry registry,
			AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
		String className = annotationMetadata.getClassName();
		//定义Feign组件的创建工厂FeignClientFactoryBean
		BeanDefinitionBuilder definition = BeanDefinitionBuilder
				.genericBeanDefinition(FeignClientFactoryBean.class);
		validate(attributes);
		definition.addPropertyValue("url", getUrl(attributes));
		definition.addPropertyValue("path", getPath(attributes));
		String name = getName(attributes);
		definition.addPropertyValue("name", name);
		definition.addPropertyValue("type", className);
		definition.addPropertyValue("decode404", attributes.get("decode404"));
		definition.addPropertyValue("fallback", attributes.get("fallback"));
		definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
		definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);

		String alias = name + "FeignClient";
		AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();

		boolean primary = (Boolean)attributes.get("primary"); // has a default, won't be null

		beanDefinition.setPrimary(primary);

		String qualifier = getQualifier(attributes);
		if (StringUtils.hasText(qualifier)) {
			alias = qualifier;
		}

		BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
				new String[] { alias });
		BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
	}
}
```

核心代码实际可以只关注以下片段，定义好了创建Feign组件Bean的FactoryBean：

```java
//定义Feign组件的创建工厂
BeanDefinitionBuilder definition = BeanDefinitionBuilder.genericBeanDefinition(FeignClientFactoryBean.class);
validate(attributes);
```

接下来可以查看Feign代理Bean实例是如何创建的的，参见FeignClientFactoryBean源码：

```java
@Data
@EqualsAndHashCode(callSuper = false)
class FeignClientFactoryBean implements FactoryBean<Object>, InitializingBean,
		ApplicationContextAware {
	/***********************************
	 * WARNING! Nothing in this class should be @Autowired. It causes NPEs because of some lifecycle race condition.
	 ***********************************/

	private Class<?> type;

	private String name;

	private String url;

	private String path;

	private boolean decode404;

	private ApplicationContext applicationContext;

	private Class<?> fallback = void.class;

	private Class<?> fallbackFactory = void.class;

	@Override
	public void afterPropertiesSet() throws Exception {
		Assert.hasText(this.name, "Name must be set");
	}


	@Override
	public void setApplicationContext(ApplicationContext context) throws BeansException {
		this.applicationContext = context;
	}

	protected Feign.Builder feign(FeignContext context) {
		FeignLoggerFactory loggerFactory = get(context, FeignLoggerFactory.class);
		Logger logger = loggerFactory.create(this.type);

		// @formatter:off
		//FeignContext继承自BeanFactoty，所以可以用于获取Bean
		//1、builder使用的Encoder、Decoder、Contract都来自FeignClientsConfiguration自动配置类中定义
		Feign.Builder builder = get(context, Feign.Builder.class)
				// required values
				.logger(logger)
				.encoder(get(context, Encoder.class))
				.decoder(get(context, Decoder.class))
				.contract(get(context, Contract.class));
		// @formatter:on

		// optional values
		Logger.Level level = getOptional(context, Logger.Level.class);
		if (level != null) {
			builder.logLevel(level);
		}
		Retryer retryer = getOptional(context, Retryer.class);
		if (retryer != null) {
			builder.retryer(retryer);
		}
		ErrorDecoder errorDecoder = getOptional(context, ErrorDecoder.class);
		if (errorDecoder != null) {
			builder.errorDecoder(errorDecoder);
		}
		Request.Options options = getOptional(context, Request.Options.class);
		if (options != null) {
			builder.options(options);
		}
		//2、装入Feign请求的拦截器
		Map<String, RequestInterceptor> requestInterceptors = context.getInstances(
				this.name, RequestInterceptor.class);
		if (requestInterceptors != null) {
			builder.requestInterceptors(requestInterceptors.values());
		}

		if (decode404) {
			builder.decode404();
		}

		return builder;
	}

	protected <T> T get(FeignContext context, Class<T> type) {
		T instance = context.getInstance(this.name, type);
		if (instance == null) {
			throw new IllegalStateException("No bean found of type " + type + " for "
					+ this.name);
		}
		return instance;
	}

	protected <T> T getOptional(FeignContext context, Class<T> type) {
		return context.getInstance(this.name, type);
	}

	protected <T> T loadBalance(Feign.Builder builder, FeignContext context,
			HardCodedTarget<T> target) {
		Client client = getOptional(context, Client.class);
		if (client != null) {
			builder.client(client);
			Targeter targeter = get(context, Targeter.class);
			return targeter.target(this, builder, context, target);
		}

		throw new IllegalStateException(
				"No Feign Client for loadBalancing defined. Did you forget to include spring-cloud-starter-ribbon?");
	}

	@Override
	public Object getObject() throws Exception {
		FeignContext context = applicationContext.getBean(FeignContext.class);
		//从Spring Context中获取到Feign的Builder
		Feign.Builder builder = feign(context);
		//@FeignClient注解没有配置URL属性
		if (!StringUtils.hasText(this.url)) {
			String url;
			if (!this.name.startsWith("http")) {
				url = "http://" + this.name;
			}
			else {
				url = this.name;
			}
			url += cleanPath();
			return loadBalance(builder, context, new HardCodedTarget<>(this.type,
					this.name, url));
		}
		//处理@FeignClient URL属性(主机名)存在的情况
		if (StringUtils.hasText(this.url) && !this.url.startsWith("http")) {
			this.url = "http://" + this.url;
		}
		String url = this.url + cleanPath();
		//获取到调用客户端：Spring封装了基于Ribbon的客户端（LoadBalancerFeignClient）
		//1、Feign自己封装的Request（基于java.net原生），2、OkHttpClient（新一代/HTTP2），3、ApacheHttpClient（常规）
		Client client = getOptional(context, Client.class);
		if (client != null) {
			if (client instanceof LoadBalancerFeignClient) {
				// not lod balancing because we have a url,
				// but ribbon is on the classpath, so unwrap
				client = ((LoadBalancerFeignClient)client).getDelegate();
			}
			//设置调用客户端
			builder.client(client);
		}
		//DefaultTargeter或者HystrixTargeter，其中HystrixTargeter带熔断和降级功能
		//主要用户在Builder中配置调用失败回调方法
		Targeter targeter = get(context, Targeter.class);
		//Bean创建实际目标封装，最终生成InvocationHandler
		return targeter.target(this, builder, context, new HardCodedTarget<>(
				this.type, this.name, url));
	}

	private String cleanPath() {
		String path = this.path.trim();
		if (StringUtils.hasLength(path)) {
			if (!path.startsWith("/")) {
				path = "/" + path;
			}
			if (path.endsWith("/")) {
				path = path.substring(0, path.length() - 1);
			}
		}
		return path;
	}

	@Override
	public Class<?> getObjectType() {
		return this.type;
	}

	@Override
	public boolean isSingleton() {
		return true;
	}

}
```

在上述源码中，最核心的是`getObject`方法，方法中定义好了如何去初始化一个FeignClient组件，在代理Bean中织入了哪些方法，具体可以参见代码中文注释。
我们先暂时略过上述工厂Bean创建代理对象时，使用到Feign的其他组件，如：Encoder、Decoder、Contract等，后面再详细阐述。
当调用`target`方法时，实际会触发FeignBuiler组件的`newInstance`源码如下：

```java
public <T> T newInstance(Target<T> target) {
//核心方法，解析定义的@FeignClient组件中的方法和请求路径
   Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
   Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
   List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();

   for (Method method : target.type().getMethods()) {
     if (method.getDeclaringClass() == Object.class) {
       continue;
     } else if(Util.isDefault(method)) {
       DefaultMethodHandler handler = new DefaultMethodHandler(method);
       defaultMethodHandlers.add(handler);
       methodToHandler.put(method, handler);
     } else {
       methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
     }
   }
//调用工厂Bean，创建执行Handler
   InvocationHandler handler = factory.create(target, methodToHandler);
   T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(), new Class<?>[]{target.type()}, handler);

   for(DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
     defaultMethodHandler.bindTo(proxy);
   }
   return proxy;
 }
 //请求方法和路径解析器
 static final class ParseHandlersByName {

   private final Contract contract;
   private final Options options;
   private final Encoder encoder;
   private final Decoder decoder;
   private final ErrorDecoder errorDecoder;
   private final SynchronousMethodHandler.Factory factory;

   ParseHandlersByName(Contract contract, Options options, Encoder encoder, Decoder decoder,
                       ErrorDecoder errorDecoder, SynchronousMethodHandler.Factory factory) {
     this.contract = contract;
     this.options = options;
     this.factory = factory;
     this.errorDecoder = errorDecoder;
     this.encoder = checkNotNull(encoder, "encoder");
     this.decoder = checkNotNull(decoder, "decoder");
   }

   public Map<String, MethodHandler> apply(Target key) {
 //核心方法：解析@FeignClient组件，Feign自带的则解析自己的注解格式，Spring提供了解析MVC注解的SpringMvcContract
     List<MethodMetadata> metadata = contract.parseAndValidatateMetadata(key.type());
     Map<String, MethodHandler> result = new LinkedHashMap<String, MethodHandler>();
     for (MethodMetadata md : metadata) {
       BuildTemplateByResolvingArgs buildTemplate;
       if (!md.formParams().isEmpty() && md.template().bodyTemplate() == null) {
         buildTemplate = new BuildFormEncodedTemplateFromArgs(md, encoder);
       } else if (md.bodyIndex() != null) {
         buildTemplate = new BuildEncodedTemplateFromArgs(md, encoder);
       } else {
         buildTemplate = new BuildTemplateByResolvingArgs(md);
       }
       result.put(md.configKey(),
                  factory.create(key, md, buildTemplate, options, decoder, errorDecoder));
     }
     return result;
   }
 }
```

先顺着最后的创建流程看下Targeter对象，暂时叫做目标对象吧，看起来很奇怪的名字，有2个实现，一个为DefaultTargeter，源码如下:

```java
class DefaultTargeter implements Targeter {

	@Override
	public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign, FeignContext context,
						Target.HardCodedTarget<T> target) {
		//简单暴力，直接调用Feign Builer字段的设置目标对象方法
		return feign.target(target);
	}
}
```

上面的方法最终会调用Feign原生的InvocationHandlerFactory.Default工厂来创建FeignInvocationHandler:

```java
//第一部分：封装处理请求的Hander
static class FeignInvocationHandler implements InvocationHandler {

    private final Target target;
    private final Map<Method, MethodHandler> dispatch;

    FeignInvocationHandler(Target target, Map<Method, MethodHandler> dispatch) {
      this.target = checkNotNull(target, "target");
      this.dispatch = checkNotNull(dispatch, "dispatch for %s", target);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      if ("equals".equals(method.getName())) {
        try {
          Object
              otherHandler =
              args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
          return equals(otherHandler);
        } catch (IllegalArgumentException e) {
          return false;
        }
      } else if ("hashCode".equals(method.getName())) {
        return hashCode();
      } else if ("toString".equals(method.getName())) {
        return toString();
      }
      return dispatch.get(method).invoke(args);
    }

    @Override
    public boolean equals(Object obj) {
      if (obj instanceof FeignInvocationHandler) {
        FeignInvocationHandler other = (FeignInvocationHandler) obj;
        return target.equals(other.target);
      }
      return false;
    }

    @Override
    public int hashCode() {
      return target.hashCode();
    }

    @Override
    public String toString() {
      return target.toString();
    }
  }
  //第二部分：解析出来的实际处理请求的方法
  final class SynchronousMethodHandler implements MethodHandler {

  private static final long MAX_RESPONSE_BUFFER_SIZE = 8192L;

  private final MethodMetadata metadata;
  private final Target<?> target;
  private final Client client;
  private final Retryer retryer;
  private final List<RequestInterceptor> requestInterceptors;
  private final Logger logger;
  private final Logger.Level logLevel;
  private final RequestTemplate.Factory buildTemplateFromArgs;
  private final Options options;
  private final Decoder decoder;
  private final ErrorDecoder errorDecoder;
  private final boolean decode404;

  private SynchronousMethodHandler(Target<?> target, Client client, Retryer retryer,
                                   List<RequestInterceptor> requestInterceptors, Logger logger,
                                   Logger.Level logLevel, MethodMetadata metadata,
                                   RequestTemplate.Factory buildTemplateFromArgs, Options options,
                                   Decoder decoder, ErrorDecoder errorDecoder, boolean decode404) {
    this.target = checkNotNull(target, "target");
    this.client = checkNotNull(client, "client for %s", target);
    this.retryer = checkNotNull(retryer, "retryer for %s", target);
    this.requestInterceptors =
        checkNotNull(requestInterceptors, "requestInterceptors for %s", target);
    this.logger = checkNotNull(logger, "logger for %s", target);
    this.logLevel = checkNotNull(logLevel, "logLevel for %s", target);
    this.metadata = checkNotNull(metadata, "metadata for %s", target);
    this.buildTemplateFromArgs = checkNotNull(buildTemplateFromArgs, "metadata for %s", target);
    this.options = checkNotNull(options, "options for %s", target);
    this.errorDecoder = checkNotNull(errorDecoder, "errorDecoder for %s", target);
    this.decoder = checkNotNull(decoder, "decoder for %s", target);
    this.decode404 = decode404;
  }

  @Override
  public Object invoke(Object[] argv) throws Throwable {
    RequestTemplate template = buildTemplateFromArgs.create(argv);
    Retryer retryer = this.retryer.clone();
    while (true) {
      try {
		//执行请求并解析
        return executeAndDecode(template);
      } catch (RetryableException e) {
        retryer.continueOrPropagate(e);
        if (logLevel != Logger.Level.NONE) {
          logger.logRetry(metadata.configKey(), logLevel);
        }
        continue;
      }
    }
  }

  Object executeAndDecode(RequestTemplate template) throws Throwable {
    Request request = targetRequest(template);

    if (logLevel != Logger.Level.NONE) {
      logger.logRequest(metadata.configKey(), logLevel, request);
    }

    Response response;
    long start = System.nanoTime();
    try {
      response = client.execute(request, options);
      // ensure the request is set. TODO: remove in Feign 10
      response.toBuilder().request(request).build();
    } catch (IOException e) {
      if (logLevel != Logger.Level.NONE) {
        logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime(start));
      }
      throw errorExecuting(request, e);
    }
    long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);

    boolean shouldClose = true;
    try {
      if (logLevel != Logger.Level.NONE) {
        response =
            logger.logAndRebufferResponse(metadata.configKey(), logLevel, response, elapsedTime);
        // ensure the request is set. TODO: remove in Feign 10
        response.toBuilder().request(request).build();
      }
      if (Response.class == metadata.returnType()) {
        if (response.body() == null) {
          return response;
        }
        if (response.body().length() == null ||
                response.body().length() > MAX_RESPONSE_BUFFER_SIZE) {
          shouldClose = false;
          return response;
        }
        // Ensure the response body is disconnected
        byte[] bodyData = Util.toByteArray(response.body().asInputStream());
        return response.toBuilder().body(bodyData).build();
      }
      if (response.status() >= 200 && response.status() < 300) {
        if (void.class == metadata.returnType()) {
          return null;
        } else {
          return decode(response);
        }
      } else if (decode404 && response.status() == 404) {
        return decode(response);
      } else {
        throw errorDecoder.decode(metadata.configKey(), response);
      }
    } catch (IOException e) {
      if (logLevel != Logger.Level.NONE) {
        logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime);
      }
      throw errorReading(request, response, e);
    } finally {
      if (shouldClose) {
        ensureClosed(response.body());
      }
    }
  }

  long elapsedTime(long start) {
    return TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);
  }

  Request targetRequest(RequestTemplate template) {
    for (RequestInterceptor interceptor : requestInterceptors) {
      interceptor.apply(template);
    }
    return target.apply(new RequestTemplate(template));
  }

  Object decode(Response response) throws Throwable {
    try {
      return decoder.decode(response, metadata.returnType());
    } catch (FeignException e) {
      throw e;
    } catch (RuntimeException e) {
      throw new DecodeException(e.getMessage(), e);
    }
  }
```

再看一下带熔断功能的HystrixTargeter：

```java
class HystrixTargeter implements Targeter {

	@Override
	public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign, FeignContext context,
						Target.HardCodedTarget<T> target) {
		//Feign实例不是HystrixFeign.Builder的则忽略，调用原生的构建方法
		if (!(feign instanceof feign.hystrix.HystrixFeign.Builder)) {
			return feign.target(target);
		}
		//以下方法则使用HystrixFeign Builder （自定义了构建实际代理对象HystrixInvocationHandler）构建一个执行处理器
		feign.hystrix.HystrixFeign.Builder builder = (feign.hystrix.HystrixFeign.Builder) feign;
		SetterFactory setterFactory = getOptional(factory.getName(), context,
			SetterFactory.class);
		if (setterFactory != null) {
			builder.setterFactory(setterFactory);
		}
		Class<?> fallback = factory.getFallback();
		if (fallback != void.class) {
			return targetWithFallback(factory.getName(), context, target, builder, fallback);
		}
		Class<?> fallbackFactory = factory.getFallbackFactory();
		if (fallbackFactory != void.class) {
			return targetWithFallbackFactory(factory.getName(), context, target, builder, fallbackFactory);
		}

		return feign.target(target);
	}

	private <T> T targetWithFallbackFactory(String feignClientName, FeignContext context,
											Target.HardCodedTarget<T> target,
											HystrixFeign.Builder builder,
											Class<?> fallbackFactoryClass) {
		FallbackFactory<? extends T> fallbackFactory = (FallbackFactory<? extends T>)
			getFromContext("fallbackFactory", feignClientName, context, fallbackFactoryClass, FallbackFactory.class);
		/* We take a sample fallback from the fallback factory to check if it returns a fallback
		that is compatible with the annotated feign interface. */
		Object exampleFallback = fallbackFactory.create(new RuntimeException());
		Assert.notNull(exampleFallback,
			String.format(
			"Incompatible fallbackFactory instance for feign client %s. Factory may not produce null!",
				feignClientName));
		if (!target.type().isAssignableFrom(exampleFallback.getClass())) {
			throw new IllegalStateException(
				String.format(
					"Incompatible fallbackFactory instance for feign client %s. Factory produces instances of '%s', but should produce instances of '%s'",
					feignClientName, exampleFallback.getClass(), target.type()));
		}
		return builder.target(target, fallbackFactory);
	}


	private <T> T targetWithFallback(String feignClientName, FeignContext context,
									 Target.HardCodedTarget<T> target,
									 HystrixFeign.Builder builder, Class<?> fallback) {
		T fallbackInstance = getFromContext("fallback", feignClientName, context, fallback, target.type());
		return builder.target(target, fallbackInstance);
	}

	private <T> T getFromContext(String fallbackMechanism, String feignClientName, FeignContext context,
								 Class<?> beanType, Class<T> targetType) {
		Object fallbackInstance = context.getInstance(feignClientName, beanType);
		if (fallbackInstance == null) {
			throw new IllegalStateException(String.format(
				"No " + fallbackMechanism + " instance of type %s found for feign client %s",
				beanType, feignClientName));
		}

		if (!targetType.isAssignableFrom(beanType)) {
			throw new IllegalStateException(
					String.format(
						"Incompatible " + fallbackMechanism + " instance. Fallback/fallbackFactory of type %s is not assignable to %s for feign client %s",
						beanType, targetType, feignClientName));
		}
		return (T) fallbackInstance;
	}

	private <T> T getOptional(String feignClientName, FeignContext context,
		Class<T> beanType) {
		return context.getInstance(feignClientName, beanType);
	}
```

最终HystrixTargeter创建出HystrixInvocationHandler，其源码为：

```java
final class HystrixInvocationHandler implements InvocationHandler {

  private final Target<?> target;
  private final Map<Method, MethodHandler> dispatch;
  private final FallbackFactory<?> fallbackFactory; // Nullable
  private final Map<Method, Method> fallbackMethodMap;
  private final Map<Method, Setter> setterMethodMap;

  HystrixInvocationHandler(Target<?> target, Map<Method, MethodHandler> dispatch,
                           SetterFactory setterFactory, FallbackFactory<?> fallbackFactory) {
    this.target = checkNotNull(target, "target");
    this.dispatch = checkNotNull(dispatch, "dispatch");
    this.fallbackFactory = fallbackFactory;
    this.fallbackMethodMap = toFallbackMethod(dispatch);
    this.setterMethodMap = toSetters(setterFactory, target, dispatch.keySet());
  }

  /**
   * If the method param of InvocationHandler.invoke is not accessible, i.e in a package-private
   * interface, the fallback call in hystrix command will fail cause of access restrictions. But
   * methods in dispatch are copied methods. So setting access to dispatch method doesn't take
   * effect to the method in InvocationHandler.invoke. Use map to store a copy of method to invoke
   * the fallback to bypass this and reducing the count of reflection calls.
   *
   * @return cached methods map for fallback invoking
   */
  static Map<Method, Method> toFallbackMethod(Map<Method, MethodHandler> dispatch) {
    Map<Method, Method> result = new LinkedHashMap<Method, Method>();
    for (Method method : dispatch.keySet()) {
      method.setAccessible(true);
      result.put(method, method);
    }
    return result;
  }

  /**
   * Process all methods in the target so that appropriate setters are created.
   */
  static Map<Method, Setter> toSetters(SetterFactory setterFactory, Target<?> target,
                                       Set<Method> methods) {
    Map<Method, Setter> result = new LinkedHashMap<Method, Setter>();
    for (Method method : methods) {
      method.setAccessible(true);
      result.put(method, setterFactory.create(target, method));
    }
    return result;
  }

  @Override
  public Object invoke(final Object proxy, final Method method, final Object[] args)
      throws Throwable {
    // early exit if the invoked method is from java.lang.Object
    // code is the same as ReflectiveFeign.FeignInvocationHandler
    if ("equals".equals(method.getName())) {
      try {
        Object otherHandler =
            args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
        return equals(otherHandler);
      } catch (IllegalArgumentException e) {
        return false;
      }
    } else if ("hashCode".equals(method.getName())) {
      return hashCode();
    } else if ("toString".equals(method.getName())) {
      return toString();
    }
	//核心方法位置，在此，则实现把客户端请求封装为Hystrix命令的方式进行调用，实现了调用的熔断与降级
    HystrixCommand<Object> hystrixCommand = new HystrixCommand<Object>(setterMethodMap.get(method)) {
      @Override
      protected Object run() throws Exception {
        try {
		//实际调用的请求方法已在上一个默认Handler的第二部分指出
          return HystrixInvocationHandler.this.dispatch.get(method).invoke(args);
        } catch (Exception e) {
          throw e;
        } catch (Throwable t) {
          throw (Error) t;
        }
      }

      @Override
      protected Object getFallback() {
        if (fallbackFactory == null) {
          return super.getFallback();
        }
        try {
          Object fallback = fallbackFactory.create(getExecutionException());
          Object result = fallbackMethodMap.get(method).invoke(fallback, args);
          if (isReturnsHystrixCommand(method)) {
            return ((HystrixCommand) result).execute();
          } else if (isReturnsObservable(method)) {
            // Create a cold Observable
            return ((Observable) result).toBlocking().first();
          } else if (isReturnsSingle(method)) {
            // Create a cold Observable as a Single
            return ((Single) result).toObservable().toBlocking().first();
          } else if (isReturnsCompletable(method)) {
            ((Completable) result).await();
            return null;
          } else {
            return result;
          }
        } catch (IllegalAccessException e) {
          // shouldn't happen as method is public due to being an interface
          throw new AssertionError(e);
        } catch (InvocationTargetException e) {
          // Exceptions on fallback are tossed by Hystrix
          throw new AssertionError(e.getCause());
        }
      }
    };

    if (isReturnsHystrixCommand(method)) {
      return hystrixCommand;
    } else if (isReturnsObservable(method)) {
      // Create a cold Observable
      return hystrixCommand.toObservable();
    } else if (isReturnsSingle(method)) {
      // Create a cold Observable as a Single
      return hystrixCommand.toObservable().toSingle();
    } else if (isReturnsCompletable(method)) {
      return hystrixCommand.toObservable().toCompletable();
    }
    return hystrixCommand.execute();
  }
```

至此，一个Feign代理Bean终于初始化完成。接下来讲述上述创建流程中使用到的一些Feign核心组件Bean和配置。

### Feign组件配置

Feign组件的配置主要有3个自动化配置类：

- FeignAutoConfiguration：配置Feign上下文（FeignContext）、配置Targeter、配置Client(仅仅组件)

  ```java
  @Configuration
  @ConditionalOnClass(Feign.class)
  public class FeignAutoConfiguration {

  	@Autowired(required = false)
  	private List<FeignClientSpecification> configurations = new ArrayList<>();

  	@Bean
  	public HasFeatures feignFeature() {
  		return HasFeatures.namedFeature("Feign", Feign.class);
  	}
  	/*FeignContext 继承自NamedContextFactory，可以用此对象根据Bean名字或者对象获取到实例*/
  	@Bean
  	public FeignContext feignContext() {
  		FeignContext context = new FeignContext();
  		context.setConfigurations(this.configurations);
  		return context;
  	}
  	/*默认引入的包依赖已带此类，所以默认使用的Targeter是这个带熔断的HystrixTargeter*/
  	@Configuration
  	@ConditionalOnClass(name = "feign.hystrix.HystrixFeign")
  	protected static class HystrixFeignTargeterConfiguration {
  		@Bean
  		@ConditionalOnMissingBean
  		public Targeter feignTargeter() {
  			return new HystrixTargeter();
  		}
  	}

  	@Configuration
  	@ConditionalOnMissingClass("feign.hystrix.HystrixFeign")
  	protected static class DefaultFeignTargeterConfiguration {
  		@Bean
  		@ConditionalOnMissingBean
  		public Targeter feignTargeter() {
  			return new DefaultTargeter();
  		}
  	}

  	// the following configuration is for alternate feign clients if
  	// ribbon is not on the class path.
  	// see corresponding configurations in FeignRibbonClientAutoConfiguration
  	// for load balanced ribbon clients.
  	@Configuration
  	@ConditionalOnClass(ApacheHttpClient.class)
  	@ConditionalOnMissingClass("com.netflix.loadbalancer.ILoadBalancer")
  	@ConditionalOnProperty(value = "feign.httpclient.enabled", matchIfMissing = true)
  	protected static class HttpClientFeignConfiguration {

  		@Autowired(required = false)
  		private HttpClient httpClient;

  		@Bean
  		@ConditionalOnMissingBean(Client.class)
  		public Client feignClient() {
  			if (this.httpClient != null) {
  				return new ApacheHttpClient(this.httpClient);
  			}
  			return new ApacheHttpClient();
  		}
  	}

  	@Configuration
  	@ConditionalOnClass(OkHttpClient.class)
  	@ConditionalOnMissingClass("com.netflix.loadbalancer.ILoadBalancer")
  	@ConditionalOnProperty(value = "feign.okhttp.enabled", matchIfMissing = true)
  	protected static class OkHttpFeignConfiguration {

  		@Autowired(required = false)
  		private okhttp3.OkHttpClient okHttpClient;

  		@Bean
  		@ConditionalOnMissingBean(Client.class)
  		public Client feignClient() {
  			if (this.okHttpClient != null) {
  				return new OkHttpClient(this.okHttpClient);
  			}
  			return new OkHttpClient();
  		}
  	}
  ```

- FeignClientsConfiguration：Decoder、Encoder、Retryer、Contract（SpringMvcContract）、FeignBuilder

  ```java
  @Configuration
  public class FeignClientsConfiguration {

  	@Autowired
  	private ObjectFactory<HttpMessageConverters> messageConverters;

  	@Autowired(required = false)
  	private List<AnnotatedParameterProcessor> parameterProcessors = new ArrayList<>();

  	@Autowired(required = false)
  	private List<FeignFormatterRegistrar> feignFormatterRegistrars = new ArrayList<>();

  	@Autowired(required = false)
  	private Logger logger;

  	@Bean
  	@ConditionalOnMissingBean
  	public Decoder feignDecoder() {
  		return new ResponseEntityDecoder(new SpringDecoder(this.messageConverters));
  	}

  	@Bean
  	@ConditionalOnMissingBean
  	public Encoder feignEncoder() {
  		return new SpringEncoder(this.messageConverters);
  	}

  	@Bean
  	@ConditionalOnMissingBean
  	public Contract feignContract(ConversionService feignConversionService) {
  		return new SpringMvcContract(this.parameterProcessors, feignConversionService);
  	}

  	@Bean
  	public FormattingConversionService feignConversionService() {
  		FormattingConversionService conversionService = new DefaultFormattingConversionService();
  		for (FeignFormatterRegistrar feignFormatterRegistrar : feignFormatterRegistrars) {
  			feignFormatterRegistrar.registerFormatters(conversionService);
  		}
  		return conversionService;
  	}
  	/*默认使用的Feign Builer*/
  	@Configuration
  	@ConditionalOnClass({ HystrixCommand.class, HystrixFeign.class })
  	protected static class HystrixFeignConfiguration {
  		@Bean
  		@Scope("prototype")
  		@ConditionalOnMissingBean
  		@ConditionalOnProperty(name = "feign.hystrix.enabled", matchIfMissing = false)
  		public Feign.Builder feignHystrixBuilder() {
  			return HystrixFeign.builder();
  		}
  	}

  	@Bean
  	@ConditionalOnMissingBean
  	public Retryer feignRetryer() {
  		return Retryer.NEVER_RETRY;
  	}

  	@Bean
  	@Scope("prototype")
  	@ConditionalOnMissingBean
  	public Feign.Builder feignBuilder(Retryer retryer) {
  		return Feign.builder().retryer(retryer);
  	}

  	@Bean
  	@ConditionalOnMissingBean(FeignLoggerFactory.class)
  	public FeignLoggerFactory feignLoggerFactory() {
  		return new DefaultFeignLoggerFactory(logger);
  	}

  }
  ```

- FeignRibbonClientAutoConfiguration：Request Options（超时配置）、配置Client(带负载均衡)

  ```java
  @ConditionalOnClass({ ILoadBalancer.class, Feign.class })
  @Configuration
  @AutoConfigureBefore(FeignAutoConfiguration.class)
  public class FeignRibbonClientAutoConfiguration {

  	@Bean
  	@Primary
  	@ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
  	public CachingSpringLoadBalancerFactory cachingLBClientFactory(
  			SpringClientFactory factory) {
  		return new CachingSpringLoadBalancerFactory(factory);
  	}

  	@Bean
  	@Primary
  	@ConditionalOnClass(name = "org.springframework.retry.support.RetryTemplate")
  	public CachingSpringLoadBalancerFactory retryabeCachingLBClientFactory(
  			SpringClientFactory factory, LoadBalancedRetryPolicyFactory retryPolicyFactory) {
  		return new CachingSpringLoadBalancerFactory(factory, retryPolicyFactory, true);
  	}

  	@Bean
  	@ConditionalOnMissingBean
  	public Client feignClient(CachingSpringLoadBalancerFactory cachingFactory,
  			SpringClientFactory clientFactory) {
  		return new LoadBalancerFeignClient(new Client.Default(null, null),
  				cachingFactory, clientFactory);
  	}

  	/*配置超时设置，默认连接超时10秒，读取超时60秒*/
  	@Bean
  	@ConditionalOnMissingBean
  	public Request.Options feignRequestOptions() {
  		return LoadBalancerFeignClient.DEFAULT_OPTIONS;
  	}

  	@Configuration
  	@ConditionalOnClass(ApacheHttpClient.class)
  	@ConditionalOnProperty(value = "feign.httpclient.enabled", matchIfMissing = true)
  	protected static class HttpClientFeignLoadBalancedConfiguration {

  		@Autowired(required = false)
  		private HttpClient httpClient;

  		@Bean
  		@ConditionalOnMissingBean(Client.class)
  		public Client feignClient(CachingSpringLoadBalancerFactory cachingFactory,
  				SpringClientFactory clientFactory) {
  			ApacheHttpClient delegate;
  			if (this.httpClient != null) {
  				delegate = new ApacheHttpClient(this.httpClient);
  			}
  			else {
  				delegate = new ApacheHttpClient();
  			}
  			return new LoadBalancerFeignClient(delegate, cachingFactory, clientFactory);
  		}
  	}

  	@Configuration
  	@ConditionalOnClass(OkHttpClient.class)
  	@ConditionalOnProperty(value = "feign.okhttp.enabled", matchIfMissing = true)
  	protected static class OkHttpFeignLoadBalancedConfiguration {

  		@Autowired(required = false)
  		private okhttp3.OkHttpClient okHttpClient;

  		@Bean
  		@ConditionalOnMissingBean(Client.class)
  		public Client feignClient(CachingSpringLoadBalancerFactory cachingFactory,
  				SpringClientFactory clientFactory) {
  			OkHttpClient delegate;
  			if (this.okHttpClient != null) {
  				delegate = new OkHttpClient(this.okHttpClient);
  			}
  			else {
  				delegate = new OkHttpClient();
  			}
  			return new LoadBalancerFeignClient(delegate, cachingFactory, clientFactory);
  		}
  	}
  }
  ```

## 配置与优化

### 超时时间配置

Spring Feign组件的超时配置主要存在3块：

- 一块是`FeignClientsConfiguration`的`LoadBalancerFeignClient.DEFAULT_OPTIONS`，连接超时10S，读取超时60S;
- 一块是`RibbonClientConfiguration`的`ribbonClientConfig()`,连接超时2S，读取超时5S；
- 一块是Hystrix组件的执行超时配置，`hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds`;

在Feign中上述配置优先级与顺序为：

- 当`Request.Options`等于`LoadBalancerFeignClient.DEFAULT_OPTIONS`时，以`RibbonClientConfiguration`中配置的超时时间为准；
- 当`Request.Options`不等于`LoadBalancerFeignClient.DEFAULT_OPTIONS`，以自定义的超时时间配置为准；
- Hystrix组件的执行超时配置用于控制调用请求执行超时，和具体HTTP请求无关，需要和上述2个配置配合使用，需大于上述2条超时时间配置；

> `RibbonClientConfiguration`中的`FeignOptionsClientConfig`支持动态配置，具体可以查看`DefaultClientConfigImpl`源码。

### 请求重试

Feign的重试是通过配置Retryer来实现，在`FeignClientsConfiguration`自动配置类中，配置了一个默认的`Retryer.NEVER_RETRY`，表示用不重试。不重试和重试超过限制次数都是抛出异常来停止重试。可以通过自定义Retryer来覆盖默认的配置，但需要注意Hystrix的执行超时，2个中较短的时间为请求最终执行的时间。

### 编码/消息头处理

对具体调用请求可以通过3种方式来进行编解码处理：

- 定义`RequestInterceptor`，在请求前处理编码与附加消息头;
- 自定义`Encoder`，在编码阶段处理请求头;
- 在`@FeignClient`组件接口`@RequestMapping`的`headers`属性中附带消息头信息；

### 连接池

Feign默认使用的请求客户端并没有建立连接池，可以考虑使用ApacheHttpClient与OkHttpClient进行Client的替换，并池话发起HTTP请求。

### 其他事项

- 在定义Feign组件时，`@RequestMapping`注解只加在方法上，不要放在类上，Spring MVC自带的Dispacher请求映射会扫描所有带`@RequestMapping`类，容易导致一些不必要的问题；
- Feign组件目前只支持`@RequestMapping`注解定义请求路径和配置，赞不支持Spring MVC新出的`@GetMapping`、`@PostMapping`等注解，具体扫描和处理参考`SpringMvcContract`源码。

# 666. 彩蛋

如果你对 Ribbon   感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)