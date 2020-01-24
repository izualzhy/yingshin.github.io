---
title: "《Functional Programming Principles in Scala》"
date: 2019-08-01 16:43:33
tags: [scala]
---

最近借着几个周末在 Coursera 上完成了 [《Functional Programming Principles in Scala
》](https://www.coursera.org/learn/progfun1/home/welcome)，算是终于把之前立的上一门 Coursera 课程的 flag 给做到了。

## 1. 课程之外

这门课是 Scala 的作者 Martin Odersky, Professor 开的，相比看书，从中能看到更多原汁原味的设计思想，例如为什么会有 Call-By-Name 和 Call-By-Value、Curring 函数等。

学习这门课的时候我脑海里不断的浮现出这条曲线

![Dunning-Kruger Effect](/assets/images/Dunning_Kruger_Effect.jpeg)

感觉自己刚看了一个月 scala，就敢写[《scala 30分钟极简入门》](https://izualzhy.cn/scala-beginner)也是自信心爆棚了。

严格来讲，这门课程不算难，看过[《Scala 编程》](https://book.douban.com/subject/5377415/)这本书的话，估计会觉得更轻松一些。不过我之前看的是《Scala 实用指南》，系统知识不全，按照规定时间差不多刚好能做完。几个编程作业都是 100%，最近每个周六日都熬到凌晨三点多也算是值了。

Coursera 上相关课程很多，单独通关或者组队一块上一门课程，都是不错的选择。

## 2. 课程笔记

由于是笔记，记录的比较杂。课件及作业我上传到了[这里](https://github.com/yingshin/Distributed-Systems/tree/master/FunctionalProgrammingPrinciplesInScala)

### 2.1. Week1:Getting Started + Functions & Evaluation


+ Call-By-Name Call-By-Value 会带来计算次数的不同，但不是绝对的哪个更加节省计算。If CBV evaluation of an expression e terminates, then CBN evaluation of an expression e terminates, too.The other direction is not true.  
+ `def loop: Boolean = loop`, A defination `def x = loop` is OK, but a defination `val x = loop` will lead to an infinite loop.  
+ `def and(x: Boolean, y: Boolean) = if (x) y else false;def or(x: Boolean, y: Boolean) = if (!x) y else true`
+ @tailrec 可以帮我们验证是否是尾递归
+ Exercise 1: Pascal’s Triangle，两种方式，一种是`pascal(r, c) = pascal(r - 1, c - 1) + pascal(r, c - 1)`，还有一种是`pascal(r, c) = C(c, r) = factorial(c) / (factorial(r) * factorial(c -r))`。第一种好处是不容易越界，但是重复计算多，递归层数较多。第二种恰好相反，比如 `factorial(13)`就已经比 `INT32_MAX` 更大了，容易越界。  
+ Exercise 2: Parentheses Balancing，对()计数，类似于栈的操作  
+ Exercise 3: Counting Change：<https://stackoverflow.com/questions/12629721/coin-change-algorithm-in-scala-using-recursion>，自己最开始只按照coins有序来写的代码，没有通过，思路是一致的。

### 2.2. Week2:Higher Order Functions

+ Functions that take other functions as parameters or that return functions as results is called higher order functions.  
+ Currying: `def f(args1)...(argsn−1)(argsn) = E` is equivalent to `def f = (args1 ⇒ (args2 ⇒ ...(argsn ⇒ E)…))`  
+ 构造函数里可以调用 require，require is a predefined function: `require(y != 0, ”denominator must be nonzero”)`  

### 2.3. Week3:Data and Abstraction

+ 如果 class 有未定义的 method，那么必须在class前增加 abstract  
+ Any => the base type of all types  
+ AnyRef => the base type of all reference types, Alias of 'java.lang.object'  
+ AnyVal => the base type of all primitive types.  
+ Nothing is at the bottom of Scala’s type hierarchy.It’s a subtype of every other type.There is no value of type Nothing.  Null is a subtype of every class that inherits from Object;it’s incompatible with subtypes of AnyVal.  
+ `if (true) 1 else false` 的返回值是 AnyVal

### 2.4. Week4:Types and Pattern Matching

+ `S <: T` means: S is a subtype of T.`S >: T` means: S is a supertype of T, or T is a subtype of S.  

```
Say C[T] is a parameterized type and A, B are types such that A <: B.
In general, there are three possible relationships between C[A] and C[B]
C[A] <: C[B] => C is convariant
C[A] >: C[B] => C is contravariant
neither C[A] nor C[B] is a subtype of the other => C is nonvariant
```

### 2.5. Week5:Lists

+ Difference between FoldLeft and FoldRight: For operators that are associaive and commutative, foldLeft and foldRight are equivalent(event through there may be a difference in efficiency).  

### 2.6. Week6:Collections

```
//N-Queens
def isSafe(col: Int, queens: List[Int]): Boolean = {
  val row = queens.length
  val queensWithRow =
    (row - 1 to 0 by -1) zip queens
  //println(s"col:$col queensWithRow:${queensWithRow}")
  queensWithRow forall {
    case (r, c) => col != c && math.abs(col - c) != row - r
  }
}
def nqueens(n: Int): Set[List[Int]] = {
  def placeQueens(k: Int): Set[List[Int]] = {
    if (k == 0) Set(List())
    else {
      for {
        queens <- placeQueens(k - 1)
        col <- 0 until n
        if (isSafe(col, queens))
      } yield col :: queens
    }
  }
  placeQueens(n)
}

```

## 3. 杂想

编程语言其实是个挺神奇的东西，可以基础到某些设计思想或者趋势，比如越来越多的语言把类型放到了变量名后面，[Types are moving to the right](https://medium.com/@elizarov/types-are-moving-to-the-right-22c0ef31dd4a)。

函数式编程语言特别像大学里 函数空间 这个概念，大学时最开始学习实数、复数空间时感觉难度不大，到了函数空间，复杂度就一下变高了，天书一样根本听不懂。从对象编程、过程式编程到函数式编程的过程，跟这个非常像，函数变成了一等公民，基于函数的各种组合，我们可以构造出更简洁、高效的模块。

函数式编程给我的最大的感觉是注重函数以及不变量，由于没有可变变量，并发变得比较容易。这门课整体来讲属于入门课程，作者如果开一门 Scala 语言设计思想的课，我应该会第一时间去听吧。

万事开头难，找到知乎的一篇[总结帖](https://www.zhihu.com/question/22436320/answer/32665792)，准备用业务时间刷下一节课，欢迎组团。

课程里一些简洁同时非常有用的链接：

1. <https://www.coursera.org/learn/progfun1/supplement/Sauv3/cheat-sheet>
2. <https://www.coursera.org/learn/progfun1/supplement/D9pm0/learning-resources>
3. <https://www.coursera.org/learn/progfun1/supplement/KPiGt/scala-style-guide>
4. <https://www.scala-lang.org/api/current/index.html>
5. <http://twitter.github.io/scala_school/>
