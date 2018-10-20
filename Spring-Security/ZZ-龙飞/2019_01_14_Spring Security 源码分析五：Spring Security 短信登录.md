title: 【龙飞】Spring Security 源码分析五：Spring Security 短信登录
date: 2019-01-14
tag:
categories: Spring Security
permalink: Spring-Security/longfei/SMS-login
author: 龙飞
from_url: http://niocoder.com/2018/01/14/Spring-Security源码分析五-Spring-Security短信登录/

-------

摘要: 原创出处 http://niocoder.com/2018/01/14/Spring-Security源码分析五-Spring-Security短信登录/ 「龙飞」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Spring-Security/longfei/SMS-login/)
  - [1.1 目录结构](http://www.iocoder.cn/Spring-Security/longfei/SMS-login/)
  - [1.2 SmsCodeAuthenticationFilter](http://www.iocoder.cn/Spring-Security/longfei/SMS-login/)
  - [1.3 SmsCodeAuthenticationToken](http://www.iocoder.cn/Spring-Security/longfei/SMS-login/)
  - [1.4 SmsCodeAuthenticationProvider](http://www.iocoder.cn/Spring-Security/longfei/SMS-login/)
  - [1.5 MerryyouSecurityConfig 主配置文件](http://www.iocoder.cn/Spring-Security/longfei/SMS-login/)
  - [1.6 调试过程](http://www.iocoder.cn/Spring-Security/longfei/SMS-login/)
- [2. 代码下载](http://www.iocoder.cn/Spring-Security/longfei/SMS-login/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

> 目前常见的社交软件、购物软件、支付软件、理财软件等，均需要用户进行登录才可享受软件提供的服务。目前主流的登录方式主要有 3 种：账号密码登录、短信验证码登录和第三方授权登录。我们已经实现了账号密码和第三方授权登录。本章我们将使用`Spring Security`实现短信验证码登录。


# 1. 概述

在[Spring Security源码分析一：Spring Security认证过程](https://longfeizheng.github.io/2018/01/02/Spring-Security%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B8%80-Spring-Security%E8%AE%A4%E8%AF%81%E8%BF%87%E7%A8%8B/#%E7%B1%BB%E5%9B%BE)和[Spring Security源码分析二：Spring Security授权过程](https://longfeizheng.github.io/2018/01/05/Spring-Security%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%BA%8C-Spring-Security%E6%8E%88%E6%9D%83%E8%BF%87%E7%A8%8B/)两章中。我们已经详细解读过`Spring Security`如何处理用户名和密码登录。（其实就是过滤器链）本章我们将仿照用户名密码来显示短信登录。


## 1.1 目录结构

[![http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Security-sms.png](http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Security-sms.png "http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Security-sms.png")](http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Security-sms.png "http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Security-sms.png")

## 1.2 SmsCodeAuthenticationFilter

`SmsCodeAuthenticationFilter`对应用户名密码登录的[UsernamePasswordAuthenticationFilter](https://longfeizheng.github.io/2018/01/05/Spring-Security%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%BA%8C-Spring-Security%E6%8E%88%E6%9D%83%E8%BF%87%E7%A8%8B/#usernamepasswordauthenticationfilter-1)同样继承[AbstractAuthenticationProcessingFilter](https://longfeizheng.github.io/2018/01/05/Spring-Security%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%BA%8C-Spring-Security%E6%8E%88%E6%9D%83%E8%BF%87%E7%A8%8B/#abstractauthenticationprocessingfilter)

```java
public class SmsCodeAuthenticationFilter extends AbstractAuthenticationProcessingFilter {

    /**
     * request中必须含有mobile参数
     */
    private String mobileParameter = SecurityConstants.DEFAULT_PARAMETER_NAME_MOBILE;
    /**
     * post请求
     */
    private boolean postOnly = true;

    protected SmsCodeAuthenticationFilter() {
        /**
         * 处理的手机验证码登录请求处理url
         */
        super(new AntPathRequestMatcher(SecurityConstants.DEFAULT_SIGN_IN_PROCESSING_URL_MOBILE, "POST"));
    }

    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException, IOException, ServletException {
        //判断是是不是post请求
        if (postOnly && !request.getMethod().equals("POST")) {
            throw new AuthenticationServiceException("Authentication method not supported: " + request.getMethod());
        }
        //从请求中获取手机号码
        String mobile = obtainMobile(request);

        if (mobile == null) {
            mobile = "";
        }

        mobile = mobile.trim();
        //创建SmsCodeAuthenticationToken(未认证)
        SmsCodeAuthenticationToken authRequest = new SmsCodeAuthenticationToken(mobile);

        //设置用户信息
        setDetails(request, authRequest);
        //返回Authentication实例
        return this.getAuthenticationManager().authenticate(authRequest);
    }

    /**
     * 获取手机号
     */
    protected String obtainMobile(HttpServletRequest request) {
        return request.getParameter(mobileParameter);
    }

    protected void setDetails(HttpServletRequest request, SmsCodeAuthenticationToken authRequest) {
        authRequest.setDetails(authenticationDetailsSource.buildDetails(request));
    }

    public void setMobileParameter(String usernameParameter) {
        Assert.hasText(usernameParameter, "Username parameter must not be empty or null");
        this.mobileParameter = usernameParameter;
    }

    public void setPostOnly(boolean postOnly) {
        this.postOnly = postOnly;
    }

    public final String getMobileParameter() {
        return mobileParameter;
    }
}
```
1. 认证请求的方法必须为`POST`
2. 从request中获取手机号
3. 封装成自己的`Authenticaiton`的实现类`SmsCodeAuthenticationToken`（未认证）
4. 调用 `AuthenticationManager` 的 `authenticate` 方法进行验证（即`SmsCodeAuthenticationProvider`）

## 1.3 SmsCodeAuthenticationToken

`SmsCodeAuthenticationToken`对应用户名密码登录的`UsernamePasswordAuthenticationToken`
```java
public class SmsCodeAuthenticationToken extends AbstractAuthenticationToken {
    private static final long serialVersionUID = 2383092775910246006L;

    /**
     * 手机号
     */
    private final Object principal;

    /**
     * SmsCodeAuthenticationFilter中构建的未认证的Authentication
     * @param mobile
     */
    public SmsCodeAuthenticationToken(String mobile) {
        super(null);
        this.principal = mobile;
        setAuthenticated(false);
    }

    /**
     * SmsCodeAuthenticationProvider中构建已认证的Authentication
     * @param principal
     * @param authorities
     */
    public SmsCodeAuthenticationToken(Object principal,
                                      Collection<? extends GrantedAuthority> authorities) {
        super(authorities);
        this.principal = principal;
        super.setAuthenticated(true); // must use super, as we override
    }

    @Override
    public Object getCredentials() {
        return null;
    }

    @Override
    public Object getPrincipal() {
        return this.principal;
    }

    /**
     * @param isAuthenticated
     * @throws IllegalArgumentException
     */
    public void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException {
        if (isAuthenticated) {
            throw new IllegalArgumentException(
                    "Cannot set this token to trusted - use constructor which takes a GrantedAuthority list instead");
        }

        super.setAuthenticated(false);
    }

    @Override
    public void eraseCredentials() {
        super.eraseCredentials();
    }
}
```

## 1.4 SmsCodeAuthenticationProvider

`SmsCodeAuthenticationProvider`对应用户名密码登录的[DaoAuthenticationProvider](https://longfeizheng.github.io/2018/01/02/Spring-Security%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B8%80-Spring-Security%E8%AE%A4%E8%AF%81%E8%BF%87%E7%A8%8B/#daoauthenticationprovider)
```java
public class SmsCodeAuthenticationProvider implements AuthenticationProvider {

    private UserDetailsService userDetailsService;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        SmsCodeAuthenticationToken authenticationToken = (SmsCodeAuthenticationToken) authentication;
        //调用自定义的userDetailsService认证
        UserDetails user = userDetailsService.loadUserByUsername((String) authenticationToken.getPrincipal());

        if (user == null) {
            throw new InternalAuthenticationServiceException("无法获取用户信息");
        }
        //如果user不为空重新构建SmsCodeAuthenticationToken（已认证）
        SmsCodeAuthenticationToken authenticationResult = new SmsCodeAuthenticationToken(user, user.getAuthorities());

        authenticationResult.setDetails(authenticationToken.getDetails());

        return authenticationResult;
    }
	
	/**
     * 只有Authentication为SmsCodeAuthenticationToken使用此Provider认证
     * @param authentication
     * @return
     */
    @Override
    public boolean supports(Class<?> authentication) {
        return SmsCodeAuthenticationToken.class.isAssignableFrom(authentication);
    }

    public UserDetailsService getUserDetailsService() {
        return userDetailsService;
    }

    public void setUserDetailsService(UserDetailsService userDetailsService) {
        this.userDetailsService = userDetailsService;
    }
}
```

### 1.4.1 SmsCodeAuthenticationSecurityConfig短信登录配置

```java
@Component
public class SmsCodeAuthenticationSecurityConfig extends SecurityConfigurerAdapter<DefaultSecurityFilterChain, HttpSecurity> {

    @Autowired
    private AuthenticationFailureHandler merryyouAuthenticationFailureHandler;

    @Autowired
    private UserDetailsService userDetailsService;

    @Override
    public void configure(HttpSecurity http) throws Exception {
        //自定义SmsCodeAuthenticationFilter过滤器
        SmsCodeAuthenticationFilter smsCodeAuthenticationFilter = new SmsCodeAuthenticationFilter();
        smsCodeAuthenticationFilter.setAuthenticationManager(http.getSharedObject(AuthenticationManager.class));
        smsCodeAuthenticationFilter.setAuthenticationFailureHandler(merryyouAuthenticationFailureHandler);

        //设置自定义SmsCodeAuthenticationProvider的认证器userDetailsService
        SmsCodeAuthenticationProvider smsCodeAuthenticationProvider = new SmsCodeAuthenticationProvider();
        smsCodeAuthenticationProvider.setUserDetailsService(userDetailsService);
        //在UsernamePasswordAuthenticationFilter过滤前执行
        http.authenticationProvider(smsCodeAuthenticationProvider)
                .addFilterAfter(smsCodeAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
    }
}

```

## 1.5 MerryyouSecurityConfig 主配置文件

```java
 @Override
    protected void configure(HttpSecurity http) throws Exception {
//        http.addFilterBefore(validateCodeFilter, UsernamePasswordAuthenticationFilter.class)
        http
                .formLogin()//使用表单登录，不再使用默认httpBasic方式
                .loginPage(SecurityConstants.DEFAULT_UNAUTHENTICATION_URL)//如果请求的URL需要认证则跳转的URL
                .loginProcessingUrl(SecurityConstants.DEFAULT_SIGN_IN_PROCESSING_URL_FORM)//处理表单中自定义的登录URL
                .and()
                .apply(validateCodeSecurityConfig)//验证码拦截
                .and()
                .apply(smsCodeAuthenticationSecurityConfig)
                .and()
                .apply(merryyouSpringSocialConfigurer)//社交登录
                .and()
                .rememberMe()
......
```

## 1.6 调试过程

短信登录拦截请求`/authentication/mobile`

[![http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/spring-sms-01.png](http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/spring-sms-01.png "http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/spring-sms-01.png")](http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/spring-sms-01.png "http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/spring-sms-01.png")

自定义`SmsCodeAuthenticationProvider`
[![http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/spring-sms-02.png](http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/spring-sms-02.png "http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/spring-sms-02.png")](http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/spring-sms-02.png "http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/spring-sms-02.png")


效果如下：
[![http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/121.gif.gif](http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/121.gif.gif "http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/121.gif.gif")](http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/121.gif.gif "http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/121.gif.gif")

# 2. 代码下载

从我的 github 中下载，[https://github.com/longfeizheng/logback](https://github.com/longfeizheng/logback)






