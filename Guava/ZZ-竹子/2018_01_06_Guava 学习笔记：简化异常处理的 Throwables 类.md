title: Guava 学习笔记：简化异常处理的 Throwables 类
date: 2018-01-06
tag: 
categories: Guava
permalink: Guava/peida/Throwables
author: 竹子
from_url: http://www.cnblogs.com/peida/p/Guava_Throwables.html
wechat_url: 

-------

摘要: 原创出处 http://www.cnblogs.com/peida/p/Guava_Throwables.html 「竹子」欢迎转载，保留摘要，谢谢！


-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

　　有时候, 当我们我们捕获异常, 并且像把这个异常传递到下一个try/catch块中。Guava提供了一个异常处理工具类, 可以简单地捕获和重新抛出多个异常。例如：

```Java
import java.io.IOException;
import org.junit.Test;
import com.google.common.base.Throwables;

public class ThrowablesTest {
    
    @Test
    public void testThrowables(){
        try {
            throw new Exception();
        } catch (Throwable t) {
            String ss = Throwables.getStackTraceAsString(t);
            System.out.println("ss:"+ss);
            Throwables.propagate(t);
        }
    }
    
    @Test
    public void call() throws IOException {
        try {
            throw new IOException();
        } catch (Throwable t) {
            Throwables.propagateIfInstanceOf(t, IOException.class);
            throw Throwables.propagate(t);
        }
    }    
}
```

 　　将检查异常转换成未检查异常,例如：

```Java
import java.io.InputStream;
import java.net.URL;
import org.junit.Test;
import com.google.common.base.Throwables;

public class ThrowablesTest {
    
    @Test
    public void testCheckException(){
        try {  
            URL url = new URL("http://ociweb.com");  
            final InputStream in = url.openStream();  
            // read from the input stream  
            in.close();  
        } catch (Throwable t) {  
            throw Throwables.propagate(t);  
        }  
    }
}
```

传递异常的常用方法：

1. RuntimeException propagate(Throwable)：把throwable包装成RuntimeException，用该方法保证异常传递，抛出一个RuntimeException异常
2. void propagateIfInstanceOf(Throwable, Class<X extends Exception>) throws X：当且仅当它是一个X的实例时，传递throwable
3. void propagateIfPossible(Throwable)：当且仅当它是一个RuntimeException和Error时，传递throwable
4. void propagateIfPossible(Throwable, Class<X extends Throwable>) throws X：当且仅当它是一个RuntimeException和Error时，或者是一个X的实例时，传递throwable。

　　使用实例：

```Java
import java.io.IOException;
import org.junit.Test;
import com.google.common.base.Throwables;

public class ThrowablesTest {    
    @Test
    public void testThrowables(){
        try {
            throw new Exception();
        } catch (Throwable t) {
            String ss = Throwables.getStackTraceAsString(t);
            System.out.println("ss:"+ss);
            Throwables.propagate(t);
        }
    }
    
    @Test
    public void call() throws IOException {
        try {
            throw new IOException();
        } catch (Throwable t) {
            Throwables.propagateIfInstanceOf(t, IOException.class);
            throw Throwables.propagate(t);
        }
    }
    
    public Void testPropagateIfPossible() throws Exception {
          try {
              throw new Exception();
          } catch (Throwable t) {
            Throwables.propagateIfPossible(t, Exception.class);
            Throwables.propagate(t);
          }

          return null;
    }
}
```

Guava的异常链处理方法：

1. Throwable getRootCause(Throwable)
2. List<Throwable> getCausalChain(Throwable)
3. String getStackTraceAsString(Throwable)