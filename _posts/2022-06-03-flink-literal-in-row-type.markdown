---
title: "浅谈 Flink - Row 里使用字符串的 ParseException"
date: 2022-06-03 02:37:26
tags: [flink-1.15]
---

## 1. Row 里显式使用字符串的问题

在 20 年初最开始使用 Flink 1.9.1 时，有一些看似普通的 SQL 也会执行失败。比如下面这个:

```scala
package cn.izualzhy

import org.apache.flink.streaming.api.scala._
import org.apache.flink.table.api.bridge.scala.StreamTableEnvironment

object LiteralInRowTest extends App {
  val env = StreamExecutionEnvironment.createLocalEnvironment(1)
  val tEnv = StreamTableEnvironment.create(env)

  val source = env.fromElements("test_f0_value")
  val source_table = tEnv.fromDataStream(source).as("f0")

  tEnv.createTemporaryView(
    "source_table",
    source_table)
  val sql = "SELECT Row(f0, 'literal_column') FROM source_table"
//  val sql = "SELECT Row('literal_column', f0) FROM source_table"
  tEnv.executeSql(sql).print()
}
```

我试了下直到 1.15.0 里仍会报错：

```
Caused by: org.apache.flink.sql.parser.impl.ParseException: Encountered "\'literal_column\'" at line 1, column 16.
```

不过神奇的是，把 Row(...) 里的元素换一下顺序，执行另外一行 sql 就能成功：

```
+----+--------------------------------+
| op |                         EXPR$0 |
+----+--------------------------------+
| +I | (literal_column, test_f0_va... |
+----+--------------------------------+
```

这种错误，如同最开始在调研 FlinkSQL 时踩的其他坑一样莫名其妙，只能通过临时绕过去的方式解决。最近重新分析了下这个问题的原因和可能的解决方案。

## 2. Calcite 复现

Flink 使用的是 Calcite 1.26.0，我们看下该版本 Calcite 解析相同 SQL 的效果：

```scala
package cn.izualzhy

import org.apache.calcite.sql.parser.SqlParser

object LiteralInRowTest extends App {
  val sql = "SELECT Row(f0, 'literal_column') FROM source_table"
//  val sql = "SELECT Row('literal_column', f0) FROM source_table"
  val sqlParser = SqlParser.create(sql)

  val sqlNode = sqlParser.parseQuery()
}
```

可以看到几乎一样的报错：

```
Caused by: org.apache.calcite.sql.parser.impl.ParseException: Encountered "\'literal_column\'" at line 1, column 16.
```

更换 Row(...) 元素顺序后，sql 同样执行正确。

**结论：Flink 的执行行为跟 Calcite 一致，根源在 Calcite.**

## 3. Calcaite - ROW 的处理

分析Parser.jj<sup>1</sup>里 ROW 的处理逻辑： 

```
    LOOKAHEAD(3)
    <ROW> {
        s = span();
    }
    list = ParenthesizedSimpleIdentifierList() {
        if (exprContext != ExprContext.ACCEPT_ALL
            && exprContext != ExprContext.ACCEPT_CURSOR
            && !this.conformance.allowExplicitRowValueConstructor())
        {
            throw SqlUtil.newContextException(s.end(list),
                RESOURCE.illegalRowExpression());
        }
        return SqlStdOperatorTable.ROW.createCall(list);
    }
|
    [
        <ROW> { rowSpan = span(); }
    ]
    list1 = ParenthesizedQueryOrCommaList(exprContext) {
```

有两个分支，其中`ParenthesizedSimpleIdentifierList`方法要求匹配的字符串形如`(identifier_a, identifier_b, ...)`，即括号包括的列名(identifier)，如果括号内非列名例如字符串就报错。而另一个分支`ParenthesizedQueryOrCommaList`则没有这个限制。

这就是为什么`Row(f0, 'literal_column')`执行失败，而`Row('literal_column', f0)`正常执行的原因。

而匹配到哪个逻辑，则是`LOOKAHEAD(3)`的效果

## 4. Calcaite - LOOKAHEAD

LOOKAHEAD简言之，是为了解决在词法分析时的二义性以及性能问题。

举个例子：

```
    void basic_expr() :
    {}
    {
      <ID> "(" expr() ")"   // Choice 1
    |
      "(" expr() ")"    // Choice 2
    |
      "new" <ID>        // Choice 3
    }
```

该 jj 文件逻辑上解析为：

```
    if (next token is <ID>) {
      choose Choice 1
    } else if (next token is "(") {
      choose Choice 2
    } else if (next token is "new") {
      choose Choice 3
    } else {
      produce an error message
    }
```

