title: Spring AOP,AspectJ,CGLIB 有点晕
date: 2018-10-14
tags:
categories: 精进
permalink: Fight/Spring-AOP-AspectJ-and-CGLIB-are-a-bit-dizzy
author: dolphin叔叔
from_url: https://www.jianshu.com/p/fe8d1e8bd63e
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247485373&idx=1&sn=40ef0ac1b6c1f0bd6cb90f336c02ba14&chksm=fa49780ccd3ef11a0e62ebb8520eb881a9998548e79782bfc85b641d42ee73e904737fff48a1&token=1125060202&lang=zh_CN#rd

----

摘要: 原创出处 https://www.jianshu.com/p/fe8d1e8bd63e 「dolphin叔叔」欢迎转载，保留摘要，谢谢！

- [1. 先说说AspectJ](http://www.iocoder.cn/Fight/Spring-AOP-AspectJ-and-CGLIB-are-a-bit-dizzy/)
  - [1.1 AspectJ 安装](http://www.iocoder.cn/Fight/Spring-AOP-AspectJ-and-CGLIB-are-a-bit-dizzy/)
  - [1.2 AspectJ HelloWorld 实现](http://www.iocoder.cn/Fight/Spring-AOP-AspectJ-and-CGLIB-are-a-bit-dizzy/)
- [2. 再谈 Spring AOP](http://www.iocoder.cn/Fight/Spring-AOP-AspectJ-and-CGLIB-are-a-bit-dizzy/)
  - [2.1 Spring AOP HelloWorld 实现](http://www.iocoder.cn/Fight/Spring-AOP-AspectJ-and-CGLIB-are-a-bit-dizzy/)
  - [2.2 Hello World 后的总结](http://www.iocoder.cn/Fight/Spring-AOP-AspectJ-and-CGLIB-are-a-bit-dizzy/)
- [3. 最后说说CGLIB](http://www.iocoder.cn/Fight/Spring-AOP-AspectJ-and-CGLIB-are-a-bit-dizzy/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

> AOP（Aspect Orient Programming），作为面向对象编程的一种补充，广泛应用于处理一些具有横切性质的系统级服务，如事务管理、安全检查、缓存、对象池管理等。AOP 实现的关键就在于 AOP 框架自动创建的 AOP 代理，AOP 代理则可分为静态代理和动态代理两大类，其中静态代理是指使用 AOP 框架提供的命令进行编译，从而在编译阶段就可生成 AOP 代理类，因此也称为编译时增强；而动态代理则在运行时借助于 JDK 动态代理、CGLIB 等在内存中“临时”生成 AOP 动态代理类，因此也被称为运行时增强。

# 1. 先说说AspectJ

在今天之前，我还以为AspectJ 是Spring的一部分，因为我们谈到Spring AOP一般都会提到AspectJ。原来AspectJ是一套独立的面向切面编程的解决方案。下面我们抛开Spring，单纯的看看AspectJ。

## 1.1 AspectJ 安装
    
AspectJ 下载地址([http://www.eclipse.org/aspectj/downloads.php](https://link.jianshu.com?t=http://www.eclipse.org/aspectj/downloads.php))。
    
下载AspectJ  jar包，然后双击安装。安装好的目录结构为:

* bin：存放了 aj、aj5、ajc、ajdoc、ajbrowser 等命令，其中 ajc 命令最常用，它的作用类似于 javac
* doc：存放了 AspectJ 的使用说明、参考手册、API 文档等文档
* lib：该路径下的 4 个 JAR 文件是 AspectJ 的核心类库

## 1.2 AspectJ HelloWorld 实现

```Java
// 业务组件  SayHelloService

package com.ywsc.fenfenzhong.aspectj.learn;
public class SayHelloService {
    public void say(){
        System.out.print("Hello  AspectJ");
    }
}
```

需要来了，在需要在调用say()方法之后，需要记录日志。那就是通过AspectJ的后置增强吧。

LogAspect 日志记录组件，实现对`com.ywsc.fenfenzhong.aspectj.learn.SayHelloService` 后置增强

```Java
package com.ywsc.fenfenzhong.aspectj.learn;
public aspect LogAspect {
    pointcut logPointcut():execution(void SayHelloService.say());
    after():logPointcut(){
         System.out.println("记录日志 ...");
    }
}
```

编译 SayHelloService：

```BASH
执行命令   ajc -d . SayHelloService.java LogAspect.java
生成 SayHelloService.class
执行命令    java SayHelloService
输出  Hello AspectJ  记录日志
```

* ajc.exe 可以理解为 javac.exe 命令，都用于编译 Java 程序，区别是 ajc.exe 命令可识别 AspectJ 的语法；我们可以将 ajc.exe 当成一个增强版的 javac.exe 命令.执行ajc命令后的 SayHelloService.class 文件不是由原来的 SayHelloService.java 文件编译得到的，该 SayHelloService.class 里新增了打印日志的内容——这表明 AspectJ 在编译时“自动”编译得到了一个新类，这个新类增强了原有的 SayHelloService.java 类的功能，因此 AspectJ 通常被称为编译时增强的 AOP 框架。


与 AspectJ 相对的还有另外一种 AOP 框架，它不需要在编译时对目标类进行增强，而是运行时生成目标类的代理类，该代理类要么与目标类实现相同的接口，要么是目标类的子类——总之，代理类的实例可作为目标类的实例来使用。一般来说，编译时增强的 AOP 框架在性能上更有优势——因为运行时动态增强的 AOP 框架需要每次运行时都进行动态增强。

# 2. 再谈 Spring AOP

Spring AOP也是对目标类增强，生成代理类。但是与AspectJ的最大区别在于---Spring AOP的运行时增强，而AspectJ是编译时增强。

曾经以为AspectJ是Spring AOP一部分，是因为Spring AOP使用了AspectJ的Annotation。使用了Aspect来定义切面,使用Pointcut来定义切入点，使用Advice来定义增强处理。虽然使用了Aspect的Annotation，但是并没有使用它的编译器和织入器。其实现原理是JDK 动态代理，在运行时生成代理类。

为了启用 Spring 对 @AspectJ 方面配置的支持，并保证 Spring 容器中的目标 Bean 被一个或多个方面自动增强，必须在 Spring 配置文件中添加如下配置

```XML
<aop:aspectj-autoproxy/>
```

当启动了 @AspectJ 支持后，在 Spring 容器中配置一个带 @Aspect 注释的 Bean，Spring 将会自动识别该 Bean，并将该 Bean 作为方面 Bean 处理。方面Bean与普通 Bean 没有任何区别，一样使用 `<bean.../>` 元素进行配置，一样支持使用依赖注入来配置属性值。

## 2.1 Spring AOP HelloWorld 实现

使用Spring AOP的改写 Hello World的例子。

```Java
// 业务组件  SayHelloService

package com.ywsc.fenfenzhong.aspectj.learn;
import org.springframework.stereotype.Component;
@Component
public class SayHelloService {
    public void say(){
        System.out.print("Hello  AspectJ");
    }
}
```

做后置增强的日志处理。

```Java
package com.ywsc.fenfenzhong.aspectj.learn;
import org.aspectj.lang.annotation.After;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class LogAspect {
     @After("execution(* com.ywsc.fenfenzhong.aspectj.learn.SayHelloService.*(..))")
     public void log(){
         System.out.println("记录日志 ...");
     }
}
package com.ywsc.fenfenzhong.mongodb;
import com.ywsc.fenfenzhong.aspectj.learn.SayHelloService;
public class TestCase {
    public static void main(String[] args) {
        SayHelloService sayHelloService = ApplicationUtil.getContext().getBean(SayHelloService.class);
        sayHelloService.say();
    }
}
```

输出结果

```Java
Hello  AspectJ
```

记录日志...

## 2.2 Hello World 后的总结

AOP 代理 = 原来的业务类+增强处理。

这个生成AOP 代理由 Spring 的 IoC 容器负责生成。也由 IoC 容器负责管理。因此，AOP 代理可以直接使用容器中的其他 Bean 实例作为目标，这种关系可由 IoC 容器的依赖注入提供。回顾Hello World的例子，其中程序员参与的只有 3 个部分：

* 定义普通业务组件。
* 定义切入点，一个切入点可能横切多个业务组件。
* 定义增强处理，增强处理就是在 AOP 框架为普通业务组件织入的处理动作。

# 3. 最后说说CGLIB

CGLIB（Code Generation Library）它是一个代码生成类库。它可以在运行时候动态是生成某个类的子类。代理模式为要访问的目标对象提供了一种途径，当访问对象时，它引入了一个间接的层。JDK自从1.3版本开始，就引入了动态代理，并且经常被用来动态地创建代理。JDK的动态代理用起来非常简单，唯一限制便是使用动态代理的对象必须实现一个或多个接口。而CGLIB缺不必有此限制。要想Spring AOP 通过CGLIB生成代理，只需要在Spring 的配置文件引入

```XML
<aop:aspectj-autoproxy proxy-target-class="true"/>
```

CGLIB包的底层是通过使用一个小而快的字节码处理框架ASM(Java字节码操控框架)，来转换字节码并生成新的类。由于没有了解过class 文件和字节码，因而也就写不下去了。

也许学习下来最大的收获便是 弄清楚了 AspectJ 和 Spring AOP 在实现上几乎无关。