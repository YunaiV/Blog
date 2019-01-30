title: 一起来学 SpringBoot 2.x | 第二十六篇：轻松搞定安全框架（Shiro）
date: 2018-01-27
tag: 
categories: Spring Boot
permalink: Spring-Boot/battcn/v2-other-shiro/
author: 唐亚峰
from_url: http://blog.battcn.com/2018/07/03/springboot/v2-other-shiro/
wechat_url: 

-------

摘要: 原创出处 http://blog.battcn.com/2018/07/03/springboot/v2-other-shiro/ 「唐亚峰」欢迎转载，保留摘要，谢谢！

- [Shiro](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-shiro//)
- [本章目标](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-shiro//)
  - [导入依赖](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-shiro//)
  - [属性配置](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-shiro//)
  - [缓存配置](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-shiro//)
  - [实体类](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-shiro//)
  - [伪造数据](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-shiro//)
  - [ShiroConfiguration](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-shiro//)
  - [AuthRealm](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-shiro//)
  - [控制器](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-shiro//)
  - [主函数](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-shiro//)
  - [测试](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-shiro//)
- [总结](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-shiro//)
- [666. 彩蛋](http://www.iocoder.cn/Spring-Boot/battcn/v2-other-shiro//)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

> `SpringBoot` 是为了简化 `Spring` 应用的创建、运行、调试、部署等一系列问题而诞生的产物，**自动装配的特性让我们可以更好的关注业务本身而不是外部的XML配置，我们只需遵循规范，引入相关的依赖就可以轻易的搭建出一个 WEB 工程**

**Shiro 是 Apache 旗下开源的一款强大且易用的Java安全框架，身份验证、授权、加密、会话管理。** 相比 `Spring Security` 而言 `Shiro`更加轻量级，且 API 更易于理解…

# Shiro

`Shiro` 主要分为 **安全认证** 和 **接口授权** 两个部分，其中的核心组件为 `Subject`、`SecurityManager`、`Realms`，公共部分 `Shiro` 都已经为我们封装好了，我们只需要按照一定的规则去编写响应的代码即可…

- **Subject** **即表示主体，将用户的概念理解为当前操作的主体，因为它即可以是一个通过浏览器请求的用户，也可能是一个运行的程序，外部应用与 Subject 进行交互，记录当前操作用户。Subject 代表了当前用户的安全操作，SecurityManager 则管理所有用户的安全操作。**
- **SecurityManager** **即安全管理器，对所有的 Subject 进行安全管理，并通过它来提供安全管理的各种服务（认证、授权等）**
- **Realm** 充当了应用与数据安全间的 **桥梁** 或 **连接器**。当对用户执行认证（登录）和授权（访问控制）验证时，Shiro 会从应用配置的 Realm 中查找用户及其权限信息。

# 本章目标

利用 `Spring Boot` 与 `Shiro` 实现安全认证和授权….

## 导入依赖

依赖 `spring-boot-starter-web`…

```xml
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <java.version>1.8</java.version>
    <shiro.version>1.4.0</shiro.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- shiro 相关包 -->
    <dependency>
        <groupId>org.apache.shiro</groupId>
        <artifactId>shiro-core</artifactId>
        <version>${shiro.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.shiro</groupId>
        <artifactId>shiro-spring</artifactId>
        <version>${shiro.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.shiro</groupId>
        <artifactId>shiro-ehcache</artifactId>
        <version>${shiro.version}</version>
    </dependency>
    <!-- End  -->
</dependencies>
```

## 属性配置

## 缓存配置

Shiro 为我们提供了 `CacheManager` 即缓存管理，将用户权限数据存储在缓存，可以提高它的性能。支持 `EhCache`、`Redis` 等常规缓存，这里为了简单起见就用 `EhCache` 了 ， 在`resources` 目录下创建一个 `ehcache-shiro.xml` 文件

```XML
<?xml version="1.0" encoding="UTF-8"?>
<ehcache updateCheck="false" name="shiroCache">
    <defaultCache
            maxElementsInMemory="10000"
            eternal="false"
            timeToIdleSeconds="120"
            timeToLiveSeconds="120"
            overflowToDisk="false"
            diskPersistent="false"
            diskExpiryThreadIntervalSeconds="120"
    />
</ehcache>
```

## 实体类

创建一个 `User.java` ，标记为数据库用户

```java
package com.battcn.entity;

/**
 * @author Levin
 * @since 2018/6/28 0028
 */
public class User {
    /** 自增ID */
    private Long id;
    /** 账号 */
    private String username;
    /** 密码 */
    private String password;
    /** 角色名：Shiro 支持多个角色，而且接收参数也是 Set<String> 集合，但这里为了简单起见定义成 String 类型了 */
    private String roleName;
    /** 是否禁用 */
    private boolean locked;
    // 省略 GET SET 构造函数...
}
```

## 伪造数据

支持 `roles`、`permissions`，比如你一个接口可以允许用户拥有某一个角色，也可以是拥有某一个 `permission` …

```java
package com.battcn.config;

import com.battcn.entity.User;

import java.util.*;

/**
 * 主要不想连接数据库..
 *
 * @author Levin
 * @since 2018/6/28 0028
 */
public class DBCache {

    /**
     * K 用户名
     * V 用户信息
     */
    public static final Map<String, User> USERS_CACHE = new HashMap<>();
    /**
     * K 角色ID
     * V 权限编码
     */
    public static final Map<String, Collection<String>> PERMISSIONS_CACHE = new HashMap<>();

    static {
        // TODO 假设这是数据库记录
        USERS_CACHE.put("u1", new User(1L, "u1", "p1", "admin", true));
        USERS_CACHE.put("u2", new User(2L, "u2", "p2", "admin", false));
        USERS_CACHE.put("u3", new User(3L, "u3", "p3", "test", true));

        PERMISSIONS_CACHE.put("admin", Arrays.asList("user:list", "user:add", "user:edit"));
        PERMISSIONS_CACHE.put("test", Collections.singletonList("user:list"));

    }
}
```

## ShiroConfiguration

Shiro 的主要配置信息都在此文件内实现；

```java
package com.battcn.config;


import org.apache.shiro.cache.ehcache.EhCacheManager;
import org.apache.shiro.spring.LifecycleBeanPostProcessor;
import org.apache.shiro.spring.security.interceptor.AuthorizationAttributeSourceAdvisor;
import org.apache.shiro.spring.web.ShiroFilterFactoryBean;
import org.apache.shiro.web.mgt.DefaultWebSecurityManager;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.LinkedHashMap;
import java.util.Map;

/**
 * Shiro 配置
 *
 * @author Levin
 */
@Configuration
public class ShiroConfiguration {

    private static final Logger log = LoggerFactory.getLogger(ShiroConfiguration.class);

    @Bean
    public EhCacheManager getEhCacheManager() {
        EhCacheManager em = new EhCacheManager();
        em.setCacheManagerConfigFile("classpath:ehcache-shiro.xml");
        return em;
    }


    @Bean(name = "lifecycleBeanPostProcessor")
    public LifecycleBeanPostProcessor getLifecycleBeanPostProcessor() {
        return new LifecycleBeanPostProcessor();
    }


    /**
     * 加密器：这样一来数据库就可以是密文存储，为了演示我就不开启了
     *
     * @return HashedCredentialsMatcher
     */
//    @Bean
//    public HashedCredentialsMatcher hashedCredentialsMatcher() {
//        HashedCredentialsMatcher hashedCredentialsMatcher = new HashedCredentialsMatcher();
//        //散列算法:这里使用MD5算法;
//        hashedCredentialsMatcher.setHashAlgorithmName("md5");
//        //散列的次数，比如散列两次，相当于 md5(md5(""));
//        hashedCredentialsMatcher.setHashIterations(2);
//        return hashedCredentialsMatcher;
//    }


    @Bean
    public DefaultAdvisorAutoProxyCreator getDefaultAdvisorAutoProxyCreator() {
        DefaultAdvisorAutoProxyCreator autoProxyCreator = new DefaultAdvisorAutoProxyCreator();
        autoProxyCreator.setProxyTargetClass(true);
        return autoProxyCreator;
    }

    @Bean(name = "authRealm")
    public AuthRealm authRealm(EhCacheManager cacheManager) {
        AuthRealm authRealm = new AuthRealm();
        authRealm.setCacheManager(cacheManager);
        return authRealm;
    }

    @Bean(name = "securityManager")
    public DefaultWebSecurityManager getDefaultWebSecurityManager(AuthRealm authRealm) {
        DefaultWebSecurityManager defaultWebSecurityManager = new DefaultWebSecurityManager();
        defaultWebSecurityManager.setRealm(authRealm);
        // <!-- 用户授权/认证信息Cache, 采用EhCache 缓存 -->
        defaultWebSecurityManager.setCacheManager(getEhCacheManager());
        return defaultWebSecurityManager;
    }

    @Bean
    public AuthorizationAttributeSourceAdvisor getAuthorizationAttributeSourceAdvisor(
            DefaultWebSecurityManager securityManager) {
        AuthorizationAttributeSourceAdvisor advisor = new AuthorizationAttributeSourceAdvisor();
        advisor.setSecurityManager(securityManager);
        return advisor;
    }

    /**
     * ShiroFilter<br/>
     * 注意这里参数中的 StudentService 和 IScoreDao 只是一个例子，因为我们在这里可以用这样的方式获取到相关访问数据库的对象，
     * 然后读取数据库相关配置，配置到 shiroFilterFactoryBean 的访问规则中。实际项目中，请使用自己的Service来处理业务逻辑。
     *
     * @param securityManager 安全管理器
     * @return ShiroFilterFactoryBean
     */
    @Bean(name = "shiroFilter")
    public ShiroFilterFactoryBean getShiroFilterFactoryBean(DefaultWebSecurityManager securityManager) {
        ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
        // 必须设置 SecurityManager
        shiroFilterFactoryBean.setSecurityManager(securityManager);
        // 如果不设置默认会自动寻找Web工程根目录下的"/login"页面
        shiroFilterFactoryBean.setLoginUrl("/login");
        // 登录成功后要跳转的连接
        shiroFilterFactoryBean.setSuccessUrl("/index");
        shiroFilterFactoryBean.setUnauthorizedUrl("/denied");
        loadShiroFilterChain(shiroFilterFactoryBean);
        return shiroFilterFactoryBean;
    }

    /**
     * 加载shiroFilter权限控制规则（从数据库读取然后配置）
     */
    private void loadShiroFilterChain(ShiroFilterFactoryBean shiroFilterFactoryBean) {
        /////////////////////// 下面这些规则配置最好配置到配置文件中 ///////////////////////
        // TODO 重中之重啊，过滤顺序一定要根据自己需要排序
        Map<String, String> filterChainDefinitionMap = new LinkedHashMap<>();
        // 需要验证的写 authc 不需要的写 anon
        filterChainDefinitionMap.put("/resource/**", "anon");
        filterChainDefinitionMap.put("/install", "anon");
        filterChainDefinitionMap.put("/hello", "anon");
        // anon：它对应的过滤器里面是空的,什么都没做
        log.info("##################从数据库读取权限规则，加载到shiroFilter中##################");

        // 不用注解也可以通过 API 方式加载权限规则
        Map<String, String> permissions = new LinkedHashMap<>();
        permissions.put("/users/find", "perms[user:find]");
        filterChainDefinitionMap.putAll(permissions);
        filterChainDefinitionMap.put("/**", "authc");
        shiroFilterFactoryBean.setFilterChainDefinitionMap(filterChainDefinitionMap);
    }
}
```

## AuthRealm

上面介绍过 `Realm` ，安全认证和权限验证的核心处理就是重写 `AuthorizingRealm` 中的 `doGetAuthenticationInfo（登录认证）` 与 `doGetAuthorizationInfo（权限验证）`

```java
package com.battcn.config;

import com.battcn.entity.User;
import org.apache.shiro.SecurityUtils;
import org.apache.shiro.authc.*;
import org.apache.shiro.authz.AuthorizationInfo;
import org.apache.shiro.authz.SimpleAuthorizationInfo;
import org.apache.shiro.realm.AuthorizingRealm;
import org.apache.shiro.session.Session;
import org.apache.shiro.subject.PrincipalCollection;
import org.apache.shiro.util.ByteSource;
import org.springframework.context.annotation.Configuration;

import java.util.*;

/**
 * 认证领域
 *
 * @author Levin
 * @version 2.5.1
 * @since 2018-01-10
 */
@Configuration
public class AuthRealm extends AuthorizingRealm {

    /**
     * 认证回调函数,登录时调用
     * 首先根据传入的用户名获取User信息；然后如果user为空，那么抛出没找到帐号异常UnknownAccountException；
     * 如果user找到但锁定了抛出锁定异常LockedAccountException；最后生成AuthenticationInfo信息，
     * 交给间接父类AuthenticatingRealm使用CredentialsMatcher进行判断密码是否匹配，
     * 如果不匹配将抛出密码错误异常IncorrectCredentialsException；
     * 另外如果密码重试此处太多将抛出超出重试次数异常ExcessiveAttemptsException；
     * 在组装SimpleAuthenticationInfo信息时， 需要传入：身份信息（用户名）、凭据（密文密码）、盐（username+salt），
     * CredentialsMatcher使用盐加密传入的明文密码和此处的密文密码进行匹配。
     */
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token)
            throws AuthenticationException {
        String principal = (String) token.getPrincipal();
        User user = Optional.ofNullable(DBCache.USERS_CACHE.get(principal)).orElseThrow(UnknownAccountException::new);
        if (!user.isLocked()) {
            throw new LockedAccountException();
        }
        // 从数据库查询出来的账号名和密码,与用户输入的账号和密码对比
        // 当用户执行登录时,在方法处理上要实现 user.login(token)
        // 然后会自动进入这个类进行认证
        // 交给 AuthenticatingRealm 使用 CredentialsMatcher 进行密码匹配，如果觉得人家的不好可以自定义实现
        // TODO 如果使用 HashedCredentialsMatcher 这里认证方式就要改一下 SimpleAuthenticationInfo authenticationInfo = new SimpleAuthenticationInfo(principal, "密码", ByteSource.Util.bytes("密码盐"), getName());
        SimpleAuthenticationInfo authenticationInfo = new SimpleAuthenticationInfo(principal, user.getPassword(), getName());
        Session session = SecurityUtils.getSubject().getSession();
        session.setAttribute("USER_SESSION", user);
        return authenticationInfo;
    }

    /**
     * 只有需要验证权限时才会调用, 授权查询回调函数, 进行鉴权但缓存中无用户的授权信息时调用.在配有缓存的情况下，只加载一次.
     * 如果需要动态权限,但是又不想每次去数据库校验,可以存在ehcache中.自行完善
     */
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principal) {
        Session session = SecurityUtils.getSubject().getSession();
        User user = (User) session.getAttribute("USER_SESSION");
        // 权限信息对象info,用来存放查出的用户的所有的角色（role）及权限（permission）
        SimpleAuthorizationInfo info = new SimpleAuthorizationInfo();
        // 用户的角色集合
        Set<String> roles = new HashSet<>();
        roles.add(user.getRoleName());
        info.setRoles(roles);
        // 用户的角色对应的所有权限，如果只使用角色定义访问权限，下面可以不要
        // 只有角色并没有颗粒度到每一个按钮 或 是操作选项  PERMISSIONS 是可选项
        final Map<String, Collection<String>> permissionsCache = DBCache.PERMISSIONS_CACHE;
        final Collection<String> permissions = permissionsCache.get(user.getRoleName());
        info.addStringPermissions(permissions);
        return info;
    }
}
```

## 控制器

在 `ShiroConfiguration` 中的 `shiroFilter` 处配置了 `/hello = anon`，意味着可以不需要认证也可以访问，那么除了这种方式外 `Shiro` 还为我们提供了一些注解相关的方式…

> 常用注解

- **@RequiresGuest** 代表无需认证即可访问，同理的就是 `/path = anon`
- **@RequiresAuthentication** 需要认证，只要登录成功后就允许你操作
- **@RequiresPermissions** 需要特定的权限，没有则抛出`AuthorizationException`
- **@RequiresRoles** 需要特定的橘色，没有则抛出`AuthorizationException`
- **@RequiresUser** 不太清楚，不常用…

### LoginController

```Java
package com.battcn.controller;

import com.battcn.config.ShiroConfiguration;
import org.apache.shiro.SecurityUtils;
import org.apache.shiro.authc.*;
import org.apache.shiro.subject.Subject;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

/**
 * @author Levin
 * @since 2018/6/28 0028
 */
@RestController
public class LoginController {

    private static final Logger log = LoggerFactory.getLogger(ShiroConfiguration.class);

    @GetMapping(value = "/hello")
    public String hello() {
        log.info("不登录也可以访问...");
        return "hello...";
    }

    @GetMapping(value = "/index")
    public String index() {
        log.info("登陆成功了...");
        return "index";
    }

    @GetMapping(value = "/denied")
    public String denied() {
        log.info("小伙子权限不足,别无谓挣扎了...");
        return "denied...";
    }

    @GetMapping(value = "/login")
    public String login(String username, String password, RedirectAttributes model) {
        // 想要得到 SecurityUtils.getSubject() 的对象．．访问地址必须跟 shiro 的拦截地址内．不然后会报空指针
        Subject sub = SecurityUtils.getSubject();
        // 用户输入的账号和密码,,存到UsernamePasswordToken对象中..然后由shiro内部认证对比,
        // 认证执行者交由 com.battcn.config.AuthRealm 中 doGetAuthenticationInfo 处理
        // 当以上认证成功后会向下执行,认证失败会抛出异常
        UsernamePasswordToken token = new UsernamePasswordToken(username, password);
        try {
            sub.login(token);
        } catch (UnknownAccountException e) {
            log.error("对用户[{}]进行登录验证,验证未通过,用户不存在", username);
            token.clear();
            return "UnknownAccountException";
        } catch (LockedAccountException lae) {
            log.error("对用户[{}]进行登录验证,验证未通过,账户已锁定", username);
            token.clear();
            return "LockedAccountException";
        } catch (ExcessiveAttemptsException e) {
            log.error("对用户[{}]进行登录验证,验证未通过,错误次数过多", username);
            token.clear();
            return "ExcessiveAttemptsException";
        } catch (AuthenticationException e) {
            log.error("对用户[{}]进行登录验证,验证未通过,堆栈轨迹如下", username, e);
            token.clear();
            return "AuthenticationException";
        }
        return "success";
    }
}
```

### UserController

```Java
package com.battcn.controller;

import org.apache.shiro.authz.annotation.Logical;
import org.apache.shiro.authz.annotation.RequiresRoles;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @author Levin
 * @since 2018/6/28 0028
 */
@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping
    public String get() {
        return "get.....";
    }

    /**
     * RequiresRoles 是所需角色 包含 AND 和 OR 两种
     * RequiresPermissions 是所需权限 包含 AND 和 OR 两种
     *
     * @return msg
     */
    @RequiresRoles(value = {"admin", "test"}, logical = Logical.OR)
    //@RequiresPermissions(value = {"user:list", "user:query"}, logical = Logical.OR)
    @GetMapping("/query")
    public String query() {
        return "query.....";
    }

    @GetMapping("/find")
    public String find() {
        return "find.....";
    }
}
```

## 主函数

```Java
package com.battcn;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;


/**
 * @author Levin
 */
@SpringBootApplication
public class Chapter25Application {

    public static void main(String[] args) {

        SpringApplication.run(Chapter25Application.class, args);

    }
}
```

## 测试

启动 `Chapter25Application.java` 中的 `main` 方法，为了更好的演示效果这里打开了 `postman` 做的测试，只演示其中一个流程，剩下的可以自己复制代码测试…

先登录，由于 `u3` 在 `DBCache` 中拥有的角色是 `test`，只有 `user:list` 这一个权限

[![登录](https://image.battcn.com/article/images/20180703/springboot/v2-other-shiro/1.png)](https://image.battcn.com/article/images/20180703/springboot/v2-other-shiro/1.png)登录

访问 `/users/query` 成功，因为我们符合响应的角色/权限

[![访问Query接口](https://image.battcn.com/article/images/20180703/springboot/v2-other-shiro/2.png)](https://image.battcn.com/article/images/20180703/springboot/v2-other-shiro/2.png)访问Query接口

访问 `/users/find` 失败，并重定向到了 `/denied` 接口，问题来了为什么 `/users/find` 没有写注解也权限不足呢？

[![权限不足](https://image.battcn.com/article/images/20180703/springboot/v2-other-shiro/3.png)](https://image.battcn.com/article/images/20180703/springboot/v2-other-shiro/3.png)权限不足

细心的朋友肯定会发现 **在 ShiroConfiguration 中写了一句 permissions.put(“/users/find”, “perms[user:find]”);** 意味着我们不仅可以通过注解方式，同样可以通过初始化时加载数据库中的权限树做控制，看各位喜好了….

# 总结

目前很多大佬都写过关于 **SpringBoot** 的教程了，如有雷同，请多多包涵，本教程基于最新的 `spring-boot-starter-parent：2.0.3.RELEASE` 编写，包括新版本的特性都会一起介绍…

全文代码：https://github.com/battcn/spring-boot2-learning/tree/master/chapter26

# 666. 彩蛋

如果你对 SpringBoot 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)