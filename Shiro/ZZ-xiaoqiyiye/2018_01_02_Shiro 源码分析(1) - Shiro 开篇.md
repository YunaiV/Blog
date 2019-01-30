title: Shiro 源码分析(1) - Shiro 开篇
date: 2018-01-02
tag: 
categories: Shiro
permalink: Shiro/xiaoqiyiye/intro
author: xiaoqiyiye
from_url: https://my.oschina.net/xiaoqiyiye/blog/1618279
wechat_url: 

-------

摘要: 原创出处 https://my.oschina.net/xiaoqiyiye/blog/1618279 「xiaoqiyiye」欢迎转载，保留摘要，谢谢！

- [简介](http://www.iocoder.cn/Shiro/xiaoqiyiye/intro/)
- [SecurityUtils分析](http://www.iocoder.cn/Shiro/xiaoqiyiye/intro/)
  - [1. 获取SecurityManager](http://www.iocoder.cn/Shiro/xiaoqiyiye/intro/)
  - [2. 获取Subject](http://www.iocoder.cn/Shiro/xiaoqiyiye/intro/)
- [总结](http://www.iocoder.cn/Shiro/xiaoqiyiye/intro/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

```
本文在于分析Shiro源码，对于新学习的朋友可以参考
[开涛博客](http://jinnianshilongnian.iteye.com/blog/2018398)进行学习。
```

# 简介

- SecurityManager：安全管理器，Shiro最核心组件。Shiro通过SecurityManager来管理内部组件实例，并通过它来提供安全管理的各种服务。
- Authenticator：认证器，认证AuthenticationToken是否有效。
- Authorizer：授权器，处理角色和权限。
- SessionManager：Session管理器，管理Session。
- Subject：当前操作主体，表示当前操作用户。
- SubjectContext：Subject上下文数据对象。
- AuthenticationToken：认证的token信息(用户名、密码等)。
- ThreadContext：线程上下文对象，负责绑定对象到当前线程。

在学习和使用Shiro过程中，我们都知道SecurityManager接口在Shiro中是最为核心的接口。我们就沿着这个接口进行分析。

下面的代码是SecurityManager接口的定义：

```Java
public interface SecurityManager extends Authenticator, Authorizer, SessionManager {

    /**
     * 登录
     */
    Subject login(Subject subject, AuthenticationToken authenticationToken) throws AuthenticationException;

    /**
     * 登出
     */
    void logout(Subject subject);

    /**
     * 创建Subject
     */
    Subject createSubject(SubjectContext context);

}
```

在SecurityManager 中定义了三个方法，分别是登录、登出和创建Subject。通常我们在使用的时候是这样使用的。首先创建Subject对象，然后通过调用login方法传入认证信息token对登录进行认证。

```Java
Subject subject = SecurityUtils.getSubject();
UsernamePasswordToken token = new UsernamePasswordToken("zhang", "123");
subject.login(token);
```

# SecurityUtils分析

在Shiro中提供了一个方便使用的工具类SecurityUtils，SecurityUtils核心功能是获取SecurityManager以及Subject。这两个接口是Shiro提供的外围接口，供开发时使用。

在SecurityUtils使用static定义SecurityManager，也就是说SecurityManager对象在应用中是单一存在的。

```Java
private static SecurityManager securityManager;
```

## 1. 获取SecurityManager

首先从ThreadContext中获取，如果没有，则从SecurityUtils属性securityManager中获取。一定要存在一个SecurityManager实例对象，否则抛异常。

```Java
public static SecurityManager getSecurityManager() throws UnavailableSecurityManagerException {
	SecurityManager securityManager = ThreadContext.getSecurityManager();
	if (securityManager == null) {
		securityManager = SecurityUtils.securityManager;
	}
	if (securityManager == null) {
		String msg = "No SecurityManager accessible to the calling code, either bound to the " +
				ThreadContext.class.getName() + " or as a vm static singleton.  This is an invalid application " +
				"configuration.";
		throw new UnavailableSecurityManagerException(msg);
	}
	return securityManager;
}
```

## 2. 获取Subject

首先从ThreadContext中获取，如果不存在，则创建新的Subject，再存放到ThreadContext中，以便下次可以获取。

```Java
public static Subject getSubject() {
	Subject subject = ThreadContext.getSubject();
	if (subject == null) {
		subject = (new Subject.Builder()).buildSubject();
		ThreadContext.bind(subject);
	}
	return subject;
}
```

在上面的代码中重要是通过 Subject.Builder类提供的buildSubject()方法来创建Subject。在创建Subject时同时还创建了SubjectContext对象，也就是说Subject和SubjectContext是一一对应的。下面的代码是Subject.Builder类的构造方法。

```Java
public Builder(SecurityManager securityManager) {
	if (securityManager == null) {
		throw new NullPointerException("SecurityManager method argument cannot be null.");
	}
	this.securityManager = securityManager;
	// 创建了SubjectContext实例对象
	this.subjectContext = newSubjectContextInstance();
	if (this.subjectContext == null) {
		throw new IllegalStateException("Subject instance returned from 'newSubjectContextInstance' " +
				"cannot be null.");
	}
	this.subjectContext.setSecurityManager(securityManager);
}
```

而buildSubject()方法则实际上是调用SecurityManager接口中的createSubject(SubjectContext subjectContext)方法。

```Java
public Subject buildSubject() {
	return this.securityManager.createSubject(this.subjectContext);
}
```

# 总结

本篇主要通过SecurityUtils.getSubject()对SecurityManager接口中的createSubject(SubjectContext subjectContext)方法进行了详细的分析。另外两个方法我们在分析Subject时做详细分析。

另外，我们会发现SecurityManager继承了 Authenticator, Authorizer, SessionManager三个接口，这样才能实现SecurityManager提供安全管理的各种服务。在接下来的文章中会对Authenticator, Authorizer, SessionManager分别进行分析，这样我们对SecurityManager基本上就掌握了。