title: 【龙飞】Spring Security 源码分析十三：Spring Security 权限控制
date: 2019-01-30
tag:
categories: Spring Security
permalink: Spring-Security/longfei/Access0control
author: 龙飞
from_url: http://niocoder.com/2018/01/30/Spring-Security源码分析十三-Spring-Security权限控制/

-------

摘要: 原创出处 http://niocoder.com/2018/01/30/Spring-Security源码分析十三-Spring-Security权限控制/ 「龙飞」欢迎转载，保留摘要，谢谢！

- [1. 前言](http://www.iocoder.cn/Spring-Security/longfei/Access0control/)
  - [1.1 常见的表达式](http://www.iocoder.cn/Spring-Security/longfei/Access0control/)
  - [1.2 Method安全表达式](http://www.iocoder.cn/Spring-Security/longfei/Access0control/)
- [2. 代码下载](http://www.iocoder.cn/Spring-Security/longfei/Access0control/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

> Spring Security是一个能够为基于Spring的企业应用系统提供声明式的安全访问控制解决方案的安全框架。它提供了一组可以在Spring应用上下文中配置的Bean，充分利用了Spring IoC，DI（控制反转Inversion of Control ,DI:Dependency Injection 依赖注入）和AOP（面向切面编程）功能，为应用系统提供声明式的安全访问控制功能，减少了为企业系统安全控制编写大量重复代码的工作。

# 1. 前言

`spring security 3.0`已经可以使用`spring el`表达式来控制授权，允许在表达式中使用复杂的布尔逻辑来控制访问的权限。

## 1.1 常见的表达式

Spring Security可用表达式对象的基类是SecurityExpressionRoot。

| 表达式  |  描述 |
| ------------ | ------------ |
|  `hasRole([role]`) | 用户拥有制定的角色时返回true （`Spring security` 默认会带有`ROLE_`前缀）,去除参考[Remove the ROLE_](https://github.com/spring-projects/spring-security/issues/4134) |
|  `hasAnyRole([role1,role2])` |  用户拥有任意一个制定的角色时返回true|
| `hasAuthority([authority])`  | 等同于`hasRole`,但不会带有`ROLE_`前缀 |
| `hasAnyAuthority([auth1,auth2])`  | 等同于`hasAnyRole` |
|  `permitAll` | 永远返回true  |
|  `denyAll` |  永远返回false |
| `anonymous`  |  当前用户是`anonymous`时返回true |
|  `rememberMe` | 当前勇士是`rememberMe`用户返回true  |
| `authentication`  |  当前登录用户的`authentication`对象 |
| `fullAuthenticated`  | 当前用户既不是`anonymous`也不是`rememberMe`用户时返回true  |
| `hasIpAddress('192.168.1.0/24'))`  | 请求发送的IP匹配时返回true  |


部分代码：
```java
......
private String defaultRolePrefix = "ROLE_"; //ROLE_前缀

	/** Allows "permitAll" expression */
	public final boolean permitAll = true; //全部true

	/** Allows "denyAll" expression */
	public final boolean denyAll = false; //全部false
public final boolean permitAll() {
		return true;
	}

	public final boolean denyAll() {
		return false;
	}

	public final boolean isAnonymous() {
		//是否是anonymous
		return trustResolver.isAnonymous(authentication);
	}

	public final boolean isRememberMe() {
		//是否是rememberme
		return trustResolver.isRememberMe(authentication);
	}
......
```

### 1.1.1 URL安全表达式
```java
onfig.antMatchers("/person/*").access("hasRole('ADMIN') or hasRole('USER')")
                .anyRequest().authenticated();
```
这里我们定义了应用`/person/*`URL的范围，该URL只针对拥有`ADMIN`或者`USER`权限的用户有效。

### 在Web安全表达式中引用bean
```java
config.antMatchers("/person/*").access("hasRole('ADMIN') or hasRole('USER')")
                .antMatchers("/person/{id}").access("@rbacService.checkUserId(authentication,#id)")
                .anyRequest()
                .access("@rbacService.hasPermission(request,authentication)");
```

#### 1.1.1.1 RbacServiceImpl

```java
@Component("rbacService")
@Slf4j
public class RbacServiceImpl implements RbacService {
    /**
     * uri匹配工具
     */
    private AntPathMatcher antPathMatcher = new AntPathMatcher();

    @Override
    public boolean hasPermission(HttpServletRequest request, Authentication authentication) {
        log.info("【RbacServiceImpl】  --hasPermission={}", authentication.getPrincipal());
        Object principal = authentication.getPrincipal();

        boolean hasPermission = false;
        //有可能是匿名的anonymous
        if (principal instanceof SysUser) {
            //admin永远放回true
            if (StringUtils.equals("admin", ((SysUser) principal).getUsername())) {
                hasPermission = true;
            } else {
                //读取用户所拥有权限所有的URL 在这里全部返回true
                Set<String> urls = new HashSet<>();

                for (String url : urls) {
                    if (antPathMatcher.match(url, request.getRequestURI())) {
                        hasPermission = true;
                        break;
                    }
                }
            }
        }
        return hasPermission;
    }

	  public boolean checkUserId(Authentication authentication, int id) {
        return true;
    }
}
```
效果如下：

[![https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-el01.gif](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-el01.gif "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-el01.gif")](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-el01.gif "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-el01.gif")

## 1.2 Method安全表达式

针对方法级别的访问控制比较复杂，`Spring Security`提供了四种注解，分别是`@PreAuthorize` , `@PreFilter` , `@PostAuthorize` 和 `@PostFilter`

### 1.2.1 使用method注解

1. 开启方法级别注解的配置
```java
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class MerryyouSecurityConfig extends WebSecurityConfigurerAdapter {
```
2. 配置相应的bean
```java
 @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService).passwordEncoder(passwordEncoder());
    }

    @Bean
    @ConditionalOnMissingBean(PasswordEncoder.class)
    public PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }
```
3. 在方法上面使用注解
```java
 /**
     * 查询所有人员
     */
    @PreAuthorize("hasRole('ADMIN')")
    @ApiOperation(value = "获得person列表", notes = "")
    @GetMapping(value = "/persons")
    public List<Person> getPersons() {
        return personService.findAll();
    }
```

#### 1.2.1.1 PreAuthorize

@PreAuthorize 注解适合进入方法前的权限验证
```java
@PreAuthorize("hasRole('ADMIN')")
    List<Person> findAll();
```
#### PostAuthorize 
@PostAuthorize 在方法执行后再进行权限验证,适合验证带有返回值的权限。`Spring EL` 提供 返回对象能够在表达式语言中获取返回的对象`returnObject`。
```java
@PostAuthorize("returnObject.name == authentication.name")
    Person findOne(Integer id);
```
#### PreAuthorize 针对参数进行过滤
```java
//当有多个对象是使用filterTarget进行标注
@PreFilter(filterTarget="ids", value="filterObject%2==0")
   public void delete(List<Integer> ids, List<String> usernames) {
      ...

   }
```

#### 1.2.1.2 PostFilter 针对返回结果进行过滤

```java
 @PreAuthorize("hasRole('ADMIN')")
    @PostFilter("filterObject.name == authentication.name")
    List<Person> findAll();
```

效果如下：

[![https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-el02.gif](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-el02.gif "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-el02.gif")](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-el02.gif "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-el02.gif")

# 2. 代码下载

从我的 github 中下载，[https://github.com/longfeizheng/logback](https://github.com/longfeizheng/logback)

