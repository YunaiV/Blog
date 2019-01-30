title: Guava 学习笔记：EventBus
date: 2018-01-13
tag: 
categories: Guava
permalink: Guava/peida/EventBus
author: 竹子
from_url: http://www.cnblogs.com/peida/p/EventBus.html
wechat_url: 

-------

摘要: 原创出处 http://www.cnblogs.com/peida/p/EventBus.html 「竹子」欢迎转载，保留摘要，谢谢！


-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

EventBus是Guava的事件处理机制，是设计模式中的观察者模式（生产/消费者编程模型）的优雅实现。对于事件监听和发布订阅模式，EventBus是一个非常优雅和简单解决方案，我们不用创建复杂的类和接口层次结构。

Observer模式是比较常用的设计模式之一，虽然有时候在具体代码里，它不一定叫这个名字，比如改头换面叫个Listener，但模式就是这个模式。手工实现一个Observer也不是多复杂的一件事，只是因为这个设计模式实在太常用了，Java就把它放到了JDK里面：Observable和Observer，从JDK 1.0里，它们就一直在那里。从某种程度上说，它简化了Observer模式的开发，至少我们不用再手工维护自己的Observer列表了。不过，如前所述，JDK里的Observer从1.0就在那里了，直到Java 7，它都没有什么改变，就连通知的参数还是Object类型。要知道，Java 5就已经泛型了。Java 5是一次大规模的语法调整，许多程序库从那开始重新设计了API，使其更简洁易用。当然，那些不做应对的程序库，多半也就过时了。这也就是这里要讨论知识更新的原因所在。今天，对于普通的应用，如果要使用Observer模式该如何做呢？答案是Guava的EventBus。

**基本用法**

　　使用Guava之后, 如果要订阅消息, 就不用再继承指定的接口, 只需要在指定的方法上加上@Subscribe注解即可。代码如下：

　　消息封装类：

```Java
public class TestEvent {
    private final int message;
    public TestEvent(int message) {
        this.message = message;
        System.out.println("event message:"+message);
    }
    public int getMessage() {
        return message;
    }
}
```

　　消息接受类：

```Java
public class EventListener {
    public int lastMessage = 0;

    @Subscribe
    public void listen(TestEvent event) {
        lastMessage = event.getMessage();
        System.out.println("Message:"+lastMessage);
    }

    public int getLastMessage() {
        return lastMessage;
    }
}
```

　　测试类及输出结果：

```Java
public class TestEventBus {
    @Test
    public void testReceiveEvent() throws Exception {

        EventBus eventBus = new EventBus("test");
        EventListener listener = new EventListener();

        eventBus.register(listener);

        eventBus.post(new TestEvent(200));
        eventBus.post(new TestEvent(300));
        eventBus.post(new TestEvent(400));

        System.out.println("LastMessage:"+listener.getLastMessage());
        ;
    }
}

//输出信息
event message:200
Message:200
event message:300
Message:300
event message:400
Message:400
LastMessage:400
```

**MultiListener的使用：**

　　只需要在要订阅消息的方法上加上@Subscribe注解即可实现对多个消息的订阅，代码如下：

```Java
public class MultipleListener {
    public Integer lastInteger;
    public Long lastLong;

    @Subscribe
    public void listenInteger(Integer event) {
        lastInteger = event;
        System.out.println("event Integer:"+lastInteger);
    }

    @Subscribe
    public void listenLong(Long event) {
        lastLong = event;
        System.out.println("event Long:"+lastLong);
    }

    public Integer getLastInteger() {
        return lastInteger;
    }

    public Long getLastLong() {
        return lastLong;
    }
}
```

　　测试类：

```Java
public class TestMultipleEvents {
    @Test
    public void testMultipleEvents() throws Exception {

        EventBus eventBus = new EventBus("test");
        MultipleListener multiListener = new MultipleListener();

        eventBus.register(multiListener);

        eventBus.post(new Integer(100));
        eventBus.post(new Integer(200));
        eventBus.post(new Integer(300));
        eventBus.post(new Long(800));
        eventBus.post(new Long(800990));
        eventBus.post(new Long(800882934));

        System.out.println("LastInteger:"+multiListener.getLastInteger());
        System.out.println("LastLong:"+multiListener.getLastLong());
    }
}

//输出信息
event Integer:100
event Integer:200
event Integer:300
event Long:800
event Long:800990
event Long:800882934
LastInteger:300
LastLong:800882934
```

