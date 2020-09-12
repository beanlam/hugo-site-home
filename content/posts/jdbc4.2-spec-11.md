---
title: "JDBC4.2规范-第十一章 连接池"
date: 2017-01-02T21:58:47+08:00
categories : ["jdbc"]
---

在基本的 `DataSource` 实现中，客户端的 Connection 对象与物理数据库连接有着1:1的关系。当 Connection 被关闭以后，物理连接也会被关闭。因此，连接的频繁打开、初始化以及关闭，会在一个客户端会话中上演多次，带来了过重的性能消耗。
而连接池就能解决这个问题，连接池维护了一系列物理数据库连接的缓存，可以被多个客户端会话重复使用。连接池能够极大地提高性能和可扩展性，特别是在一个三层架构的环境中，大量的客户端可以共享一个数量比较小的物理数据库连接池。在图11-1中，JDBC 驱动提供了一个 ConnectionPoolDataSource 的实现，应用服务器可以用它来创建和管理连接池。

![连接池datasource](/assets/jdbc4.2-spec-11/connections.png)

连接池的管理策略跟具体的实现有关，也跟具体的应用服务器有关。应用服务器对客户端提供了一个 DataSource 接口的具体实现，使得连接池化对于客户端来说是透明的。最终，客户端使用 DataSource API 就能和之前使用 JNDI 一样，获得了更好的性能和可扩展性。

下文将会介绍 ConnectionPoolDataSource 接口、PooledConnection 接口以及 ConnectionEvent 类，这三个组成部分是一个相互合作的关系，下文将以一个经典线程池的实现的角度，逐步描述这几部分。这一章也会介绍基本的 DataSource 对象和池化的 DataSource 对象之间的区别，此外，还会讨论一个池化的连接如何能够维护一堆可重用的 PreparedStatement 对象。

尽管本章中的所有讨论都是假设在三层架构环境下的，但连接的池化在两层架构的环境下也同样有用。
在两层架构的环境中，JDBC 驱动既实现了 DataSource 接口，也实现 ConnectionPoolDataSource 接口，这种实现方式允许客户端打开或者关闭多个连接。
# 11.1 ConnectionPoolDataSource 和 PooledConnection
一般来说， 一个 JDBC 驱动会去实现 ConnectionPoolDataSource 接口，应用服务器可以使用这个接口来获得 PooledConnection 对象，以下代码展示了 getPooledConnection 方法的两种版本
```java
public interface ConnectionPoolDataSource {
    PooledConnection getPooledConnection() throws SQLException;
    PooledConnection getPooledConnection(String user, String password) throws SQLException;
}
```
一个 PooledConnection 对象代表一条与数据源之间的物理连接。JDBC 驱动对于 PooledConnection 的实现，则会封装所有与维护这条连接相关的细节。
应用服务器则会在它的 DataSource 接口的实现中，缓存和重用这些 PooledConnection。当客户端调用 DataSource.getConnection 方法时，应用服务器将会使用物理 PooledConnection 去获取一个逻辑 Connection 对象。以下代码是 PooledConnection 接口的一些方法定义：
```java
public interface PooledConnection {
    Connection getConnection() throws SQLException;
    void close() throws SQLException;
    void addConnectionEventListener(
    ConnectionEventListener listener);
    void addStatementEventListener(
    StatementEventListener listener);
    void removeConnectionEventListener(
    ConnectionEventListener listener);
    void removeStatementEventListener(
    StatementEventListener listener);
}
```
当客户端使用完连接以后，它使用 Connection.close 方法来关闭这条逻辑连接，这个动作只是关闭了逻辑连接，但并不会关闭物理连接。物理连接会被归还到池子里，以待重用。
在这里，连接的池化对于客户端来说完全是透明的，客户端能像使用非池化连接那样去使用池化连接。

> 需要注意的是，当对池化的连接调用 Connection.close() 方法时，之前通过 Connection.setClientInfo 设置的属性将会被清除掉。

