---
title: "JDBC4.2规范-第五章 类与接口"
date: 2017-01-02T13:55:10+08:00
categories : ["jdbc"]
---


以下的类和接口，组成了 JDBC API

# 5.1 `java.sql`包

JDBC API 的核心部分都藏在了 `java.sql` 这个包里，所有的枚举类，普通类以及接口，都在下方列了出来，其中，枚举和普通类是粗体，接口是正常字体。

java.sql.Array

**java.sql.BatchUpdateException**

java.sql.Blob

java.sql.CallableStatement

java.sql.Clob

java.sql.ClientinfoStatus

java.sql.Connection

**java.sql.DataTruncation**

java.sql.DatabaseMetaData

**java.sql.Date**

java.sql.Driver

java.sql.DriverAction

**java.sql.DriverManager**

**java.sql.DriverPropertyInfo**

**java.sql.JDBCType**

java.sql.NClob

java.sql.ParameterMetaData

java.sql.PreparedStatement

**java.sql.PseudoColumnUsage**

java.sql.Ref

java.sql.ResultSet

java.sql.ResultSetMetaData

java.sql.RowId

**java.sql.RowIdLifeTime**

java.sql.Savepoint

**java.sql.SQLClientInfoException**

java.sql.SQLData

**java.sql.SQLDataException**

**java.sql.SQLException**

**java.sql.SQLFeatureNotSupportedException**

java.sql.SQLInput

**java.sql.SQLIntegrityConstraintViolationException**

**java.sql.SQLInvalidAuthorizationSpecException**

**java.sql.SQLNonTransientConnectionException**

**java.sql.SQLNonTransientException**

java.sql.SQLOutput

java.sql.SQLPermission

java.sql.SQLSyntaxErrorException

java.sql.SQLTimeoutException

java.sql.SQLTransactionRollbackException

java.sql.SQLTransientConnectionException

java.sql.SQLTransientException

java.sql.SQLType

java.sql.SQLXML

**java.sql.SQLWarning**

java.sql.Statement

java.sql.Struct

**java.sql.Time**

**java.sql.Timestamp**

**java.sql.Types**

java.sql.Wrapper

下面这些类和接口是在 JDBC 4.2 API 中新增加或有过改动的，其中新增加的类和接口以粗体的形式表示

java.sql.BatchUpdateException

java.sql.CallableStatement

java.sql.Connection

java.sql.DatabaseMetaData

java.sql.Date

java.sql.Driver

**java.sql.DriverAction**

java.sql.DriverManager

**java.sql.JDBCType**

java.sql.Permission

java.sql.PreparedStatement

java.sql.ResultSet

java.sql.SQLInput

java.sql.SQLOutput

**java.sql.SQLType**

java.sql.SQLXML

java.sql.Statement

java.sql.Types

java.sql.Timestamp

javax.sql.XADataSource

下面这个图展示了 `java.sql` 包里关键的类和接口之间关系

![](/classes-interfaces.jpg)

# 5.2 `javax.sql` 包

下面列出来的是 `javax.sql` 这个包中的类和接口，类用粗体表示，接口是普通字体

javax.sql.CommonDataSource

**javax.sql.ConnectionEvent**

javax.sql.ConnectionEventListener

javax.sql.ConnectionPoolDataSource

javax.sql.DataSource

javax.sql.PooledConnection

javax.sql.RowSet

**javax.sql.RowSetEvent**

javax.sql.RowSetInternal

javax.sql.RowSetListener

javax.sql.RowSetMetaData

javax.sql.RowSetReader

javax.sql.RowSetWriter

**javax.sql.StatementEvent**

javax.sql.StatementEventListener

javax.sql.XAConnection

javax.sql.XADataSource

> 注意 — `javax.sql` 这个包中的类和接口在 JDBC 2.0 API 中初次使用，在 J2SE 1.2 中，并没有包含这个包，这个包是作为 J2SE 1.2 平台的一个可选包。但在 J2SE 1.4 后，`javax.sql` 和 `java.sql` 一样，也成为了 Java 平台的一部分。

以下的图展示了 `javax.sql.DataSource` 与 `java.sql.Connection` 的关系

![DataSource与Connection](/assets/jdbc4.2-spec-5/ds_conn.jpg)

下图展示了与连接池有关的组成部分

![连接池](/assets/jdbc4.2-spec-5/pool.jpg)

下图展示了分布式事务有关的组成部分

![分布式事务](/assets/jdbc4.2-spec-5/dt.jpg)

下图展示与 RowSet 有关的组成部分

![rowset](/assets/jdbc4.2-spec-5/rowset.jpg)