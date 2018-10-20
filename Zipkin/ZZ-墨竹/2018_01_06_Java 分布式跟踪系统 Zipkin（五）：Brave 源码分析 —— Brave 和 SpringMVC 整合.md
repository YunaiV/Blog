title: Java 分布式跟踪系统 Zipkin（五）：Brave 源码分析 —— Brave 和 SpringMVC 整合
date: 2018-01-06
tag: 
categories: Zipkin
permalink: Zipkin/mozhu/brave-with-spring-mvc
author: v墨竹v
from_url: https://blog.csdn.net/apei830/article/details/78722244
wechat_url: 

-------

摘要: 原创出处 https://blog.csdn.net/apei830/article/details/78722244 「v墨竹v」欢迎转载，保留摘要，谢谢！

- [XML配置方式](http://www.iocoder.cn/Zipkin/mozhu/brave-with-spring-mvc/)
- [Annotation注解方式](http://www.iocoder.cn/Zipkin/mozhu/brave-with-spring-mvc/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

上一篇博文中，我们分析了Brave是如何在普通Web项目中使用的，这一篇博文我们继续分析Brave和SpringMVC项目的整合方法及原理。
我们分两个部分来介绍和SpringMVC的整合，及XML配置方式和Annotation注解方式

pom.xml添加相关依赖spring-web和spring-webmvc

```XML
<dependency>
  <groupId>io.zipkin.brave</groupId>
  <artifactId>brave-instrumentation-spring-web</artifactId>
  <version>${brave.version}</version>
</dependency>
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-web</artifactId>
  <version>${spring.version}</version>
</dependency>

<dependency>
  <groupId>io.zipkin.brave</groupId>
  <artifactId>brave-instrumentation-spring-webmvc</artifactId>
  <version>${brave.version}</version>
</dependency>
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-webmvc</artifactId>
  <version>${spring.version}</version>
</dependency>
```

## XML配置方式

在Servlet2.5规范中，必须配置web.xml，我们只需要配置DispatcherServlet，SpringMVC的核心控制器就可以了
相关代码在Chapter5/springmvc-servlet25中
web.xml

```XML
<web-app xmlns="http://java.sun.com/xml/ns/javaee"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
	http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
    version="2.5">

  <display-name>SpringMVC Servlet2.5 Application</display-name>

  <servlet>
    <servlet-name>spring-webmvc</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
  </servlet>

  <servlet-mapping>
    <servlet-name>spring-webmvc</servlet-name>
    <url-pattern>/</url-pattern>
  </servlet-mapping>
</web-app>
```

然后在WEB-INF下配置spring-webmvc-servlet.xml

```XML
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:context="http://www.springframework.org/schema/context"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:mvc="http://www.springframework.org/schema/mvc"
    xmlns:util="http://www.springframework.org/schema/util"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
        http://www.springframework.org/schema/mvc
        http://www.springframework.org/schema/mvc/spring-mvc-3.2.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context-3.2.xsd
        http://www.springframework.org/schema/util
        http://www.springframework.org/schema/util/spring-util-3.2.xsd">

  <context:property-placeholder/>

  <bean id="sender" class="zipkin2.reporter.okhttp3.OkHttpSender" factory-method="create">
    <constructor-arg type="String" value="http://localhost:9411/api/v2/spans"/>
  </bean>

  <bean id="tracing" class="brave.spring.beans.TracingFactoryBean">
    <property name="localServiceName" value="${zipkin.service:springmvc-servlet25-example}"/>
    <property name="spanReporter">
      <bean class="brave.spring.beans.AsyncReporterFactoryBean">
        <property name="encoder" value="JSON_V2"/>
        <property name="sender" ref="sender"/>
        <!-- wait up to half a second for any in-flight spans on close -->
        <property name="closeTimeout" value="500"/>
      </bean>
    </property>
    <property name="propagationFactory">
      <bean id="propagationFactory" class="brave.propagation.ExtraFieldPropagation" factory-method="newFactory">
        <constructor-arg index="0">
          <util:constant static-field="brave.propagation.B3Propagation.FACTORY"/>
        </constructor-arg>
        <constructor-arg index="1">
          <list>
            <value>user-name</value>
          </list>
        </constructor-arg>
      </bean>
    </property>
    <property name="currentTraceContext">
      <bean class="brave.context.log4j2.ThreadContextCurrentTraceContext" factory-method="create"/>
    </property>
  </bean>

  <bean id="httpTracing" class="brave.spring.beans.HttpTracingFactoryBean">
    <property name="tracing" ref="tracing"/>
  </bean>

  <bean id="restTemplate" class="org.springframework.web.client.RestTemplate">
    <property name="interceptors">
      <list>
        <bean class="brave.spring.web.TracingClientHttpRequestInterceptor" factory-method="create">
          <constructor-arg type="brave.http.HttpTracing" ref="httpTracing"/>
        </bean>
      </list>
    </property>
  </bean>

  <mvc:interceptors>
    <bean class="brave.spring.webmvc.TracingHandlerInterceptor" factory-method="create">
      <constructor-arg type="brave.http.HttpTracing" ref="httpTracing"/>
    </bean>
  </mvc:interceptors>

  <!-- Loads the controller -->
  <context:component-scan base-package="org.mozhu.zipkin.springmvc"/>
  <mvc:annotation-driven/>
</beans>
```

使用brave.spring.beans.TracingFactoryBean创建tracing
使用brave.spring.beans.HttpTracingFactoryBean创建httpTracing
配置springmvc的拦截器brave.spring.webmvc.TracingHandlerInterceptor
并配置org.springframework.web.client.RestTemplate作为客户端发送http请求

再来看看两个Controller：Frontend和Backend，和前面FrontendServlet，BackendServlet功能一样

### Frontend

```java
@RestController
public class Frontend {
    private final static Logger LOGGER = LoggerFactory.getLogger(Frontend.class);
    @Autowired
    RestTemplate restTemplate;

    @RequestMapping("/")
    public String callBackend() {
        LOGGER.info("frontend receive request");
        return restTemplate.getForObject("http://localhost:9000/api", String.class);
    }
}
```

Frontend中使用Spring提供的restTemplate向Backend发送请求

### Backend

```java
@RestController
public class Backend {

  private final static Logger LOGGER = LoggerFactory.getLogger(Backend.class);

  @RequestMapping("/api")
  public String printDate(@RequestHeader(name = "user-name", required = false) String username) {
    LOGGER.info("backend receive request");
    if (username != null) {
      return new Date().toString() + " " + username;
    }
    return new Date().toString();
  }
}
```

Backend中收到来自Frontend的请求，并给出响应，打出当前的时间戳，如果headers中存在user-name，也会添加到响应字符串尾部

跟前面博文一样，启动Zipkin，然后分别运行

```shell
mvn jetty:run -Pbackend
```

```shell
mvn jetty:run -Pfrontend
```

浏览器访问 <http://localhost:8081/> 会显示当前时间
在Zipkin的Web界面中，也能查询到这次跟踪信息

现在来分析下两个Spring相关的类
brave.spring.webmvc.TracingHandlerInterceptor - 服务端请求的拦截器，在这个类里会处理服务端的trace信息
brave.spring.web.TracingClientHttpRequestInterceptor - 客户端请求的拦截器，在这个类里会处理客户端的trace信息

TracingHandlerInterceptor

```java
public final class TracingHandlerInterceptor extends HandlerInterceptorAdapter {

  public static AsyncHandlerInterceptor create(Tracing tracing) {
    return new TracingHandlerInterceptor(HttpTracing.create(tracing));
  }

  public static AsyncHandlerInterceptor create(HttpTracing httpTracing) {
    return new TracingHandlerInterceptor(httpTracing);
  }

  final Tracer tracer;
  final HttpServerHandler<HttpServletRequest, HttpServletResponse> handler;
  final TraceContext.Extractor<HttpServletRequest> extractor;

  @Autowired TracingHandlerInterceptor(HttpTracing httpTracing) { // internal
    tracer = httpTracing.tracing().tracer();
    handler = HttpServerHandler.create(httpTracing, new HttpServletAdapter());
    extractor = httpTracing.tracing().propagation().extractor(HttpServletRequest::getHeader);
  }

  @Override
  public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object o) {
    if (request.getAttribute(SpanInScope.class.getName()) != null) {
      return true; // already handled (possibly due to async request)
    }

    Span span = handler.handleReceive(extractor, request);
    request.setAttribute(SpanInScope.class.getName(), tracer.withSpanInScope(span));
    return true;
  }

  @Override
  public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
      Object o, Exception ex) {
    Span span = tracer.currentSpan();
    if (span == null) return;
    ((SpanInScope) request.getAttribute(SpanInScope.class.getName())).close();
    handler.handleSend(response, ex, span);
  }
}
```

TracingHandlerInterceptor继承了HandlerInterceptorAdapter，覆盖了其中preHandle和afterCompletion方法，分别在请求执行前，和请求完成后执行。
这里没办法向前面几篇博文的一样，使用try-with-resources来自动关闭SpanInScope，所以只能在preHandle中将SpanInScope放在request的attribute中，然后在afterCompletion中将其取出来手动close，其他代码逻辑和前面TracingFilter里一样

TracingClientHttpRequestInterceptor

```java
public final class TracingClientHttpRequestInterceptor implements ClientHttpRequestInterceptor {
  static final Propagation.Setter<HttpHeaders, String> SETTER = HttpHeaders::set;

  public static ClientHttpRequestInterceptor create(Tracing tracing) {
    return create(HttpTracing.create(tracing));
  }

  public static ClientHttpRequestInterceptor create(HttpTracing httpTracing) {
    return new TracingClientHttpRequestInterceptor(httpTracing);
  }

  final Tracer tracer;
  final HttpClientHandler<HttpRequest, ClientHttpResponse> handler;
  final TraceContext.Injector<HttpHeaders> injector;

  @Autowired TracingClientHttpRequestInterceptor(HttpTracing httpTracing) {
    tracer = httpTracing.tracing().tracer();
    handler = HttpClientHandler.create(httpTracing, new HttpAdapter());
    injector = httpTracing.tracing().propagation().injector(SETTER);
  }

  @Override public ClientHttpResponse intercept(HttpRequest request, byte[] body,
      ClientHttpRequestExecution execution) throws IOException {
    Span span = handler.handleSend(injector, request.getHeaders(), request);
    ClientHttpResponse response = null;
    Throwable error = null;
    try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
      return response = execution.execute(request, body);
    } catch (IOException | RuntimeException | Error e) {
      error = e;
      throw e;
    } finally {
      handler.handleReceive(response, error, span);
    }
  }

  static final class HttpAdapter
      extends brave.http.HttpClientAdapter<HttpRequest, ClientHttpResponse> {

    @Override public String method(HttpRequest request) {
      return request.getMethod().name();
    }

    @Override public String url(HttpRequest request) {
      return request.getURI().toString();
    }

    @Override public String requestHeader(HttpRequest request, String name) {
      Object result = request.getHeaders().getFirst(name);
      return result != null ? result.toString() : null;
    }

    @Override public Integer statusCode(ClientHttpResponse response) {
      try {
        return response.getRawStatusCode();
      } catch (IOException e) {
        return null;
      }
    }
  }
}
```

TracingClientHttpRequestInterceptor里的逻辑和前面博文分析的brave.okhttp3.TracingInterceptor类似，此处不再展开分析

下面再来介绍用Annotation注解方式来配置SpringMVC和Brave

## Annotation注解方式

相关代码在Chapter5/springmvc-servlet3中
在Servlet3以后，web.xml不是必须的了，org.mozhu.zipkin.springmvc.Initializer是我们整个应用的启动器

```java
public class Initializer extends AbstractAnnotationConfigDispatcherServletInitializer {

  @Override protected String[] getServletMappings() {
    return new String[] {"/"};
  }

  @Override protected Class<?>[] getRootConfigClasses() {
    return null;
  }

  @Override protected Class<?>[] getServletConfigClasses() {
    return new Class[] {TracingConfiguration.class};
  }
}
```

org.mozhu.zipkin.springmvc.Initializer，继承自AbstractDispatcherServletInitializer，实现了WebApplicationInitializer
WebApplicationInitializer

```java
public interface WebApplicationInitializer {

	void onStartup(ServletContext servletContext) throws ServletException;

}
```

关于Servlet3的容器是如何启动的，我们再来看一个类SpringServletContainerInitializer，该类实现了javax.servlet.ServletContainerInitializer接口，并且该类上有一个javax.servlet.annotation.HandlesTypes注解
Servlet3规范规定实现Servlet3的容器，必须加载classpath里所有实现了ServletContainerInitializer接口的类，并调用其onStartup方法，传入的第一个参数是类上HandlesTypes中所指定的类，这里是WebApplicationInitializer的集合
在SpringServletContainerInitializer的onStartup方法中，会将传入的WebApplicationInitializer类，全部实例化，并且排序，然后依次调用它们的initializer.onStartup(servletContext)。

```java
@HandlesTypes(WebApplicationInitializer.class)
public class SpringServletContainerInitializer implements ServletContainerInitializer {

	@Override
	public void onStartup(Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
			throws ServletException {

		List<WebApplicationInitializer> initializers = new LinkedList<WebApplicationInitializer>();

		if (webAppInitializerClasses != null) {
			for (Class<?> waiClass : webAppInitializerClasses) {
				// Be defensive: Some servlet containers provide us with invalid classes,
				// no matter what @HandlesTypes says...
				if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers()) &&
						WebApplicationInitializer.class.isAssignableFrom(waiClass)) {
					try {
						initializers.add((WebApplicationInitializer) waiClass.newInstance());
					}
					catch (Throwable ex) {
						throw new ServletException("Failed to instantiate WebApplicationInitializer class", ex);
					}
				}
			}
		}

		if (initializers.isEmpty()) {
			servletContext.log("No Spring WebApplicationInitializer types detected on classpath");
			return;
		}

		servletContext.log(initializers.size() + " Spring WebApplicationInitializers detected on classpath");
		AnnotationAwareOrderComparator.sort(initializers);
		for (WebApplicationInitializer initializer : initializers) {
			initializer.onStartup(servletContext);
		}
	}

}
```

另外Servlet3在ServletContext中提供了addServlet方法，允许以编码方式向容器中添加Servlet

```java
public ServletRegistration.Dynamic addServlet(String servletName, Servlet servlet);
```

而在AbstractDispatcherServletInitializer中registerDispatcherServlet方法会将SpringMVC的核心控制器DispatcherServlet添加到Web容器中。

```java
protected void registerDispatcherServlet(ServletContext servletContext) {
	String servletName = getServletName();
	Assert.hasLength(servletName, "getServletName() must not return empty or null");

	WebApplicationContext servletAppContext = createServletApplicationContext();
	Assert.notNull(servletAppContext,
			"createServletApplicationContext() did not return an application " +
			"context for servlet [" + servletName + "]");

	FrameworkServlet dispatcherServlet = createDispatcherServlet(servletAppContext);
	dispatcherServlet.setContextInitializers(getServletApplicationContextInitializers());

	ServletRegistration.Dynamic registration = servletContext.addServlet(servletName, dispatcherServlet);
	Assert.notNull(registration,
			"Failed to register servlet with name '" + servletName + "'." +
			"Check if there is another servlet registered under the same name.");

	registration.setLoadOnStartup(1);
	registration.addMapping(getServletMappings());
	registration.setAsyncSupported(isAsyncSupported());

	Filter[] filters = getServletFilters();
	if (!ObjectUtils.isEmpty(filters)) {
		for (Filter filter : filters) {
			registerServletFilter(servletContext, filter);
		}
	}

	customizeRegistration(registration);
}
protected FrameworkServlet createDispatcherServlet(WebApplicationContext servletAppContext) {
	return new DispatcherServlet(servletAppContext);
}
```

以前用xml配置bean的方式，全改为在TracingConfiguration类里用@Bean注解来配置，并且使用@ComponentScan注解指定controller的package，让Spring容器可以扫描到这些Controller

```java
@Configuration
@EnableWebMvc
@ComponentScan(basePackages = "org.mozhu.zipkin.springmvc")
@Import({TracingClientHttpRequestInterceptor.class, TracingHandlerInterceptor.class})
public class TracingConfiguration extends WebMvcConfigurerAdapter {

  @Bean Sender sender() {
    return OkHttpSender.create("http://127.0.0.1:9411/api/v2/spans");
  }

  @Bean AsyncReporter<Span> spanReporter() {
    return AsyncReporter.create(sender());
  }

  @Bean Tracing tracing(@Value("${zipkin.service:springmvc-servlet3-example}") String serviceName) {
    return Tracing.newBuilder()
        .localServiceName(serviceName)
        .propagationFactory(ExtraFieldPropagation.newFactory(B3Propagation.FACTORY, "user-name"))
        .currentTraceContext(ThreadContextCurrentTraceContext.create()) // puts trace IDs into logs
        .spanReporter(spanReporter()).build();
  }

  @Bean HttpTracing httpTracing(Tracing tracing) {
    return HttpTracing.create(tracing);
  }

  @Autowired
  private TracingHandlerInterceptor serverInterceptor;

  @Autowired
  private TracingClientHttpRequestInterceptor clientInterceptor;

  @Bean RestTemplate restTemplate() {
    RestTemplate restTemplate = new RestTemplate();
    List<ClientHttpRequestInterceptor> interceptors =
      new ArrayList<>(restTemplate.getInterceptors());
    interceptors.add(clientInterceptor);
    restTemplate.setInterceptors(interceptors);
    return restTemplate;
  }

  @Override
  public void addInterceptors(InterceptorRegistry registry) {
    registry.addInterceptor(serverInterceptor);
  }
}
```

然后在getServletConfigClasses方法中指定TracingConfiguration，让Spring容器可以加载所有的配置

```java
@Override protected Class<?>[] getServletConfigClasses() {
  return new Class[] {TracingConfiguration.class};
}
```

Annotation和XML配置的方式相比，简化了不少，而其中使用的Tracing相关的类都一样，这里就不用再分析了

# 666. 彩蛋

如果你对 Zipkin 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)