# 11.2 连接事件
回忆先前说过的，当 Connection.close 方法被调用后，底层的物理连接 PooledConnection 就可以再次被重用。当一个 PooledConnection 可以被回收的时候，将会使用 JavaBean 风格的事件去通知连接池管理器（应用服务器）。
为了发生连接事件时能被通知到，连接池管理器必须实现 ConnectionEventListener 接口，然后 PooledConnection 会将其注册为连接事件的一个监听者。ConnectionEventListener 接口定义了两个方法，也体现出了可能发生的两种不同的事件：
- connectionClosed 事件 --- 当逻辑连接 Connection.close 被调用时产生此事件
- connectionErrorOccurred --- 当出现一些致命的错误，比如说数据库宕机导致连接丢失的时候，会触发这个事件

连接池管理器通过调用 PooledConnection.addConnectionEventListener 方法来将自己注册为一个 PooledConnection 的监听者。一般情况下，注册的动作都发生在将连接归还到池子里之前。
JDBC 驱动负责在对应的事件发生的时候，调用回调方法，这两个方法都需要一个 ConnectionEvent 对象作为参数，通过这个对象可以判断到底是哪个 PooledConnection 被关闭了或者发生了错误。
当客户端关闭了逻辑连接的时候，JDBC 驱动会通过调用监听者所实现的 connectionClosed 方法来通知监听者，此时，监听者（连接池管理器）可以将该连接归还到池子里以便重用。当致命性错误发生时，JDBC 驱动首先会调用监听者实现的 connectionErrorOccurred 方法，然后再抛出一个 SQLException 异常。这个时候，监听者就可以通过 PooledConnection.close 方法来将物理连接关闭。

# 11.3 三层架构环境中的连接池化
以下步骤列出了客户端使用连接池池化时，实际上发生的事情：
- 客户端调用 DataSource.getConnection 方法
- 应用服务器在它自己支持连接池的 DataSource 实现中，查找是否有可用的 PooledConnection 对象
- 如果没有可用的 PooledConnection 对象，应用服务器调用 ConnectionPoolDataSource.getPooledConnection 来创建一条物理连接，JDBC 驱动的具体实现会负责连接创建的具体细节，并把它交给应用服务器管理。
- 无论是新建的 PooledConnection 还是已经创建好的处于可用状态的，应用服务器会对这条连接进行一些标识，标记它处于正在使用的状态。
- 应用服务器调用 PooledConnection.getConnection 方法来获得一个逻辑上的 Connection 对象，这个对象底层实际上关联了一个物理的 PooledConnection 对象，客户端调用 DataSource.getConnection 方法，返回值拿到的是逻辑上的 Connection 对象。
- 应用服务器通过调用 PooledConnection.addConnectionEventListener 方法将它自己注册为一个 ConnectionEventListener，当 PooledConnection 处于可用状态时，应用服务器就会得到相应的事件通知。
- 由 DataSource.getConnection 方法返回的逻辑 Connection 对象，依然是使用 Connection API，直到 Connection.close 被调用之前，底层的 PooledConnection 都处于使用状态，不可被重用。

即使在没有应用服务器的两层架构环境中，连接依然可以做到池化。这种情况下，JDBC 驱动需要实现 DataSource 接口和 ConnectionPoolDataSource 接口。

# 11.4 DataSource 实现与连接池化
抛开对性能和扩展性的提升不说，客户端使用 DataSource 接口的时候，不需要去关心它底层的实现是否池化，客户端面向的是一套统一的，无差别的使用方式。
常规的 DataSource 实现，即不实现连接池化功能的实现，一般由 JDBC 驱动实现，通常有以下两个观点被认为是正确的：
- DataSource.getConnection 方法创建一个新的 Connection 对象来代表一条真正的物理连接，并且封装了所有维护和管理这条物理连接的细节。
- Connection.close 方法关闭底层的物理连接并释放相关的资源

在一个实现了池化的 DataSource 实现中，情况则有些不一样，以下几个观点被认为是正确的：
- 在 DataSource 的实现中，包含了一个提供了连接池化功能的模块，这个模块要怎么实现没有一个统一的标准，因人而异。这个模块会缓存一系列 PooledConnection 对象。DataSource 的实现类，通常处于驱动实现的 ConnectionPoolDataSource 和 PooledConnection 接口的上层。
- DataSource.getConnection 方法会调用 PooledConnection 方法去获得对底层物理连接的一个句柄，如果已有的连接池里没有现成可用的连接，那么这个时候就需要新建物理连接，只有在这种情况下，新建物理连接对性能的消耗才体现出来。当需要创建新的物理连接的时候，ConnectionPoolDataSource 的 getPooledConnection 会被调用，对于物理连接的管理细节，则委托给了 PooledConnection 对象。
- Connection.close 方法被调用时，只是关闭逻辑上的连接句柄，并不会关闭实际上的物理连接。连接池管理者此时会收到一个事件通知，被告知一个 PooledConnection 处于可重用状态了。如果此时客户端仍然企图使用这个逻辑上的连接句柄，那么只会得到一个 SQLException 异常。
- 一个物理 PooledConnection 在它的整个生命周期中，可能会产生许多的逻辑 Connection 对象，但只有最近一次产生的 Connection 对象才是有效的，当 PooledConnection.getConnection 方法被调用时，先前已经存在的 Connection 对象，将会被自动关闭。这种情况下，关闭不会产生相应的事件去通知监听者。