**Dead Event：**

　　如果EventBus发送的消息都不是订阅者关心的称之为Dead Event。实例如下：

```Java
public class DeadEventListener {
    boolean notDelivered = false;

    @Subscribe
    public void listen(DeadEvent event) {

        notDelivered = true;
    }

    public boolean isNotDelivered() {
        return notDelivered;
    }
}
```

　　测试类：

```Java
public class TestDeadEventListeners {
    @Test
    public void testDeadEventListeners() throws Exception {

        EventBus eventBus = new EventBus("test");
        DeadEventListener deadEventListener = new DeadEventListener();
        eventBus.register(deadEventListener);

        eventBus.post(new TestEvent(200));
        eventBus.post(new TestEvent(300));

        System.out.println("deadEvent:"+deadEventListener.isNotDelivered());

    }
}

//输出信息
event message:200
event message:300
deadEvent:true
```

　　说明：如果没有消息订阅者监听消息， EventBus将发送DeadEvent消息，这时我们可以通过log的方式来记录这种状态。

　　**Event的继承：**

　　如果Listener A监听Event A, 而Event A有一个子类Event B, 此时Listener A将同时接收Event A和B消息，实例如下：

　　Listener 类：

```Java
public class NumberListener {

    private Number lastMessage;

    @Subscribe
    public void listen(Number integer) {
        lastMessage = integer;
        System.out.println("Message:"+lastMessage);
    }

    public Number getLastMessage() {
        return lastMessage;
    }
}

public class IntegerListener {

    private Integer lastMessage;

    @Subscribe
    public void listen(Integer integer) {
        lastMessage = integer;
        System.out.println("Message:"+lastMessage);
    }

    public Integer getLastMessage() {
        return lastMessage;
    }
}
```

　　测试类：

```Java
public class TestEventsFromSubclass {
    @Test
    public void testEventsFromSubclass() throws Exception {

        EventBus eventBus = new EventBus("test");
        IntegerListener integerListener = new IntegerListener();
        NumberListener numberListener = new NumberListener();
        eventBus.register(integerListener);
        eventBus.register(numberListener);

        eventBus.post(new Integer(100));

        System.out.println("integerListener message:"+integerListener.getLastMessage());
        System.out.println("numberListener message:"+numberListener.getLastMessage());

        eventBus.post(new Long(200L));

        System.out.println("integerListener message:"+integerListener.getLastMessage());
        System.out.println("numberListener message:"+numberListener.getLastMessage());
    }
}

//输出类
Message:100
Message:100
integerListener message:100
numberListener message:100
Message:200
integerListener message:100
numberListener message:200
```

　　说明：在这个方法中,我们看到第一个事件(新的整数(100))是收到两个听众,但第二个(新长(200 l))只能到达NumberListener作为整数一不是创建这种类型的事件。可以使用此功能来创建更通用的监听器监听一个广泛的事件和更详细的具体的特殊的事件。

 　　一个综合实例：

```Java
public class UserThread extends Thread {
    private Socket connection;
    private EventBus channel;
    private BufferedReader in;
    private PrintWriter out;

    public UserThread(Socket connection, EventBus channel) {
        this.connection = connection;
        this.channel = channel;
        try {
            in = new BufferedReader(new InputStreamReader(connection.getInputStream()));
            out = new PrintWriter(connection.getOutputStream(), true);
        } catch (IOException e) {
            e.printStackTrace();
            System.exit(1);
        }
    }

    @Subscribe
    public void recieveMessage(String message) {
        if (out != null) {
            out.println(message);
            System.out.println("recieveMessage:"+message);
        }
    }

    @Override
    public void run() {
        try {
            String input;
            while ((input = in.readLine()) != null) {
                channel.post(input);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }

        //reached eof
        channel.unregister(this);
        try {
            connection.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        in = null;
        out = null;
    }
}
```

```Java
mport java.io.IOException;
import java.net.ServerSocket;
import java.net.Socket;

import com.google.common.eventbus.EventBus;

public class EventBusChat {
    public static void main(String[] args) {
        EventBus channel = new EventBus();
        ServerSocket socket;
        try {
            socket = new ServerSocket(4444);
            while (true) {
                Socket connection = socket.accept();
                UserThread newUser = new UserThread(connection, channel);
                channel.register(newUser);
                newUser.start();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

　　说明：用telnet命令登录：telnet 127.0.0.1 4444 ，如果你连接多个实例你会看到任何消息发送被传送到其他实例。