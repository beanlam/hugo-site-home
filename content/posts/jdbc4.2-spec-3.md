---
title: "JDBC4.2规范-第三章 新特性"
date: 2017-01-02T13:42:51+08:00
categories : ["jdbc"]
---

JDBC API 4.2 规范在以下几个方面有所改动

# 3.1 增加对 `REF CURSOR` 的支持

有些数据库支持 `REF CURSOR` 数据类型，在调用存储过程后返回该类型的结果集。

# 3.2 支持大数量的更新

JDBC 当前的方法里返回一个更新数量时，返回的是一个 `int`，在某些场景下这会导致问题，因为数据集还在不停地增长。

# 3.3 增加 `java.sql.DriverAction` 接口

如果一个 driver 想要在它被 `DriverManager` 注销时得到通知，就要实现这个接口。

# 3.4 增加 `java.sql.SQLType` 接口

用来创建一个代表 SQL 类型的对象

# 3.5 增加 `java.sql.JDBCType` 枚举类
用来识别通用的 SQL 类型，目的是为了取代定义在 `Types.java` 类里的常量。

# 3.6 增加 Java Object 类型与 JDBC 类型的映射（附录表B-4）

增加 `java.time.LocalDate` 映射到 `JDBC DATE`

增加 `java.time.LocalTime` 映射到 `JDBC TIME`

增加 `java.time.LocalDateTime` 映射到 `JDBC TIMESTAMP`

增加 `java.time.LocalOffsetTime` 映射到 `JDBC TIME_WITH_TIMEZONE`

增加 `java.time.LocalOffsetDateTime` 映射到 `JDBC TIMESTAMP_WITH_TIMEZONE`
# 3.7 增加调用 `setObject` 和 `setNull` 方法时 Java 类型和 JDBC 类型的转换（附录表B-5）

允许 `java.time.LocalDate` 转化为 `CHAR, VARCHAR, LONGVARCHAR, DATE`

允许 `java.time.LocalTime` 转化为 `CHAR, VARCHAR, LONGVARCHAR, TIME`

允许 `java.time.LocalTime` 转化为 `CHAR, VARCHAR, LONGVARCHAR, TIMESTAMP`

允许 `java.time.OffsetTime` 转化为 `CHAR, VARCHAR, LONGVARCHAR, TIME_WITH_TIMESTAMP`

允许 `java.time.OffsetDateTime` 转化为 `CHAR, VARCHAR, LONGVARCHAR, TIME_WITH_TIMESTAMP, TIMESTAMP_WITH_TIMESTAMP`

# 3.8 使用 `ResultSet` getter 方法来获得 JDBC 类型（附录表B-6）

允许 `getObject` 方法返回 `TIME_WITH_TIMEZONE, TIMESTAMP_WITH_TIMEZONE`

# 3.9 JDBC API 的变化

以下的 JDBC API 有了一些变化

## 3.9.1 BatchUpdateException

增加了一个新的构造函数来支持大量的 update，增加 `getLargeUpdateCounts` 方法。

## 3.9.2 Connection

增加了 `abort,getNetworkTimeout, getSchema, setNetworkTimeout, setSchema` 方法。
调整了 `getMapType, setSchema, setMapType` 方法。

## 3.9.3 CallableStatement

重载了 `registerOutParameter, setObject `方法。
调整了 `getObject` 方法

## 3.9.4 Date

增加了 `toInstant, toLocalDate` 方法。
重载了 `valueOf` 方法。

## 3.9.5 DatabaseMetaData

增加了 `supportsRefCursor, getMaxLogicalLobSize` 方法。
调整了 `getIndexInfo` 方法。

## 3.9.6 Driver

调整了 `acceptsURL, connect` 方法。

## 3.9.7 DriverManager

重载了 `registerDriver` 方法。
调整了 `getConnection, deregisterDriver, registerDriver` 方法。

## 3.9.8 PreparedStatement

增加了 `executeLargeUpdate` 方法。
重载了 `setObject` 方法。

## 3.9.9 ResultSet

重载了 `updateObject` 方法。
调整了 `getObject` 方法。

## 3.9.10 Statement

增加了 `executeLargeBatch, executeLargeUpdate,getLargeUpdateCount, getLargeMaxRows, setLargeMaxRows`方法。
调整了 `setEscapeProcessing` 方法。

## 3.9.11 SQLInput

增加了 `readObject` 方法

## 3.9.12 SQLOutput

增加了 `readObject` 方法

## 3.9.13 Time

增加了 `toInstant, toLocalTime` 方法
重载了 `valueOf` 方法

## 3.9.14 Timestamp

增加了 `from, toInstant, toLocalTime` 方法
重载了 `valueOf` 方法

## 3.9.15 Types

增加了 `REF_CURSOR, TIME_WITH_TIMEZONE, TIMESTAMP_WITH_TIEMZONE` 类型

## 3.9.16 SQLXML

调整了 `getSource setResult` 方法

## 3.9.17 DataSource 与 XADataSource

调整了必须提供一个无参构造函数