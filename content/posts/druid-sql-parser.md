---
title: "Druid SQL 解析器"
date: 2017-06-07T17:43:33+08:00
categories : ["解析器"]
---


# 认识 Druid

Druid 是阿里巴巴公司开源的一个数据库连接池，它的口号是：**为监控而生的数据库连接池**

根据[官方 wiki](https://github.com/alibaba/druid/wiki)的介绍

> Druid 是一个 JDBC 组件库，包括数据库连接池、SQL Parser 等组件，DruidDataSource 是最好的数据库连接池。

显然，官方有意无意地强调了 DruidDataSource 是最好的数据库连接池 -_- ...


# Druid SQL 解析器

Druid 作为一个数据库连接池，功能很多，但我接触 Druid 的时候，却不是因为它有世界上最好的数据库连接池实现。而是因为有些开源项目(比如，mycat)，借用了 Druid 的 SQL 解析功能。我需要研究这个开源项目，发现作为一个数据库中间件，它的 SQL 解析功能是直接引用的 Druid，Druid 包除了 SQL 解析模块的代码外，其它的代码并没有使用到。而这部分代码显然让人在研究 SQL 解析器代码时容易分心，产生厚重感和焦虑感。

Druid 本来的代码结构如下：
![提取前代码结构](/assets/druid-sql-parser/code-structure-before.jpg)

# 提取 Druid SQL 解析器

在确认我并不需要使用到全世界最好的数据库连接池后，我想把除了 SQL 解析部分的代码全部剔除，仅仅留下 SQL 解析器模块。

一开始的做法当然是“暴力删除”，通过对代码的整体浏览，大概判断出哪些 package 与 SQL 解析有关，其余的直接删除。这样做会有些问题，比如说直接删除后在 IDE 中会立马浮现一些小红叉叉，但令人感到愉悦的是，Druid 的模块分解做得十分优秀，SQL 解析模块基本上作为一个工具模块，与其它模块实际上是分离的。

因此虽然是“暴力删除”，却也得到了一个令人满意的结果。

由于我只关注的是 Druid 对 MySQL 方言的解析，并且也不想看到 Druid 解析其它数据库方言的内容，也不愿被 Druid 那些为了适应多种数据库的“兼容性代码”混淆视听，因此狠下心来，把对其它 SQL 方言的支持也全都剔除，只留下与 MySQL 相关的代码。

剔除其它 SQL 方言并不是一个麻烦事，这也得益于 Druid 优秀的代码层次结构，基本上，只是拿类似以下形式的代码动刀
```java
if type is oracle:
	do something;
else if type is db2:
	do something;
else if type is H2:
	do something;
else if type is MySQL:
	do something;
```

把上面的代码，修改成：

```java
if type is MySQL:
	do something;
else:
	throw some exception;
```

经过两层提取后，整个 Druid 就只剩下这些代码了

![提取后代码结构](/assets/druid-sql-parser/code-structure-after.jpg)

# 解析器概览

Druid 的[官方 wiki](https://github.com/alibaba/druid/wiki) 对 SQL 解析器部分的讲解内容并不多，但虽然不多，也有利于完全没接触过 Druid 的人对 SQL 解析器有个初步的印象。

说到解析器，脑海里便很容易浮现 **parser** 这个单词，然后便很容易联想到计算机科学中理论性比较强的学科------**编译原理**。想必很多人都知道（即使不知道，应该也耳濡目染）能够手写编译器的人并不多，并且这类人呢，理论知识和工程能力都比较强。在缺乏人力的条件下，大多数时候实现一个编译器，往往是选择采用一些工具，比如说 **ANTLR**，只需要描述好语法规则，这个工具就能生成对应的编译器。

不过，Druid 的 SQL 解析器是手写的，官方宣称性能是 ANTLR 这类工具的10倍以上。

# 解析器组成部分

在 Druid 的 SQL 解析器中，有三个重要的组成部分，它们分别是：
- Parser
	- 词法分析
	- 语法分析
- AST(Abstract Syntax Tree，抽象语法树)
- Visitor

这三者的关系如下图所示：

![关系](/assets/druid-sql-parser/relationship.jpg)

Parser 由两部分组成，词法分析和语法分析。
当拿到一条形如 `select id, name from user` 的 SQL 语句后，首先需要解析出每个独立的单词，select，id，name，from，user。这一部分，称为**词法分析**，也叫作 **Lexer**。
通过词法分析后，便要进行语法分析了。
经常能听到很多人在调侃自己英文水平很一般时会说：**26个字母我都知道，但是一组合在一起我就不知道是什么意思了**。这说明他掌握了词法分析的技能，却没有掌握语法分析的技能。
那么对于 SQL 解析器来说呢，它不仅需要知道每个单词，而且要知道这些单词组合在一起后，表达了什么含义。语法分析的职责就是明确一个语句的语义，表达的是什意思。
自然语言和形式语言的一个重要区别是，自然语言的一个语句，可能有多重含义，而形式语言的一个语句，只能有一个语义;形式语言的语法是人为规定的，有了一定的语法规则，语法解析器就能根据语法规则，解析出一个语句的一个唯一含义。

AST 是 Parser 的产物，语句经过词法分析，语法分析后，它的结构需要以一种计算机能读懂的方式表达出来，最常用的就是抽象语法树。
树的概念很接近于一个语句结构的表示，一个语句，我们经常会对它这样看待：它由哪些部分组成？其中一个组成部分又有哪些部分组成？例如一条 select 语句，它由 select 列表、where 子句、排序字段、分组字段等组成，而 select 列表则由一个或多个 select 项组成，where 子句又由一个或者多个 where条件组成。
在我们人类的思维中，这种组成结构就是一个总分的逻辑结构，用树来表达，最合适不过。并且对于计算机来说，它显然比人类更擅长处理“树”。

AST 仅仅是语义的表示，但如何对这个语义进行表达，便需要去访问这棵 AST，看它到底表达什么含义。通常遍历语法树，使用 VISITOR 模式去遍历，从根节点开始遍历，一直到最后一个叶子节点，在遍历的过程中，便不断地收集信息到一个上下文中，整个遍历过程完成后，对这棵树所表达的语法含义，已经被保存到上下文了。有时候一次遍历还不够，需要二次遍历。遍历的方式，广度优先的遍历方式是最常见的。

# 快速上手

使用 Druid SQL Parser 来解析 SQL 语句，一般需要进行以下几个步骤：
1. 新建一个 Parser
2. 使用 Parser 解析 SQL，生成 AST
3. 使用 Visitor 访问 AST

如下代码所示：
```java
package io.beansoft.demo;

import com.alibaba.druid.sql.ast.SQLStatement;
import com.alibaba.druid.sql.dialect.mysql.parser.MySqlStatementParser;
import com.alibaba.druid.sql.dialect.mysql.visitor.MySqlSchemaStatVisitor;
import com.alibaba.druid.sql.parser.SQLStatementParser;

/**
 *
 *
 * @author beanlam
 * @date 2017年1月10日 下午11:06:26
 * @version 1.0
 *
 */
public class ParserMain {

    public static void main(String[] args) {
        String sql = "select id,name from user";

        // 新建 MySQL Parser
        SQLStatementParser parser = new MySqlStatementParser(sql);

        // 使用Parser解析生成AST，这里SQLStatement就是AST
        SQLStatement statement = parser.parseStatement();

        // 使用visitor来访问AST
        MySqlSchemaStatVisitor visitor = new MySqlSchemaStatVisitor();
        statement.accept(visitor);
				
        // 从visitor中拿出你所关注的信息        
        System.out.println(visitor.getColumns());
    }
}
```

以上代码运行后控制台的输出为
```bash
[user.id, user.name]
```

当然，不使用 Visitor，直接操作 AST 也可以得到 SQL 语句的信息，在 Druid 现有内置的 Visitor 不能满足需求时，可以自己去实现 Visitor，对于用 Visitor 无法解析到的信息，可以直接访问 AST 去获取。

# 解析过程
以这份代码为例

```java
/**
 *
 *
 * @author beanlam
 * @date 2017年1月10日 下午11:06:26
 * @version 1.0
 *
 */
public class ParserMain {

    public static void main(String[] args) {
        
        String sql = "select * from user order by id";

        // 新建 MySQL Parser
        SQLStatementParser parser = new MySqlStatementParser(sql);

        // 使用Parser解析生成AST，这里SQLStatement就是AST
        SQLStatement statement = parser.parseStatement();

        // 使用visitor来访问AST
        MySqlSchemaStatVisitor visitor = new MySqlSchemaStatVisitor();
        statement.accept(visitor);
        
        System.out.println(visitor.getColumns());
        System.out.println(visitor.getOrderByColumns());
    }
}
```

一开始，需要初始化一个 Parser，在这里 `SQLStatementParser` 是一个父类，真正解析 SQL 语句的 Parser 实现是 `MySqlStatementParser`。
Parser 的解析结果是一个 `SQLStatement`，这是一个内部维护了树状逻辑结构的类。

## 词法分析

Druid 的代码里，代表**语法分析**和**词法分析**的类分别是 `SQLParser` 和 `Lexer`。并且， Parser 拥有一个 Lexer。
```java
public class SQLParser {

    protected final Lexer lexer;

    protected String      dbType;

    public SQLParser(String sql, String dbType){
        this(new Lexer(sql), dbType);
        this.lexer.nextToken();
    }

    public SQLParser(String sql){
        this(sql, null);
    }

    public SQLParser(Lexer lexer){
        this(lexer, null);
    }

    public SQLParser(Lexer lexer, String dbType){
        this.lexer = lexer;
        this.dbType = dbType;
    }
}
```

经过瘦身后的 Druid 代码，其 Lexer 只有两个，分别是 `Lexer`，以及它的子类 `MySqlLexer`
Lexer 作为词法分析器，必然拥有其词汇表，在Lexer里，以 `Keywords` 表示。
```java
protected Keywords keywods = Keywords.DEFAULT_KEYWORDS;
```

Keywords 实际上是 key 为单词，value 为 Token 的字典型结构，其中 Token 是单词的类型，比如说，“select” 的 Token 类型就是 Select Token，而 “abc” 的 Token 类型，则是标识符，也表示为 Identifier Token。

而 `MySqlLexer` 类，除了沿用其父类的 Keywords 外，自己还有自己的 Keywords。可以理解为 Lexer 所维护的关键字集合，是通用的；而 MySqlLexer 除了有通用的关键字集合，也有属于 MySQL 数据库 SQL 方言的关键字集合。

Parser 是 Lexer 的使用者，站在 Parser 的角度看，它会怎么去使用 Lexer，或者说，Lexer 应该具备怎样的功能，才能满足 Parser 的使用需求。
Lexer 应该具备一个函数，能让使用者命令它解析一个单词，并且 Lexer 还必须提供一个函数，供使用者获取 Lexer 上一次解析到的单词以及单词的类型。
在 Lexer 中，`nextToken()` 这个方法提供了第一个需求，只要被调用，它就按顺序从 SQL 语句的开头到结尾，解析出下一个单词；`token()` 方法，则返回了上一次解析的单词的 Token 类型，如果 Token 类型是标识符（Identifier），Lexer 还提供了一个 `stringVal()` 方法，让使用者能拿到标识符的值。

走进 Lexer 的 `nextToken()` 方法，可以发现它的代码充斥着 `if` 语句和 `switch` 语句，因为解析单词的时候，是一个字符一个字符地解析，这就意味着，这个方法每次扫描一个字符，都必须判断单词是否结束，应该用什么方式来验证这个单词等等。这个过程，就是一个**状态机**运作的过程，每解析到一个字符，都要判断当前的状态，以决定应该进入下一个什么状态。

## Select 语法分析


有了 Lexer 这样的犀利工具，接下来就是 Parser 发挥的时候了，从 Demo 代码里可以看到，解析的开始，在于调用 `parser.parseStatement()` 方法。进到这个方法看看，发现清一色是形似如下格式的代码：

```java
if (lexer.token() == Token.xxx) {
	// 这里解析 xxx 类型
	return;
}

if (lexer.token() == Token.aaa) {
	// 这里解析 aaa 类型
	return;
}
```

显然，如果是分析对 Select 类型的语句的解析，那么应该关注以下的代码：

```java
if (lexer.token() == Token.SELECT) {
    statementList.add(parseSelect());
    continue;
}
```
重点是 `parseSelect()` 方法，`MySqlStatementParser` 重载了它的父类的这个方法，因此这个方法实际上的实现细节是这样的
```java
    @Override
    public SQLStatement parseSelect() {
        MySqlSelectParser selectParser = new MySqlSelectParser(this.exprParser);
        
        SQLSelect select = selectParser.select();
        
        if (selectParser.returningFlag) {
            return selectParser.updateStmt;
        }
        
        return new SQLSelectStatement(select, JdbcConstants.MYSQL);
    }
```

初始化一个针对 MySQL Select 语句的 Parser，然后调用 `select()` 方法进行解析，把返回结果 `SQLSelect` 放到 `SQLSelectStatement` 里，而这个 `SQLSelectStatement`，便是我最关心的 AST 抽象语法树，SQLSelect 是它的第一个子节点。

抛开解析的细节不谈，实际上我会非常关心这棵 AST 的层次结构。

## Select 抽象语法树

打开 `SQLSelectStatement` 的代码，扫描它的子成员，便分析出这样的一棵语法树：


![语法树](/assets/druid-sql-parser/ast.jpg)


这意味着，在 Druid 眼里，它是这样看待一条 Select 语句的所有成员部分的。

## Visitor

从 demo 代码中可以看到，有了 AST 语法树后，则需要一个 visitor 来访问它
```java
        // 使用visitor来访问AST
        MySqlSchemaStatVisitor visitor = new MySqlSchemaStatVisitor();
        statement.accept(visitor);
        
        System.out.println(visitor.getColumns());
        System.out.println(visitor.getOrderByColumns());
```

statement 调用 accept 方法，以 visitor 作为参数，开始了访问之旅。在这里 statement 的实际类型是 `SQLSelectStatement`。

在 Druid 中，一条 SQL 语句中的元素，无论是高层次还是低层次的元素，都是一个 `SQLObject`，statement 是一种 SQLObject，表达式 expr 也是一种 SQLObject，函数、字段、条件等等，这些都是一种 SQLObject，SQLObject 是一个接口，`accept` 方法便是它定义的，目的是为了让访问者在访问 SQLObject 时，告知访问者一些事情，好让访问者在访问的过程中能够收集到关于该 SQLObject 的一些信息。

具体的 `accept()` 实现，在 `SQLObjectImpl` 这个类中，代码如下所示：
```java
    public final void accept(SQLASTVisitor visitor) {
        if (visitor == null) {
            throw new IllegalArgumentException();
        }

        visitor.preVisit(this);

        accept0(visitor);

        visitor.postVisit(this);
    }
```

这是一个 final 方法，意味着所有的子类都要遵循这个模板，首先 accept 方法前和后，visitor 都会做一些工作。真正的访问流程定义在 `accept0()` 方法里，而它是一个**抽象方法**。

因此要知道 Druid 中是如何访问 AST 的，先拿 SQLSelectStatement 的 accept0() 方法来探探究竟。
```java
    protected void accept0(SQLASTVisitor visitor) {
        if (visitor.visit(this)) {
            acceptChild(visitor, this.select);
        }
        visitor.endVisit(this);
    }
```

首先，使 visitor 访问自己，访问自己后，visitor 会决定是否还要访问自己的子元素。
打开 `MySqlSchemaStateVisitor` 的 visit 方法，可以看到，visitor 做了一些事，初始化了自己的 aliasMap，然后 return true，这意味着还要访问 SQLSelectStatement 的子节点。
```java
    public boolean visit(SQLSelectStatement x) {
        setAliasMap();
        return true;
    }
```

接下来访问子元素
```java
    protected final void acceptChild(SQLASTVisitor visitor, SQLObject child) {
        if (child == null) {
            return;
        }

        child.accept(visitor);
    }
```

由此可以看出，SQLObject 负责通知 visitor 要访问自己的哪些元素，而 visitor 则负责访问相应元素前，中，后三个过程的逻辑处理。