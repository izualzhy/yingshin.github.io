---
title: "《Scala实用指南》读书笔记五：递归和惰性求值"
date: 2019-06-30 11:10:00
tags: scala
---

## 1. 递归

使用解决子问题的方案解决一个问题，也就是递归，这种想法十分诱人。许多算法和问题本质上都是递归的。一旦我们找到窍门，使用递归来设计解决方案就变得极富表现力且直观。

一般来说，递归最大的问题是大规模的输入值会造成栈溢出。但幸运的是，在Scala中可以使用特殊构造的递归来规避这个问题。在本章中，我们将分别探讨强大的尾调用优化（tail call optimization）技术以及Scala标准库中的相关支持类。使用这些易于访问的工具，就可以在高度递归的算法实现中既可以处理大规模的输入值又能同时规避栈溢出（即触发StackOverflowError）的风险。

### 1.1. factorial

```scala
  //factorial
  def factorial(number: Int): BigInt = {
      if (number == 0) 1
      else number * factorial(number - 1)
  }                                               //> factorial: (number: Int)BigInt
  factorial(5)                                    //> res0: BigInt = 120
```

### 1.2. 尾调用优化(Tail Call Optimization)

我们故意制造一个异常，来观察`mad(5)`的调用，可以看到函数一共调用了 6 次

```scala
  def mad(parameter: Int): Int = {
      if (parameter == 0) throw new RuntimeException("Error")
      else 1 * mad(parameter - 1)
  }                                               //> mad: (parameter: Int)Int
  mad(5)                                          //> java.lang.RuntimeException: Error
                                                  //|   at Recursions$.mad$1(Recursions.scala:12)
                                                  //|   at Recursions$.mad$1(Recursions.scala:13)
                                                  //|   at Recursions$.mad$1(Recursions.scala:13)
                                                  //|   at Recursions$.mad$1(Recursions.scala:13)
                                                  //|   at Recursions$.mad$1(Recursions.scala:13)
                                                  //|   at Recursions$.mad$1(Recursions.scala:13)
                                                  //|   at Recursions$.$anonfun$main$1(Recursions.scala:16)
                                                  //|   at org.scalaide.worksheet.runtime.library.WorksheetSupport$.$anonfun$$ex
                                                  //| ecute$1(WorksheetSupport.scala:76)
                                                  //|   at org.scalaide.worksheet.runtime.library.WorksheetSupport$.redirected(W
                                                  //| orksheetSupport.scala:65)
                                                  //|   at org.scalaide.worksheet.runtime.library.WorksheetSupport$.$execute(Wor
                                                  //| ksheetSupport.scala:76)
                                                  //|   at Recursions$.main(Recursions.scala:1)
                                                  //|   at Recursions.main(Recursions.scala)
```

代码仅做一个小小的改动，去除多余的乘1操作。这将使调用在尾部递归--对函数的调用在最后，即在尾部。然而，修改版的栈跟踪表明，在抛出异常时，深度只有1层，而不是6层。这是因为 Scala 的尾递归优化做了一些改善工作。

```scala
  def mad(parameter: Int): Int = {
      if (parameter == 0) throw new RuntimeException("Error")
      // else 1 * mad(parameter - 1)
      else mad(parameter - 1)
  }                                               //> mad: (parameter: Int)Int
  mad(5)                                          //> java.lang.RuntimeException: Error
                                                  //|   at Recursions$.mad$1(Recursions.scala:11)
                                                  //|   at Recursions$.$anonfun$main$1(Recursions.scala:15)
                                                  //|   at org.scalaide.worksheet.runtime.library.WorksheetSupport$.$anonfun$$ex
                                                  //| ecute$1(WorksheetSupport.scala:76)
                                                  //|   at org.scalaide.worksheet.runtime.library.WorksheetSupport$.redirected(W
                                                  //| orksheetSupport.scala:65)
                                                  //|   at org.scalaide.worksheet.runtime.library.WorksheetSupport$.$execute(Wor
                                                  //| ksheetSupport.scala:76)
                                                  //|   at Recursions$.main(Recursions.scala:1)
                                                  //|   at Recursions.main(Recursions.scala)
```

编译器会将尾递归自动转化成迭代。这种隐性优化非常好，但也让人略感不安——没有直接可见的反馈可供辨别。为了推断是否是尾递归，我们需要检查字节码或者检查大的输入值是否会导致代码运行失败。这样做太麻烦了。

Scala 提供了一个 tailrec 的注解，在编译时检查函数是否是尾递归的。如果不是，那么函数不能被优化，编译器会严格地报错。

```scala
// <console>:14: error: could not optimize @tailrec annotated method g: it contains a recursive call not in tail position else number * g(number - 1)
@scala.annotation.tailrec￼ 
def factorial(number: Int): BigInt = {￼ 
  if (number == 0)￼  1￼ 
  else￼  number * factorial(number -1)￼ 
}
println(factorial(10000))
```

