title: Sharding-JDBC 源码分析 —— SQL 解析（五）之更新SQL
date: 2017-07-31
tags:
categories: Sharding-JDBC
permalink: Sharding-JDBC/sql-parse-5
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
- [2. UpdateStatement](#)
- [3. #parse()](#)
	- [3.1 #skipBetweenUpdateAndTable()](#)
	- [3.2 #parseSingleTable()](#)
	- [3.3 #parseSetItems()](#)
	- [3.4 #parseWhere()](#)
- [666. 彩蛋](#)

-------

# 1. 概述

本文前置阅读：

* [《SQL 解析（一）之词法解析》](http://www.yunai.me/Sharding-JDBC/sql-parse-1/?self)
* [《SQL 解析（二）之SQL解析》](http://www.yunai.me/Sharding-JDBC/sql-parse-2/?self)

本文分享**更新SQL解析**的源码实现。

更新SQL解析比查询SQL解析复杂度低的多的多。不同数据库在插入SQL语法上也统一的多。**本文分享 MySQL 更新SQL解析器 MySQLUpdateParser**。

MySQL UPDATE 语法一共有 2 种 ：

* 第一种：**Single-table syntax**

```SQL
UPDATE [LOW_PRIORITY] [IGNORE] table_reference
    SET col_name1={expr1|DEFAULT} [, col_name2={expr2|DEFAULT}] ...
    [WHERE where_condition]
    [ORDER BY ...]
    [LIMIT row_count]
```

* 第二种：**Multiple-table syntax**

```SQL
UPDATE [LOW_PRIORITY] [IGNORE] table_references
    SET col_name1={expr1|DEFAULT} [, col_name2={expr2|DEFAULT}] ...
    [WHERE where_condition]
```

Sharding-JDBC 目前仅支持第一种。业务场景上使用第二种的很少很少。

Sharding-JDBC 更新SQL解析主流程如下：

![](http://www.yunai.me/images/Sharding-JDBC/2017_07_31/01.png)

```Java
// AbstractUpdateParser.java
@Override
public UpdateStatement parse() {
   sqlParser.getLexer().nextToken(); // 跳过 UPDATE
   skipBetweenUpdateAndTable(); // 跳过关键字，例如：MYSQL 里的 LOW_PRIORITY、IGNORE
   sqlParser.parseSingleTable(updateStatement); // 解析表
   parseSetItems(); // 解析 SET
   sqlParser.skipUntil(DefaultKeyword.WHERE);
   sqlParser.setParametersIndex(parametersIndex);
   sqlParser.parseWhere(updateStatement);
   return updateStatement; // 解析 WHERE
}
```

> **Sharding-JDBC 正在收集使用公司名单：[传送门](https://github.com/dangdangdotcom/sharding-jdbc/issues/234)。  
> 🙂 你的登记，会让更多人参与和使用 Sharding-JDBC。[传送门](https://github.com/dangdangdotcom/sharding-jdbc/issues/234)  
> Sharding-JDBC 也会因此，能够覆盖更多的业务场景。[传送门](https://github.com/dangdangdotcom/sharding-jdbc/issues/234)  
> 登记吧，骚年！[传送门](https://github.com/dangdangdotcom/sharding-jdbc/issues/234)**

# 2. UpdateStatement

更新SQL 解析结果。

```Java
public final class UpdateStatement extends AbstractSQLStatement {
}
```

😈 对，没有其他属性。

我们来看下 `UPDATE t_user SET nickname = ?, age = ? WHERE user_id = ?` 的**解析结果**：

![](http://www.yunai.me/images/Sharding-JDBC/2017_07_31/02.png)

# 3. #parse()

## 3.1 #skipBetweenUpdateAndTable()

在 `UPDATE` 和 表名 之间有些词法，对 SQL 路由和改写无影响，进行跳过。

```Java
// MySQLUpdateParser.java
@Override
protected void skipBetweenUpdateAndTable() {
   getSqlParser().skipAll(MySQLKeyword.LOW_PRIORITY, MySQLKeyword.IGNORE);
}

// OracleUpdateParser.java
@Override
protected void skipBetweenUpdateAndTable() {
   getSqlParser().skipIfEqual(OracleKeyword.ONLY);
}
```

## 3.2 #parseSingleTable()

解析**表**，请看[《SQL 解析（二）之SQL解析》的 `#parseSingleTable()` 小节](http://www.yunai.me/Sharding-JDBC/sql-parse-2/?self)。

## 3.3 #parseSetItems()

解析`SET`后语句。 

```Java
// AbstractUpdateParser.java
/**
* 解析多个 SET 项
*/
private void parseSetItems() {
   sqlParser.accept(DefaultKeyword.SET);
   do {
       parseSetItem();
   } while (sqlParser.skipIfEqual(Symbol.COMMA)); // 以 "," 分隔
}
/**
* 解析单个 SET 项
*/
private void parseSetItem() {
   parseSetColumn();
   sqlParser.skipIfEqual(Symbol.EQ, Symbol.COLON_EQ);
   parseSetValue();
}
/**
* 解析单个 SET 项
*/
private void parseSetColumn() {
   if (sqlParser.equalAny(Symbol.LEFT_PAREN)) {
       sqlParser.skipParentheses();
       return;
   }
   int beginPosition = sqlParser.getLexer().getCurrentToken().getEndPosition();
   String literals = sqlParser.getLexer().getCurrentToken().getLiterals();
   sqlParser.getLexer().nextToken();
   if (sqlParser.skipIfEqual(Symbol.DOT)) { // 字段有别名
       // TableToken
       if (updateStatement.getTables().getSingleTableName().equalsIgnoreCase(SQLUtil.getExactlyValue(literals))) {
           updateStatement.getSqlTokens().add(new TableToken(beginPosition - literals.length(), literals));
       }
       sqlParser.getLexer().nextToken();
   }
}
/**
* 解析单个 SET 值
*/
private void parseSetValue() {
   sqlParser.parseExpression(updateStatement);
   parametersIndex = sqlParser.getParametersIndex();
}
```

## 3.4 #parseWhere()

解析 WHERE 条件。解析代码：[《SQL 解析（二）之SQL解析》的#parseWhere()小节](http://www.yunai.me/Sharding-JDBC/sql-parse-2/?self)。

# 666. 彩蛋

😝 比更新SQL解析是不是简单，更不用对比查询SQL解析。😳有一种在水更的感觉。嘿嘿，下一篇（[《删除SQL解析》](http://www.yunai.me/images/common/wechat_mp_2017_07_31.jpg)）会更加容易。

道友，帮我分享一波怎么样？