解释后的逻辑清晰，没有什么问题。不过如果对 jj 文件做一些改动：

```
    void basic_expr() :
    {}
    {
      <ID> "(" expr() ")"   // Choice 1
    |
      "(" expr() ")"    // Choice 2
    |
      "new" <ID>        // Choice 3
    |
      <ID> "." <ID>     // Choice 4
    }
```

就会报错：

```
 A common prefix is: <ID>
 Consider using a lookahead of 2 for earlier expansion.
```

原因是`if (next token is <ID>)`有两种选择，词法分析器懵了。

当然从我们的角度，只要再分析下下个 token 是 "(" or "." 就可以区分，换成 javacc 的语法，就是 LOOKAHEAD(2)，即往前查看两个 token.

```
    void basic_expr() :
    {}
    {
      LOOKAHEAD(2)
      <ID> "(" expr() ")"   // Choice 1
    |
      "(" expr() ")"    // Choice 2
    |
      "new" <ID>        // Choice 3
    |
      <ID> "." <ID>     // Choice 4
    }
```

逻辑上第一个 if 条件就发生了变化：

```
    if (next 2 tokens are <ID> and "(" ) {
      choose Choice 1
    } else if (next token is "(") {
      choose Choice 2
    } else if (next token is "new") {
      choose Choice 3
    } else if (next token is <ID>) {
      choose Choice 4
    } else {
      produce an error message
    }
```

词法分析有这样的问题是正常的，比如 C++ 里也会有让初学者困惑的[Two phase lookup](https://izualzhy.cn/two-phase-lookup)。而对 javacc，往前查找太多虽然会更精确，但是性能较差，所以交给了用户来根据自己的场景决定。

本质上还是召回率和准确率的取舍问题。

关于 LOOKAHEAD 更详细的用法在<sup>2</sup>

## 5. 解决

首先是 Calcite，回到 Parser.jj 文件。

`LOOKAHEAD(3)`即往前查找 3 个 token 后，如果满足条件，就会执行`ParenthesizedSimpleIdentifierList`的逻辑。而实际上如果继续查找的话，很快就会发现`'literal_column'`不符合 if 条件了。

因此，一个可能的解决方案就是增大 LOOKAHEAD 或者直接走到下一个逻辑，比如修改为 LOOKAHEAD(5) 或者 LOOKAHEAD({false})。

按照前者来修改后重新编译下 Calcite 安装到本地仓库，重新执行第二节的测试代码，就会发现可以执行成功了。

*注：修改后生成的文件:SqlParserImpl-change-row-lookahead-args.java<sup>3</sup>*

接着是 Flink，注意 1.15 里引入了 flink-table-planner-loader 这货，因此我们必须重新编译、安装该 jar。之后第一节的测试代码，也可以执行成功了：

```
+----+--------------------------------+
| op |                         EXPR$0 |
+----+--------------------------------+
| +I | (test_f0_value, literal_col... |
+----+--------------------------------+
```

当然合理的解决方案应该是先明确 ROW 类型是如何定义的，从 1.26.0 代码反推的话，如果Row(...)内第一个元素是列名，那么之后的应当都是列名？

翻了下 Calcite master的 Parser.jj 代码<sup>4</sup>

```
    LOOKAHEAD(3)
    <ROW> {
        s = span();
    }
    list = ParenthesizedQueryOrCommaList(exprContext) {
```

可以看到 2021.01 是这么提交修改的：后续处理逻辑统一成`ParenthesizedQueryOrCommaList`方法，应该也可以解决这个问题，Flink 相关的是这个嵌套类型的 issue<sup>5</sup>，还是 OPEN.

## 6. 参考资料

1. [Parser.jj on 1.26.0](https://github.com/apache/calcite/blob/calcite-1.26.0/core/src/main/codegen/templates/Parser.jj#L3660)
2. [JavaCC LOOKAHEAD MiniTutorial](https://www.cs.purdue.edu/homes/hosking/javacc/doc/lookahead.html)
3. [SqlParserImpl-change-row-lookahead-args](https://github.com/izualzhy/BigData-Systems/blob/main/calcite/src/main/resources/SqlParserImpl-change-row-lookahead-args.java)
4. [Parser.jj on Master](https://github.com/apache/calcite/blame/main/core/src/main/codegen/templates/Parser.jj#L3831)
5. [ROW value constructor cannot deal with complex expressions](https://issues.apache.org/jira/browse/FLINK-18027)
