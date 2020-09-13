---
title: "Java 各版本的特性（更新到 Java 12）"
date: 2019-03-21T17:36:44+08:00
categories : ["java"]
---

# JDK 1.1 (1997 年 2 月 19 号)
- 引入 JDBC (Java Database Connectivity)；
- 引入内部类 (Inner Classes)；
- 引入 Java Beans；
- 引入 RMI (Remote Method Invocation)；
- 添加只支持内省(Introspection)，但不允许在运行时改动的Java反射机制；
- 对 AWT(java.awt) 事件模型进行大范围的改进；
- 支持 Internationalization 和 Unicode；

# J2SE 1.2 (1998 年 12 月 4 号)
- 引入 Java 插件；
- 引入集合(Collection)框架；
- 引入 JIT(Just In Time)编译器；
- 引入 JFC(Java Foundation Classes)，包括 Swing 1.0、拖放和 Java 2D 类库；
- 引入对打包的 Java 文件进行数字签名；
- 引入控制授权访问系统资源的策略工具；
- 对字符串常量做内存映射；
- 在 Applet 中添加声音支持；
- 在 JDBC 中引入可滚动结果集、BLOB、CLOB、批量更新和用户自定义类型；
- 新增关键字 strictfp(strict float point)；
- 添加可与 CORBA 协同交互的 Java IDL；

# J2SE 1.3 (2000 年 5 月 8 号)
- Jar 文件索引；
- 新增复合代理类(Synthetic proxy classes)；
- 包含了HotSpot JVM(HotSpot JVM 第一次被发布是在 1999 年 4 月，名为 J2SE 1.2 JVM)；
- 改进 RMI(Java remote method’s invocation)对 CORBA 的兼容性；
- JNDI(Java Naming and Directory Interface / Java 命名和目录接口)包含在主程序库中(先前为扩充组件的形式)；
- 添加 JPDA(Java Platform Debugger Architecture / Java 平台调试器体系)，为调试 Java 代码提供了统一的 API；
- 添加 JavaSound API(javax.sound.midi 和 javax.sound.sampled)，提供对语音处理的支持。该平台以前的版本只有有限的音频支持，只能对音频片段进行基本播放。在此新版本中，Java 2 平台定义了一系列标准类和接口，用于低级音频支持；

# J2SE 1.4 (2002 年 2 月 6 号)
- 新增 assert 关键字；
- 新增模仿 Perl 正则表达式的 Java 正则表达式；
- 新增 Exception Chaining (异常链)机制，允许一个异常封装最初的低级异常；
- 添加对网络协议 IPv6 的支持；
- 新增 nio(java.nio)，意即非阻塞式的 I/O(non-blocking I/O)。由于 nio 是不同于以往 I/O 的一种新的 API，因此也被称作 New I/O；
- 新增日志 API(java.util.logging)；
- 新增图像 I/O API，用于支持类似于 JPEG、PNG 等格式的图像的读写操作；
- 集成 XML 解析器和 XSLT 处理器(JAXP)；
- 集成安全和加密扩充组件(JCE, JSSE, JAAS)；
- 内置 Java Web Start 软件，使你可以方便地从 Web 下载和运行 Java 应用程序；
- 新增配置参数 API(java.util.prefs)，它允许应用程序存储并获取用户和系统首选项和配置数据；
- Java 打印服务；
- 引入 JDBC 3.0 API；

# J2SE 5.0 (2004年 9 月 30 号)
- 引入泛型；
- 增强循环，可以使用迭代方式；
- 自动封装与解封装：基本的数据类型(如 int )和基本的的封装器类型(如 Integer )之间能够自动转换；
- 引入枚举( Enumerations )：以 enum 关键字创造出一种类型安全、有排序值的集合(如 Day.MONDAY、 Day.TUESDAY 等)；
- 支持可变长度参数；
- 支持静态引入；
- 支持中继数据(Metadata)：也称作注释，让语言结构能够用额外的数据标记；
- 引入 Instrumentation；
- 自动产生 stub 给 RMI 对象；
- Swing 增加新的接口外观 synth；
- 用 Scanner 类别来解析来自各式各样的输入和缓冲；
- 支持Unicode 4.0；

# Java SE 6 (2006 年 12 月 11 号)
- 引入 JDBC 4.0 API；
- 引入 Java Compiler API：允许 Java 程序以写程序的方式选择和调用 Java 编译器的 API；
- 可插拔注解；
- 增加对 Native PKI(Public Key Infrastructure)、Java GSS(Generic Security Service)、Kerberos 和 LDAP(Lightweight Directory Access Protocol) 的支持；
- 继承 Web Services；
- 通过 JAX-WS 改善网络服务支持；
- 将 JAXB 升级到版本 2.0：包括 StAX解析器的集成；
- 支持 pluggable annotations；
- 许多 GUI 支持的改善，比如 SwingWorker 在 API 中的集成、表格排序和筛选以及真正的 Swing 双缓冲；

# Java SE 7 (2011 年 7 月 28 号)
- 支持动态语言；
- 支持钻石型语法；
- null值的自动处理；
- 压缩了 64 比特的指针；
- 引入Java NIO.2开发包；
- 支持try with resources；
- 在一个语句块中捕获多种异常；
- 在创建泛型对象时应用类型推断；
- switch语句块中允许以字符串作为分支条件；
- 对 elliptic curve cryptography 算法程序库档次的支持；
- Timsort 被用来排序对象的集合和数组，取代 merge sort；
- 数值类型可以用二进制字符串表示，并且可以在字符串表示中添加下划线；
- 增强了对新网络通信协议(包括 SCTP 和 Sockets Direct Protocol )的程序库档次的支持；
- Upstream 对 XML 和 Unicode 的更新；