> 这给了应用服务器一种从客户端强行拿走连接的方式，这种情形可能很少见，但是当应用服务器需要进行强制关闭时，这个特性可能会很有用

- 连接池的管理者通过调用 PooledConnection.close 方法来关闭物理连接，一般发生以下情况时，才会这么做：当应用服务器正常退出时，当需要重新初始化连接的缓存时，或者是该连接上发生一些不可恢复的致命性错误时。

# 11.5 部署
进行连接池化的部署，需要提供一个客户端代码可以接触到的 DataSource 对象，并且还需要把一个 ConnectionPoolDataSource 对象注册到 JNDI 中。
第一步，部署 ConnectionPoolDataSource，如下代码所示：
```java
// ConnectionPoolDS implements the ConnectionPoolDataSource
// interface. Create an instance and set properties.
com.acme.jdbc.ConnectionPoolDS cpds = new com.acme.jdbc.ConnectionPoolDS();
cpds.setServerName(“bookserver”);
cpds.setDatabaseName(“booklist”);
cpds.setPortNumber(9040);
cpds.setDescription(“Connection pooling for bookserver”);
// Register the ConnectionPoolDS with JNDI, using the logical name
// “jdbc/pool/bookserver_pool”
Context ctx = new InitialContext();
ctx.bind(“jdbc/pool/bookserver_pool”, cpds);    
```
上述步骤做好以后，ConnectionPoolDataSource 对象就可以被对客户端代码可见的 DataSource 使用了，DataSource 的部署需要依赖于先前部署的 ConnectionPoolDataSource，如下代码所示：
```java
// PooledDataSource implements the DataSource interface.
// Create an instance and set properties.
com.acme.appserver.PooledDataSource ds =
new com.acme.appserver.PooledDataSource();
ds.setDescription(“Datasource with connection pooling”);
// Reference the previously registered ConnectionPoolDataSource
ds.setDataSourceName(“jdbc/pool/bookserver_pool”);
// Register the DataSource implementation with JNDI, using the logical
// name “jdbc/bookserver”.
Context ctx = new InitialContext();
ctx.bind(“jdbc/bookserver”, ds);
```
到此，客户端代码就可以使用这个 DataSource 了。

# 11.6 池化连接的 Statement 重用
JDBC 规范对于 statement 的池化也提供了一些支持。statement 池化这个特性，能让应用层像 connection 重用一样，对 PreparedStatement 进行重用，这个特性需要以连接池化为基础。
下图展示了 PooledConnection 与 PreparedStament 之间的关系。逻辑 Connection 可以透明地使用多个 PreparedStatement 对象。

![PooledConnection 与 PreparedStament 的关系](/assets/jdbc4.2-spec-11/statements.png)

上图中，连接池和 statement 池由应用服务器来实现。不过，这些功能其实也可以由驱动来实现，或者是数据源来实现。这里我们对于 statement 池化的讨论，其实是适用于以上提到的所有实现方式的。

## 11.6.1 使用池化 Statement
对于 statement 的重用，必须对应用透明。也就是说，从应用开发的角度，对一个 statement 的使用，不需要关心它是否是池化的实现。statement 在底层会一直保持处于打开状态，应用层的代码也不需要改变。如果应用层关闭了这个 statement，它依然需要调用 Connection.prepareStatement 方法来继续使用它。statement 的池化对于应用层来说，使用方式上是透明的，应用层唯一能感知到不同的，是它带来的明显的性能提升。
应用层需要通过调用 DatabaseMetadata 的 supportStatementPooling 方法，来判断一个数据源是否支持 statement 重用。
在很多情况下，对于 statement 的重用，是一种非常有意义的优化，尤其是负责的 prepared statement。不过，需要注意的是，大量的 statement 处于打开状态，有可能会对资源带来影响。