由于`factorial`实现上不是尾递归的，因此编译报错。直到我们修改为这个样子，编译器才会表示同意：

```scala
  import scala.annotation.tailrec
  @tailrec
  def factorial_tco(fact: BigInt, number: Int): BigInt = {
    if (number == 0) fact
    else factorial_tco(fact * number, number - 1)
  }
  factorial_tco(1L, 1000)
```

## 2. 惰性求值

许多编程语言都支持条件的短路求值￼（short-circuit evaluation）。在具有多个&&或者||符号的条件表达式中，如果某个参数的求值结果就足以确定整个表达式的值，那么表达式中剩下的参数都不会被求值。

例如:

```scala
  def expensiveComputation() = {
      println("...assume slow operator...")
      false
  }                                               //> expensiveComputation: ()Boolean

  def evaluate(input: Int): Unit = {
      println(s"evaluate called with $input")

      if (input >= 10 && expensiveComputation())
        println("doing work...")
      else
        println("skipping")
  }                                               //> evaluate: (input: Int)Unit

  evaluate(0)                                     //> evaluate called with 0
                                                  //| skipping
  evaluate(100)                                   //> evaluate called with 100
                                                  //| ...assume slow operator...
                                                  //| skipping
```

如果参数值小于 10，那么 evaluate() 方法中的 expensiveComputation() 方法将不会被执行。

然而，这依赖于我们预先的判断和对代码逻辑的设计，如果先调用 expensiveComputation，将结果存储在 perform 变量，那么跟参数无关，expensiveComputation 一定会执行:

```scala
      val perform = expensiveComputation()
      // lazy 推迟 perform 的计算
      // lazy val perform = expensiveComputation()
      if (input >= 10 && perform)
        println("doing work...")
      else
        println("skipping")
```

如果想要达到最开始的效果，就是把 perform 标记为 lazy 的，即代码L3的写法。

lazy 关键字的作用，是告诉Scala编译器推迟绑定变量和它的值，直到该值被使用时为止。如果我们从未使用过该值，那么该变量将不会被绑定，因此，也永远不会对生成该值的函数求值。

### 2.1. 惰性的困境

可以将任何变量标记为 lazy，这样对该变量值的绑定将会推迟到他首次被使用时。这样看起来很美好，但是却有副作用。因为会改变多个值的求值顺序。例如：

```scala
  // lazy_test.scala
  // 调用 read() 函数所读取到的值，将以随机顺序绑定到这两个变量。
  // 在这种情况下结果就是，不可交换(non-commutative)计算将变得不可预知。
  // 输入 1 2 结果不确定，-1 or 1
  import scala.io._
  def read = StdIn.readInt()
  
  lazy val first = read
  lazy val second = read
  
  if (Math.random() < 0.5)
    second
  println(first - second)
```

### 2.2. 释放严格集合的惰性

假定我们有一个 list,

```scala
  val people = List(
    ("Mark", 32),
    ("Bob", 22),
    ("Jane", 8),
    ("Jill", 21),
    ("Nick", 50),
    ("Nancy", 42),
    ("Mike", 19),
    ("Sara", 12),
    ("Paula", 42),
    ("John", 21))
```

分别定义两个过滤方法，isOlderThan17 和 isNameStartsWithJ:

```scala
  def isOlderThan17(person: (String, Int)) = {
    println(s"isOlderThan17 called for $person")
    val (_, age) = person
    age > 17
  }

  def isNameStartsWithJ(person: (String, Int)) = {
    println(s"isNameStartsWithJ called for $person")
    val (name, _) = person
    name.startsWith("J")
  }
```

注：*people.filter(isOlderThan17) 作用和 people.filter(_._2 > 17) 是一样的*

我们想找到满足这两个条件的第一个人，可以这么写：

```scala
  // 18次调用
  println(people.filter {isOlderThan17}.filter {isNameStartsWithJ}.head)
```

执行时跟我们预料的一致，先找到所有满足`isOlderThan17`的列表，再在其中找到所有满足`isNameStartsWithJ`的列表，取出列表 header.

通过使用严格集合的惰性视图 view，我们看下效果：

```scala
  // 7次调用
  println(people.view.filter {isOlderThan17}.filter {isNameStartsWithJ}.head)
```

检查元素的操作并没有立即执行，也没有按照严格顺序执行，调用次数节省到了 7 次。

但是，同 lazy 关键字标记一样，惰性求值并不总是高效的。

```scala
  // 依旧18次调用
  println(people.filter {isOlderThan17}.filter {isNameStartsWithJ}.last)
  // 25次调用，惰性求值比严格求值调用次数更多
  println(people.view.filter {isOlderThan17}.filter {isNameStartsWithJ}.last)
```

