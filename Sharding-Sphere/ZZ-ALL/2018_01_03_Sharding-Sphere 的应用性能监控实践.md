title: Sharding-Sphere 的应用性能监控实践
date: 2018-01-03
tags:
categories: Sharding Sphere
permalink: Sharding-Sphere/Application-performance-monitoring-practice-of-Sharding-Sphere
author: 陈清阳
from_url: https://mp.weixin.qq.com/s/FHRnLwUGD32a7mAPNqJ3ug

---

摘要: 原创出处 https://mp.weixin.qq.com/s/FHRnLwUGD32a7mAPNqJ3ug 「陈清阳」欢迎转载，保留摘要，谢谢！

- [1. Sharding-Opentracing简介](http://www.iocoder.cn/Sharding-Sphere/Application-performance-monitoring-practice-of-Sharding-Sphere/)
- [2. 基于Opentracing规范的性能追踪——SkyWalking实践](http://www.iocoder.cn/Sharding-Sphere/Application-performance-monitoring-practice-of-Sharding-Sphere/)
- [3. 基于字节码增强的性能追踪——Pinpoint实践](http://www.iocoder.cn/Sharding-Sphere/Application-performance-monitoring-practice-of-Sharding-Sphere/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

Sharding-Sphere是一套开源的分布式数据库中间件解决方案组成的生态圈，它为分布式数据存储提供了多种形式的解决方案。其中的Sharding-JDBC产品在甜橙金融的技术架构中得到了广泛应用，并且在业内已成为最重要的解决方案之一。在使用Sharding-JDBC的过程中，少不了性能测试与运维性能监控的工作。作为Sharding-Sphere开源项目的重要支持者，甜橙金融技术创新中心团队很荣幸的参与并负责解决Sharing-JDBC的应用性能监控问题。本文将针对这个问题，进行Sharding-Sphere与APM之间结合的实践探索。

——前言


# 1. Sharding-Opentracing简介

Sharding-Opentracing是Sharding-Sphere为大家提供的基于Opentracing规范的APM工具包。它采用发布 - 订阅式的组件通信机制实现了对核心链路方法的跟踪记录，包括sql解析路由方法、sql执行方法和结果归并方法。

Opentracing（https://github.com/opentracing）规范的产生，目的是为了解决不同的分布式追踪系统API不兼容的问题。它是一个轻量级的标准化层，位于应用程序/类库和追踪或日志分析程序之间。

Sharding-Sphere使用Opentracing进行方法的追踪，使得系统开发人员可以方便的添加或替换追踪系统的实现。使用ShardingJDBCTracer类可以方便的完成Tracer的注入。



```java
public final class ShardingJDBCTracer {

   /**
    * 通过读取系统属性进行注册
    * -Dshardingjdbc.opentracing.tracer.class=*****
    */
   public static void init() {...}

   /**
    * 通过方法参数进行注册
    */
   public static void init(final Tracer tracer) {...}

}
```

然后通过事件监听器订阅核心方法事件，完成方法拦截实现方法追踪。目前实现的事件监听器如下：

- SqlRoutingEventListener

  Sql路由事件追踪

- ExecuteEventListener

  Sql执行事件追踪

- MergeEventListener

  Sql执行结果合并事件追踪



# 2. 基于Opentracing规范的性能追踪——SkyWalking实践



SkyWalking（https://github.com/apache/incubator-skywalking）是由国人大神吴晟创建的一款开源的针对分布式系统的APM系统，它提供了Opentracing标准的支持。下面我们通过SkyWalking来看看如何在你的应用中使用Sharding-Opentracing工具包。

在应用系统中引入相关依赖包

```xml
<dependency>
  <groupId>io.shardingsphere</groupId>
  <artifactId>sharding-opentracing</artifactId>
  <version>3.0.0.M2-SNAPSHOT</version>
</dependency>
<dependency>
  <groupId>org.apache.skywalking</groupId>
  <artifactId>apm-toolkit-opentracing</artifactId>
  <version>5.0.0-beta</version>
</dependency>
```

SkyWalking的apm-toolkit-opentracing包中的SkyWalkingTracer实现了Tracer接口，在应用系统启动时完成Tracer的注入。

```java
ShardingJDBCTracer.init(new SkywalkingTracer());
```

通过简单的两步就完成了SkyWalking的接入。通过SkyWalking的Trace视图可以查看业务请求的调用链信息,包括Sharding-JDBC在sql路由，sql执行和结果归并的执行信息。其中sql执行是采用多条sql并行执行的策略，这里也可以看到每个线程的执行情况。

![img](http://static.iocoder.cn/3e97324ed3714eb696010d46edc2fce4)

通过SkyWalking的Span Info视图可以看到每个追踪Span的相关参数信息。包括路由库信息、sql解析语句与执行入参等。

![img](http://static.iocoder.cn/7b44278d9c8faf712ba17f3495e15b17)



# 3. 基于字节码增强的性能追踪——Pinpoint实践

上面我们了解了如何使用实现了Opentracing规范的APM框架进行Sharding-JDBC的性能监控，那么对于没有实现Opentracing规范，采用字节码增强技术的APM框架，我们又该如何入手了？针对这种采用字节码增强技术的APM框架，我们的策略是编写对应的插件，实现对Sharding-Sphere核心方法的增强。接下来我们看看如何在Pinpoint框架中实现一个针对Sharding-Sphere组件的插件。

实现Interceptor接口，完成组件方法拦截器。在组件方法执行前开启追踪，在组件方法执行后结束追踪，这里展示结果归并方法追踪的实现。

```java
public class ResultSetMergeInterceptor implements AroundInterceptor1 {

   private final TraceContext traceContext;
   private final MethodDescriptor descriptor;
   private final PLogger logger = PLoggerFactory.getLogger(getClass());

   public ResultSetMergeInterceptor(TraceContext traceContext, MethodDescriptor descriptor) {
       this.traceContext = traceContext;
       this.descriptor = descriptor;
   }

   @Override
   public void before(Object target, Object arg0) {
       if (logger.isDebugEnabled()) {
           logger.beforeInterceptor(target, new Object[]{arg0});
       }
       final Trace trace = traceContext.currentTraceObject();
       if (trace == null) {
           return;
       }
       trace.traceBlockBegin();
   }

   @Override
   public void after(Object target, Object arg0, Object result, Throwable throwable) {
       if (logger.isDebugEnabled()) {
           logger.afterInterceptor(target, new Object[]{arg0}, result, throwable);
       }
       final Trace trace = traceContext.currentTraceObject();
       if (trace == null) {
           return;
       }
       try {
           SpanEventRecorder recorder = trace.currentSpanEventRecorder();
           recorder.recordServiceType(ShardingSphereConstants.SHARDING_SPHERE_MERGE);
           recorder.recordApi(descriptor);
           recorder.recordException(throwable);
       } finally {
           trace.traceBlockEnd();
       }
   }
}
```

实现ProfilerPlugin和TransformTemplateAware接口，完成插件标识和转换模版注册。其中需要通过TransformTemplate的transform方法来完成追踪方法和方法拦截器的绑定关系。例如ShardingPreparedStatement类的merge方法与上面实现的ResultSetMergeInterceptor进行绑定。

```java
public class ShardingSpherePlugin implements ProfilerPlugin, TransformTemplateAware {

   private final PLogger logger = PLoggerFactory.getLogger(this.getClass());

   private static final String SHARDINGSPHERE_SCOPE = "SHARDINGSPHERE_SCOPE";

   private TransformTemplate transformTemplate;

   @Override
   public void setup(ProfilerPluginSetupContext context) {
       ShardingSphereConfig config = new ShardingSphereConfig(context.getConfig());
       logger.debug("[ShardingSphere] pluginEnable={}", config.isPluginEnable());
       if (config.isPluginEnable()) {
           addSqlRouteTransformer();
           addSqlExecutorTransformer();
           addResultMergeTransformer();
       }
   }

   @Override
   public void setTransformTemplate(TransformTemplate transformTemplate) {
       this.transformTemplate = transformTemplate;
   }

   private void addResultMergeTransformer() {
       TransformCallback transformCallback = new TransformCallback() {
           @Override
           public byte[] doInTransform(Instrumentor instrumentor, ClassLoader classLoader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws InstrumentException {
               InstrumentClass target = instrumentor.getInstrumentClass(classLoader, className, classfileBuffer);
               InstrumentMethod method = target.getDeclaredMethod("merge", "io.shardingsphere.core.merger.MergeEngine");
               method.addScopedInterceptor("com.navercorp.pinpoint.plugin.shardingsphere.interceptor.ResultSetMergeInterceptor", SHARDINGSPHERE_SCOPE);
               return target.toBytecode();
           }
       };
       transformTemplate.transform("io.shardingsphere.core.jdbc.core.statement.ShardingPreparedStatement", transformCallback);
       transformTemplate.transform("io.shardingsphere.core.jdbc.core.statement.ShardingStatement", transformCallback);
   }

}
```

完整插件代码请参考：

https://github.com/beckhampu/pinpoint/tree/sharding-sphere-1.7.2/plugins

通过Pinpoint的CallTree效果图，我们可以清楚的看到一次业务过程中Sharding-JDBC核心方法的调用效果。

![img](http://static.iocoder.cn/ad0c487be206f57a76a88d086a59b770)

至此，关于Sharding-Sphere与APM的相关实践就介绍到这里。希望自己的一得之见可以让大家有所收获。如果大家有什么想法、意见和建议，欢迎留言与我交流。在这里也呼吁大家一起贡献自己的力量，让Sharding-Sphere发展的更好。