# Java SE 8 (2014 年 3 月 18 号)
- 支持 lamdba expressions；
- 移除了 PermGen Error；
- 支持方法引用；
- 支持默认方法；
- TLS1.1 和 TLS1.2 被设为默认启动，提高安全性；
- 改进了类型注解和重复注解；
- 引入流操作( Stream )：通过该操作可以实现对集合（Collection）的并行处理和函数式操作；
- 新增流操作 API(java.util.stream)；
- 新增 Date / Time API；
- 新增 JavaScript 引擎 Nashorn；
- 类库新增 Base64 编码支持；
- 支持并行(Parallel)数组；
- 支持并发(Concurrency)；
- 新的Java工具：Nashorn引擎: jjs 和 类依赖分析器 jdeps；
- 新增 Optional 类来解决空指针异常；

# Java SE 9 (2017 年 9 月 122 号)
- 模块系统：模块是一个包的容器，Java 9 最大的变化之一是引入了模块系统（Jigsaw 项目）。 
- REPL (JShell)：交互式编程环境。
- HTTP 2 客户端：HTTP/2标准是HTTP协议的最新版本，新的 HTTPClient API 支持 WebSocket 和 HTTP2 流以及服务器推送特性。
- 改进的 Javadoc：Javadoc 现在支持在 API 文档中的进行搜索。另外，Javadoc 的输出现在符合兼容 HTML5 标准。 
- 多版本兼容 JAR 包：多版本兼容 JAR 功能能让你创建仅在特定版本的 Java 环境中运行库程序时选择使用的 class 版本。
- 集合工厂方法：List，Set 和 Map 接口中，新的静态工厂方法可以创建这些集合的不可变实例。
- 私有接口方法：在接口中使用private私有方法。我们可以使用 private 访问修饰符在接口中编写私有方法。 
- 进程 API: 改进的 API 来控制和管理操作系统进程。引进 java.lang.ProcessHandle 及其嵌套接口 Info 来让开发者逃离时常因为要获取一个本地进程的 PID 而不得不使用本地代码的窘境。 
- 改进的 Stream API：改进的 Stream API 添加了一些便利的方法，使流处理更容易，并使用收集器编写复杂的查询。
- 改进 try-with-resources：如果你已经有一个资源是 final 或等效于 final 变量,您可以在 try-with-resources 语句中使用该变量，而无需在 try-with-resources 语句中声明一个新变量。 
- 改进的弃用注解 @Deprecated：注解 @Deprecated 可以标记 Java API 状态，可以表示被标记的 API 将会被移除，或者已经破坏。
- 改进钻石操作符(Diamond Operator) ：匿名类可以使用钻石操作符(Diamond Operator)。 
- 改进 Optional 类：java.util.Optional 添加了很多新的有用方法，Optional 可以直接转为 stream。 
- 多分辨率图像 API：定义多分辨率图像API，开发者可以很容易的操作和展示不同分辨率的图像了。 
- 改进的 CompletableFuture API ： CompletableFuture 类的异步机制可以在 ProcessHandle.onExit 方法退出时执行操作。 
- 轻量级的 JSON API：内置了一个轻量级的JSON API 
- 响应式流（Reactive Streams) API: Java 9中引入了新的响应式流 API 来支持 Java 9 中的响应式编程。

# Java SE 10 (2018 年 3 月 21 号)
- var 局部变量类型推断
- 将原来用 Mercurial 管理的众多 JDK 仓库代码，合并到一个仓库中，简化开发和管理过程
- 统一的垃圾回收接口
- G1 垃圾回收器的并行完整垃圾回收，实现并行性来改善最坏情况下的延迟
- 应用程序类数据 (AppCDS) 共享，通过跨进程共享通用类元数据来减少内存占用空间，和减少启动时间
- ThreadLocal 握手交互。在不进入到全局 JVM 安全点 (Safepoint) 的情况下，对线程执行回调。优化可以只停止单个线程，而不是停全部线程或一个都不停
- 移除 JDK 中附带的 javah 工具。可以使用 javac -h 代替
- 使用附加的 Unicode 语言标记扩展
- 能将堆内存占用分配给用户指定的备用内存设备
- 使用 Graal 基于 Java 的编译器，可以预先把 Java 代码编译成本地代码来提升效能

# Java SE 11 (2018 年 9 月 25 号）
- 基于嵌套的访问控制
- 动态的类文件常量
- 改进 Aarch64 Intrinsics
- Epsilon 垃圾回收器，又被称为"No-Op（无操作）"回收器
- 移除 Java EE 和 CORBA 模块，JavaFX 也已被移除
- 标准 HTTP CLIENT
- 用于 Lambda 参数的局部变量语法
- 采用 Curve25519 和 Curve448 算法实现的密钥协议
- Unicode 10
- 飞行记录仪
- 实现 ChaCha20 和 Poly1305 加密算法
- 启动单个 Java 源代码文件的程序
- 低开销的堆分配采样方法
- 对 TLS 1.3 的支持
- ZGC：可伸缩的低延迟垃圾回收器，处于实验性阶段
- 弃用 Nashorn JavaScript 引擎
- 弃用 Pack200 工具及其 API

# Java SE 12 (2019 年 3 月 19 号）
- 低暂停时间的 GC
- 微基准测试套件
-  Switch 表达式
- JVM 常量 API
- 只保留一个 AArch64 实现
- 默认类数据共享归档文件
- 可中止的 G1 Mixed GC
-  G1 及时返回未使用的已分配内存