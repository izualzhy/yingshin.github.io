---
title: "Calcite笔记之四：代码生成与编译"
date: 2022-05-14 10:33:20
tags: [Calcite]
---

## 1. Csv表

之前记录了 Calcite 的架构和简化后的代码流程，这篇笔记回归下最开始[Tutorial](https://izualzhy.cn/calcite-tutorial)笔记里的 Csv 表，仿照写了一个可以 Debug 的例子([TutorialTest.scala](https://github.com/izualzhy/BigData-Systems/blob/main/calcite/src/main/scala/cn/izualzhy/TutorialTest.scala)):   

```scala
  val csvPath = getClass.getClassLoader.getResource("sales_from_calcite_sources").getPath
//  val csvSchema = new CsvSchema(new File(csvPath), CsvTable.Flavor.SCANNABLE)
  val csvSchema = new CsvSchema(new File(csvPath), CsvTable.Flavor.TRANSLATABLE)
  val properties = new Properties()
  properties.setProperty("caseSensitive", "false")
  val connection = DriverManager.getConnection("jdbc:calcite:", properties)
  val calciteConnection = connection.unwrap(classOf[CalciteConnection])
  val rootSchema = calciteConnection.getRootSchema
  rootSchema.add("sales", csvSchema)

  query("SELECT empno, name, deptno FROM sales.emps WHERE deptno > 20")

  def query(sql: String): Unit = {
    println(s"****** $sql ******")
    val statement = calciteConnection.createStatement()
    val resultSet = statement.executeQuery(sql)
    dumpResultSet(resultSet)
  }
```

分别创建 SCANNABLE or TRANSLATABLE 的表，通过`executeQuery`执行 SQL，结果存储到`ResultSet`

源数据跟 Calcite [源码数据](https://github.com/apache/calcite/tree/main/example/csv/src/test/resources/sales)里一致，程序输出：

```
****** SELECT empno, name, deptno FROM sales.emps WHERE deptno > 20 ******
EMPNO:110 NAME:John DEPTNO:40
EMPNO:130 NAME:Alice DEPTNO:40
```

`executeQuery`的实现里，能够看到对应的一直到生成 RelRoot 的几个核心步骤：

![Statement.executeQuery](/assets/images/calcite/calcite/Statement.executeQuery.png)

不过结果查询出来存储到`ResultSet`，光生成到关系运算符的树不够，还需要代码生成和编译的环节。我在应用 FlinkSQL 的时候，遇到问题，第一时间还是想看到代码反查问题。

## 2. 编译: Janino

Calcite 使用 Janino<sup>[1]</sup> 编译 Java 代码，Janino 可以比较方便的执行一段 Java 的表达式、代码、或者获取编译后的符号。

基本流程都是类似的，根据不同需求初始化不同的 Evaluator，然后编译或者运行。看下常见的几个用法：

### 2.1. ExpressionEvaluator

```java
    public static void ExpressionEvaluator() {
        try {
            IExpressionEvaluator ee = CompilerFactoryFactory.getDefaultCompilerFactory().newExpressionEvaluator();
            ee.setExpressionType(int.class);
            ee.setParameters(
                    new String[] {"a", "b"},
                    new Class[] {int.class, int.class}
            );

            ee.cook("a + b");
            Integer res = (Integer) ee.evaluate(
                    new Object[] {10, 11}
            );
            System.out.println("res = " + res);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

执行表达式`a+b`，输出结果：`res = 21`

### 2.2. ScriptEvaluator

```java
    public static void ScriptEvaluator() {
        try {
            IScriptEvaluator se = CompilerFactoryFactory.getDefaultCompilerFactory().newScriptEvaluator();
            se.setReturnType(boolean.class);
            se.cook("static void method1() {\n"
                    + "    System.out.println(1);\n"
                    + "}\n"
                    + "\n"
                    + "method1();\n"
                    + "method2();\n"
                    + "\n"
                    + "static void method2() {\n"
                    + "    System.out.println(2);\n"
                    + "}\n"
                    + "return true;\n"
            );
            Object res = se.evaluate(new Object[0]);
            System.out.println("res = " + res);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

执行给定的代码字符串，输出结果：

```
1
2
res = true
```

### 2.3. ClassBodyEvaluator

```java
    public static void ClassBodyEvaluator() {
        try {
            IClassBodyEvaluator cbe = CompilerFactoryFactory.getDefaultCompilerFactory().newClassBodyEvaluator();
            String code = ""
                    + "public static void main(String[] args) {\n"
                    + "    System.out.println(java.util.Arrays.asList(args));\n"
                    + "}";
            cbe.cook(code);
            Class<?> c = cbe.getClazz();
            Method mainMethod = c.getMethod("main", String[].class);
            String[] args = {"Hello", "World", "Hello", "Janino"};
            mainMethod.invoke(null, (Object)args);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

提取代码片段里的`main`方法，执行后输出结果：`[Hello, World, Hello, Janino]`

### 2.4. SimpleCompilerEvaluator

```java
    public static void SimpleCompilerEvaluator() {
        try {
            ISimpleCompiler simpleCompiler = CompilerFactoryFactory.getDefaultCompilerFactory().newSimpleCompiler();
            String code = "" +
                    "public class Foo {\n" +
                    "    public static void main(String[] args) {\n" +
                    "        new Bar().meth(\"Foo\");\n" +
                    "    }\n" +
                    "}\n" +
                    "\n" +
                    "public class Bar {\n" +
                    "    public void meth(String who) {\n" +
                    "        System.out.println(\"Hello! \" + who);\n" +
                    "    }\n" +
                    "}";
            simpleCompiler.cook(code);
            Class<?> fooClass = simpleCompiler.getClassLoader().loadClass("Foo");
            fooClass.getDeclaredMethod("main", String[].class).invoke(null, (Object)new String[]{"hello", "world"});

            Class<?> barClass = simpleCompiler.getClassLoader().loadClass("Bar");
            barClass.getDeclaredMethod("meth", String.class).invoke(barClass.newInstance(), "Janino");

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

跟上一节类似，提取代码片段里的类及方法，输出结果：

```
Hello! Foo
Hello! Janino
```

## 3. 代码生成: Expressions

`Expressions`源码来自于 linq4j，是 Julian Hyde 的个人项目，用于生成代码。

我们写代码时，常量、变量、参数、循环、函数、类等，都可以通过`Expressions`的子类表示出来，然后生成为一段代码。

假定我们想要生成这么一段 Hello World 的待执行代码：

```java
public class MyHelloWorld implements java.io.Serializable {
  public static void hello(String who) {
    System.out.println("Hello World, " + who + "!");
  }

}
```

需要将代码块里的所有元素拆解为`Expressions`对应的子类，比如：

1. ConstantExpression: 例如字符串"Hello World, "、"!"
2. ParameterExpression: 例如`String who`
3. MemberExpression: 例如`System.out`
4. MethodCallExpression: 例如`println`
5. MethodDeclaration、ClassDeclaration等

完整的映射关系为：

![Expressions Example](/assets/images/calcite/calcite/Expressions-Example.png)

然后通过上一节的`SimpleCompiler`编译：

```scala
  val sc = CompilerFactoryFactory.getDefaultCompilerFactory.newSimpleCompiler
  sc.cook(helloWorldSrc)
  val helloWorldClass = sc.getClassLoader.loadClass(className)
  helloWorldClass.getMethod(funcName, classOf[String]).invoke(null,  "Expressions")
```

程序输出：

```
public class MyHelloWorld implements java.io.Serializable {
  public static void hello(String who) {
    System.out.println("Hello World, " + who + "!");
  }

}

Hello World, Expressions!
```

具体代码: [ExpressionsHelloWorld.scala](https://github.com/izualzhy/BigData-Systems/blob/main/calcite/src/main/scala/cn/izualzhy/ExpressionsHelloWorld.scala)，更多`Expressions`的用法可以参考`ExpressionTest`这个单测的实现。

## 4. Calcite 源码分析

如第一节里的处理流程，代码生成在`implement(root)`部分，入口类是`CalcitePrepareImpl`，核心实现在`EnumerableRelImplementor.implementRoot`.

该方法首先调用 root 节点的`implement`，由于是树的结构，接下来会循环调用子节点(input)的`implement`方法：

对应第一节的例子，一共有两个节点：

```
EnumerableCalc.ENUMERABLE.[](input=CsvTableScan#25,expr#0..2={inputs},expr#3=20,expr#4=>($t2, $t3),proj#0..2={exprs},$condition=$t4)
  CsvTableScan.ENUMERABLE.[](table=[sales, EMPS],fields=[0, 1, 2])
```

root 节点是`EnumerableCalc`，递归调用`CsvTableScan`的`implement`方法，也是用户需要自己实现的：

```java
public class CsvTableScan extends TableScan implements EnumerableRel {
  ...

  public Result implement(EnumerableRelImplementor implementor, Prefer pref) {
    PhysType physType =
        PhysTypeImpl.of(
            implementor.getTypeFactory(),
            getRowType(),
            pref.preferArray());

    if (table instanceof JsonTable) {
      return implementor.result(
          physType,
          Blocks.toBlock(
              Expressions.call(table.getExpression(JsonTable.class),
                  "enumerable")));
    }
    return implementor.result(
        physType,
        Blocks.toBlock(
            Expressions.call(table.getExpression(CsvTranslatableTable.class),
                "project", implementor.getRootExpression(),
                Expressions.constant(fields))));
  }

  ...
}
```

`Blocks.toBlock` 生成了这样一段代码：

```java
{
  return ((org.apache.calcite.adapter.csv.CsvTranslatableTable) root.getRootSchema().getSubSchema("sales").getTable("EMPS")).project(root, new int[] {
      0,
      1,
      2});
}

```

可以看到调用了`CsvTranslatable.project`方法，也是需要用户自定义实现的：

```java
  @SuppressWarnings("unused") // called from generated code
  public Enumerable<Object> project(final DataContext root,
      final int[] fields) {
    final AtomicBoolean cancelFlag = DataContext.Variable.CANCEL_FLAG.get(root);
    return new AbstractEnumerable<Object>() {
      public Enumerator<Object> enumerator() {
        return new CsvEnumerator<>(
            source,
            cancelFlag,
            getFieldTypes(root.getTypeFactory()),
            ImmutableIntList.of(fields));
      }
    };
  }
```

如注释所说，是在生成代码里调用的。接着会生成调用`current` `moveNext`遍历数据的代码。Debug 时，你可能会看到很多`Baz`的类名，也能够顺着找到出处。

以上步骤里有好几处都会自动生成代码，感兴趣的可以参考<sup>[2]</sup>的方式将生成代码打印出来。

`EnumerableInterpretable.getBindable`对应了编译部分，使用`IClassBodyEvaluator`，通过 Debug 第一节的例子完整的看遍执行流程效果会更好，不再赘述。

## 5. 参考资料

1. [Janino : A super-small, super-fast Java compiler](https://janino-compiler.github.io/janino/#janino-as-an-expression-evaluator)
2. [HOWTO](https://calcite.apache.org/docs/howto.html#debugging-generated-classes-in-intellij)
