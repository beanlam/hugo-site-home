---
title: "JDBC4.2规范-第十章 事务"
date: 2017-01-02T21:55:52+08:00
categories : ["jdbc"]
---


事务用来提供数据集成性、正确的应用语义，以及并发访问时数据的一致性视图。所有符合 JDBC 规范的驱动都必须支持事务，JDBC 的事务管理 API 参照 SQL:2003 标准并且包含了以下的概念：
- 自动提交模式
- 事务隔离级别
- Savepoints

本章讨论单个连接上的事务，涉及多条连接的事务将会在第十二章《分布式事务》中讨论。

# 10.1 事务边界和自动提交
什么时候应该开启一个事务，是 JDBC 驱动或者底层的数据源做的一个隐式的决定，尽管有一些数据源支持 begin transaction 语句，但这个语句没有对应的 JDBC API。当一条 SQL 语句要求开启一个事务并且当前没有事务未执行完，那么新事务就会被开启。
Connection 有一个属性 autocommit 来表明什么时候应该结束事务。如果 autocommit 启用，那么每一条 SQL 语句**完全执行**后，都会自动执行事务的提交。以下几种情况，视为**完全执行**：
- 对于 DML 语句来说，例如 Insert，Update，Delete；以及 DDL 语句。这些语句在数据源端执行完毕就代表语句完全执行。
- 对于 Select 语句来说，完全执行意味着对应的结果集被关闭。
- 对于 CallableStatement 对象或者对于那些返回多个结果集的语句，完全执行意味着所有的结果集都关闭，以及所有的影响行数和出参都被获取到了。

## 10.1.1 关闭自动提交模式
以下代码示范了如何关闭自动提交模式：
```java
// Assume con is a Connection object
con.setAutoCommit(false);
```
当关闭自动提交，必须显式地调用 Connection 的 commit 方法提交事务或者调用 rollback 方法回滚事务。这种处理方式是合理的，因为事务的管理工作不是驱动应该做的，应用层应该自己管理事务，例如：
- 当应用需要将一组 SQL  组成一个事务的时候
- 当应用服务器管理事务的时候

autocommit 的默认值为 true，如果在一个事务的过程中，autocommit 的值被改变了，那么将会导致当前事务被提交。如果调用了 setAutocommit 方法，但没有改变原来的值，则不会产生其它附加影响，相当于没有调过一样。

如果一条连接参加了分布式事务，那 autocommit 不能设置为 true。第12章将会介绍到。

# 10.2 事务隔离级别
事务隔离级别定义了在一个事务中，哪些数据是对当前执行的语句“可见”的。在并发访问数据库时，事务隔离级别定义了多个事务之间对于同个目标数据源访问时的可交叉程度。可交叉程度可分为以下几类：
- dirty reads(脏读)
当一个事务能看见另外一个事务未提交的数据时，就称为脏读，换言之，一个事务修改数据后再未提交之前，就能被其它事务看见，如果这个事务被回滚了而不是提交了，那么其它事务看到的数据则是不正确的，是“脏”的。
- nonrepeatable reads(前后不一致读)
假设事务 A 读取了一行数据，接下来事务 B 改变了这行数据，之后事务 A 又再一次读取这行数据，这时候事务 A 就取到了两个不同的结果。
- phantom reads(幻读)
假设事务 A 通过一个 where 条件读取到了一个结果集，事务 B 这时插入了一条符合事务 A 的 where 条件的数据，之后事务 A 通过同样的 where 条件再次进行查询时，发现了多出来一条数据。