## 11.6.2 关闭池化 Statement
一旦应用层关闭了一个 statement，无论它是否是池化的，它都不能再继续被使用了，否则会导致异常抛出。
以下几个方法会关闭一个池化的 statement:

- Statement.close --- 由应用层调用。如果一个 statement 是池化的，调用这个方法会关闭逻辑上的 statement，但不会关闭底层的已经池化的物理 statement。
- Connection.close --- 由应用层调用。
    - 非池化连接 --- 关闭底层的物理连接和由这个连接创建的所有 statement。这样做是必要的，因为垃圾回收机制无法检测到外部的资源什么时候会被释放。
    - 池化连接 --- 仅关闭逻辑上的连接和这个连接所创建的逻辑 statement，底层的物理连接以及相关的 statement 不会被关闭。
- PooledConnection.close --- 由连接池管理者调用。会关闭底层的物理连接以及所有相关的 statement。

应用层无法直接关闭一个已经池化的物理 statement，这是连接池管理器做的事情。PooledConnection.close 方法关闭物理连接以及所有的关联 statement，释放掉相关的资源。
应用层也无法直接控制 statement 应该如何被池化。一个池化的 statement 总是与一个 PooledConnection 相关联的，ConnectionPoolDataSource 可以用来对池化做一些属性设置。
# 11.7 statement 事件
如果连接池管理器支持 statement 池化，它必须实现 StatementEventListener 接口，然后将自己注册为 PooledConnection 对象的监听者。这个接口定义了以下两个方法，用来监听有可能发生在一个 PreparedStatement 对象上的两种事件。
- statementClosed --- 当与 PooledConnection 对象相关联的逻辑 PreparedStatement 对象被关闭时触发，也就是说，当应用层调用 PreparedStatement.close 方法时。
- statementErrorOccurred --- 当 JDBC 驱动监测到 PreparedStatement 对象不可用时触发。

连接池管理器通过  PooledConnection.addStatementEventListener 方法将自己注册为监听者。一般来说，在连接池管理器返回一个 PreparedStatement 对象给应用层使用之前，它必须先把自己注册为一个监听者。
当对应的事件发生时，驱动会调用 StatementEventListener 的 statementClosed 方法和 statementErrorOccurred 方法，这两个方法都接收一个 statementEvent 对象作为参数，这个参数就可以用来判断是发生了关闭事件还是异常事件。当 JDBC 应用关闭逻辑 statement ，或者一些错误发生时，JDBC 驱动会调用相关的方法，这个时候，连接池管理器它就可以将这个 statement 放回池子以便重用，或者是抛出异常。

# 11.8 ConnectionPoolDataSource 属性
JDBC 的 API 定义了一系列的属性来设置与池化相关的属性：
|属性名|类型|描述|
|---|---|---|
|maxStatements|int|允许池化的最大 statement 数，0 代表不池化|
|initialPoolSize|int|当连接池创建时需要创建的初始物理连接数|
|minPoolSize|int|连接池最小物理连接数|
|maxPoolSize|int|连接池最大物理连接数，0代表无限制|
|maxIdleTime|int|连接空闲最大空闲时间，0代表无限制|
|propertyCycle|int|属性生效时间，单位为秒|

连接池的配置风格遵循 JavaBean 风格。连接池厂商如果需要增加配置属性，那这些新增的属性名不应与已有的标准属性名重复。
与 DataSource 的实现一样，ConnectionPoolDataSource 的实现也必须为每个属性增加 setter 和 getter 方法，以下代码是一个示例：
```java
VendorConnectionPoolDS vcp = new VendorConnectionPoolDS();
vcp.setMaxStatements(25);
vcp.setInitialPoolSize(10);
vcp.setMinPoolSize(1);
vcp.setMaxPoolSize(0);
vcp.setMaxIdleTime(0);
vcp.setPropertyCycle(300);
```
应用服务器会根据设置的属性，来决定应该如何管理相关的池子。
ConnectionPoolDataSource 的配置属性无须被 JDBC 客户端直接访问。一些管理工具需要访问的话，建议通过反射的方式。