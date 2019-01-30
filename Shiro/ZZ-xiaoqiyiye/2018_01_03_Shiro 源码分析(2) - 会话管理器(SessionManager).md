title: Shiro 源码分析(2) - 会话管理器(SessionManager)
date: 2018-01-03
tag: 
categories: Shiro
permalink: Shiro/xiaoqiyiye/SessionManager
author: xiaoqiyiye
from_url: https://my.oschina.net/xiaoqiyiye/blog/1618313
wechat_url: 

-------

摘要: 原创出处 https://my.oschina.net/xiaoqiyiye/blog/1618313 「xiaoqiyiye」欢迎转载，保留摘要，谢谢！

- [Session接口定义](http://www.iocoder.cn/Shiro/xiaoqiyiye/SessionManager/)
  - [SimpleSession实现](http://www.iocoder.cn/Shiro/xiaoqiyiye/SessionManager/)
  - [DelegatingSession实现](http://www.iocoder.cn/Shiro/xiaoqiyiye/SessionManager/)
- [SessionManager分析](http://www.iocoder.cn/Shiro/xiaoqiyiye/SessionManager/)
  - [AbstractNativeSessionManager分析](http://www.iocoder.cn/Shiro/xiaoqiyiye/SessionManager/)
  - [AbstractValidatingSessionManager分析](http://www.iocoder.cn/Shiro/xiaoqiyiye/SessionManager/)
  - [DefaultSessionManager分析](http://www.iocoder.cn/Shiro/xiaoqiyiye/SessionManager/)
- [SessionDAO CRUD操作](http://www.iocoder.cn/Shiro/xiaoqiyiye/SessionManager/)
  - [CachingSessionDAO分析](http://www.iocoder.cn/Shiro/xiaoqiyiye/SessionManager/)
- [总结](http://www.iocoder.cn/Shiro/xiaoqiyiye/SessionManager/)

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

本文对Shiro中的SessionManager进行分析，SessionManager用于管理Shiro中的Session信息。Session也就是我们通常说的会话，会话是用户在使用应用程序一段时间内携带的数据。传统的会话一般是基于Web容器(如:Tomcat、EJB环境等)。Shiro提供的Session可以在任何环境中使用，不再依赖于其他容器。

Shiro还提供了一些其他的特性：

- 基于POJO/J2SE：Session和SessionManager都是基于接口实现的，可以通过POJO来进行实现。可以使用任务JavaBean兼容的格式(如：Json，YAML，spring xml或类似机制)来轻松配置所有会话组件。能够根据需要完全地扩展会话组件。
- 会话存储：因为Shiro的Session对象是基于POJO的，所以会话数据可以容易地存储在任何数据源中。 这允许您精确定制应用程序的会话数据所在的位置，例如文件系统，企业缓存，关系数据库或专有数据存储。
- 简单强大的集群：Shiro的Session可以通过缓存来实现集群功能，这样的Session并不会依赖于Web容器，不需要对Web容器进行特殊的配置。
- 事件监听：事件监听器允许在会话生命周期中接受生命周期事件，可以监听这些事件并对自定义应用程序行为做出反应。例如，在会话过期时更新用户记录。
- 主机地址保留：会记录Session创建时的IP地址。
- 会话过期：会话由于预期的不活动而过期，但如果需要，可以通过touch（）方法延长活动以保持活动状态。

# Session接口定义

Shiro提供的Session和Servlet中的Session其实是一样的作用，只是Shiro中的Sesion不需要再依赖于WEB容器存在。下面是Session接口提供的方法：

```Java
/**
 * 返回表示Session的唯一ID
 */
Serializable getId();

/**
 * 返回Session的开始时间
 */
Date getStartTimestamp();

/**
 * 返回最近使用的时间
 */
Date getLastAccessTime();

/**
 * 返回还有多久Session过期(毫秒)
 */
long getTimeout() throws InvalidSessionException;

/**
 * 设置超时时间(毫秒)
 */
void setTimeout(long maxIdleTimeInMillis) throws InvalidSessionException;    

/**
 * 返回Session创建时的主机名或地址
 */
String getHost();

/**
 * 更新最近访问时间，确保Session不会过期
 */
void touch() throws InvalidSessionException;

/**
 * 设置Session停止使用，并释放相关资源
 */
void stop() throws InvalidSessionException;

/**
 * 返回该Session存储的所有属性键
 */
Collection<Object> getAttributeKeys() throws InvalidSessionException;

/**
 * 获取Session属性
 */
Object getAttribute(Object key) throws InvalidSessionException;

/**
 * 设置Session属性
 */
void setAttribute(Object key, Object value) throws InvalidSessionException;

/**
 * 删除Session属性
 */
Object removeAttribute(Object key) throws InvalidSessionException;
```

从接口中我们可以看出。Session有一个唯一ID，开始时间，最近活动时间等属性，另外提供了两个相对应的方法touch()和stop()，其他方法就是对属性的操作了。Session接口还是相当简单清晰的，下面我们看看接口的实现类。

Session有一些实现类，包括SimpleSession、HttpServletSession和DelegatingSession等。SimpleSession是Shiro提供的一种简单实现，HttpServletSession是基于Sevlet中Session来实现的，DelegatingSession是一种委托机制，委托给SessionManager来实现。下面，我们主要以SimpleSession和DelegatingSession来分析。

## SimpleSession实现

SimpleSession从Session接口继承过来的方法实现非常简单，就不过多分析了。我们主要分析一下属性，通过属性就可以了解到SimpleSession中功能的实现了。

```Java
// Session唯一ID
private transient Serializable id;
// 开始时间
private transient Date startTimestamp;
// 结束时间
private transient Date stopTimestamp;
// 最近访问时间
private transient Date lastAccessTime;
// 还有多久过期
private transient long timeout;
// 是否过期
private transient boolean expired;
// Session创建时的主机名
private transient String host;
// Session存储的属性值
private transient Map<Object, Object> attributes;
```

## DelegatingSession实现

DelegatingSession是一种委托实现方式，所有的操作都委托给SessionManager接口来实现。我们还是先看有哪些属性。

```Java
// SessionKey就是Session唯一ID的对象形式
private final SessionKey key;

// 委派给NativeSessionManager对象，这是SessionManager接口的一个子接口
private final transient NativeSessionManager sessionManager;
```

很显然，DelegatingSession的委托方式是用过SessionKey从SessionManager中获取相应的Session来进行处理的。在SessionManager中管理着很多Session。

在分析SessionManager前，先说明一下SessionContext这个接口(上一篇中提到过 )。SessionContext表示的是Session创建时的上下文参数，SessionContext有DefaultSessionContext，DefaultWebSessionContext两个实现。但功能都是从MapContext继承来，简单地说SessionContext就是一个Map对象，提供了一个更方便获取具体类型的方法getTypedValue(String key, Class<E> type)。

# SessionManager分析

SessionManager管理着Session的创建、操作以及清除等。SessionManager有一些子接口，包括NativeSessionManager、ValidatingSessionManager和WebSessionManager。每个接口都提供了相关的抽象类AbstractSessionManager、AbstractNativeSessionManager、AbstractValidatingSessionManager。我们还是先看看接口提供了哪些方法。

```Java
public interface SessionManager {

    /**
     * 开始一个新的Session，context提供一些初始化的数据
     */
    Session start(SessionContext context);

    /**
     * 通过SessionKey查找Session
     * 如果存在则被找到，如果不存在则返回null；如果找到的Session无效(被停止或过期)则会抛异常
     */
    Session getSession(SessionKey key) throws SessionException;
}   


public interface NativeSessionManager extends SessionManager {

    /**
     * 返回Session被开启的时间
     */
    Date getStartTimestamp(SessionKey key);

    /**
     * 获取Session最近访问的时间
     */
    Date getLastAccessTime(SessionKey key);

    /**
     * 判断Session是否有效
     */
    boolean isValid(SessionKey key);

    /**
     * 检测Session是否有效，如果无效则抛出异常
     */
    void checkValid(SessionKey key) throws InvalidSessionException;

    /**
     * 返回Session还有多久过期(毫秒)
     * 如果是负数，表示Session不会过期；如果是正数，表示Session在这个时间后就会过期。
     * 如果Session无效调用这个方法会抛异常
     */
    long getTimeout(SessionKey key) throws InvalidSessionException;

    /**
     * 设置过期时间(毫秒)，设置负数表示Session不会过期。
     * 如果Session无效调用这个方法会抛异常
     */
    void setTimeout(SessionKey key, long maxIdleTimeInMillis) throws InvalidSessionException;

    /**
     * 会设置最近访问时间，确保Session没有超时，当然，如果Session已经无效也会抛异常
     */
    void touch(SessionKey key) throws InvalidSessionException;

    /**
     * 返回主机名或Ip
     */
    String getHost(SessionKey key);

    /**
     * 停用Session，释放资源
     */
    void stop(SessionKey key) throws InvalidSessionException;

    /**
     * 返回Session中存储的所有属性键
     */
    Collection<Object> getAttributeKeys(SessionKey sessionKey);

    /**
     * 获取Session中的属性
     */
    Object getAttribute(SessionKey sessionKey, Object attributeKey) throws InvalidSessionException;

    /**
     * 设置Session中的属性
     */
    void setAttribute(SessionKey sessionKey, Object attributeKey, Object value) throws InvalidSessionException;

    /**
     * 删除Session中的属性
     */
    Object removeAttribute(SessionKey sessionKey, Object attributeKey) throws InvalidSessionException;

}


public interface ValidatingSessionManager extends SessionManager {

    /**
     * 为那些有效的Session进行校验，如果Session发现是无效的，将会被更改。
     * 这个方法期望有规律的运行，如1小时一次，一天一次或一星期一次。运行的频率取决于应用的性能，用户活跃数等。
     */
    void validateSessions();
}
```

从上面的接口中可以看出，SessionManager主要负责创建Session和获取Session，NativeSessionManager接口中包含了所有对Session的操作，这些操作方法和Session接口中是一致的，而ValidatingSessionManager接口提供了对Session校验的支持。这些接口从功能上分工很明确。

------

## AbstractNativeSessionManager分析

AbstractNativeSessionManager类对NativeSessionManager接口做了一个整体的结构实现，定型了整个接口的实现基础。我们先列出AbstractNativeSessionManager中主要功能：

- 引用了SessionListen接口来负责对Session状态的监听。
- 提供了创建Session的抽象方法createSession(SessionContext context)和获取Session的抽象方法doGetSession(SessionKey key)，这两个方法都应该从SessionManager接口来实现的。
- 提供了onChange，onStart，onStop，afterStopped钩子方法。

我们先分析SessionManager中的两个方法。start(SessionContext context)和getSession(SessionKey key)。

```Java
public Session start(SessionContext context) {
    // 抽象方法创建Session
    Session session = createSession(context);
    applyGlobalSessionTimeout(session);
    // 钩子方法，子类实现
    onStart(session, context);
    // Session监听器
    notifyStart(session);
    // 使用DelegatingSession来委托SessionManger处理
    return createExposedSession(session, context);
}

protected Session createExposedSession(Session session, SessionContext context) {
	return new DelegatingSession(this, new DefaultSessionKey(session.getId()));
}

public Session getSession(SessionKey key) throws SessionException {
    Session session = lookupSession(key);
    return session != null ? createExposedSession(session, key) : null;
}

private Session lookupSession(SessionKey key) throws SessionException {
    if (key == null) {
        throw new NullPointerException("SessionKey argument cannot be null.");
    }
    // 抽象方法
    return doGetSession(key);
}
```

接下来再看看Session监听器的方法。监听器提供了对Session状态的监听：Session启动，Session停止，Session过期。

```Java
protected void notifyStart(Session session) {
    for (SessionListener listener : this.listeners) {
        listener.onStart(session);
    }
}

protected void notifyStop(Session session) {
    Session forNotification = beforeInvalidNotification(session);
    for (SessionListener listener : this.listeners) {
        listener.onStop(forNotification);
    }
}

protected void notifyExpiration(Session session) {
    Session forNotification = beforeInvalidNotification(session);
    for (SessionListener listener : this.listeners) {
        listener.onExpiration(forNotification);
    }
}
```

最后，我们再看看对NativeSessionManager接口方法的实现，方法很多，基本上实现思路一样。我们以touch和stop来说明。

```Java
public void touch(SessionKey key) throws InvalidSessionException {
    // 获取Session
    Session s = lookupRequiredSession(key);
    // 调用Session自己提供的功能
    s.touch();
    // 调用onChange方法作为变更后的后置处理方法
    onChange(s);
}

public void stop(SessionKey key) throws InvalidSessionException {
    // 获取Session
    Session session = lookupRequiredSession(key);
    try {
        if (log.isDebugEnabled()) {
            log.debug("Stopping session with id [" + session.getId() + "]");
        }
        // 调用Session自己提供的功能
        session.stop();
        // 调用后置处理方法
        onStop(session, key);
        // 通知监听器
        notifyStop(session);
    } finally {
        // 调用停止后的后置方法
        afterStopped(session);
    }
}
```

我们可以给AbstractNativeSessionManager类作一个总结。该类负责管理Session的操作，但操作的具体实现是由Session自己实现的。相当于对Session操作前后做代理(不管是提供钩子方法还是监听Session)。

------

## AbstractValidatingSessionManager分析

AbstractValidatingSessionManager的作用是定期的校验所有有效的Session状态，因为Session可能被停止或过期。AbstractValidatingSessionManager类继承了上面分析的AbstractNativeSessionManager，然后实现了ValidatingSessionManager接口中的validateSessions()方法。我们还是先从属性开始分析，下面是AbstractValidatingSessionManager类中的属性。

```Java
// 标识是否需要校验Sesion
protected boolean sessionValidationSchedulerEnabled;

// 从名称上看，我们就知道这个一个调度器。用来定时调度校验Session
protected SessionValidationScheduler sessionValidationScheduler;

// 调度器调度的时间间隔
protected long sessionValidationInterval;
```

AbstractValidatingSessionManager定义了一些和调度相关的属性，我们先了解一下SessionValidationScheduler接口有哪些方法。SessionValidationScheduler提供了3个方法：

- isEnabled() - 表示是否已经开始了调度作业
- enableSessionValidation() - 开启具体调度的作业内容
- disableSessionValidation() - 停止调度作业

那么，我们应该想想，调度的作业内容是什么？很显然，这个作业内容是校验Session相关的事情，也就是为什么在ValidatingSessionManager接口中提供了validateSessions()方法的原因。

明白了SessionValidationScheduler接口之后，我们在回过来分析AbstractValidatingSessionManager就相当容易了。我们先看看是如何校验的，分析enableSessionValidationIfNecessary()方法。

```Java
/**
 * 这个方法就是判断是否需要校验，如果需要校验就去开启调度作业
 **/
private void enableSessionValidationIfNecessary() {
    SessionValidationScheduler scheduler = getSessionValidationScheduler();
    // 首先，判断是否需要校验Session，如果为false，则永远不会开启校验，根本就不判断scheduler的状态；
    // 其次，如果需要校验Session，会去判断scheduler== null或scheduler没有启动，在这两种情况下都会去开启调度任务
    // 也就是说，如果任务没有开启就去开启，如果已经开启了，就不会在处理。
    if (isSessionValidationSchedulerEnabled() && (scheduler == null || !scheduler.isEnabled())) {
        enableSessionValidation();
    }
}

/**
 * 具体的开启实现在这里
 **/
protected void enableSessionValidation() {
    // 是否设置了scheduler，如果没有就创建一个
    SessionValidationScheduler scheduler = getSessionValidationScheduler();
    if (scheduler == null) {
        // 创建scheduler
        scheduler = createSessionValidationScheduler();
        setSessionValidationScheduler(scheduler);
    }
    // 开启调度作业
    scheduler.enableSessionValidation();
    // 开启作业后的后置处理方法(钩子方法)
    afterSessionValidationEnabled();
}

/**
 * 创建scheduler
 **/
protected SessionValidationScheduler createSessionValidationScheduler() {
    ExecutorServiceSessionValidationScheduler scheduler;
    // 注意：这个的this指的就是ValidatingSessionManager接口啦，前面分析过，调度作业的具体内容时由这个接口来提供的。
    scheduler = new ExecutorServiceSessionValidationScheduler(this);
    scheduler.setInterval(getSessionValidationInterval());
    return scheduler;
}
```

上面我们只是从代码片段上分析了如何开启校验的，现在我们继续跟随AbstractNativeSessionManager的分析。我们说在AbstractNativeSessionManager中已经定型了整个类的基本实现，提供了两个抽象方法createSession(SessionContext context)和doGetSession(SessionKey key)。在AbstractValidatingSessionManager中对这两个方法进行了实现。下面主要分析这两个方法是怎么实现的。

```Java
// 实现createSession方法
protected Session createSession(SessionContext context) throws AuthorizationException {
    // 这就是上面分析过的方法，判断是否需要校验和启动调度作业
    enableSessionValidationIfNecessary();
    // 继续提供抽象方法由子类实现怎么创建Session
    return doCreateSession(context);
}

protected abstract Session doCreateSession(SessionContext initData) throws AuthorizationException;

// 实现doGetSession方法
protected final Session doGetSession(final SessionKey key) throws InvalidSessionException {
    // 这就是上面分析过的方法，判断是否需要校验和启动调度作业
    enableSessionValidationIfNecessary();
    // 继续提供抽象方法有子类实现怎么获取Session
    Session s = retrieveSession(key);
    if (s != null) {
        // 验证Session：
        // 首先，必须是ValidatingSession接口类型的Session
        // 其次，验证Session是否过期
        // 最后，验证Session是否无效
        validate(s, key);
    }
    return s;
}

protected void validate(Session session, SessionKey key) throws InvalidSessionException {
    try {
        doValidate(session);
    } catch (ExpiredSessionException ese) {
        onExpiration(session, ese, key);
        throw ese;
    } catch (InvalidSessionException ise) {
        onInvalidation(session, ise, key);
        throw ise;
    }
}

protected void doValidate(Session session) throws InvalidSessionException {
    if (session instanceof ValidatingSession) {
        ((ValidatingSession) session).validate();
    } else {
        String msg = "The " + getClass().getName() + " implementation only supports validating " +
                "Session implementations of the " + ValidatingSession.class.getName() + " interface.  " +
                "Please either implement this interface in your session implementation or override the " +
                AbstractValidatingSessionManager.class.getName() + ".doValidate(Session) method to perform validation.";
        throw new IllegalStateException(msg);
    }
}
```

从上面的分析可以得出：AbstractValidatingSessionManager类负责校验Session，对于如何创建和获取Session，并不是它的需要处理的任务。另外AbstractValidatingSessionManager还实现了Destroyable接口，表示销毁时应该处理销毁功能。

```Java
protected void disableSessionValidation() {
    // 销毁前置处理方法(钩子方法)
    beforeSessionValidationDisabled();
    SessionValidationScheduler scheduler = getSessionValidationScheduler();
    if (scheduler != null) {
        try {
            // 停止调度任务
            scheduler.disableSessionValidation();
        } catch (Exception e) {
            if (log.isDebugEnabled()) {
                String msg = "Unable to disable SessionValidationScheduler.  Ignoring (shutting down)...";
                log.debug(msg, e);
            }
        }
        // 生命周期管理
        LifecycleUtils.destroy(scheduler);
        // 重置scheduler为null
        setSessionValidationScheduler(null);
    }
}
```

最后，还有一个重要的功能。上面说到过，调度任务处理的内容是什么？也就是我们说的ValidatingSessionManager接口提供的validateSessions()方法。

```Java
public void validateSessions() {

    // 标志无效的Session个数
    int invalidCount = 0;

    // 获取所有有效的Session
    // 这是一个抽象方法。因为目前来说，也不知道Session存放在哪里，要从哪里获取呢？
    Collection<Session> activeSessions = getActiveSessions();
    
    // 遍历判断Session是否有效
    if (activeSessions != null && !activeSessions.isEmpty()) {
        for (Session s : activeSessions) {
            try {
                SessionKey key = new DefaultSessionKey(s.getId());
                validate(s, key);
            } catch (InvalidSessionException e) {
                invalidCount++;
            }
        }
    }

    if (log.isInfoEnabled()) {
        if (invalidCount > 0) {
            msg += "  [" + invalidCount + "] sessions were stopped.";
        } else {
            msg += "  No sessions were stopped.";
        }
        log.info(msg);
    }
}
```

同样，我们也可以给AbstractValidatingSessionManager来总结一下。AbstractValidatingSessionManager类会启动调度作业来校验Session，而调度作业的真正内容是检测每个Session的validate()方法，Session必须是ValidatingSession类型。validate()方法会抛出两个异常StoppedSessionException和ExpiredSessionException异常，这个两个异常都是InvalidSessionException的子类，所以抛出异常的时候会处理onExpiration()或onInvalidation()方法。

------

## DefaultSessionManager分析

上面分析的都是抽象类，抽象类只是提供了一个基础的框架，在Shiro中DefaultSessionManager才是我们所使用的SessionManager接口的具体实现类。在分析AbstractValidatingSessionManager的时候，我们说过对于创建和获取Session，并不是它的职责。Session如何创建的？Session存放在哪里？我们都还不清楚。在DefaultSessionManager中我们将会知道Shiro是如何做的。还是按照习惯的方式，先看看有哪些属性和构造方法。

```Java
// 从名称上看就知道这个创建Session的工厂接口
private SessionFactory sessionFactory;

// DAO是做数据存储的，所以SessionDAO负责Session的CRUD操作
protected SessionDAO sessionDAO;

// 缓存管理器，这个很好理解，Session是频繁使用的对象，需要采用缓存功能
private CacheManager cacheManager;

// 是否删除无效的Session
private boolean deleteInvalidSessions;

// 默认构造方法
public DefaultSessionManager() {
    // 删除无效Session
    this.deleteInvalidSessions = true;
    // SimpleSessionFactory工厂创建SimpleSession实例
    this.sessionFactory = new SimpleSessionFactory();
    // 用内存存储Session
    this.sessionDAO = new MemorySessionDAO();
}
```

关于如何创建Session的，放在后面说。上面我们说了抛出异常的时候会处理onExpiration()或onInvalidation()方法。继续讨论检测Session无效后是怎么处理的，下面的方法就是处理Session的实现，如果Session被检测出被停止或过期就会调用相应的方法处理，返回将无效的Session删除(如果参数配置需要删除的话)或将Session更新。

```Java
@Override
protected void onStop(Session session) {
    if (session instanceof SimpleSession) {
        SimpleSession ss = (SimpleSession) session;
        Date stopTs = ss.getStopTimestamp();
        ss.setLastAccessTime(stopTs);
    }
    onChange(session);
}

@Override
protected void afterStopped(Session session) {
    if (isDeleteInvalidSessions()) {
        delete(session);
    }
}

protected void onExpiration(Session session) {
    if (session instanceof SimpleSession) {
        ((SimpleSession) session).setExpired(true);
    }
    onChange(session);
}

@Override
protected void afterExpired(Session session) {
    if (isDeleteInvalidSessions()) {
        delete(session);
    }
}

protected void onChange(Session session) {
    sessionDAO.update(session);
}
```

------

# SessionDAO CRUD操作

Session的创建是由SessionFactory来实现的。Shiro只提供了SimpleSessionFactory一个实现类，创建SimpleSession实例。如果需要则可以根据业务扩展接口。创建Session后会调用SessionDAO#create(session)方法，将Session存储起来。对Session的CRUD操作都是通过SessionDAO来处理的，SessionDAO负责将Session存储在哪里，怎么存储。下面简单地描述一下SessionDAO接口。

```Java
public interface SessionDAO {

    /**
     * 插入Session(可以是数据库，文件系统，内存，缓存等)
     */
    Serializable create(Session session);

    /**
     * 获取Session
     */
    Session readSession(Serializable sessionId) throws UnknownSessionException;

    /**
     * 更新Session
     */
    void update(Session session) throws UnknownSessionException;

    /**
     * 删除Session
     */
    void delete(Session session);

    /**
     * 返回所有活动的Session
     */
    Collection<Session> getActiveSessions();
}
```

Shiro提供了两种SessionDAO，第一种是MemorySessionDAO，它将Session存储在内存中；第二种是EnterpriseCacheSessionDAO，可以定义将Session存入到缓存中。由于在使用中我们很大可能需要自定义SessionDAO，下面对SessionDAO也展开分析。MemorySessionDAO是以Map作为存储的，很简单不再说明。我们以EnterpriseCacheSessionDAO来分析。EnterpriseCacheSessionDAO类的继承关系是这样的：EnterpriseCacheSessionDAO->CachingSessionDAO->AbstractSessionDAO。在AbstractSessionDAO中提供了SessionIdGenerator类型的属性，用于生成Session唯一ID值。我们需要分析的重点是CachingSessionDAO。

## CachingSessionDAO分析

CachingSessionDAO是一个抽象类，负责对Session进行缓存管理，所有Session都存入到一个缓存中。Shiro提供自己的Cache和CacheManager两个接口。CacheManger负责管理Cache对象实例。下面是CachingSessionDAO的属性，我们可以看出Session就是存放在activeSessions中。

```Java
/**
 * 默认Session缓存名称
 */
public static final String ACTIVE_SESSION_CACHE_NAME = "shiro-activeSessionCache";

/**
 * 缓存管理器
 */
private CacheManager cacheManager;

/**
 * 存储Session的缓存对象
 */
private Cache<Serializable, Session> activeSessions;

/**
 * Session缓存名
 */
private String activeSessionsCacheName = ACTIVE_SESSION_CACHE_NAME;
```

看看在CachingSessionDAO是怎样来创建Session，获取Session，更新Session的。

```Java
// 接口方法，创建Session
public Serializable create(Session session) {
    Serializable sessionId = super.create(session);
    // 相当于对父类方法做了后置缓存处理
    // 实际上就是调用Cache#put()方法，将Session存储
    cache(session, sessionId);
    return sessionId;
}

// 实现父类抽象方法，获取Session
public Session readSession(Serializable sessionId) throws UnknownSessionException {
    // 相当于对父类方法做了前置处理
    // 先从缓存中获取，如果缓存中没有再调用真实方法
    Session s = getCachedSession(sessionId);
    if (s == null) {
        s = super.readSession(sessionId);
    }
    return s;
}

// 修改Session
public void update(Session session) throws UnknownSessionException {
    // 抽象方法，修改Session前置处理
    doUpdate(session);
    // 将Session更新到缓存中
    if (session instanceof ValidatingSession) {
        if (((ValidatingSession) session).isValid()) {
            cache(session, session.getId());
        } else {
            uncache(session);
        }
    } else {
        cache(session, session.getId());
    }
}

// 删除Session
public void delete(Session session) {
    // 从缓存中删除
    uncache(session);
    // 删除后置方法
    doDelete(session);
}
```

# 总结

在本篇中我们了解了Session、SessionManager以及SessionDAO。Session表示用户的会话数据，Session的核心点是状态，Session有无效和活动两种状态。其中，无效又包括被停止和过期状态，我们可以对Session状态进行监听和检测。

另外，就是Session的存储，在Shiro中使用SessionDAO接口来处理Session的存储，Shiro中提供了基于内存和基于缓存的两种方式来存储Session。