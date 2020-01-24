---
title: "scala 30分钟极简入门"
date: 2019-05-11 18:12:16
tags: [scala]
---

本文介绍下 scala 的入门知识，主要面向对象是像我这样没有任何 Java 经验的 C++ 程序员。

## 1. Why?

我的初衷就是想深入学习函数式编程的思想、可以写 Spark.

跟最开始拿 go 写[raft](https://github.com/yingshin/Distributed-Systems/tree/master/6.824/src)一样，还是抱着了解基础，随用随学的原则，因此这篇笔记会不断的更新。但是仍然保持极简的风格，希望整体能控制在 30 分钟以内。

## 2. 环境

需要下载 Java 及 scala 的安装包，网上教程很多，不再赘述，具体见参考资料1。

### 2.1 REPL

执行`scala`或者`sbt console`都可以到 scala 的交互界面(Read-Eval-Print Loop):

```scala
scala> println("Hello World" substring(0, 3) toUpperCase() indexOf "E")
1
```

Eclipse 工程里新建 Scala Worksheet 的交互功能更完善一些。

### 2.2 scala 文件

新建文件 Hello.scala

```scala
object Hello extends App {
  println("Hello, " + args(0) + "!")
}
```

执行以下命令:

```
$ scalac Hello.scala
$ scala Hello izualzhy
Hello, izualzhy!
```

或者带`main`的文件也可以，例如另外新建一个文件 Hi.scala

```scala
object Hi {
  def main(args: Array[String]) {
    println("Hi, " + args(0) + "!")
  }
}
```

在 Eclipse 里新建工程和 scala 文件的方式，效率更高。

因此，环境上推荐 Eclipse 的 IDE 来学习 scala。

## 3. 基础类型

跟 C++ 类似，有以下几种基础类型：

```
Byte Short Int Long Float Double Char String Boolean
```

你可以直接通过`println`打印:

```scala
println(1, " I am a String", true, 2.3)         //> (1, I am a String,true,2.3)
```

另外，特殊一点的:

`Unit`类似于 void，在函数返回值时使用。

`Null Nothing Any AnyRef`待补充

## 4. 定义变量

scala 支持自动推导，例如我们可以直接指定变量的值而不用指定类型：

```scala
// var VariableName : DataType [=  Initial Value]
scala> val i = 1
i: Int = 1

scala> val f = 1.1
f: Double = 1.1

scala> val s = "Welcome to scala's world."
s: String = Welcome to scala's world.
```

当然也可以手动指定：

```scala
scala> val i:Double = 1
i: Double = 1.0
```

其中 val 表示不可变变量，初期可以理解为 C++ 的 const，定义后不可以修改，如果尝试修改则会报错：

```scala
scala> i = 2
<console>:12: error: reassignment to val
```

定义可变变量使用 var，支持修改。

## 5. if else

```
if (expr) {
   s1
} else {
   s2
}
```

例如:

```
  val i = 1                                 //> i  : Int = 1
  if (i == 1) {
      println("i is 1")
  } else {
      println("i is not 1")
  }                                         //> i is 1
```

当然其实可以写的更简洁一些：

```scala
  val i = 1                                       //> i  : Int = 1
  if (i == 1) println("i is 1") else println("i is not 1")
                                                  //> i is 1
```

也就是这种形式：

```scala
if (expr) s1 else s2
```

## 6. for && while

```scala
for( var x <- Range ) {
    statement(s);
}

while(condition) {
    statement(s);
}
```

例如，我们定义一个列表，然后通过 for 遍历：

```
val l = List("Baidu", "Google", "Facebook")     //> l  : List[String] = List(Baidu, Google, Facebook)
for (
    s <- l
) println(s)                                    //> Baidu
                                                  //| Google
                                                  //| Facebook
```

也可以在 for 里加入 if 判断

```
for (
    s <- l
    if (s.length > 5)
) println(s)                                    //> Google
                                                  //| Facebook
```

如果要打印[1, 10]所有的偶数:

```
for (s <- 1 to 10 if (s % 2 ==0)) println(s)
                                                //> 2
                                                //| 4
                                                //| 6
                                                //| 8
                                                //| 10
```

也可以使用`yield`返回一个新的 List:

```
    val l = List("Baidu", "Google", "Facebook")     //> l  : List[String] = List(Baidu, Google, Facebook)

    var result_for = for {
        s <- l
        s1 = s.toUpperCase()
        if (s1 != "")
    } yield(s1)  //> result_for  : List[String] = List(BAIDU, GOOGLE, FACEBOOK)
```

## 7. 函数

### 7.1. 基本概念

函数式编程里一直在说函数是一等公民，主要体现在支持：

1. 把函数作为实参传递给另一个函数  
2. 把函数作为返回值  
3. 把函数赋值给变量  
4. 把函数存储在数据结构里  

其实 C++ 大部分也可以支持，但是使用的方便性上，跟函数式编程语言原生支持还是差了很多。

函数的定义方式:

```scala
def functionName(param: ParamType): ReturnType = {
    //function body: expressions
}
```

例如：

```scala
def Hello(name: String): String = {
    s"My name is ${name}"
}                        //> Hello: (name: String)String
Hello("izualzhy")        //> res0: String = My name is izualzhy
```

当然，也可以直接省略大括号

```scala
def newHello(name: String): String = s"My name is ${name}"
                        //> newHello: (name: String)String
newHello("izualzhy")    //> res1: String = My name is izualzhy
```

### 7.2. 匿名函数

`Hello`这个函数，也可以使用更简洁的方式定义在一行，这种形式称为匿名函数:

```scala
// Anonymous Function:(形参列表) => {函数体}
def simpleHello = (name: String) => s"My name is ${name}"
                         //> simpleHello: => String => String
simpleHello("izualzhy")  //> res1: String = My name is izualzhy
```

匿名函数除了定义更加简洁，还可以赋值给其他变量：

```scala
val func = simpleHello
```

感觉叫匿名函数不符合 C++ 的习惯(anonymous namespace)

### 7.3. Evaluation Strategy

scala 解析函数参数时有两种方式：

1. Call By Value：形如`def foo(x: Int) = x`，x 只在函数入口时计算一次。  
2. Call By Name：形如`def foo(x: => Int) = x`，x 在函数每次具体使用时计算。  

其中 1 符合我们平时的理解，因此介绍下 2:

```scala
def now_time() = System.nanoTime          //> now_time: ()Long
now_time()                                //> res5: Long = 289060277656521

def foo(t: => Long) = {
    println("first:", t)
    println("second:", t)
}                                         //> foo: (t: => Long)Unit
foo(now_time())                           //> (first:,289060278984412)
                                          //| (second:,289060279131130)
```

可以看到在同一个函数内部，t 的结果发生了变化，这就是` => `产生的神奇效果。

### 7.4. 柯里化函数(Curried Function)

柯里化的作用就是将一个多参数的函数转为单参数的多个函数定义，例如：

```scala
def curried_add(x:Int)(y:Int) = x + y     //> curried_add: (x: Int)(y: Int)Int
curried_add(100)(11)                      //> res11: Int = 111
val add_100 = curried_add(100)_           //> add_100  : Int => Int = test$$$Lambda$30/559670971@4439f31e
add_100(11)                               //> res12: Int = 111
```

类似于 C++ 的偏特化？

### 7.5. 高阶函数

用函数作为形参或者返回值的函数，称为高阶函数

```scala
def two_elements(f: (Int, Int) => Int) {
    println(f(4, 4))
}                                 //> two_elements: (f: (Int, Int) => Int)Unit
two_elements(add)                 //> 8
```

### 7.6. 递归函数

```scala
def factorial(n: Int): Int = {
    if (n <= 0) 1
    else n * factorial(n - 1)
}                                 //> factorial: (n: Int)Int
println(factorial(10))            //> 3628800
```

## 8. 集合

类似 C++ stl 的容器：vector/list/map/set 等，当然要更加丰富一些。按照使用的场景，分为可变和不可变集合两种。分别位于`scala.collection.mutable` `scala.collection.immutable`，默认是`immutable`的，因此仅介绍下 immutable 的集合。

### 8.1. immutable

#### 8.1.1. List

定义或者修改一个链表

```scala
//自动推导为T=Int
scala> val a = List(1, 2, 3, 4)
a: List[Int] = List(1, 2, 3, 4)

//::连接操作符
scala> val b = 0 :: a
b: List[Int] = List(0, 1, 2, 3, 4)
scala> val c = "x"::"y"::"z"::Nil
c: List[String] = List(x, y, z)

//连接时将List作为一个元素，还是连接List内的元素
scala> a::c
res0: List[Object] = List(List(1, 2, 3, 4), x, y, z)
scala> a:::c
res1: List[Any] = List(1, 2, 3, 4, x, y, z)
```

`List`基础接口有`head tail isEmpty`，`head last`分别返回第一个和最后一个元素，`tail`返回除了`head`外所有的元素，这个设计有利于递归的实现。

例如：

```scala
def walk_through(l: List[Int]): String = {
    if (l.isEmpty) ""
    else l.head.toString + " " + walk_through(l.tail)
    }                        //> walk_through: (l: List[Int])String
val a = List(1, 2, 3)        //> a  : List[Int] = List(1, 2, 3)
walk_through(a)              //> res14: String = "1 2 3 "
```

此外还有各类修改的接口：

```scala
// filter 函数返回值为true则保留元素，例如取奇数
scala> a.filter(x => x % 2 == 1)
res4: List[Int] = List(1, 3)
// _ 可以用来表示任意值，这里用于简化输入的函数形式
scala> a.filter(_ % 2 == 1)
res12: List[Int] = List(1, 3)

// String to List
scala> "99 Red Balloons".toList
res5: List[Char] = List(9, 9,  , R, e, d,  , B, a, l, l, o, o, n, s)

scala> res5.filter(x => Character.isDigit(x))
res6: List[Char] = List(9, 9)

// takeWhile 函数返回值为true时则停止，否则一直取元素
scala> res5.takeWhile(x => x != 'e')
res7: List[Char] = List(9, 9,  , R)

// map 对每个元素执行函数转化为新的元素值
scala> val c = List("x", "y", "z")
c: List[String] = List(x, y, z)
scala> c.map(_.toUpperCase)
res10: List[String] = List(X, Y, Z)

// filter map 混用
scala> a.filter(_ % 2 == 1).map(_ + 100)
res13: List[Int] = List(101, 103)

scala> val q = List(List(1, 2, 3), List(4, 5, 6))
q: List[List[Int]] = List(List(1, 2, 3), List(4, 5, 6))
scala> q.map(_.filter(_ % 2 == 0))
res15: List[List[Int]] = List(List(2), List(4, 6))

// flatMap 会把两层转为一层 map
scala> q.flatMap(_.filter(_ % 2 == 0))
res18: List[Int] = List(2, 4, 6)

// reduceLeft 用于规约所有元素到一个值
scala> val a = List(1, 2, 3)
a: List[Int] = List(1, 2, 3)
scala> a.reduceLeft((x, y) => x + y)
res0: Int = 6
scala> a.reduceLeft(_ + _)
res1: Int = 6

// foldLeft 与 reduceLeft 类似，支持传入初始值
scala> a.foldLeft(0)(_ + _)
res3: Int = 6
scala> a.foldLeft(2)(_ + _)
res4: Int = 8
```

#### 8.1.2. Range

Range 在前面介绍 for 循环时已经提过：

```scala
scala> 1 to 10
res5: scala.collection.immutable.Range.Inclusive = Range 1 to 10

// 也可以通过 by 指定 step
scala> 1 to 10 by 2
res6: scala.collection.immutable.Range = inexact Range 1 to 10 by 2
scala> (1 to 10 by 2).toList
res6: List[Int] = List(1, 3, 5, 7, 9)

// toList转化为Llist类型
scala> (1 to 10).toList
res0: List[Int] = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

// until 与 to 的区别就是字面意思
scala> 1 until 10
res1: scala.collection.immutable.Range = Range 1 until 10
scala> (1 until 10).toList
res3: List[Int] = List(1, 2, 3, 4, 5, 6, 7, 8, 9)
```

#### 8.1.3. Stream

Stream is a lazy list.

```scala
scala> 1 #:: 2 #:: 3 #:: Stream.empty
res4: scala.collection.immutable.Stream[Int] = Stream(1, ?)
scala> val stream = (1 to 100000000).toStream
stream: scala.collection.immutable.Stream[Int] = Stream(1, ?)
scala> stream.head
res5: Int = 1

scala> stream.tail
res6: scala.collection.immutable.Stream[Int] = Stream(2, ?)
```

?表示一个 lazy 值，只有取值时才会计算。

#### 8.1.4. Tuple

Tuple 就是多个元素的集合

```scala
scala> (1, "Alice", "Math", 147)
res10: (Int, String, String, Int) = (1,Alice,Math,147)
```

`_下标值`用于取值：

```scala
scala> res10._3
res13: String = Math
```

两个元素又称为 pair，可以用于`Map`（接下来介绍）

```scala
scala> (1, 2)
res8: (Int, Int) = (1,2)
// 可以使用 -> 简写
scala> 1 -> 2
res9: (Int, Int) = (1,2)
```

结合`List`我们看个例子：

```scala
    val a = List(1, 2, 3)
    def sumSq(in: List[Int]): (Int, Int, Int) = {
        in.foldLeft((0, 0, 0))((t, v) => (t._1 + 1, t._2 + v, t._3 + v * v))
    }                           //> sumSq: (in: List[Int])(Int, Int, Int)
    sumSq(a)                    //> res13: (Int, Int, Int) = (3,6,14)
```

`foldLeft`的初始值传入一个 Tuple，包含 3 个元素，分别作为 元素个数，元素和，元素平方和的初始值。

#### 8.1.5. Map

Map[K, V]由多个 pair 组成

```scala
// 构造
scala> val p = Map(1 -> "David", 9 -> "Elwood")
p: scala.collection.immutable.Map[Int,String] = Map(1 -> David, 9 -> Elwood)
scala> val p2 = Map((1, "David"), (9, "Elwood"))
p2: scala.collection.immutable.Map[Int,String] = Map(1 -> David, 9 -> Elwood)

// 取value，注释不是[]而是()
scala> p(1)
res16: String = David
scala> p(9)
res17: String = Elwood

// 是否包含指定的key
scala> p.contains(1)
res18: Boolean = true
scala> p.contains(2)
res19: Boolean = false

// keys values
scala> p.keys
res20: Iterable[Int] = Set(1, 9)
scala> p.keys.foreach(key => println("key:" + key))
key:1
key:9
scala> p.values
res21: Iterable[String] = MapLike.DefaultValuesIterable(David, Elwood)

// 增加单个 pair
scala> p + (8 -> "Archer")
res22: scala.collection.immutable.Map[Int,String] = Map(1 -> David, 9 -> Elwood, 8 -> Archer)
// 删除 key
scala> p - 1
res23: scala.collection.immutable.Map[Int,String] = Map(9 -> Elwood)
// 增加多个 pair
scala> p ++ List(2 -> "Alice", 5 -> "Bob")
res24: scala.collection.immutable.Map[Int,String] = Map(1 -> David, 9 -> Elwood, 2 -> Alice, 5 -> Bob)
// 删除多个 key
scala> p -- List(1, 9, 2)
res25: scala.collection.immutable.Map[Int,String] = Map()
// 混合操作
scala> p ++ List(2 -> "Alice", 5 -> "Bob") -- List(1, 9, 2)
res26: scala.collection.immutable.Map[Int,String] = Map(5 -> Bob)
```

篇幅所限，仅介绍一些常用的接口，更多的可以参考 scala 官方文档，例如[List](https://www.scala-lang.org/api/current/scala/collection/immutable/List.html)

## 9. 例子

快排：

```scala
def qSort(in: List[Int]): List[Int] =
    if (in.length < 2) in
    else
        qSort(in.filter(_ < in.head)) ++ in.filter(_ == in.head) ++ qSort(in.filter(_ > in.head))
    //> qSort: (in: List[Int])List[Int]
qSort(List(5, 2, 9, 8, 0, 1, 3, 6, 7, 4))
    //> res16: List[Int] = List(0, 1, 2, 3, 4, 5, 6, 7, 8, 9)
```

是不是很简洁？

## 参考资料

1. [慕课网：Scala程序设计—基础篇
](https://www.imooc.com/learn/613)  
2. [菜鸟教程：Scala教程](Scala 教程
)  
