---
title: "JDBC4.2规范-第六章 遵守规范"
date: 2017-01-02T21:41:10+08:00
categories : ["jdbc"]
---


本章指出了实现一个 JDBC 驱动所需要遵守的规范，在本章中没有指出的规范，则作为可选项来遵守。 

# 6.1 准则与要求

以下的准则是 JDBC API 规范要求实现者遵守的基本准则

- JDBC API 的实现者必须支持 Entry Level SQL92 标准，以及 ` Drop Table` 命令。对 Entry Level SQL92 标准的支持是实现 JDBC API 的最小要求，对于 SQL99 和 SQL2003 特性的实现，必须遵照 SQL99 和 SQL2003 的规范。

- JDBC 驱动必须支持转义语法，转义语法在 第十三章 中有详细解释。

- JDBC 驱动必须支持事务，参考 第十章。

- 如果 `DatabaseMetaData` 的某个方法指明某个特性的可用的，那么驱动必须根据这个特性的相关规范中规定的标准语法实现这个特性，如果该特性需要使用到数据源的原生 API 或者是 SQL 方言，那么由驱动负责实现从标准 SQL 语法到原生 API 或者 SQL 方言的映射关系。如果支持了某个特性，那么 `DatabaseMetaData` 中与这个特性相关的方法也必须提供实现。比如说，如果一个驱动实现了 `RowSet` 接口，那么它也应该实现 `RowSetMetaData` 接口。

- 驱动必须提供对底层数据源特性的访问方式，包括扩展了 JDBC API 的特性。这么规定的目的是能让使用了 JDBC API 的应用程度能像数据源的原生程序一样，访问与数据源有关的特性。

- 如果一个 JDBC 驱动不支持，或者部分支持某个可选的数据库特性，那么 `DatabaseMetaData` 的方法必须指明这个特性还没受到支持，任何还没实现或者还没支持的特性，如果应用程序使用到了，那么应该给应用程序抛一个 `SQLFeatureNotSupportedException`。

> 注意 —— 根据 SQL92 的规定， JDBC 驱动需要支持 `DROP TABLE` 命令，不过，是否实现 `CASCADE` 和 `RESTRICT`，则是可选的，不是必须的。此外， 当数据源里需要 `drop` 的表定义了视图、完整性约束时，如何实现 `DROP TABLE` 来处理这种情况，则每个驱动允许有不同的做法。

# 6.2 JDBC 4.2 API 要求驱动遵守的准则

- 上一节所述的所有准则

- 支持自动加载所有实现了 `java.sql.Driver` 的驱动类

- `ResultSet` 支持 `TYPE_FORWARD_ONLY` 类型

- `ResultSet` 支持 `CONCUR_READ_ONLY` 并发级别

- 支持批量更新

- 完全实现以下接口
    - java.sql.DatabaseMetaData
    - java.sql.ParameterMetaData
    - java.sql.ResultSetMetaData
    - java.sql.Wrapper
- 必须实现 `DataSource` 接口，但以下方法是可选的
    - getParentLogger
- 必须实现 `Driver` 接口，但以下方法是可选的
    - getParentLogger
- 必须实现 `Connection` 接口，但以下方法是可选的
    - createArrayOf
    - createBlob
    - createClob
    - createNClob
    - createSQLXML
    - createStruct
    - getNetworkTimeout
    - getTypeMap
    - setTypeMap
    - prepareStatement(String sql, Statement.RETURN_GENERATED_KEYS)
    - prepareStatement(String sql, int[] columnIndexes)
    - prepareStatement(String sql, String[] columnNames)
    - setSavePoint
    - rollback(java.sql.SavePoint savepoint)
    - releaseSavePoint
    - setNetworkTimeout
- 必须实现 `Statement` 接口，但以下方法是可选的
    - cancel
    - execute(String sql, Statement.RETURN_GENERATED_KEYS)
    - execute(String sql, int[] columnIndexes)
    - execute(String sql, String[] columnNames)
    - executeUpdate(String sql, Statement.RETURN_GENERATED_KEYS)
    - executeUpdate(String sql, int[] columnIndexes)
    - executeUpdate(String sql, String[] columnNames)
    - getGeneratedKeys
    - getMoreResults(Statement.KEEP_CURRENT_RESULT)，除非 DatabasemetaData.supportsMultipleOpenResults() 返回 true，否则是可选的。
    - getMoreResults(Statement.CLOSE_ALL_RESULTS) 除非 DatabasemetaData.supportsMultipleOpenResults() 返回 true，否则是可选的。
    - setCursorName
