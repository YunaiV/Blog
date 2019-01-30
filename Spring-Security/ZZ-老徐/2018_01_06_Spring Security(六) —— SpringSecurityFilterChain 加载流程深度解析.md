title: 【老徐】Spring Security(五) —— SpringSecurityFilterChain 加载流程深度解析
date: 2018-01-06
tag: 
categories: Spring Security
permalink: Spring-Security/laoxu/SpringSecurityFilterChain-2
author: 老徐
from_url: https://www.cnkirito.moe/spring-security-7/
wechat_url:  https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247483980&idx=1&sn=cb40ba4fea5cf100a98896d9a0404a43&chksm=fa497dfdcd3ef4ebdd162db2f674d882fd87d2648c775272a8af238c0500289d439d858804e5#rd

-------

摘要: 原创出处 https://www.cnkirito.moe/spring-security-7/ 「老徐」欢迎转载，保留摘要，谢谢！

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

SpringSecurityFilterChain 作为 SpringSecurity 的核心过滤器链在整个认证授权过程中起着举足轻重的地位，每个请求到来，都会经过该过滤器链，前文[《Spring Security(四)–核心过滤器源码分析》](https://www.cnkirito.moe/spring-security-4/) 中我们分析了 SpringSecurityFilterChain 的构成，但还有很多疑问可能没有解开：

1. 这个 SpringSecurityFilterChain 是怎么注册到 web 环境中的？
2. 有读者发出这样的疑问：”SpringSecurityFilterChain 的实现类到底是什么，我知道它是一个 Filter，但是在很多配置类中看到了 BeanName=SpringSecurityFilterChain 相关的类，比如 DelegatingFilterProxy，FilterChainProxy，SecurityFilterChain，他们的的名称实在太相似了，到底哪个才是真正的实现，SpringSecurity 又为什么要这么设计？“
3. 我们貌似一直在配置 WebSecurity ，但没有对 SpringSecurityFilterChain 进行什么配置，WebSecurity 相关配置是怎么和 SpringSecurityFilterChain 结合在一起的？

以上是个人 YY 的一些 SpringSecurityFilterChain 相关的问题，因为我当初研究了一段时间 SpringSecurity 源码，依旧没有理清这么多错综复杂的类。那么本文就主要围绕 SpringSecurityFilterChain 展开我们的探索。

###6.1 SpringSecurityFilterChain是怎么注册的？

这个问题并不容易解释，因为 SpringSecurity 仅仅在 web 环境下（SpringSecurity 还支持非 web 环境）就有非常多的支持形式：

**Java 配置方式**

1. 作为独立的 SpringSecurity 依赖提供给朴素的 java web 项目使用，并且项目不使用 Spring！没错，仅仅使用 servlet，jsp 的情况下也是可以集成 SpringSecurity 的。
2. 提供给包含 SpringMVC 项目使用。
3. 提供给具备 Servlet3.0+ 的 web 项目使用。
4. SpringBoot 内嵌容器环境下使用 SpringSecurity，并且包含了一定程度的自动配置。

**XML 配置方式**

1. 使用 XML 中的命名空间配置 SpringSecurity。

注意，以上条件可能存在交集，比如我的项目是一个使用 servlet3.0 的 web 项目同时使用了 SpringMVC；也有可能使用了 SpringBoot 同时配合 SpringMVC；还有可能使用了 SpringBoot，却打成了 war 包，部署在外置的支持 Servlet3.0+ 规范的应用容器中…各种组合方式会导致配置 SpringSecurityFilterChain 的注册方式产生差异，所以，这个问题说复杂还真有点，需要根据你的环境来分析。我主要分析几种较为常见的注册方式。

SpringSecurityFilterChain 抽象概念里最重要的三个类：DelegatingFilterProxy，FilterChainProxy 和 SecurityFilterChain，对这三个类的源码分析和设计将会贯彻本文。不同环境下 DelegatingFilterProxy 的注册方式区别较大，但 FilterChainProxy 和 SecurityFilterChain 的差异不大，所以重点就是分析 DelegatingFilterProxy 的注册方式。它们三者的分析会放到下一节中。

####6.1.1 servlet3.0+环境下SpringSecurity的java config方式

这是一个比较常见的场景，你可能还没有使用 SpringBoot 内嵌的容器，将项目打成 war 包部署在外置的应用容器中，比如最常见的 tomcat，一般很少 web 项目低于 servlet3.0 版本的，并且该场景摒弃了 XML 配置。

```Java
import org.springframework.security.web.context.*;

public class SecurityWebApplicationInitializer
	extends AbstractSecurityWebApplicationInitializer {

}
```

主要自定义一个 SecurityWebApplicationInitializer 并且让其继承自 AbstractSecurityWebApplicationInitializer 即可。如此简单的一个继承背后又经历了 Spring 怎样的封装呢？自然要去 AbstractSecurityWebApplicationInitializer 中去一探究竟。经过删减后的源码如下

```Java
public abstract class AbstractSecurityWebApplicationInitializer
      implements WebApplicationInitializer {//<1>

   public static final String DEFAULT_FILTER_NAME = "springSecurityFilterChain";

   // <1> 父类WebApplicationInitializer的加载入口
   public final void onStartup(ServletContext servletContext) throws ServletException {
      beforeSpringSecurityFilterChain(servletContext);
      if (this.configurationClasses != null) {
         AnnotationConfigWebApplicationContext rootAppContext = new AnnotationConfigWebApplicationContext();
         rootAppContext.register(this.configurationClasses);
         servletContext.addListener(new ContextLoaderListener(rootAppContext));
      }
      if (enableHttpSessionEventPublisher()) {
         servletContext.addListener(
               "org.springframework.security.web.session.HttpSessionEventPublisher");
      }
      servletContext.setSessionTrackingModes(getSessionTrackingModes());
      insertSpringSecurityFilterChain(servletContext);//<2>
      afterSpringSecurityFilterChain(servletContext);
   }

    // <2> 在这儿初始化了关键的DelegatingFilterProxy
    private void insertSpringSecurityFilterChain(ServletContext servletContext) {
		String filterName = DEFAULT_FILTER_NAME;
        // <2> 该方法中最关键的一个步骤，DelegatingFilterProxy在此被创建
		DelegatingFilterProxy springSecurityFilterChain = new DelegatingFilterProxy(
				filterName);
		String contextAttribute = getWebApplicationContextAttribute();
		if (contextAttribute != null) {
			springSecurityFilterChain.setContextAttribute(contextAttribute);
		}
		registerFilter(servletContext, true, filterName, springSecurityFilterChain);
	}

    // <3> 使用servlet3.0的新特性，动态注册springSecurityFilterChain(实际上注册的是springSecurityFilterChain代理类)
    private final void registerFilter(ServletContext servletContext,
			boolean insertBeforeOtherFilters, String filterName, Filter filter) {
		Dynamic registration = servletContext.addFilter(filterName, filter);
		registration.setAsyncSupported(isAsyncSecuritySupported());
		EnumSet<DispatcherType> dispatcherTypes = getSecurityDispatcherTypes();
		registration.addMappingForUrlPatterns(dispatcherTypes, !insertBeforeOtherFilters,
				"/*");
	}

}
```

<1><3> 放在一起讲，因为他们都和 servlet3.0 新特性以及 spring 对 servlet3.0 的支持相关，这也是为什么在场景描述中我特地强调了需要 servlet3.0 环境。如果你对 servlet3.0 的新特性不了解，这儿准备了一篇详细的介绍为你阐述[《Spring揭秘–寻找遗失的web.xml》](https://www.cnkirito.moe/servlet-explore/) 。得益于 Spring 的封装，在 servlet3.0 环境下，web 容器启动时会自行去寻找类路径下所有实现了 WebApplicationInitializer 接口的 Initializer 实例，并调用他们的 onStartup 方法。所以，我们只需要继承 AbstractSecurityWebApplicationInitializer ，便可以自动触发 web 容器的加载，进而配置和 SpringSecurityFilterChain 第一个密切相关的类，第<2>步中的 DelegatingFilterProxy。

<2> DelegatingFilterProxy 在此被实例化出来。在第<3>步中，它作为一个 Filter 正式注册到了 web 容器中。

#### 6.1.2 XML 配置

这个真的是简单易懂，因为它是被指名道姓配置成一个 Filter 的。

`web.xml`

```XML
<filter>
	<filter-name>springSecurityFilterChain</filter-name>
	<filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
</filter>

<filter-mapping>
	<filter-name>springSecurityFilterChain</filter-name>
	<url-pattern>/*</url-pattern>
</filter-mapping>
```

`web.xml` 的存在注定了其无所谓当前环境是不是 servlet3.0+，虽然我个人不太喜欢 xml 的配置方式，但不得不说，这样真的很简单粗暴。

#### 6.1.3 SpringBoot 内嵌应用容器并且使用自动配置

[《Spring揭秘–寻找遗失的web.xml》](https://www.cnkirito.moe/servlet-explore/)中我曾经得出一个结论，内嵌容器是完全不会使用 SPI 机制加载 servlet3.0 新特性的那些 Initializer 的，springboot 又推崇 java configuration，所以上述两种方案完全被抛弃了。那么 SpringBoot 如何注册 DelegatingFilterProxy 呢？

```Java
@Configuration
@ConditionalOnWebApplication
@EnableConfigurationProperties
@ConditionalOnClass({ AbstractSecurityWebApplicationInitializer.class,
      SessionCreationPolicy.class })
@AutoConfigureAfter(SecurityAutoConfiguration.class)
public class SecurityFilterAutoConfiguration {

   private static final String DEFAULT_FILTER_NAME = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME;//springSecurityFilterChain

    // <1>
   @Bean
   @ConditionalOnBean(name = DEFAULT_FILTER_NAME)
   public DelegatingFilterProxyRegistrationBean securityFilterChainRegistration(
         SecurityProperties securityProperties) {
      DelegatingFilterProxyRegistrationBean registration = new DelegatingFilterProxyRegistrationBean(
            DEFAULT_FILTER_NAME);
      registration.setOrder(securityProperties.getFilterOrder());
      registration.setDispatcherTypes(getDispatcherTypes(securityProperties));
      return registration;
   }

   @Bean
   @ConditionalOnMissingBean
   public SecurityProperties securityProperties() {
      return new SecurityProperties();
   }
}
```

<1> DelegatingFilterProxyRegistrationBean 的分析在之前那篇文章中也有详细的介绍，其作用便是在 SpringBoot 环境下通过 TomcatStarter 等内嵌容器启动类来注册一个 DelegatingFilterProxy。这下，和前面两种配置方式都对应上了。

###SpringSecurityFilterChain三个核心类的源码分析

理解 SpringSecurityFilterChain 的工作流程必须搞懂三个类：`org.springframework.web.filter.DelegatingFilterProxy`，`org.springframework.security.web.FilterChainProxy` ， `org.springframework.security.web.SecurityFilterChain`

#### DelegatingFilterProxy

上面一节主要就是介绍 DelegatingFilterProxy 在不同环境下的注册方式，可以很明显的发现，DelegatingFilterProxy 是 SpringSecurity 的“门面”，注意它的包结构：org.springframework.web.filter，它本身是 Spring Web 包中的类，并不是 SpringSecurity 中的类。因为 Spring 考虑到了多种使用场景，自然希望将侵入性降到最低，所以使用了这个委托代理类来代理真正的 SpringSecurityFilterChain。DelegatingFilterProxy 实现了 javax.servlet.Filter 接口，使得它可以作为一个 java web 的标准过滤器，其职责也很简单，只负责调用真正的 SpringSecurityFilterChain。

删减掉非重要代码后的 DelegatingFilterProxy：

```Java
public class DelegatingFilterProxy extends GenericFilterBean {

   private WebApplicationContext webApplicationContext;
   // springSecurityFilterChain
   private String targetBeanName;
   // <1> 关键点
   private volatile Filter delegate;
   private final Object delegateMonitor = new Object();

   public DelegatingFilterProxy(String targetBeanName, WebApplicationContext wac) {
      Assert.hasText(targetBeanName, "Target Filter bean name must not be null or empty");
      this.setTargetBeanName(targetBeanName);
      this.webApplicationContext = wac;
      if (wac != null) {
         this.setEnvironment(wac.getEnvironment());
      }
   }

   @Override
   protected void initFilterBean() throws ServletException {
      synchronized (this.delegateMonitor) {
         if (this.delegate == null) {
            if (this.targetBeanName == null) {
               this.targetBeanName = getFilterName();
            }
            // Fetch Spring root application context and initialize the delegate early,
            // if possible. If the root application context will be started after this
            // filter proxy, we'll have to resort to lazy initialization.
            WebApplicationContext wac = findWebApplicationContext();
            if (wac != null) {
               this.delegate = initDelegate(wac);
            }
         }
      }
   }

   @Override
   public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain)
         throws ServletException, IOException {

      // 过滤器代理支持懒加载
      Filter delegateToUse = this.delegate;
      if (delegateToUse == null) {
         synchronized (this.delegateMonitor) {
            delegateToUse = this.delegate;
            if (delegateToUse == null) {
               WebApplicationContext wac = findWebApplicationContext();
               delegateToUse = initDelegate(wac);
            }
            this.delegate = delegateToUse;
         }
      }

      // 让代理过滤器执行实际的过滤行为
      invokeDelegate(delegateToUse, request, response, filterChain);
   }

   // 初始化过滤器代理
   // <2>
   protected Filter initDelegate(WebApplicationContext wac) throws ServletException {
      Filter delegate = wac.getBean(getTargetBeanName(), Filter.class);
      if (isTargetFilterLifecycle()) {
         delegate.init(getFilterConfig());
      }
      return delegate;
   }


   // 调用代理过滤器
   protected void invokeDelegate(
         Filter delegate, ServletRequest request, ServletResponse response, FilterChain filterChain)
         throws ServletException, IOException {
      delegate.doFilter(request, response, filterChain);
   }

}
```

<1> 可以发现整个 DelegatingFilterProxy 的逻辑就是为了调用 `private volatile Filter delegate;`那么问题来了，这个 delegate 的真正实现是什么呢？

<2> 可以看到，DelegatingFilterProxy 尝试去容器中获取名为 targetBeanName 的类，而 targetBeanName 的默认值便是 Filter 的名称，也就是 springSecurityFilterChain！也就是说，DelegatingFilterProxy 只是名称和 targetBeanName 叫 springSecurityFilterChain，真正容器中的 Bean(name=”springSecurityFilterChain”) 其实另有其人（这里springboot稍微有点区别，不过不影响理解，我们不纠结这个细节了）。通过 debug，我们发现了真正的 springSecurityFilterChain — FilterChainProxy。

[![delegate](http://kirito.iocoder.cn/C811CC2A-9434-49C8-9240-15BD0EE5A21E.png)](http://kirito.iocoder.cn/C811CC2A-9434-49C8-9240-15BD0EE5A21E.png)delegate

#### FilterChainProxy和SecurityFilterChain

`org.springframework.security.web.FilterChainProxy` 已经是 SpringSecurity 提供的类了，原来它才是真正的 springSecurityFilterChain，我们来看看它的源码（有删减，不影响理解）。

```Java
public class FilterChainProxy extends GenericFilterBean {
   // <1> 包含了多个SecurityFilterChain
   private List<SecurityFilterChain> filterChains;

   public FilterChainProxy(SecurityFilterChain chain) {
      this(Arrays.asList(chain));
   }

   public FilterChainProxy(List<SecurityFilterChain> filterChains) {
      this.filterChains = filterChains;
   }

   @Override
   public void afterPropertiesSet() {
      filterChainValidator.validate(this);
   }

   public void doFilter(ServletRequest request, ServletResponse response,
         FilterChain chain) throws IOException, ServletException {
         doFilterInternal(request, response, chain);
   }

   private void doFilterInternal(ServletRequest request, ServletResponse response,
         FilterChain chain) throws IOException, ServletException {

      FirewalledRequest fwRequest = firewall
            .getFirewalledRequest((HttpServletRequest) request);
      HttpServletResponse fwResponse = firewall
            .getFirewalledResponse((HttpServletResponse) response);
	  // <1>
      List<Filter> filters = getFilters(fwRequest);

      if (filters == null || filters.size() == 0) {
         fwRequest.reset();
         chain.doFilter(fwRequest, fwResponse);
         return;
      }

      VirtualFilterChain vfc = new VirtualFilterChain(fwRequest, chain, filters);
      vfc.doFilter(fwRequest, fwResponse);
   }

   /**
    * <1> 可能会有多个过滤器链，返回第一个和请求URL匹配的过滤器链
    */
   private List<Filter> getFilters(HttpServletRequest request) {
      for (SecurityFilterChain chain : filterChains) {
         if (chain.matches(request)) {
            return chain.getFilters();
         }
      }
      return null;
   }

}
```

看 FilterChainProxy 的名字就可以发现，它依旧不是真正实施过滤的类，它内部维护了一个 SecurityFilterChain，这个过滤器链才是请求真正对应的过滤器链，并且同一个 Spring 环境下，可能同时存在多个安全过滤器链，如 private List filterChains 所示，需要经过 chain.matches(request) 判断到底哪个过滤器链匹配成功，每个 request 最多只会经过一个 SecurityFilterChain。为何要这么设计？因为 Web 环境下可能有多种安全保护策略，每种策略都需要有自己的一条链路，比如我曾经设计过 Oauth2 服务，在极端条件下，可能同一个服务本身既是资源服务器，又是认证服务器，还需要做 Web 安全！

[![多个SecurityFilterChain](http://kirito.iocoder.cn/F0EAD340-B206-4FB0-A660-4CEB28AB8609.png)](http://kirito.iocoder.cn/F0EAD340-B206-4FB0-A660-4CEB28AB8609.png)多个SecurityFilterChain

如上图，4 个 SecurityFilterChain 存在于 FilterChainProxy 中，值得再次强调：实际每次请求，最多只有一个安全过滤器链被返回。

SecurityFilterChain 才是真正意义上的 SpringSecurityFilterChain：

```Java
public final class DefaultSecurityFilterChain implements SecurityFilterChain {
   private final RequestMatcher requestMatcher;
   private final List<Filter> filters;

   public List<Filter> getFilters() {
      return filters;
   }

   public boolean matches(HttpServletRequest request) {
      return requestMatcher.matches(request);
   }
}
```

其中的 List filters 就是我们在 [《Spring Security(四)–核心过滤器源码分析》](https://www.cnkirito.moe/spring-security-4/) 中分析的诸多核心过滤器，包含了 UsernamePasswordAuthenticationFilter，SecurityContextPersistenceFilter，FilterSecurityInterceptor 等之前就介绍过的 Filter。

###SecurityFilterChain的注册过程

还记得 DelegatingFilterProxy 从 Spring 容器中寻找了一个 targetBeanName=springSecurityFilterChain 的 Bean 吗？我们通过 debug 直接定位到了其实现是 SecurityFilterChain，但它又是什么时候被放进去的呢？

这就得说到老朋友 WebSecurity 了，还记得一般我们都会选择使用 @EnableWebSecurity 和 WebSecurityConfigurerAdapter 来进行 web 安全配置吗，来到 WebSecurity 的源码：

```Java
public final class WebSecurity extends
      AbstractConfiguredSecurityBuilder<Filter, WebSecurity> implements
      SecurityBuilder<Filter>, ApplicationContextAware {

    @Override
	protected Filter performBuild() throws Exception {
		int chainSize = ignoredRequests.size() + securityFilterChainBuilders.size();
		List<SecurityFilterChain> securityFilterChains = new ArrayList<SecurityFilterChain>(
				chainSize);
		for (RequestMatcher ignoredRequest : ignoredRequests) {
			securityFilterChains.add(new DefaultSecurityFilterChain(ignoredRequest));
		}
		for (SecurityBuilder<? extends SecurityFilterChain> securityFilterChainBuilder : securityFilterChainBuilders) {
			securityFilterChains.add(securityFilterChainBuilder.build());
		}
        // <1> FilterChainProxy 由 WebSecurity 构建
		FilterChainProxy filterChainProxy = new FilterChainProxy(securityFilterChains);
		if (httpFirewall != null) {
			filterChainProxy.setFirewall(httpFirewall);
		}
		filterChainProxy.afterPropertiesSet();

		Filter result = filterChainProxy;
		postBuildAction.run();
		return result;
	}
}
```

<1> 最终定位到 WebSecurity 的 performBuild 方法，我们之前配置了一堆参数的 WebSecurity 最终帮助我们构建了 FilterChainProxy。

[![WebSecurityConfiguration](http://kirito.iocoder.cn/8E09B17E-EC83-4824-9ED9-AF2814AC6B3A.png)](http://kirito.iocoder.cn/8E09B17E-EC83-4824-9ED9-AF2814AC6B3A.png)WebSecurityConfiguration

并且，最终在 `org.springframework.security.config.annotation.web.configuration.WebSecurityConfiguration`中被注册为默认名称为 SpringSecurityFilterChain。

### 总结

一个名称 SpringSecurityFilterChain，借助于 Spring 的 IOC 容器，完成了 DelegatingFilterProxy 到 FilterChainProxy 的连接，并借助于 FilterChainProxy 内部维护的 List 中的某一个 SecurityFilterChain 来完成最终的过滤。

# 666. 彩蛋

如果你对 Spring Security 感兴趣，欢迎加入我的知识一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)