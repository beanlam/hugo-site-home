---
title: "JDBC4.2规范-第八章 异常"
date: 2017-01-02T21:49:40+08:00
categories : ["jdbc"]
---


当访问一个数据源时发生错误或者警告，JDBC 用 `SQLException` 这个类及其子类来表示并提供相关的异常信息。

# 8.1 SQLException
SQLException 由一下几部分组成：
- 描述错误的文本信息。可以通过 `SQLException.getMessage()` 来获取。
- 一个 SQLState 对象。可以通过 `SQLException.getSQLStateType()` 来获取。
- 错误码，是某种错误类型的一个编码，int 类型，可以通过 `SQLException.getErrorCode()` 来获取。
- 底层的异常，是一个 `Throwable` 对象，用来代表引起 `SQLException` 发生的真正原因，通过 `SQLException.getCause()` 来获取。
- 异常链的引用，如果有不止一个 SQLException 异常，可以通过递归式地调用 `SQLException.getNextException` 来获取整个异常链，直到这个方法返回 null，异常链结束。

## 8.1.1 对 Java SE 异常链的支持
SQLException 和它的子类都支持 Java SE 的异常链机制，为了实现这个功能，有以下几个地方特意做了处理：
- 增加额外的四个构造函数
- 支持对异常链的 For-Each 语法
- getCause 方法不一定会返回 SQLException 类型或者其子类型

可以参考 JDBC API Java DOC 获取更详细的信息

## 8.1.2 遍历多个 SQLException
在一次 SQL 语句的执行中，很有可能会发生一个或者多个异常，多个异常之间有着相应的联系，这意味着，当捕获到一个 SQLException 时，它的背后可能还有多个 SQLException 链接在它身上，为了遍历这条异常链，通常可以通过循环调用 `SQLException.getNextException` 方法，直到该方法返回 Null。
同时，也可以通过循环调用 `SQLException.getCause()` 来获得引起一个 SQLException 的真正原因，直到该方法返回 Null。
以下代码示范了如何获取所有的 SQLException 以及其原因：
```java
catch(SQLException ex) {
  while(ex != null) {
    System.out.println("SQLState:" + ex.getSQLState());
    System.out.println("Error Code:" + ex.getErrorCode());
    System.out.println("Message:" + ex.getMessage());
    Throwable t = ex.getCause();
    while(t != null) {
      System.out.println("Cause:" + t);
      t = t.getCause();
    }
    ex = ex.getNextException();
  }
}
```

### 8.1.2.1 使用 For-Each 循环遍历 SQLException
以下代码示范了如何使用：
```java
catch(SQLException ex) {
    for(Throwable e : ex ) {
      System.out.println("Error encountered: " + e);
    }
}
```

# 8.2 SQLWarning
`SQLWarning` 是 `SQLException` 的子类，  下面这些接口的方法都有可能在对数据库进行操作的时候产生 SQLWarning
- Connection
- DataSet
- Statement
- ResultSet
当使用者调用其中一个方法后，如果产生了 SQLWarning，使用者并不会马上就被通知到，而必须使用者主动地去调用 `getWarnings` 方法才能获得。SQLWarning 也可以支持链式的机制，通过`SQLWarning.getNextWarning` 方法可以获取整个链条。

# 8.3 DataTruncation
`DataTruncation` 类是 `SQLWarning` 的一个子类，用来表示当数据被裁截时的警告。如果向数据库写入数据时发生了数据裁截，`DataTruncation` 会像异常一样被抛出来通知调用者，如果是在读取数据库数据的时候发生裁截，那么只会产生一个警告。
一个 `DataTruncation` 对象由以下部分组成：
- 描述文本
- 一个代码为 01004 的 SQLState（当读取数据时发生数据裁截）
- 一个代码为 22001 的 SQLState（当写入数据的时候发生数据裁截）
- 一个 Boolean 类型的标识，标识是字段值被裁剪还是参数被裁剪，DataTruncation.getParameter 方法如果返回 true 则是参数被裁截，否则是字段值被裁截
- 一个 int 值代表被裁截的字段或者参数的下标值
- 一个 Boolean 类型的标识，用来标识是读取还是写入的时候发生的数据裁截，通过 `DataTruncation.getRead ` 来判断
- `DataTruncation.getDataSize` 方法返回一个 int 值，用来代表被裁截前的数据大小，如果返回 -1 代表大小未知
- `DataTruncation.getTransferSize` 方法返回一个 int 值，代表裁截后的实际数据大小

## 8.3.1 静默裁截
`Statement.setMaxFieldSize` 方法允许设置一个最大的字段长度限制，这个限制只对这些数据类型有效：BINARY, VARBINARY, LONGVARBINARY, CHAR, VARCHAR, LONGVARCHAR, NCHAR, NVARCHAR 和 LONGNVARCHAR。如果设置了这个限制，并且从数据库读出超过这个限制的长度的字段值时，将不会抛出 DataTruncation

# 8.4 BatchUpdateException
当批量的 SQL 语句被执行时，有可能会抛出 `BatchUpdateException`，具体见第十四章。

# 8.5 可分类的 SQLExceptions
SQLExceptions 可分为以下三种类型：
- SQLNonTransientException
- SQLTransientException
- SQLRecoverableException

## 8.5.1 NonTransient SQLExceptions
一个 NonTransient SQLExceptions 必须继承自 SQLNonTransientException 类，如果一次数据库操作发生错误，在这个错误的原因未解决之前，继续调用相同的数据库操作，将会抛出一个 NonTransient SQLException，如果抛出的异常不是 SQLNonTransientConnectionException，那么还可以认为数据库连接依然是有效的，一下列表定义了具体的 SQLNonTransientSQLException 与 SQLState 的对应关系：

|SQL State|SQLNonTransientSQLException 子类|
|---|---|
|0A|SQLFeatureNotSupportedException|
|08|SQLNonTransientConnectionException|
|22|SQLDataException|
|23|SQLIntegrityConstraintViolationException|
|28|SQLInvalidAuthorizationExceptio|
|42|SQLSyntaxErrorException|

除了上述表格提到的，具体的数据库驱动实现也可以增加自己的 NonTransient Exception 类型

## 8.5.2 Transient Exceptions
Transient Exception 与 NonTransient Exception 的不同是，如果一次数据库操作发生了错误，那么在连接还有效的情况下，可以假设第二次相同的操作有可能成功。一个 Transient Exception 必须是 SQLTransientException 的子类。与 SQL State 的对应关系如下所示：

|SQL State|SQLTransientSQLException 子类|
|---|---|
|08|SQLTransientConnectionException|
|40|SQLTransactionRollbackException|
|N/A|SQLTimeoutException|

除了上述表格提到的，具体的数据库驱动实现也可以增加自己的 Transient Exception 类型

## 8.5.3 SQLRecoverableException
这个异常代表可恢复异常，所谓的可恢复是指，如果一次数据库操作失败了，那么应用层可以通过某些步骤来进行恢复设置，并且重新做同样的数据库操作后能成功。例如说，弃用旧连接，重新获取一条数据库连接。如果发生了 SQLRecoverableException，应用必须假设当前的连接已经是不可用的。

# 8.6 SQLClientinfoException
当调用 `Connection.setClientInfo` 方法来设置客户端信息时，可能会抛出这个异常。