- 必须实现 `PreparedStatement` 接口，但以下方法是可选的
    - getMetaData
    - setArray, setBlob, setClob, setNClob, setNCharacterStream, setNString, setRef, setRowId, setSQLXML and setURL
    - setNull(int parameterIndex,int sqlType, String typeName)
    - setUnicodeStream
    - setAsciiStream, setBinaryStream, setCharacterStream,
    - setNCharacterStream
- 如果 `DatabaseMetaData.supportsStoredProcedures()` 返回 true, 那么必须实现 `CallableStatement` 接口，但以下方法是可选的
    - 所有的 setXXX, getXXX 方法，以及所有支持命名参数的 registerOutputParameter 方法
    - getArray, getBlob, getClob, getNClob, getNCharacterStream, getNString, getRef, getRowId, getSQLXML and getURL
    - getBigDecimal(int parameterIndex,int scale)
    - getObject(int i, Class<T> type)
    - getObject(String colName, Class<T> type)
    - getObject(int parameterIndex, java.util.Map<java.lang.String,java.lang.Class<?>> map)
    - registerOutputParam(String parameterName,int sqlType, String typeName)
    - setNull(String parameterName,int sqlType, String typeName)
    - setAsciiStream, setBinaryStream, setCharacterStream, setNCharacterStream
- 必须实现 `RowSet` 接口，但以下方法是可选的
    - 所有的 updateXXX 方法
    - absolute
    - afterLast
    - beforeFirst
    - cancelRowUpdates
    - deleteRow
    - first
    - getArray, getBlob, getClob, getNClob, getNCharacterStream, getNString, getRef, getRowId, getSQLXML and getURL
    - getBigDecimal(int i,int scale)
    - getBigDecimal(String colName,int scale)
    - getCursorName
    - getObject(int i, Class<T> type)
    - getObject(String colName, Class<T> type)
    - getObject(int i, Map<String,Class<?>> map)
    - getObject(String colName, Map<String,Class<?>> map)
    - getRow
    - getUnicodeStream
    - insertRow
    - isAfterLast
    - isBeforeFirst
    - isFirst
    - isLast
    - last
    - moveToCurrentRow
    - moveToInsertRow
    - previous
    - refreshRow
    - relative
    - rowDeleted
    - rowInserted
    - rowUpdated
    - updateRow
- 如果一个 JDBC 驱动支持 `ResultSet` 的 `CONCUR_UPDATABLE` 并发级别，那么必须实现以下方法
    - 除了 updateArray, updateBlob, updateClob, updateNClob, updateNCharacterstream, updateNString, updateRef, updateRowId, updateSQLXML, updateURL, updateBlob, updateClob, updateNClob, updateAsciiStream, updateBinaryStream, updateCharacterStream and updateNCharacterstream 之外的所有 updateXXX 方法。
    - cancelRowUpdates
    - deleteRow
    - rowDeleted
    - rowUpdated
    - updateRow
- 如果一个 JDBC 驱动支持 `TYPE_SCROLL_SENSITIVE` 和 `TYPE_SCROLL_INSENSITIVE` 类型的 `ResultSet`，那么必须实现以下方法
    - absolute
    - afterLast
    - beforeFirst
    - first
    - isAfterLast
    - isBeforeFirst
    - isFirst
    - isLast
    - last
    - previous
    - relative
- 如果一个可选实现的接口被实现了，那么这个接口的所有方法必须全部实现，除了以下的例外
    - java.sql.SQLInput 和 java.sql.SQLOutput 接口不要求实现 Array, Blob, Clob, NClob, NString, Ref, RowId, SQLXML and URL 这些数据类型。
    
# 6.3 Java EE 中的 JDBC 规范准则

在 Java EE 环境中使用的 JDBC 驱动，除了必须遵守前文中提到所有规定外，还必须遵守以下规定：

- 驱动必须支持存储过程，`DatabaseMetaData` 接口的 `supportsStoredProcedures` 方法必须返回 true，驱动也需要在调用 Statement, PreparedStatement, and CallableStatement 的方法时，支持转义语法，这些方法是：
    - executeUpdate
    - executeQuery
    - execute

对于存储过程的支持，仅仅需要驱动在调用 Statement, PreparedStatement, and CallableStatement 接口的 execute 方法时，要么返回一个更新数量，要么返回一个单一的 ResultSet 对象。这是因为有些数据库不支持调用存储过程后返回多个 ResultSet 对象。

同时也要支持所有的参数类型，包括 IN, OUT, INOUT

- 驱动必须支持下面这些函数的转义语法
    - ABS
    - CONCAT
    - LCASE
    - LENGTH
    - LOCATE (two argument version only)
    - LTRIM
    - MOD
    - RTRIM
    - SQRT
    - SUBSTRING
    - UCASE