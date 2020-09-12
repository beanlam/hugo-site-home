---
title: "JDBC4.2规范-第九章 连接"
date: 2017-01-02T21:52:29+08:00
categories : ["jdbc"]
---

一个 `Connection` 对象，表示了与某个数据源的一条连接，数据源的种类可以是关系型数据库，文件系统等等之类，只要有对应的 JDBC 驱动，都可以称之为数据源。应用程序使用 JDBC API 来维护多条连接，这些连接可能访问的是多个数据源，也可能访问的只是一个数据源。从 JDBC 驱动的角度来看，一个 `Connection` 对象就意味着一个客户端会话，一个会话会保持许多状态，例如用户 ID，一系列的 SQL Statement 以及结果集，也保存了当前使用的事务处理策略。

可以通过以下两种方式之一来获取一条连接：
- 使用 `DriverManager` 这个类以及各种各样的驱动实现
- 使用 `DataSource` 类

更推荐使用 `DataSource` 对象来获取连接，因为这增强了应用的可移植性，使得代码更容易维护了，并且使得对连接池和分布式事务的使用更加地透明。所有的 Java EE 组件，都会使用 DataSource 对象来获取连接。

这一章将会介绍各种不同的 JDBC 驱动以及如何使用 `Driver` 接口、`DriverManager` 类以及基本的 `DataSource` 接口。关于连接池和分布式事务的介绍分别在第11章和第12章做介绍。

# 9.1 驱动的种类
- Type 1
这种类型的 JDBC 驱动是对另外一种访问 API 的映射，比如说 ODBC，一般需要依赖本地库，这就导致了它的可移植性不行。JDBC-ODBC 桥就是这种类型的驱动。
- Type 2
这种类型的 JDBC 驱动一部分是用 Java 语言写的，一部分是用本地代码写的。这种驱动使用一个本地的客户端库来连接数据源。由于对本地代码的使用，可移植性也不行。
- Type 3 
这种类型的驱动使用纯 Java 语言编写，但是通信的时候需要经过一个中间件，使用的是与数据库具体协议无关的独立协议。这个中间件转发客户端的请求给后面的数据源。
- Type 4
这种类型的驱动使用纯 Java 语言编写，并且使用网络协议或者文件 IO 与具体的数据源通信，客户端直接与数据源连接。

# 9.2 Driver 接口
编写 JDBC 驱动，必须实现 Driver 接口，并且实现类中必须包含一个静态初始化块，当驱动被加载时，这块代码会被调用。这块代码的主要工作是讲自己注册给 DriverManager，如下代码所示：
```java
public class AcmeJdbcDriver implements java.sql.Driver {
  static {
    java.sql.DriverManager.registerDriver(new
    AcmeJdbcDriver());
  }
}
```
驱动的实现类必须提供一个无参构造函数，当 DriverManager 想要与 Driver 交互时，它会直接调用它的方法，Driver 接口包含了一个 acceptsURL 方法，DriverManager 可以调用这个方法来判断该驱动是否能处理对应的 JDBC URL。

当 DriverManager 想要建立一条数据库连接时，它会调用驱动实现类的 connect 方法，并把 URL 作为参数穿给它，这个方法会返回一个 Connection 对象，或者是当无法建立数据库连接时，抛出一个 SQLException。如果驱动实现类无法解析 URL，这个方法将会返回 null。

## 9.2.1 加载一个实现了 java.sql.Driver 接口的驱动类
DriverManager 初始化的时候，会先通过 “java.drivers” 这个系统属性来尝试加载驱动，如以下例子：
```bash
java -Djdbc.drivers=com.acme.jdbc.AcmeJdbcDriver Test
```
DriverManager 的 getConnection 方法能够支持 Java SE 的 SPI 服务发现机制，JDBC 4.0 的驱动必须包含以下文件 “META-INF/services/java.sql.Driver”，这个文件会包含实现了 Driver 接口的类名

# 9.3 DriverAction 接口
当 DriverManager 的 deregisterDriver 方法被调用时，如果想要被通知到，那么 JDBC 驱动就得有对应的实现了 DriverAction 接口的类，DriverAction 的具体实现类并不希望直接被上层应用拿来使用，所以实现 JDBC 驱动的时候，应该将这个类定义为私有的类，以防止被直接使用。

