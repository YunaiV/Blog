title: MyCAT 源码分析  —— PreparedStatement 重新入门
date: 2017-07-17
tags:
categories: MyCAT
permalink: MyCAT/what-is-PreparedStatement

---

![](http://www.yunai.me/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

- [1. 概述](#)
- [2. JDBC Client 实现](#)
- [3. MyCAT Server 实现](#)
	- [3.1 创建 PreparedStatement](#)
	- [3.2 执行 SQL](#)
- [4. 彩蛋](#)

-------

# 1. 概述

相信很多同学在学习 JDBC 时，都碰到 `PreparedStatement` 和 `Statement`。究竟该使用哪个呢？最终很可能是**懵里懵懂**的看了各种总结，使用 `PreparedStatement`。那么本文，通过 MyCAT 对 `PreparedStatement` 的实现对大家能够重新理解下。

本文主要分成两部分：

1. JDBC Client 如何实现 `PreparedStatement`。
2. MyCAT Server 如何处理 `PreparedStatement`。

😈 Let's Go。

# 2. JDBC Client 实现

首先，我们来看一段大家最喜欢复制粘贴之一的代码，JDBC PreparedStatement 查询 MySQL 数据库：

```Java
public class PreparedStatementDemo {

    public static void main(String[] args) throws ClassNotFoundException, SQLException {
        // 1. 获得数据库连接
        Class.forName("com.mysql.jdbc.Driver");
        Connection conn = DriverManager.getConnection("jdbc:mysql://127.0.0.1:8066/dbtest?useServerPrepStmts=true", "root", "123456");

        // PreparedStatement
        PreparedStatement ps = conn.prepareStatement("SELECT id, username, password FROM t_user WHERE id = ?");
        ps.setLong(1, Math.abs(new Random().nextLong()));

        // execute
        ps.executeQuery();
    }

}
```

获取 MySQL 连接时，`useServerPrepStmts=true`  是**非常非常非常重要**的参数。如果不配置，`PreparedStatement` 实际是个**假**的 `PreparedStatement`（新版本默认为 FALSE，据说部分老版本默认为 TRUE），未开启服务端级别的 SQL 预编译。

WHY ？来看下 JDBC 里面是怎么实现的。

``` Java
// com.mysql.jdbc.ConnectionImpl.java
public PreparedStatement prepareStatement(String sql, int resultSetType, int resultSetConcurrency) throws SQLException {
   synchronized (getConnectionMutex()) {
       checkClosed();
       
       PreparedStatement pStmt = null;
       boolean canServerPrepare = true;
       String nativeSql = getProcessEscapeCodesForPrepStmts() ? nativeSQL(sql) : sql;

       if (this.useServerPreparedStmts && getEmulateUnsupportedPstmts()) {
           canServerPrepare = canHandleAsServerPreparedStatement(nativeSql);
       }

       if (this.useServerPreparedStmts && canServerPrepare) {
           if (this.getCachePreparedStatements()) { // 从缓存中获取 pStmt
               synchronized (this.serverSideStatementCache) {
                   pStmt = (com.mysql.jdbc.ServerPreparedStatement) this.serverSideStatementCache
                           .remove(makePreparedStatementCacheKey(this.database, sql));

                   if (pStmt != null) {
                       ((com.mysql.jdbc.ServerPreparedStatement) pStmt).setClosed(false);
                       pStmt.clearParameters(); // 清理上次留下的参数
                   }

                   if (pStmt == null) {
                        // .... 省略代码 ：向 Server 提交 SQL 预编译。
                   }
               }
           } else {
               try {
                   // 向 Server 提交 SQL 预编译。
                   pStmt = ServerPreparedStatement.getInstance(getMultiHostSafeProxy(), nativeSql, this.database, resultSetType, resultSetConcurrency);

                   pStmt.setResultSetType(resultSetType);
                   pStmt.setResultSetConcurrency(resultSetConcurrency);
               } catch (SQLException sqlEx) {
                   // Punt, if necessary
                   if (getEmulateUnsupportedPstmts()) {
                       pStmt = (PreparedStatement) clientPrepareStatement(nativeSql, resultSetType, resultSetConcurrency, false);
                   } else {
                       throw sqlEx;
                   }
               }
           }
       } else {
           pStmt = (PreparedStatement) clientPrepareStatement(nativeSql, resultSetType, resultSetConcurrency, false);
       }

       return pStmt;
   }
}
```

* 【前者】当 Client 开启 `useServerPreparedStmts` 并且 Server 支持 `ServerPrepare`，**Client 会向 Server 提交 SQL 预编译请求**。

```Java
if (this.useServerPreparedStmts && canServerPrepare) {
    pStmt = ServerPreparedStatement.getInstance(getMultiHostSafeProxy(), nativeSql, this.database, resultSetType, resultSetConcurrency);
}
```

* 【后者】当 Client 未开启 `useServerPreparedStmts` 或者 Server 不支持 `ServerPrepare`，Client 创建 `PreparedStatement`，**_不会_向 Server 提交 SQL 预编译请求**。

```Java
pStmt = (PreparedStatement) clientPrepareStatement(nativeSql, resultSetType, resultSetConcurrency, false);
```

**即使这样，究竟为什么性能会更好呢？**

* 【前者】返回的 `PreparedStatement` 对象类是 `JDBC42ServerPreparedStatement.java`，后续每次执行 SQL 只需将对应占位符?对应的值提交给 Server即可，减少网络传输和 SQL 解析开销。  
* 【后者】返回的 `PreparedStatement` 对象类是 `JDBC42PreparedStatement.java`，后续每次执行 SQL 需要将**完整**的 SQL 提交给 Server，增加了网络传输和 SQL 解析开销。

*🌚：【前者】性能一定比【后者】好吗？相信你已经有了正确的答案。*

# 3. MyCAT Server 实现

## 3.1 创建 PreparedStatement

该操作对应 Client `conn.prepareStatement(....)`。

![](http://www.yunai.me/images/MyCAT/2017_07_17/01.png)

MyCAT 接收到请求后，创建 `PreparedStatement`，并返回 `statementId` 等信息。Client 发起 SQL 执行时，需要将 `statementId` 带给 MyCAT。核心代码如下：

```Java
// ServerPrepareHandler.java
@Override
public void prepare(String sql) {
LOGGER.debug("use server prepare, sql: " + sql);

   PreparedStatement pstmt = pstmtForSql.get(sql);
   if (pstmt == null) { // 缓存中获取
   	// 解析获取字段个数和参数个数
   	int columnCount = getColumnCount(sql);
   	int paramCount = getParamCount(sql);
       pstmt = new PreparedStatement(++pstmtId, sql, columnCount, paramCount);
       pstmtForSql.put(pstmt.getStatement(), pstmt);
       pstmtForId.put(pstmt.getId(), pstmt);
   }
   PreparedStmtResponse.response(pstmt, source);
}
// PreparedStmtResponse.java
public static void response(PreparedStatement pstmt, FrontendConnection c) {
   byte packetId = 0;

   // write preparedOk packet
   PreparedOkPacket preparedOk = new PreparedOkPacket();
   preparedOk.packetId = ++packetId;
   preparedOk.statementId = pstmt.getId();
   preparedOk.columnsNumber = pstmt.getColumnsNumber();
   preparedOk.parametersNumber = pstmt.getParametersNumber();
   ByteBuffer buffer = preparedOk.write(c.allocate(), c,true);

   // write parameter field packet
   int parametersNumber = preparedOk.parametersNumber;
   if (parametersNumber > 0) {
       for (int i = 0; i < parametersNumber; i++) {
           FieldPacket field = new FieldPacket();
           field.packetId = ++packetId;
           buffer = field.write(buffer, c,true);
       }
       EOFPacket eof = new EOFPacket();
       eof.packetId = ++packetId;
       buffer = eof.write(buffer, c,true);
   }

   // write column field packet
   int columnsNumber = preparedOk.columnsNumber;
   if (columnsNumber > 0) {
       for (int i = 0; i < columnsNumber; i++) {
           FieldPacket field = new FieldPacket();
           field.packetId = ++packetId;
           buffer = field.write(buffer, c,true);
       }
       EOFPacket eof = new EOFPacket();
       eof.packetId = ++packetId;
       buffer = eof.write(buffer, c,true);
   }

   // send buffer
   c.write(buffer);
}
```

**每个连接之间，PreparedStatement 不共享，即不同连接，即使 SQL相同，对应的 PreparedStatement 不同。**

## 3.2 执行 SQL

该操作对应 Client `conn.execute(....)`。

![](http://www.yunai.me/images/MyCAT/2017_07_17/02.png)

MyCAT 接收到请求后，将 PreparedStatement 使用请求的参数格式化成可执行的 SQL 进行执行。伪代码如下：

```Java
String sql = pstmt.sql.format(request.params);
execute(sql);
```

核心代码如下：

```Java
// ServerPrepareHandler.java
@Override
public void execute(byte[] data) {
   long pstmtId = ByteUtil.readUB4(data, 5);
   PreparedStatement pstmt = null;
   if ((pstmt = pstmtForId.get(pstmtId)) == null) {
       source.writeErrMessage(ErrorCode.ER_ERROR_WHEN_EXECUTING_COMMAND, "Unknown pstmtId when executing.");
   } else {
       // 参数读取
       ExecutePacket packet = new ExecutePacket(pstmt);
       try {
           packet.read(data, source.getCharset());
       } catch (UnsupportedEncodingException e) {
           source.writeErrMessage(ErrorCode.ER_ERROR_WHEN_EXECUTING_COMMAND, e.getMessage());
           return;
       }
       BindValue[] bindValues = packet.values;
       // 还原sql中的动态参数为实际参数值
       String sql = prepareStmtBindValue(pstmt, bindValues);
       // 执行sql
       source.getSession2().setPrepared(true);
       source.query(sql);
   }
}

private String prepareStmtBindValue(PreparedStatement pstmt, BindValue[] bindValues) {
   String sql = pstmt.getStatement();
   int[] paramTypes = pstmt.getParametersType();

   StringBuilder sb = new StringBuilder();
   int idx = 0;
   for (int i = 0, len = sql.length(); i < len; i++) {
       char c = sql.charAt(i);
       if (c != '?') {
           sb.append(c);
           continue;
       }
       // 处理占位符?
       int paramType = paramTypes[idx];
       BindValue bindValue = bindValues[idx];
       idx++;
       // 处理字段为空的情况
       if (bindValue.isNull) {
           sb.append("NULL");
           continue;
       }
       // 非空情况, 根据字段类型获取值
       switch (paramType & 0xff) {
           case Fields.FIELD_TYPE_TINY:
               sb.append(String.valueOf(bindValue.byteBinding));
               break;
           case Fields.FIELD_TYPE_SHORT:
               sb.append(String.valueOf(bindValue.shortBinding));
               break;
           case Fields.FIELD_TYPE_LONG:
               sb.append(String.valueOf(bindValue.intBinding));
               break;
           // .... 省略非核心代码
        }
   }

   return sb.toString();
}

```

# 4. 彩蛋

💯 看到此处是不是真爱？！反正我信了。  
给老铁们额外加个🍗。

细心的同学们可能已经注意到 JDBC Client 是支持缓存 `PreparedStatement`，无需每次都让 Server 进行创建。

当配置 MySQL 数据连接 `cachePrepStmts=true` 时开启 Client 级别的缓存。But，**此处的缓存又和一般的缓存不一样**，是使用 `remove` 的方式获得的，并且创建好 `PreparedStatement` 时也不添加到缓存。那什么时候添加缓存呢？在 `pstmt.close()` 时，并且**`pstmt` 是通过缓存获取时**，添加到缓存。核心代码如下：

```Java
// ServerPreparedStatement.java
public void close() throws SQLException {
   MySQLConnection locallyScopedConn = this.connection;

   if (locallyScopedConn == null) {
       return; // already closed
   }

   synchronized (locallyScopedConn.getConnectionMutex()) {
       if (this.isCached && isPoolable() && !this.isClosed) {
           clearParameters();
           this.isClosed = true;
           this.connection.recachePreparedStatement(this);
           return;
       }

       realClose(true, true);
   }
}
// ConnectionImpl.java
public void recachePreparedStatement(ServerPreparedStatement pstmt) throws SQLException {
   synchronized (getConnectionMutex()) {
       if (getCachePreparedStatements() && pstmt.isPoolable()) {
           synchronized (this.serverSideStatementCache) {
               this.serverSideStatementCache.put(makePreparedStatementCacheKey(pstmt.currentCatalog, pstmt.originalSql), pstmt);
           }
       }
   }
}
```

为什么要这么实现？`PreparedStatement` 是有状态的变量，我们会去 `setXXX(pos, value)`，一旦多线程共享，会导致错乱。

🗿 这个“彩蛋”还满意么？**请关注我的公众号：芋道源码**。下一篇更新：《MyCAT源码解析 —— MongoDB》，极大可能就在本周噢。

![wechat_mp](http://www.yunai.me/images/common/wechat_mp_2017_07_31.jpg)

另外推荐一篇文章：[《JDBC PreparedStatement》](https://www.zybuluo.com/stefanlu/note/254899)。