问题的性质和算法对于是否能够从惰性求值中得到效率提升有很大的影响。花点儿时间去试验和测试惰性求值，以确保结果是正确的，而且执行也是高效的。

### 2.3. 终极惰性流

流代表了无限的序列，远远不断的产出新的值。Scala 使用 Stream 来表示流，按需产生值。例如：

```scala
  def generate(starting: Int): Stream[Int] = {
    starting #:: generate(starting + 1)
  }                                         //> generate: (starting: Int)Stream[Int]
  println(generate(25))                     //> Stream(25, ?)
```

generate()函数接受一个整数starting作为它的参数，并返回一个Stream[Int]。它的实现使用了一个特殊的函数#:：来将starting变量的值和递归调用generate()函数的值连接起来。在概念上，Stream的#:：函数很像List的：：函数；它们都将连接或者将一个值前拼接到各自对应的集合或者流上。然而，Stream上的#:：函数是惰性的，它只会在需要的时候进行连接，并在最终结果被请求之前推迟执行。

`Stream(25, ?)`告诉我们，这是一个初始值为25的流，后面跟着一个尚未计算的值（流设计成天然是惰性的）。

使用`take`取出值：

```scala
  println(generate(25).take(10).force)      //> Stream(25, 26, 27, 28, 29, 30, 31, 32, 33, 34)
  println(generate(25).take(10).toList)     //> List(25, 26, 27, 28, 29, 30, 31, 32, 33, 34)
  println(generate(25).takeWhile {_ < 40}.force)
                                            //> Stream(25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39)
```

### 2.4. 从流中获取质数

这段代码用于计算质数：`primes`参数接收一个整数，如果这个整数为质数，则添加到流，否则该数+1，递归执行`primes`。

```scala
  def isPrime(number: Int) =
      number > 1 && ! (2 until number).exists(isDivisibleBy(number, _))
                                              //> isPrime: (number: Int)Boolean
    def primes(starting: Int): Stream[Int] = {
      println(s"computing for $starting")
      if (isPrime(starting))
        starting #:: primes(starting + 1)
      else
        primes(starting + 1)
    }                                         //> primes: (starting: Int)Stream[Int]
```

每计算一个数，都会打印`computing for $starting`，因此我们能够观察到流的惰性：是否对同一个数字计算。

```scala
    val primesFrom100 = primes(100)           //> computing for 100
                                                  //| computing for 101
                                                  //| primesFrom100  : Stream[Int] = Stream(101, ?)
    println(primesFrom100.take(3).toList)     //> computing for 102
                                                  //| computing for 103
                                                  //| computing for 104
                                                  //| computing for 105
                                                  //| computing for 106
                                                  //| computing for 107
                                                  //| List(101, 103, 107)
    println("Let's ask for more...")          //> Let's ask for more...
    println(primesFrom100.take(4).toList)     //> computing for 108
                                                  //| computing for 109
                                                  //| List(101, 103, 107, 109)
  println(primesFrom100.take(10).toList)    //> computing for 110
                                                  //| computing for 111
                                                  //| computing for 112
                                                  //| computing for 113
                                                  //| computing for 114
                                                  //| computing for 115
                                                  //| computing for 116
                                                  //| computing for 117
                                                  //| computing for 118
                                                  //| computing for 119
                                                  //| computing for 120
                                                  //| computing for 121
                                                  //| computing for 122
                                                  //| computing for 123
                                                  //| computing for 124
                                                  //| computing for 125
                                                  //| computing for 126
                                                  //| computing for 127
                                                  //| computing for 128
                                                  //| computing for 129
                                                  //| computing for 130
                                                  //| computing for 131
                                                  //| computing for 132
                                                  //| computing for 133
                                                  //| computing for 134
                                                  //| computing for 135
                                                  //| computing for 136
                                                  //| computing for 137
                                                  //| computing for 138
                                                  //| computing for 139
                                                  //| computing for 140
                                                  //| computing for 141
                                                  //| computing for 142
                                                  //| computing for 143
                                                  //| computing for 144
                                                  //| computing for 145
                                                  //| computing for 146
                                                  //| computing for 147
                                                  //| computing for 148
                                                  //| computing for 149
                                                  //| List(101, 103, 107, 109, 113, 127, 131, 137, 139, 149)
```

可以看到`take`执行了三次，每次返回 >= 100 的 N 个质数，但是没有重新开始计算。这是流的一个特性：

它们记住（memoize）它们已经生成的值。这其实也没什么，只不过是缓存（caching）而已，但是在我们的（计算机）领域，我们就喜欢给熟知的概念取奇怪的名字，以便看起来很有“深度”。￼当流按需产生了一个新值时，它将会在返回该值之前缓存它——我的意思是记住它。然后，如果再次请求相同的值，就没有必要进行重复计算了。
