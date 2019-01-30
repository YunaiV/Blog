title: Canal 源码分析 —— server 模块
date: 2018-01-11
tag:
categories: Canal
permalink: Canal/tianshouzhi/server
author: 田守枝
from_url: http://www.tianshouzhi.com/api/tutorials/canal/382
wechat_url:

-------

摘要: 原创出处 http://www.tianshouzhi.com/api/tutorials/canal/382 「田守枝」欢迎转载，保留摘要，谢谢！

- [CanalServer接口](http://www.iocoder.cn/Canal/tianshouzhi/server/)
- [CanalServerWithNetty](http://www.iocoder.cn/Canal/tianshouzhi/server/)
- [SessionHandler](http://www.iocoder.cn/Canal/tianshouzhi/server/)
- [CanalServerWithEmbeded](http://www.iocoder.cn/Canal/tianshouzhi/server/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

server模块的核心接口是`CanalServer`，其有2个实现类`CanalServerWithNetty`、`CanalServerWithEmbeded`。关于CanalServer，官方文档中有有以下描述：

![4068E89A-F5FA-47DD-AA48-A39761487DE6.png](http://static.tianshouzhi.com/ueditor/upload/image/20180506/1525577031833085107.png)

  下图是笔者对官方文档的进一步描述：

![AD919783-7B87-4A19-93CF-FF542ED9E242.png](http://static.tianshouzhi.com/ueditor/upload/image/20180506/1525577045451010771.png)

**左边的图**

表示的是Canal独立部署。不同的应用通过canal client与canal server进行通信，所有的canal client的请求统一由CanalServerWithNetty接受，之后CanalServerWithNetty会将客户端请求派给CanalServerWithEmbeded 进行真正的处理。CannalServerWithEmbeded内部维护了多个canal instance，每个canal instance伪装成不同的mysql实例的slave，而CanalServerWithEmbeded会根据客户端请求携带的destination参数确定要由哪一个canal instance为其提供服务。

**右边的图**

是直接在应用中嵌入CanalServerWithEmbeded，不需要独立部署canal。很明显，网络通信环节少了，同步binlog信息的效率肯定更高。但是对于使用者的技术要求比较高。在应用中，我们可以通过CanalServerWithEmbeded.instance()方法来获得CanalServerWithEmbeded实例，这一个单例。

整个server模块源码目录结构如下所示：

![BE7FA9CC-0301-4B52-888A-BCD523BA60A1.png](http://static.tianshouzhi.com/ueditor/upload/image/20180506/1525577063564004656.png)

其中上面的红色框就是嵌入式实现，而下面的绿色框是基于Netty的实现。

看起来基于netty的实现代码虽然多一点，这其实只是幻觉，CanalServerWithNetty会将所有的请求委派给CanalServerWithEmbedded处理。

而内嵌的方式只有CanalServerWithEmbedded一个类， 是因为CanalServerWithEmbedded又要根据`destination`选择某个具体的`CanalInstance`来处理客户端请求，而CanalInstance的实现位于instance模块，我们将在之后分析。因此从canal server的角度来说，CanalServerWithEmbedded才是server模块真正的核心。

​        CanalServerWithNetty和CanalServerWithEmbedded都是单例的，提供了一个静态方法instance()获取对应的实例。回顾前一节分析CanalController源码时，在CanalController构造方法中准备CanalServer的相关代码，就是通过这两个静态方法获取对应的实例的。

```java
 public CanalController(final Properties properties){        ....     // 准备canal server        ip = getProperty(properties, CanalConstants.CANAL_IP);        port = Integer.valueOf(getProperty(properties, CanalConstants.CANAL_PORT));        embededCanalServer = CanalServerWithEmbedded.instance();        embededCanalServer.setCanalInstanceGenerator(instanceGenerator);// 设置自定义的instanceGenerator        canalServer = CanalServerWithNetty.instance();        canalServer.setIp(ip);        canalServer.setPort(port);       ....   }
```

# CanalServer接口

 CanalServer接口继承了CanalLifeCycle接口，主要是为了重新定义`start`和`stop`方法，抛出`CanalServerException`。

```java
public interface CanalServer extends CanalLifeCycle {     void start() throws CanalServerException;     void stop() throws CanalServerException;}
```

# CanalServerWithNetty

CanalServerWithNetty主要用于接受客户端的请求，然后将其委派给CanalServerWithEmbeded处理。下面的源码显示了CanalServerWithNetty种定义的字段和构造方法

```java
public class CanalServerWithNetty extends AbstractCanalLifeCycle implements CanalServer {    //监听的所有客户端请求都会为派给CanalServerWithEmbedded处理     private CanalServerWithEmbedded embeddedServer;      // 嵌入式server     //监听的ip和port，client通过此ip和port与服务端通信    private String                  ip;    private int                     port;     //netty组件    private Channel                 serverChannel = null;    private ServerBootstrap         bootstrap     = null;    //....单例模式实现    private CanalServerWithNetty(){        //给embeddedServer赋值        this.embeddedServer = CanalServerWithEmbedded.instance();    }    //... start and stop method    //...setters and getters...}
```

字段说明：

- **embeddedServer：**因为CanalServerWithNetty需要将请求委派给CanalServerWithEmbeded处理，因此其维护了embeddedServer对象。
- **ip、port：**这是netty监听的网络ip和端口，client通过这个ip和端口与server通信
- **serverChannel、bootstrap：**这是netty的API。其中`ServerBootstrap`用于启动服务端，通过调用其bind方法，返回一个类型为`Channel`的serverChannel对象，代表服务端通道。关于netty知识不是本教程重点，如果读者不熟悉，可以参考笔者的[netty教程](http://www.tianshouzhi.com/api/tutorials/netty)。

### **start方法**

start方法中包含了netty启动的核心逻辑，如下所示：

com.alibaba.otter.canal.server.netty.CanalServerWithNetty#start

```java
public void start() {        super.start();        //优先启动内嵌的canal server，因为基于netty的实现需要将请求委派给其处理        if (!embeddedServer.isStart()) {            embeddedServer.start();        }                 /* 创建bootstrap实例，参数NioServerSocketChannelFactory也是Netty的API，其接受2个线程池参数         其中第一个线程池是Accept线程池，第二个线程池是woker线程池，         Accept线程池接收到client连接请求后，会将代表client的对象转发给worker线程池处理。         这里属于netty的知识，不熟悉的用户暂时不必深究，简单认为netty使用线程来处理客户端的高并发请求即可。*/        this.bootstrap = new ServerBootstrap(new NioServerSocketChannelFactory(Executors.newCachedThreadPool(),            Executors.newCachedThreadPool()));                    /*pipeline实际上就是netty对客户端请求的处理器链，        可以类比JAVA EE编程中Filter的责任链模式，上一个filter处理完成之后交给下一个filter处理，        只不过在netty中，不再是filter，而是ChannelHandler。*/        bootstrap.setPipelineFactory(new ChannelPipelineFactory() {            public ChannelPipeline getPipeline() throws Exception {                ChannelPipeline pipelines = Channels.pipeline();               //主要是处理编码、解码。因为网路传输的传入的都是二进制流，FixedHeaderFrameDecoder的作用就是对其进行解析                pipelines.addLast(FixedHeaderFrameDecoder.class.getName(), new FixedHeaderFrameDecoder());               //处理client与server握手                pipelines.addLast(HandshakeInitializationHandler.class.getName(), new HandshakeInitializationHandler());               //client身份验证               pipelines.addLast(ClientAuthenticationHandler.class.getName(),                    new ClientAuthenticationHandler(embeddedServer));                //SessionHandler用于真正的处理客户端请求，是本文分析的重点               SessionHandler sessionHandler = new SessionHandler(embeddedServer);                pipelines.addLast(SessionHandler.class.getName(), sessionHandler);                return pipelines;            }        });                // 启动，当bind方法被调用时，netty开始真正的监控某个端口，此时客户端对这个端口的请求可以被接受到        if (StringUtils.isNotEmpty(ip)) {            this.serverChannel = bootstrap.bind(new InetSocketAddress(this.ip, this.port));        } else {            this.serverChannel = bootstrap.bind(new InetSocketAddress(this.port));        }    }
```

关于stop方法无非是一些关闭操作，代码很简单，这里不做介绍。

# SessionHandler

​    很明显的，canal处理client请求的核心逻辑都在SessionHandler这个处理器中。注意其在实例化时，传入了embeddedServer对象，前面我们提过，CanalServerWithNetty要将请求委派给CanalServerWithEmbedded处理，显然SessionHandler也要维护embeddedServer实例。

​    这里我们主要分析SessionHandler的 messageReceived方法，这个方法表示接受到了一个客户端请求，我们主要看的是SessionHandler如何对客户端请求进行解析，然后委派给CanalServerWithEmbedded处理的。为了体现其转发请求处理的核心逻辑，以下代码省去了大量源码片段，如下

SessionHandler#messageReceived

```java
public class SessionHandler extends SimpleChannelHandler {....//messageReceived方法表示收到客户端请求public void messageReceived(ChannelHandlerContext ctx, MessageEvent e) throws Exception {    ....      //根据客户端发送的网路通信包请求类型type，将请求委派embeddedServer处理        switch (packet.getType()) {            case SUBSCRIPTION://订阅请求                ...                embeddedServer.subscribe(clientIdentity);                         ...                break;            case UNSUBSCRIPTION://取消订阅请求                ...                embeddedServer.unsubscribe(clientIdentity);                ...                break;            case GET://获取binlog请求                ....                    if (get.getTimeout() == -1) {// 根据客户端是否指定了请求超时时间调用embeddedServer不同方法获取binlog                        message = embeddedServer.getWithoutAck(clientIdentity, get.getFetchSize());                    } else {                        ...                        message = embeddedServer.getWithoutAck(clientIdentity,                            get.getFetchSize(),                            get.getTimeout(),                            unit);                    }                ...                   break;            case CLIENTACK://客户端消费成功ack请求               ...                  embeddedServer.ack(clientIdentity, ack.getBatchId());               ...                break;            case CLIENTROLLBACK://客户端消费失败回滚请求                ...                    if (rollback.getBatchId() == 0L) {                        embeddedServer.rollback(clientIdentity);// 回滚所有批次                    } else {                        embeddedServer.rollback(clientIdentity, rollback.getBatchId()); // 只回滚单个批次                    }                ...                break;            default://无法判断请求类型                NettyUtils.error(400, MessageFormatter.format("packet type={} is NOT supported!", packet.getType())                    .getMessage(), ctx.getChannel(), null);                break;        }    ...}...}
```

​    可以看到，SessionHandler对client请求进行解析后，根据请求类型，委派给CanalServerWithEmbedded的相应方法进行处理。因此核心逻辑都在CanalServerWithEmbedded中。

# CannalServerWithEmbeded

`CanalServerWithEmbedded`实现了CanalServer和`CanalService`Can两个接口。其内部维护了一个Map，key为destination，value为对应的CanalInstance，根据客户端请求携带的destination参数将其转发到对应的CanalInstance上去处理。

```java
public class CanalServerWithEmbedded extends AbstractCanalLifeCycle implements CanalServer, CanalService {    ...    //key为destination，value为对应的CanalInstance。    private Map<String, CanalInstance> canalInstances;    ...}
```



对于CanalServer接口中定义的start和stop这两个方法实现比较简单，这里不再赘述。

​     在上面的SessionHandler源码分析中，我们已经看到，会根据请求报文的类型，会调用CanalServerWithEmbedded的相应方法，这些方法都定义在`CanalService`接口中，如下：

```java
public interface CanalService {   //订阅    void subscribe(ClientIdentity clientIdentity) throws CanalServerException;   //取消订阅    void unsubscribe(ClientIdentity clientIdentity) throws CanalServerException;   //比例获取数据，并自动自行ack    Message get(ClientIdentity clientIdentity, int batchSize) throws CanalServerException;   //超时时间内批量获取数据，并自动进行ack    Message get(ClientIdentity clientIdentity, int batchSize, Long timeout, TimeUnit unit) throws CanalServerException;    //批量获取数据，不进行ack    Message getWithoutAck(ClientIdentity clientIdentity, int batchSize) throws CanalServerException;   //超时时间内批量获取数据，不进行ack    Message getWithoutAck(ClientIdentity clientIdentity, int batchSize, Long timeout, TimeUnit unit)                                                                                               throws CanalServerException;   //ack某个批次的数据    void ack(ClientIdentity clientIdentity, long batchId) throws CanalServerException;   //回滚所有没有ack的批次的数据    void rollback(ClientIdentity clientIdentity) throws CanalServerException;   //回滚某个批次的数据    void rollback(ClientIdentity clientIdentity, Long batchId) throws CanalServerException;}
```

 细心地的读者会发现，每个方法中都包含了一个`ClientIdentity`类型参数，这就是客户端身份的标识。

```java
public class ClientIdentity implements Serializable {    private String destination;    private short  clientId;    private String filter; ...}
```

CanalServerWithEmbedded就是根据ClientIdentity中的destination参数确定这个请求要交给哪个CanalInstance处理的。



下面一次分析每一个方法的作用：

subscribe方法：

subscribe主要用于处理客户端的订阅请求，目前情况下，一个CanalInstance只能由一个客户端订阅，不过可以重复订阅。订阅主要的处理步骤如下：

1、根据客户端要订阅的destination，找到对应的CanalInstance

2、通过这个CanalInstance的`CanalMetaManager`组件记录下有客户端订阅。

3、获取客户端当前订阅位置(`Position`)。首先尝试从CanalMetaManager中获取，CanalMetaManager 中记录了某个client当前订阅binlog的位置信息。如果是第一次订阅，肯定无法获取到这个位置，则尝试从CanalEventStore中获取第一个binlog的位置。从`CanalEventStore`中获取binlog位置信息的逻辑是：CanalInstance一旦启动，就会立刻去拉取binlog，存储到CanalEventStore中，在第一次订阅的情况下，CanalEventStore中的第一条binlog的位置，就是当前客户端当前消费的开始位置。

4、通知CanalInstance订阅关系变化

```java
/** * 客户端订阅，重复订阅时会更新对应的filter信息 */@Overridepublic void subscribe(ClientIdentity clientIdentity) throws CanalServerException {    checkStart(clientIdentity.getDestination());    //1、根据客户端要订阅的destination，找到对应的CanalInstance     CanalInstance canalInstance = canalInstances.get(clientIdentity.getDestination());    if (!canalInstance.getMetaManager().isStart()) {        canalInstance.getMetaManager().start();    }  //2、通过CanalInstance的CanalMetaManager组件进行元数据管理，记录一下当前这个CanalInstance有客户端在订阅    canalInstance.getMetaManager().subscribe(clientIdentity); // 执行一下meta订阅  //3、获取客户端当前订阅的binlog位置(Position)，首先尝试从CanalMetaManager中获取    Position position = canalInstance.getMetaManager().getCursor(clientIdentity);    if (position == null) {  //3.1 如果是第一次订阅，尝试从CanalEventStore中获取第一个binlog的位置，作为客户端订阅开始的位置。        position = canalInstance.getEventStore().getFirstPosition();// 获取一下store中的第一条        if (position != null) {            canalInstance.getMetaManager().updateCursor(clientIdentity, position); // 更新一下cursor        }        logger.info("subscribe successfully, {} with first position:{} ", clientIdentity, position);    } else {        logger.info("subscribe successfully, use last cursor position:{} ", clientIdentity, position);    }    //4 通知下订阅关系变化    canalInstance.subscribeChange(clientIdentity);}
```

unsubscribe方法：

   unsubscribe方法主要用于取消订阅关系。在下面的代码中，我们可以看到，其实就是找到CanalInstance对应的CanalMetaManager，调用其unsubscribe取消这个订阅记录。需要注意的是，取消订阅并不意味着停止CanalInstance。当某个客户端取消了订阅，还会有新的client来订阅这个CanalInstance，所以不能停。

```java
/** * 取消订阅 */@Overridepublic void unsubscribe(ClientIdentity clientIdentity) throws CanalServerException {    CanalInstance canalInstance = canalInstances.get(clientIdentity.getDestination());    canalInstance.getMetaManager().unsubscribe(clientIdentity); // 执行一下meta订阅    logger.info("unsubscribe successfully, {}", clientIdentity);}
```



listAllSubscribe方法：

​    这一个管理方法，其作用是列出订阅某个destination的所有client。这里返回的是一个List<ClientIdentity>，不过我们已经多次提到，目前一个destination只能由一个client订阅。这里之所以返回一个list，是canal原先计划要支持多个client订阅同一个destination。不过，这个功能一直没有实现。所以List中，实际上只会包含一个ClientIdentity。

```java
/** * 查询所有的订阅信息 */public List<ClientIdentity> listAllSubscribe(String destination) throws CanalServerException {    CanalInstance canalInstance = canalInstances.get(destination);    return canalInstance.getMetaManager().listAllSubscribeInfo(destination);}
```

listBatchIds方法:

```java
/** * 查询当前未被ack的batch列表，batchId会按照从小到大进行返回 */public List<Long> listBatchIds(ClientIdentity clientIdentity) throws CanalServerException {    checkStart(clientIdentity.getDestination());    checkSubscribe(clientIdentity);    CanalInstance canalInstance = canalInstances.get(clientIdentity.getDestination());    Map<Long, PositionRange> batchs = canalInstance.getMetaManager().listAllBatchs(clientIdentity);    List<Long> result = new ArrayList<Long>(batchs.keySet());    Collections.sort(result);    return result;}
```

getWithoutAck方法：

getWithoutAck方法用于客户端获取binlog消息 ，一个获取一批(batch)的binlog，canal会为这批binlog生成一个唯一的batchId。客户端如果消费成功，则调用ack方法对这个批次进行确认。如果失败的话，可以调用rollback方法进行回滚。客户端可以连续多次调用getWithoutAck方法来获取binlog，在ack的时候，需要按照获取到binlog的先后顺序进行ack。如果后面获取的binlog被ack了，那么之前没有ack的binlog消息也会自动被ack。

getWithoutAck方法大致工作步骤如下所示：

1. 根据destination找到要从哪一个CanalInstance中获取binlog消息。
2. 确定从哪一个位置(Position)开始继续消费binlog。通常情况下，这个信息是存储在CanalMetaManager中。特别的，在第一次获取的时候，CanalMetaManager 中还没有存储任何binlog位置信息。此时CanalEventStore中存储的第一条binlog位置，则应该client开始消费的位置。
3. 根据Position从CanalEventStore中获取binlog。为了尽量提高效率，一般一次获取一批binlog，而不是获取一条。这个批次的大小(batchSize)由客户端指定。同时客户端可以指定超时时间，在超时时间内，如果获取到了batchSize的binlog，会立即返回。 如果超时了还没有获取到batchSize指定的binlog个数，也会立即返回。特别的，如果没有设置超时时间，如果没有获取到binlog也立即返回。
4. 在CanalMetaManager中记录这个批次的binlog消息。CanalMetaManager会为获取到的这个批次的binlog生成一个唯一的batchId，batchId是递增的。如果binlog信息为空，则直接把batchId设置为-1。

```java
@Overridepublic Message getWithoutAck(ClientIdentity clientIdentity, int batchSize) throws CanalServerException {    return getWithoutAck(clientIdentity, batchSize, null, null);}/** * <pre> * 几种case: * a. 如果timeout为null，则采用tryGet方式，即时获取 * b. 如果timeout不为null *    1. timeout为0，则采用get阻塞方式，获取数据，不设置超时，直到有足够的batchSize数据才返回 *    2. timeout不为0，则采用get+timeout方式，获取数据，超时还没有batchSize足够的数据，有多少返回多少 * </pre> */@Overridepublic Message getWithoutAck(ClientIdentity clientIdentity, int batchSize, Long timeout, TimeUnit unit)                                                                                                       throws CanalServerException {    checkStart(clientIdentity.getDestination());    checkSubscribe(clientIdentity);      // 1、根据destination找到要从哪一个CanalInstance中获取binlog消息    CanalInstance canalInstance = canalInstances.get(clientIdentity.getDestination());    synchronized (canalInstance) {        //2、从CanalMetaManager中获取最后一个没有ack的binlog批次的位置信息。        PositionRange<LogPosition> positionRanges = canalInstance.getMetaManager().getLastestBatch(clientIdentity);   //3 从CanalEventStore中获取binlog        Events<Event> events = null;        if (positionRanges != null) { // 3.1 如果从CanalMetaManager获取到了位置信息，从当前位置继续获取binlog            events = getEvents(canalInstance.getEventStore(), positionRanges.getStart(), batchSize, timeout, unit);        } else { //3.2 如果没有获取到binlog位置信息，从当前store中的第一条开始获取            Position start = canalInstance.getMetaManager().getCursor(clientIdentity);            if (start == null) { // 第一次，还没有过ack记录，则获取当前store中的第一条                start = canalInstance.getEventStore().getFirstPosition();            }      // 从CanalEventStore中获取binlog消息            events = getEvents(canalInstance.getEventStore(), start, batchSize, timeout, unit);        }        //4 记录批次信息到CanalMetaManager中        if (CollectionUtils.isEmpty(events.getEvents())) {          //4.1 如果获取到的binlog消息为空，构造一个空的Message对象，将batchId设置为-1返回给客户端            logger.debug("getWithoutAck successfully, clientId:{} batchSize:{} but result is null", new Object[] {                    clientIdentity.getClientId(), batchSize });            return new Message(-1, new ArrayList<Entry>()); // 返回空包，避免生成batchId，浪费性能        } else {           //4.2 如果获取到了binlog消息，将这个批次的binlog消息记录到CanalMetaMaager中，并生成一个唯一的batchId            Long batchId = canalInstance.getMetaManager().addBatch(clientIdentity, events.getPositionRange());            //将Events转为Entry            List<Entry> entrys = Lists.transform(events.getEvents(), new Function<Event, Entry>() {                public Entry apply(Event input) {                    return input.getEntry();                }            });            logger.info("getWithoutAck successfully, clientId:{} batchSize:{}  real size is {} and result is [batchId:{} , position:{}]",                clientIdentity.getClientId(),                batchSize,                entrys.size(),                batchId,                events.getPositionRange());            //构造Message返回            return new Message(batchId, entrys);        }    }}/** * 根据不同的参数，选择不同的方式获取数据 */private Events<Event> getEvents(CanalEventStore eventStore, Position start, int batchSize, Long timeout,                                TimeUnit unit) {    if (timeout == null) {        return eventStore.tryGet(start, batchSize);    } else {        try {            if (timeout <= 0) {                return eventStore.get(start, batchSize);            } else {                return eventStore.get(start, batchSize, timeout, unit);            }        } catch (Exception e) {            throw new CanalServerException(e);        }    }}
```

ack方法：

ack方法时客户端用户确认某个批次的binlog消费成功。进行 batch id 的确认。确认之后，小于等于此 batchId 的 Message 都会被确认。注意：进行反馈时必须按照batchId的顺序进行ack(需有客户端保证)

ack时需要做以下几件事情：

1. 从CanalMetaManager中，移除这个批次的信息。在getWithoutAck方法中，将批次的信息记录到了CanalMetaManager中，ack时移除。
2. 记录已经成功消费到的binlog位置，以便下一次获取的时候可以从这个位置开始，这是通过CanalMetaManager记录的。
3. 从CanalEventStore中，将这个批次的binlog内容移除。因为已经消费成功，继续保存这些已经消费过的binlog没有任何意义，只会白白占用内存。

```java
@Overridepublic void ack(ClientIdentity clientIdentity, long batchId) throws CanalServerException {    checkStart(clientIdentity.getDestination());    checkSubscribe(clientIdentity);    CanalInstance canalInstance = canalInstances.get(clientIdentity.getDestination());    PositionRange<LogPosition> positionRanges = null;   //1 从CanalMetaManager中，移除这个批次的信息    positionRanges = canalInstance.getMetaManager().removeBatch(clientIdentity, batchId); // 更新位置    if (positionRanges == null) { // 说明是重复的ack/rollback        throw new CanalServerException(String.format("ack error , clientId:%s batchId:%d is not exist , please check",            clientIdentity.getClientId(),            batchId));    }    //2、记录已经成功消费到的binlog位置，以便下一次获取的时候可以从这个位置开始，这是通过CanalMetaManager记录的    if (positionRanges.getAck() != null) {        canalInstance.getMetaManager().updateCursor(clientIdentity, positionRanges.getAck());        logger.info("ack successfully, clientId:{} batchId:{} position:{}",            clientIdentity.getClientId(),            batchId,            positionRanges);    }      //3、从CanalEventStore中，将这个批次的binlog内容移除    canalInstance.getEventStore().ack(positionRanges.getEnd());}
```

rollback方法：

```java
/** * 回滚到未进行 {@link #ack} 的地方，下次fetch的时候，可以从最后一个没有 {@link #ack} 的地方开始拿 */@Overridepublic void rollback(ClientIdentity clientIdentity) throws CanalServerException {    checkStart(clientIdentity.getDestination());    CanalInstance canalInstance = canalInstances.get(clientIdentity.getDestination());    // 因为存在第一次链接时自动rollback的情况，所以需要忽略未订阅    boolean hasSubscribe = canalInstance.getMetaManager().hasSubscribe(clientIdentity);    if (!hasSubscribe) {        return;    }    synchronized (canalInstance) {        // 清除batch信息        canalInstance.getMetaManager().clearAllBatchs(clientIdentity);        // rollback eventStore中的状态信息        canalInstance.getEventStore().rollback();        logger.info("rollback successfully, clientId:{}", new Object[] { clientIdentity.getClientId() });    }}/** * 回滚到未进行 {@link #ack} 的地方，下次fetch的时候，可以从最后一个没有 {@link #ack} 的地方开始拿 */@Overridepublic void rollback(ClientIdentity clientIdentity, Long batchId) throws CanalServerException {    checkStart(clientIdentity.getDestination());    CanalInstance canalInstance = canalInstances.get(clientIdentity.getDestination());    // 因为存在第一次链接时自动rollback的情况，所以需要忽略未订阅    boolean hasSubscribe = canalInstance.getMetaManager().hasSubscribe(clientIdentity);    if (!hasSubscribe) {        return;    }    synchronized (canalInstance) {        // 清除batch信息        PositionRange<LogPosition> positionRanges = canalInstance.getMetaManager().removeBatch(clientIdentity,            batchId);        if (positionRanges == null) { // 说明是重复的ack/rollback            throw new CanalServerException(String.format("rollback error, clientId:%s batchId:%d is not exist , please check",                clientIdentity.getClientId(),                batchId));        }        // lastRollbackPostions.put(clientIdentity,        // positionRanges.getEnd());// 记录一下最后rollback的位置        // TODO 后续rollback到指定的batchId位置        canalInstance.getEventStore().rollback();// rollback                                                 // eventStore中的状态信息        logger.info("rollback successfully, clientId:{} batchId:{} position:{}",            clientIdentity.getClientId(),            batchId,            positionRanges);    }}
```

get方法：

与getWithoutAck主要流程完全相同，唯一不同的是，在返回数据给用户前，直接进行了ack，而不管客户端消费是否成功。

```java
@Overridepublic Message get(ClientIdentity clientIdentity, int batchSize) throws CanalServerException {    return get(clientIdentity, batchSize, null, null);} /* * 几种case: * a. 如果timeout为null，则采用tryGet方式，即时获取 * b. 如果timeout不为null *    1. timeout为0，则采用get阻塞方式，获取数据，不设置超时，直到有足够的batchSize数据才返回 *    2. timeout不为0，则采用get+timeout方式，获取数据，超时还没有batchSize足够的数据，有多少返回多少 */@Overridepublic Message get(ClientIdentity clientIdentity, int batchSize, Long timeout, TimeUnit unit)                                                                                             throws CanalServerException {    checkStart(clientIdentity.getDestination());    checkSubscribe(clientIdentity);    CanalInstance canalInstance = canalInstances.get(clientIdentity.getDestination());    synchronized (canalInstance) {        // 获取到流式数据中的最后一批获取的位置        PositionRange<LogPosition> positionRanges = canalInstance.getMetaManager().getLastestBatch(clientIdentity);        if (positionRanges != null) {            throw new CanalServerException(String.format("clientId:%s has last batch:[%s] isn't ack , maybe loss data",                clientIdentity.getClientId(),                positionRanges));        }        Events<Event> events = null;        Position start = canalInstance.getMetaManager().getCursor(clientIdentity);        events = getEvents(canalInstance.getEventStore(), start, batchSize, timeout, unit);        if (CollectionUtils.isEmpty(events.getEvents())) {            logger.debug("get successfully, clientId:{} batchSize:{} but result is null", new Object[] {                    clientIdentity.getClientId(), batchSize });            return new Message(-1, new ArrayList<Entry>()); // 返回空包，避免生成batchId，浪费性能        } else {            // 记录到流式信息            Long batchId = canalInstance.getMetaManager().addBatch(clientIdentity, events.getPositionRange());            List<Entry> entrys = Lists.transform(events.getEvents(), new Function<Event, Entry>() {                public Entry apply(Event input) {                    return input.getEntry();                }            });            logger.info("get successfully, clientId:{} batchSize:{} real size is {} and result is [batchId:{} , position:{}]",                clientIdentity.getClientId(),                batchSize,                entrys.size(),                batchId,                events.getPositionRange());            // 直接提交ack            ack(clientIdentity, batchId);            return new Message(batchId, entrys);        }    }}
```

