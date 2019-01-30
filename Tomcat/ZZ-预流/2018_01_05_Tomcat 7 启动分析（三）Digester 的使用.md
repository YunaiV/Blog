title: Tomcat 7 启动分析（三）Digester 的使用
date: 2018-01-05
tag: 
categories: Tomcat
permalink: Tomcat/yuliu/Start-analysis-3-Digester
author: 预流
from_url: https://juejin.im/post/5a6d1ff6f265da3e243bc1de
wechat_url: 

-------

摘要: 原创出处 https://juejin.im/post/5a6d15af518825732258f10d 「预流」欢迎转载，保留摘要，谢谢！

- [一、分析 startup.bat 文件](http://www.iocoder.cn/Tomcat/yuliu/Start-analysis-1-start-script/)
- [二、分析 catalina.bat 文件](http://www.iocoder.cn/Tomcat/yuliu/Start-analysis-1-start-script/)
- [三、总结](http://www.iocoder.cn/Tomcat/yuliu/Start-analysis-1-start-script/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

前一篇文章里最后看到 Bootstrap 的 main 方法最后会调用 org.apache.catalina.startup.Catalina 对象的 load 和 start 两个方法，那么就来看看这两个方法里面到底做了些什么。

load 方法：

```Java
     1	    /**
     2	     * Start a new server instance.
     3	     */
     4	    public void load() {
     5
     6	        long t1 = System.nanoTime();
     7
     8	        initDirs();
     9
    10	        // Before digester - it may be needed
    11
    12	        initNaming();
    13
    14	        // Create and execute our Digester
    15	        Digester digester = createStartDigester();
    16
    17	        InputSource inputSource = null;
    18	        InputStream inputStream = null;
    19	        File file = null;
    20	        try {
    21	            file = configFile();
    22	            inputStream = new FileInputStream(file);
    23	            inputSource = new InputSource(file.toURI().toURL().toString());
    24	        } catch (Exception e) {
    25	            if (log.isDebugEnabled()) {
    26	                log.debug(sm.getString("catalina.configFail", file), e);
    27	            }
    28	        }
    29	        if (inputStream == null) {
    30	            try {
    31	                inputStream = getClass().getClassLoader()
    32	                    .getResourceAsStream(getConfigFile());
    33	                inputSource = new InputSource
    34	                    (getClass().getClassLoader()
    35	                     .getResource(getConfigFile()).toString());
    36	            } catch (Exception e) {
    37	                if (log.isDebugEnabled()) {
    38	                    log.debug(sm.getString("catalina.configFail",
    39	                            getConfigFile()), e);
    40	                }
    41	            }
    42	        }
    43
    44	        // This should be included in catalina.jar
    45	        // Alternative: don't bother with xml, just create it manually.
    46	        if( inputStream==null ) {
    47	            try {
    48	                inputStream = getClass().getClassLoader()
    49	                        .getResourceAsStream("server-embed.xml");
    50	                inputSource = new InputSource
    51	                (getClass().getClassLoader()
    52	                        .getResource("server-embed.xml").toString());
    53	            } catch (Exception e) {
    54	                if (log.isDebugEnabled()) {
    55	                    log.debug(sm.getString("catalina.configFail",
    56	                            "server-embed.xml"), e);
    57	                }
    58	            }
    59	        }
    60
    61
    62	        if (inputStream == null || inputSource == null) {
    63	            if  (file == null) {
    64	                log.warn(sm.getString("catalina.configFail",
    65	                        getConfigFile() + "] or [server-embed.xml]"));
    66	            } else {
    67	                log.warn(sm.getString("catalina.configFail",
    68	                        file.getAbsolutePath()));
    69	                if (file.exists() && !file.canRead()) {
    70	                    log.warn("Permissions incorrect, read permission is not allowed on the file.");
    71	                }
    72	            }
    73	            return;
    74	        }
    75
    76	        try {
    77	            inputSource.setByteStream(inputStream);
    78	            digester.push(this);
    79	            digester.parse(inputSource);
    80	        } catch (SAXParseException spe) {
    81	            log.warn("Catalina.start using " + getConfigFile() + ": " +
    82	                    spe.getMessage());
    83	            return;
    84	        } catch (Exception e) {
    85	            log.warn("Catalina.start using " + getConfigFile() + ": " , e);
    86	            return;
    87	        } finally {
    88	            try {
    89	                inputStream.close();
    90	            } catch (IOException e) {
    91	                // Ignore
    92	            }
    93	        }
    94
    95	        getServer().setCatalina(this);
    96
    97	        // Stream redirection
    98	        initStreams();
    99
   100	        // Start the new server
   101	        try {
   102	            getServer().init();
   103	        } catch (LifecycleException e) {
   104	            if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE")) {
   105	                throw new java.lang.Error(e);
   106	            } else {
   107	                log.error("Catalina.start", e);
   108	            }
   109
   110	        }
   111
   112	        long t2 = System.nanoTime();
   113	        if(log.isInfoEnabled()) {
   114	            log.info("Initialization processed in " + ((t2 - t1) / 1000000) + " ms");
   115	        }
   116
   117	    }

```

这个 117 行的代码看起来东西挺多，把注释、异常抛出、记录日志、流关闭、非空判断这些放在一边就会发现实际上真正做事的就这么几行代码：

```Java
Digester digester = createStartDigester();
inputSource.setByteStream(inputStream);
digester.push(this);
digester.parse(inputSource);
getServer().setCatalina(this);
getServer().init();

```

做的事情就两个，一是创建一个 Digester 对象，将当前对象压入 Digester 里的对象栈顶，根据 inputSource 里设置的文件 xml 路径及所创建的 Digester 对象所包含的解析规则生成相应对象，并调用相应方法将对象之间关联起来。二是调用 Server 接口对象的 init 方法。

这里碰到了 Digester，有必要介绍一下 Digester 的一些基础知识。一般来说 Java 里解析 xml 文件有两种方式：一种是 Dom4J 之类将文件全部读取到内存中，在内存里构造一棵 Dom 树的方式来解析。一种是 SAX 的读取文件流，在流中碰到相应的xml节点触发相应的节点事件回调相应方法，基于事件的解析方式，优点是不需要先将文件全部读取到内存中。

Digester 本身是采用 SAX 的解析方式，在其上提供了一层包装，对于使用者更简便友好罢了。最早是在 struts 1 里面用的，后来独立出来成为 apache 的 Commons 下面的一个单独的子项目。Tomcat 里又把它又封装了一层，为了描述方便，直接拿 Tomcat 里的 Digester 建一个单独的 Digester 的例子来介绍。

```Java
     1	package org.study.digester;
     2
     3	import java.io.IOException;
     4	import java.io.InputStream;
     5	import java.util.ArrayList;
     6	import java.util.HashMap;
     7	import java.util.List;
     8
     9	import junit.framework.Assert;
    10
    11	import org.apache.tomcat.util.digester.Digester;
    12	import org.xml.sax.InputSource;
    13
    14	public class MyDigester {
    15
    16	    private MyServer myServer;
    17
    18	    public MyServer getMyServer() {
    19	        return myServer;
    20	    }
    21
    22	    public void setMyServer(MyServer myServer) {
    23	        this.myServer = myServer;
    24	    }
    25
    26	    private Digester createStartDigester() {
    27	        // 实例化一个Digester对象
    28	        Digester digester = new Digester();
    29
    30	        // 设置为false表示解析xml时不需要进行DTD的规则校验
    31	        digester.setValidating(false);
    32
    33	        // 是否进行节点设置规则校验,如果xml中相应节点没有设置解析规则会在控制台显示提示信息
    34	        digester.setRulesValidation(true);
    35
    36	        // 将xml节点中的className作为假属性，不必调用默认的setter方法（一般的节点属性在解析时将会以属性值作为入参调用该节点相应对象的setter方法，而className属性的作用是提示解析器用该属性的值来实例化对象）
    37	        HashMap, List> fakeAttributes = new HashMap, List>();
    38	        ArrayList attrs = new ArrayList();
    39	        attrs.add("className");
    40	        fakeAttributes.put(Object.class, attrs);
    41	        digester.setFakeAttributes(fakeAttributes);
    42
    43	        // addObjectCreate方法的意思是碰到xml文件中的Server节点则创建一个MyStandardServer对象
    44	        digester.addObjectCreate("Server",
    45	                "org.study.digester.MyStandardServer", "className");
    46	        // 根据Server节点中的属性信息调用相应属性的setter方法，以上面的xml文件为例则会调用setPort、setShutdown方法，入参分别是8005、SHUTDOWN
    47	        digester.addSetProperties("Server");
    48	        // 将Server节点对应的对象作为入参调用栈顶对象的setMyServer方法，这里的栈顶对象即下面的digester.push方法所设置的当前类的对象this，就是说调用MyDigester类的setMyServer方法
    49	        digester.addSetNext("Server", "setMyServer",
    50	                "org.study.digester.MyServer");
    51
    52	        // 碰到xml的Server节点下的Listener节点时取className属性的值作为实例化类实例化一个对象
    53	        digester.addObjectCreate("Server/Listener", null, "className");
    54	        digester.addSetProperties("Server/Listener");
    55	        digester.addSetNext("Server/Listener", "addLifecycleListener",
    56	                "org.apache.catalina.LifecycleListener");
    57
    58	        digester.addObjectCreate("Server/Service",
    59	                "org.study.digester.MyStandardService", "className");
    60	        digester.addSetProperties("Server/Service");
    61	        digester.addSetNext("Server/Service", "addMyService",
    62	                "org.study.digester.MyService");
    63
    64	        digester.addObjectCreate("Server/Service/Listener", null, "className");
    65	        digester.addSetProperties("Server/Service/Listener");
    66	        digester.addSetNext("Server/Service/Listener", "addLifecycleListener",
    67	                "org.apache.catalina.LifecycleListener");
    68	        return digester;
    69	    }
    70
    71	    public MyDigester() {
    72	        Digester digester = createStartDigester();
    73
    74	        InputSource inputSource = null;
    75	        InputStream inputStream = null;
    76	        try {
    77	            String configFile = "myServer.xml";
    78	            inputStream = getClass().getClassLoader().getResourceAsStream(
    79	                    configFile);
    80	            inputSource = new InputSource(getClass().getClassLoader()
    81	                    .getResource(configFile).toString());
    82
    83	            inputSource.setByteStream(inputStream);
    84	            digester.push(this);
    85	            digester.parse(inputSource);
    86	        } catch (Exception e) {
    87	            e.printStackTrace();
    88	        } finally {
    89	            try {
    90	                inputStream.close();
    91	            } catch (IOException e) {
    92	                // Ignore
    93	            }
    94	        }
    95
    96	        getMyServer().setMyDigester(this);
    97	    }
    98
    99	    public static void main(String[] agrs) {
   100	        MyDigester md = new MyDigester();
   101	        Assert.assertNotNull(md.getMyServer());
   102	    }
   103	}

```

上面是我自己写的一个拿 Tomcat 里的 Digester 的工具类解析 xml 文件的例子，关键方法的调用含义已经在注释中写明，解析的是项目源文件发布的根目录下的 myServer.xml 文件。

```XML
<?xml version='1.0' encoding='utf-8'?>

<Server port="8005" shutdown="SHUTDOWN">

	<Listener
		className="org.apache.catalina.core.JasperListener" />

	<Listener
		className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />

	<Service name="Catalina">

		<Listener
			className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />
	</Service>
</Server>

```

Digester 的使用一般来说有4步：

1. 实例化一个 Digester 对象，并在对象里设置相应的节点解析规则。
2. 设置要解析的文件作为输入源( InputSource )，这里 InputSource 与 SAX 里的一样，用以表示一个 xml 实体。
3. 将当前对象压入栈。
4. 调用 Digester 的 parse 方法解析 xml，生成相应的对象。

第 3 步中将当前对象压入栈中的作用是可以保存一个到 Digester 生成的一系列对象直接的引用，方便后续使用而已，所以不必是当前对象，只要有一个地方存放这个引用即可。

这里有必要说明的是很多文章里按照代码顺序来描述 Digester 的所谓栈模型来讲述 addSetNext 方法时的调用对象，实际上我的理解 addSetNext 方法具体哪个调用对象与XML文件里的节点树形结构相关，当前节点的父节点是哪个对象该对象就是调用对象。可以试验一下把这里的代码顺序打乱仍然可以按规则解析出来，createStartDigester 方法只是在定义解析规则，具体解析与 addObjectCreate、addSetProperties、addSetNext 这些方法的调用顺序无关。Digester 自己内部在解析 xml 的节点元素时增加了一个 rule 的概念，addObjectCreate、addSetProperties、addSetNext 这些方法内部实际上是在添加 rule，在最后解析 xml 时将会根据读取到的节点匹配相应节点路径下的 rule，调用内部的方法。关于 Tomcat 内的 Digester 的解析原理以后可以单独写篇文章分析一下。

该示例的代码在附件中，将其放入 tomcat 7 的源代码环境中即可直接运行。

看懂了上面的例子 Catalina 的 createStartDigester 方法应该就可以看懂了，它只是比示例要处理的节点类型更多，并且增加几个自定义的解析规则，如 384 行在碰到 Server/GlobalNamingResources/ 节点时将会调用 org.apache.catalina.startup.NamingRuleSet 类中的 addRuleInstances 方法添加解析规则。

要解析的 XML 文件默认会先找 conf/server.xml，如果当前项目找不到则通过其他路径找 xml 文件来解析，这里以默认情况为例将会解析 server.xml

```XML
<?xml version='1.0' encoding='utf-8'?>
<!--
  Licensed to the Apache Software Foundation (ASF) under one or more
  contributor license agreements.  See the NOTICE file distributed with
  this work for additional information regarding copyright ownership.
  The ASF licenses this file to You under the Apache License, Version 2.0
  (the "License"); you may not use this file except in compliance with
  the License.  You may obtain a copy of the License at

      http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->
<!-- Note:  A "Server" is not itself a "Container", so you may not
     define subcomponents such as "Valves" at this level.
     Documentation at /docs/config/server.html
 -->
<Server port="8005" shutdown="SHUTDOWN">
  <!-- Security listener. Documentation at /docs/config/listeners.html
  <Listener className="org.apache.catalina.security.SecurityListener" />
  -->
  <!--APR library loader. Documentation at /docs/apr.html -->
  <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
  <!--Initialize Jasper prior to webapps are loaded. Documentation at /docs/jasper-howto.html -->
  <Listener className="org.apache.catalina.core.JasperListener" />
  <!-- Prevent memory leaks due to use of particular java/javax APIs-->
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />

  <!-- Global JNDI resources
       Documentation at /docs/jndi-resources-howto.html
  -->
  <GlobalNamingResources>
    <!-- Editable user database that can also be used by
         UserDatabaseRealm to authenticate users
    -->
    <Resource name="UserDatabase" auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database that can be updated and saved"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat-users.xml" />
  </GlobalNamingResources>

  <!-- A "Service" is a collection of one or more "Connectors" that share
       a single "Container" Note:  A "Service" is not itself a "Container",
       so you may not define subcomponents such as "Valves" at this level.
       Documentation at /docs/config/service.html
   -->
  <Service name="Catalina">

    <!--The connectors can use a shared executor, you can define one or more named thread pools-->
    <!--
    <Executor name="tomcatThreadPool" namePrefix="catalina-exec-"
        maxThreads="150" minSpareThreads="4"/>
    -->


    <!-- A "Connector" represents an endpoint by which requests are received
         and responses are returned. Documentation at :
         Java HTTP Connector: /docs/config/http.html (blocking & non-blocking)
         Java AJP  Connector: /docs/config/ajp.html
         APR (HTTP/AJP) Connector: /docs/apr.html
         Define a non-SSL HTTP/1.1 Connector on port 8080
    -->
    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
    <!-- A "Connector" using the shared thread pool-->
    <!--
    <Connector executor="tomcatThreadPool"
               port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
    -->
    <!-- Define a SSL HTTP/1.1 Connector on port 8443
         This connector uses the JSSE configuration, when using APR, the
         connector should be using the OpenSSL style configuration
         described in the APR documentation -->
    <!--
    <Connector port="8443" protocol="HTTP/1.1" SSLEnabled="true"
               maxThreads="150" scheme="https" secure="true"
               clientAuth="false" sslProtocol="TLS" />
    -->

    <!-- Define an AJP 1.3 Connector on port 8009 -->
    <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />


    <!-- An Engine represents the entry point (within Catalina) that processes
         every request.  The Engine implementation for Tomcat stand alone
         analyzes the HTTP headers included with the request, and passes them
         on to the appropriate Host (virtual host).
         Documentation at /docs/config/engine.html -->

    <!-- You should set jvmRoute to support load-balancing via AJP ie :
    <Engine name="Catalina" defaultHost="localhost" jvmRoute="jvm1">
    -->
    <Engine name="Catalina" defaultHost="localhost">

      <!--For clustering, please take a look at documentation at:
          /docs/cluster-howto.html  (simple how to)
          /docs/config/cluster.html (reference documentation) -->
      <!--
      <Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"/>
      -->

      <!-- Use the LockOutRealm to prevent attempts to guess user passwords
           via a brute-force attack -->
      <Realm className="org.apache.catalina.realm.LockOutRealm">
        <!-- This Realm uses the UserDatabase configured in the global JNDI
             resources under the key "UserDatabase".  Any edits
             that are performed against this UserDatabase are immediately
             available for use by the Realm.  -->
        <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
               resourceName="UserDatabase"/>
      </Realm>

      <Host name="localhost"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">

        <!-- SingleSignOn valve, share authentication between web applications
             Documentation at: /docs/config/valve.html -->
        <!--
        <Valve className="org.apache.catalina.authenticator.SingleSignOn" />
        -->

        <!-- Access log processes all example.
             Documentation at: /docs/config/valve.html
             Note: The pattern used is equivalent to using pattern="common" -->
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log." suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />

      </Host>
    </Engine>
  </Service>
</Server>

```

这样经过对 xml 文件的解析将会产生 org.apache.catalina.core.StandardServer、org.apache.catalina.core.StandardService、org.apache.catalina.connector.Connector、org.apache.catalina.core.StandardEngine、org.apache.catalina.core.StandardHost、org.apache.catalina.core.StandardContext 等等一系列对象，这些对象从前到后前一个包含后一个对象的引用（一对一或一对多的关系）。

解析完 xml 之后关闭文件流，接着设置 StandardServer 对象（该对象在上面解析 xml 时）的 catalina 的引用为当前对象，这种对象间的双向引用在 Tomcat 的很多地方都会碰到。

接下来将调用 StandardServer 对象的 init 方法。

上面分析的是 Catalina 的 load 方法，上一篇文章里看到 Bootstrap 类启动时还会调用 Catalina 对象的 start 方法，代码如下：

```Java
     1	    /**
     2	     * Start a new server instance.
     3	     */
     4	    public void start() {
     5
     6	        if (getServer() == null) {
     7	            load();
     8	        }
     9
    10	        if (getServer() == null) {
    11	            log.fatal("Cannot start server. Server instance is not configured.");
    12	            return;
    13	        }
    14
    15	        long t1 = System.nanoTime();
    16
    17	        // Start the new server
    18	        try {
    19	            getServer().start();
    20	        } catch (LifecycleException e) {
    21	            log.fatal(sm.getString("catalina.serverStartFail"), e);
    22	            try {
    23	                getServer().destroy();
    24	            } catch (LifecycleException e1) {
    25	                log.debug("destroy() failed for failed Server ", e1);
    26	            }
    27	            return;
    28	        }
    29
    30	        long t2 = System.nanoTime();
    31	        if(log.isInfoEnabled()) {
    32	            log.info("Server startup in " + ((t2 - t1) / 1000000) + " ms");
    33	        }
    34
    35	        // Register shutdown hook
    36	        if (useShutdownHook) {
    37	            if (shutdownHook == null) {
    38	                shutdownHook = new CatalinaShutdownHook();
    39	            }
    40	            Runtime.getRuntime().addShutdownHook(shutdownHook);
    41
    42	            // If JULI is being used, disable JULI's shutdown hook since
    43	            // shutdown hooks run in parallel and log messages may be lost
    44	            // if JULI's hook completes before the CatalinaShutdownHook()
    45	            LogManager logManager = LogManager.getLogManager();
    46	            if (logManager instanceof ClassLoaderLogManager) {
    47	                ((ClassLoaderLogManager) logManager).setUseShutdownHook(
    48	                        false);
    49	            }
    50	        }
    51
    52	        if (await) {
    53	            await();
    54	            stop();
    55	        }
    56	    }

```

这里最主要的是调用 StandardServer 对象的 start 方法。

经过以上分析发现，在解析 xml 产生相应一系列对象后会顺序调用 StandardServer 对象的 init、start 方法，这里将会涉及到 Tomcat 的容器生命周期（ Lifecycle ），关于这点留到下一篇文章中分析。

