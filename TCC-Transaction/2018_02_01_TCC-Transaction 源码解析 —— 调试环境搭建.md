title: TCC-Transaction 源码分析 —— 调试环境搭建
date: 2018-02-01
tags:
categories: TCC-Transaction
permalink: TCC-Transaction/build-debugging-environment

---

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

**本文主要基于 TCC-Transaction 1.2.3.3 正式版**  

- [1. 依赖工具](#)
- [2. 源码拉取](#)
- [3. 初始化数据库](#)
- [4. 启动 capital 项目](#)
- [5. 启动 redpacket 项目](#)
- [6. 启动 order 项目](#)
- [7. 交流](#)

# 1. 依赖工具

* Maven
* Git
* JDK
* MySQL
* IntelliJ IDEA

# 2. 源码拉取

从官方仓库 [https://github.com/changmingxie/tcc-transaction.git](https://github.com/changmingxie/tcc-transaction.git) `Fork` 出属于自己的仓库。为什么要 `Fork` ？既然开始阅读、调试源码，我们可能会写一些注释，有了自己的仓库，可以进行自由的提交。😈

使用 `IntelliJ IDEA` 从 `Fork` 出来的仓库拉取代码。拉取完成后，`Maven` 会下载依赖包，可能会花费一些时间，耐心等待下。

本文基于 `master-1.2.x` 分支。

# 3. 初始化数据库

官方提供了两个 Demo 项目例子：

* tcc-transaction-dubbo-sample
* tcc-transaction-http-sample

考虑到不是所有所有同学都使用过 Dubbo 服务化框架，我们以 tcc-transaction-http-sample 项为例子。

打开 tcc-transaction-http-sample/src/main/dbscripts 目录，有四个 SQL 脚本文件：

* `create_db_cap.sql` ：tcc-transaction-http-capital  项目数据库初始化脚本。
* `create_db_ord.sql` ：tcc-transaction-http-order 项目数据库初始化脚本。
* `create_db_red.sql` ：tcc-transaction-http-redpacket 项目数据库初始化脚本。
* `create_db_tcc.sql` ：tcc-transaction **底层**数据库初始化脚本。

笔者使用 Navicat 进行数据库脚本执行。使用方式为：Navicat 菜单 Connection -> Execute SQL File，选择脚本文件，逐个执行。

目前数据库脚本未使用 `USE` 语句选择对应数据库，每个脚本都需要进行添加。以 `create_db_cap.sql` 举例子：

```SQL
CREATE DATABASE `tcc_cap` /*!40100 DEFAULT CHARACTER SET utf8 */;
-- 新增 USE
USE `tcc_cap`;
```

# 4. 启动 capital 项目

1. 修改项目下 `jdbc.properties` 文件，**填写成你的数据库地址**。
2. 使用 IDEA 配置 Tomcat 进行启动。这里要注意下：

    ```XML
    // appcontext-service-provider.xml
    <bean id="httpServer"
         class="org.springframework.remoting.support.SimpleHttpServerFactoryBean">
       <property name="contexts">
           <util:map>
               <entry key="/remoting/CapitalTradeOrderService" value-ref="capitalTradeOrderServiceExporter"/>
               <entry key="/remoting/CapitalAccountService" value-ref="capitalAccountServiceExporter"/>
           </util:map>
       </property>
       <property name="port" value="8081"/>
    </bean>
    ```
    * 默认开启 8081 端口提供接口服务。所以配置 Tomcat 的端口不能再使用 8081，避免冲突。例如，笔者使用 18081。

3. 访问 `http://127.0.0.1:18081/`，看到 "hello tcc transacton http sample capital"，代表项目启动完成。**`18081` 为你填写的 Tomcat 端口**。

# 5. 启动 redpacket 项目

同 tcc-transaction-http-capital 项目。

# 6. 启动 order 项目

1. 修改项目下 `jdbc.properties` 文件，**填写成你的数据库地址**。
2. 使用 IDEA 配置 Tomcat 进行启动。
3. 访问 `http://127.0.0.1:8080/`，看到 "sample 说明..."，代表项目启动完成。**`8080` 为你填写的 Tomcat 端口**。
4. 点击 [商品列表链接] -> [购买] -> [支付]，如果看到 "支付成功" 或者 "支付失败"，恭喜你🎉，你已经成功搭建完你的调试环境。愉快的开始玩耍把。


# 666. 彩蛋

调试环境搭建是阅读源码的第一步，如果你碰到无法搭建成功的情况，请给笔者公众号( **芋道源码** )留言。笔者会给你 1:1 的高级( **搞基** )支持。

另外这是一个系列文，本系列更新 TCC-Transaction ，下一个系列更新 ByteTCC 。嗨皮不？！

![](http://www.iocoder.cn/images/TCC-Transaction/2018_02_01/01.png)

道友，赶紧上车，分享一波朋友圈！

