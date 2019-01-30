title: Spring-Cloud-Gateway 源码解析 —— 路由（1.3）之 RouteDefinitionRepository 存储器
date: 2020-01-20
tags:
categories: Spring-Cloud-Gateway
permalink: Spring-Cloud-Gateway/route-definition-locator-repository

---

摘要: 原创出处 http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-repository/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 Spring-Cloud-Gateway 2.0.X M4**  

- [1. 概述](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-repository/)
- [2. RouteDefinitionWriter](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-repository/)
- [3. RouteDefinitionRepository](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-repository/)
- [4. InMemoryRouteDefinitionRepository](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-repository/)
- [5. GatewayWebfluxEndpoint](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-repository/)
- [6. 自定义 RouteDefinitionRepository](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-repository/)
- [666. 彩蛋](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-repository/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2018_05_18.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

---

# 1. 概述

本文主要对 **RouteDefinitionRepository 的源码实现**。

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_01_20/01.jpeg)

* **蓝色**部分 ：RouteDefinitionRepository 。

本文涉及到的类图如下 ：

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_01_20/02.png)

* 下面我们来逐个类进行解析。

-------

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。



# 2. RouteDefinitionWriter

`org.springframework.cloud.gateway.route.RouteDefinitionWriter` ，路由配置写入**接口**。该接口定义了**保存**与**删除**两个方法，代码如下 ：

```Java
public interface RouteDefinitionWriter {

    /**
     * 保存路由配置
     *
     * @param route 路由配置
     * @return Mono<Void>
     */
    Mono<Void> save(Mono<RouteDefinition> route);

    /**
     * 删除路由配置
     *
     * @param routeId 路由编号
     * @return Mono<Void>
     */
    Mono<Void> delete(Mono<String> routeId);
}
```

* 该接口有什么用呢？我们继续往下看。

# 3. RouteDefinitionRepository

`org.springframework.cloud.gateway.route.RouteDefinitionRepository` ，存储器 RouteDefinitionLocator **接口**，代码如下 ：

```Java
public interface RouteDefinitionRepository extends RouteDefinitionLocator, RouteDefinitionWriter {
}
```

* 继承 RouteDefinitionLocator **接口**。
* 继承 RouteDefinitionWriter **接口**。

通过实现该接口，实现从**存储器**( 例如，内存 / Redis / MySQL 等 )读取、保存、删除路由配置。

目前 Spring Cloud Gateway 实现了**基于内存为存储器**的 InMemoryRouteDefinitionRepository 。

# 4. InMemoryRouteDefinitionRepository

`org.springframework.cloud.gateway.route.InMemoryRouteDefinitionRepository` ，**基于内存为存储器**的 RouteDefinitionLocator ，代码如下 ：

```Java
public class InMemoryRouteDefinitionRepository implements RouteDefinitionRepository {

    /**
     * 路由配置映射
     * key ：路由编号 {@link RouteDefinition#id}
     */
    private final Map<String, RouteDefinition> routes = synchronizedMap(new LinkedHashMap<String, RouteDefinition>());

    @Override
    public Mono<Void> save(Mono<RouteDefinition> route) {
        return route.flatMap( r -> {
            routes.put(r.getId(), r);
            return Mono.empty();
        });
    }

    @Override
    public Mono<Void> delete(Mono<String> routeId) {
        return routeId.flatMap(id -> {
            if (routes.containsKey(id)) {
                routes.remove(id);
                return Mono.empty();
            }
            return Mono.error(new NotFoundException("RouteDefinition not found: "+routeId));
        });
    }

    @Override
    public Flux<RouteDefinition> getRouteDefinitions() {
        return Flux.fromIterable(routes.values());
    }
}
```

* 代码比较易懂，瞅瞅就好。
* `InMemoryRouteDefinitionRepository#getRouteDefinitions()` 方法的调用，我们已经在 CompositeRouteDefinitionLocator 看到。
* `InMemoryRouteDefinitionRepository#save()` / `InMemoryRouteDefinitionRepository#delete()` 方法，下面在 GatewayWebfluxEndpoint 可以看到。

