---
title: "Calcite笔记之三：处理流程的代码例子"
date: 2022-05-01 10:33:20
tags: [Calcite]
---

单纯看[Caclite 架构](https://izualzhy.cn/calcite-arch)理解不深，这篇笔记通过代码示例补充下处理流程：Parser、Valiator、SqlToRelConverter 等。

实现主要参考了<sup>1</sup>和源码单测里的`CsvTest`，使用 Book & Author 表，查询 SQL 跟[关系代数](https://izualzhy.cn/calcite-arch#1-relational-algebra)里基本一致以方便前后对比，完整的代码可以参考[CalciteProcessSteps.scala](https://github.com/yingshin/BigData-Systems/blob/main/calcite/src/main/scala/cn/izualzhy/CalciteProcessSteps.scala)

## 1. SqlParser

```scala
  val query =
    """SELECT b.id,
      |         b.title,
      |         b.publish_year,
      |         a.fname,
      |         a.lname
      |FROM Book AS b
      |LEFT OUTER JOIN Author a
      |    ON b.author_id = a.id
      |WHERE b.publish_year > 1830
      |ORDER BY b.id LIMIT 5
      |""".stripMargin
  // parse: sql -> SqlNode
  val sqlParser = SqlParser.create(query)
  val sqlNodeParsed = sqlParser.parseQuery()
  println(s"[Parsed query]\n${sqlNodeParsed}")
```

解析阶段没有引入依赖，所以不容易出错，主要是将 Query 解析为各类 SqlNode，以 sqlNodeParsed 为根节点的一棵语法树：

```
sqlNodeParsed(SqlOrderBy)
├── query(SqlSelect)
│   ├── keyWordList(SqlNodeList)
│   ├── selectList(SqlNodeList)
│   ├── from(SqlJoin)
│   │   ├── left(SqlBasicCall)
│   │   ├── right(SqlBasicCall)
│   │   ├── condition(SqlBasicCall)
│   │   ├── joinType(SqlLiteral)
│   │   ├── ...
│   ├── where(SqlBasicCall)
│   ├── ...
├── orderList(SqlNodeList)
├── fetch(SqlNumericLiteral)
```

用图表示的话：

![AST](/assets/images/calcite/ast.png)

## 2. SqlValidator

检测合法性的阶段，就需要引入表结构，使用到的两张表结构如下：

**Book**表结构：

|字段名  |字段类型  |
|--|--|
|id  |int  |
|title  |string  |
|price  |decimal  |
|publish_year  |int  |
|author_id  |int  |

**Author**表结构：

|字段名  |字段类型  |
|--|--|
|id  |int  |
|fname  |string  |
|lname  |string  |
|birth  |date  |

```scala
  // 参考第一篇笔记里的 CsvSchema 初始化 author book 表，resources/book_author 目录
  val rootSchema = Frameworks.createRootSchema(true)
  val csvPath = getClass.getClassLoader.getResource("book_author").getPath
  val csvSchema = new CsvSchema(new File(csvPath.toString), CsvTable.Flavor.SCANNABLE)
  rootSchema.add("author", csvSchema.getTable("author"))
  rootSchema.add("book", csvSchema.getTable("book"))

  val sqlTypeFactory = new JavaTypeFactoryImpl()
  val properties = new Properties()
  properties.setProperty(CalciteConnectionProperty.CASE_SENSITIVE.camelName(), "false")
  // reader 接收 schema，用于检测字段名、字段类型、表名等是否存在和一致
  val catalogReader = new CalciteCatalogReader(
    CalciteSchema.from(rootSchema),
    CalciteSchema.from(rootSchema).path(null),
    sqlTypeFactory,
    new CalciteConnectionConfigImpl(properties))
  // 简单示例，大部分参数采用默认值即可
  val validator = SqlValidatorUtil.newValidator(
    SqlStdOperatorTable.instance(),
    catalogReader,
    sqlTypeFactory,
    SqlValidator.Config.DEFAULT)
  // validate: SqlNode -> SqlNode
  val sqlNodeValidated = validator.validate(sqlNodeParsed)
  println(s"[Validated query]\n${sqlNodeParsed}")
```

代码注释比较详细，就不再赘述了

## 3. SqlToRelConverter

```scala
  val rexBuilder = new RexBuilder(sqlTypeFactory)
  val hepProgramBuilder = new HepProgramBuilder()
  hepProgramBuilder.addRuleInstance(CoreRules.FILTER_INTO_JOIN)
  val hepPlanner = new HepPlanner(hepProgramBuilder.build())
  hepPlanner.addRelTraitDef(ConventionTraitDef.INSTANCE)

  val relOptCluster = RelOptCluster.create(hepPlanner, rexBuilder)
  val sqlToRelConverter = new SqlToRelConverter(
    // 没有使用 view
    new ViewExpander {
      override def expandView(rowType: RelDataType, queryString: String, schemaPath: util.List[String], viewPath: util.List[String]): RelRoot = null
    },
    validator,
    catalogReader,
    relOptCluster,
    // 均使用标准定义即可
    StandardConvertletTable.INSTANCE,
    SqlToRelConverter.config())
  var logicalPlan = sqlToRelConverter.convertQuery(sqlNodeValidated, false, true).rel
  println(RelOptUtil.dumpPlan("[Logical plan]", logicalPlan, SqlExplainFormat.TEXT, SqlExplainLevel.NON_COST_ATTRIBUTES))
```

这一步则由语法树转换为了关系运算符组成的树：

```
[Logical plan]
LogicalSort(sort0=[$0], dir0=[ASC], fetch=[5]), id = 8
  LogicalProject(ID=[$0], TITLE=[$1], PUBLISH_YEAR=[$3], FNAME=[$6], LNAME=[$7]), id = 7
    LogicalFilter(condition=[>($3, 1830)]), id = 6
      LogicalJoin(condition=[=($4, $5)], joinType=[left]), id = 5
        LogicalTableScan(table=[[book]]), id = 1
        LogicalTableScan(table=[[author]]), id = 3
```

![logical plan](/assets/images/calcite/logical plan.png)

到这一步，其实我们发现跟[关系运算树](https://izualzhy.cn/calcite-arch#1-relational-algebra)在逻辑上是一致的，只是用计算机的语言表达了出来。

## 4. RelOptPlanner

这一步即优化部分，比如我们在第一篇笔记里提到的，扫描各表时，仅获取需要的列。

观察上一节的图，还有一个比较明显的优化手段，就是扫描 Book 表时，提前应用`b.publish_year > 1830`这个条件，减少 b 表的 IO。这个想法，对应就是 Calcite 的内置 FilterIntoJoinRule 规则。

```scala
  // Start the optimization process to obtain the most efficient physical plan based on the
  // provided rule set.
  hepPlanner.setRoot(logicalPlan)
  val phyPlan = hepPlanner.findBestExp()
  println(RelOptUtil.dumpPlan("[Physical plan]", phyPlan, SqlExplainFormat.TEXT, SqlExplainLevel.NON_COST_ATTRIBUTES))
```

对比 Logical Plan 和 Physical Plan，`LogicalFilter`下推到了 Scan 阶段：

```
[Physical plan]
LogicalSort(sort0=[$0], dir0=[ASC], fetch=[5]), id = 17
  LogicalProject(ID=[$0], TITLE=[$1], PUBLISH_YEAR=[$3], FNAME=[$6], LNAME=[$7]), id = 15
    LogicalJoin(condition=[=($4, $5)], joinType=[left]), id = 22
      LogicalFilter(condition=[>($3, 1830)]), id = 19
        LogicalTableScan(table=[[book]]), id = 1
      LogicalTableScan(table=[[author]]), id = 3
```

Rule 优化就跟第三方的实现有关了，比如之前看在看阿里 MaxCompute 时就提到过优化 MergeJoin + GroupAggregate 算子。我的理解是，MergeJoin 前本身就需要对字段排序，所以如果字段在同一区间，就会分配到同一实例实现拉链式 Join，也就是此时完全可以提前进行该字段的 GroupAggregate 而不影响其准确性。

可见其思路也是尽量降低不必要的 IO，我觉得跟[Z-Order](https://izualzhy.cn/lakehouse-zorder#2-data-skipping)一样，核心是 DataSkipping.

## 5. RelBuilder

在 Calcite 的架构图里，还有一个入口是 Expressions Builder (RelBuilder)，即不通过 SQL 直接构建关系运算符的树结构。

写了一个代码示例：

```scala
  val rootSchema = Frameworks.createRootSchema(true)
  val csvPath = getClass.getClassLoader.getResource("book_author").getPath
  val csvSchema = new CsvSchema(new File(csvPath.toString), CsvTable.Flavor.SCANNABLE)
  rootSchema.add("author", csvSchema.getTable("author"))
  rootSchema.add("book", csvSchema.getTable("book"))

  val frameworkConfig = Frameworks.newConfigBuilder()
    .parserConfig(SqlParser.Config.DEFAULT)
    .defaultSchema(rootSchema)
    .build()
  val relBuilder = RelBuilder.create(frameworkConfig)

  val node = relBuilder
    .scan("book")
    .scan("author")
    .join(JoinRelType.LEFT, "id")
    .filter(
      relBuilder.call(SqlStdOperatorTable.GREATER_THAN,
      relBuilder.field("publish_year"),
      relBuilder.literal(1830)))
    .project(
      relBuilder.field("id"),
      relBuilder.field("title"),
      relBuilder.field("publish_year"),
      relBuilder.field("lname"),
      relBuilder.field("fname"))
    .sortLimit(0, 5, relBuilder.field("id"))
    .build()

  println(RelOptUtil.toString(node))
```

可见跟 SQL 是类似的，省了解析 AST 的部分，效果上跟我们的 SQL 也类似，只是 JOIN 条件没有看懂应该如何指定不同的字段名。感兴趣的可以看下源码里`RelBuilderExample`这个类。

## 参考资料

1. [Calcite tutorial at BOSS 2021](https://www.youtube.com/watch?v=meI0W12f_nw)
2. [春蔚专访--MaxCompute 与 Calcite 的技术和故事](https://developer.aliyun.com/article/710764)
3. [Algebra](https://calcite.apache.org/docs/algebra.html)