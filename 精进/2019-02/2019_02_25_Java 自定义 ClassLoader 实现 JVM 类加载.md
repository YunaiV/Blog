title: Java 自定义 ClassLoader 实现 JVM 类加载
date: 2019-02-25
tags:
categories: 精进
permalink: Fight/Java-custom-ClassLoader-implements-JVM-class-loading
author: 小小俊子
from_url: https://www.jianshu.com/p/3036b46f1188
wechat_url:

-------

摘要: 原创出处 https://www.jianshu.com/p/3036b46f1188 「小小俊子」欢迎转载，保留摘要，谢谢！

- [定义需要加载的类](http://www.iocoder.cn/Fight/Java-custom-ClassLoader-implements-JVM-class-loading/)
- [定义类加载器](http://www.iocoder.cn/Fight/Java-custom-ClassLoader-implements-JVM-class-loading/)
- [编译需要加载的类文件](http://www.iocoder.cn/Fight/Java-custom-ClassLoader-implements-JVM-class-loading/)
- [编译自定义的类加载器并支行程序](http://www.iocoder.cn/Fight/Java-custom-ClassLoader-implements-JVM-class-loading/)
- [总结](http://www.iocoder.cn/Fight/Java-custom-ClassLoader-implements-JVM-class-loading/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# 定义需要加载的类

为了能够实现类加载，并展示效果，定义一个Hello类，再为其定义一个sayHello()方法，加载Hello类之后，调用它的sayHello()方法。

```Java
public class Hello {
    public static void sayHello(){
        System.out.println("Hello,I am ....");
    }
}
```

# 定义类加载器

自定义加载器，需要继承ClassLoader,并重写里面的protected Class<?> findClass(String name) throws ClassNotFoundException方法。

```Java
import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;
import java.lang.reflect.Method;
import java.nio.MappedByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.channels.FileChannel.MapMode;

public class MyClassLoader extends ClassLoader {
    /**
     * 重写父类方法，返回一个Class对象
     * ClassLoader中对于这个方法的注释是:
     * This method should be overridden by class loader implementations
     */
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        Class clazz = null;
        String classFilename = name + ".class";
        File classFile = new File(classFilename);
        if (classFile.exists()) {
            try (FileChannel fileChannel = new FileInputStream(classFile)
                    .getChannel();) {
                MappedByteBuffer mappedByteBuffer = fileChannel
                        .map(MapMode.READ_ONLY, 0, fileChannel.size());
                byte[] b = mappedByteBuffer.array();
                clazz = defineClass(name, b, 0, b.length);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        if (clazz == null) {
            throw new ClassNotFoundException(name);
        }
        return clazz;
    }

    public static void main(String[] args) throws Exception{
        MyClassLoader myClassLoader = new MyClassLoader();
        Class clazz = myClassLoader.loadClass(args[0]);
        Method sayHello = clazz.getMethod("sayHello");
        sayHello.invoke(null, null);
    }
}
```

# 编译需要加载的类文件

类加载的时候加载的是字节码文件，所以需要预先把定义的Hello类编译成字节友文件。

```
javac Hello.java
```

验证字节码文件是否编译成功，利用二进制文件查看器查看我们编译之后的文件，样式如下：

```
0000000 177312 137272 000000 032000 016000 000012 000006 004416
0000020 007400 010000 000010 005021 011000 011400 000007 003424
0000040 012400 000001 036006 067151 072151 000476 001400 024450
0000060 000526 002000 067503 062544 000001 046017 067151 047145
0000100 066565 062542 052162 061141 062554 000001 071410 074541
0000120 062510 066154 000557 005000 067523 071165 062543 064506
0000140 062554 000001 044012 066145 067554 065056 073141 006141
0000160 003400 004000 000007 006026 013400 014000 000001 044017
0000200 066145 067554 044454 060440 020155 027056 027056 000007
0000220 006031 015000 015400 000001 044005 066145 067554 000001
0000240 065020 073141 027541 060554 063556 047457 065142 061545
0000260 000564 010000 060552 060566 066057 067141 027547 074523
0000300 072163 066545 000001 067403 072165 000001 046025 060552
0000320 060566 064457 027557 071120 067151 051564 071164 060545
0000340 035555 000001 065023 073141 027541 067551 050057 064562
0000360 072156 072123 062562 066541 000001 070007 064562 072156
0000400 067154 000001 024025 065114 073141 027541 060554 063556
0000420 051457 071164 067151 035547 053051 020400 002400 003000
0000440 000000 000000 001000 000400 003400 004000 000400 004400
0000460 000000 016400 000400 000400 000000 002400 133452 000400
0000500 000261 000000 000001 000012 000000 000006 000001 000000
0000520 000002 000011 000013 000010 000001 000011 000000 000045
0000540 000002 000000 000000 131011 001000 001422 000266 130404
0000560 000000 000400 005000 000000 005000 001000 000000 002000
0000600 004000 002400 000400 006000 000000 001000 006400
0000616
```

# 编译自定义的类加载器并支行程序

```Bash
 编译代码
javac MyClassLoader.java
 当然我们也可以同时编译我们所有的java源文件
javac *.java
```

执行成功之后，我们用下面的语句执行代码，测试是否成功，并查看结果

```Java
java MyClassLoader Hello
 运行结果
Hello,I am ....
```

当程序按照预期显示，就证明我自定义类加载器成功了。

# 总结

通过上面的程序代码，简单的实现JVM的类加载过程，知道了程序运行的一点流程。但是在编写的时候有如下坑需要注意

- 类文件不需要指定包，否则加载的时候我们需要额外的处理，把包中的"."替换成文件系统的路径"/"。
- 需要加载的Hello类中的反射调用的方法要用static修饰，这样invoke的时候第一个参数才可以使用null关键字代替，否则需要创建一个对应的类实例。
   官方文档中有这样一句话`If the underlying method is static, then the specified obj argument is ignored. It may be null.`