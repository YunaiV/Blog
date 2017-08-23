title: Sharding-JDBC 源码分析 —— SQL 解析（六）之删除SQL
date: 2017-08-02
tags:
categories: Sharding-JDBC
permalink: Sharding-JDBC/sql-parse-6
keywords: Sharding-JDBC,ShardingJDBC,Sharding-JDBC 源码,SQL解析, SQL 解析

-------

![](https://www.yunai.me/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

**本文主要基于 Sharding-JDBC 1.5.0 正式版**  

- [1. 概述](#)
- [2. DeleteStatement](#)
- [3. #parse()](#)
	- [3.1 #skipBetweenDeleteAndTable()](#)
	- [3.2 #parseSingleTable()](#)
	- [3.3 #parseWhere()](#)
- [666. 彩蛋](#)

-------

# 1. 概述

本文前置阅读：

* [《SQL 解析（一）之词法解析》](http://www.yunai.me/Sharding-JDBC/sql-parse-1/?self)
* [《SQL 解析（二）之SQL解析》](http://www.yunai.me/Sharding-JDBC/sql-parse-2/?self)

本文分享**删除SQL解析**的源码实现。

🙂 如果你已经理解[《SQL 解析（三）之查询SQL》](http://www.yunai.me/Sharding-JDBC/sql-parse-3/?self)，那本文会是一篇水文，当成一种放松吧。还是跟前文一样，以 MySQL 举例子。我们来一起看看 MySQLDeleteParser。

MySQL DELETE 语法一共有 2 种 ：

* 第一种：**Single-table syntax**

```SQL
DELETE [LOW_PRIORITY] [QUICK] [IGNORE] FROM tbl_name
    [PARTITION (partition_name,...)]
    [WHERE where_condition]
    [ORDER BY ...]
    [LIMIT row_count]
```

* 第二种：**Multiple-table syntax**

```SQL
DELETE [LOW_PRIORITY] [QUICK] [IGNORE]
    tbl_name[.*] [, tbl_name[.*]] ...
    FROM table_references
    [WHERE where_condition]
    
【OR】

DELETE [LOW_PRIORITY] [QUICK] [IGNORE]
    FROM tbl_name[.*] [, tbl_name[.*]] ...
    USING table_references
    [WHERE where_condition]    
```

Sharding-JDBC 目前仅支持第一种。业务场景上使用第二种的很少很少。

Sharding-JDBC 更新SQL解析主流程如下：

![](http://www.yunai.me/images/Sharding-JDBC/2017_08_02/01.png)

```Java
// AbstractDeleteParser.java
@Override
public DeleteStatement parse() {
   sqlParser.getLexer().nextToken(); // 跳过 DELETE
   skipBetweenDeleteAndTable(); // // 跳过关键字，例如：MYSQL 里的 LOW_PRIORITY、IGNORE 和 FROM
   sqlParser.parseSingleTable(deleteStatement); // 解析表
   sqlParser.skipUntil(DefaultKeyword.WHERE); // 跳到 WHERE
   sqlParser.parseWhere(deleteStatement); // 解析 WHERE
   return deleteStatement;
}
```

> **Sharding-JDBC 正在收集使用公司名单：[传送门](https://github.com/dangdangdotcom/sharding-jdbc/issues/234)。  
> 🙂 你的登记，会让更多人参与和使用 Sharding-JDBC。[传送门](https://github.com/dangdangdotcom/sharding-jdbc/issues/234)  
> Sharding-JDBC 也会因此，能够覆盖更多的业务场景。[传送门](https://github.com/dangdangdotcom/sharding-jdbc/issues/234)  
> 登记吧，骚年！[传送门](https://github.com/dangdangdotcom/sharding-jdbc/issues/234)**

# 2. DeleteStatement

删除SQL 解析结果。

```Java
public final class UpdateStatement extends AbstractSQLStatement {
}
```

😈 对，没有其他属性。

我们来看下 `DELETE IGNORE FROM t_user WHERE user_id = ?` 的**解析结果**：

![](http://www.yunai.me/images/Sharding-JDBC/2017_08_02/02.png)

# 3. #parse()

## 3.1 #skipBetweenDeleteAndTable()

在 `DELETE` 和 表名 之间有些词法，对 SQL 路由和改写无影响，进行跳过。

```Java
// MySQLDeleteParser.java
@Override
protected void skipBetweenDeleteAndTable() {
   getSqlParser().skipAll(MySQLKeyword.LOW_PRIORITY, MySQLKeyword.QUICK, MySQLKeyword.IGNORE);
   getSqlParser().skipIfEqual(DefaultKeyword.FROM);
}

// OracleDeleteParser.java
@Override
protected void skipBetweenDeleteAndTable() {
   getSqlParser().skipIfEqual(DefaultKeyword.FROM);
   getSqlParser().skipIfEqual(OracleKeyword.ONLY);
}
```

## 3.2 #parseSingleTable()

解析**表**，请看[《SQL 解析（二）之SQL解析》的 `#parseSingleTable()` 小节](http://www.yunai.me/Sharding-JDBC/sql-parse-2/?self)。

## 3.3 #parseWhere()

解析 WHERE 条件。解析代码：[《SQL 解析（二）之SQL解析》的#parseWhere()小节](http://www.yunai.me/Sharding-JDBC/sql-parse-2/?self)。

# 666. 彩蛋

道友，帮我分享一波怎么样？

**后面 SQL 路由和改写会更加有趣哟！**

