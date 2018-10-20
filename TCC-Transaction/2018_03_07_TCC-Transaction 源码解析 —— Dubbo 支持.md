title: TCC-Transaction 源码分析 —— Dubbo 支持
date: 2018-02-28
tags:
categories: TCC-Transaction
permalink: TCC-Transaction/dubbo-support

---

摘要: 原创出处 http://www.iocoder.cn/TCC-Transaction/dubbo-support/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 TCC-Transaction 1.2.3.3 正式版**  

- [1. 概述](http://www.iocoder.cn/TCC-Transaction/dubbo-support/)
- [2. Dubbo 代理](http://www.iocoder.cn/TCC-Transaction/dubbo-support/)
  - [2.1 JavassistProxyFactory](http://www.iocoder.cn/TCC-Transaction/dubbo-support/)
    - [2.1.1 Javassist](http://www.iocoder.cn/TCC-Transaction/dubbo-support/)
    - [2.1.2 TccJavassistProxyFactory](http://www.iocoder.cn/TCC-Transaction/dubbo-support/)
    - [2.1.3 TccProxy & TccClassGenerator](http://www.iocoder.cn/TCC-Transaction/dubbo-support/)
    - [2.1.4 配置 Dubbo Proxy](http://www.iocoder.cn/TCC-Transaction/dubbo-support/)
  - [2.2 JdkProxyFactory](http://www.iocoder.cn/TCC-Transaction/dubbo-support/)
    - [2.2.1 JDK Proxy](http://www.iocoder.cn/TCC-Transaction/dubbo-support/)
    - [2.2.2 TccJdkProxyFactory](http://www.iocoder.cn/TCC-Transaction/dubbo-support/)
    - [2.2.3 TccInvokerInvocationHandler](http://www.iocoder.cn/TCC-Transaction/dubbo-support/)
    - [2.2.4 配置 Dubbo Proxy](http://www.iocoder.cn/TCC-Transaction/dubbo-support/)
- [3. Dubbo 事务上下文编辑器](http://www.iocoder.cn/TCC-Transaction/dubbo-support/)
- [666. 彩蛋](http://www.iocoder.cn/TCC-Transaction/dubbo-support/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2018_05_18.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

---

# 1. 概述

本文分享 **Dubbo 支持**。

TCC-Transaction 通过 Dubbo **隐式传参**的功能，避免自己对业务代码的入侵。可能有同学不太理解为什么说 TCC-Transaction 对业务代码有一定的入侵性，一起来看个代码例子：

```Java
public interface CapitalTradeOrderService {
    String record(TransactionContext transactionContext, CapitalTradeOrderDto tradeOrderDto);
}
```
* 代码来自 `tcc-transaction-http-sample` 。声明远程调用时，增加了参数 TransactionContext。当然你也可以通过自己使用的远程调用框架做一定封装，避免入侵。

如下是对 Dubbo 封装了后，Dubbo Service 方法的例子：

```Java
public interface CapitalTradeOrderService {

    @Compensable
    String record(CapitalTradeOrderDto tradeOrderDto);

}
```

* 代码来自 `http-transaction-dubbo-sample` 。是不是不需要传入参数 TransactionContext。当然，注解是肯定需要的，否则 TCC-Transaction 怎么知道哪些方法是 TCC 方法。

TCC-Transaction 通过 Dubbo Proxy 的机制，实现 `@Compensable` 属性自动生成，增加开发体验，也避免出错。

-------


Dubbo 支持( Maven 项目 `tcc-transaction-dubbo` ) 整体代码结构如下：

![](http://www.iocoder.cn/images/TCC-Transaction/2018_03_07/01.png)

* `proxy`
* `context`

我们分成两个小节分享这两个包实现的功能。

**笔者暂时对 Dubbo 了解的不够深入，如果有错误的地方，还烦请指出，谢谢。**

> 你行好事会因为得到赞赏而愉悦  
> 同理，开源项目贡献者会因为 Star 而更加有动力  
> 为 TCC-Transaction 点赞！[传送门](https://github.com/changmingxie/tcc-transaction)

ps：笔者假设你已经阅读过[《tcc-transaction 官方文档 —— 使用指南1.2.x》](https://github.com/changmingxie/tcc-transaction/wiki/%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%971.2.x)。

# 2. Dubbo 代理

将 Dubbo Service 方法上的**注解** `@Compensable` ，自动生成注解的 `confirmMethod`、`cancelMethod`、`transactionContextEditor` 属性，例子代码如下：

```Java
@Compensable(propagation=Propagation.SUPPORTS, confirmMethod="record", cancelMethod="record", transactionContextEditor=DubboTransactionContextEditor.class)
public String record(RedPacketTradeOrderDto paramRedPacketTradeOrderDto) {
    // ... 省略代码
}
```

* 该代码通过 Javassist 生成的 Proxy 代码的示例。
* `propagation=Propagation.SUPPORTS` ：支持当前事务，如果当前没有事务，就以非事务方式执行。**为什么不使用 REQUIRED** ？如果使用 REQUIRED 事务传播级别，事务恢复重试时，会发起新的事务。
* `confirmMethod`、`cancelMethod` 使用和 try 方法**相同方法名**：**本地发起**远程服务 TCC confirm / cancel 阶段，调用相同方法进行事务的提交或回滚。远程服务的 CompensableTransactionInterceptor 会根据事务的状态是 CONFIRMING / CANCELLING 来调用对应方法。
    * ![](http://www.iocoder.cn/images/TCC-Transaction/2018_03_07/02.png) 
* `transactionContextEditor=DubboTransactionContextEditor.class`，使用 Dubbo 事务上下文编辑器，在[「3. Dubbo 事务上下文编辑器」](#)详细分享。

Dubbo Service Proxy 提供了两种生成方式：

* JavassistProxyFactory，基于 Javassist 方式
* JdkProxyFactory，基于 JDK 动态代理机制

这块内容我们不拓展开，感兴趣的同学点击如下文章：

* [《Dubbo学习-理解动态代理》](http://daveztong.github.io/2016/11/23/Dubbo%E5%AD%A6%E4%B9%A0-%E7%90%86%E8%A7%A3%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86/)
* [《Dubbo 作者博客 —— 动态代理方案性能对比》](http://javatar.iteye.com/blog/814426)
* [《Dubbo原理解析-代理之Javassist生成的伪代码》](http://blog.csdn.net/quhongwei_zhanqiu/article/details/41597261)
* **[《Dubbo的服务暴露细节》](http://blog.kazaff.me/2015/01/27/dubbo%E4%B8%AD%E6%9C%8D%E5%8A%A1%E6%9A%B4%E9%9C%B2%E7%9A%84%E7%BB%86%E8%8A%82/)**

Dubbo 的 Invoker 模型是非常关键的概念，看下图：

![](http://www.iocoder.cn/images/TCC-Transaction/2018_03_07/03.jpeg)

## 2.1 JavassistProxyFactory

### 2.1.1 Javassist

> Javassist 是一个开源的分析、编辑和创建 Java 字节码的类库。通过使用Javassist 对字节码操作可以实现动态 ”AOP” 框架。  
> 
> 关于 Java 字节码的处理，目前有很多工具，如 bcel，asm( cglib只是对asm又封装了一层 )。不过这些都需要直接跟虚拟机指令打交道。  
> 
> Javassist 的主要的优点，在于简单，而且快速，直接使用 Java 编码的形式，而不需要了解虚拟机指令，就能动态改变类的结构，或者动态生成类。  

* 粗略一看，可能不够形象，下面我们通过看 TCC-Transaction 如何使用来理解理解。
* [《Java学习之javassist》](http://www.cnblogs.com/sunfie/p/5154246.html)
* [《Javassist 字节码操作》](http://blog.csdn.net/qbg19881206/article/details/8993562)

### 2.1.2 TccJavassistProxyFactory

`org.mengyun.tcctransaction.dubbo.proxy.javassist.TccJavassistProxyFactory`，TCC Javassist 代理工厂。实现代码如下：

```Java
public class TccJavassistProxyFactory extends JavassistProxyFactory {

    @SuppressWarnings("unchecked")
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        return (T) TccProxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
    }

}
```

* **项目启动时**，调用 `TccJavassistProxyFactory#getProxy(...)` 方法，生成 Dubbo Service 调用 Proxy。
* `com.alibaba.dubbo.rpc.proxy.InvokerInvocationHandler`，Dubbo 调用处理器，点击[连接](https://github.com/alibaba/dubbo/blob/17619dfa974457b00fe27cf68ae3f9d266709666/dubbo-rpc/dubbo-rpc-api/src/main/java/com/alibaba/dubbo/rpc/proxy/InvokerInvocationHandler.java)查看代码。

### 2.1.3 TccProxy & TccClassGenerator

`org.mengyun.tcctransaction.dubbo.proxy.javassist.TccProxy`，TCC Proxy 工厂，生成 Dubbo Service 调用 Proxy 。笔者认为，TccProxy 改成 TccProxyFactory 更合适，原因在下文。

`org.mengyun.tcctransaction.dubbo.proxy.javassist.TccClassGenerator`，TCC 类代码生成器，基于 Javassist 实现。 

**🦅案例**

一个 Dubbo Service，TccProxy 会动态生成两个类：

* Dubbo Service 调用 Proxy
* Dubbo Service 调用 ProxyFactory，生成对应的 Dubbo Service Proxy

例如 Dubbo Service 接口如下：

```Java
public interface RedPacketTradeOrderService {

    @Compensable
    String record(RedPacketTradeOrderDto tradeOrderDto);
}
```

生成 Dubbo Service 调用 **ProxyFactory** 如下 ：

```Java
public class TccProxy3 extends TccProxy implements TccClassGenerator.DC {
  public Object newInstance(InvocationHandler paramInvocationHandler) {
    return new proxy3(paramInvocationHandler);
  }
}
```
* TccProxy 提供 `#newInstance(handler)` 方法，创建 Proxy，所以笔者认为，TccProxy 改成 TccProxyFactory 更合适。
* `org.mengyun.tcctransaction.dubbo.proxy.javassist.TccClassGenerator.DC` 动态生成类标记，标记该类由 TccClassGenerator 生成的。

生成 Dubbo Service 调用 **Proxy** 如下 ：

```Java
public class proxy3 implements TccClassGenerator.DC, RedPacketTradeOrderService, EchoService {
    
    public static Method[] methods;
    private InvocationHandler handler;

    public proxy3() {}

    public proxy3(InvocationHandler paramInvocationHandler) {
        this.handler = paramInvocationHandler;
    }

    @Compensable(propagation = Propagation.SUPPORTS, confirmMethod = "record", cancelMethod = "record", transactionContextEditor = DubboTransactionContextEditor.class)
    public String record(RedPacketTradeOrderDto paramRedPacketTradeOrderDto) {
        Object[] arrayOfObject = new Object[1];
        arrayOfObject[0] = paramRedPacketTradeOrderDto;
        Object localObject = this.handler.invoke(this, methods[0], arrayOfObject);
        return (String) localObject;
    }

    public Object $echo(Object paramObject) {
        Object[] arrayOfObject = new Object[1];
        arrayOfObject[0] = paramObject;
        Object localObject = this.handler.invoke(this, methods[1], arrayOfObject);
        return (Object) localObject;
    }
}
```
* `com.alibaba.dubbo.rpc.service.EchoService`，Dubbo Service 回声服务接口，用于服务健康检查，Dubbo Service 默认自动实现该接口，点击[连接](https://github.com/alibaba/dubbo/blob/17619dfa974457b00fe27cf68ae3f9d266709666/dubbo-rpc/dubbo-rpc-api/src/main/java/com/alibaba/dubbo/rpc/service/EchoService.java)查看代码。
* `org.mengyun.tcctransaction.dubbo.proxy.javassist.TccClassGenerator.DC` 动态生成类标记，标记该类由 TccClassGenerator 生成的。

**🦅实现**

调用 `TccProxy#getProxy(...)` 方法，获得 **TCC Proxy 工厂**，实现代码如下：

```Java
  1: // 【TccProxy.java】
  2: public static TccProxy getProxy(ClassLoader cl, Class<?>... ics) {
  3:     // 校验接口超过上限
  4:     if (ics.length > 65535) {
  5:         throw new IllegalArgumentException("interface limit exceeded");
  6:     }
  7: 
  8:     // use interface class name list as key.
  9:     StringBuilder sb = new StringBuilder();
 10:     for (Class<?> ic : ics) {
 11:         String itf = ic.getName();
 12:         // 校验是否为接口
 13:         if (!ic.isInterface()) {
 14:             throw new RuntimeException(itf + " is not a interface.");
 15:         }
 16:         // 加载接口类
 17:         Class<?> tmp = null;
 18:         try {
 19:             tmp = Class.forName(itf, false, cl);
 20:         } catch (ClassNotFoundException ignored) {
 21:         }
 22:         if (tmp != ic) { // 加载接口类失败
 23:             throw new IllegalArgumentException(ic + " is not visible from class loader");
 24:         }
 25:         sb.append(itf).append(';');
 26:     }
 27:     String key = sb.toString();
 28: 
 29:     // get cache by class loader.
 30:     Map<String, Object> cache;
 31:     synchronized (ProxyCacheMap) {
 32:         cache = ProxyCacheMap.get(cl);
 33:         if (cache == null) {
 34:             cache = new HashMap<String, Object>();
 35:             ProxyCacheMap.put(cl, cache);
 36:         }
 37:     }
 38: 
 39:     // 获得 TccProxy 工厂
 40:     TccProxy proxy = null;
 41:     synchronized (cache) {
 42:         do {
 43:             // 从缓存中获取 TccProxy 工厂
 44:             Object value = cache.get(key);
 45:             if (value instanceof Reference<?>) {
 46:                 proxy = (TccProxy) ((Reference<?>) value).get();
 47:                 if (proxy != null) {
 48:                     return proxy;
 49:                 }
 50:             }
 51:             // 缓存中不存在，设置生成 TccProxy 代码标记。创建中时，其他创建请求等待，避免并发。
 52:             if (value == PendingGenerationMarker) {
 53:                 try {
 54:                     cache.wait();
 55:                 } catch (InterruptedException ignored) {
 56:                 }
 57:             } else {
 58:                 cache.put(key, PendingGenerationMarker);
 59:                 break;
 60:             }
 61:         }
 62:         while (true);
 63:     }
 64: 
 65:     long id = PROXY_CLASS_COUNTER.getAndIncrement();
 66:     String pkg = null;
 67:     TccClassGenerator ccp = null; // proxy class generator
 68:     TccClassGenerator ccm = null; // proxy factory class generator
 69:     try {
 70:         // 创建 Tcc class 代码生成器
 71:         ccp = TccClassGenerator.newInstance(cl);
 72: 
 73:         Set<String> worked = new HashSet<String>(); // 已处理方法签名集合。key：方法签名
 74:         List<Method> methods = new ArrayList<Method>(); // 已处理方法集合。
 75: 
 76:         // 处理接口
 77:         for (Class<?> ic : ics) {
 78:             // 非 public 接口，使用接口包名
 79:             if (!Modifier.isPublic(ic.getModifiers())) {
 80:                 String npkg = ic.getPackage().getName();
 81:                 if (pkg == null) {
 82:                     pkg = npkg;
 83:                 } else {
 84:                     if (!pkg.equals(npkg)) { // 实现了两个非 public 的接口，
 85:                         throw new IllegalArgumentException("non-public interfaces from different packages");
 86:                     }
 87:                 }
 88:             }
 89:             // 添加接口
 90:             ccp.addInterface(ic);
 91:             // 处理接口方法
 92:             for (Method method : ic.getMethods()) {
 93:                 // 添加方法签名到已处理方法签名集合
 94:                 String desc = ReflectUtils.getDesc(method);
 95:                 if (worked.contains(desc)) {
 96:                     continue;
 97:                 }
 98:                 worked.add(desc);
 99:                 // 生成接口方法实现代码
100:                 int ix = methods.size();
101:                 Class<?> rt = method.getReturnType();
102:                 Class<?>[] pts = method.getParameterTypes();
103:                 StringBuilder code = new StringBuilder("Object[] args = new Object[").append(pts.length).append("];");
104:                 for (int j = 0; j < pts.length; j++) {
105:                     code.append(" args[").append(j).append("] = ($w)$").append(j + 1).append(";");
106:                 }
107:                 code.append(" Object ret = handler.invoke(this, methods[").append(ix).append("], args);");
108:                 if (!Void.TYPE.equals(rt)) {
109:                     code.append(" return ").append(asArgument(rt, "ret")).append(";");
110:                 }
111:                 methods.add(method);
112:                 // 添加方法
113:                 Compensable compensable = method.getAnnotation(Compensable.class);
114:                 if (compensable != null) {
115:                     ccp.addMethod(true, method.getName(), method.getModifiers(), rt, pts, method.getExceptionTypes(), code.toString());
116:                 } else {
117:                     ccp.addMethod(false, method.getName(), method.getModifiers(), rt, pts, method.getExceptionTypes(), code.toString());
118:                 }
119:             }
120:         }
121: 
122:         // 设置包路径
123:         if (pkg == null) {
124:             pkg = PACKAGE_NAME;
125:         }
126: 
127:         // create ProxyInstance class.
128:         // 设置类名
129:         String pcn = pkg + ".proxy" + id;
130:         ccp.setClassName(pcn);
131:         // 添加静态属性 methods
132:         ccp.addField("public static java.lang.reflect.Method[] methods;");
133:         // 添加属性 handler
134:         ccp.addField("private " + InvocationHandler.class.getName() + " handler;");
135:         // 添加构造方法，参数 handler
136:         ccp.addConstructor(Modifier.PUBLIC, new Class<?>[]{InvocationHandler.class}, new Class<?>[0], "handler=$1;");
137:         // 添加构造方法，参数 空
138:         ccp.addDefaultConstructor();
139:         // 生成类
140:         Class<?> clazz = ccp.toClass();
141:         // 设置静态属性 methods
142:         clazz.getField("methods").set(null, methods.toArray(new Method[0]));
143: 
144:         // create TccProxy class.
145:         // 创建 Tcc class 代码生成器
146:         ccm = TccClassGenerator.newInstance(cl);
147:         // 设置类名
148:         String fcn = TccProxy.class.getName() + id;
149:         ccm.setClassName(fcn);
150:         // 添加构造方法，参数 空
151:         ccm.addDefaultConstructor();
152:         // 设置父类为 TccProxy.class
153:         ccm.setSuperClass(TccProxy.class);
154:         // 添加方法 #newInstance(handler)
155:         ccm.addMethod("public Object newInstance(" + InvocationHandler.class.getName() + " h){ return new " + pcn + "($1); }");
156:         // 生成类
157:         Class<?> pc = ccm.toClass();
158:         // 创建 TccProxy 对象
159:         proxy = (TccProxy) pc.newInstance();
160:     } catch (RuntimeException e) {
161:         throw e;
162:     } catch (Exception e) {
163:         throw new RuntimeException(e.getMessage(), e);
164:     } finally {
165:         // release TccClassGenerator
166:         if (ccp != null) {
167:             ccp.release();
168:         }
169:         if (ccm != null) {
170:             ccm.release();
171:         }
172:         // 唤醒缓存 wait
173:         synchronized (cache) {
174:             if (proxy == null) {
175:                 cache.remove(key);
176:             } else {
177:                 cache.put(key, new WeakReference<TccProxy>(proxy));
178:             }
179:             cache.notifyAll();
180:         }
181:     }
182:     return proxy;
183: }
```

* 第 3 至 7 行 ：校验接口超过上限。
* 第 8 至 27 行 ：使用接口集合类名以 `;` 分隔拼接，作为 Proxy 的唯一标识。例如 ：`key=org.mengyun.tcctransaction.sample.dubbo.redpacket.api.RedPacketAccountService;com.alibaba.dubbo.rpc.service.EchoService;` 。
* 第 29 至 37 行 ：获得 Proxy 对应的 ClassLoader。这里我们看下静态属性 `ProxyCacheMap` 的定义：

    ```Java
    /**
    * Proxy 对象缓存
    * key ：ClassLoader
    * value.key ：Tcc Proxy 标识。使用 Tcc Proxy 实现接口名拼接
    * value.value ：Tcc Proxy 工厂对象
    */
    private static final Map<ClassLoader, Map<String, Object>> ProxyCacheMap = new WeakHashMap<ClassLoader, Map<String, Object>>();
    ```
    * 使用 WeakHashMap，当 ClassLoader 被回收时，其对应的值一起被移除。
    * [《WeakHashMap和HashMap的区别》](http://blog.csdn.net/yangzl2008/article/details/6980709)
    * [《Java 集合系列13之 WeakHashMap详细介绍(源码解析)和使用示例》](http://www.cnblogs.com/skywang12345/p/3311092.html)
* 第 39 至 63 行 ：一直获得 **TCC Proxy 工厂**直到成功。
    * 第 43 至 50 行 ：从**缓存**中获取 TCC Proxy 工厂。
    * 第 51 至 60 行 ：若缓存中不存在，设置**正在生成 TccProxy 代码标记**。创建中时，其他创建请求等待，避免并发。
* 第 65 行 ：`PROXY_CLASS_COUNTER`，Proxy Class 计数，用于生成 Proxy 类名自增。代码如下：

    ```Java
    private static final AtomicLong PROXY_CLASS_COUNTER = new AtomicLong(0);
    ```

* 第 66 至 67 行
    * `ccm`，生成 Dubbo Service 调用 **ProxyFactory** 的代码生成器
    * `ccp`，生成 Dubbo Service 调用 **Proxy** 的代码生成器

* **第 70 至 142 行 ：生成 Dubbo Service 调用 Proxy 的代码**。

    * 第 70 至 71 行 ：调用 `TccClassGenerator#newInstance(loader)` 方法， 创建生成 Dubbo Service 调用 **Proxy** 的代码生成器。实现代码如下：
    
        ```Java
        // TccClassGenerator.java
        public final class TccClassGenerator {
            
            /**
             * CtClass hash 集合
             * key：类名
             */
            private ClassPool mPool;
            
            public static TccClassGenerator newInstance(ClassLoader loader) {
                return new TccClassGenerator(getClassPool(loader));
            }
            
            private TccClassGenerator(ClassPool pool) {
                mPool = pool;
            }
        }
        ```
        * **ClassPool** 是一个 CtClass 对象的 hash 表，类名做为 key 。ClassPool 的 `#get(key)` 搜索 hash 表找到与指定 key 关联的 CtClass 对象。如果没有找到 CtClass 对象，`#get(key)` 读一个类文件构建新的 CtClass 对象，它是被记录在 hash 表中然后返回这个对象。

    * 第 76 至 120 行，处理接口。
        * 第 79 至 88 行，生成类的包名。
        * 第 89 至 90 行，调用 `TccClassGenerator#addInterface(cl)` 方法，添加生成类的接口( **Dubbo Service 接口** )。实现代码如下：

            ```Java
            /**
            * 生成类的接口集合
            */
            private Set<String> mInterfaces;
            
            public TccClassGenerator addInterface(Class<?> cl) {
               return addInterface(cl.getName());
            }
            
            public TccClassGenerator addInterface(String cn) {
               if (mInterfaces == null) {
                   mInterfaces = new HashSet<String>();
               }
               mInterfaces.add(cn);
               return this;
            }
            ```
            * x

        * 第 93 至 98 行，添加方法签名到已处理方法签名集合。多个接口可能存在相同的接口方法，跳过相同的方法，避免冲突。
        * 第 99 至 110 行，生成 Dubbo Service 调用实现代码。案例代码如下：
            
            ```Java
              public String record(RedPacketTradeOrderDto paramRedPacketTradeOrderDto) {
                Object[] arrayOfObject = new Object[1];
                arrayOfObject[0] = paramRedPacketTradeOrderDto;
                Object localObject = this.handler.invoke(this, methods[0], arrayOfObject);
                return (String)localObject;
              }
            ```
            * ![](http://www.iocoder.cn/images/TCC-Transaction/2018_03_07/04.png)

        * 第 112 至 118 行 ：调用 `TccClassGenerator#addMethod(...)` 方法，添加生成的方法。实现代码如下：

            ```Java
            /**
            * 生成类的方法代码集合
            */
            private List<String> mMethods;
            
            /**
            * 带 @Compensable 方法代码集合
            */
            private Set<String> compensableMethods = new HashSet<String>();
            
            public TccClassGenerator addMethod(boolean isCompensableMethod, String name, int mod, Class<?> rt, Class<?>[] pts, Class<?>[] ets, String body) {
               // 拼接方法
               StringBuilder sb = new StringBuilder();
               sb.append(modifier(mod)).append(' ').append(ReflectUtils.getName(rt)).append(' ').append(name);
               sb.append('(');
               for (int i = 0; i < pts.length; i++) {
                   if (i > 0)
                       sb.append(',');
                   sb.append(ReflectUtils.getName(pts[i]));
                   sb.append(" arg").append(i);
               }
               sb.append(')');
               if (ets != null && ets.length > 0) {
                   sb.append(" throws ");
                   for (int i = 0; i < ets.length; i++) {
                       if (i > 0)
                           sb.append(',');
                       sb.append(ReflectUtils.getName(ets[i]));
                   }
               }
               sb.append('{').append(body).append('}');
               // 是否有 @Compensable 注解
               if (isCompensableMethod) {
                   compensableMethods.add(sb.toString());
               }
               return addMethod(sb.toString());
            }
            
            public TccClassGenerator addMethod(String code) {
               if (mMethods == null) {
                   mMethods = new ArrayList<String>();
               }
               mMethods.add(code);
               return this;
            }
            ```

    * 第 122 至 130 行，生成类名( 例如，`org.mengyun.tcctransaction.dubbo.proxy.javassist.proxy3` )，并调用 `TccClassGenerator#setClassName(...)` 方法，设置类名。实现代码如下：

        ```Java
        /**
        * 生成类的类名
        */
        private String mClassName;
        
        public TccClassGenerator setClassName(String name) {
           mClassName = name;
           return this;
        }
        ```
        * x
        
    * 第 131 至 134 行，调用 `TccClassGenerator#addField(...)` 方法，添加**静态**属性 `methods` ( Dubbo Service 方法集合 )和属性 `handler` ( Dubbo InvocationHandler )。实现代码如下：

        ```Java
        /**
        * 生成类的属性集合
        */
        private List<String> mFields;
        
        public TccClassGenerator addField(String code) {
           if (mFields == null) {
               mFields = new ArrayList<String>();
           }
           mFields.add(code);
           return this;
        }
        ```
        * x
 
    * 第 135 至 136 行，调用 `TccClassGenerator#addConstructor(...)` 方法，添加参数为 `handler` 的构造方法。实现代码如下：

        ```Java
        /**
        * 生成类的非空构造方法代码集合
        */
        private List<String> mConstructors;
        
        public TccClassGenerator addConstructor(int mod, Class<?>[] pts, Class<?>[] ets, String body) {
           // 构造方法代码
           StringBuilder sb = new StringBuilder();
           sb.append(modifier(mod)).append(' ').append(SIMPLE_NAME_TAG);
           sb.append('(');
           for (int i = 0; i < pts.length; i++) {
               if (i > 0)
                   sb.append(',');
               sb.append(ReflectUtils.getName(pts[i]));
               sb.append(" arg").append(i);
           }
           sb.append(')');
           if (ets != null && ets.length > 0) {
               sb.append(" throws ");
               for (int i = 0; i < ets.length; i++) {
                   if (i > 0)
                       sb.append(',');
                   sb.append(ReflectUtils.getName(ets[i]));
               }
           }
           sb.append('{').append(body).append('}');
           //
           return addConstructor(sb.toString());
        }
        
        public TccClassGenerator addConstructor(String code) {
           if (mConstructors == null) {
               mConstructors = new LinkedList<String>();
           }
           mConstructors.add(code);
           return this;
        }
        
        public TccClassGenerator addConstructor(String code) {
           if (mConstructors == null) {
               mConstructors = new LinkedList<String>();
           }
           mConstructors.add(code);
           return this;
        }
        ```
        * x
   
    * 第 137 至 138 行，调用 `TccClassGenerator#addDefaultConstructor()` 方法，添加默认空构造方法。实现代码如下：
  
        ```Java
        /**
        * 默认空构造方法
        */
        private boolean mDefaultConstructor = false;
        
        public TccClassGenerator addDefaultConstructor() {
           mDefaultConstructor = true;
           return this;
        }
        ```
        * x
   
    * 第 139 行，调用 `TccClassGenerator#toClass()` 方法，**生成类**。实现代码如下：

        ```Java
          1: public Class<?> toClass() {
          2:    // mCtc 非空时，进行释放；下面会进行创建 mCtc
          3:    if (mCtc != null) {
          4:        mCtc.detach();
          5:    }
          6:    long id = CLASS_NAME_COUNTER.getAndIncrement();
          7:    try {
          8:        CtClass ctcs = mSuperClass == null ? null : mPool.get(mSuperClass);
          9:        if (mClassName == null) { // 类名
         10:            mClassName = (mSuperClass == null || javassist.Modifier.isPublic(ctcs.getModifiers())
         11:                    ? TccClassGenerator.class.getName() : mSuperClass + "$sc") + id;
         12:        }
         13:        // 创建 mCtc
         14:        mCtc = mPool.makeClass(mClassName);
         15:        if (mSuperClass != null) { // 继承类
         16:            mCtc.setSuperclass(ctcs);
         17:        }
         18:        mCtc.addInterface(mPool.get(DC.class.getName())); // add dynamic class tag.
         19:        if (mInterfaces != null) { // 实现接口集合
         20:            for (String cl : mInterfaces) {
         21:                mCtc.addInterface(mPool.get(cl));
         22:            }
         23:        }
         24:        if (mFields != null) { // 属性集合
         25:            for (String code : mFields) {
         26:                mCtc.addField(CtField.make(code, mCtc));
         27:            }
         28:        }
         29:        if (mMethods != null) { // 方法集合
         30:            for (String code : mMethods) {
         31:                if (code.charAt(0) == ':') {
         32:                    mCtc.addMethod(CtNewMethod.copy(getCtMethod(mCopyMethods.get(code.substring(1))), code.substring(1, code.indexOf('(')), mCtc, null));
         33:                } else {
         34:                    CtMethod ctMethod = CtNewMethod.make(code, mCtc);
         35:                    if (compensableMethods.contains(code)) {
         36:                        // 设置 @Compensable 属性
         37:                        ConstPool constpool = mCtc.getClassFile().getConstPool();
         38:                        AnnotationsAttribute attr = new AnnotationsAttribute(constpool, AnnotationsAttribute.visibleTag);
         39:                        Annotation annot = new Annotation("org.mengyun.tcctransaction.api.Compensable", constpool);
         40:                        EnumMemberValue enumMemberValue = new EnumMemberValue(constpool);
         41:                        enumMemberValue.setType("org.mengyun.tcctransaction.api.Propagation");
         42:                        enumMemberValue.setValue("SUPPORTS");
         43:                        annot.addMemberValue("propagation", enumMemberValue);
         44:                        annot.addMemberValue("confirmMethod", new StringMemberValue(ctMethod.getName(), constpool));
         45:                        annot.addMemberValue("cancelMethod", new StringMemberValue(ctMethod.getName(), constpool));
         46:                        ClassMemberValue classMemberValue = new ClassMemberValue("org.mengyun.tcctransaction.dubbo.context.DubboTransactionContextEditor", constpool);
         47:                        annot.addMemberValue("transactionContextEditor", classMemberValue);
         48:                        attr.addAnnotation(annot);
         49:                        ctMethod.getMethodInfo().addAttribute(attr);
         50:                    }
         51:                    mCtc.addMethod(ctMethod);
         52:                }
         53:            }
         54:        }
         55:        if (mDefaultConstructor) { // 空参数构造方法
         56:            mCtc.addConstructor(CtNewConstructor.defaultConstructor(mCtc));
         57:        }
         58:        if (mConstructors != null) { // 带参数构造方法
         59:            for (String code : mConstructors) {
         60:                if (code.charAt(0) == ':') {
         61:                    mCtc.addConstructor(CtNewConstructor.copy(getCtConstructor(mCopyConstructors.get(code.substring(1))), mCtc, null));
         62:                } else {
         63:                    String[] sn = mCtc.getSimpleName().split("\\$+"); // inner class name include $.
         64:                    mCtc.addConstructor(CtNewConstructor.make(code.replaceFirst(SIMPLE_NAME_TAG, sn[sn.length - 1]), mCtc));
         65:                }
         66:            }
         67:        }
         68: //            mCtc.debugWriteFile("/Users/yunai/test/" + mCtc.getSimpleName().replaceAll(".", "/") + ".class");
         69:        // 生成
         70:        return mCtc.toClass();
         71:    } catch (RuntimeException e) {
         72:        throw e;
         73:    } catch (NotFoundException e) {
         74:        throw new RuntimeException(e.getMessage(), e);
         75:    } catch (CannotCompileException e) {
         76:        throw new RuntimeException(e.getMessage(), e);
         77:    }
         78: }
        ```
        * 基于 Javassist 生成类。这里不做拓展解释，配合[《Java学习之javassist》](http://www.cnblogs.com/sunfie/p/5154246.html)一起理解。
        * 第 18 行，添加 `org.mengyun.tcctransaction.dubbo.proxy.javassist.TccClassGenerator.DC` 动态生成类标记，标记该类由 TccClassGenerator 生成的。
        * 第 34 至 50 行，设置 @Compensable 默认属性。

    * 第 141 至 142 行，设置 Dubbo Service 方法集合设置到静态属性 `methods` 上。

* **第 144 至 157 行，生成 Dubbo Service 调用 Proxy 工厂的代码**。
    * 第 146 行，调用 `TccClassGenerator#newInstance(loader)` 方法， 创建生成 Dubbo Service 调用 **Proxy 工厂** 的代码生成器。
    * 第 147 至 149 行，生成类名( 例如，`org.mengyun.tcctransaction.dubbo.proxy.javassist.TccProxy3` )，并调用 `TccClassGenerator#setClassName(...)` 方法，设置类名。
    * 第 150 至 151 行，调用 `TccClassGenerator#addDefaultConstructor()` 方法，添加默认空构造方法。
    * 第 152 至 153 行，调用 `TccClassGenerator#mSuperClass()` 方法，设置继承父类 **`TccProxy`**。实现代码如下：

        ```Java
        /**
        * 生成类的父类名字
        */
        private String mSuperClass;
        
        public TccClassGenerator setSuperClass(Class<?> cl) {
           mSuperClass = cl.getName();
           return this;
        }
        ```
        * x
   
    * 第 154 至 155 行，调用 `TccClassGenerator#addInterface(cl)` 方法，添加生成 Proxy 实现代码的方法。代码案例如下：

       ```Java
       public Object newInstance(InvocationHandler paramInvocationHandler) {
            return new proxy3(paramInvocationHandler);
       }
       ```
       * x

    * 第 156 至 157 行，调用 `TccClassGenerator#toClass()` 方法，**生成类**。
* 第 159 行，调用 `TccProxy#newInstance()` 方法，创建 Proxy 。实现代码如下：

    ```Java
    /**
    * get instance with default handler.
    *
    * @return instance.
    */
    public Object newInstance() {
       return newInstance(THROW_UNSUPPORTED_INVOKER);
    }
    
    /**
    * get instance with special handler.
    *
    * @return instance.
    */
    abstract public Object newInstance(InvocationHandler handler);
    ```
    * `#newInstance(handler)`，抽象方法，上面第 154 至 155 行生成。TccJavassistProxyFactory 调用该方法，获得 Proxy 。

* 第 165 至 171 行，释放 TccClassGenerator 。实现代码如下：

    ```Java
    public void release() {
       if (mCtc != null) {
           mCtc.detach();
       }
       if (mInterfaces != null) {
           mInterfaces.clear();
       }
       if (mFields != null) {
           mFields.clear();
       }
       if (mMethods != null) {
           mMethods.clear();
       }
       if (mConstructors != null) {
           mConstructors.clear();
       }
       if (mCopyMethods != null) {
           mCopyMethods.clear();
       }
       if (mCopyConstructors != null) {
           mCopyConstructors.clear();
       }
    }
    ```

* 第 172 至 180 行，设置 Proxy 工厂缓存，并唤醒等待线程。

**ps：**代码比较多，收获会比较多，算是 Javassist 实战案例了。TCC-Transaction 作者在实现上述类，可能参考了 Dubbo 自带的实现：

* [`com.alibaba.dubbo.common.bytecode.Proxy`](https://github.com/alibaba/dubbo/blob/8f20e3a68efc350e3fbaa965e0a8e8a59fef1b3c/dubbo-common/src/main/java/com/alibaba/dubbo/common/bytecode/Proxy.java)
* [`com.alibaba.dubbo.common.bytecode.ClassGenerator`](https://github.com/alibaba/dubbo/blob/8f20e3a68efc350e3fbaa965e0a8e8a59fef1b3c/dubbo-common/src/main/java/com/alibaba/dubbo/common/bytecode/ClassGenerator.java)
* [`com.alibaba.dubbo.common.bytecode.Wrapper`](https://github.com/alibaba/dubbo/blob/8f20e3a68efc350e3fbaa965e0a8e8a59fef1b3c/dubbo-common/src/main/java/com/alibaba/dubbo/common/bytecode/Wrapper.java)

### 2.1.4 配置 Dubbo Proxy
       
```XML
// META-INF.dubbo/com.alibaba.dubbo.rpc.ProxyFactory
tccJavassist=org.mengyun.tcctransaction.dubbo.proxy.javassist.TccJavassistProxyFactory

// tcc-transaction-dubbo.xml
<dubbo:provider proxy="tccJavassist"/>
```

目前 Maven 项目 `tcc-transaction-dubbo` 已经**默认**配置，引入即可。

![](http://www.iocoder.cn/images/TCC-Transaction/2018_03_07/05.png)
       
## 2.2 JdkProxyFactory

### 2.2.1 JDK Proxy

[《 Java JDK 动态代理（AOP）使用及实现原理分析》](http://blog.csdn.net/jiankunking/article/details/52143504#)

### 2.2.2 TccJdkProxyFactory


`org.mengyun.tcctransaction.dubbo.proxy.jd.TccJdkProxyFactory`，TCC JDK 代理工厂。实现代码如下：

```Java
public class TccJdkProxyFactory extends JdkProxyFactory {

    @SuppressWarnings("unchecked")
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        T proxy = (T) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), interfaces, new InvokerInvocationHandler(invoker));
        return (T) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), interfaces, new TccInvokerInvocationHandler(proxy, invoker));
    }

}
```

* **项目启动时**，调用 `TccJavassistProxyFactory#getProxy(...)` 方法，生成 Dubbo Service 调用 Proxy。
* **第一次**调用 `Proxy#newProxyInstance(...)` 方法，创建调用 Dubbo Service 服务的 Proxy。`com.alibaba.dubbo.rpc.proxy.InvokerInvocationHandler`，Dubbo 调用处理器，点击[连接](https://github.com/alibaba/dubbo/blob/17619dfa974457b00fe27cf68ae3f9d266709666/dubbo-rpc/dubbo-rpc-api/src/main/java/com/alibaba/dubbo/rpc/proxy/InvokerInvocationHandler.java)查看代码。
* **第二次**调用 `Proxy#newProxyInstance(...)` 方法，创建对调用 Dubbo Service 的 Proxy 的 Proxy。为什么会有两层 Proxy？答案在下节 TccInvokerInvocationHandler 。

### 2.2.3 TccInvokerInvocationHandler

`org.mengyun.tcctransaction.dubbo.proxy.jdk.TccInvokerInvocationHandler`，TCC 调用处理器，在调用 Dubbo Service 服务时，使用 ResourceCoordinatorInterceptor 拦截处理。实现代码如下：

```Java
  1: public class TccInvokerInvocationHandler extends InvokerInvocationHandler {
  2: 
  3:     /**
  4:      * proxy
  5:      */
  6:     private Object target;
  7: 
  8:     public TccInvokerInvocationHandler(Invoker<?> handler) {
  9:         super(handler);
 10:     }
 11: 
 12:     public <T> TccInvokerInvocationHandler(T target, Invoker<T> invoker) {
 13:         super(invoker);
 14:         this.target = target;
 15:     }
 16: 
 17:     public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
 18:         Compensable compensable = method.getAnnotation(Compensable.class);
 19:         if (compensable != null) {
 20:             // 设置 @Compensable 属性
 21:             if (StringUtils.isEmpty(compensable.confirmMethod())) {
 22:                 ReflectionUtils.changeAnnotationValue(compensable, "confirmMethod", method.getName());
 23:                 ReflectionUtils.changeAnnotationValue(compensable, "cancelMethod", method.getName());
 24:                 ReflectionUtils.changeAnnotationValue(compensable, "transactionContextEditor", DubboTransactionContextEditor.class);
 25:                 ReflectionUtils.changeAnnotationValue(compensable, "propagation", Propagation.SUPPORTS);
 26:             }
 27:             // 生成切面
 28:             ProceedingJoinPoint pjp = new MethodProceedingJoinPoint(proxy, target, method, args);
 29:             // 执行
 30:             return FactoryBuilder.factoryOf(ResourceCoordinatorAspect.class).getInstance().interceptTransactionContextMethod(pjp);
 31:         } else {
 32:             return super.invoke(target, method, args);
 33:         }
 34:     }
 35: 
 36: }
```

* 第 18 至 26 行，设置带有 @Compensable 属性的默认属性。
* 第 28 行，生成方法切面 `org.mengyun.tcctransaction.dubbo.proxy.jdk.MethodProceedingJoinPoint`。实现代码如下：

    ```Java
    public class MethodProceedingJoinPoint implements ProceedingJoinPoint, JoinPoint.StaticPart {
    
        /**
         * 代理对象
         */
        private Object proxy;
        /**
         * 目标对象
         */
        private Object target;
        /**
         * 方法
         */
        private Method method;
        /**
         * 参数
         */
        private Object[] args;
        
        @Override
        public Object proceed() throws Throwable {
            // Use reflection to invoke the method.
            try {
                ReflectionUtils.makeAccessible(method);
                return method.invoke(target, args);
            } catch (InvocationTargetException ex) {
                // Invoked method threw a checked exception.
                // We must rethrow it. The client won't see the interceptor.
                throw ex.getTargetException();
            } catch (IllegalArgumentException ex) {
                throw new SystemException("Tried calling method [" +
                        method + "] on target [" + target + "] failed", ex);
            } catch (IllegalAccessException ex) {
                throw new SystemException("Could not access method [" + method + "]", ex);
            }
        }
    
        @Override
        public Object proceed(Object[] objects) throws Throwable {
            //        throw new UnsupportedOperationException(); // TODO 芋艿：疑问
            return proceed();
        }
        
        // ... 省略不重要的方法和对象
    }
    ```
    * 该类参考 [`org.springframework.aop.aspectj.MethodInvocationProceedingJoinPoint`](https://github.com/spring-projects/spring-framework/blob/master/spring-aop/src/main/java/org/springframework/aop/aspectj/MethodInvocationProceedingJoinPoint.java) 实现。
    * TODO【1】 proxy 和 target 是否保留一个即可？
    * 在切面处理完成后，调用 `#proceed(...)` 方法，进行远程 Dubbo Service 服务调用。
    * TODO【2】`#proceed(objects)` 抛出 throw new UnsupportedOperationException();。需要跟作者确认下。
* 调用 `ResourceCoordinatorAspect#interceptTransactionContextMethod(...)` 方法，对方法切面拦截处理。**为什么无需调用 CompensableTransactionAspect 切面**？因为传播级别为 Propagation.SUPPORTS，不会发起事务。

### 2.2.4 配置 Dubbo Proxy

```XML
// META-INF.dubbo/com.alibaba.dubbo.rpc.ProxyFactory
tccJdk=org.mengyun.tcctransaction.dubbo.proxy.jdk.TccJdkProxyFactory

// appcontext-service-dubbo.xml
<dubbo:provider proxy="tccJdk"/>

<dubbo:reference proxy="tccJdk" id="captialTradeOrderService"
                     interface="org.mengyun.tcctransaction.sample.dubbo.capital.api.CapitalTradeOrderService" timeout="5000"/>
```

* ProxyFactory 的 `tccJdk` 在 Maven 项 `tcc-transaction-dubbo` 已经声明。
* 声明 `dubbo:provider` 的 `proxy="tccJdk"`。
* 声明 `dubbo:reference` 的 `proxy="tccJdk"`，否则不生效。

# 3. Dubbo 事务上下文编辑器

`org.mengyun.tcctransaction.dubbo.context.DubboTransactionContextEditor`，Dubbo 事务上下文编辑器实现，实现代码如下：

```Java
public class DubboTransactionContextEditor implements TransactionContextEditor {

    @Override
    public TransactionContext get(Object target, Method method, Object[] args) {
        String context = RpcContext.getContext().getAttachment(TransactionContextConstants.TRANSACTION_CONTEXT);
        if (StringUtils.isNotEmpty(context)) {
            return JSON.parseObject(context, TransactionContext.class);
        }
        return null;
    }

    @Override
    public void set(TransactionContext transactionContext, Object target, Method method, Object[] args) {
        RpcContext.getContext().setAttachment(TransactionContextConstants.TRANSACTION_CONTEXT, JSON.toJSONString(transactionContext));
    }

}
```

* 通过 Dubbo 的隐式传参的方式，避免在 Dubbo Service 接口上声明 TransactionContext 参数，对接口产生一定的入侵。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

HOHO，对动态代理又学习了一遍，蛮 High 的。

这里推荐动态代理无关，和 Dubbo 相关的文章：

* [《Dubbo的服务暴露细节》](http://blog.kazaff.me/2015/01/27/dubbo%E4%B8%AD%E6%9C%8D%E5%8A%A1%E6%9A%B4%E9%9C%B2%E7%9A%84%E7%BB%86%E8%8A%82/)。
* [《Dubbo Provider启动主流程》](http://weifuwu.io/2016/01/03/dubbo-provider-start/)

胖友，分享一波朋友圈可好。



