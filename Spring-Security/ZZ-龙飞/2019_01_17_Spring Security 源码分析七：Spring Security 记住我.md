title: 【龙飞】Spring Security 源码分析七：Spring Security 记住我
date: 2019-01-17
tag:
categories: Spring Security
permalink: Spring-Security/longfei/Remember-Me
author: 龙飞
from_url: http://niocoder.com/2018/01/17/Spring-Security源码分析七-Spring-Security-记住我/

-------

摘要: 原创出处 http://niocoder.com/2018/01/17/Spring-Security源码分析七-Spring-Security-记住我/ 「龙飞」欢迎转载，保留摘要，谢谢！

- [1. 记住我基本原理](http://www.iocoder.cn/Spring-Security/longfei/Remember-Me/)
  - [1.1 记住我实现](http://www.iocoder.cn/Spring-Security/longfei/Remember-Me/)
  - [1.2 效果如下](http://www.iocoder.cn/Spring-Security/longfei/Remember-Me/)
  - [1.3 源码分析](http://www.iocoder.cn/Spring-Security/longfei/Remember-Me/)
- [2. 代码下载](http://www.iocoder.cn/Spring-Security/longfei/Remember-Me/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

> 有这样一个场景——有个用户初访并登录了你的网站，然而第二天他又来了，却必须再次登录。于是就有了“记住我”这样的功能来方便用户使用，然而有一件不言自明的事情，那就是这种认证状态的”旷日持久“早已超出了用户原本所需要的使用范围。这意味着，他们可以关闭浏览器，然后再关闭电脑，下周或者下个月，乃至更久以后再回来，只要这间隔时间不要太离谱，该网站总会知道谁是谁，并一如既往的为他们提供所有相同的功能和服务——与许久前他们离开的时候别无二致。

# 1. 记住我基本原理

[![http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Security-remeber.png](http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Security-remeber.png "http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Security-remeber.png")](http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Security-remeber.png "http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Security-remeber.png")

1. 用户认证成功之后调用`RemeberMeService`根据用户名名生成`Token`由`TokenRepository`写入到数据库，同时也将`Token`写入到浏览器的`Cookie`中
2. 重启服务之后，用户再次登入系统会由`RememberMeAuthenticationFilter`拦截，从`Cookie`中读取`Token`信息,与`persistent_logins`表匹配判断是否使用记住我功能。最中由`UserDetailsService`查询用户信息


## 1.1 记住我实现

1. 创建`persistent_logins`表
```sql
create table persistent_logins (username varchar(64) not null, series varchar(64) primary key, token varchar(64) not null, last_used timestamp not null);
```
2. 登陆页面添加记住我复选款（name必须是remeber-me）
```html
<input name="remember-me" type="checkbox"> 下次自动登录
```
3. 配置[MerryyouSecurityConfig](https://github.com/longfeizheng/logback/blob/master/src/main/java/cn/merryyou/logback/security/MerryyouSecurityConfig.java#L72)
```java
http.
......
                .and()
                .rememberMe()
                .tokenRepository(persistentTokenRepository())//设置操作表的Repository
                .tokenValiditySeconds(securityProperties.getRememberMeSeconds())//设置记住我的时间
                .userDetailsService(userDetailsService)//设置userDetailsService
                .and()
	......
```

## 1.2 效果如下

[![https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-remeberme.gif](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-remeberme.gif "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-remeberme.gif")](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-remeberme.gif "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-remeberme.gif")

## 1.3 源码分析

### 1.3.1 首次登录

#### 1.3.1.1 AbstractAuthenticationProcessingFilter#successfulAuthentication

```java
protected void successfulAuthentication(HttpServletRequest request,
			HttpServletResponse response, FilterChain chain, Authentication authResult)
			throws IOException, ServletException {

		if (logger.isDebugEnabled()) {
			logger.debug("Authentication success. Updating SecurityContextHolder to contain: "
					+ authResult);
		}
		//# 1.将已认证过的Authentication放入到SecurityContext中
		SecurityContextHolder.getContext().setAuthentication(authResult);
		//# 2.登录成功调用rememberMeServices
		rememberMeServices.loginSuccess(request, response, authResult);

		// Fire event
		if (this.eventPublisher != null) {
			eventPublisher.publishEvent(new InteractiveAuthenticationSuccessEvent(
					authResult, this.getClass()));
		}

		successHandler.onAuthenticationSuccess(request, response, authResult);
	}
```

1. 将已认证过的Authentication放入到SecurityContext中
2. 登录成功调用rememberMeServices

#### 1.3.1.2 AbstractRememberMeServices#loginSuccess

```java
private String parameter = DEFAULT_PARAMETER;//remember-me

public final void loginSuccess(HttpServletRequest request,
			HttpServletResponse response, Authentication successfulAuthentication) {
		// #1.判断是否勾选记住我
		if (!rememberMeRequested(request, parameter)) {
			logger.debug("Remember-me login not requested.");
			return;
		}

		onLoginSuccess(request, response, successfulAuthentication);
	}
```
1. 判断是否勾选记住我

#### 1.3.1.3 PersistentTokenBasedRememberMeServices#onLoginSuccess

```java
protected void onLoginSuccess(HttpServletRequest request,
			HttpServletResponse response, Authentication successfulAuthentication) {
		//#1.获取用户名
		String username = successfulAuthentication.getName();

		logger.debug("Creating new persistent login for user " + username);
		//#2.创建Token
		PersistentRememberMeToken persistentToken = new PersistentRememberMeToken(
				username, generateSeriesData(), generateTokenData(), new Date());
		try {
			//#3.存储都数据库
			tokenRepository.createNewToken(persistentToken);
			//#4.写入到浏览器的Cookie中
			addCookie(persistentToken, request, response);
		}
		catch (Exception e) {
			logger.error("Failed to save persistent token ", e);
		}
	}
```

1. 获取用户名
2. 创建Token
3. 存储都数据库
4. 写入到浏览器的Cookie中


### 1.3.2 二次登录Remember-me

#### 1.3.2.1 RememberMeAuthenticationFilter#doFilter

```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {
		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;
		//#1.判断SecurityContext中没有Authentication
		if (SecurityContextHolder.getContext().getAuthentication() == null) {
			//#2.从Cookie查询用户信息返回RememberMeAuthenticationToken
			Authentication rememberMeAuth = rememberMeServices.autoLogin(request,
					response);

			if (rememberMeAuth != null) {
				// Attempt authenticaton via AuthenticationManager
				try {
					//#3.如果不为空则由authenticationManager认证
					rememberMeAuth = authenticationManager.authenticate(rememberMeAuth);

					// Store to SecurityContextHolder
					SecurityContextHolder.getContext().setAuthentication(rememberMeAuth);

					onSuccessfulAuthentication(request, response, rememberMeAuth);
......
```

1. 判断SecurityContext中没有Authentication
2. 从Cookie查询用户信息返回RememberMeAuthenticationToken
3. 如果不为空则由authenticationManager认证

#### 1.3.2.2 AbstractRememberMeServices#autoLogin

```java
public final Authentication autoLogin(HttpServletRequest request,
			HttpServletResponse response) {
			//#1.获取Cookie
		String rememberMeCookie = extractRememberMeCookie(request);

		if (rememberMeCookie == null) {
			return null;
		}

		logger.debug("Remember-me cookie detected");

		if (rememberMeCookie.length() == 0) {
			logger.debug("Cookie was empty");
			cancelCookie(request, response);
			return null;
		}

		UserDetails user = null;

		try {
			//#2.解析Cookie
			String[] cookieTokens = decodeCookie(rememberMeCookie);
			//#3.获取用户凭证
			user = processAutoLoginCookie(cookieTokens, request, response);
			//#4.检查用户凭证
			userDetailsChecker.check(user);

			logger.debug("Remember-me cookie accepted");
			//#5.返回Authentication
			return createSuccessfulAuthentication(request, user);
		}
		catch (CookieTheftException cte) {
			cancelCookie(request, response);
			throw cte;
		}
		catch (UsernameNotFoundException noUser) {
			logger.debug("Remember-me login was valid but corresponding user not found.",
					noUser);
		}
		catch (InvalidCookieException invalidCookie) {
			logger.debug("Invalid remember-me cookie: " + invalidCookie.getMessage());
		}
		catch (AccountStatusException statusInvalid) {
			logger.debug("Invalid UserDetails: " + statusInvalid.getMessage());
		}
		catch (RememberMeAuthenticationException e) {
			logger.debug(e.getMessage());
		}

		cancelCookie(request, response);
		return null;
	}
```

1. 获取Cookie
2. 解析Cookie
3. 获取用户凭证
4. 检查用户凭证

# 2. 代码下载

从我的 github 中下载，[https://github.com/longfeizheng/logback](https://github.com/longfeizheng/logback)



