title: 【龙飞】Spring Security 源码分析八：Spring Security 退出
date: 2019-01-18
tag:
categories: Spring Security
permalink: Spring-Security/longfei/logout
author: 龙飞
from_url: http://niocoder.com/2018/01/18/Spring-Security源码分析八-Spring-Security-退出/

-------

摘要: 原创出处 http://niocoder.com/2018/01/18/Spring-Security源码分析八-Spring-Security-退出/ 「龙飞」欢迎转载，保留摘要，谢谢！

- [1. 退出原理](http://www.iocoder.cn/Spring-Security/longfei/logout/)
  - [1.1 退出的实现](http://www.iocoder.cn/Spring-Security/longfei/logout/)
  - [1.2 效果如下](http://www.iocoder.cn/Spring-Security/longfei/logout/)
  - [1.3 源码分析](http://www.iocoder.cn/Spring-Security/longfei/logout/)
- [2. 代码下载](http://www.iocoder.cn/Spring-Security/longfei/logout/)

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

# 1. 退出原理

1. 清除`Cookie`
2. 清除当前用户的`remember-me`记录
3. 使当前`session`失效
4. 清空当前的`SecurityContext`
5. 重定向到登录界面

`Spring Security`的退出请求（默认为`/logout`）由[LogoutFilter](https://longfeizheng.github.io/2018/01/05/Spring-Security%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%BA%8C-Spring-Security%E6%8E%88%E6%9D%83%E8%BF%87%E7%A8%8B/#%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90)过滤器拦截处理。

## 1.1 退出的实现

1. 主页中添加退出链接
```html
<a href="/signOut">退出</a>
```
2. 配置[MerryyouSecurityConfig](https://github.com/longfeizheng/logback/blob/master/src/main/java/cn/merryyou/logback/security/MerryyouSecurityConfig.java#L84)
```java
......
				.and()
                .logout()
                .logoutUrl("/signOut")//自定义退出的地址
                .logoutSuccessUrl("/register")//退出之后跳转到注册页面
                .deleteCookies("JSESSIONID")//删除当前的JSESSIONID
                .and()
......
```

## 1.2 效果如下

[![https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/Spring-Security-logout.gif](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/Spring-Security-logout.gif "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/Spring-Security-logout.gif")](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/Spring-Security-logout.gif "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/Spring-Security-logout.gif")

## 1.3 源码分析

### 1.3.1 LogoutFilter#doFilter

```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {
		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;
		//#1.匹配到/logout请求
		if (requiresLogout(request, response)) {
			Authentication auth = SecurityContextHolder.getContext().getAuthentication();

			if (logger.isDebugEnabled()) {
				logger.debug("Logging out user '" + auth
						+ "' and transferring to logout destination");
			}
			//#2.处理1-4步
			this.handler.logout(request, response, auth);
			//#3.重定向到注册界面
			logoutSuccessHandler.onLogoutSuccess(request, response, auth);

			return;
		}

		chain.doFilter(request, response);
	}
```
1. 匹配当前拦截的请求
2. 处理 清空`Cookie`、`remember-me`、`session`和`SecurityContext`
3. 重定向到登录界面

### 1.3.2 handler

[![https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/Spring-Security-LogoutFilter.png](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/Spring-Security-LogoutFilter.png "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/Spring-Security-LogoutFilter.png")](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/Spring-Security-LogoutFilter.png "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/Spring-Security-LogoutFilter.png")

1. `CookieClearingLogoutHandler`清空`Cookie`
2. `PersistentTokenBasedRememberMeServices`清空`remember-me`
3. `SecurityContextLogoutHandler` 使当前`session`无效,清空当前的`SecurityContext`

#### 1.3.2.1 CookieClearingLogoutHandler#logout

```java
public void logout(HttpServletRequest request, HttpServletResponse response,
			Authentication authentication) {
		for (String cookieName : cookiesToClear) {
			//# 1.Cookie置为null
			Cookie cookie = new Cookie(cookieName, null);
			String cookiePath = request.getContextPath();
			if (!StringUtils.hasLength(cookiePath)) {
				cookiePath = "/";
			}
			cookie.setPath(cookiePath);
			cookie.setMaxAge(0);
			response.addCookie(cookie);
		}
	}
```

1. `Cookie`置为null

### 1.3.3 PersistentTokenBasedRememberMeServices#logout

```java
public void logout(HttpServletRequest request, HttpServletResponse response,
			Authentication authentication) {
		super.logout(request, response, authentication);

		if (authentication != null) {
			//#1.清空persistent_logins表中记录
			tokenRepository.removeUserTokens(authentication.getName());
		}
	}
```

1. 清空persistent_logins表中记录

### 1.3.4 SecurityContextLogoutHandler#logout

````java
public void logout(HttpServletRequest request, HttpServletResponse response,
			Authentication authentication) {
		Assert.notNull(request, "HttpServletRequest required");
		if (invalidateHttpSession) {
			HttpSession session = request.getSession(false);
			if (session != null) {
				logger.debug("Invalidating session: " + session.getId());
				//#1.使当前session失效
				session.invalidate();
			}
		}

		if (clearAuthentication) {
			SecurityContext context = SecurityContextHolder.getContext();
			//#2.清空当前的`SecurityContext`
			context.setAuthentication(null);
		}

		SecurityContextHolder.clearContext();
	}
````

1. 使当前session失效
2. 清空当前的`SecurityContext`

### 1.3.5 AbstractAuthenticationTargetUrlRequestHandler#handle

```java
	protected void handle(HttpServletRequest request, HttpServletResponse response,
			Authentication authentication) throws IOException, ServletException {
		//#1.获取配置的跳转地址
		String targetUrl = determineTargetUrl(request, response);

		if (response.isCommitted()) {
			logger.debug("Response has already been committed. Unable to redirect to "
					+ targetUrl);
			return;
		}
		//#2.跳转请求
		redirectStrategy.sendRedirect(request, response, targetUrl);
	}
```

1. 获取配置的跳转地址
2. 跳转请求


# 2. 代码下载

从我的 github 中下载，[https://github.com/longfeizheng/logback](https://github.com/longfeizheng/logback)