# 5. GatewayWebfluxEndpoint

`org.springframework.cloud.gateway.actuate.GatewayWebfluxEndpoint` ，提供**管理**网关的 HTTP API 。代码如下 ：

```Java
@RestController
@RequestMapping("${management.context-path:/application}/gateway")
public class GatewayWebfluxEndpoint implements ApplicationEventPublisherAware {

    /**
     * 存储器 RouteDefinitionLocator 对象
     */
    private RouteDefinitionWriter routeDefinitionWriter;

    // ... 省略代码

}
```

* 从注解 `@RestController` 我们可以得知，GatewayWebfluxEndpoint 是一个 **Controller** 。

GatewayWebfluxEndpoint 有**两个** HTTP API 调用了 RouteDefinitionWriter 的**两个**方法。

* `POST "/routes/{id}"` ，保存路由配置，代码如下 ：

    ```Java
    @PostMapping("/routes/{id}")
    @SuppressWarnings("unchecked")
    public Mono<ResponseEntity<Void>> save(@PathVariable String id, @RequestBody Mono<RouteDefinition> route) {
    	return this.routeDefinitionWriter.save(route.map(r ->  { // 设置 ID
    		r.setId(id);
    		log.debug("Saving route: " + route);
    		return r;
    	})).then(Mono.defer(() -> // status ：201 ，创建成功。参见 HTTP 规范 ：https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/201
    		Mono.just(ResponseEntity.created(URI.create("/routes/"+id)).build())
    	));
    }
    ```
    * 例如，HTTP 请求如下 ：

        > http POST :8080/application/gateway/routes/apiaddreqhead uri=http://httpbin.org:80 predicates:='["Host=**.apiaddrequestheader.org", "Path=/headers"]' filters:='["AddRequestHeader=X-Request-ApiFoo, ApiBar"]'    

* `DELETE "/routes/{id}"` ，删除路由配置，代码如下 ：

    ```Java
    @DeleteMapping("/routes/{id}")
    public Mono<ResponseEntity<Object>> delete(@PathVariable String id) {
    	return this.routeDefinitionWriter.delete(Mono.just(id))
    			.then(Mono.defer(() -> Mono.just(ResponseEntity.ok().build()))) // 删除成功
    			.onErrorResume(t -> t instanceof NotFoundException, t -> Mono.just(ResponseEntity.notFound().build())); // 删除失败
    }
    ```

# 6. 自定义 RouteDefinitionRepository

使用 InMemoryRouteDefinitionRepository 来维护 RouteDefinition 信息，在网关实例重启或者崩溃后，RouteDefinition 就会丢失。此时我们可以实现 RouteDefinitionRepository **接口**，以实现例如 MySQLRouteDefinitionRepository 。

通过类似 MySQL 等**持久化**、**可共享**的存储器，也可以带来 Spring Cloud Gateway 实例**集群**获得一致的、相同的 RouteDefinition 信息。

另外，我们看到 RouteDefinitionRepository 初始化的代码如下 ：

```Java
// GatewayAutoConfiguration.java
@Bean // 4.2
@ConditionalOnMissingBean(RouteDefinitionRepository.class)
public InMemoryRouteDefinitionRepository inMemoryRouteDefinitionRepository() {
    return new InMemoryRouteDefinitionRepository();
}
```

* 注解 `@ConditionalOnMissingBean(RouteDefinitionRepository.class)` ，当**不存在** RouteDefinitionRepository 的 Bean 对象时，初始化 InMemoryRouteDefinitionRepository 。也就是说，我们可以初始化自定义的 RouteDefinitionRepository 以**"注入"** 。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

比较干爽( 水更 )的一篇文章。

胖友，分享一波朋友圈可好！

