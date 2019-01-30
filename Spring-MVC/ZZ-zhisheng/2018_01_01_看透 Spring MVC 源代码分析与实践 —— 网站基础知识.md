title: 【zhisheng】看透 Spring MVC 源代码分析与实践 —— 网站基础知识
date: 2018-01-01
tag: 
categories: Spring-MVC
permalink: Spring-MVC/zhisheng/Spring-MVC01/
author: zhisheng
from_url: http://www.54tianzhisheng.cn/2017/07/14/Spring-MVC01/
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247484162&idx=1&sn=17c1816deb182ce2612591429d4a58db&chksm=fa497cb3cd3ef5a512c17d7df2fc7d95e083721c1b747f6f1d6111eccabcfeecebc81d67971d#rd

-------

摘要: 原创出处 http://www.54tianzhisheng.cn/2017/07/14/Spring-MVC01/ 「zhisheng」欢迎转载，保留摘要，谢谢！

- [网站架构及其演变过程](http://www.iocoder.cn/Spring-MVC/zhisheng/)
  - [基础结构](http://www.iocoder.cn/Spring-MVC/zhisheng/)
  - [海量数据的解决方案](http://www.iocoder.cn/Spring-MVC/zhisheng/)
  - [高并发的解决方案](http://www.iocoder.cn/Spring-MVC/zhisheng/)
- [常见协议和标准](http://www.iocoder.cn/Spring-MVC/zhisheng/)
  - [TCP/IP 协议](http://www.iocoder.cn/Spring-MVC/zhisheng/)
- [DNS 的设置](http://www.iocoder.cn/Spring-MVC/zhisheng/)
  - [DNS 解析](http://www.iocoder.cn/Spring-MVC/zhisheng/)
- [Java 中 Socket 的用法](http://www.iocoder.cn/Spring-MVC/zhisheng/)
  - [普通 SOKET 的用法](http://www.iocoder.cn/Spring-MVC/zhisheng/)
  - [NIOSOCKET 的用法](http://www.iocoder.cn/Spring-MVC/zhisheng/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

### 网站架构及其演变过程

#### 基础结构

网络传输分解方式：

- 标准的 OSI 参考模型
- TCP/IP 参考模型

![OSI-TCPIP](http://zhisheng.iocoder.cn/OSI-TCPIP.jpg)

#### 海量数据的解决方案

- 缓存和页面静态化

  - 缓存
    - 通过程序直接保存在内存中
    - 使用缓存框架 （Encache、Redis、Memcache）
  - 页面静态化
    - 使用模板技术生成（Velocity、FreeMaker等）

- 数据库优化

  - 表结构优化
  - SQL 语句优化
  - 分区
  - 分表
  - 索引优化
  - 使用存储过程代替直接操作过程

- 分离活跃数据

- 批量读取和延迟修改

- 读写分离

  ![read-write](http://zhisheng.iocoder.cn/read-write.jpg)

- 分布式数据库

  ![sql-1](http://zhisheng.iocoder.cn/sql-1.jpg)

- NoSQL 和 Hadoop

#### 高并发的解决方案

- 应用和静态资源的分离：静态文件（图片、视频、JS、CSS等）放在专门的服务器上
- 页面缓存（Nginx 服务器、Squid 服务器）
- 集群与分布式
- 反向代理
- CDN
- 底层优化：网络传输协议

### 常见协议和标准

#### TCP/IP 协议

IP：查找地址，对应着国际互联网

TCP：规范传输规则，对应着传输层

TCP 在传输之前会进行三次沟通，称 “三次握手”，传完数据断开的时候要进行四次沟通，称 “四次挥手”。

TCP 两个序号，三个标志位含义：

- seq：表示所传数据的序号。TCP 传输时每一个字节都有一个序号，发送数据的时候会将数据的第一个序号发送给对方，接收方会按序号检查是否接收完整了，如果没接收完整就需要重新传送，这样就可以保证数据的完整性。
- ack：表示确认号。接收端用它来给发送端反馈已经成功接收到的数据信息，它的值为希望接收的下一个数据包起始序号。
- ACK：确认位，只有 ACK = 1 的时候 ack 才起作用。正常通信时 ACK 为 1，第一次发起请求时因为没有需要确认接收的数据所以 ACK 为 0。
- SYN：同步位，用于在建立连接时同步序号。刚开始建立连接时并没有历史接收的数据，所以 ack 也就没有办法设置，这是按照正常的机制就无法运行了，SYN 的作用就是解决这个问题的，当接收端接收到 SYN = 1 的报文时就会直接将 ack 设置为接收到的 seq + 1 的值，注意这里的值并不是检验后设置的，而是根据 SYN 直接设置的，这样正常的机制就可以运行了，所以 SYN 叫同步位。SYN 会在前两次握手时都为 1，这是因为通信的双方的 ack 都需要设置一个初始值。
- FIN：终止位，用来在数据传输完毕后释放连接。

![TCP](http://zhisheng.iocoder.cn/TCP.jpg)

### DNS 的设置

#### DNS 解析

参考域名设置，如下是我在腾讯云域名的设置

![dns](http://zhisheng.iocoder.cn/dns.jpg)

记录类型：

**A记录：** 将域名指向一个IPv4地址（例如：8.8.8.8）
**CNAME**：将域名指向另一个域名（例如 www.54tianzhisheng.cn）
**MX**： 将域名指向邮件服务器地址
**TXT**： 可任意填写，长度限制255，通常做SPF记录（反垃圾邮件）
**NS**： 域名服务器记录，将子域名指定其他DNS服务器解析
**AAAA**：将域名指向一个iPv6地址（例如：ff06:0:0:0:0:0:0:c3）
**SRV**：记录提供特定服务的服务器（例如_xmpp-server._tcp）
**显性URL**：将域名301重定向到另一个地址
**隐性URL**：类似显性URL，但是会隐藏真实目标地址

主机记录：

**要解析 www.54tianzhisheng.cn，请填写 www。**
**主机记录就是域名前缀，常见用法有：**

**www:** 解析后的域名为 www.54tianzhisheng.cn。
**@:** 直接解析主域名 54tianzhisheng.cn。
***:** 泛解析，匹配其他所有域名 *.54tianzhisheng.cn。\**mail:** 将域名解析为 mail.54tianzhisheng.cn，通常用于解析邮箱服务器。**二级域名:** 如：abc.54tianzhisheng.cn，填写abc。*手机网站:* 如：m.54tianzhisheng.cn，填写m。

### Java 中 Socket 的用法

#### 普通 SOKET 的用法

Socket 分为 ServerSocket 和 Socket 两大类。

ServerSocket 用于服务器端，可以通过 accept 方法监听请求，监听到请求后返回 Socket；

Socket 用户具体完成数据传输，客户端直接使用 Socket 发送请求并传输数据。

随便写了个单方面发送消息的 demo：

客户端：

```Java
import java.io.IOException;
import java.io.OutputStream;
import java.net.Socket;

/**
 * Created by 10412 on 2017/5/2.
 * TCP客户端：
 ①：建立tcp的socket服务，最好明确具体的地址和端口。这个对象在创建时，就已经可以对指定ip和端口进行连接(三次握手)。
 ②：如果连接成功，就意味着通道建立了，socket流就已经产生了。只要获取到socket流中的读取流和写入流即可，只要通过getInputStream和getOutputStream就可以获取两个流对象。
 ③：关闭资源。
 */
//单方面的输入！
public class TcpClient
{
    public static void main(String[] args) {
        try {
            Socket s = new Socket("127.0.0.1", 9999);
            OutputStream o = s.getOutputStream();
            o.write("tcp sssss".getBytes());
            s.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

服务器端：

```Java
import java.io.IOException;
import java.io.InputStream;
import java.net.ServerSocket;
import java.net.Socket;

/**
 * Created by 10412 on 2017/5/2.
 */
public class TcpServer
{
    public static void main(String[] args) {
        try {
            ServerSocket ss = new ServerSocket(9999);//建立服务端的socket服务
            Socket s = ss.accept();//获取客户端对象
            String ip = s.getInetAddress().getHostAddress();
            int port = s.getPort();
            System.out.println(ip + " : " + port + " connected");

            // 可以通过获取到的socket对象中的socket流和具体的客户端进行通讯。
            InputStream ins = s.getInputStream();//读取客户端的数据，使用客户端对象的socket读取流
            byte[] bytes = new byte[1024];
            int len = ins.read(bytes);
            String text = new String(bytes, 0, len);
            System.out.println(text);
            //关闭资源
            s.close();
            ss.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

#### NIOSOCKET 的用法

见以前的一篇文章：[Java NIO 系列教程](http://www.54tianzhisheng.cn/2017/03/28/Java%20NIO%20%E7%B3%BB%E5%88%97%E6%95%99%E7%A8%8B/)

书中第五章简单的讲了下实现 HTTP 协议。第六章主要讲 Servlet，写了 Servlet 接口和其实现类。第七章把 Tomcat 分析的很不错，如果有读者感兴趣的话，可以去看看。

# 666. 彩蛋

如果你对 Java 感兴趣，欢迎加入我的知识星球一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)