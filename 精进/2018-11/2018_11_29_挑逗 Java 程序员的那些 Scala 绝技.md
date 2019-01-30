title: 挑逗 Java 程序员的那些 Scala 绝技
date: 2018-11-29
tags:
categories: 精进
permalink: Fight/Scala-stunts-that-tickle-Java-programmers
author: joymufeng
from_url: https://my.oschina.net/joymufeng/blog/2251038
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247485768&idx=1&sn=c10cad791c62ec49d8cdab771b5993d4&chksm=fa4976f9cd3effeffe2b0c6897d91914ad40a885f7f88dc95124e76a61a5f47ce98feead5b94&token=582518212&lang=zh_CN#rd

-------

摘要: 原创出处 https://my.oschina.net/joymufeng/blog/2251038 「joymufeng」欢迎转载，保留摘要，谢谢！

- [类型推断](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
  - [与 Java 7 的钻石操作符冲突](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
  - [容易导致错误的代码](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
- [字符串增强](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
  - [常用操作](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
  - [原生字符串](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
  - [字符串插值](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
- [集合操作](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
  - [简洁的初始化方式](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
  - [便捷的 Tuple 类型](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
  - [链式调用](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
  - [非典型集合操作](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
  - [并行集合](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
- [优雅的值对象](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
  - [Case Class](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
  - [简洁的实例化方式](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
  - [不可变性](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
  - [对象拷贝](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
  - [清晰的调试信息](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
  - [默认使用值比较相等性](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
- [模式匹配](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
  - [更强的可读性](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
  - [变量赋值](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
- [并发编程](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
  - [跨线程错误处理](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
- [声明式编程](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
- [面向表达式编程](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
  - [一个整数加法解释器](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
- [隐式参数和隐式转换](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
  - [隐式参数](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
  - [隐式转换](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)
- [小结](http://www.iocoder.cn/Fight/Scala-stunts-that-tickle-Java-programmers/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

有个问题一直困扰着 Scala 社区，为什么一些 Java 开发者将 Scala 捧到了天上，认为它是来自上帝之吻的完美语言；而另外一些 Java 开发者却对它望而却步，认为它过于复杂而难以理解。同样是 Java 开发者，为何会出现两种截然不同的态度，我想这其中一定有误会。Scala 是一粒金子，但是被一些表面上看起来非常复杂的概念或语法包裹的太严实，以至于人们很难在短时间内搞清楚它的价值。与此同时，Java 也在不断地摸索前进，但是由于 Java 背负了沉重的历史包袱，所以每向前一步都显得异常艰难。本文主要面向 Java 开发人员，希望从解决 Java 中实际存在的问题出发，梳理最容易吸引 Java 开发者的一些 Scala 特性。希望可以帮助大家快速找到那些真正可以打动你的点。

# 类型推断

> 挑逗指数: 四星

我们知道，Scala 一向以强大的类型推断闻名于世。很多时候，我们无须关心 Scala 类型推断系统的存在，因为很多时候它推断的结果跟直觉是一致的。 Java 在 2016 年也新增了一份提议[JEP 286](http://openjdk.java.net/jeps/286)，计划为 Java 10 引入局部变量类型推断(Local-Variable Type Inference)。利用这个特性，我们可以使用 var 定义变量而无需显式声明其类型。很多人认为这是一项激动人心的特性，但是高兴之前我们要先看看它会为我们带来哪些问题。

## 与 Java 7 的钻石操作符冲突

Java 7 引进了钻石操作符，使得我们可以降低表达式右侧的冗余类型信息，例如：

```Java
List<Integer> numbers = new ArrayList<>();
```

如果引入了 var，则会导致左侧的类型丢失，从而导致整个表达式的类型丢失：

```Java
var numbers = new ArrayList<>();
```

所以 var 和 钻石操作符必须二选一，鱼与熊掌不可兼得。

## 容易导致错误的代码

下面是一段检查用户是否存在的 Java 代码：

```Java
public boolean userExistsIn(Set<Long> userIds) {
    var userId = getCurrentUserId();
    return userIds.contains(userId);
}
```

请仔细观察上述代码，你能一眼看出问题所在吗？ userId 的类型被 var 隐去了，如果 getCurrentUserId() 返回的是 String 类型，上述代码仍然可以正常通过编译，却无形中埋下了隐患，这个方法将会永远返回 false， 因为 Set<Long>.contains 方法接受的参数类型是 Object。可能有人会说，就算显式声明了类型，不也是于事无补吗？

```Java
public boolean userExistsIn(Set<Long> userIds) {
    String userId = getCurrentUserId();
    return userIds.contains(userId);
}
```

Java 的优势在于它的类型可读性，如果显式声明了 userId 的类型，虽然还是可以正常通过编译，但是在代码审查时，这个错误将会更容易被发现。 这种类型的错误在 Java 中非常容易发生，因为 getCurrentUserId() 方法很可能因为重构而改变了返回类型，而 Java 编译器却在关键时刻背叛了你，没有报告任何的编译错误。 虽然这是由于 Java 的历史原因导致的，但是由于 var 的引入，会导致这个错误不断的蔓延。

很显然，在 Scala 中，这种低级错误是无法逃过编译器法眼的：

```Java
def userExistsIn(userIds: Set[Long]): Boolean = {
    val userId = getCurrentUserId()
    userIds.contains(userId)
}
```

如果 userId 不是 Long 类型，则上面的程序无法通过编译。

# 字符串增强

> 挑逗指数: 四星

## 常用操作

Scala 针对字符作进行了增强，提供了更多的使用操作：

```Java
//字符串去重
"aabbcc".distinct // "abc"

//取前n个字符，如果n大于字符串长度返回原字符串
"abcd".take(10) // "abcd"

//字符串排序
"bcad".sorted // "abcd"

//过滤特定字符
"bcad".filter(_ != 'a') // "bcd"

//类型转换
"true".toBoolean
"123".toInt
"123.0".toDouble
```

其实你完全可以把 String 当做 Seq[Char] 使用，利用 Scala 强大的集合操作，你可以随心所欲地操作字符串。

## 原生字符串

在 Scala 中，我们可以直接书写原生字符串而不用进行转义，将字符串内容放入一对三引号内即可：

```Java
//包含换行的字符串
val s1= """Welcome here.
   Type "HELP" for help!"""
   
//包含正则表达式的字符串   
val regex = """\d+"""   
```

## 字符串插值

通过 s 表达式，我们可以很方便地在字符串内插值：

```Java
val name = "world"
val msg = s"hello, ${name}" // hello, world
```

# 集合操作

> 挑逗指数: 五星

Scala 的集合设计是最容易让人着迷的地方，就像毒品一样，一沾上便让人深陷其中难以自拔。通过 Scala 提供的集合操作，我们基本上可以实现 SQL 的全部功能，这也是为什么 Scala 能够在大数据领域独领风骚的重要原因之一。

## 简洁的初始化方式

在 Scala 中，我们可以这样初始化一个列表：

```Java
val list1 = List(1, 2, 3)
```

可以这样初始化一个 Map：

```Java
val map = Map("a" -> 1, "b" -> 2)
```

所有的集合类型均可以用类似的方式完成初始化，简洁而富有表达力。

## 便捷的 Tuple 类型

有时方法的返回值可能不止一个，Scala 提供了 Tuple (元组)类型用于临时存放多个不同类型的值，同时能够保证类型安全性。千万不要认为使用 Java 的 Array 类型也可以同样实现 Tuple 类型的功能，它们之间有着本质的区别。Tuple 会显式声明所有元素的各自类型，而不是像 Java Array 那样，元素类型会被向上转型为所有元素的父类型。
我们可以这样初始化一个 Tuple:

```Java
val t = ("abc", 123, true)
val s: String  = t._1 // 取第1个元素
val i: Int     = t._2 // 取第2个元素
val b: Boolean = t._3 // 取第3个元素
```

需要注意的是 Tuple 的元素索引从1开始。

下面的示例代码是在一个长整型列表中寻找最大值，并返回这个最大值以及它所在的位置：

```Java
def max(list: List[Long]): (Long, Int) = list.zipWithIndex.sorted.reverse.head
```

我们通过 zipWithIndex 方法获取每个元素的索引号，从而将 List[Long] 转换成了 List[(Long, Int)]，然后对其依次进行排序、倒序和取首元素，最终返回最大值及其所在位置。

## 链式调用

通过链式调用，我们可以将关注点放在数据的处理和转换上，而无需考虑如何存储和传递数据，同时也避免了创建大量无意义的中间变量，大大增强程序的可读性。其实上面的 max 函数已经演示了链式调用。下面我们演示一下如何使用集合操作实现 SQL 的关联查询功能，待实现的 SQL 语句如下：

```SQL
SELECT p.name, p.company, c.country FROM people p JOIN companies c ON p.companyId = c.id
WHERE p.age == 20
```

上面 SQL 语句实现的功能是关联查询 people 和 companies 两张表，返回年龄为20岁的所有员工名称、年龄以及其所在公司名称。

对应的 Scala 实现代码如下：

```Java
// Entity
case class People(name: String, age: Int, companyId: String)
case class Company(id: String, name: String)

// Entity List
val people    = List(People("jack", 20, "0"))
val companies = List(Company("0", "lightbend"))

// 实现关联查询
people
  .filter(p => p.age == 20)
  .flatMap{ p =>
    companies
      .filter(c => c.id == p.companyId)
      .map(c => (p.name, p.age, c.name))
}
//结果：List((jack,20,lightbend))
```

其实使用 for 表达式看起来更加简洁：

```Java
for {
  p <- people if p.age == 20
  c <- companies if p.companyId == c.id
} yield (p.name, p.age, c.name)
```

## 非典型集合操作

Scala 的集合操作非常丰富，如果要详细说明足够写一本书了。这里仅列出一些不那么常用但却非常好用的操作。

去重：

```Java
List(1, 2, 2, 3).distinct // List(1, 2, 3)
```

交集：

```Java
Set(1, 2) & Set(2, 3)   // Set(2)
```

并集：

```Java
Set(1, 2) | Set(2, 3) // Set(1, 2, 3)
```

差集：

```Java
Set(1, 2) &~ Set(2, 3) // Set(1)
```

排列：

```Java
List(1, 2, 3).permutations.toList
//List(List(1, 2, 3), List(1, 3, 2), List(2, 1, 3), List(2, 3, 1), List(3, 1, 2), List(3, 2, 1))
```

组合：

```Java
List(1, 2, 3).combinations(2).toList 
// List(List(1, 2), List(1, 3), List(2, 3))
```

## 并行集合

Scala 的并行集合可以利用多核优势加速计算过程，通过集合上的 par 方法，我们可以将原集合转换成并行集合。并行集合利用分治算法将计算任务分解成很多子任务，然后交给不同的线程执行，最后将计算结果进行汇总。下面是一个简单的示例：

```Java
(1 to 10000).par.filter(i => i % 2 == 1).sum
```

# 优雅的值对象

> 挑逗指数: 五星

## Case Class

Scala 标准库包含了一个特殊的 Class 叫做 Case Class，专门用于领域层值对象的建模。它的好处是所有的默认行为都经过了合理的设计，开箱即用。下面我们使用 Case Class 定义了一个 User 值对象：

```Java
case class User(name: String, role: String = "user", addTime: Instant = Instant.now())
```

仅仅一行代码便完成了 User 类的定义，请脑补一下 Java 的实现。

## 简洁的实例化方式

我们为 role 和 addTime 两个属性定义了默认值，所以我们可以只使用 name 创建一个 User 实例：

```Java
val u = User("jack")
```

在创建实例时，我们也可以命名参数(named parameter)语法改变默认值：

```Java
val u = User("jack", role = "admin")
```

在实际开发中，一个模型类或值对象可能拥有很多属性，其实很多属性都可以设置一个合理的默认值。利用默认值和命名参数，我们可以非常方便地创建模型类和值对象的实例。 所以在 Scala 中基本上不需要使用工厂模式或构造器模式创建对象，如果对象的创建过程确实非常复杂，则可以放在伴生对象中创建，例如:

```Java
object User {
  def apply(name: String): User = User(name, "user", Instant.now())
}
```

在使用伴生对象方法创建实例时可以省略方法名 apply，例如：

```Java
User("jack") // 等价于 User.apply("jack")
```

在这个例子里，使用伴生对象方法实例化对象的代码，与上面使用类构造器的代码完全一样，编译器会优先选择伴生对象的 apply 方法。

## 不可变性

Case Class 在默认情况下实例是不可变的，意味着它可以被任意共享，并发访问时也无需同步，大大地节省了宝贵的内存空间。而在 Java 中，对象被共享时需要进行深拷贝，否则一个地方的修改会影响到其它地方。例如在 Java 中定义了一个 Role 对象：

```Java
public class Role {
    public String id = "";
    public String name = "user";
    
    public Role(String id, String name) {
        this.id = id;
        this.name = name;
    }
}
```

如果在两个 User 之间共享 Role 实例就会出现问题，就像下面这样：

```Java
u1.role = new Role("user", "user");
u2.role = u1.role;
```

当我们修改 u1.role 时，u2 就会受到影响，Java 的解决方式是要么基于 u1.role 深度克隆一个新对象出来，要么新创建一个 Role 对象赋值给 u2。

## 对象拷贝

在 Scala 中，既然 Case Class 是不可变的，那么如果想改变它的值该怎么办呢？其实很简单，利用命名参数可以很容易拷贝一个新的不可变对象出来：

```Java
val u1 = User("jack")
val u2 = u1.copy(name = "role", role = "admin")
```

## 清晰的调试信息

我们不需要编写额外的代码便可以得到清晰的调试信息，例如：

```Java
val users = List(User("jack"), User("rose"))
println(users)

```

输出内容如下：

```Java
List(User(jack,user,2018-10-20T13:03:16.170Z), User(rose,user,2018-10-20T13:03:16.170Z))

```

## 默认使用值比较相等性

在 Scala 中，默认采用值比较而非引用比较，使用起来更加符合直觉：

```Java
User("jack") == User("jack") // true

```

上面的值比较是开箱即用的，无需重写 hashCode 和 equals 方法。

# 模式匹配

> 挑逗指数: 五星

## 更强的可读性

当你的代码中存在多个 if 分支并且 if 之间还会有嵌套，那么代码的可读性将会大大降低。而在 Scala 中使用模式匹配可以很容易地解决这个问题，下面的代码演示货币类型的匹配：

```Java
sealed trait Currency
case class Dollar(value: Double) extends Currency
case class Euro(value: Double) extends Currency
val Currency = ...
currency match {
    case Dollar(v) => "$" + v
    case Euro(v) => "€" + v
    case _ => "unknown"
}

```

我们也可以进行一些复杂的匹配，并且在匹配时可以增加 if 判断：

```Java
use match {
    case User("jack", _, _) => ...
    case User(_, _, addTime) if addTime.isAfter(time) => ...
    case _ => ...
}

```

## 变量赋值

利用模式匹配，我们可以快速提取特定部分的值并完成变量定义。 我们可以将 Tuple 中的值直接赋值给变量：

```Java
val tuple = ("jack", "user", Instant.now())
val (name, role, addTime) = tuple
// 变量 name, role, addTime 在当前作用域内可以直接使用

```

对于 Case Class 也是一样：

```Java
val User(name, role, addTime) = User("jack")
// 变量 name, role, addTime 在当前作用域内可以直接使用

```

# 并发编程

> 挑逗指数: 五星

在 Scala 中，我们在编写并发代码时只需要关心业务逻辑即可，而不需要关注任务如何执行。我们可以通过显式或隐式方式传入一个线程池，具体的执行过程由线程池完成。Future 用于启动一个异步任务并且保存执行结果，我们可以用 for 表达式收集多个 Future 的执行结果，从而避免回调地狱：

```Java
val f1 = Future{ 1 + 2 }
val f2 = Future{ 3 + 4 }
for {
    v1 <- f1
    v2 <- f2
}{
    println(v1 + v2) // 10
}

```

使用 Future 开发爬虫程序将会让你事半功倍，假如你想同时抓取 100 个页面数据，一行代码就可以了：

```Java
Future.sequence(urls.map(url => http.get(url))).foreach{ contents => ...}

```

Future.sequence 方法用于收集所有 Future 的执行结果，通过 foreach 方法我们可以取出收集结果并进行后续处理。

当我们要实现完全异步的请求限流时，就需要精细地控制每个 Future 的执行时机。也就是说我们需要一个控制Future的开关，没错，这个开关就是Promise。每个Promise实例都会有一个唯一的Future与之相关联：

```Java
val p = Promise[Int]()
val f = p.future
for (v <- f) { println(v) } // 3秒后才会执行打印操作

//3秒钟之后返回3
Thread.sleep(3000)
p.success(3)

```

## 跨线程错误处理

Java 通过异常机制处理错误，但是问题在于 Java 代码只能捕获当前线程的异常，而无法跨线程捕获异常。而在 Scala 中，我们可以通过 Future 捕获任意线程中发生的异常。
异步任务可能成功也可能失败，所以我们需要一种既可以表示成功，也可以表示失败的数据类型，在 Scala 中它就是 Try[T]。Try[T] 有两个子类型，Success[T]表示成功，Failure[T]表示失败。就像量子物理学中薛定谔的猫，在异步任务执行之前，你根本无法预知返回的结果是 Success[T] 还是 Failure[T]，只有当异步任务完成执行以后结果才能确定下来。

```Java
val f = Future{ /*异步任务*/ } 

// 当异步任务执行完成时
f.value.get match {
  case Success(v) => // 处理成功情况
  case Failure(t) => // 处理失败情况
}

```

我们也可以让一个 Future 从错误中恢复：

```Java
val f = Future{ /*异步任务*/ }
for{
  result <- f.recover{ case t => /*处理错误*/ }
} yield {
  // 处理结果
}

```

# 声明式编程

> 挑逗指数: 四星

Scala 鼓励声明式编程，采用声明式编写的代码可读性更强。与传统的过程式编程相比，声明式编程更关注我想做什么而不是怎么去做。例如我们经常要实现分页操作，每页返回 10 条数据：

```Java
val allUsers = List(User("jack"), User("rose"))
val pageList = 
  allUsers
    .sortBy(u => (u.role, u.name, u.addTime)) // 依次按 role, name, addTime 进行排序
    .drop(page * 10) // 跳过之前页数据
    .take(10) // 取当前页数据，如不足10个则全部返回

```

你只需要告诉 Scala 要做什么，比如说先按 role 排序，如果 role 相同则按 name 排序，如果 role 和 name 都相同，再按 addTime 排序。底层具体的排序实现已经封装好了，开发者无需实现。

# 面向表达式编程

> 挑逗指数: 四星

在 Scala 中，一切都是表达式，包括 if, for, while 等常见的控制结构均是表达式。表达式和语句的不同之处在于每个表达式都有明确的返回值。

```Java
val i = if(true){ 1 } else { 0 } // i = 1
val list1 = List(1, 2, 3)
val list2 = for(i <- list1) yield { i + 1 }

```

不同的表达式可以组合在一起形成一个更大的表达式，再结合上模式匹配将会发挥巨大的威力。下面我们以一个计算加法的解释器来做说明。

## 一个整数加法解释器

我们首先定义基本的表达式类型：

```Java
abstract class Expr
case class Number(num: Int) extends Expr
case class PlusExpr(left: Expr, right: Expr) extends Expr

```

上面定义了两个表达式类型，Number 表示一个整数表达式， PlusExpr 表示一个加法表达式。
下面我们基于模式匹配实现表达式的求值运算：

```Java
def evalExpr(expr: Expr): Int = {
  expr match {
    case Number(n) => n
    case PlusExpr(left, right) => evalExpr(left) + evalExpr(right)
  }
}

```

我们来尝试针对一个较大的表达式进行求值：

```Java
evalExpr(PlusExpr(PlusExpr(Number(1), Number(2)), PlusExpr(Number(3), Number(4)))) // 10

```

# 隐式参数和隐式转换

> 挑逗指数: 五星

## 隐式参数

如果每当要执行异步任务时，都需要显式传入线程池参数，你会不会觉得很烦？Scala 通过隐式参数为你解除这个烦恼。例如 Future 在创建异步任务时就声明了一个 ExecutionContext 类型的隐式参数，编译器会自动在当前作用域内寻找合适的 ExecutionContext，如果找不到则会报编译错误：

```Java
implicit val ec: ExecutionContext = ???
val f = Future { /*异步任务*/ }

```

当然我们也可以显式传递 ExecutionContext 参数，明确指定使用的线程池：

```Java
implicit val ec: ExecutionContext = ???
val f = Future { /*异步任务*/ }(ec)

```

## 隐式转换

隐式转换相比较于隐式参数，使用起来更来灵活。如果 Scala 在编译时发现了错误，在报错之前，会先对错误代码应用隐式转换规则，如果在应用规则之后可以使得其通过编译，则表示成功地完成了一次隐式转换。

### 在不同的库间实现无缝对接

当传入的参数类型和目标类型不匹配时，编译器会尝试隐式转换。利用这个功能，我们将已有的数据类型无缝对接到三方库上。例如我们想在 Scala 项目中使用 MongoDB 的官方 Java 驱动执行数据库查询操作，但是查询接口接受的参数类型是 BsonDocument，由于使用 BsonDocument 构建查询比较笨拙，我们希望能够使用 Scala 的 JSON 库构建一个查询对象，然后直接传递给官方驱动的查询接口，而无需改变官方驱动的任何代码，利用隐式转换可以非常轻松地实现这个功能：

```Java
implicit def toBson(json: JsObject): BsonDocument =  ...

val json: JsObject = Json.obj("_id" -> "0")
jCollection.find(json) // 编译器会自动调用 toBson(json)

```

利用隐式转换，我们可以在不改动三方库代码的情况下，将我们的数据类型与其进行无缝对接。例如我们通过实现一个隐式转换，将 Scala 的 JsObject 类型无缝地对接到了 MongoDB 的官方 Java 驱动的查询接口中，看起就像是 MongoDB 官方驱动真的提供了这个接口一样。

同时我们也可以将来自三方库的数据类型无缝集成到现有的接口中，也只需要实现一个隐式转换方法即可。

### 扩展已有类的功能

例如我们定义了一个美元货币类型 Dollar:

```Java
class Dollar(value: Double) {
  def + (that: Dollar): Dollar = ...
  def + (that: Int): Dollar = ...
}

```

于是我们可以执行如下操作：

```Java
val halfDollar = new Dollar(0.5)
halfDollar + halfDollar // 1 dollar
halfDollar + 0.5 // 1 dollar

```

但是我们却无法执行像 0.5 + halfDollar 这样的运算，因为在 Double 类型上无法找到一个合适的 + 方法。

在 Scala 中，为了实现上面的运算，我们只需要实现一个简单的隐式转换就可以了：

```Java
implicit def doubleToDollar(d: Double) = new Dollar(d)

0.5 + halfDollar // 等价于 doubleToDollar(0.5) + halfDollar

```

### 更好的运行时性能

在日常开发中，我们通常需要将值对象转换成 Json 格式以方便数据传输。Java 的通常做法是使用反射，但是我们知道使用反射是要付出代价的，要承受运行时的性能开销。而 Scala 则可以在编译时为值对象生成隐式的 Json 编解码器，这些编解码器只不过是普通的函数调用而已，不涉及任何反射操作，在很大程度上提升了系统的运行时性能。

# 小结

如果你坚持读到了这里，我会觉得非常欣慰，很大可能上 Scala 的某些特性已经吸引了你。但是 Scala 的魅力远不止如此，以上列举的仅仅是一些最容易抓住你眼球的一些特性。如果你愿意推开 Scala 这扇大门，你将会看到一个完全不一样的编程世界。本文欢迎转载，请注明作者沐风(joymufeng)。

> **后记**
> 首先感谢大家关注本文，也非常感谢 [@dwingo](https://my.oschina.net/u/3558761) 参与讨论。Scala 和 Java 同根同源，并且完全拥抱现有 Java 生态，在开发中我们也经常使用两种语言混合编程，所以 Scala = Java and More。全文都是围绕 Scala 的"简洁性"展开，Scala 在保持简洁性的同时，提供了非常好的可读性，可以参考本文的"声明式编程"一节。Scala的很多设计都是对 Java 的改良与超越，所以学习 Scala 的过程其实是一次对 Java 的深度回顾。在这里向大家推荐一本书《快学Scala 第二版》(高宇翔 译)，作者 Cay S. Horstmann 是一位 Java 语言大师，并且是《Java核心技术》卷1和卷2的作者，全书围绕Java与Scala展开，行文简洁通透，让人醍醐灌顶，尤其闭包一节甚为精彩。难能可贵的是本书翻译也很棒，如果你对 Scala 不感冒，仅读 Java 部分也会受益颇多。