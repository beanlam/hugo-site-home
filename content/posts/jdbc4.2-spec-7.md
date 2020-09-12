---
title: "JDBC4.2规范-第七章 数据库元数据"
date: 2017-01-02T21:45:57+08:00
categories : ["jdbc"]
---


JDBC 驱动需要实现 `DatabaseMetaData` 这个接口，以便向驱动的使用者提供一些关于底层数据源的信息。这个接口主要被应用服务器以及一些工具型代码使用，以决定如何与一个数据源交互，应用程序有时候也会使用这个接口里的方法去获得数据源的信息，但这种用法并不是典型的用法。

`DatabaseMetaData` 这个接口拥有超过 150 个方法，这么多的方法，可以根据它们提供的信息的类型进行分类，信息类型有：
- 关于数据源通用的信息
- 关于数据源是否支持某个特性或者是否具有某种能力的信息
- 数据源的限制
- 数据源有哪些 SQL 对象，以及这些 SQL 对象都拥有哪些属性
- 数据源是否支持事务

`DatabaseMetaData` 接口也拥有超过 40 个字段，当调用接口的方法时，这些字段常量会作为返回值。

本章会对 `DatabaseMetaData` 做一个概览，给出一些实例来验证接口定义的方法，并介绍一些新的方法。如果希望更深入地理解所有的方法，建议读者参考 JDBC API Specification。

> 注意，JDBC 也定义了 `ResultSetMetaData` 接口，这个接口我们将会在第十五章遇见它。

# 7.1 创建一个 DatabaseMetaData 对象
通过 `Connection` 接口的 `getMetaData` 方法来创建一个 `DatabaseMetaData ` 对象。一旦这个对象创建完成，就可以使用这个对象动态地查询与底层数据源有关的信息。以下代码示例创建了一个 `DatabaseMetaData` 对象，并使用这个对象来获取底层数据源支持的表名最大字符数是多少

```java
// 在这里 con 是一个 Connection 对象
DatabaseMetaData dbmd = con.getMetadata();
int maxLen = dbmd.getMaxTableNameLength();
```

# 7.2 获取通用的信息
有一些 `DatabaseMetaData` 接口的方法用来动态地获取底层数据源的一些通用的信息和实现细节。例如以下这些方法：
- getURL
- getUserName
- getDatabaseProductVersion, getDriverMajorVersion 和
getDriverMinorVersion
- getSchemaTerm, getCatalogTerm 和 getProcedureTerm
- nullsAreSortedHigh 和 nullsAreSortedLow
- usesLocalFiles 和 usesLocalFilePerTable
- getSQLKeyword

# 7.3 确认数据源是否支持某些特性
`DatabaseMetaData` 有一大堆方法可以用来确定驱动或者底层数据源是否支持某个特性或特性集合。不仅如此，有一些方法还描述了对该特性支持到了哪个层次。
能用来确认是否支持某个特性的方法有：
- supportsAlterTableWithDropColumn
- supportsBatchUpdates
- supportsTableCorrelationNames
- supportsPositionedDelete
- supportsFullOuterJoins
- supportsStoredProcedures
- supportsMixedCaseQuotedIdentifiers

能用来确认对某个特性的支持层次的方法有：
- supportsANSI92EntryLevelSQL
- supportsCoreSQLGrammar

# 7.4 数据源限制
`DatabaseMetaData` 有另外一组方法可以用来获取指定数据源在某些方面的限制，其中一些方法如下所示：
- getMaxRowSize
- getMaxStatementLength
- getMaxTablesInSelect
- getMaxConnections
- getMaxCharLiteralLength
- getMaxColumnsInTable
这一类方法的返回值都是一个 int 类型的数字，如果返回为0则代表该项资源**没有限制或者是不确定的**。
  
# 7.5 SQL 对象以及属性
`DatabaseMetaData` 类有一些方法，可以提供给我们那些组成了一个具体数据源的 SQL 对象的信息，也相应地提供了可以获取这些 SQL 对象的属性的方法，这些方法都返回一个 ResultSet 作为结果，其中的一行代表一个 SQL 对象，例如，方法 `getUDTs()` 会返回一个 ResultSet 对象作为结果，其中的每一行都代表一个*用户自定义类型*。常见的方法如以下：
- getSchemas
- getCatalogs
- getTables
- getPrimaryKeys
- getProcedures
- getProcedureColumns
- getUDTs
- getFunctions
- getFunctionColumns

这一类的方法返回的 ResultSet 对象，是 **TYPE_FORWARD_ONLY** 也是 **CONCUR_READ_ONLY** 的，这代表这个 ResultSet 的指针只能向后滚动，不能向前滚动，并且对结果集的更新不会同步到数据库里。`ResultSet.getHoldability` 方法可以用来获取 ResultSet 的可保存性的默认值，这个默认值不同的驱动实现都有可能不同。

如果 ResultSet 返回的结果有不同的厂商自定义的一些额外的字段，那么在实现驱动的时候，取出这个字段的值时必须使用字段的名称作为参数来取，因为 JDBC 规范可能以后也会对 DatabaseMetaData 这个类的方法返回的结果集增加字段值，使用字段名来取值可以使得 JDBC 规范的更新或改动不会对旧版的驱动实现造成影响。

# 7.6 事务支持
`DatabaseMetaData` 提供了少量的方法，用来查看底层的数据源是否支持某些事务特性，例如：
- supportsMultipleTransactions
- getDefaultTransactionIsolation

# 7.7 新增的方法
jdbc 4.2 规范为 `DatabaseMetaData` 引进了一些新的方法：
- supportsRefCursors 
- getMaxLogicalLobSize
这两个方法的描述信息在 JDBC API 的 JavaDoc 里有更详细的说明

# 7.8 修改的方法
- getIndexInfo 返回的 CARDINALITY 和 PAGES 字段的类型现在变成了 long