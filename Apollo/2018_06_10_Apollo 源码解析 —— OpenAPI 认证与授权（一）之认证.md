title: Apollo 源码解析 —— OpenAPI 认证与授权（一）之认证
date: 2018-06-10
tags:
categories: Apollo
permalink: Apollo/openapi-auth-1

-------

摘要: 原创出处 http://www.iocoder.cn/Apollo/openapi-auth-1/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Apollo/openapi-auth-1/)
- [3. 实体](http://www.iocoder.cn/Apollo/openapi-auth-1/)
  - [3.1 Consumer](http://www.iocoder.cn/Apollo/openapi-auth-1/)
  - [3.2 ConsumerToken](http://www.iocoder.cn/Apollo/openapi-auth-1/)
  - [3.3 ConsumerAudit](http://www.iocoder.cn/Apollo/openapi-auth-1/)
- [4. ConsumerAuthenticationFilter](http://www.iocoder.cn/Apollo/openapi-auth-1/)
  - [4.1 AuthFilterConfiguration](http://www.iocoder.cn/Apollo/openapi-auth-1/)
- [5. Util](http://www.iocoder.cn/Apollo/openapi-auth-1/)
  - [5.1 ConsumerAuthUtil](http://www.iocoder.cn/Apollo/openapi-auth-1/)
  - [5.2 ConsumerAuditUtil](http://www.iocoder.cn/Apollo/openapi-auth-1/)
- [6. ConsumerService](http://www.iocoder.cn/Apollo/openapi-auth-1/)
  - [6.1 构造方法](http://www.iocoder.cn/Apollo/openapi-auth-1/)
  - [6.2 createConsumer](http://www.iocoder.cn/Apollo/openapi-auth-1/)
  - [6.3 generateAndSaveConsumerToken](http://www.iocoder.cn/Apollo/openapi-auth-1/)
  - [6.4 其他方法](http://www.iocoder.cn/Apollo/openapi-auth-1/)
- [7. ConsumerController](http://www.iocoder.cn/Apollo/openapi-auth-1/)
- [666. 彩蛋](http://www.iocoder.cn/Apollo/openapi-auth-1/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2018_05_18.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

# 1. 概述

> 老艿艿：本系列假定胖友已经阅读过 [《Apollo 官方 wiki 文档》](https://github.com/ctripcorp/apollo/wiki/) ，特别是 [《Apollo 开放平台》](https://github.com/ctripcorp/apollo/wiki/Apollo%E5%BC%80%E6%94%BE%E5%B9%B3%E5%8F%B0) 。

考虑到 [Portal 的认证与授权](#)  分成了两篇，所以本文分享 OpenAPI 的认证与授权， **侧重在认证部分**。

在 [《Apollo 开放平台》](https://github.com/ctripcorp/apollo/wiki/Apollo%E5%BC%80%E6%94%BE%E5%B9%B3%E5%8F%B0) 文档的开头：

> Apollo 提供了一套的 Http REST 接口，使第三方应用能够自己管理配置。虽然 Apollo 系统本身提供了 Portal 来管理配置，但是在有些情景下，应用需要通过程序去管理配置。

* OpenAPI 和 Portal 都在 `apollo-portal` 项目中，但是他们是**两套** API ，包括 `package` 都是两个不同的，如下图所示：![项目结构](http://www.iocoder.cn/images/Apollo/2018_06_10/01.png)

# 3. 实体

## 3.1 Consumer

**Consumer** 表，对应实体 `com.ctrip.framework.apollo.openapi.entity.Consumer` ，代码如下：

```Java
@Entity
@Table(name = "Consumer")
@SQLDelete(sql = "Update Consumer set isDeleted = 1 where id = ?")
@Where(clause = "isDeleted = 0")
public class Consumer extends BaseEntity {

    /**
     * 应用名称
     */
    @Column(name = "Name", nullable = false)
    private String name;
    /**
     * 应用编号
     *
     * 注意，和 {@link com.ctrip.framework.apollo.common.entity.App} 不是一个东西
     */
    @Column(name = "AppId", nullable = false)
    private String appId;
    /**
     * 部门编号
     */
    @Column(name = "OrgId", nullable = false)
    private String orgId;
    /**
     * 部门名
     */
    @Column(name = "OrgName", nullable = false)
    private String orgName;
    /**
     * 项目负责人名，使用 {@link com.ctrip.framework.apollo.portal.entity.po.UserPO#username}
     */
    @Column(name = "OwnerName", nullable = false)
    private String ownerName;
    /**
     * 项目负责人邮箱，使用 {@link com.ctrip.framework.apollo.portal.entity.po.UserPO#email}
     */
    @Column(name = "OwnerEmail", nullable = false)
    private String ownerEmail;
}
```

* 字段比较简单，胖友自己看注释。
* 例子如下图：![例子](http://www.iocoder.cn/images/Apollo/2018_06_10/02.png)

## 3.2 ConsumerToken

**ConsumerToken** 表，对应实体 `com.ctrip.framework.apollo.openapi.entity.ConsumerToken` ，代码如下：

```Java
@Entity
@Table(name = "ConsumerToken")
@SQLDelete(sql = "Update ConsumerToken set isDeleted = 1 where id = ?")
@Where(clause = "isDeleted = 0")
public class ConsumerToken extends BaseEntity {

    /**
     * 第三方应用编号，使用 {@link Consumer#id}
     */
    @Column(name = "ConsumerId", nullable = false)
    private long consumerId;
    /**
     * Token
     */
    @Column(name = "token", nullable = false)
    private String token;
    /**
     * 过期时间
     */
    @Column(name = "Expires", nullable = false)
    private Date expires;
}
```

* `consumerId` 字段，第三方应用编号，指向对应的 Consumer 记录。ConsumerToken 和 Consumer 是**多对一**的关系。
* `token` 字段，Token 。
    * 调用 OpenAPI 时，放在请求 Header `"Authorization"` 中，作为身份标识。
    * 通过 `ConsumerService#generateToken(consumerAppId, generationTime, consumerTokenSalt)` 方法生成，代码如下：

        ```Java
        String generateToken(String consumerAppId, Date generationTime, String consumerTokenSalt) {
            return Hashing.sha1().hashString(KEY_JOINER.join(consumerAppId, TIMESTAMP_FORMAT.format(generationTime), consumerTokenSalt), Charsets.UTF_8).toString();
        }
        ```
        * x

* `expires` 字段，过期时间。
* 例子如下图：![例子](http://www.iocoder.cn/images/Apollo/2018_06_10/03.png)

## 3.3 ConsumerAudit

**ConsumerAudit** 表，对应实体 `com.ctrip.framework.apollo.openapi.entity.ConsumerAudit` ，代码如下：

> ConsumerAudit 和 Audit 功能类似，我们在 [《Apollo 源码解析 —— Config Service 操作审计日志 Audit》](http://www.iocoder.cn/Apollo/config-service-audit/?self) 中已经分享。

```Java
@Entity
@Table(name = "ConsumerAudit")
public class ConsumerAudit {

    /**
     * 日志编号，自增
     */
    @Id
    @GeneratedValue
    @Column(name = "Id")
    private long id;
    /**
     * 第三方应用编号，使用 {@link Consumer#id}
     */
    @Column(name = "ConsumerId", nullable = false)
    private long consumerId;
    /**
     * 请求 URI
     */
    @Column(name = "Uri", nullable = false)
    private String uri;
    /**
     * 请求 Method
     */
    @Column(name = "Method", nullable = false)
    private String method;
    /**
     * 数据创建时间
     */
    @Column(name = "DataChange_CreatedTime")
    private Date dataChangeCreatedTime;
    /**
     * 数据最后更新时间
     */
    @Column(name = "DataChange_LastTime")
    private Date dataChangeLastModifiedTime;

    @PrePersist
    protected void prePersist() {
        if (this.dataChangeCreatedTime == null) {
            this.dataChangeCreatedTime = new Date();
        }
        if (this.dataChangeLastModifiedTime == null) {
            dataChangeLastModifiedTime = this.dataChangeCreatedTime;
        }
    }
}
```

* 字段比较简单，胖友自己看注释。
* 如果胖友希望更加详细，可以添加如下字段：
    * `token` 字段，请求时的 Token 。
    * `params` 字段，请求参数。
    * `responseStatus` 字段， 响应状态码。
    * `ip` 字段，请求 IP 。
    * `ua` 字段，请求 User-Agent 。

# 4. ConsumerAuthenticationFilter

`com.ctrip.framework.apollo.openapi.filter.ConsumerAuthenticationFilter` ，实现 Filter 接口，OpenAPI **认证**( Authentication )过滤器。代码如下：

```Java
  1: public class ConsumerAuthenticationFilter implements Filter {
  2: 
  3:     private ConsumerAuthUtil consumerAuthUtil;
  4:     private ConsumerAuditUtil consumerAuditUtil;
  5: 
  6:     public ConsumerAuthenticationFilter(ConsumerAuthUtil consumerAuthUtil, ConsumerAuditUtil consumerAuditUtil) {
  7:         this.consumerAuthUtil = consumerAuthUtil;
  8:         this.consumerAuditUtil = consumerAuditUtil;
  9:     }
 10: 
 11:     @Override
 12:     public void init(FilterConfig filterConfig) throws ServletException {
 13:         // nothing
 14:     }
 15: 
 16:     @Override
 17:     public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain) throws IOException, ServletException {
 18:         HttpServletRequest request = (HttpServletRequest) req;
 19:         HttpServletResponse response = (HttpServletResponse) resp;
 20: 
 21:         // 从请求 Header 中，获得 token
 22:         String token = request.getHeader("Authorization");
 23:         // 获得 Consumer 编号
 24:         Long consumerId = consumerAuthUtil.getConsumerId(token);
 25:         // 若不存在，返回错误状态码 401
 26:         if (consumerId == null) {
 27:             response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Unauthorized");
 28:             return;
 29:         }
 30:         // 存储 Consumer 编号到请求中
 31:         consumerAuthUtil.storeConsumerId(request, consumerId);
 32:         // 记录 ConsumerAudit 记录
 33:         consumerAuditUtil.audit(request, consumerId);
 34: 
 35:         // 继续过滤器
 36:         chain.doFilter(req, resp);
 37:     }
 38: 
 39:     @Override
 40:     public void destroy() {
 41:         // nothing
 42:     }
 43: 
 44: }
```

* ConsumerToken 相关
    * 第 22 行：从**请求 Header** `"Authorization"` 中，获得作为身份标识的 Token 。
    * 第 24 行：调用 `ConsumerAuthUtil#getConsumerId(token)` 方法，获得 Token 对应的 **Consumer 编号**。详细解析，在 [「5.1 ConsumerAuthUtil」](#) 中。
    * 第 25 至 29 行：若 Consumer 不存在时，返回错误状态码 **401** 。
    * 第 31 行：调用 `ConsumerAuthUtil#storeConsumerId(request, consumerId)` 方法，存储 **Consumer 编号**到 Request 中。
* ConsumerAudit 相关
    * 第 33 行：调用 `ConsumerAuditUtil#audit(request, consumerId)`  方法，记录 ConsumerAudit 记录。详细解析，在 [「5.2 ConsumerAuditUtil」](#) 中。

## 4.1 AuthFilterConfiguration

`com.ctrip.framework.apollo.portal.spi.configuration.AuthFilterConfiguration` ，**AuthFilterConfigurationFilter** Spring Java 配置。代码如下：

```Java
@Configuration
public class AuthFilterConfiguration {

    @Bean
    public FilterRegistrationBean openApiAuthenticationFilter(ConsumerAuthUtil consumerAuthUtil, ConsumerAuditUtil consumerAuditUtil) {
        FilterRegistrationBean openApiFilter = new FilterRegistrationBean();

        openApiFilter.setFilter(new ConsumerAuthenticationFilter(consumerAuthUtil, consumerAuditUtil));
        openApiFilter.addUrlPatterns("/openapi/*"); // 匹配 `"/openapi/*"` 路径

        return openApiFilter;
    }

}
```

* 匹配 `"/openapi/*"` 路径。

# 5. Util

## 5.1 ConsumerAuthUtil

`com.ctrip.framework.apollo.openapi.util.ConsumerAuthUtil` ，Consumer 认证工具类。代码如下：

```Java
@Service
public class ConsumerAuthUtil {

    /**
     * Request Attribute —— Consumer 编号
     */
    static final String CONSUMER_ID = "ApolloConsumerId";

    @Autowired
    private ConsumerService consumerService;

    /**
     * 获得 Token 获得对应的 Consumer 编号
     *
     * @param token Token
     * @return Consumer 编号
     */
    public Long getConsumerId(String token) {
        return consumerService.getConsumerIdByToken(token);
    }

    /**
     * 设置 Consumer 编号到 Request
     *
     * @param request 请求
     * @param consumerId Consumer 编号
     */
    public void storeConsumerId(HttpServletRequest request, Long consumerId) {
        request.setAttribute(CONSUMER_ID, consumerId);
    }

    /**
     * 获得 Consumer 编号从 Request
     *
     * @param request 请求
     * @return Consumer 编号
     */
    public long retrieveConsumerId(HttpServletRequest request) {
        Object value = request.getAttribute(CONSUMER_ID);
        try {
            return Long.parseLong(value.toString());
        } catch (Throwable ex) {
            throw new IllegalStateException("No consumer id!", ex);
        }
    }

}
```

* 代码比较简单，胖友自己阅读理解。

## 5.2 ConsumerAuditUtil

`com.ctrip.framework.apollo.openapi.util.ConsumerAuditUtill` ，实现 InitializingBean 接口，ConsumerAudit 工具类。代码如下：

```Java
@Service
public class ConsumerAuditUtil implements InitializingBean {

    private static final int CONSUMER_AUDIT_MAX_SIZE = 10000;
    /**
     * 队列
     */
    private BlockingQueue<ConsumerAudit> audits = Queues.newLinkedBlockingQueue(CONSUMER_AUDIT_MAX_SIZE);
    /**
     * ExecutorService 对象
     */
    private final ExecutorService auditExecutorService;
    /**
     * 是否停止
     */
    private final AtomicBoolean auditStopped;
    /**
     * 批任务 ConsumerAudit 数量
     */
    private int BATCH_SIZE = 100;
    /**
     * 批任务 ConsumerAudit 等待超时时间
     */
    private long BATCH_TIMEOUT = 5;
    /**
     * {@link #BATCH_TIMEOUT} 单位
     */
    private TimeUnit BATCH_TIMEUNIT = TimeUnit.SECONDS;

    @Autowired
    private ConsumerService consumerService;

    public ConsumerAuditUtil() {
        auditExecutorService = Executors.newSingleThreadExecutor(ApolloThreadFactory.create("ConsumerAuditUtil", true));
        auditStopped = new AtomicBoolean(false);
    }

    public boolean audit(HttpServletRequest request, long consumerId) {
        // ignore GET request
        // 忽略 GET 请求
        if ("GET".equalsIgnoreCase(request.getMethod())) {
            return true;
        }
        // 组装 URI
        String uri = request.getRequestURI();
        if (!Strings.isNullOrEmpty(request.getQueryString())) {
            uri += "?" + request.getQueryString();
        }

        // 创建 ConsumerAudit 对象
        ConsumerAudit consumerAudit = new ConsumerAudit();
        Date now = new Date();
        consumerAudit.setConsumerId(consumerId);
        consumerAudit.setUri(uri);
        consumerAudit.setMethod(request.getMethod());
        consumerAudit.setDataChangeCreatedTime(now);
        consumerAudit.setDataChangeLastModifiedTime(now);

        // throw away audits if exceeds the max size
        // 添加到队列
        return this.audits.offer(consumerAudit);
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        auditExecutorService.submit(() -> {
            // 循环【批任务】，直到停止
            while (!auditStopped.get() && !Thread.currentThread().isInterrupted()) {
                List<ConsumerAudit> toAudit = Lists.newArrayList();
                try {
                    // 获得 ConsumerAudit 批任务，直到到达上限，或者超时
                    Queues.drain(audits, toAudit, BATCH_SIZE, BATCH_TIMEOUT, BATCH_TIMEUNIT);
                    // 批量保存到数据库
                    if (!toAudit.isEmpty()) {
                        consumerService.createConsumerAudits(toAudit);
                    }
                } catch (Throwable ex) {
                    Tracer.logError(ex);
                }
            }
        });
    }

    public void stopAudit() {
        auditStopped.set(true);
    }

}
```

* `#audit(request, consumerId)` 方法，创建 ConsumerAudit 对象，添加到**队列** `audits` 中。
* `#afterPropertiesSet()` 方法，初始化**后台**任务。该任务，调用 `Queues#drain(BlockingQueue, buffer, numElements, timeout, TimeUnit)` 方法，获得 ConsumerAudit 批任务，直到**到达上限**( `BATCH_SIZE` )，或者**超时**( `BATCH_TIMEOUT` ) 。若获得到任务，调用 `ConsumerService@createConsumerAudit(Iterable<ConsumerAudit>)` 方法，**批量**保存到数据库中。
    * Google Guava **Queues** ，感兴趣的胖友，可以自己去研究下。
    * Eureka Server 集群同步实例，也有相同处理。 

# 6. ConsumerService

`com.ctrip.framework.apollo.openapi.service.ConsumerService` ，提供 Consumer、ConsumerToken、ConsumerAudit、ConsumerRole 相关的 **Service** 逻辑。

## 6.1 构造方法

```Java
private static final FastDateFormat TIMESTAMP_FORMAT = FastDateFormat.getInstance("yyyyMMddHHmmss");
private static final Joiner KEY_JOINER = Joiner.on("|");

@Autowired
private UserInfoHolder userInfoHolder;
@Autowired
private ConsumerTokenRepository consumerTokenRepository;
@Autowired
private ConsumerRepository consumerRepository;
@Autowired
private ConsumerAuditRepository consumerAuditRepository;
@Autowired
private ConsumerRoleRepository consumerRoleRepository;
@Autowired
private PortalConfig portalConfig;
@Autowired
private RolePermissionService rolePermissionService;
@Autowired
private UserService userService;
```

## 6.2 createConsumer

`#createConsumer(Consumer)` 方法，保存 Consumer 到数据库中。代码如下：

```Java
public Consumer createConsumer(Consumer consumer) {
    String appId = consumer.getAppId();

    // 校验 appId 对应的 Consumer 不存在
    Consumer managedConsumer = consumerRepository.findByAppId(appId);
    if (managedConsumer != null) {
        throw new BadRequestException("Consumer already exist");
    }

    // 校验 ownerName 对应的 UserInfo 存在
    String ownerName = consumer.getOwnerName();
    UserInfo owner = userService.findByUserId(ownerName);
    if (owner == null) {
        throw new BadRequestException(String.format("User does not exist. UserId = %s", ownerName));
    }
    consumer.setOwnerEmail(owner.getEmail());

    // 设置 Consumer 的创建和最后修改人为当前管理员
    String operator = userInfoHolder.getUser().getUserId();
    consumer.setDataChangeCreatedBy(operator);
    consumer.setDataChangeLastModifiedBy(operator);

    // 保存 Consumer 到数据库中
    return consumerRepository.save(consumer);
}
```

## 6.3 generateAndSaveConsumerToken

`#generateAndSaveConsumerToken(Consumer, expires)` 方法，基于 Consumer 对象，创建其对应的 ConsumerToken 对象，并保存到数据库中。代码如下：

```Java
public ConsumerToken generateAndSaveConsumerToken(Consumer consumer, Date expires) {
    Preconditions.checkArgument(consumer != null, "Consumer can not be null");

    // 生成 ConsumerToken 对象
    ConsumerToken consumerToken = generateConsumerToken(consumer, expires);
    consumerToken.setId(0); //for protection

    // 保存 ConsumerToken 到数据库中
    return consumerTokenRepository.save(consumerToken);
}
```

* 调用 `#generateConsumerToken(Consumer, expires)` 方法，基于 Consumer 对象，创建其对应的 ConsumerToken 对象。代码如下：

    ```Java
    private ConsumerToken generateConsumerToken(Consumer consumer, Date expires) {
        long consumerId = consumer.getId();
        String createdBy = userInfoHolder.getUser().getUserId();
        Date createdTime = new Date();
    
        // 创建 ConsumerToken
        ConsumerToken consumerToken = new ConsumerToken();
        consumerToken.setConsumerId(consumerId);
        consumerToken.setExpires(expires);
        consumerToken.setDataChangeCreatedBy(createdBy);
        consumerToken.setDataChangeCreatedTime(createdTime);
        consumerToken.setDataChangeLastModifiedBy(createdBy);
        consumerToken.setDataChangeLastModifiedTime(createdTime);
    
        // 生成 ConsumerToken 的 `token`
        generateAndEnrichToken(consumer, consumerToken);
    
        return consumerToken;
    }
    ```

* 调用 `#generateAndEnrichToken(Consumer, ConsumerToken)` 方法，生成 ConsumerToken 的 `token` 。代码如下：

    ```Java
    void generateAndEnrichToken(Consumer consumer, ConsumerToken consumerToken) {
        Preconditions.checkArgument(consumer != null);
    
        // 设置创建时间
        if (consumerToken.getDataChangeCreatedTime() == null) {
            consumerToken.setDataChangeCreatedTime(new Date());
        }
        // 生成 ConsumerToken 的 `token`
        consumerToken.setToken(generateToken(consumer.getAppId(), consumerToken.getDataChangeCreatedTime(), portalConfig.consumerTokenSalt()));
    }
    ```

## 6.4 其他方法

在 ConsumerService 中，还有授权相关的方法，在下一篇文章分享。

* `#assignNamespaceRoleToConsumer(token, appId, namespaceName)` 方法
* `#assignAppRoleToConsumer(token, appId)` 方法

# 7. ConsumerController

在 `apollo-portal` 项目中，`com.ctrip.framework.apollo.portal.controller.ConsumerController` ，提供 Consumer、ConsumerToken、ConsumerAudit 相关的 **API** 。

在**创建第三方应用**的界面中，点击【创建】按钮，调用**创建 Consumer 的 API** 。

![创建第三方应用](http://www.iocoder.cn/images/Apollo/2018_06_10/04.png)

代码如下：

```Java
@Transactional
@PreAuthorize(value = "@permissionValidator.isSuperAdmin()")
@RequestMapping(value = "/consumers", method = RequestMethod.POST)
public ConsumerToken createConsumer(@RequestBody Consumer consumer,
                                    @RequestParam(value = "expires", required = false)
                                    @DateTimeFormat(pattern = "yyyyMMddHHmmss") Date
                                            expires) {
    // 校验非空
    if (StringUtils.isContainEmpty(consumer.getAppId(), consumer.getName(), consumer.getOwnerName(), consumer.getOrgId())) {
        throw new BadRequestException("Params(appId、name、ownerName、orgId) can not be empty.");
    }

    // 创建 Consumer 对象，并保存到数据库中
    Consumer createdConsumer = consumerService.createConsumer(consumer);

    // 创建 ConsumerToken 对象，并保存到数据库中
    if (Objects.isNull(expires)) {
        expires = DEFAULT_EXPIRES;
    }
    return consumerService.generateAndSaveConsumerToken(createdConsumer, expires);
}
```

* **POST `/consumers` 接口**，Request Body 传递 **JSON** 对象。
* `@PreAuthorize(...)` 注解，调用 `PermissionValidator#isSuperAdmin(a)` 方法，校验是否超级管理员。
* 调用 ConsumerService ，创建 **Consumer** 和 **ConsumerToken** 对象，并保存到数据库中。

# 666. 彩蛋

😈 小文一篇，周日 00:00 点啦。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)


