title: Spring 5 源码解析 —— Spring 中的处理器 handlers
date: 2018-01-09
tag: 
categories: Spring
permalink: Spring/handlers
author: 一叶知秋
from_url: https://muyinchen.github.io/2017/08/05/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— Spring%E4%B8%AD%E7%9A%84%E5%A4%84%E7%90%86%E5%99%A8handlers/
wechat_url: 

-------

摘要: 原创出处 https://muyinchen.github.io/2017/08/05/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90 —— Spring%E4%B8%AD%E7%9A%84%E5%A4%84%E7%90%86%E5%99%A8handlers/ 「一叶知秋」欢迎转载，保留摘要，谢谢！

- [Spring中的两种handler类型](http://www.iocoder.cn/Spring/handlers/)
- [Spring框架中的handler](http://www.iocoder.cn/Spring/handlers/)
- [编写自定义的Spring handler程序](http://www.iocoder.cn/Spring/handlers/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

> Spring Web应用程序的中心站是`DispatcherServlet`(具体请查看[Spring5源码解析-论Spring DispatcherServlet的生命周期](https://muyinchen.github.io/2017/08/02/Spring5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-%E8%AE%BASpring%20DispatcherServlet%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F/))。这是所有传入请求的中心入口。但如果没有但如果没有众多的handlers，我们依然不能做任何事情。

首先，在本文中，我们先解读`handler`到底是个啥。之后，我们将介绍一些`Spring`框架中`handlers`的两种处理类型。最后我们加点`salt`来让我们学的东西进行落地，我们将编写我们自己的handler。

## Spring中的两种handler类型

首先，在Spring的世界中，这些`handler`到底做了些什么。简单的讲，这个就和我们听到一句话或者看到某个场景，然后有相关的反应是一样的，由很多处理最后转换到我们大脑皮层所能理解的东西。从机器语言的角度就是词法分析，语法分析，好咯，大家知道编译语言的重要性也就是基础的重要性了吧，回到框架中来，对于Spring来讲，这些处理程序就是一种将用户操作转换为Spring可以理解的元素。说到`用户操作`，我们可以考虑像`http://xxx.com/login`这样的URL类型。而我们的`handler`，在这里作为翻译处理，将尝试找到为此地址应该调用哪个控制器来处理。通常我们写`Spring controller`代码都知道，处理程序可以查找`@RequestMapping`的注解，并检查哪些映射与`/login` 这个URL匹配。由上一篇文章我们可以知道，这个处理程序将在`DispatcherServlet`的内被调用。

更准确地说，Spring中存在两种类型的handlers。第一种是**handler mappings(处理程序映射)**。它们的角色定位与前面所描述的功能完全相同。它们尝试将当前请求与相应的`controller`以及其中的方法相匹配。第二种是**handler adapter(处理器适配器)**。`handler adapter`从`handler mappings`中获取映射的`controllers` 和方法并调用它们。这种类型的适配器必须实现**org.springframework.web.servlet.HandlerAdapter**接口，它只有3种方法：

- **boolean supports(Object handler)**:检查传入参数的对象是否可以由此适配器处理
- **ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)** : 将请求翻译成视图。
- **long getLastModified(HttpServletRequest request, Object handler)**:返回给定`HttpServletRequest`的最后修改日期，以毫秒为单位。

但这里要注意的是，在Spring版本中有一些重要变化。作为`DefaultAnnotationHandlerMapping`，`AnnotationMethodHandlerAdapter`或`AnnotationMethodHandlerExceptionResolver`的处理程序适配器自Spring 3.2版本以来已经废弃，在`Spring4.x`里还可以看到，在`Spring5`内已经删除掉了，替代品为`RequestMappingHandlerMapping`，`RequestMappingHandlerAdapter`和`ExceptionHandlerExceptionResolver`。通过这些新类以便于自定义映射。另外，通过在`since 3.1` 版本中**org.springframework.web.method.HandlerMethod**类中引入，来将所处理的对象转换为其方法表示。我们可以通过这个方法来判断对象返回的类型或者哪些参数是我们所期望的(看着拗口的话请打开源码查看此类注释)。

## Spring框架中的handler

除了已经提供的处理程序适配器之外，Spring也有本地处理程序映射，最基本的处理程序映射器是**org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping**类。它将`URL`与相应的bean进行匹配。例如，请看下面的配置：

```xml
<bean name="/friends" class="com.migo.controller.FriendsController" />
```

正如你所看到的，这种配置在很多URL的情况下是很不实用的。一些更灵活的处理映射器是**org.springframework.web.servlet.handler.SimpleUrlHandlerMapping**。而不是为每个请求创建`bean`，我们可以创建一个映射文件，其中包含URL作为键和`controller`作为值，看下面的配置:

```xml
<bean id="simpleUrlMapping" class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping">
  <property name="mappings">
    <props>
      <prop key="/friends.html">FriendsController</props>
  </property>
</bean>
```

但是，在`SimpleUrlHandlerMapping`中，处理稍微复杂URL也是一个头疼的问题。这也是为什么我们要用`DefaultAnnotationHandlerMapping`或者在最新的`Spring`版本中使用

`org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping`的

原因。它们的映射检测是基于注解。这样，所有的逻辑都保留在Java代码这块，例如：

```java
@Controller
public class FriendsController {

    @RequestMapping(value = "/friends.html", method = RequestMethod.GET)
    public String showFriendsList() {
        return "friendsListView";
    }

    @RequestMapping(value = "/friends/potential-friends.html" method = RequestMethod.GET)
    public String showPotentialFriends() {
        return "potentialFriendsView";
    }
}
```

与以前的处理程序不同的是，基于注解允许更灵活的配置。不仅不需要在`XML`进行各种繁琐的配置，一旦`URL`很多的情况下，想象一下`XML`,各种头大，现在通过注解，我们可以把一条路上不同岔口的`URL`在一个`controller`里进行接收处理就好。当配置文件中定义`<mvc:annotation-driven/>`时，此处理程序将被激活。另外，为了更细粒度的处理`controller`注解，我们可以通过添加`<context:annotation-config />`(有后面这个之后此处也可以省略，后者本身 就具有此功能)和`<context:component-scan base-package =“path.with.my.services.and.controllers”/>`来启用它们。

## 编写自定义的Spring handler程序

现在我们更深入了解一下Spring mapping handlers。我们来实现个我们自己的URL处理程序。其实很简单(因为只需要达到最基本的处理目的就可以了) ，我们将替换`RequestMappingHandlerMapping`，并使一个简单的映射器来处理URL地址。我们的映射器将只处理静态URL，如:`/home.html`。它无须也无法从方法签名中获取动态参数以及也无须知道`@PathVariable`元素。主要目标是让大家从中发现Spring处理一个请求所进行的步骤。

我们这个`handler`将扩展`RequestMappingHandlerMapping`并覆盖其方法(有些方法可以从`RequestMappingInfoHandlerMapping`找到，其实就是重写或实现`AbstractHandlerMethodMapping`里的几个抽象方法)：

- **protected void registerHandlerMethod(Object handler，Method method，RequestMappingInfo mapping)**:
- **protected boolean isHandler(Class beanType)**: 检查bean是否符合给定处理程序的条件。
- **protected RequestMappingInfo getMappingForMethod(Method method，Class handlerType)**: 为给定的Method实例提供映射的方法，该方法表示处理的方法（例如，使用`@RequestMapping`注解的`controller`的方法上所对应的`URL`）。


- **protected HandlerMethod handleNoMatch(Set requestMappingInfos, String lookupPath, HttpServletRequest request)** : 在给定的`HttpServletRequest`对象找不到匹配的处理方法时被调用。
- **protected void handleMatch(RequestMappingInfo info, String lookupPath, HttpServletRequest request)** : 当为给定的`HttpServletRequest`对象找到匹配的处理方法时调用。

在写这个`handler`之前，让我们写个自定义的`@RequestMapping`的注解：

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DumberRequestMapping {
    String value() default "";
}
```

唯一的属性是代表`URL路径`的值，与`@RequestMapping`注解中的`value属性`完全相同。现在我们可以传入我们的处理程序映射类。该课程在内部进行评论。这就是为什么它不会在通常的“文本模式”中包含任何补充评论。

```java
public class DumberRequestHandlerMapping extends RequestMappingHandlerMapping {
    private static final Logger LOGGER = LoggerFactory.getLogger(DumberRequestHandlerMapping.class);

    /**
     * Checks if handler should be applied to given bean's class. The check is made through looking for DumberRequestMapping annotation.
     */
    @Override
    protected boolean isHandler(Class<?> beanType) {
        Method[] methods = ReflectionUtils.getAllDeclaredMethods(beanType);
        for (Method method : methods) {
            if (AnnotationUtils.findAnnotation(method, DumberRequestMapping.class) != null) {
                LOGGER.debug("[DumberRequestHandlerMapping] Method "+method+" supports @DumberRequestMapping ");
                return true;
            }
        }
        return false;
    }

    /**
     * Make some operations directly before returning HttpServletRequest instance into mapped controller's method. For example, if you add here some attributes to this object, those attributes will be reachable from controller's method which handles the request.
     * RequestMappingInfoHandlerMapping does some of more complicated stuff here like exposing URI template variables or extracting
     * "matrix variable".
     * NOTE : "matrix variables" are name-value pairs within path segments, separated with a semicolon (;). For example in this URL
     * /clubs;country=France;division=Ligue 1, Ligue 2) we can find 2 matrix variables: country (France) and division (list composed by
     * Ligue 1 and Ligue 2)
     */
    @Override
    protected void handleMatch(RequestMappingInfo info, String lookupPath, HttpServletRequest request) {
        LOGGER.debug("[DumberRequestHandlerMapping] handleMatch info "+info+  ", lookupPath ="+ lookupPath + ", request ="+request);
        request.setAttribute("isDumber", true);
        request.setAttribute("handledTime", System.nanoTime());
    }

    /**
     * Method invoked when given lookupPath doesn't match with this handler mapping.
     * Native RequestMappingInfoHandlerMapping uses this method to launch two exceptions :
     * - HttpRequestMethodNotSupportedException - if some URLs match, but no theirs HTTP methods.
     * - HttpMediaTypeNotAcceptableException - if some URLs match, but no theirs content types. For example, a handler can match an URL
     * like /my-page/test, but can expect that the request should be send as application/json. Or, the handler can match the URL but
     * returns an inappropriate response type, for example: text/html instead of application/json.
     */
    @Override
    protected HandlerMethod handleNoMatch(Set<RequestMappingInfo> requestMappingInfos, String lookupPath, HttpServletRequest request) throws ServletException {
        LOGGER.debug("[DumberRequestHandlerMapping] handleNoMatch info "+requestMappingInfos+  ", lookupPath ="+ lookupPath + ", request ="+request);
        return null;
    }

    /**
     * Here we constructs RequestMappingInfo instance for given method.
     * RequestMappingInfo - this object is used to encapsulate mapping conditions. For example, it contains an instance of
     * PatternsRequestCondition which  is used in native Spring's RequestMappingInfoHandlerMapping  handleMatch() method to put URI
     * variables into @RequestMapping pattern.
     * Ie, it will take the following URL /test/1 and match it for URI template /test/{id}. In occurrence, it will found that 1
     * corresponding to @PathVariable represented  by id variable ({id}) and will set its value to 1.
     */
    @Override
    protected RequestMappingInfo getMappingForMethod(Method method, Class<?> handlerType) {
        LOGGER.debug("[DumberRequestHandlerMapping] getMappingForMethod method "+method+  ", handlerType ="+handlerType);
        RequestMappingInfo info = null;
        // look for @DumberRequestMapping annotation for the Method method from signature
        DumberRequestMapping methodAnnotation = AnnotationUtils.findAnnotation(method, DumberRequestMapping.class);
        if (methodAnnotation != null) {
            RequestCondition<?> methodCondition = getCustomMethodCondition(method);
            info = createRequestMappingInfo(methodAnnotation, methodCondition);
        }
        LOGGER.debug("[DumberRequestHandlerMapping] getMappingForMethod method; returns info mapping "+info);
        return info;
    }

    /**
     * Creates RequestMappingInfo object which encapsulates:
     * - PatternsRequestCondition: represents URI template to resolve. Resolving is helped by UrlPathHelper utility class from
     * package org.springframework.web.util.
     * - RequestMethodsRequestCondition: methods accepted by this handler. You can make a test and replace RequestMethod.GET by
     * RequestMethod.POST. You will able to observe that our test won't work.
     * - ParamsRequestCondition:
     * - HeadersRequestCondition: headers which should be send in request to given handler should handle this request. You can,
     * for exemple, put there an header value like "my-header:test" and observe the program behavior.
     * - ConsumesRequestCondition: this condition allows to specify the content-type of request. We can use it for, for example,
     * specify that a method can be handled only for application/json request.
     * - ProducesRequestCondition: this condition allows to specify the content-type of response. We can use it for, for example,
     * specify that a method can be applied only for text/plain response.
     */
    protected RequestMappingInfo createRequestMappingInfo(DumberRequestMapping annotation, RequestCondition<?> customCondition) {
        return new RequestMappingInfo(
                new PatternsRequestCondition(new String[] {annotation.value()}),
                new RequestMethodsRequestCondition(new RequestMethod[]{RequestMethod.GET}),
                new ParamsRequestCondition(new String[]{}),
                new HeadersRequestCondition(new String[] {}),
                new ConsumesRequestCondition(new String[]{}, new String[]{}),
                new ProducesRequestCondition(new String[]{}, new String[]{}, getContentNegotiationManager()),
                customCondition);
    }

}
```

我们需要向我们的应用程序上下文添加新的`HandlerMapping`。请看下面这个基于XML的配置：

```xml
<bean class="com.mypackage.handler.DumberRequestHandlerMapping">//此处根据自己的包进行配置
  <property name="order" value="0" />
</bean>
```

请注意，order属性的存在确定了按顺序将请求由`HandlerMapping`处理。在这里，如果`DumberRequestHandlerMapping`可以应用于一个请求，Spring将立即使用它，而不需要寻找另一个可用的处理程序。

最后一件事是使用`@DumberRequestMapping`在方法上添加注解：

```java
@Controller
public class TestController {
 private static final Logger LOGGER = LoggerFactory.getLogger(TestController.class);
    @DumberRequestMapping(value = "/test")
    public String testSession(HttpServletRequest request) {
        LOGGER.debug("Is dumber request ?"+request.getAttribute("isDumber"));
        LOGGER.debug("Handled time ?"+request.getAttribute("handledTime"));
        return "testTemplate";
    }

}
```

通过执行`http://localhost:8084/test`，您将看到在`DumberRequestHandlerMapping`的`handleMatch`方法中设置的请求的属性存在。如果您部署有应用程序的日志，您将看到有关controller执行流程的一些信息：

```shell
2017-08-05 23:31:00,027 [http-bio-8084-exec-1] [org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping]-[DEBUG] Looking up handler method for path /test   //先在RequestMappingHandlerMapping找的，也就是先找有@RequestMapping注解相应处理逻辑的方法来处理
2017-08-05 23:31:00,028 [http-bio-8084-exec-1] [org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping]-[DEBUG] Did not find handler method for [/test]   //在RequestMappingHandlerMapping中没找到相应的处理逻辑
2017-08-05 23:31:00,028 [http-bio-8084-exec-1] [com.migo.sso.DumberRequestHandlerMapping]-[DEBUG] Looking up handler method for path /test
  //从DumberRequestHandlerMapping里找，发现@DumberRequestMapping所注解的方法可以处理，那就处理咯
2017-08-05 23:31:00,029 [http-bio-8084-exec-1] [com.migo.sso.DumberRequestHandlerMapping]-[DEBUG] [DumberRequestHandlerMapping] handleMatch info {[/test],methods=[GET],params=[],headers=[],consumes=[],produces=[],custom=[]}, lookupPath =/test, request =org.apache.catalina.connector.RequestFacade@24a7274b
2017-08-05 23:31:00,030 [http-bio-8084-exec-1] [com.migo.sso.DumberRequestHandlerMapping]-[DEBUG] Returning handler method [public java.lang.String com.migo.sso.controller.TestController.testSession(javax.servlet.http.HttpServletRequest)]
2017-08-05 23:31:00,030 [http-bio-8084-exec-1] [org.springframework.beans.factory.support.DefaultListableBeanFactory]-[DEBUG] Returning cached instance of singleton bean 'testController'
2017-08-05 23:31:00,030 [http-bio-8084-exec-1] [org.springframework.web.servlet.DispatcherServlet]-[DEBUG] Last-Modified value for [/test] is: -1
2017-08-05 23:31:00,040 [http-bio-8084-exec-1] [com.migo.sso.controller.TestController]-[DEBUG] Is dumber request ?true
2017-08-05 23:31:00,040 [http-bio-8084-exec-1] [com.migo.sso.controller.TestController]-[DEBUG] Handled time ?21230126522470
Handled time ?17452005683775
```

我们可以看到， handler mapping是Spring生态系统中的一个关键概念。所有的URL都由对应的处理程序处理，由此，Spring可以匹配传入的HTTP请求和所加注解配置的controller的方法。我们也看到了如何根据不同规则来过滤请求，例如：Content-Type，Accept或其他headers 或HTTP方法。我们还编写了一个poor版本的Spring的`RequestMappingInfoHandlerMapping`，它拦截一些URL处理并将结果通过视图输出给用户。

总结起来就是，通过一定的方式确定相应请求的处理位置(我们通常通过注解来确定)，仅此而已，啰嗦了太多的东西，最后也就是如此的直白

# 666. 彩蛋

如果你对 Spring 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)