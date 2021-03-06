---
title: "JDBC4.2规范-第四章 JDBC API 概览"
date: 2017-01-02T13:46:42+08:00
categories : ["jdbc"]
---

JDBC API 给 Java 程序提供了一种访问一个或者多个数据源的途径，在大多数情况下，数据源是关系型数据库，使用 SQL 语言来访问。但是，JDBC Driver 也可以实现为能够访问其它类型的数据源，比如说文件系统或面向对象的系统。 JDBC API 最主要的动机就是提供一种标准的 API ，让应用程序访问多种多样的数据源。

这一章介绍了 JDBC API 的一些关键概念，此外，也介绍 JDBC 程序的两种使用场景，分别是**两层模型**和**三层模型**，在不同的场景中，JDBC API 的功能是不一样的。


# 4.1 建立连接

JDBC API 定义了 `Connection` 接口来代表与某个数据源的一条连接。

典型情况下，JDBC 应用可以使用以下两种机制来与目标数据源建立连接

- **`DriverManager`** — 这个类从 JDBC API 1.0 版本开始就有了，当应用程序第一次尝试去连接一个数据源时，它需要指定一个**url**，`DriverManager` 将会自动加载所有它能在 CLASSPATH 下找到的 JDBC 驱动（任何 JDBC API 4.0 版本前的驱动，需要手动去加载）。

- **`DataSource`** — 这个接口在 JDBC 2.0 Optionnal Package API 中首次被引进，更推荐使用 `DataSource`， 因为它允许关于底层数据源的具体信息对于应用来说是透明的。需要设置 `DataSource` 对象的一些属性，这样才能让它代表某个数据源。当这个接口的 `getConnection` 方法被调用时，这个方法会返回一条与数据源建立好的连接。应用程序可以通过改变 `DataSource` 对象的属性，从而让它指向不同的数据源，无须改动应用代码；同时 `DataSource` 接口的具体实现类也可以在不改动应用程序代码的情况下，进行改变。

JDBC API 也对 `DataSource` 接口有两方面的扩展，目的是为了支持企业应用，这两个扩展的接口如下所示：

- **`ConnectionPoolDataSource`** — 支持对物理连接的缓存和重用，这能提高应用的性能和可扩展性

- **`XADataSource`** — 使连接能在分布式事务中使用

# 4.2 执行 SQL 并操作结果集

一旦建立好一个连接，应用程序便可以通过这条连接，调用响应的 API 来对底层的数据源执行查询或者更新操作， JDBC API 提供了对于 SQL2003 标准的实现的访问。由于不同的厂商对这个标准的支持程度不同，所以 JDBC API 提供了 `DatabaseMetadata` 这个接口，应用程序可以使用这个接口来查看某个特性是否受到底层数据库的支持。JDBC API 也定义了转义语法，允许应用程序去访问一些非标准的、某个数据库厂商独有的特性。使用转义语法能够让使用 JDBC API 的应用程序像原生应用程序一样去访问某些特性，并且也提高了应用的可移植性。

应用可以使用 `Connection` 接口中定义的方法，去指定事务的属性，并创建 `Statement` 对象、`PreparedStatement` 对象，或者 `CallableStatement` 对象。 这些 statement 用来执行 SQL 语句，并获取执行结果。`ResultSet` 接口包装一次 SQL 查询的结果。 statements 可以是批量的，应用能够在一次执行中，向数据库提交多条更新语句，作为一个执行单元。

JDBC API 的 `ResultSet` 接口扩展了 `RowSet` 接口，提供了一个功能更全面的对表格型数据进行封装和访问的容器。一个 `RowSet` 对象是一个 Java Bean 组件，在于底层数据源断开连接的情况下，也能对数据进行操作，比如说，一个 `RowSet` 对象可以被序列化，然后通过网络发送出去，这对于那些不想对表格型数据进行处理的客户端来说特别有用，并且无须在连接建立的情况下进行，就减轻了驱动程序的负担。`RowSet` 的另外一个特性是，它能够包含一个定制化的 reader，来对表格型数据进行访问，并非只能访问关系型数据库的数据。此外，一个 `RowSet` 对象，能在与数据源断开连接的情况下，对行数据进行改写，并且能够包含一个定制化的 writer，把改写后的数据写回底层的数据源。

