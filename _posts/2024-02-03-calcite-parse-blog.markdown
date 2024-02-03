---
title: "Calcite笔记之6：使用 SQL 分析博客文章"
date: 2024-02-03 04:04:12
tags: calcite
---

博客里的文章，命名和规则都比较有规律。现在工作里 SQL 接触非常多，今天突然在想用 SQL 分析一把博客文章，介绍下 calcite .

## 1. 文章数据结构化

文章结构化的数据，可以粗略提炼出以下结构：
1. 标题：从文章内容提取 title 字段
2. 发布时间：从文章内容提取 date 字段
3. tags: 从文章内容提取 tags 字段
4. url: 从文件名提取

直接上代码：

```scala
case class PostMeta(url: String, title: String, date: String, tags: String) {
  def toArray: Array[AnyRef] = Array(url, title, date, tags)
}

object BlogPostMetaExtractor {

  def extractMetaInformation(filePath: String): Seq[PostMeta] = {
    val dir = Paths.get(filePath)
    if (!Files.isDirectory(dir)) {
      Seq.empty
    } else {
      val paths = Files.walk(dir).iterator().asScala.toSeq

      paths.filter{i => {
          Files.isRegularFile(i) && i.getFileName.toString.endsWith(".markdown")
        }}
        .flatMap { path =>
          try {
            val fileName = path.getFileName.toString
            val url = fileName.substring(11, fileName.length - ".markdown".length)

            val source = Source.fromFile(path.toFile)

            val lines = source.getLines().dropWhile(_ != "---").drop(1).takeWhile(_ != "---") // Drop the first "---" and stop at the second "---"
            source.close()

            val title = lines.find(_.startsWith("title"))
              .map(i => i.substring(i.indexOf(":") + 1).trim.stripPrefix("\"").stripSuffix("\""))
              .head

            val date = lines.find(_.startsWith("date"))
              .map(i => i.substring(i.indexOf(":") + 1).trim.stripPrefix("\"").stripSuffix("\""))
              .head

            val tags = lines.find(_.startsWith("tags:"))
              .map(i => i.substring(i.indexOf(":") + 1).trim.stripPrefix("\"").stripSuffix("\""))
              .head

            Some(PostMeta(url, title, date, tags))
          } catch {
            case e: Exception => {
              println(s"error with path:${path} e:${e}")
              throw e
            }
          }
        }
    }
  }
}
```

## 2. calcite 实现

参考之前的例子[Calcite笔记之1：Tutorial](https://izualzhy.cn/calcite-tutorial)，也是官网的 CsvTable，整体结构上需要定义：

1. BlogSchemaFactory: Schema 工厂类，构造对应的 Schema
2. BlogSchema: Schema 类，定义了表的结构，以及表的字段名和类型
3. BlogTable: Table 类，定义了表的查询逻辑，以及表的查询方法
4. BlogEnumerator: 实际查询逻辑，定义了如何读取数据，以及如何将数据转换为 Row

这里我们定义一个简单的支持遍历的表：

```scala
class BlogTable(blogPostPath: String) extends BlogAbstractTable with ScannableTable {
  override def scan(root: DataContext): Enumerable[Array[AnyRef]] = {
    new AbstractEnumerable[Array[AnyRef]] {
      override def enumerator(): Enumerator[Array[AnyRef]] = {
        new BlogEnumerator(blogPostPath)
      }
    }
  }
}
```

实际遍历是通过`BlogEnumerator`:

```scala
class BlogEnumerator(blogPostPath: String) extends Enumerator[Array[AnyRef]] {
  private val blogPostMetaList: Seq[PostMeta] = BlogPostMetaExtractor.extractMetaInformation(blogPostPath)
  private var currentIndex: Int = -1 // 使用索引来跟踪当前元素，初始值为 -1

  override def current(): Array[AnyRef] = {
    if (currentIndex >= 0 && currentIndex < blogPostMetaList.size) {
      blogPostMetaList(currentIndex).toArray
    } else {
      throw new NoSuchElementException("No current element")
    }
  }

  override def moveNext(): Boolean = {
    if (currentIndex + 1 < blogPostMetaList.size) {
      currentIndex += 1
      true
    } else {
      false
    }
  }

  override def reset(): Unit = {
    currentIndex = -1 // 将索引重置为 -1，从而实现重置遍历的效果
  }

  override def close(): Unit = {
    // 在这里实现资源释放逻辑，如果有的话
  }
}
```

通过`SchemaPlus`注册函数，例如：

```scala
parentSchema.add("BLOG_SUBSTR", ScalarFunctionImpl.create(classOf[BlogScalarFunction], "blogSubstr"))
```

完整代码就不贴了，放在了[Bigdata-Systems/calcite](https://github.com/izualzhy/Bigdata-Systems/tree/main/calcite)

## 3. sqlline 验证

yaml 文件指定 SchemaFactory:

```yaml
version: 1.0
defaultSchema: BLOG
schemas:
  - name: BLOG
    type: custom
    factory:  cn.izualzhy.blog.BlogSchemaFactory
    operand:
      directory: sales
```

通过 sqlline 连接：

```bash
➜  calcite git:(main) ✗ ./src/sqlline                                                                              (6518s)[14:43:14] 
sqlline version 1.12.0
sqlline> !connect jdbc:calcite:model=src/main/resources/blog.yaml admin admin
BLOG_SUBSTR
Transaction isolation level TRANSACTION_REPEATABLE_READ is not supported. Default (TRANSACTION_NONE) will be used instead.
```

查看表：

```bash
0: jdbc:calcite:model=src/main/resources/blog> !tables
+-----------+-------------+------------+--------------+---------+----------+------------+-----------+---------------------------+----+
| TABLE_CAT | TABLE_SCHEM | TABLE_NAME |  TABLE_TYPE  | REMARKS | TYPE_CAT | TYPE_SCHEM | TYPE_NAME | SELF_REFERENCING_COL_NAME | RE |
+-----------+-------------+------------+--------------+---------+----------+------------+-----------+---------------------------+----+
|           | BLOG        | BLOG       | TABLE        |         |          |            |           |                           |    |
|           | metadata    | COLUMNS    | SYSTEM TABLE |         |          |            |           |                           |    |
|           | metadata    | TABLES     | SYSTEM TABLE |         |          |            |           |                           |    |
+-----------+-------------+------------+--------------+---------+----------+------------+-----------+---------------------------+----+
```

统计 tag，年份:

```bash
0: jdbc:calcite:model=src/main/resources/blog> select tags, count(1) from BLOG group by tags;
+-------------+--------+
|    TAGS     | EXPR$1 |
+-------------+--------+
| Patronum    | 13     |
| protobuf    | 11     |
...

0: jdbc:calcite:model=src/main/resources/blog> select blog_substr(pub_date, 0, 4), count(1) from blog group by blog_substr(pub_date, 0, 4);
+--------+--------+
| EXPR$0 | EXPR$1 |
+--------+--------+
| 2019   | 42     |
| 2018   | 28     |
| 2017   | 19     |
| 2016   | 30     |
| 2015   | 14     |
| 2014   | 23     |
| 2024   | 5      |
| 2023   | 22     |
| 2022   | 27     |
| 2020   | 11     |
+--------+--------+
10 rows selected (0.466 seconds)
```


