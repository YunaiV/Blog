title: 【龙飞】Spring Security 源码分析九：Spring Security Session 管理
date: 2019-01-19
tag:
categories: Spring Security
permalink: Spring-Security/longfei/Session-Manager
author: 龙飞
from_url: http://niocoder.com/2018/01/19/Spring-Security源码分析九-Spring-Security-Session管理/

-------

摘要: 原创出处 http://niocoder.com/2018/01/19/Spring-Security源码分析九-Spring-Security-Session管理/ 「龙飞」欢迎转载，保留摘要，谢谢！

- [1. Session管理](http://www.iocoder.cn/Spring-Security/longfei/Session-Manager/)
  - [1.1 Session超时](http://www.iocoder.cn/Spring-Security/longfei/Session-Manager/)
  - [1.2 Session的并发策略](http://www.iocoder.cn/Spring-Security/longfei/Session-Manager/)
  - [1.3 集群环境Session处理](http://www.iocoder.cn/Spring-Security/longfei/Session-Manager/)
- [2. 代码下载](http://www.iocoder.cn/Spring-Security/longfei/Session-Manager/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

> Session:在计算机中，尤其是在网络应用中，称为“会话控制”。Session 对象存储特定用户会话所需的属性及配置信息。这样，当用户在应用程序的 Web 页之间跳转时，存储在 Session 对象中的变量将不会丢失，而是在整个用户会话中一直存在下去。当用户请求来自应用程序的 Web 页时，如果该用户还没有会话，则 Web 服务器将自动创建一个 Session 对象。当会话过期或被放弃后，服务器将终止该会话。Session 对象最常见的一个用法就是存储用户的首选项。

# 1. Session管理

本文主要描述在 `Spring Security `下 `Session`的以下三种管理，

1. `Session`超时时间
2. `Session`的并发策略
3. 集群环境`Session`处理

## 1.1 Session超时

1. `application.yml`配置超时时间
```java
server:
  port: 80
  session:
    timeout: 60
```
2. 配置[MerryyouSecurityConfig](https://github.com/longfeizheng/logback/blob/master/src/main/java/cn/merryyou/logback/security/MerryyouSecurityConfig.java#L77)
```java
http.
......
	       .sessionManagement()
            .invalidSessionUrl("/session/invalid")//session失效跳转的链接
.....
```
3. `Cotroller`中`/session/invalid`
```java
@GetMapping("/session/invalid")
    @ResponseStatus(code = HttpStatus.UNAUTHORIZED)
    public Result<String> sessionInvalid() {
        return ResultUtil.error(HttpStatus.UNAUTHORIZED.value(), "session失效");
    }
```

效果如下:
[![https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/Spring-Security-sessionTimeout.gif](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/Spring-Security-sessionTimeout.gif "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/Spring-Security-sessionTimeout.gif")](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/Spring-Security-sessionTimeout.gif "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/Spring-Security-sessionTimeout.gif")

## 1.2 Session的并发策略

1. 配置[MerryyouSecurityConfig](https://github.com/longfeizheng/logback/blob/master/src/main/java/cn/merryyou/logback/security/MerryyouSecurityConfig.java#L79)
```java
http.
......
	       .maximumSessions(1)//最大session并发数量1
           .maxSessionsPreventsLogin(false)//false之后登录踢掉之前登录,true则不允许之后登录
           .expiredSessionStrategy(new MerryyounExpiredSessionStrategy())//登录被踢掉时的自定义操作
.....
```
2. `MerryyounExpiredSessionStrategy`
```java
@Slf4j
public class MerryyounExpiredSessionStrategy implements SessionInformationExpiredStrategy {
    @Override
    public void onExpiredSessionDetected(SessionInformationExpiredEvent eventØ) throws IOException, ServletException {
        eventØ.getResponse().setContentType("application/json;charset=UTF-8");
        eventØ.getResponse().getWriter().write("并发登录!");
    }
}
```

效果如下：
[![https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/Spring-Security-session01.gif](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/Spring-Security-session01.gif "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/Spring-Security-session01.gif")](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/Spring-Security-session01.gif "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/Spring-Security-session01.gif")

当`maxSessionsPreventsLogin(true)`可参考：[Spring-Security](https://github.com/spring-projects/spring-security/issues/3078)和[security-oauth2](https://github.com/longfeizheng/security-oauth2)

## 1.3 集群环境Session处理


1. 添加spring-session-data-redis依赖


```xml
<dependency>
			<groupId>org.springframework.session</groupId>
			<artifactId>spring-session-data-redis</artifactId>
			<version>1.3.1.RELEASE</version>
		</dependency>
```

2. 配置Spring-session存储策略


```xml
spring:
  redis:
    host: localhost
    port: 6379
  session:
    store-type: redis
```

3. 测试`8080`和`8081`端口分别启动项目


```xml
java -jar spring-security.jar --server.port=8080
java -jar spring-security.jar --server.port=8081
```
效果如下：

[![https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/Spring-Security-session02.gif](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/Spring-Security-session02.gif "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/Spring-Security-session02.gif")](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/Spring-Security-session02.gif "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/Spring-Security-session02.gif")

关于更多Spring Session可参考：[程序猿DD](http://blog.didispace.com/tags/Spring-Session/)

# 2. 代码下载

从我的 github 中下载，[https://github.com/longfeizheng/logback](https://github.com/longfeizheng/logback)