JDBC 驱动的静态初始化块里面，必须调用 DriverManager.registerDriver(java.sql.Driver, java.sql.DriverAction)  方法，这样当一个 JDBC 驱动被 DriverManager 注销的时候，才能被通知到，如下所示：
```java
public class AcmeJdbcDriver implements java.sql.Driver {
  static DriverAction da;
  static {
    java.sql.DriverManager.registerDriver(new
    AcmeJdbcDriver(), da);
  }
}
```
# 9.4 DriverManager 类
DriverManager 类与 Driver 接口一起协作，维护所有可用的 JDBC 驱动。当应用程序通过一个 URL 来获取一个连接的时候， DriverManager 负责找到一个适用该 URL 的驱动，用这个驱动来获取对应的数据源的连接。
DriverManager 的关键方法如下所示：
- registerDriver 
这个方法会将某个驱动加进可用驱动的集合里，它在一个驱动被装载的时候隐式地调用，一般情况下是驱动的静态代码块里调用这个方法。
- getConnection
这个方法用来获取一个连接，要调用这个方法，必须提供一个 JDBC URL，DriverManager 会使用这个 URL 来轮询所有已经注册的驱动，并找到一个可以识别这个 URL 的驱动，驱动会返回一个 Connection 给 DriverManager，然后再把它交给应用程序。

JDBC URL 的格式如下所示：
```java
jdbc:<subprotocol>:<subname>
```
subprotocol 定义是要连接的是哪种类型的数据库，subname 则会根据 subprotoco 的不同而不同。
以下代码示范了如何从 DriverManager 获取一个连接：
```java
String url = "jdbc:derby:sample";
String user = "SomeUser";
String passwd = "SomePwd";

Connection con = DriverManager.getConnection(url, user, passwd);
```

DriverManager 类也提供了另外一些获取连接的方法：
- getConnection(String url)
这个方法适用于不需要提供用户名和密码的情况
- getConnection(String url, java.util.Properties prop)
这个方法允许在 prop 参数里加入用户名和密码，以及其它属性

DriverPropertyInfo 这个类提供了一个驱动可以理解的所有的属性，详见 Java JDBC API DOC

# 9.5 SQLPermission 类
这个类代表了一个代码基所拥有的权限。当前唯一定义的权限是 setLog 权限。当一个 Applet 调用了 DriverManager 的 setLogWriter 或者 setLogStream 方法时，SecurityManager 将会检查是否有权限。如果没有权限，将会抛出一个 java.lang.SecurityException 异常

# 9.6 DataSource 接口
DataSource 这个接口是在 JDBC2.0 的可选属性里引进的，这是 JDBC 规范推荐的用来获取数据源连接的方式。实现了 DataSource 接口的 JDBC 驱动会返回和通过 DriverManager 获取的相同的 Connection 实例，使用 DataSource 接口使应用程序更加具有可移植性，因为应用程序不需要为某个特定的驱动提供相关的连接信息，仅仅需要提供一个逻辑的数据源名。逻辑数据源名用来映射到 JNDI 提供的 DataSource 实例。这个 DataSource 实例代表了一个物理上的数据源，并提供获取相应连接的方法。如果关于数据源的属性或者信息发生了变化，DataSource 对象可以感知到对应的变化，完全不需要改变应用代码。
实现 DataSource 接口时，应该透明地提供以下功能：
- 通过连接的池化来提高性能和可扩展性
- 通过 XADataSource 接口来支持分布式事务

还需要注意的是，DataSource 的实现类必须提供一个无参构造函数
接下来的3个小节主要讨论：
1. 基本的 DataSource 属性
2. 使用 JNDI API 如何提供应用的可移植性以及可维护性
3. 如果获取一个连接

## 9.6.1 DataSource 的属性
JDBC API 定义了一系列的属性来描述 DataSource 的实现，具体的属性有哪些，取决于具体的 DataSource 实现，也就是说，取决于该实现是一个基本的 DataSource 对象，还是 ConnectionPoolDataSource，或者是 XADataSource，无论什么实现，它们都会有共同的属性 description，以下是标准的 DataSource 属性：

