title: 【龙飞】Spring Security 源码分析六：Spring Social 社交登录源码解析
date: 2019-01-16
tag:
categories: Spring Security
permalink: Spring-Security/longfei/Spring-Social-login-source-parsing
author: 龙飞
from_url: http://niocoder.com/2018/01/16/Spring-Security源码分析六-Spring-Social社交登录源码解析/

-------

摘要: 原创出处 http://niocoder.com/2018/01/16/Spring-Security源码分析六-Spring-Social社交登录源码解析/ 「龙飞」欢迎转载，保留摘要，谢谢！

- [1. 引言](http://www.iocoder.cn/Spring-Security/longfei/Spring-Social-login-source-parsing/)
  - [1.1 Spring Social 基本原理](http://www.iocoder.cn/Spring-Security/longfei/Spring-Social-login-source-parsing/)
  - [1.2 时序图](http://www.iocoder.cn/Spring-Security/longfei/Spring-Social-login-source-parsing/)
- [2. 总结](http://www.iocoder.cn/Spring-Security/longfei/Spring-Social-login-source-parsing/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

> 在[Spring Security源码分析三：Spring Social实现QQ社交登录](https://longfeizheng.github.io/2018/01/09/Spring-Security%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B8%89-Spring-Social%E7%A4%BE%E4%BA%A4%E7%99%BB%E5%BD%95%E8%BF%87%E7%A8%8B/)和[Spring Security源码分析四：Spring Social实现微信社交登录](https://longfeizheng.github.io/2018/01/12/Spring-Security%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E5%9B%9B-Spring-Social%E7%A4%BE%E4%BA%A4%E7%99%BB%E5%BD%95%E8%BF%87%E7%A8%8B/)这两章中，我们使用`Spring Social`已经实现了国内最常用的`QQ`和`微信`社交登录。本章我们来简单分析一下`Spring Social`在社交登录的过程中做了哪些事情？（`微博`社交登录也已经实现，由于已经连续两篇介绍社交登录，所以不在单开一章节描述）

# 1. 引言

OAuth2是一种授权协议，简单理解就是它可以让用户在不将用户名密码交给第三方应用的情况下，第三方应用有权访问用户存在服务提供商上面的数据。


## 1.1 Spring Social 基本原理

[![http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Social-Sequence.png](http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Social-Sequence.png "http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Social-Sequence.png")](http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Social-Sequence.png "http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Social-Sequence.png")

1. 访问第三方应用
2. 将用户请求导向服务提供商
3. 用户同意授权
4. 携带授权码返回第三方莹莹
5. 第三方应用携带授权码到服务提供商申请令牌
6. 服务提供商返回令牌
7. 获取用户基本信息
8. 根据用户信息构建`Authentication`放入`SecurityContext中`
如果在`SecurityContext`中放入一个已经认证过的`Authentication`实例，那么对于`Spring Security`来说，已经成功登录


`Spring Social`就是为我们将`OAuth2`认证流程封装到`SocialAuthenticationFilter`过滤器中，并根据返回的用户信息构建`Authentication`。然后使用`Spring Security`的[验证逻辑](https://longfeizheng.github.io/2018/01/02/Spring-Security%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B8%80-Spring-Security%E8%AE%A4%E8%AF%81%E8%BF%87%E7%A8%8B/#%E9%AA%8C%E8%AF%81%E9%80%BB%E8%BE%91)从而实现使用社交登录。

启动[logback](https://github.com/longfeizheng/logback)断点调试；

[![http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Security-chain.png](http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Security-chain.png "http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Security-chain.png")](http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Security-chain.png "http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Security-chain.png")

1. `ValidateCodeFilter`校验验证码过滤器
2. `SocialAuthenticationFilter`社交登录过滤器
3. `UsernamePasswordAuthenticationFilter`用户名密码登录过滤器
4. `SmsCodeAuthenticationFilter`短信登录过滤器
5. `AnonymousAuthenticationFilter`前面过滤器都没校验时匿名验证的过滤器
6. `ExceptionTranslationFilter`处理`FilterSecurityInterceptor`授权失败时的过滤器
7. `FilterSecurityInterceptor`授权过滤器

本章我们主要讲解`SocialAuthenticationFilter`

### 1.1.1 SocialAuthenticationFilter

```java
public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
		//#1.判断用户是否允许授权
		if (detectRejection(request)) {
			if (logger.isDebugEnabled()) {
				logger.debug("A rejection was detected. Failing authentication.");
			}
			throw new SocialAuthenticationException("Authentication failed because user rejected authorization.");
		}
		
		Authentication auth = null;
		//#2.获取所有的社交配置providerId(本项目中三个：qq,weixin,weibo)
		Set<String> authProviders = authServiceLocator.registeredAuthenticationProviderIds();
		//#3.根据请求获取当前的是那种类型的社交登录
		String authProviderId = getRequestedProviderId(request);
		//#4.判断是否系统中是否配置当前社交providerId
		if (!authProviders.isEmpty() && authProviderId != null && authProviders.contains(authProviderId)) {
			//#5.获取当前社交的处理类即OAuth2AuthenticationService用于获取Authentication
			SocialAuthenticationService<?> authService = authServiceLocator.getAuthenticationService(authProviderId);
			//#6.获取SocialAuthenticationToken
			auth = attemptAuthService(authService, request, response);
			if (auth == null) {
				throw new AuthenticationServiceException("authentication failed");
			}
		}
		return auth;
	}
	
	private Authentication attemptAuthService(final SocialAuthenticationService<?> authService, final HttpServletRequest request, HttpServletResponse response) 
			throws SocialAuthenticationRedirectException, AuthenticationException {
		//获取SocialAuthenticationToken
		final SocialAuthenticationToken token = authService.getAuthToken(request, response);
		if (token == null) return null;
		
		Assert.notNull(token.getConnection());
		//#7.从SecurityContext获取Authentication判断是否认证
		Authentication auth = getAuthentication();
		if (auth == null || !auth.isAuthenticated()) {
			//#8.进行认证
			return doAuthentication(authService, request, token);
		} else {
			//#9.返回当前的登录账户的一些信息
			addConnection(authService, request, token, auth);
			return null;
		}		
	}
	
```

1. 判断用户是否允许授权
2. 获取系统的允许的社交登录配置信息
3. 获取当前的社交登录信息
4. 判断当前的信息是否存在系统配置中
5. 获取处理社交的`OAuth2AuthenticationService`（用于获取`SocialAuthenticationToken`）
6. 从`SecurityContext`获取`Authentication`判断是否授权

### 1.1.2 OAuth2AuthenticationService#getAuthToken

```java
public SocialAuthenticationToken getAuthToken(HttpServletRequest request, HttpServletResponse response) throws SocialAuthenticationRedirectException {
		//#1. 获取code
		String code = request.getParameter("code");
		//#2. 判断code值
		if (!StringUtils.hasText(code)) {
			//#3.如果code不存在则抛出SocialAuthenticationRedirectException
			OAuth2Parameters params =  new OAuth2Parameters();
			params.setRedirectUri(buildReturnToUrl(request));
			setScope(request, params);
			params.add("state", generateState(connectionFactory, request));
			addCustomParameters(params);
			throw new SocialAuthenticationRedirectException(getConnectionFactory().getOAuthOperations().buildAuthenticateUrl(params));
		} else if (StringUtils.hasText(code)) {
			try {
				//#4.如果code存在则根据code获得access_token
				String returnToUrl = buildReturnToUrl(request);
				AccessGrant accessGrant = getConnectionFactory().getOAuthOperations().exchangeForAccess(code, returnToUrl, null);
				// TODO avoid API call if possible (auth using token would be fine)
				//#5.用access_token获取用户的信息并返回spring Social标准信息模型
				Connection<S> connection = getConnectionFactory().createConnection(accessGrant);
				//#6.使用返回的用户信息构建SocialAuthenticationToken
				return new SocialAuthenticationToken(connection, null);
			} catch (RestClientException e) {
				logger.debug("failed to exchange for access", e);
				return null;
			}
		} else {
			return null;
		}
	}
```

1. 获取`code` 
2. 判断当前`code`是否存在值
3. 如果不存在则将用户导向授权的地址
4. 如果存在则根据`code`获取`access_token`
5. 根据`access_token`返回用户信息（该信息为`Spring Social`标准信息模型）
6. 使用用户返回的信息构建`SocialAuthenticationToken`

### 1.1.3 SocialAuthenticationFilter#doAuthentication

```java
private Authentication doAuthentication(SocialAuthenticationService<?> authService, HttpServletRequest request, SocialAuthenticationToken token) {
		try {
			if (!authService.getConnectionCardinality().isAuthenticatePossible()) return null;
			token.setDetails(authenticationDetailsSource.buildDetails(request));
			//#重点熟悉的AuhenticationManage
			Authentication success = getAuthenticationManager().authenticate(token);
			Assert.isInstanceOf(SocialUserDetails.class, success.getPrincipal(), "unexpected principle type");
			updateConnections(authService, token, success);			
			return success;
		} catch (BadCredentialsException e) {
			// connection unknown, register new user?
			if (signupUrl != null) {
				// store ConnectionData in session and redirect to register page
				sessionStrategy.setAttribute(new ServletWebRequest(request), ProviderSignInAttempt.SESSION_ATTRIBUTE, new ProviderSignInAttempt(token.getConnection()));
				throw new SocialAuthenticationRedirectException(buildSignupUrl(request));
			}
			throw e;
		}
	}
```

### 1.1.4 SocialAuthenticationProvider#authenticate

```java
public Authentication authenticate(Authentication authentication) throws AuthenticationException {
		//#1.一些判断信息
		Assert.isInstanceOf(SocialAuthenticationToken.class, authentication, "unsupported authentication type");
		Assert.isTrue(!authentication.isAuthenticated(), "already authenticated");
		SocialAuthenticationToken authToken = (SocialAuthenticationToken) authentication;
		//#2.从SocialAuthenticationToken中获取providerId（表示当前是那个第三方登录）
		String providerId = authToken.getProviderId();
		//#3.从SocialAuthenticationToken中获取获取用户信息 即ApiAdapter设置的用户信息
		Connection<?> connection = authToken.getConnection();
		//#4.从UserConnection表中查询数据
		String userId = toUserId(connection);
		//#5.如果不存在抛出BadCredentialsException异常
		if (userId == null) {
			throw new BadCredentialsException("Unknown access token");
		}
		//#6.调用我们自定义的MyUserDetailsService查询
		UserDetails userDetails = userDetailsService.loadUserByUserId(userId);
		if (userDetails == null) {
			throw new UsernameNotFoundException("Unknown connected account id");
		}
		//#7.返回已经认证的SocialAuthenticationToken
		return new SocialAuthenticationToken(connection, userDetails, authToken.getProviderAccountData(), getAuthorities(providerId, userDetails));
	}
```
1. 从SocialAuthenticationToken中获取providerId（表示当前是那个第三方登录）
2. 从SocialAuthenticationToken中获取获取用户信息 即ApiAdapter设置的用户信息
3. 从UserConnection表中查询数据
4. 调用我们自定义的MyUserDetailsService查询
5. 都正常之后返回已经认证的SocialAuthenticationToken
UserConnection表中是如何添加添加数据的？

### 1.1.5 JdbcUsersConnectionRepository#findUserIdsWithConnection

```java
public List<String> findUserIdsWithConnection(Connection<?> connection) {
		ConnectionKey key = connection.getKey();
		List<String> localUserIds = jdbcTemplate.queryForList("select userId from " + tablePrefix + "UserConnection where providerId = ? and providerUserId = ?", String.class, key.getProviderId(), key.getProviderUserId());		
		//# 重点conncetionSignUp
		if (localUserIds.size() == 0 && connectionSignUp != null) {
			String newUserId = connectionSignUp.execute(connection);
			if (newUserId != null)
			{
				createConnectionRepository(newUserId).addConnection(connection);
				return Arrays.asList(newUserId);
			}
		}
		return localUserIds;
	}
```
因此我们自定义`MyConnectionSignUp`实现`ConnectionSignUp`接口后，`Spring Social`会插入数据后返回`userId`

```java
@Component
public class MyConnectionSignUp implements ConnectionSignUp {
    @Override
    public String execute(Connection<?> connection) {
        //根据社交用户信息，默认创建用户并返回用户唯一标识
        return connection.getDisplayName();
    }
}
```

## 1.2 时序图

[![http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Social-Sequence%20Diagram0.png](http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Social-Sequence%20Diagram0.png "http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Social-Sequence%20Diagram0.png")](http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Social-Sequence%20Diagram0.png "http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Social-Sequence%20Diagram0.png")

至于`OAuth2AuthenticationService`中获取`code`和`AccessToken`,`Spring Social`已经我们提供了基本的实现。开发中，根据不通的服务提供商提供不通的实现，具体可参考以下类图，代码可参考[logback](https://github.com/longfeizheng/logback)项目`social`包下面的类。
[![http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Social-Class%20Diagram0.png](http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Social-Class%20Diagram0.png "http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Social-Class%20Diagram0.png")](http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Social-Class%20Diagram0.png "http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/Spring-Social-Class%20Diagram0.png")

# 2. 总结

以上便是使用`Spring Social`实现社交登录的核心类，其实和用户名密码登录，短信登录原理一样.都有`Authentication`，和实现认证的`AuthenticationProvider`。

