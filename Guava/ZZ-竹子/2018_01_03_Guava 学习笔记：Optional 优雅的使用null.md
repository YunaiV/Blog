title: Guava 学习笔记：Optional 优雅的使用null
date: 2018-01-03
tag: 
categories: Guava
permalink: Guava/peida/Optional
author: 竹子
from_url: http://www.cnblogs.com/peida/archive/2013/06/14/Guava_Optional.html
wechat_url: 

-------

摘要: 原创出处 http://www.cnblogs.com/peida/archive/2013/06/14/Guava_Optional.html 「竹子」欢迎转载，保留摘要，谢谢！

- [null 代表不确定的对象](http://www.iocoder.cn/Guava/peida/Optional/)
- [null 本身不是对象，也不是 Objcet 的实例](http://www.iocoder.cn/Guava/peida/Optional/)
- [null 对象的使用](http://www.iocoder.cn/Guava/peida/Optional/)
- [Guava 的 Optional](http://www.iocoder.cn/Guava/peida/Optional/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

在我们学习和使用Guava的Optional之前，我们需要来了解一下Java中null。因为，只有我们深入的了解了null的相关知识，我们才能更加深入体会领悟到Guava的Optional设计和使用上的优雅和简单。

# null 代表不确定的对象

　　Java中，null是一个关键字，用来标识一个不确定的对象。因此可以将null赋给引用类型变量，但不可以将null赋给基本类型变量。

　　Java中，变量的使用都遵循一个原则：先定义，并且初始化后，才可以使用。例如如下代码中，我们不能定义int age后，不给age指定值，就去打印age的值。这条对对于引用类型变量也是适用的（String name也同样适用），在编译的时候就会提示为初始化。

```Java
public class NullTest {
    public static void testNull(){
        int age;
        System.out.println("user age:"+age);

        long money;
        money=10L;
        System.out.println("user money"+money);

        String name;
        System.out.println("user name:"+name);
    }
}
```

　　在Java中，Java默认给变量赋值：在定义变量的时候，如果定义后没有给变量赋值，则Java在运行时会自动给变量赋值。赋值原则是整数类型int、byte、short、long的自动赋值为0，带小数点的float、double自动赋值为0.0，boolean的自动赋值为false，其他各供引用类型变量自动赋值为null。上面代码会变为如下可运行代码：

```Java
public class NullTest {
    public static void testNull(){
        int age = 0;
        System.out.println("user Age:"+age);

        long money;
        money=10L;
        System.out.println("user money"+money);

        String name = null;
        System.out.println("user name:"+name);
    }
}
```

# null 本身不是对象，也不是 Objcet 的实例

　　null只是一个关键字，用来标识一个不确定的对象，他既不是对象，也不是Objcet对象的实例。下面我们通过代码确定一下null是不是Object对象实例：

```Java
public class NullTest {
    public static void main(String[] args) {
        testNullObject();
    }

    public static void testNullObject() {
        if (null instanceof java.lang.Object) {
            System.out.println("null属于java.lang.Object类型");
        } else {
            System.out.println("null不属于java.lang.Object类型");
        }
    }
}
```

　　运行上面代码，输出：null不属于java.lang.Object类型，可见，null对象不是Object对象的实例。

------

# null 对象的使用

**1. 常见使用场景：**

　　有时候，我们定义一个引用类型变量，在刚开始的时候，无法给出一个确定的值，但是不指定值，程序可能会在try语句块中初始化值。这时候，我们下面使用变量的时候就会报错。这时候，可以先给变量指定一个null值，问题就解决了。例如：

```Java
 Connection conn = null;
 try {
　　conn = DriverManager.getConnection("url", "user", "password");
 } catch (SQLException e) {
　　 e.printStackTrace();
 }
 String catalog = conn.getCatalog();
```

　　如果刚开始的时候不指定conn = null，则最后一句就会报错。

**2. 容器类型与null：**
​     
* List：允许重复元素，可以加入任意多个null。
* Set：不允许重复元素，最多可以加入一个null。
* Map：Map的key最多可以加入一个null，value字段没有限制。
* 数组：基本类型数组，定义后，如果不给定初始值，则java运行时会自动给定值。引用类型数组，不给定初始值，则所有的元素值为null。

**3.null的其他作用**
    
* 1>、判断一个引用类型数据是否null。用==来判断。
* 2>、释放内存，让一个非null的引用类型变量指向null。这样这个对象就不再被任何对象应用了。等待JVM垃圾回收机制去回收。

**4.null的使用建议：**

* 1>. 在Set或者Map中使用null作为键值指向的value，千万别这么用。很显然，在Set和Map的查询操作中，将null作为特殊的例子可以使查询结果更浅显易懂。
* 2>. 在Map中包含value是null值的键值对，你应该把这种键值对移出map，使用一个独立的Set来包含所有null或者非null的键。很容易混淆的是，一个Map是不是包含value是　null的key，还是说这个Map中没有这样的键值对。最好的办法就是把这类key值分立开来，并且好好想想到底一个value是null的键值对对于你的程序来说到底意味着什么。
* 3>. 在列表中使用null，并且这个列表的数据是稀疏的，或许你最好应该使用一个Map<Integer,E>字典来代替这个列表。因为字典更高效，并且也许更加精确的符合你潜意识里对程序的需求。
* 4>. 想象一下如果有一种自然的“空对象”可以使用，比方说对于枚举类型，添加一个枚举常数实例，这个实例用来表示你想用null值所表示的情形。比如：Java.math.RoundingMode有一个常数实例UNNECESSARY来表示“不需要四舍五入”，任何精度计算的方法若传以RoundingMode.UNNECESSARY为参数来计算，必然抛出一个异常来表示不需要舍取精度。
　　

**5.问题和困惑:**

　　首先，对于null的随意使用会一系列难以预料的问题。通过对大量代码的研究和分析，我们发现大概95%以上的集合类默认并不接受null值，如果有null值将被放入集合中，代码会立刻中断并报错而不是默认存储null值，对于开发来说，这样能够更加容易的定位程序出错的地方。
　　
　　另外，null值是一种令人不满的模糊含义。有的时候会产生二义性，这时候我们就很难搞清楚具体的意思，如果程序返回一个null值，其代表的含义到底是什么，例如：Map.get(key)若返回value值为null，其代表的含义可能是该键指向的value值是null，亦或者该键在map中并不存在。null值可以表示失败，可以表示成功，几乎可以表示任何情况。用其它一些值(而不是null值)可以让你的代码表述的含义更清晰。
　　
　　反过来说，使用null值在有些情况下是一种正确的选择，因为从内存消耗和效率方面考虑，使用null更加廉价，而且在对象数组中出现null也是不可避免的。但是在程序代码中，比方说在函数库中，null值的使用会变成导致误解的元凶，也会导致一些莫名的，模糊的，很难修正的问题。就像上述map的例子，字典返回null可以代表的是该键指向的值存在且为空，或者也可以代表字典中没有这个键。关键在于，null值不能指明到底null代表了什么含义。

# Guava 的 Optional

　　大多数情况下程序员使用null是为了表示某种不存在的意思，也许应该有一个value，但是这个value是空或者这个value找不到。比方说，在用不存在的key值从map中取　　value，Map.get返回null表示没有该map中不包含这个key。　

　　若T类型数据可以为null，Optional<T>是用来以非空值替代T数据类型的一种方法。一个Optional对象可以包含一个非空的T引用（这种情况下我们称之为“存在的”）或者不包含任何东西（这种情况下我们称之为“空缺的”）。但Optional从来不会包含对null值的引用。

```Java
import com.google.common.base.Optional;

public class OptionalTest {

    public void testOptional() throws Exception {
        Optional<Integer> possible=Optional.of(6);
        if(possible.isPresent()){
            System.out.println("possible isPresent:"+possible.isPresent());
            System.out.println("possible value:"+possible.get());
        }
    }
}
```



　　由于这些原因，Guava库设计了Optional来解决null的问题。许多Guava的工具被设计成如果有null值存在即刻报错而不是只要上下文接受处理null值就默认使用null值继续运行。而且，Guava提供了Optional等一些工具让你在不得不使用null值的时候，可以更加简便的使用null并帮助你避免直接使用null。

　　Optional<T>的最常用价值在于，例如，假设一个方法返回某一个数据类型，调用这个方法的代码来根据这个方法的返回值来做下一步的动作，若该方法可以返回一个null值表示成功，或者表示失败，在这里看来都是意义含糊的，所以使用Optional<T>作为返回值，则后续代码可以通过isPresent()来判断是否返回了期望的值（原本期望返回null或者返回不为null，其意义不清晰），并且可以使用get()来获得实际的返回值。


**Optional方法说明和使用实例：**

**1.常用静态方法：**

Optional.of(T)：获得一个Optional对象，其内部包含了一个非null的T数据类型实例，若T=null，则立刻报错。

Optional.absent()：获得一个Optional对象，其内部包含了空值

Optional.fromNullable(T)：将一个T的实例转换为Optional对象，T的实例可以不为空，也可以为空[Optional.fromNullable(null)，和Optional.absent()等价。

使用实例如下：

```Java
import com.google.common.base.Optional;


public class OptionalTest {

    @Test
    public void testOptional() throws Exception {
        Optional<Integer> possible=Optional.of(6);
        Optional<Integer> absentOpt=Optional.absent();
        Optional<Integer> NullableOpt=Optional.fromNullable(null);
        Optional<Integer> NoNullableOpt=Optional.fromNullable(10);
        if(possible.isPresent()){
            System.out.println("possible isPresent:"+possible.isPresent());
            System.out.println("possible value:"+possible.get());
        }
        if(absentOpt.isPresent()){
            System.out.println("absentOpt isPresent:"+absentOpt.isPresent()); ;
        }
        if(NullableOpt.isPresent()){
            System.out.println("fromNullableOpt isPresent:"+NullableOpt.isPresent()); ;
        }
        if(NoNullableOpt.isPresent()){
            System.out.println("NoNullableOpt isPresent:"+NoNullableOpt.isPresent()); ;
        }
    }
}
```

**2.实例方法：**

* 1>. boolean isPresent()：如果Optional包含的T实例不为null，则返回true；若T实例为null，返回false
* 2>. T get()：返回Optional包含的T实例，该T实例必须不为空；否则，对包含null的Optional实例调用get()会抛出一个IllegalStateException异常
* 3>. T or(T)：若Optional实例中包含了传入的T的相同实例，返回Optional包含的该T实例，否则返回输入的T实例作为默认值
* 4>. T orNull()：返回Optional实例中包含的非空T实例，如果Optional中包含的是空值，返回null，逆操作是fromNullable()
* 5>. Set<T> asSet()：返回一个不可修改的Set，该Set中包含Optional实例中包含的所有非空存在的T实例，且在该Set中，每个T实例都是单态，如果Optional中没有非空存在的T实例，返回的将是一个空的不可修改的Set。

使用实例如下：

```Java
import java.util.Set;
import com.google.common.base.Optional;

public class OptionalTest {

    public void testMethodReturn() {
        Optional<Long> value = method();
        if(value.isPresent()==true){
            System.out.println("获得返回值: " + value.get());
        }else{

            System.out.println("获得返回值: " + value.or(-12L));
        }

        System.out.println("获得返回值 orNull: " + value.orNull());

        Optional<Long> valueNoNull = methodNoNull();
        if(valueNoNull.isPresent()==true){
            Set<Long> set=valueNoNull.asSet();
            System.out.println("获得返回值 set 的 size : " + set.size());
            System.out.println("获得返回值: " + valueNoNull.get());
        }else{
            System.out.println("获得返回值: " + valueNoNull.or(-12L));
        }

        System.out.println("获得返回值 orNull: " + valueNoNull.orNull());
    }

    private Optional<Long> method() {
        return Optional.fromNullable(null);
    }
    private Optional<Long> methodNoNull() {
        return Optional.fromNullable(15L);
    }

}
```

输出结果：

```
获得返回值: -12
获得返回值 orNull: null
获得返回值 set 的 size : 1
获得返回值: 15
获得返回值 orNull: 15
```

Optional除了给null值命名所带来的代码可阅读性的提高，最大的好处莫过于Optional是傻瓜式的。Optional对象的使用强迫你去积极的思考这样一种情况，如果你想让你的程序返回null值，这null值代表的含义是什么，因为你想要取得返回值，必然从Optional对象内部去获得，所以你必然会这么去思考。但是只是简单的使用一个Null值会很轻易的让人忘记去思索代码所要表达的含义到底是什么，尽管FindBugs有些帮助，但是我们还是认为它并没有尽可能的解决好帮助程序员去思索null值代表的含义这个问题。

这种思考会在你返回某些存在的值或者不存在的值的时候显得特别相关。和其他人一样，你绝对很可能会忘记别人写的方法method(a,b)可能会返回一个null值，就好像当你去写method(a,b)的实现时，你也很可能忘记输入参数a也可以是null。如果返回的是Optional对象，对于调用者来说，就可以忘却怎么去度量null代表的是什么含义，因为他们始终要从optional对象中去获得真正的返回值。