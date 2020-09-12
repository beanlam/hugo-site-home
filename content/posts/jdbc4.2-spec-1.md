---
title: "JDBC4.2规范-第一章 简介"
date: 2017-01-02T13:34:42+08:00
categories : ["jdbc"]
---


# 1.1 JDBC API 简介

JDBC API 为 Java 语言提供了一种访问关系型数据库的方法。有了 JDBC API，应用程序便可以通过它来执行 SQL 语句、获取执行结果，以及对底层的数据库进行写入。JDBC API 也可以用来和多个数据源交互，这些数据源以分布式的形式部署。

JDBC API 基于 **X/Open SQL CLI** 标准，ODBC 也以它为标准。对于 **X/Open SQL CLI** 标准所定义的一些抽象概念，JDBC API 提供了对应的概念的映射，并且非常易用以及容易理解。

自从 1997 年 JDBC API 首次被提出以后， 它的被接受程度越来越大，并且也出现了大量的对于 JDBC API 规范的实现，这都归功于 JDBC API 本身的灵活性。

# 1.2 平台

JDBC API 是 Java 语言的一部分，JDBC API 分为两个 package，分别是 `java.sql` 和 `javax.sql`。在 Java SE 平台和 Java EE 平台都存在这两个 package。

# 1.3 目标读者

- 需要实现相应的驱动的数据库厂商
- 需要在 JDBC 驱动上建立多层服务的应用服务器厂商
- 需要基于 JDBC API 来开发应用的厂商

本文档也适用于任何在 JDBC API 上层开发应用的开发者

# 1.4 致谢

译者省略