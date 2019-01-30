title: Shiro 源码分析(6) - 用户主体(Subject)
date: 2018-01-07
tag: 
categories: Shiro
permalink: Shiro/xiaoqiyiye/Subject
author: xiaoqiyiye
from_url: https://my.oschina.net/xiaoqiyiye/blog/1619214
wechat_url: 

-------

摘要: 原创出处 https://my.oschina.net/xiaoqiyiye/blog/1619214 「xiaoqiyiye」欢迎转载，保留摘要，谢谢！

- [Subject接口](http://www.iocoder.cn/Shiro/xiaoqiyiye/Subject/)
- [Shiro登录](http://www.iocoder.cn/Shiro/xiaoqiyiye/Subject/)
- [Shiro登出](http://www.iocoder.cn/Shiro/xiaoqiyiye/Subject/)
- [runAs](http://www.iocoder.cn/Shiro/xiaoqiyiye/Subject/)

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

# Subject接口

在Shiro中Subject表示系统进行交互的用户或某一个第三方服务，所有Subject实例都被绑定到(且这是必须的)一个SecurityManager上。Subject接口提供了很多方法，主要包括进行认证的登录登出方法、进行授权判断的方法和Session访问的方法。在Shiro中获取当前运行的Subject要使用SecurityUtils.getSubject()方法来获取一个Subject实例。

下面将Subject接口方法进行分类：

1. 和认证的相关的方法

```Java
// 返回可以鉴定Subject唯一性的身份信息，例如：用户名、用户ID等等
Object getPrincipal();

// 以PrincipalCollection形式返回身份信息
PrincipalCollection getPrincipals();

// 是否已经被认证通过
boolean isAuthenticated();

// 登录认证
void login(AuthenticationToken token) throws AuthenticationException;

// 认证登出 
void logout();
```

1. 和授权相关的方法

```Java
// 是否有指定的权限
boolean isPermitted(String permission);

// 是否有指定的权限
boolean isPermitted(Permission permission);

// 是否有指定的权限
boolean[] isPermitted(String... permissions);

// 是否有指定的权限
boolean[] isPermitted(List<Permission> permissions);

// 是否有所有指定的权限
boolean isPermittedAll(String... permissions);

// 是否有所有指定的权限
boolean isPermittedAll(Collection<Permission> permissions);

// 检测是否有权限，如果没有将抛出异常
void checkPermission(String permission) throws AuthorizationException;

// 检测是否有权限，如果没有将抛出异常
void checkPermission(Permission permission) throws AuthorizationException;

// 检测是否有权限，如果没有将抛出异常
void checkPermissions(String... permissions) throws AuthorizationException;

// 检测是否有权限，如果没有将抛出异常
void checkPermissions(Collection<Permission> permissions) throws AuthorizationException;

// 是否有指定角色
boolean hasRole(String roleIdentifier);

// 是否有指定角色
boolean[] hasRoles(List<String> roleIdentifiers);

// 是否有所有指定角色
boolean hasAllRoles(Collection<String> roleIdentifiers);

// 检测是否有指定角色，如果没有将抛出异常
void checkRole(String roleIdentifier) throws AuthorizationException;

// 检测是否有所有指定角色，如果没有将抛出异常
void checkRoles(Collection<String> roleIdentifiers) throws AuthorizationException;

// 检测是否有所有指定角色，如果没有将抛出异常
void checkRoles(String... roleIdentifiers) throws AuthorizationException;
```

1. 和Session相关方法

```Java
// 获取关联的Session，如果没有将会创建新Session
Session getSession();

// 获取关联的Session，如果Session不存在并且create=false，将不会创建新Session
Session getSession(boolean create);
```

1. runAs相关的方法

```Java
void runAs(PrincipalCollection principals) throws NullPointerException, IllegalStateException;

boolean isRunAs();

PrincipalCollection getPreviousPrincipals();

PrincipalCollection releaseRunAs();
```

在Subject接口中还提供了一个静态内部类Builder，这个类辅助创建Subject实例。在Builder中需要引用SubjectContext和SecurityManager，SubjectContext负责收集Subject的上下文信息，SecurityManger才是真实创建Subject的对象，通过createSubject(SubjectContext subjectContext)方法来创建，SubjectContext和SessionContext类是，我们可以将它看作一个Map对象。创建Subject已经在[Shiro源码分析(1) - Shiro开篇](https://my.oschina.net/u/3409112/blog/1618279)中分析过了，这里不再赘述。

# Shiro登录

在[Shiro源码分析(1) - Shiro开篇](https://my.oschina.net/u/3409112/blog/1618279)中我们已经了解了Subject是如何创建的，这里我们将分析Shiro是如何处理登录和登出的。为了方便大家看明白些，还是贴出之前使用的代码块。

```Java
Subject subject = SecurityUtils.getSubject(); 
UsernamePasswordToken token = new UsernamePasswordToken("zhang", "123");
subject.login(token);
```

在下面的分析中我们以DelegatingSubject类来说明，DelegatingSubject是Subject接口的直接实现类。我们先看看DelegatingSubject的属性和构造方法。

```Java
// 身份凭证集合对象
protected PrincipalCollection principals;
// 是否被认证，表示该Subject是否已经被认证通过
protected boolean authenticated;
// 主机
protected String host;
// 关联的Session
protected Session session;

// 是否创建Session
protected boolean sessionCreationEnabled;

// 安全管理器，DelegatingSubject中所有的操作都是委派给SecurityManager来处理
protected transient SecurityManager securityManager;

public DelegatingSubject(SecurityManager securityManager) {
    this(null, false, null, null, securityManager);
}

public DelegatingSubject(PrincipalCollection principals, boolean authenticated, String host,
                         Session session, SecurityManager securityManager) {
    this(principals, authenticated, host, session, true, securityManager);
}

public DelegatingSubject(PrincipalCollection principals, boolean authenticated, String host,
                         Session session, boolean sessionCreationEnabled, SecurityManager securityManager) {
    // securityManager 是必须存在的，前面已经多次强调
    if (securityManager == null) {
        throw new IllegalArgumentException("SecurityManager argument cannot be null.");
    }
    this.securityManager = securityManager;
    this.principals = principals;
    this.authenticated = authenticated;
    this.host = host;
    if (session != null) {
        // 包装Session为StoppingAwareProxiedSession
        // session销毁时会进行资源释放
        this.session = decorate(session);
    }
    this.sessionCreationEnabled = sessionCreationEnabled;
}
```

在创建好Subject实例后，就可以调用login(AuthenticationToken token)方法操作登录了。具体分析如下：

```Java
public void login(AuthenticationToken token) throws AuthenticationException {
    
    // 清除runAs身份
    clearRunAsIdentitiesInternal();
    
    // 委派给SecurityManager去执行登录，如果登录成功会返回一个
    // 携带有认证成功数据的Subject对象
    Subject subject = securityManager.login(this, token);

    PrincipalCollection principals;

    String host = null;

    // 获取登录后的身份和主机信息
    if (subject instanceof DelegatingSubject) {
        DelegatingSubject delegating = (DelegatingSubject) subject;
        principals = delegating.principals;
        host = delegating.host;
    } else {
        principals = subject.getPrincipals();
    }

    // 如果没有身份，抛出异常
    if (principals == null || principals.isEmpty()) {
        String msg = "Principals returned from securityManager.login( token ) returned a null or " +
                "empty value.  This value must be non null and populated with one or more elements.";
        throw new IllegalStateException(msg);
    }
    
    // 设置身份到当前这个Subject实例中
    this.principals = principals;
    // 标记为已经认证过
    this.authenticated = true;
    
    // 获取主机
    if (token instanceof HostAuthenticationToken) {
        host = ((HostAuthenticationToken) token).getHost();
    }
    if (host != null) {
        this.host = host;
    }
    
    // 获取Session(就算认证成功了，Session也不一定存在)
    // false参数表示在Session不存在的情况下不会主动创建新Session
    Session session = subject.getSession(false);
    if (session != null) {
        this.session = decorate(session);
    } else {
        this.session = null;
    }
}
```

整理一下Subject的登录流程：

- 需要清除runAs身份
- 委派给SecurityManager做登录认证操作
- 将认证成功的Subject信息设置到当前的Subject中去，标记已经被认证。

# Shiro登出

Shiro登出过程很简单，代码如下，不用详细分析。

```Java
public void logout() {
    try {
        // 清除runAs身份
        clearRunAsIdentitiesInternal();
        // 委派给SecurityManager做登出操作
        this.securityManager.logout(this);
    } finally {
        // 重置属性
        this.session = null;
        this.principals = null;
        this.authenticated = false;
    }
}
```

从上面的分析看，登录登出都是由SecurityManager来做的，后续我们会对SecurityManager进行分析。以及对授权的分析也在SecurityManager中进行说明。

# runAs

runAs是指当前用户以一个给定的身份进行认证。给定的身份信息是存放在Session中的，也就是是如果要进行runAs操作，必须开启创建Session参数sessionCreationEnabled=true，否则会抛异常。

```Java
public void runAs(PrincipalCollection principals) {
    // 首先Subject本身需要存在唯一身份
    if (!hasPrincipals()) {
        String msg = "This subject does not yet have an identity.  Assuming the identity of another " +
                "Subject is only allowed for Subjects with an existing identity.  Try logging this subject in " +
                "first, or using the " + Subject.Builder.class.getName() + " to build ad hoc Subject instances " +
                "with identities as necessary.";
        throw new IllegalStateException(msg);
    }
    // 将身份信息存放到Session属性中去
    pushIdentity(principals);
}

public boolean isRunAs() {
    // 判断runAs栈中是否有身份
    List<PrincipalCollection> stack = getRunAsPrincipalsStack();
    return !CollectionUtils.isEmpty(stack);
}

/**
 * 从runAs栈中获取身份信息
 */ 
public PrincipalCollection getPreviousPrincipals() {
    PrincipalCollection previousPrincipals = null;
    List<PrincipalCollection> stack = getRunAsPrincipalsStack();
    int stackSize = stack != null ? stack.size() : 0;
    if (stackSize > 0) {
        if (stackSize == 1) {
            previousPrincipals = this.principals;
        } else {
            //always get the one behind the current:
            assert stack != null;
            previousPrincipals = stack.get(1);
        }
    }
    return previousPrincipals;
}

public PrincipalCollection releaseRunAs() {
    return popIdentity();
}

// 从Session中获取runAs栈
private List<PrincipalCollection> getRunAsPrincipalsStack() {
    Session session = getSession(false);
    if (session != null) {
        return (List<PrincipalCollection>) session.getAttribute(RUN_AS_PRINCIPALS_SESSION_KEY);
    }
    return null;
}

private void clearRunAsIdentities() {
    Session session = getSession(false);
    if (session != null) {
        session.removeAttribute(RUN_AS_PRINCIPALS_SESSION_KEY);
    }
}

private void pushIdentity(PrincipalCollection principals) throws NullPointerException {
    if (CollectionUtils.isEmpty(principals)) {
        String msg = "Specified Subject principals cannot be null or empty for 'run as' functionality.";
        throw new NullPointerException(msg);
    }
    // 获取runAs栈，如果不存在就创建
    List<PrincipalCollection> stack = getRunAsPrincipalsStack();
    if (stack == null) {
        stack = new CopyOnWriteArrayList<PrincipalCollection>();
    }
    // 保存身份到最前面
    stack.add(0, principals);
    // 获取Session，将runAs栈保存到Session中
    Session session = getSession();
    session.setAttribute(RUN_AS_PRINCIPALS_SESSION_KEY, stack);
}

private PrincipalCollection popIdentity() {
    PrincipalCollection popped = null;

    List<PrincipalCollection> stack = getRunAsPrincipalsStack();
    if (!CollectionUtils.isEmpty(stack)) {
        popped = stack.remove(0);
        Session session;
        if (!CollectionUtils.isEmpty(stack)) {
            session = getSession();
            session.setAttribute(RUN_AS_PRINCIPALS_SESSION_KEY, stack);
        } else {
            //stack is empty, remove it from the session:
            clearRunAsIdentities();
        }
    }

    return popped;
}
```