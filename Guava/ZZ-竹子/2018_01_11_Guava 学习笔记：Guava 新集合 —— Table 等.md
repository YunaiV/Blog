title: Guava 学习笔记：Guava 新集合 —— Table 等
date: 2018-01-11
tag: 
categories: Guava
permalink: Guava/peida/Table
author: 竹子
from_url: http://www.cnblogs.com/peida/p/3183505.html
wechat_url: 

-------

摘要: 原创出处 http://www.cnblogs.com/peida/p/3183505.html 「竹子」欢迎转载，保留摘要，谢谢！

- [Table](http://www.iocoder.cn/Guava/peida/Table/)
- [ClassToInstanceMap](http://www.iocoder.cn/Guava/peida/Table/)
- [RangeSet](http://www.iocoder.cn/Guava/peida/Table/)
- [RangeMap](http://www.iocoder.cn/Guava/peida/Table/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

# Table

　　当我们需要多个索引的数据结构的时候，通常情况下，我们只能用这种丑陋的Map<FirstName, Map<LastName, Person>>来实现。为此Guava提供了一个新的集合类型－Table集合类型，来支持这种数据结构的使用场景。Table支持“row”和“column”，而且提供多种视图。　

```Java
@Test
    public void TableTest(){
        Table<String, Integer, String> aTable = HashBasedTable.create();  
 
        for (char a = 'A'; a <= 'C'; ++a) {  
            for (Integer b = 1; b <= 3; ++b) {   
                aTable.put(Character.toString(a), b, String.format("%c%d", a, b));  
            }  
        }  
   
        System.out.println(aTable.column(2));  
        System.out.println(aTable.row("B"));   
        System.out.println(aTable.get("B", 2));  

        System.out.println(aTable.contains("D", 1));   
        System.out.println(aTable.containsColumn(3));   
        System.out.println(aTable.containsRow("C"));  
        System.out.println(aTable.columnMap()); 
        System.out.println(aTable.rowMap());   

        System.out.println(aTable.remove("B", 3)); 
    }
```

　　输出：

```Java
{A=A2, B=B2, C=C2}
{1=B1, 2=B2, 3=B3}
B2
false
true
true
{1={A=A1, B=B1, C=C1}, 2={A=A2, B=B2, C=C2}, 3={A=A3, B=B3, C=C3}}
{A={1=A1, 2=A2, 3=A3}, B={1=B1, 2=B2, 3=B3}, C={1=C1, 2=C2, 3=C3}}
B3
```

Table视图： 

* rowMap()返回一个Map<R, Map<C, V>>的视图。rowKeySet()类似地返回一个Set<R>。 
* row(r)返回一个非null的Map<C, V>。修改这个视图Map也会导致原表格的修改。 
* 和列相关的方法有columnMap(), columnKeySet()和column(c)。（基于列的操作会比基于行的操作效率差些） 
* cellSet()返回的是以Table.Cell<R, C, V>为元素的Set。这里的Cell就类似Map.Entry，但是它是通过行和列来区分的。

Table有以下实现： 

* HashBasedTable：基于HashMap<R, HashMap<C, V>>的实现。 
* TreeBasedTable：基于TreeMap<R, TreeMap<C, V>>的实现。 
* ImmutableTable：基于ImmutableMap<R, ImmutableMap<C, V>>的实现。（注意，ImmutableTable已对稀疏和密集集合做了优化） 
* ArrayTable：ArrayTable是一个需要在构建的时候就需要定下行列的表格。这种表格由二维数组实现，这样可以在密集数据的表格的场合，提高时间和空间的效率。

# ClassToInstanceMap

　　有的时候，你的map的key并不是一种类型，他们是很多类型，你想通过映射他们得到这种类型，guava提供了ClassToInstanceMap满足了这个目的。

　　除了继承自Map接口，ClassToInstaceMap提供了方法 T getInstance(Class<T>) 和 T putInstance(Class<T>, T),消除了强制类型转换。 

　　该类有一个简单类型的参数，通常称为B，代表了map控制的上层绑定，例如：

```Java
ClassToInstanceMap<Number> numberDefaults = MutableClassToInstanceMap.create();
numberDefaults.putInstance(Integer.class, Integer.valueOf(0));
```

从技术上来说，`ClassToInstanceMap<B>` 实现了 `Map<Class<? extends B>, B>`，或者说，这是一个从B的子类到B对象的映射，这可能使得 ClassToInstanceMap 的泛型轻度混乱，但是只要记住B总是Map的上层绑定类型，通常来说B只是一个对象。

guava提供了有用的实现， MutableClassToInstanceMap 和 ImmutableClassToInstanceMap.

重点：像其他的Map<Class,Object>,ClassToInstanceMap 含有的原生类型的项目，一个原生类型和他的相应的包装类可以映射到不同的值；

```Java
import org.junit.Test;

import com.google.common.collect.ClassToInstanceMap;
import com.google.common.collect.HashBasedTable;
import com.google.common.collect.MutableClassToInstanceMap;

public class OtherTest {
    
    @Test
    public  void ClassToInstanceMapTest() {
        ClassToInstanceMap<String> classToInstanceMapString =MutableClassToInstanceMap.create();
        ClassToInstanceMap<Person> classToInstanceMap =MutableClassToInstanceMap.create();
        Person person= new Person("peida",20);
        System.out.println("person name :"+person.name+" age:"+person.age);
        classToInstanceMapString.put(String.class, "peida");
        System.out.println("string:"+classToInstanceMapString.getInstance(String.class));
        
        classToInstanceMap.putInstance(Person.class,person);
        Person person1=classToInstanceMap.getInstance(Person.class);
        System.out.println("person1 name :"+person1.name+" age:"+person1.age);
    }
}

class Person {
    public String name;
    public int age;

    Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
} 
```

# RangeSet

　　RangeSet用来处理一系列不连续，非空的range。当添加一个range到一个RangeSet之后，任何有连续的range将被自动合并，而空的range将被自动去除。例如：

```Java
    @Test
    public void RangeSetTest(){
        RangeSet<Integer> rangeSet = TreeRangeSet.create();
        rangeSet.add(Range.closed(1, 10));
        System.out.println("rangeSet:"+rangeSet);
        rangeSet.add(Range.closedOpen(11, 15)); 
        System.out.println("rangeSet:"+rangeSet);
        rangeSet.add(Range.open(15, 20));
        System.out.println("rangeSet:"+rangeSet);
        rangeSet.add(Range.openClosed(0, 0)); 
        System.out.println("rangeSet:"+rangeSet);
        rangeSet.remove(Range.open(5, 10)); 
        System.out.println("rangeSet:"+rangeSet);
    }
```

输出：

```Java
rangeSet:{[1‥10]}
rangeSet:{[1‥10][11‥15)}
rangeSet:{[1‥10][11‥15)(15‥20)}
rangeSet:{[1‥10][11‥15)(15‥20)}
rangeSet:{[1‥5][10‥10][11‥15)(15‥20)}
```

　　注意，像合并Range.closed(1, 10)和Range.closedOpen(11, 15)这样的情况，我们必须先用调用Range.canonical(DiscreteDomain)传入DiscreteDomain.integers()处理一下。

**RangeSet的视图**

RangeSet的实现支持了十分丰富的视图，包括：

* complement():是个辅助的RangeSet，它本身就是一个RangeSet，因为它包含了非连续，非空的range。
* subRangeSet(Range<C>): 返回的是一个交集的视图。
* asRanges():返回可以被迭代的Set<Range<C>>的视图。
* asSet(DiscreteDomain<C>) (ImmutableRangeSet only):返回一个ImmutableSortedSet<C>类型的视图,里面的元素是range里面的元素，而不是range本身。（如果DiscreteDomain和RangeSet的上限或下限是无限的话，这个操作就不能支持）

**Queries**

除了支持各种视图，RangeSet还支持各种直接的查询操作，其中最重要的是：

* contains(C):这是RangeSet最基本的操作，它能查询给定的元素是否在RangeSet里。
* rangeContaining(C): 返回包含给定的元素的Range，如果不存在就返回null。
* encloses(Range<C>): 用来判断给定的Range是否包含在RangeSet里面。
* span():返回一个包含在这个RangeSet的所有Range的并集。

# RangeMap

RangeMap代表了非连续非空的range对应的集合。不像RangeSet，RangeMap不会合并相邻的映射，甚至相邻的range对应的是相同的值。例如：

```Java
    @Test
    public void RangeMapTest(){
        RangeMap<Integer, String> rangeMap = TreeRangeMap.create();
         rangeMap.put(Range.closed(1, 10), "foo"); 
         System.out.println("rangeMap:"+rangeMap);
         rangeMap.put(Range.open(3, 6), "bar"); 
         System.out.println("rangeMap:"+rangeMap);
         rangeMap.put(Range.open(10, 20), "foo"); 
         System.out.println("rangeMap:"+rangeMap);
         rangeMap.remove(Range.closed(5, 11)); 
         System.out.println("rangeMap:"+rangeMap);
    }

    输出：
    rangeMap:[[1‥10]=foo]
    rangeMap:[[1‥3]=foo, (3‥6)=bar, [6‥10]=foo]
    rangeMap:[[1‥3]=foo, (3‥6)=bar, [6‥10]=foo, (10‥20)=foo]
    rangeMap:[[1‥3]=foo, (3‥5)=bar, (11‥20)=foo]
```

**RangeMap的视图**

RangeMap提供了两种视图：
　　
* asMapOfRanges():返回Map<Range<K>, V>类型的视图。这个操作可以被用作迭代操作。
* subRangeMap(Range<K>)提供给定Range的交集。这个操作可以推广到传统的headMap, subMap, 和tailMap。

