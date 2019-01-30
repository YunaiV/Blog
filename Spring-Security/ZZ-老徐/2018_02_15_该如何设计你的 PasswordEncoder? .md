title: 【老徐】该如何设计你的 PasswordEncoder?
date: 2018-02-15
tag:
categories: Spring Security
permalink: Spring-Security/laoxu/PasswordEncoder
author: 老徐
from_url: https://www.cnkirito.moe/spring-security-6/
wechat_url:

-------

摘要: 原创出处 https://www.cnkirito.moe/spring-security-6/ 「老徐」欢迎转载，保留摘要，谢谢！

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

### 缘起

前端时间将一个集成了 spring-security-oauth2 的旧项目改造了一番，将 springboot 升级成了 springboot 2.0，众所周知 springboot 2.0 依赖的是 spring5，并且许多相关的依赖都发生了较大的改动，与本文相关的改动罗列如下，有兴趣的同学可以看看：[Spring Security 5.0 New Features](https://docs.spring.io/spring-security/site/docs/5.0.4.RELEASE/reference/htmlsingle/#new) ，增强了 oauth2 集成的功能以及和一个比较有意思的改动—重构了密码编码器的实现（Password Encoding，由于大多数 PasswordEncoder 相关的算法是 hash 算法，所以本文将 PasswordEncoder 翻译成‘密码编码器’和并非‘密码加密器’）官方称之为

[Modernized Password Encoding](https://docs.spring.io/spring-security/site/docs/5.0.4.RELEASE/reference/htmlsingle/#core-services-password-encoding) — 现代化的密码编码方式
另外，springboot2.0 的自动配置也做了一些调整，其中也有几点和 spring-security 相关，戳这里看所有细节 [springboot2.0 迁移指南](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.0-Migration-Guide)

一开始，我仅仅修改了依赖，将

```XML
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.5.4.RELEASE</version>
</parent>
```

升级成了

```XML
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.1.RELEASE</version>
</parent>
```

不出意料出现了兼容性的问题，我在尝试登陆时，出现了如下的报错

```Java
java.lang.IllegalArgumentException: There is no PasswordEncoder mapped for the id "null"
```

原因也很明显，正如 spring security 的更新文档中描述的那样，spring security 5 对 PasswordEncoder 做了相关的重构，原先默认配置的 PlainTextPasswordEncoder（明文密码）被移除了。这引起了我的兴趣，spring security 在新版本中对于 passwordEncoder 进行了哪些改造，这些改造背后又是出于什么样的目的呢？卖个关子，先从远古时期的案例来一步步演化出所谓的“现代化密码编码方式”。

### 密码存储演进史

自从互联网有了用户的那一刻起，存储用户密码这件事便成为了一个健全的系统不得不面对的一件事。远古时期，明文存储密码可能还不被认为是一个很大的系统缺陷（事实上这是一件很恐怖的事）。提及明文存储密码，我立刻联想到的是 CSDN 社区在 2011 年末发生的 600 万用户密码泄露的事件，谁也不会想到这个和程序员密切相关的网站会犯如此低级的错误。明文存储密码使得恶意用户可以通过 sql 注入等攻击方式来获取用户名和密码，虽然安全框架和良好的编码规范可以规避很多类似的攻击，但依旧避免不了系统管理员，DBA 有途径获取用户密码这一事实。事实上，不用明文存储存储密码，程序员们早在 n 多年前就已经达成了共识。

不能明文存储，一些 hash 算法便被广泛用做密码的编码器，对密码进行单向 hash 处理后存储数据库，当用户登录时，计算用户输入的密码的 hash 值，将两者进行比对。单向 hash 算法，顾名思义，它无法（或者用不能轻易更为合适）被反向解析还原出原密码。这杜绝了管理员直接获取密码的途径，可仅仅依赖于普通的 hash 算法（如 md5，sha256）是不合适的，他主要有 3 个特点：

1. 同一密码生成的 hash 值一定相同
2. 不同密码的生成的 hash 值可能相同（md5 的碰撞问题相比 sha256 还要严重）
3. 计算速度快。

以上三点结合在一起，破解此类算法成了不是那么困难的一件事，尤其是第三点，会在下文中再次提到，多快才算非常快？按照相关资料的说法：

> modern hardware perform billions of hash calculations a second.

考虑到大多数用户使用的密码多为数字+字母+特殊符号的组合，攻击者将常用的密码进行枚举，甚至通过排列组合来暴力破解，这被称为 rainbow table。算法爱好者能够立刻看懂到上述的方案，这被亲切地称之为—打表，一种暴力美学，这张表是可以被复用的。

虽然仅仅依赖于传统 hash 算法的思路被否决了，但这种 hash 后比对的思路，几乎被后续所有的优化方案继承。

hash 方案迎来的第一个改造是对引入一个“随机的因子”来掺杂进明文中进行 hash 计算，这样的随机因子通常被称之为盐 （salt）。salt 一般是用户相关的，每个用户持有各自的 salt。此时狗蛋和二丫的密码即使相同，由于 salt 的影响，存储在数据库中的密码也是不同的，除非…为每个用户单独建议一张 rainbow table。很明显 salted hash 相比普通的单向 hash 方案加大了 hacker 攻击的难度。但了解过 GPU 并行计算能力之强大的童鞋，都能够意识到，虽然破解 salted hash 比较麻烦，却并非不可行，勤劳勇敢的安全专家似乎也对这个方案不够满意。

为解决上述 salted hash 仍然存在的问题，一些新型的单向 hash 算法被研究了出来。其中就包括：Bcrypt，PBKDF2，Scrypt，Argon2。为什么这些 hash 算法能保证密码存储的安全性？因为他们足够慢，恰到好处的慢。这么说不严谨，只是为了给大家留个深刻的映像：慢。这类算法有一个特点，存在一个影响因子，可以用来控制计算强度，这直接决定了破解密码所需要的资源和时间，直观的体会可以见下图，在一年内破解如下算法所需要的硬件资源花费（折算成美元）

[![一年内破解如下算法所需要的硬件资源花费](http://kirito.iocoder.cn/1_QdbniDuZiiF1N7ArNJChOA.png)](http://kirito.iocoder.cn/1_QdbniDuZiiF1N7ArNJChOA.png)一年内破解如下算法所需要的硬件资源花费

这使得破解成了一件极其困难的事，并且，其中的计算强度因子是可控的，这样，即使未来量子计算机的计算能力爆表，也可以通过其控制计算强度以防破解。注意，普通的验证过程只需要计算一次 hash 计算，使用此类 hash 算法并不会影响到用户体验。

### 慢 hash 算法真的安全吗？

Bcrypt，Scrypt，PBKDF2 这些慢 hash 算法是目前最为推崇的 password encoding 方式，好奇心驱使我思考了这样一个问题：慢 hash 算法真的安全吗？

我暂时还没有精力仔细去研究他们中每一个算法的具体实现，只能通过一些文章来拾人牙慧，简单看看这几个算法的原理和安全性。

PBKDF2 被设计的很简单，它的基本原理是通过一个伪随机函数（例如 HMAC 函数），把明文和一个盐值作为输入参数，然后按照设置的计算强度因子重复进行运算，并最终产生密钥。这样的重复 hash 已经被认为足够安全，但也有人提出了不同意见，此类算法对于传统的 CPU 来说的确是足够安全，但 GPU 被搬了出来，前文提到过 GPU 的并行计算能力非常强大。

Bcrypt 强大的一点在于，其不仅仅是 CPU 密集型，还是 RAM 密集型！双重的限制因素，导致 GPU，ASIC（专用集成电路）无法应对 Bcrypt 带来的破解困境。

然后…看了 Scrypt 的相关资料之后我才意识到这个坑有多深。一个熟悉又陌生的词出现在了我面前：FPGA（现场可编程逻辑门阵列），这货就比较厉害了。现成的芯片指令结构如传统的 CPU，GPU，ASIC 都无法破解 Bcrypt，但是 FPGA 支持烧录逻辑门（如AND、OR、XOR、NOT），通过编程的方式烧录指令集的这一特性使得可以定制硬件来破解 Bcrypt。尽管我不认为懂这个技术的人会去想办法破解真正的系统，但，只要这是一个可能性，就总有方法会被发明出来与之对抗。Scrypt 比 Bcrypt 额外考虑到的就是大规模的[自定义硬件攻击](https://zh.wikipedia.org/w/index.php?title=%E5%AE%A2%E8%A3%BD%E7%A1%AC%E9%AB%94%E6%94%BB%E6%93%8A&action=edit&redlink=1) ，从而刻意设计需要大量内存运算。

理论终归是理论，实际上 Bcrypt 算法被发明至今 18 年，使用范围广，且从未因为安全问题而被修改，其有限性是已经被验证过的，相比之下 Scrypt 据我看到的文章显示是 9 年的历史，没有 Bcrypt 使用的广泛。从破解成本和权威性的角度来看，Bcrypt 用作密码编码器是不错的选择。

### spring security 废弃的接口

回到文档中，spring security 5 对 PasswordEncoder 做了相关的重构，原先默认配置的 PlainTextPasswordEncoder（明文密码）被移除了，想要做到明文存储密码，只能使用一个过期的类来过渡

```Java
@Bean
PasswordEncoder passwordEncoder(){
    return NoOpPasswordEncoder.getInstance();
}
```

实际上，spring security 提供了 BCryptPasswordEncoder 来进行密码编码，并作为了相关配置的默认配置，只不过没有暴露为全局的 Bean。使用明文存储的风险在文章一开始就已经强调过，NoOpPasswordEncoder 只能存在于 demo 中。

```Java
@Bean
PasswordEncoder passwordEncoder(){
    return new BCryptPasswordEncoder();
}
```

别忘了对你数据库中的密码进行同样的编码，否则无法对应。

### 更深层的思考

实际上，spring security 5 的另一个设计是促使我写成本文的初衷。

不知道有没有读者产生跟我相同的困扰：

1. 如果我要设计一个 QPS 很高的登录系统，使用 spring security 推荐的 BCrypt 会不会存在性能问题？
2. spring security 怎么这么坑，原来的密码编码器都给改了，我需要怎么迁移旧密码编码的应用程序？
3. 万一以后出了更高效的加密算法，这种笨重的硬编码方式配置密码编码器是不是不够灵活？

在 spring security 5 提供了这样一个思路，应该将密码编码之后的 hash 值和加密方式一起存储，并提供了一个 DelegatingPasswordEncoder 来作为众多密码密码编码方式的集合。

```Java
@Bean
PasswordEncoder passwordEncoder(){
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
}
```

负责生产 DelegatingPasswordEncoder 的工厂方法：

```Java
public class PasswordEncoderFactories {

   public static PasswordEncoder createDelegatingPasswordEncoder() {
      String encodingId = "bcrypt";
      Map<String, PasswordEncoder> encoders = new HashMap<>();
      encoders.put(encodingId, new BCryptPasswordEncoder());
      encoders.put("ldap", new LdapShaPasswordEncoder());
      encoders.put("MD4", new Md4PasswordEncoder());
      encoders.put("MD5", new MessageDigestPasswordEncoder("MD5"));
      encoders.put("noop", NoOpPasswordEncoder.getInstance());
      encoders.put("pbkdf2", new Pbkdf2PasswordEncoder());
      encoders.put("scrypt", new SCryptPasswordEncoder());
      encoders.put("SHA-1", new MessageDigestPasswordEncoder("SHA-1"));
      encoders.put("SHA-256", new MessageDigestPasswordEncoder("SHA-256"));
      encoders.put("sha256", new StandardPasswordEncoder());

      return new DelegatingPasswordEncoder(encodingId, encoders);
   }

   private PasswordEncoderFactories() {}
}
```

如此注入 PasswordEncoder 之后，我们在数据库中需要这么存储数据：

```
{bcrypt}$2a$10$dXJ3SW6G7P50lGmMkkmwe.20cQQubK3.HZWzG3YB1tlRy.fqvM/BG
{noop}password
{pbkdf2}5d923b44a6d129f3ddf3e3c8d29412723dcbde72445e8ef6bf3b508fbf17fa4ed4d6b99ca763d8dc
{scrypt}$e0801$8bWJaSu2IKSn9Z9kM+TPXfOc/9bdYSrN1oD9qfVThWEwdRTnO7re7Ei+fUZRJ68k9lTyuTeUp4of4g24hHnazw==$OAOec05+bXxvuu/1qZ6NUR+xQYvYv7BeL1QxwRpY5Pc=
{sha256}97cde38028ad898ebc02e690819fa220e88c62e0699403e94fff291cfffaf8410849f27605abcbc0
```

还记得文章开始的报错吗？

```Java
java.lang.IllegalArgumentException: There is no PasswordEncoder mapped for the id "null"
```

这个 id 就是因为我们没有为数据库中的密码添加 {bcrypt} 此类的前缀导致的。

> 你会不会担心密码泄露后，{bcrypt}，{pbkdf2}，{scrypt}，{sha256} 此类前缀会直接暴露密码的编码方式？其实这个考虑是多余的，因为密码存储的依赖算法并不是一个秘密。大多数能搞到你密码的 hacker 都可以轻松的知道你用的是什么算法，例如，bcrypt 算法通常以 \$2a$ 开头

稍微思考下，前面的三个疑问就可以迎刃而解，这就是文档中所谓的：**能够自适应服务器性能的现代化密码编码方案**。

### 参考

[Password Hashing: PBKDF2, Scrypt, Bcrypt](https://medium.com/@mpreziuso/password-hashing-pbkdf2-scrypt-bcrypt-1ef4bb9c19b3)

[core-services-password-encoding](https://docs.spring.io/spring-security/site/docs/current/reference/htmlsingle/#core-services-password-encoding)

### show me the code

spring security oauth2 的 github 代码示例，体会下 spring security 4 -> spring security 5 的相关变化。

<https://github.com/lexburner/oauth2-demo>

# 666. 彩蛋

如果你对 Spring Security 感兴趣，欢迎加入我的知识一起交流。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)