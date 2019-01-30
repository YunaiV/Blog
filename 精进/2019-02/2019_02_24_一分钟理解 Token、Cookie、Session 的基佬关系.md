title: 一分钟理解 Token、Cookie、Session 的基佬关系
date: 2019-02-24
tags:
categories: 精进
permalink: Fight/One-minute-understanding-of-Token-Cookie-and-Session
author: 一只躲在角落里的小刺猬
from_url: https://www.jianshu.com/p/8ef0c5a551d3
wechat_url:

-------

摘要: 原创出处 https://www.jianshu.com/p/8ef0c5a551d3 「一只躲在角落里的小刺猬」欢迎转载，保留摘要，谢谢！

- [Cookie](http://www.iocoder.cn/Fight/One-minute-understanding-of-Token-Cookie-and-Session/)
- [Session](http://www.iocoder.cn/Fight/One-minute-understanding-of-Token-Cookie-and-Session/)
- [Token](http://www.iocoder.cn/Fight/One-minute-understanding-of-Token-Cookie-and-Session/)
- [小结](http://www.iocoder.cn/Fight/One-minute-understanding-of-Token-Cookie-and-Session/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

在Web应用中，HTTP请求是无状态的。即：用户第一次发起请求，与服务器建立连接并登录成功后，为了避免每次打开一个页面都需要登录一下，就出现了cookie，Session。

# Cookie

Cookie是客户端保存用户信息的一种机制，用来记录用户的一些信息，也是实现Session的一种方式。Cookie存储的数据量有限，且都是保存在客户端浏览器中。不同的浏览器有不同的存储大小，但一般不超过4KB。因此使用Cookie实际上只能存储一小段的文本信息。
 例如：登录网站，今输入用户名密码登录了，第二天再打开很多情况下就直接打开了。这个时候用到的一个机制就是Cookie。

# Session

Session是另一种记录客户状态的机制，它是在服务端保存的一个数据结构（主要存储的的SessionID和Session内容，同时也包含了很多自定义的内容如：用户基础信息、权限信息、用户机构信息、固定变量等），这个数据可以保存在集群、数据库、文件中，用于跟踪用户的状态。客户端浏览器访问服务器的时候，服务器把客户端信息以某种形式记录在服务器上。这就是Session。客户端浏览器再次访问时只需要从该Session中查找该客户的状态就可以了。

用户第一次登录后，浏览器会将用户信息发送给服务器，服务器会为该用户创建一个SessionId，并在响应内容（Cookie）中将该SessionId一并返回给浏览器，浏览器将这些数据保存在本地.  当用户再次发送请求时，浏览器会自动的把上次请求存储的Cookie数据自动的携带给服务器。服务器接收到请求信息后，会通过浏览器请求的数据中的SessionId判断当前是哪个用户，然后根据SessionId在Session库中获取用户的Session数据返回给浏览器。

例如：购物车，添加了商品之后客户端处可以知道添加了哪些商品，而服务器端如何判别呢，所以也需要存储一些信息就用到了Session。

如果说Cookie机制是通过检查客户身上的“通行证”来确定客户身份的话，那么Session机制就是通过检查服务器上的“客户明细表”来确认客户身份。Session相当于程序在服务器上建立的一份客户档案，客户来访的时候只需要查询客户档案表就可以了。Session生成后，只要用户继续访问，服务器就会更新Session的最后访问时间，并维护该Session。为防止内存溢出，服务器会把长时间内没有活跃的Session从内存删除。这个时间就是Session的超时时间。如果超过了超时时间没访问过服务器，Session就自动失效了。



![](https:////upload-images.jianshu.io/upload_images/1955673-c57a27def48c6cde.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/418/format/jpeg)


# Token

HTTP请求都是以无状态的形式对接。即HTTP服务器不知道本次请求和上一次请求是否有关联。所以就有了Session的引入，即服务端和客户端都保存一段文本，客户端每次发起请求都带着，这样服务器就知道客户端是否发起过请求。
 这样，就导致客户端频繁向服务端发出请求数据，服务端频繁的去数据库查询用户名和密码并进行对比，判断用户名和密码正确与否。而Session的存储是需要空间的，频繁的查询数据库给服务器造成很大的压力。
 在这种情况下，Token应用而生。

Token是服务端生成的一串字符串，以作客户端进行请求的一个令牌。当客户端第一次访问服务端，服务端会根据传过来的唯一标识userId，运用一些算法，并加上密钥，生成一个Token，然后通过BASE64编码一下之后将这个Token返回给客户端，客户端将Token保存起来（可以通过数据库或文件形式保存本地）。下次请求时，客户端只需要带上Token，服务器收到请求后，会用相同的算法和密钥去验证Token。

最简单的Token组成:uid(用户唯一的身份标识)、time(当前时间的时间戳)、sign(签名，由Token的前几位+盐以哈希算法压缩成一定长的十六进制字符串，可以防止恶意第三方拼接Token请求服务器)。

使用基于 Token 的身份验证方法，在服务端不需要存储用户的登录记录。大概的流程是这样的：

客户端使用用户名跟密码请求登录
 服务端收到请求，去验证用户名与密码
 验证成功后，服务端会签发一个 Token，再把这个 Token 发送给客户端
 客户端收到 Token 以后可以把它存储起来，比如放在 Cookie 里或者数据库里
 客户端每次向服务端请求资源的时候需要带着服务端签发的 Token
 服务端收到请求，然后去验证客户端请求里面带着的 Token，如果验证成功，就向客户端返回请求的数据

APP登录的时候发送加密的用户名和密码到服务器，服务器验证用户名和密码，如果成功，以某种方式比如随机生成32位的字符串作为Token，存储到服务器中，并返回Token到APP，以后APP请求时，凡是需要验证的地方都要带上该Token，然后服务器端验证Token，成功返回所需要的结果，失败返回错误信息，让他重新登录。对于同一个APP同一个手机当前只有一个Token；手机APP会存储一个当前有效的Token。其中服务器上Token设置一个有效期，每次APP请求的时候都验证Token和有效期。



![](https:////upload-images.jianshu.io/upload_images/1955673-73ea1d7b7225df91.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/jpeg)


# 小结

下面这个例子，可以很好的理解：

『给我来份煎饼（token我是你对面摊卖烤冷面的，scope赊账）』『好』
 『鸡蛋（token我是你对面摊卖烤冷面的，scope赊账）』『好』
 『再加个鸡蛋（token我是你对面摊卖烤冷面的，scope赊账）』『好』

最终得到一份普通煎饼，外加两个鸡蛋……

如果服务器重启或者因为其他理由，服务器端已保存token丢失。那么用户需 要重新登录和认证。

『给我来份煎饼（token我是你对面摊卖烤冷面的）』『那个……我没见过你』