title: Tomcat 7 启动分析（二）Bootstrap 类中的 main 方法
date: 2018-01-04
tag: 
categories: Tomcat
permalink: Tomcat/yuliu/Start-analysis-2-Bootstrap
author: 预流
from_url: https://juejin.im/post/5a6d1e226fb9a01cc0267be1
wechat_url: 

-------

摘要: 原创出处 https://juejin.im/post/5a6d1e226fb9a01cc0267be1 「预流」欢迎转载，保留摘要，谢谢！


-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

之前分析了 Tomcat 的启动脚本，如果从 startup.bat 开始启动 Tomcat 的话会发现最后会调用 org.apache.catalina.startup.Bootstrap 里的 main 方法，并且传过来的最后一个命令行参数是 start，接下来的启动代码分析就从这里开始。

先看下这个 main 方法的代码：

```Java
     1	/**
     2	     * Main method and entry point when starting Tomcat via the provided
     3	     * scripts.
     4	     *
     5	     * @param args Command line arguments to be processed
     6	     */
     7	    public static void main(String args[]) {
     8	
     9	        if (daemon == null) {
    10	            // Don't set daemon until init() has completed
    11	            Bootstrap bootstrap = new Bootstrap();
    12	            try {
    13	                bootstrap.init();
    14	            } catch (Throwable t) {
    15	                handleThrowable(t);
    16	                t.printStackTrace();
    17	                return;
    18	            }
    19	            daemon = bootstrap;
    20	        } else {
    21	            // When running as a service the call to stop will be on a new
    22	            // thread so make sure the correct class loader is used to prevent
    23	            // a range of class not found exceptions.
    24	            Thread.currentThread().setContextClassLoader(daemon.catalinaLoader);
    25	        }
    26	
    27	        try {
    28	            String command = "start";
    29	            if (args.length > 0) {
    30	                command = args[args.length - 1];
    31	            }
    32	
    33	            if (command.equals("startd")) {
    34	                args[args.length - 1] = "start";
    35	                daemon.load(args);
    36	                daemon.start();
    37	            } else if (command.equals("stopd")) {
    38	                args[args.length - 1] = "stop";
    39	                daemon.stop();
    40	            } else if (command.equals("start")) {
    41	                daemon.setAwait(true);
    42	                daemon.load(args);
    43	                daemon.start();
    44	            } else if (command.equals("stop")) {
    45	                daemon.stopServer(args);
    46	            } else if (command.equals("configtest")) {
    47	                daemon.load(args);
    48	                if (null==daemon.getServer()) {
    49	                    System.exit(1);
    50	                }
    51	                System.exit(0);
    52	            } else {
    53	                log.warn("Bootstrap: command \"" + command + "\" does not exist.");
    54	            }
    55	        } catch (Throwable t) {
    56	            // Unwrap the Exception for clearer error reporting
    57	            if (t instanceof InvocationTargetException &&
    58	                    t.getCause() != null) {
    59	                t = t.getCause();
    60	            }
    61	            handleThrowable(t);
    62	            t.printStackTrace();
    63	            System.exit(1);
    64	        }
    65	
    66	    }

```

这里的 daemon 是 Bootstrap 类中的一个静态成员变量，类型就是 Bootstrap，第 10 行的注释已经说明在调用过 init 方法之后才会给该变量赋值，初始时将是 null，所以首先将实例化一个 Bootstrap 对象，接着调用 init 方法，该方法代码如下：

```Java
     1	/**
     2	     * Initialize daemon.
     3	     */
     4	    public void init()
     5	        throws Exception
     6	    {
     7	
     8	        // Set Catalina path
     9	        setCatalinaHome();
    10	        setCatalinaBase();
    11	
    12	        initClassLoaders();
    13	
    14	        Thread.currentThread().setContextClassLoader(catalinaLoader);
    15	
    16	        SecurityClassLoad.securityClassLoad(catalinaLoader);
    17	
    18	        // Load our startup class and call its process() method
    19	        if (log.isDebugEnabled())
    20	            log.debug("Loading startup class");
    21	        Class startupClass =
    22	            catalinaLoader.loadClass
    23	            ("org.apache.catalina.startup.Catalina");
    24	        Object startupInstance = startupClass.newInstance();
    25	
    26	        // Set the shared extensions class loader
    27	        if (log.isDebugEnabled())
    28	            log.debug("Setting startup class properties");
    29	        String methodName = "setParentClassLoader";
    30	        Class paramTypes[] = new Class[1];
    31	        paramTypes[0] = Class.forName("java.lang.ClassLoader");
    32	        Object paramValues[] = new Object[1];
    33	        paramValues[0] = sharedLoader;
    34	        Method method =
    35	            startupInstance.getClass().getMethod(methodName, paramTypes);
    36	        method.invoke(startupInstance, paramValues);
    37	
    38	        catalinaDaemon = startupInstance;
    39	
    40	    }

```

这里不再逐句解释代码的作用，总的来说这个方法主要做了一下几件事：

1. 设置 catalina.home、catalina.base 系统属性，
2. 创建commonLoader、catalinaLoader、sharedLoader类加载器（默认情况下这三个类加载器指向同一个对象。建议看看 createClassLoader 方法，里面做的事情还挺多，比如装载 catalina.properties 里配置的目录下的文件和jar包，后两个加载器的父加载器都是第一个，最后注册了 MBean，可以用于 JVM 监控该对象），
3. 实例化一个 org.apache.catalina.startup.Catalina 对象，并赋值给静态成员 catalinaDaemon，以 sharedLoader 作为入参通过反射调用该对象的 setParentClassLoader 方法。

接下来去命令行最后一个参数，按文章开头所说是 start，所以将执行 34 行到 36 行的代码，将会执行 Bootstrap 类中的 load、start 方法。

load 方法代码如下：

```Java
     1	    /**
     2	     * Load daemon.
     3	     */
     4	    private void load(String[] arguments)
     5	        throws Exception {
     6	
     7	        // Call the load() method
     8	        String methodName = "load";
     9	        Object param[];
    10	        Class paramTypes[];
    11	        if (arguments==null || arguments.length==0) {
    12	            paramTypes = null;
    13	            param = null;
    14	        } else {
    15	            paramTypes = new Class[1];
    16	            paramTypes[0] = arguments.getClass();
    17	            param = new Object[1];
    18	            param[0] = arguments;
    19	        }
    20	        Method method =
    21	            catalinaDaemon.getClass().getMethod(methodName, paramTypes);
    22	        if (log.isDebugEnabled())
    23	            log.debug("Calling startup class " + method);
    24	        method.invoke(catalinaDaemon, param);
    25	
    26	    }

```

就是通过反射调用 catalinaDaemon 对象的 load 方法，catalinaDaemon 对象在上面的 init 方法中已经实例化过了。

start 方法与 load 方法相似，也是通过反射调用 catalinaDaemon 对象上的 start 方法：

```Java
     1	    /**
     2	     * Start the Catalina daemon.
     3	     */
     4	    public void start()
     5	        throws Exception {
     6	        if( catalinaDaemon==null ) init();
     7	
     8	        Method method = catalinaDaemon.getClass().getMethod("start", (Class [] )null);
     9	        method.invoke(catalinaDaemon, (Object [])null);
    10	
    11	    }

```

下面一篇文章将分析 catalinaDaemon 对象中的 load、start 两个方法，里面会涉及一个有趣的话题 —— Digester 的使用。