|属性名|数据类型|描述|
|---|---|---|
|databaseName|String|数据库名|
|dataSourceName|String|数据源名，用来命名底层的 XADataSource 或者是 ConnectionPoolDataSource|
|description|String|对此 DataSource 的描述信息|
|networkProtocol|String|网络协议|
|password|String|数据库密码|
|portNumber|int|数据库监听端口|
|roleName|String|初始 SQL roleName|
|serverName|String|数据库服务器名|
|user|String|数据库用户名|

DataSource 的属性遵循 JavaBean 1.01 规范。具体 DataSource 实现可以添加属性，但是不能与原有的有冲突。这些属性必须提供对应的 setter 和 getter 方法，当一个新的 DataSource 初始化的时候，这些属性也应该相应进行初始化，如以下代码所示，这里的实现是一个 VendorDataSource：
```java
VendorDataSource vds = new VendorDataSource();
vds.setServerName("my_database_server");
String name = vds.getServerName();
```
DataSource 的属性，设计的初衷是不应该直接被应用代码获取，应该在具体的实现类里提供获取的方法，而不是在 DataSource 上定义 public 的属性，想要获取属性值，可以通过“自省”的方式（反射）来获取。

## 9.6.2 JNDI API 以及应用可移植性
Java Naming and Directory Interface (JNDI) API 提供让应用通过网络访问远程服务的统一方式，本小节将描述如何使用 JNDI 来注册并访问一个 JDBC 数据源对象。更详细的信息可以查阅 JNDI 规范。
使用 JNDI API，应用可以通过指定一个逻辑名来访问一个数据源，在这里 JNDI 需要使用到命名服务，来将逻辑名映射到对应的数据源。这个特性极大地增强了应用的可移植性，因为很多数据源的配置，可以在不修改应用层代码的情况下进行修改，例如端口号和服务器名。事实上，应用可以透明地访问另一个完全不同的数据源，只需要修改对应的配置。在三层架构的环境中，这个特性很重要，应用服务器会将访问不同数据源的细节隐藏起来，不需要对应用开放。

以下代码实例了如何使用 JNDI 来注册一个数据源对象：
```java
// Create a VendorDataSource object and set some properties
VendorDataSource vds = new VendorDataSource();
vds.setServerName("my_database_server");
vds.setDatabaseName("my_database");
vds.setDescription("data source for inventory and personnel");

// Use the JNDI API to register the new VendorDataSource object.
// Reference the root JNDI naming context and then bind the
// logical name "jdbc/AcmeDB" to the new VendorDataSource object.
Context ctx = new InitialContext();
ctx.bind("jdbc/AcmeDB", vds);
```

## 9.6.3 通过 DataSource 实例获取连接
一旦一个 DataSource 注册在 JNDI 的命名服务后，应用可以使用它来获取一条到物理数据源的连接，如下代码所示：
```java
// Get the initial JNDI naming context
Context ctx = new InitialContext();
// Get the DataSource object associated with the logical name
// "jdbc/AcmeDB" and use it to obtain a database connection
DataSource ds = (DataSource)ctx.lookup("jdbc/AcmeDB");
Connection con = ds.getConnection("user", "pwd");
```
## 9.6.4 关闭连接
Connection.close(), Connection.isclosed()  和 Connection.isValid() 这些方法可以用来关闭一条连接和判断一条连接是否还处于活跃状态。
### 9.6.4.1 Connection.close
An application calls the method当应用使用完一条连接后，可以调用 Connection.close() 来关闭这条连接，在这条连接上所有的 Statement 对象也会被关闭。
一条连接关闭后，除了 close(), isClosed() 和 isValid() 方法外，调用其它的方法将会抛出一个 SQLException。

### 9.6.4.2 Connection.isClosed
这个方法用来判断一条连接的 close() 方法是否已经被调用过，这个方法不能用来判断连接是否还可用。
但是有写 JDBC 驱动可能会增强 isClosed() 方法，使得可以利用这个方法来判断一条连接是否还可用。在这里，为了最大的可移植性，应用应该通过 Connection.isValid() 来判断一条连接是否还可用。

### 9.6.4.3 Connection.isValid
这个方法用来标识一条连接是否还可用，如果不可用，那么除了 close()，isClosed() 和isValid() 方法之外，调用其它方法将会抛出 SQLException

