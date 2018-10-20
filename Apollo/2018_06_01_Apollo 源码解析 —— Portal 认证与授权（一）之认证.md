title: Apollo 源码解析 —— Portal 认证与授权（一）之认证
date: 2018-06-01
tags:
categories: Apollo
permalink: Apollo/portal-auth-1

-------

摘要: 原创出处 http://www.iocoder.cn/Apollo/portal-auth-1/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Apollo/portal-auth-1/)
- [2. AuthConfiguration](http://www.iocoder.cn/Apollo/portal-auth-1/)
  - [2.1 SpringSecurityAuthAutoConfiguration](http://www.iocoder.cn/Apollo/portal-auth-1/)
  - [2.2 SpringSecurityConfigureration](http://www.iocoder.cn/Apollo/portal-auth-1/)
- [3. Users](http://www.iocoder.cn/Apollo/portal-auth-1/)
  - [3.1 UserInfo](http://www.iocoder.cn/Apollo/portal-auth-1/)
- [4. Authorities](http://www.iocoder.cn/Apollo/portal-auth-1/)
- [5. UserService](http://www.iocoder.cn/Apollo/portal-auth-1/)
  - [5.1 SpringSecurityUserService](http://www.iocoder.cn/Apollo/portal-auth-1/)
  - [5.2 UserInfoController](http://www.iocoder.cn/Apollo/portal-auth-1/)
- [6. UserInfoHolder](http://www.iocoder.cn/Apollo/portal-auth-1/)
  - [6.1 SpringSecurityUserInfoHolder](http://www.iocoder.cn/Apollo/portal-auth-1/)
- [7. SsoHeartbeatHandler](http://www.iocoder.cn/Apollo/portal-auth-1/)
  - [7.1 DefaultSsoHeartbeatHandler](http://www.iocoder.cn/Apollo/portal-auth-1/)
  - [7.2 SsoHeartbeatController](http://www.iocoder.cn/Apollo/portal-auth-1/)
- [8. LogoutHandler](http://www.iocoder.cn/Apollo/portal-auth-1/)
  - [8.1 DefaultLogoutHandler](http://www.iocoder.cn/Apollo/portal-auth-1/)
- [666. 彩蛋](http://www.iocoder.cn/Apollo/portal-auth-1/)

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

> 老艿艿：本系列假定胖友已经阅读过 [《Apollo 官方 wiki 文档》](https://github.com/ctripcorp/apollo/wiki/) ，特别是 [《Portal 实现用户登录功能》](https://github.com/ctripcorp/apollo/wiki/Portal-%E5%AE%9E%E7%8E%B0%E7%94%A8%E6%88%B7%E7%99%BB%E5%BD%95%E5%8A%9F%E8%83%BD) 。

本文分享 Portal 的认证与授权，**侧重在认证部分**。

在 [《Portal 实现用户登录功能》](https://github.com/ctripcorp/apollo/wiki/Portal-%E5%AE%9E%E7%8E%B0%E7%94%A8%E6%88%B7%E7%99%BB%E5%BD%95%E5%8A%9F%E8%83%BD) 文档的开头：

> Apollo 是配置管理系统，会提供权限管理（Authorization），理论上是不负责用户登录认证功能的实现（Authentication）。
> 
> 所以 Apollo 定义了一些SPI用来解耦，Apollo 接入登录的关键就是实现这些 SPI 。

和我们理解的 JDK SPI 不同，Apollo 是基于 Spring [Profile](https://docs.spring.io/autorepo/docs/spring-boot/current/reference/html/boot-features-profiles.html) 的特性，配合上 Spring Java Configuration 实现了**类似** SPI 的功能。对于大多数人，我们可能比较熟悉的是，基于不同的 Profile 加载不同**环境**的 `yaml` 或 `properties` 配置文件。所以，当笔者看到这样的玩法，也是眼前一亮。

在 `apollo-portal` 项目中，`spi` 包下，我们可以看到**认证**相关的**配置**与**实现**，如下图所示：![代码结构](http://www.iocoder.cn/images/Apollo/2018_06_01/01.png)

* 绿框：接口。
* 紫框：实现。
* 红框：配置接口对应的实现。

# 2. AuthConfiguration

`com.ctrip.framework.apollo.portal.spi.configuration.AuthConfiguration` ，**认证** Spring Java 配置。如下图：![AuthConfiguration](http://www.iocoder.cn/images/Apollo/2018_06_01/02.png)

目前有三种实现：

* 第一种， `profile=ctrip` ，携程**内部**实现，接入了SSO并实现用户搜索、查询接口。
* 第二种，`profile=auth` ，使用 Apollo 提供的 **Spring Security** 简单认证。
* 第三种，`profile` 为空，使用**默认**实现，全局只有 apollo 一个账号。

一般情况下，我们使用**第二种**，基于 **Spring Security** 的实现。所以本文仅分享这种方式。对其他方式感兴趣的胖友，可以自己读下代码哈。

整体类图如下：![类图](http://www.iocoder.cn/images/Apollo/2018_06_01/05.png)

## 2.1 SpringSecurityAuthAutoConfiguration

**UserService** ，配置如下：

```Java
@Bean
@ConditionalOnMissingBean(UserService.class)
public UserService springSecurityUserService() {
    return new SpringSecurityUserService();
}
```

* 使用 SpringSecurityUserService 实现类，在 [「5. UserService」](#) 中，详细解析。

-------

**UserInfoHolder** ，配置如下：

```Java
@Bean
@ConditionalOnMissingBean(UserInfoHolder.class)
public UserInfoHolder springSecurityUserInfoHolder() {
    return new SpringSecurityUserInfoHolder();
}
```

* 使用 SpringSecurityUserInfoHolder 实现类，在 [「6. UserInfoHolder」](#) 中，详细解析。

-------

**JdbcUserDetailsManager** ，配置如下：

```Java
@Bean
public JdbcUserDetailsManager jdbcUserDetailsManager(AuthenticationManagerBuilder auth, DataSource datasource) throws Exception {
    JdbcUserDetailsManager jdbcUserDetailsManager = auth.jdbcAuthentication() // 基于 JDBC
            .passwordEncoder(new BCryptPasswordEncoder()) // 加密方式为 BCryptPasswordEncoder
            .dataSource(datasource) // 数据源
            .usersByUsernameQuery("select Username,Password,Enabled from `Users` where Username = ?") // 使用 Username 查询 User
            .authoritiesByUsernameQuery("select Username,Authority from `Authorities` where Username = ?") // 使用 Username 查询 Authorities
            .getUserDetailsService();

    jdbcUserDetailsManager.setUserExistsSql("select Username from `Users` where Username = ?"); // 判断 User 是否存在
    jdbcUserDetailsManager.setCreateUserSql("insert into `Users` (Username, Password, Enabled) values (?,?,?)"); // 插入 User
    jdbcUserDetailsManager.setUpdateUserSql("update `Users` set Password = ?, Enabled = ? where Username = ?"); // 更新 User
    jdbcUserDetailsManager.setDeleteUserSql("delete from `Users` where Username = ?"); // 删除 User
    jdbcUserDetailsManager.setCreateAuthoritySql("insert into `Authorities` (Username, Authority) values (?,?)"); // 插入 Authorities
    jdbcUserDetailsManager.setDeleteUserAuthoritiesSql("delete from `Authorities` where Username = ?"); // 删除 Authorities
    jdbcUserDetailsManager.setChangePasswordSql("update `Users` set Password = ? where Username = ?"); // 更新 Authorities

    return jdbcUserDetailsManager;
}
```

* `org.springframework.security.provisioning.JdbcUserDetailsManager` ，继承 JdbcDaoImpl 的功能，提供了一些很有用的与 **Users 和 Authorities 表**相关的方法。
* 胖友先看下 [「3. Users」](#) 和 [「4. Authorities」](#) 小节，然后回过头继续往下看。

-------

**SsoHeartbeatHandler** ，配置如下：

```Java
@Bean
@ConditionalOnMissingBean(SsoHeartbeatHandler.class)
public SsoHeartbeatHandler defaultSsoHeartbeatHandler() {
    return new DefaultSsoHeartbeatHandler();
}
```

* 使用 DefaultSsoHeartbeatHandler 实现类，在 [「7. SsoHeartbeatHandler」](#) 中，详细解析。

-------

**LogoutHandler** ，配置如下：

```Java
@Bean
@ConditionalOnMissingBean(LogoutHandler.class)
public LogoutHandler logoutHandler() {
    return new DefaultLogoutHandler();
}
```

* 使用 DefaultLogoutHandler 实现类，在 [「8. LogoutHandler」](#) 中，详细解析。

## 2.2 SpringSecurityConfigureration

```Java
@Order(99)
@Profile("auth")
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
static class SpringSecurityConfigurer extends WebSecurityConfigurerAdapter {

    public static final String USER_ROLE = "user";

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable(); // 关闭打开的 csrf 保护
        http.headers().frameOptions().sameOrigin(); // 仅允许相同 origin 访问
        http.authorizeRequests()
                .antMatchers("/openapi/**", "/vendor/**", "/styles/**", "/scripts/**", "/views/**", "/img/**").permitAll() // openapi 和 资源不校验权限
                .antMatchers("/**").hasAnyRole(USER_ROLE); // 其他，需要登录 User
        http.formLogin().loginPage("/signin").permitAll().failureUrl("/signin?#/error").and().httpBasic(); // 登录页
        http.logout().invalidateHttpSession(true).clearAuthentication(true).logoutSuccessUrl("/signin?#/logout"); // 登出（退出）
        http.exceptionHandling().authenticationEntryPoint(new LoginUrlAuthenticationEntryPoint("/signin")); // 未身份校验，跳转到登录页
    }

}
```

* `@EnableWebSecurity` 注解，禁用 Boot 的默认 Security 配置，配合 `@Configuration` 启用自定义配置（需要继承 WebSecurityConfigurerAdapter ）。
* `@EnableGlobalMethodSecurity(prePostEnabled = true)` 注解，启用 Security 注解，例如最常用的 `@PreAuthorize` 。
* **注意**，`.antMatchers("/**").hasAnyRole(USER_ROLE);` 代码块，设置**统一**的 URL 的权限校验，**只判断是否为登陆用户**。另外，`#hasAnyRole(...)` 方法，会自动添加 `"ROLE_"` 前缀，所以此处的传参是 `"user"` 。代码如下：

    ```Java
    // ExpressionUrlAuthorizationConfigurer.java
    
    private static String hasAnyRole(String... authorities) {
    	String anyAuthorities = StringUtils.arrayToDelimitedString(authorities,
    			"','ROLE_");
    	return "hasAnyRole('ROLE_" + anyAuthorities + "')";
    }
    ```

# 3. Users

**Users** 表，对应实体 `com.ctrip.framework.apollo.portal.entity.po.UserPO` ，代码如下：

```Java
@Entity
@Table(name = "Users")
public class UserPO {

    /**
     * 编号
     */
    @Id
    @GeneratedValue
    @Column(name = "Id")
    private long id;
    /**
     * 账号
     */
    @Column(name = "Username", nullable = false)
    private String username;
    /**
     * 密码
     */
    @Column(name = "Password", nullable = false)
    private String password;
    /**
     * 邮箱
     */
    @Column(name = "Email", nullable = false)
    private String email;
    /**
     * 是否开启
     */
    @Column(name = "Enabled", nullable = false)
    private int enabled;
    
}
```

* 字段比较简单，胖友自己看注释。

## 3.1 UserInfo

`com.ctrip.framework.apollo.portal.entity.bo.UserInfo` ，User **BO** 。代码如下：

```Java
public class UserInfo {

    /**
     * 账号 {@link com.ctrip.framework.apollo.portal.entity.po.UserPO#username}
     */
    private String userId;
    /**
     * 账号 {@link com.ctrip.framework.apollo.portal.entity.po.UserPO#username}
     */
    private String name;
    /**
     * 邮箱 {@link com.ctrip.framework.apollo.portal.entity.po.UserPO#email}
     */
    private String email;
    
}
```

* 在 UserPO 的 `#toUserInfo()` 方法中，将 UserPO 转换成 UserBO ，代码如下：

    ```Java
    public UserInfo toUserInfo() {
        UserInfo userInfo = new UserInfo();
        userInfo.setName(this.getUsername());
        userInfo.setUserId(this.getUsername());
        userInfo.setEmail(this.getEmail());
        return userInfo;
    }
    ```
    * **注意**，`userId` 和 `name` 属性，都是指向 `User.username` 。

# 4. Authorities

**Authorities** 表，Spring Security 中的 Authority ，实际和 Role 角色**等价**。表结构如下：

```SQL
`Id` int(11) UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '自增Id',
`Username` varchar(50) NOT NULL,
`Authority` varchar(50) NOT NULL,
```

* 目前 Portal 只有**一种**角色 `"ROLE_user"` 。如下图所示：![Authorities](http://www.iocoder.cn/images/Apollo/2018_06_01/03.png)
* 为什么是这样的呢？在 Apollo 中，
    * **统一**的 URL 的权限校验，**只判断是否为登陆用户**，在 SpringSecurityConfigureration 中，我们可以看到。
    * 具体**每个** URL 的权限校验，通过在对应的方法上，添加 `@PreAuthorize` 方法注解，配合具体的方法参数，一起校验**功能 + 数据级**的权限校验。

# 5. UserService

`com.ctrip.framework.apollo.portal.spi.UserService` ，User 服务**接口**，用来给 Portal 提供用户搜索相关功能。代码如下：

```Java
public interface UserService {

    List<UserInfo> searchUsers(String keyword, int offset, int limit);

    UserInfo findByUserId(String userId);

    List<UserInfo> findByUserIds(List<String> userIds);

}
```

## 5.1 SpringSecurityUserService

`com.ctrip.framework.apollo.portal.spi.springsecurity.SpringSecurityUserService` ，基于 **Spring Security** 的 UserService 实现类。

### 5.5.1 构造方法

```Java
private PasswordEncoder encoder = new BCryptPasswordEncoder();
/**
 * 默认角色数组，详细见 {@link #init()}
 */
private List<GrantedAuthority> authorities;

@Autowired
private JdbcUserDetailsManager userDetailsManager;
@Autowired
private UserRepository userRepository;

@PostConstruct
public void init() {
    authorities = new ArrayList<>();
    authorities.add(new SimpleGrantedAuthority("ROLE_user"));
}
```

* `authorities` 属性，只有一个元素，为 `"ROLE_user"` 。

### 5.5.2 createOrUpdate

`#createOrUpdate(UserPO)` 方法，创建或更新 User 。代码如下：

```Java
  1: @Transactional
  2: public void createOrUpdate(UserPO user) {
  3:     String username = user.getUsername();
  4:     // 创建 Spring Security User
  5:     User userDetails = new User(username, encoder.encode(user.getPassword()), authorities);
  6:     // 若存在，则进行更新
  7:     if (userDetailsManager.userExists(username)) {
  8:         userDetailsManager.updateUser(userDetails);
  9:     // 若不存在，则进行新增
 10:     } else {
 11:         userDetailsManager.createUser(userDetails);
 12:     }
 13:     // 更新邮箱
 14:     UserPO managedUser = userRepository.findByUsername(username);
 15:     managedUser.setEmail(user.getEmail());
 16:     userRepository.save(managedUser);
 17: }
```

* 第 5 行：创建 `com.ctrip.framework.apollo.portal.spi.springsecurity.User` 对象。
    * 使用  PasswordEncoder 对 `password` 加密。
    * 传入对应的角色 `authorities` 参数。
* 第 6 至 12 行：新增或更新 User 。
* 第 13 至 16 行：更新 `email` 。不直接在【第 6 至 12 行】处理的原因是，`com.ctrip.framework.apollo.portal.spi.springsecurity.User` 中没有 `email` 属性。

### 5.5.3 其他实现方法

🙂 胖友自己查看代码。嘿嘿。

## 5.2 UserInfoController

在 `apollo-portal` 项目中，`com.ctrip.framework.apollo.portal.controller.UserInfoController` ，提供 User 的 **API** 。

### 5.2.1 createOrUpdateUser

在**用户管理**的界面中，点击【提交】按钮，调用**创建或更新 User 的 API** 。

![创建或更新 User 界面](http://www.iocoder.cn/images/Apollo/2018_06_01/04.png)

`#createOrUpdateUser(UserPO)` 方法，创建或更新 User 。代码如下：

```Java
@Autowired
private UserService userService;

@PreAuthorize(value = "@permissionValidator.isSuperAdmin()")
@RequestMapping(value = "/users", method = RequestMethod.POST)
public void createOrUpdateUser(@RequestBody UserPO user) {
    // 校验 `username` `password` 非空
    if (StringUtils.isContainEmpty(user.getUsername(), user.getPassword())) {
        throw new BadRequestException("Username and password can not be empty.");
    }
    // 新增或更新 User
    if (userService instanceof SpringSecurityUserService) {
        ((SpringSecurityUserService) userService).createOrUpdate(user);
    } else {
        throw new UnsupportedOperationException("Create or update user operation is unsupported");
    }
}
```

* **POST `/users` 接口**，Request Body 传递 **JSON** 对象。
* `@PreAuthorize(...)` 注解，调用 `PermissionValidator#isSuperAdmin()` 方法，校验是否为**超级管理员**。后续文章，详细分享。
* 调用 `SpringSecurityUserService#createOrUpdate(UserPO)` 方法，新增或更新 User 。

### 5.2.2 logout

`#logout(request, response)` 方法，User 登出。代码如下：

```Java
@Autowired
private LogoutHandler logoutHandler;
    
@RequestMapping(value = "/user/logout", method = RequestMethod.GET)
public void logout(HttpServletRequest request, HttpServletResponse response) throws IOException {
    logoutHandler.logout(request, response);
}
```

* * **GET `/user/logout` 接口**。
* 调用 `LogoutHandler#logout(request, response)` 方法，登出 User 。在 [「8. LogoutHandler」](#) 中，详细解析。

# 6. UserInfoHolder

`com.ctrip.framework.apollo.portal.spi.UserInfoHolder` ，获取当前登录用户信息，**SSO** 一般都是把当前登录用户信息放在线程 ThreadLocal 上。代码如下：

```Java
public interface UserInfoHolder {

    UserInfo getUser();

}
```

## 6.1 SpringSecurityUserInfoHolder

`com.ctrip.framework.apollo.portal.spi.springsecurity.SpringSecurityUserInfoHolder` ，实现 UserInfoHolder 接口，基于 **Spring Security** 的 UserInfoHolder 实现类。代码如下：

```Java
public class SpringSecurityUserInfoHolder implements UserInfoHolder {

    @Override
    public UserInfo getUser() {
        // 创建 UserInfo 对象，设置 `username` 到 `UserInfo.userId` 中。
        UserInfo userInfo = new UserInfo();
        userInfo.setUserId(getCurrentUsername());
        return userInfo;
    }

    /**
     * @return username
     */
    private String getCurrentUsername() {
        Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();
        if (principal instanceof UserDetails) {
            return ((UserDetails) principal).getUsername();
        }
        if (principal instanceof Principal) {
            return ((Principal) principal).getName();
        }
        return String.valueOf(principal);
    }

}
```

# 7. SsoHeartbeatHandler

`com.ctrip.framework.apollo.portal.spi.SsoHeartbeatHandler` ，Portal 页面如果长时间不刷新，登录信息会过期。通过此接口来刷新登录信息。代码如下：

```Java
public interface SsoHeartbeatHandler {

    void doHeartbeat(HttpServletRequest request, HttpServletResponse response);

}
```

## 7.1 DefaultSsoHeartbeatHandler

`com.ctrip.framework.apollo.portal.spi.defaultimpl.DefaultSsoHeartbeatHandler` ，实现 SsoHeartbeatHandler 接口，代码如下：

```Java
public class DefaultSsoHeartbeatHandler implements SsoHeartbeatHandler {

    @Override
    public void doHeartbeat(HttpServletRequest request, HttpServletResponse response) {
        try {
            response.sendRedirect("default_sso_heartbeat.html");
        } catch (IOException e) {
        }
    }

}
```

* 跳转到 `default_sso_heartbeat.html` 中。页面如下：

    ```HTML
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <title>SSO Heartbeat</title>
        <script type="text/javascript">
            var reloading = false;
            setInterval(function () {
                if (reloading) {
                    return;
                }
                reloading = true;
                location.reload(true);
            }, 60000);
        </script>
    </head>
    <body>
    </body>
    </html>
    ```
    * 每 60 秒刷新一次页面。🙂 一脸懵逼，这是干啥的？继续往下看。

## 7.2 SsoHeartbeatController

`com.ctrip.framework.apollo.portal.controller.SsoHeartbeatController` ，代码如下：

```Java
@Controller
@RequestMapping("/sso_heartbeat")
public class SsoHeartbeatController {

    @Autowired
    private SsoHeartbeatHandler handler;

    @RequestMapping(value = "", method = RequestMethod.GET)
    public void heartbeat(HttpServletRequest request, HttpServletResponse response) {
        handler.doHeartbeat(request, response);
    }

}
```

* 通过打开一个**新的窗口**，访问 `http://ip:prot/sso_hearbeat` 地址，每 60 秒刷新一次页面，从而避免 SSO 登陆过期。因此，相关类的类名都包含 Heartbeat ，代表**心跳**的意思。 

# 8. LogoutHandler

`com.ctrip.framework.apollo.portal.spi.LogoutHandler` ，用来实现登出功能。代码如下：

```Java
public interface LogoutHandler {

    void logout(HttpServletRequest request, HttpServletResponse response);

}
```

## 8.1 DefaultLogoutHandler

`com.ctrip.framework.apollo.portal.spi.defaultimpl.DefaultLogoutHandler` ，实现 LogoutHandler 接口，代码如下：

```Java
public class DefaultLogoutHandler implements LogoutHandler {

    @Override
    public void logout(HttpServletRequest request, HttpServletResponse response) {
        try {
            response.sendRedirect("/");
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

}
```

* 登出后，跳转到 `/` 地址。
* 😈 在使用 Spring Security 的请款下，不会调用到。**注意**，因为，我们配置了登出页。

# 666. 彩蛋

推荐与参考文章：

* [《【Spring】关于Boot应用中集成Spring Security你必须了解的那些事》](http://emacoo.cn/backend/spring-boot-security/)
* [《徐靖峰大彩笔 —— Spring Security》](https://www.cnkirito.moe/categories/Spring-Security/)

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)


