title: Tomcat 7 中的 JMX 使用（二）Dynamic MBean
date: 2018-01-19
tag: 
categories: Tomcat
permalink: Tomcat/yuliu/JMX-2-Dynamic-MBean
author: 预流
from_url: https://juejin.im/post/5aa4d4ba5188250f7a19e2fa
wechat_url: 

-------

摘要: 原创出处 https://juejin.im/post/5aa4d4ba5188250f7a19e2fa 「预流」欢迎转载，保留摘要，谢谢！

  - [加载`org.apache.tomcat.util.modeler.ManagedBean`对象](http://www.iocoder.cn/Tomcat/yuliu/JMX-2-Dynamic-MBean/)
  - [注册 MBean 实例](http://www.iocoder.cn/Tomcat/yuliu/JMX-2-Dynamic-MBean/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

如[上一篇文章](https://link.juejin.im?target=https%3A%2F%2Fjuejin.im%2Fpost%2F5a9f6be86fb9a028d82b4f3a)所见 Standard MBean 在 Tomcat 源码中的例子并不多，在 jconsole 中所看到的大量 MBean（如 Catalina 下的 Connector、Engine、Server、Service 等），实际上是动态 MBean（Dynamic MBean）。本文主要讲述 Tomcat 7 中如何通过动态 MBean 的方式构造 MBean 的。

接触过动态 MBean 的朋友一定知道，它的实例肯定要实现一个接口，即`javax.management.DynamicMBean`。实现这个接口就意味着同时要实现它下面的6个方法：

```Java
    public Object getAttribute(String attribute) throws AttributeNotFoundException,MBeanException, ReflectionException;

    public void setAttribute(Attribute attribute) throws AttributeNotFoundException,InvalidAttributeValueException, MBeanException, ReflectionException ;

    public AttributeList getAttributes(String[] attributes);

    public AttributeList setAttributes(AttributeList attributes);

    public Object invoke(String actionName, Object params[], String signature[]) throws MBeanException, ReflectionException ;

    public MBeanInfo getMBeanInfo();

```

通过实现这个通用接口，jvm 允许程序在运行时获取和设置 MBean 公开的属性和调用 MBean 上公开的方法。

上面简要介绍了动态 MBean 的实现方式，Tomcat 中的实际情况比这个要复杂，因为要生成很多种 MBean，如果每种类型都用代码写一个 MBean 就失去了动态 MBean 的威力，Tomcat 7 中实际是通过配置文件（即每个组件所在的包下面的`mbeans-descriptors.xml`）结合通用的动态 MBean（`org.apache.tomcat.util.modeler.BaseModelMBean`）、描述 MBean 配置信息的`org.apache.tomcat.util.modeler.ManagedBean`来简化 MBean 的构造。（实际就是用动态 MBean 实现了模型 MBean 的功能）

一般情况下动态 MBean 的产生分为两个阶段：

- 一、加载`org.apache.tomcat.util.modeler.ManagedBean`对象
- 二、注册 MBean 实例

## 加载`org.apache.tomcat.util.modeler.ManagedBean`对象

在 Tomcat 启动时加载的配置文件 server.xml 中有这么一行配置：

```XML
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />

```

因此在 Tomcat 启动时将加载这个类，在这个类中有一个静态成员变量 registry：

```Java
    /**
     * The configuration information registry for our managed beans.
     */
    protected static Registry registry = MBeanUtils.createRegistry();

```

也就是说类加载时 registry 就会获得 Registry 类的实例，这个 Registry 类很重要，在 MBean 的构造过程中将会多次涉及这个类里的方法。先看看`MBeanUtils.createRegistry()`方法：

```Java
     1	    /**
     2	     * Create and configure (if necessary) and return the registry of
     3	     * managed object descriptions.
     4	     */
     5	    public static synchronized Registry createRegistry() {
     6
     7	        if (registry == null) {
     8	            registry = Registry.getRegistry(null, null);
     9	            ClassLoader cl = MBeanUtils.class.getClassLoader();
    10
    11	            registry.loadDescriptors("org.apache.catalina.mbeans",  cl);
    12	            registry.loadDescriptors("org.apache.catalina.authenticator", cl);
    13	            registry.loadDescriptors("org.apache.catalina.core", cl);
    14	            registry.loadDescriptors("org.apache.catalina", cl);
    15	            registry.loadDescriptors("org.apache.catalina.deploy", cl);
    16	            registry.loadDescriptors("org.apache.catalina.loader", cl);
    17	            registry.loadDescriptors("org.apache.catalina.realm", cl);
    18	            registry.loadDescriptors("org.apache.catalina.session", cl);
    19	            registry.loadDescriptors("org.apache.catalina.startup", cl);
    20	            registry.loadDescriptors("org.apache.catalina.users", cl);
    21	            registry.loadDescriptors("org.apache.catalina.ha", cl);
    22	            registry.loadDescriptors("org.apache.catalina.connector", cl);
    23	            registry.loadDescriptors("org.apache.catalina.valves",  cl);
    24	        }
    25	        return (registry);
    26
    27	    }

```

注意第8行`Registry.getRegistry(null, null)`方法的调用，看下它的实现就会发现返回的实际是 Registry 类的静态变量，这种调用后面会多次看到。接着还需要看一下 MBeanUtils 类的 registry 的定义：

```Java
    /**
     * The configuration information registry for our managed beans.
     */
    private static Registry registry = createRegistry();

```

因为此时 MBeanUtils 类还没在JVM里面加载过，它的成员变量 registry 为 null ，所以会调用`Registry.getRegistry(null, null)`方法构造对象，接下来会多次调用 loadDescriptors 方法，以下面这一句代码为例：

```Java
registry.loadDescriptors("org.apache.catalina.connector", cl);

```

这里`org.apache.catalina.connector`实际上是一个 package 的路径全名，看下 loadDescriptors 方法：

```Java
     1	    /** Lookup the component descriptor in the package and
     2	     * in the parent packages.
     3	     *
     4	     * @param packageName
     5	     */
     6	    public void loadDescriptors( String packageName, ClassLoader classLoader  ) {
     7	        String res=packageName.replace( '.', '/');
     8
     9	        if( log.isTraceEnabled() ) {
    10	            log.trace("Finding descriptor " + res );
    11	        }
    12
    13	        if( searchedPaths.get( packageName ) != null ) {
    14	            return;
    15	        }
    16	        String descriptors=res + "/mbeans-descriptors.ser";
    17
    18	        URL dURL=classLoader.getResource( descriptors );
    19
    20	        if( dURL == null ) {
    21	            descriptors=res + "/mbeans-descriptors.xml";
    22	            dURL=classLoader.getResource( descriptors );
    23	        }
    24	        if( dURL == null ) {
    25	            return;
    26	        }
    27
    28	        log.debug( "Found " + dURL);
    29	        searchedPaths.put( packageName,  dURL );
    30	        try {
    31	            if( descriptors.endsWith(".xml" ))
    32	                loadDescriptors("MbeansDescriptorsDigesterSource", dURL, null);
    33	            else
    34	                loadDescriptors("MbeansDescriptorsSerSource", dURL, null);
    35	            return;
    36	        } catch(Exception ex ) {
    37	            log.error("Error loading " + dURL);
    38	        }
    39
    40	        return;
    41	    }

```

第13到15行是先在 Registry 类的缓存 searchedPaths 中查找是否已经加载了该 package 所对应的配置文件，如果没有在第16到18行会在该包路径下面查找是否有`mbeans-descriptors.ser`文件，没有则在第20到23行查找同路径下的`mbeans-descriptors.xml`文件。找到之后在第29行放入缓存 searchedPaths 。我们既然以`org.apache.catalina.connector`为例，则找到的是该路径下的`mbeans-descriptors.xml`。所以会接着执行第32行`loadDescriptors("MbeansDescriptorsDigesterSource", dURL, null)`：

```Java
    private void loadDescriptors(String sourceType, Object source,
            String param) throws Exception {
        load(sourceType, source, param);
    }

```

这段代码会执行 load 方法：

```Java
     1	    public List<ObjectName> load( String sourceType, Object source,
     2	            String param) throws Exception {
     3	        if( log.isTraceEnabled()) {
     4	            log.trace("load " + source );
     5	        }
     6	        String location=null;
     7	        String type=null;
     8	        Object inputsource=null;
     9
    10	        if( source instanceof URL ) {
    11	            URL url=(URL)source;
    12	            location=url.toString();
    13	            type=param;
    14	            inputsource=url.openStream();
    15	            if( sourceType == null ) {
    16	                sourceType = sourceTypeFromExt(location);
    17	            }
    18	        } else if( source instanceof File ) {
    19	            location=((File)source).getAbsolutePath();
    20	            inputsource=new FileInputStream((File)source);
    21	            type=param;
    22	            if( sourceType == null ) {
    23	                sourceType = sourceTypeFromExt(location);
    24	            }
    25	        } else if( source instanceof InputStream ) {
    26	            type=param;
    27	            inputsource=source;
    28	        } else if( source instanceof Class<?> ) {
    29	            location=((Class<?>)source).getName();
    30	            type=param;
    31	            inputsource=source;
    32	            if( sourceType== null ) {
    33	                sourceType="MbeansDescriptorsIntrospectionSource";
    34	            }
    35	        }
    36
    37	        if( sourceType==null ) {
    38	            sourceType="MbeansDescriptorsDigesterSource";
    39	        }
    40	        ModelerSource ds=getModelerSource(sourceType);
    41	        List<ObjectName> mbeans =
    42	            ds.loadDescriptors(this, type, inputsource);
    43
    44	        return mbeans;
    45	    }

```

第10到35行说穿是是为该方法适配多种数据源类型给 inputsource 变量赋上一个输入流。第40行会根据 sourceType 构造一个 ModelerSource 对象：

```Java
    private ModelerSource getModelerSource( String type )
            throws Exception
    {
        if( type==null ) type="MbeansDescriptorsDigesterSource";
        if( type.indexOf( ".") < 0 ) {
            type="org.apache.tomcat.util.modeler.modules." + type;
        }

        Class<?> c = Class.forName(type);
        ModelerSource ds=(ModelerSource)c.newInstance();
        return ds;
    }

```

上面看到 sourceType 传入的值是`MbeansDescriptorsDigesterSource`。所以 getModelerSource 方法最后返回的是`org.apache.tomcat.util.modeler.modules.MbeansDescriptorsDigesterSource`类的一个实例。

最后执行该 ModelerSource 对象的`loadDescriptors(this, type, inputsource)` 方法，因为该方法是一个抽象方法，所以这里实际执行的`org.apache.tomcat.util.modeler.modules.MbeansDescriptorsDigesterSource`类的 loadDescriptors 方法：

```Java
    @Override
    public List<ObjectName> loadDescriptors( Registry registry, String type,
            Object source) throws Exception {
        setRegistry(registry);
        setType(type);
        setSource(source);
        execute();
        return mbeans;
    }

```

前三个 set 方法毋庸多言，关键是最后的 execute 方法：

```Java
     1	    public void execute() throws Exception {
     2	        if (registry == null) {
     3	            registry = Registry.getRegistry(null, null);
     4	        }
     5
     6	        InputStream stream = (InputStream) source;
     7
     8	        if (digester == null) {
     9	            digester = createDigester();
    10	        }
    11	        ArrayList<ManagedBean> loadedMbeans = new ArrayList<ManagedBean>();
    12
    13	        synchronized (digester) {
    14
    15	            // Process the input file to configure our registry
    16	            try {
    17	                // Push our registry object onto the stack
    18	                digester.push(loadedMbeans);
    19	                digester.parse(stream);
    20	            } catch (Exception e) {
    21	                log.error("Error digesting Registry data", e);
    22	                throw e;
    23	            } finally {
    24	                digester.reset();
    25	            }
    26
    27	        }
    28	        Iterator<ManagedBean> iter = loadedMbeans.iterator();
    29	        while (iter.hasNext()) {
    30	            registry.addManagedBean(iter.next());
    31	        }
    32	    }
    33	}

```

在第3行又看到了前面提到的`Registry.getRegistry(null, null)`方法，这里就是获取 Registry 的静态成员的引用。这段方法作用就是对 source 进行一次 Digester 解析，如果还不了解 Digester 解析，可以看看之前文章：[Tomcat 7 启动分析（三）Digester 的使用](https://link.juejin.im?target=https%3A%2F%2Fjuejin.im%2Fpost%2F5a6d1ff6f265da3e243bc1de)。注意第18行 digester 的顶层对象是 loadedMbeans ，重点看下第9行 createDigester() 方法的调用：

```Java
     1	    protected static Digester createDigester() {
     2
     3	        Digester digester = new Digester();
     4	        digester.setNamespaceAware(false);
     5	        digester.setValidating(false);
     6	        URL url = Registry.getRegistry(null, null).getClass().getResource
     7	            ("/org/apache/tomcat/util/modeler/mbeans-descriptors.dtd");
     8	        digester.register
     9	            ("-//Apache Software Foundation//DTD Model MBeans Configuration File",
    10	                url.toString());
    11
    12	        // Configure the parsing rules
    13	        digester.addObjectCreate
    14	            ("mbeans-descriptors/mbean",
    15	            "org.apache.tomcat.util.modeler.ManagedBean");
    16	        digester.addSetProperties
    17	            ("mbeans-descriptors/mbean");
    18	        digester.addSetNext
    19	            ("mbeans-descriptors/mbean",
    20	                "add",
    21	            "java.lang.Object");
    22
    23	        digester.addObjectCreate
    24	            ("mbeans-descriptors/mbean/attribute",
    25	            "org.apache.tomcat.util.modeler.AttributeInfo");
    26	        digester.addSetProperties
    27	            ("mbeans-descriptors/mbean/attribute");
    28	        digester.addSetNext
    29	            ("mbeans-descriptors/mbean/attribute",
    30	                "addAttribute",
    31	            "org.apache.tomcat.util.modeler.AttributeInfo");
    32
    33	    ......
    34
    35	        return digester;
    36	    }

```

上面这段代码其实很长，但绝大部分都是模板代码，理解几句的含义后面代码都很相似。这就是一个xml文件的解析，第13到15行是值在碰到 xml 文件的`mbeans-descriptors`节点的子节点 mbean 时构造一个`org.apache.tomcat.util.modeler.ManagedBean`对象，第16到17行是读取该节点属性值填充到 ManagedBean 对象的 pojo 属性中，第18到21行以 ManagedBean 对象为入参调用上一段代码分析提到的 loadedMbeans 对象的add方法。类似的，第23到31行是指在碰到`mbeans-descriptors/mbean/attribute`节点时构造`org.apache.tomcat.util.modeler.AttributeInfo`对象，填充pojo属性，并调用父节点构造的对象（即 ManagedBean 对象）的 addAttribute 方法。其它代码类似，不再赘述。

接回到上面 MbeansDescriptorsDigesterSource 类的 execute 方法第28到31行，在 Digester 解析完成之后迭代 loadedMbeans 对象，并调用`registry.addManagedBean`方法将这些 ManagedBean 添加到 registry 中。这样，一次`registry.loadDescriptors("org.apache.catalina.connector", cl)`调用就会加载该包路径下相对应的 ManagedBean 对象到 Registry 类的成员变量中。

下面的时序图列出从 GlobalResourcesLifecycleListener 类加载其静态成员变量 registry 到 Registry 类加载完相应包所对应的 ManagedBean 的关键方法调用过程：

![时序图](https://user-gold-cdn.xitu.io/2018/3/11/16213f00d7a92db2?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)



## 注册 MBean 实例

### 查找ManagedBean

上面说的是一个 ManagedBean 的加载过程，但它不是一个 MBean ，可以把它看作一个描述 MBean 的配置信息的对象，以前面提到的`org.apache.catalina.connector`为例，在 Tomcat 7 的默认配置启动后实际上有两个 Connector 实例，因为在`server.xml`中配置了两条 connector 节点：

```XML
    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />

    <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />

```

所对应 jconsole 中会看到两个相应的 MBean 对象：

![img](https://user-gold-cdn.xitu.io/2018/3/11/16213f32d49c7adb?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

 但 ManageBean 实际只是加载了一次。了解了 ManagedBean 与 MBean 的对应关系，接下来看看一个 MBean 是怎么注册到 JVM 中的。



看过前面 Tomcat 启动分析的朋友知道容器各组件在启动过程中会相继调用它们的 initInternal()、startInternal() 两个方法，还是以上面提到的 Connector 组件为例，Tomcat 启动时解析`server.xml`文件过程中碰到 Connector 节点配置会构造`org.apache.catalina.connector.Connector`对象并调用它的 initInternal 方法：

```Java
    @Override
    protected void initInternal() throws LifecycleException {

        super.initInternal();

```

在这个方法的开始会调用它的父类`org.apache.catalina.util.LifecycleMBeanBase`的 initInternal 方法：

```Java
    private ObjectName oname = null;
    protected MBeanServer mserver = null;

    /**
     * Sub-classes wishing to perform additional initialization should override
     * this method, ensuring that super.initInternal() is the first call in the
     * overriding method.
     */
    @Override
    protected void initInternal() throws LifecycleException {

        // If oname is not null then registration has already happened via
        // preRegister().
        if (oname == null) {
            mserver = Registry.getRegistry(null, null).getMBeanServer();

            oname = register(this, getObjectNameKeyProperties());
        }
    }

```

先获取 MBeanServer 的实例，接着调用内部的 register 方法，将当前对象注册到 MBeanServer 中，看下 register 方法：

```Java
     1	    protected final ObjectName register(Object obj,
     2	            String objectNameKeyProperties) {
     3
     4	        // Construct an object name with the right domain
     5	        StringBuilder name = new StringBuilder(getDomain());
     6	        name.append(':');
     7	        name.append(objectNameKeyProperties);
     8
     9	        ObjectName on = null;
    10
    11	        try {
    12	            on = new ObjectName(name.toString());
    13
    14	            Registry.getRegistry(null, null).registerComponent(obj, on, null);
    15	        } catch (MalformedObjectNameException e) {
    16	            log.warn(sm.getString("lifecycleMBeanBase.registerFail", obj, name),
    17	                    e);
    18	        } catch (Exception e) {
    19	            log.warn(sm.getString("lifecycleMBeanBase.registerFail", obj, name),
    20	                    e);
    21	        }
    22
    23	        return on;
    24	    }

```

重点是第14行调用 Registry 类的 registerComponent 方法来注册：

```Java
     1	    public void registerComponent(Object bean, ObjectName oname, String type)
     2	           throws Exception
     3	    {
     4	        if( log.isDebugEnabled() ) {
     5	            log.debug( "Managed= "+ oname);
     6	        }
     7
     8	        if( bean ==null ) {
     9	            log.error("Null component " + oname );
    10	            return;
    11	        }
    12
    13	        try {
    14	            if( type==null ) {
    15	                type=bean.getClass().getName();
    16	            }
    17
    18	            ManagedBean managed = findManagedBean(bean.getClass(), type);
    19
    20	            // The real mbean is created and registered
    21	            DynamicMBean mbean = managed.createMBean(bean);
    22
    23	            if(  getMBeanServer().isRegistered( oname )) {
    24	                if( log.isDebugEnabled()) {
    25	                    log.debug("Unregistering existing component " + oname );
    26	                }
    27	                getMBeanServer().unregisterMBean( oname );
    28	            }
    29
    30	            getMBeanServer().registerMBean( mbean, oname);
    31	        } catch( Exception ex) {
    32	            log.error("Error registering " + oname, ex );
    33	            throw ex;
    34	        }
    35	    }

```

在第18行根据当前要注册的对象（即 Connector 对象）的类型查找 ManagedBean ，沿着这个方法追会发现依次调用了一堆同名的findManagedBean方法，一直到findManagedBean(String name)：

```Java
    public ManagedBean findManagedBean(String name) {
        // XXX Group ?? Use Group + Type
        ManagedBean mb = descriptors.get(name);
        if( mb==null )
            mb = descriptorsByClass.get(name);
        return mb;
    }

```

这段代码意思是依次从 Registry 类的静态成员变量 descriptors、descriptorsByClass 中查找相应 ManagedBean 。那这两个 HashMap 是什么时候put值进去的呢？答案就在上一部分分析的最后加载 ManagedBean 时最终调用 Registry 类的 addManagedBean 方法：

```Java
    public void addManagedBean(ManagedBean bean) {
        // XXX Use group + name
        descriptors.put(bean.getName(), bean);
        if( bean.getType() != null ) {
            descriptorsByClass.put( bean.getType(), bean );
        }
    }

```

### 创建 DynamicMBean

在上面的 registerComponent 方法的第21行调用查找到的 ManagedBean 对象的 createMBean 方法来获取实际的 DynamicMBean 对象：

```Java
    public DynamicMBean createMBean(Object instance)
        throws InstanceNotFoundException,
        MBeanException, RuntimeOperationsException {

        BaseModelMBean mbean = null;

        // Load the ModelMBean implementation class
        if(getClassName().equals(BASE_MBEAN)) {
            // Skip introspection
            mbean = new BaseModelMBean();
        } else {
            Class<?> clazz = null;
            Exception ex = null;
            try {
                clazz = Class.forName(getClassName());
            } catch (Exception e) {
            }

            if( clazz==null ) {
                try {
                    ClassLoader cl= Thread.currentThread().getContextClassLoader();
                    if ( cl != null)
                        clazz= cl.loadClass(getClassName());
                } catch (Exception e) {
                    ex=e;
                }
            }

            if( clazz==null) {
                throw new MBeanException
                    (ex, "Cannot load ModelMBean class " + getClassName());
            }
            try {
                // Stupid - this will set the default minfo first....
                mbean = (BaseModelMBean) clazz.newInstance();
            } catch (RuntimeOperationsException e) {
                throw e;
            } catch (Exception e) {
                throw new MBeanException
                    (e, "Cannot instantiate ModelMBean of class " +
                     getClassName());
            }
        }

        mbean.setManagedBean(this);

        // Set the managed resource (if any)
        try {
            if (instance != null)
                mbean.setManagedResource(instance, "ObjectReference");
        } catch (InstanceNotFoundException e) {
            throw e;
        }
        return (mbean);

    }

```

这段代码看起来长，仔细分析实际就是根据 ManagedBean 对象的 getClassName 方法返回的值通过反射等方式来构造一个对象返回。而 getClassName 方法调用的实际就是上面提到的 Digester 解析时构造 ManagedBean 对象时自动从 xml 文件中读取并填充的 pojo 属性 className，以现在所说的 Connector 为例，在`mbeans-descriptors.xml`中的配置：

```XML
  <mbean         name="CoyoteConnector"
            className="org.apache.catalina.mbeans.ConnectorMBean"
          description="Implementation of a Coyote connector"
               domain="Catalina"
                group="Connector"
                 type="org.apache.catalina.connector.Connector">

```

所以此时构造返回的是一个`org.apache.catalina.mbeans.ConnectorMBean`对象。可以看到这个类的继承关系，它的父类是`org.apache.catalina.mbeans.ClassNameMBean`，它父类的父类就是`org.apache.tomcat.util.modeler.BaseModelMBean`，从这三种类中可以分别看到通常的动态 MBean 要实现的6个方法的定义，有兴趣的可以继续研究这些方法的实现，实际上它们都用到了什么所说的 ManagedBean 对象的相关方法，因为与该 MBean 要暴露的方法、操作的描述信息都是在加载相应的 ManagedBean 对象时读取的，所以动态 MBean 的实现必然也是需要调用它们的。

### 注册 DynamicMBean

在上面的 registerComponent 方法的第30行 getMBeanServer().registerMBean( mbean, oname) ，这就是将该 DynamicMBean 对象注册到 MBeanServer 中。

下面的时序图列出从 Connector 的 initInternal 方法到注册 MBean 的关键方法调用过程：

![img](https://user-gold-cdn.xitu.io/2018/3/11/16213fa15f4b2d7c?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)