JDBC 规范增加了 TRANSACTION_NONE 隔离级别，来满足了 SQL:2003 定义的 4 种事务隔离级别。隔离级别从最宽松到最严格，排序如下所示：
- TRANSACTION_NONE
这意味着当前的 JDBC 驱动不支持事务，也意味着这个驱动不符合 JDBC 规范。
- TRANSACTION_READ_UNCOMMITTED
允许事务看到其它事务修改了但未提交的数据，这意味着有可能是脏读、前后不一致读或者幻读。
- TRANSACTION_READ_COMMITTED
一个事务在未提交之前，所做的修改不会被其它事务所看见。这能避免脏读，但避免不了前后不一致读和幻读。
- TRANSACTION_REPEATABLE_READ
避免了脏读和前后不一致读，但幻读依然是有可能发生的
- TRANSACTION_SERIALIZABLE
避免了脏读、前后不一致读以及幻读

## 10.2.1 使用 setTransactionIsolation 方法
一条连接的默认事务隔离级别是由驱动决定的，这个隔离级别也往往是底层的数据源默认的事务隔离级别。

应用程序可以使用 Connection 类里的 setTransactionIsolation 方法来改变一条连接的事务隔离级别。如果在一个事务的过程中调用 setTransactionIsolation 方法，会有什么样结果，完全由驱动的实现决定。

getTransactionIsolation 方法的返回值应当能正确地反映当前连接的事务隔离级别，建议实现驱动的时候要实现 setTransactionIsolation 方法，可以在一个事务开启之前去设置事务隔离级别。此外，调用 
setTransactionIsolation 这个方法时，自动提交当前事务，也是一种合理的 setTransactionIsolation 实现。

可能有些驱动实现并不支持所有的四种事务隔离级别，如果通过 setTransactionIsolation 方法设置的隔离级别驱动不支持的话，驱动可以主动将事务隔离级别设置为更高更严格的事务隔离级别，如果没法设置为更高或者更严格的，驱动应该抛出 SQLException。可以使用 DatabaseMetaData 的 supportsTransactionIsolationLevel 方法来判断驱动是否支持某个事务隔离级别。

## 10.2.2 性能考虑
事务隔离级别设置得越高，为了保证事务的正确语义，意味着会有更多的锁等待、锁竞争以及 DBMS 的附加损耗。这反过来也会降低并发访问性，所以应用程序可能会发现事务隔离级别越高时，性能反而会下降。为此，事务的管理者应该权衡两者的利弊，设置合理的事务隔离级别。

# 10.3 Savepoints 
 savepoints 可以在一个事务的中间设置一个标记点，来更灵活地控制事务。一旦事务设置了一个标记点，事务可以回滚到这个标记点，不会影响标记点之前的操作。可以使用 DatabaseMetaData.supportsSavepoints 方法来判断驱动或者数据库是否支持这个功能。

## 10.3.1 设置并回滚到标记点
Connection.setSavepoint 方法可以用来在当前事务中设置一个标记点，同时如果当前没有在事务中，调用这个方法能开启一个事务。 Connection.rollback 方法有一个重载版本，能够接收一个 savepoint 作为参数。
```java
conn.createStatement();
int rows = stmt.executeUpdate("INSERT INTO TAB1 (COL1) VALUES " +
"(’FIRST’)");
// set savepoint
Savepoint svpt1 = conn.setSavepoint("SAVEPOINT_1");
rows = stmt.executeUpdate("INSERT INTO TAB1 (COL1) " +
"VALUES (’SECOND’)");
...
conn.rollback(svpt1);
...
conn.commit();
```
上面的代码实例中，插入一行数据后，保存一个标记点，然后插入一行数据。当事务被回滚到标记点的时候，第二行数据不会被插入，第一行数据依然会被插入。当连接提交后，第一行数据将会保存在表里。

## 10.3.2 释放标记点
Connection.releaseSavepoint 方法接收一个 Savepoint 作为参数，删除这个标点以及在它之后的标记点。如果一个 savepoint 已经被释放了，还把它作为 rollback 的参数的话，将会导致 SQLException。当事务提交或者完全回滚的时候，所有的 savepoints 都会被自动释放。当回滚到某个 savepoint 后，这个 savepoint 以及在它之后定义的 savepoint 都会被自动释放掉。