## 4.2.1 对 SQL 高级数据类型的支持

JDBC API 定义了 SQL 数据类型到 JDBC 数据类型的相互转化规则，包括对 SQL2003 的高级数据类型的支持，比如说 `BLOB, CLOB, ARRAY, REF, STRUCT, XML, DISTINCT`。 JDBC 驱动的实现也可以定义个性化的转化规则（user-defined types, UDTS）, 该用户定义的UDT能够映射到 Java 语言中的某个类。JDBC API 也提供了对外部数据的访问，比如说存储在文件里，但不受数据源管理的数据。

# 4.3 两层模型

两层模型定义了客户端层和服务端层，不同层实现不同的功能，如下图所示:

![两层模型](/assets/jdbc4.2-spec-4/2layers.jpg)

客户端层包含应用程序以及一个或者多个 JDBC 驱动，这一层的主要职责是：

- 表现层逻辑
- 业务逻辑
- 对于多语句事务或者分布式事务的事务管理
- 资源管理

在这种模型中，应用程序直接与 JDBC 驱动交互，包括创建和管理物理连接，处理底层数据库的细节。应用程序可能会基于对底层数据源的类型的认知，去访问一些特有的、非标准的特性，以此来获得性能上的提升。

这个模型有一些缺点，如下所示：

- 将表现层和业务层逻辑与底层的功能直接混合，这会使代码变得难以维护。
- 应用程序不具有可移植性，因为应用程序会使用到底层特定数据库的一些独有的特性，对于需要与多种数据源进行连接的应用程序来说，要特别注意不同厂商的数据库实现以及不同的特性。
- 限制了可扩展性。典型地，应用程序将会一直持久与数据库的连接，直到应用程序退出，这就限制了并发访问数据库的并发数，在这种模型中，所谓的性能、可扩展性以及可用性，需要 JDBC 驱动以及底层的数据库来共同保证。如果应用程序使用的 JDBC 驱动不止一种，那么情况就会更加复杂。

# 4.4 三层模型

三层模型引进了一个中间层，来处理业务逻辑并作为基础设施，如下图所示

![三层模型](/assets/jdbc4.2-spec-4/3layers.jpg)

这种架构对于企业应用来说，性能、可扩展性和可用性都得到提升，各层的职责如下所示：

1. 客户端层 — 仅作为表现层，只需要与中间层交互，而不须了解中间层的基础架构以及底层数据源的功能细节

2. 中间层服务器 — 包含以下几个组成部分：
    - 实现了业务逻辑，并与客户端进行交互的应用程序。如果应用程序需要与底层数据源交互，它只需要关注高层次的抽象和逻辑连接，若不是底层的驱动 API。
    - 为应用程序提供基础设施的应用服务器。这些基础设施包括对物理数据库连接的池化和管理、事务管理，以及对不同驱动 API 的不同点进行屏蔽，最后一点使得我们很容易写出可移植的应用程序，应用服务器这个角色可以由 Java EE 服务器来承担，应用服务器主要实现提供给应用程序使用的抽象层，并负责与 JDBC 驱动交互。
    - 能够与底层数据源建立连接的 JDBC 驱动。每个驱动根据其底层数据源的特性，去实现标准的 JDBC API，驱动层可能会屏蔽掉 SQL2003 标准与数据源支持的 SQL 方言之间的不同。如果底层数据源并不是一个关系型的数据库，驱动需要去实现对应的关系层逻辑，提供给应用服务器使用。

3. 底层的数据源 — 这一层是数据所在的一层，可以包含关系型数据库，文件系统，面向对象数据库，数据仓库等等任何能组织和表现数据的东西，但它们都需要提供符合 JDBC API 规范的驱动。

# 4.5 JDBC 与 JavaEE 平台

Java EE 组件，比如说 JavaServer Pages、Servlets以及 EJB 组件，通常都需要使用 JDBC API 来访问关系型的数据，当 Java EE 组件使用 JDBC API 时，它们的容器会管理事务以及数据源。这意味着 Java EE 组件的开发者不会直接使用 JDBC API 的事务管理和数据源管理的能力。更多内容，请参考 Java EE 平台规范。