---
title: "《Scala实用指南》读书笔记二：函数值和闭包与特质"
date: 2019-06-29 23:07:39
tags: [scala]
---

## 1. 函数值与闭包

在函数式编程中，函数是一等公民。函数可以作为参数值传入其他函数中，函数的返回值可以是函数，函数甚至可以嵌套函数。这些高阶函数在Scala中被称为函数值（function value）。闭包（closure）是函数值的特殊形式，会捕获或者绑定到在另一个作用域或上下文中定义的变量。

### 1.1. 函数

函数式编程来实现求解数组最大值/和的例子：

```scala
  val array = Array(2, 3, 5, 1, 6, 4)       //> array  : Array[Int] = Array(2, 3, 5, 1, 6, 4)
  array.foldLeft(0){ (sum, elem) => sum + elem }
                                                  //> res18: Int = 21
  array.foldLeft(0)( (sum, elem) => sum + elem )
                                                  //> res19: Int = 21
  array.foldLeft(Integer.MIN_VALUE){ (large, elem) => Math.max(large, elem) }
                                                  //> res20: Int = 6
  array.foldLeft(Integer.MIN_VALUE)( (large, elem) => Math.max(large, elem) )
                                                  //> res21: Int = 6
  // /: 替代 foldLeft
  (0 /: array){ (sum, elem) => sum + elem } //> res22: Int = 21
  (0 /: array)( (sum, elem) => sum + elem ) //> res23: Int = 21
  // _ 更加简化
  (0 /: array)( _ + _ )                     //> res24: Int = 21
  // 是否存在负数
  array.exists( _ < 0)                      //> res25: Boolean = false
```

### 1.2. 柯里化

编写一个带有多个参数列表，每个参数列表只有一个参数的方法，而不要编写一个带有一个参数列表，含有多个参数的方法；在每个参数列表中，也可以接受多个参数。也就是说，要写成这样`def foo(a: Int)(b: Int)(c:Int) {}`，而不是`def foo(a: Int, b: Int, c: Int) = {}`。你可以这样调用，如`foo(1)(2)(3)、foo(1){2}{3}`，甚至可以是`foo{1}{2}{3}`。

```scala
  def currying_example(a: Int)(b: Int)(c: Int) = {}
                                                  //> currying_example: (a: Int)(b: Int)(c: Int)Unit
  currying_example _                        //> res26: Int => (Int => (Int => Unit)) = example$$$Lambda$31/1690287238@64bf3bbf
```

我们专注于REPL中的信息。它展示了一系列（3次）转换。链路中的每一个函数都接收一个Int参数，并返回一个部分应用函数。然而最后一个是例外，它返回一个Unit。

### 1.3. 参数路由

同样一个功能，书写上越来越简洁：

```scala
  // 参数路由
  (Integer.MIN_VALUE /: array) { (carry, elem) => Math.max(carry, elem) }
                                                  //> res27: Int = 6
  (Integer.MIN_VALUE /: array) { Math.max(_, _) } //> res28: Int = 6
  (Integer.MIN_VALUE /: array) { Math.max _ }     //> res29: Int = 6
  (Integer.MIN_VALUE /: array) { Math.max }       //> res30: Int = 6
```

### 1.4. 部分应用函数

调用一个函数，实际上是在一些参数上应用这个函数。如果传递了所有期望的参数，就是对这个函数的完整应用，就能得到这次应用或者调用的结果。然而，如果传递的参数比所要求的参数少，就会得到另外一个函数。这个函数被称为部分应用函数。

```scala
  // 部分应用函数
  import java.util.Date
  def log(date: Date, message: String): Unit = {
    println(s"$date --- $message")
  }                                         //> log: (date: java.util.Date, message: String)Unit

  val date = new Date(1558781026000L)       //> date  : java.util.Date = Sat May 25 18:43:46 CST 2019
  log(date, "message-1")                    //> Sat May 25 18:43:46 CST 2019 --- message-1
  log(date, "message-2")                    //> Sat May 25 18:43:46 CST 2019 --- message-2
  log(date, "message-3")                    //> Sat May 25 18:43:46 CST 2019 --- message-3

  val logWithDateBound = log(date, _: String)
                                                  //> logWithDateBound  : String => Unit = example$$$Lambda$36/1635756693@1e12798
                                                  //| 2
  logWithDateBound("message-1")             //> Sat May 25 18:43:46 CST 2019 --- message-1
  logWithDateBound("message-2")             //> Sat May 25 18:43:46 CST 2019 --- message-2
  logWithDateBound("message-3")             //> Sat May 25 18:43:46 CST 2019 --- message-3
```

### 1.5. 闭包

在前面的例子中，在函数值或者代码块中使用的变量和值都是已经绑定的。你明确地知道它们所绑定的（实体），即本地变量或者参数。除此之外，你还可以创建带有未绑定变量的代码块。这样的话，你就必须在调用函数之前，为这些变量做绑定。但它们也可以绑定到或者捕获作用域和参数列表之外的变量。这也是这样的代码块被称之为闭包（closure）的原因。

```scala
  // 闭包
  def loopThrough(number: Int)(closure: Int => Unit): Unit = {
    for (i <- 1 to number) { closure(i) }
  }                                         //> loopThrough: (number: Int)(closure: Int => Unit)Unit
  
  var result = 0                            //> result  : Int = 0
  val addIt = { value: Int => result += value }
                                                  //> addIt  : Int => Unit = example$$$Lambda$37/836514715@544fe44c

  loopThrough(5)(elem => addIt(elem))
  println(result)                           //> 15
  loopThrough(5)(addIt)
  println(result)                           //> 30
  
  var product = 1                           //> product  : Int = 1
  loopThrough(5)( product *= _ )
  println(product)                          //> 120
```

多次调用都作用到了 result 变量，虽然 loopThrough 的实现和参数都与 result 无关。

## 2. 特质

特质类似于带有部分实现的接口，提供了一种介于单继承和多继承的中间能力，因为可以将它们混入或包含到其他类中。通过这种能力，可以使用横切特性增强类或者实例。

例如，要做一个关于朋友的抽象建模，我们可以将一个Friend特质混入任何的类中，如Man、Woman、Dog等，而又不必让所有这些类都继承同一个公共基类。我们在特质中定义并初始化的val和var变量，将会在混入了该特质的类的内部被实现。任何已定义但未被初始化的val和var变量都被认为是抽象的，混入这些特质的类需要实现它们。

举个例子：

```scala
  trait Friend {
    val name: String
    def listen(): Unit = println(s"Your friend $name is listening.")
  }
  
  class Human(val name: String) extends Friend
  class Woman(override val name: String) extends Human(name)
  class Man(override val name: String) extends Human(name)
```

Human类混入了Friend特质。如果一个类没有扩展任何其他类，则使用extends关键字来混入特质。

我们可以混入任意数量的特质。如果要混入额外的特质，要使用with关键字。如果一个类已经扩展了另外一个类（如在下一个示例中的Dog类），那么我们也可以使用with关键字来混入第一个特质。

```scala
  class Animal
  class Dog(val name: String) extends Animal with Friend {
      override def listen(): Unit = println(s"$name's listening quietly.")
  }
```

定义了上述类后，我们看下使用的例子：

```scala
  val john = new Man("John")                      //> john  : UsingTraits.Man = UsingTraits$Man@506c589e
  val sara = new Woman("Sara")                    //> sara  : UsingTraits.Woman = UsingTraits$Woman@2752f6e2
  val comet = new Dog("Comet")                    //> comet  : UsingTraits.Dog = UsingTraits$Dog@e580929
  
  john.listen()                                   //> Your friend John is listening.
  sara listen()                                   //> Your friend Sara is listening.
  comet listen                                    //> Comet's listening quietly.
  
  val mansBestFriend: Friend = comet              //> mansBestFriend  : UsingTraits.Friend = UsingTraits$Dog@e580929
  mansBestFriend.listen()                         //> Comet's listening quietly.
  
  def helpAsFriend(friend: Friend): Unit = friend.listen()
                                                  //> helpAsFriend: (friend: UsingTraits.Friend)Unit
  helpAsFriend(sara)                              //> Your friend Sara is listening.
  helpAsFriend(comet)                             //> Comet's listening quietly.
```

### 2.1. 选择性混入

Cat类没有混入Friend特质，因此我们不能将一个Cat类的实例看作是一个Friend。如同我们在下面的代码中看到的，任何这样的尝试都会导致编译错误。但我们可以将该类某个实例看做一个 Friend.

```scala
  class Cat(val name: String) extends Animal
  val alf = new Cat("Alf")                        //> alf  : UsingTraits.Cat = UsingTraits$Cat$1@34ce8af7
  // type mismatch; found :UsingTraits.Cat required: UsingTraits.Friend
  // helpAsFriend(alf)
  val angel = new Cat("Angel") with Friend        //> angel  : UsingTraits.Cat with UsingTraits.Friend = UsingTraits$$anon$1@b6842
                                                  //| 86
  helpAsFriend(angel)                             //> Your friend Angel is listening.
```

Scala的特质给了开发人员很大的灵活性，可以将某个类的所有实例都看作是某个特质，也可以仅将特定的实例视为某个特质。

### 2.2. 装饰器模式

```scala
  abstract class Check {
    def check: String = "Checked Application Detais..."
  }
  
  trait CreditCheck extends Check {
      override def check: String = s"Checked Credit... ${super.check}"
  }
  
  trait EmployeeCheck extends Check {
      override def check: String = s"Checked Employment... ${super.check}"
  }
  
  trait CriminalRecordCheck extends Check {
      override def check: String = s"Check Criminal Record... ${super.check}"
    }

  val apartmentApplication = new Check with CreditCheck with CriminalRecordCheck
  //> apartmentApplication  : UsingTraits.Check with UsingTraits.CreditCheck  UsingTraits.CriminalRecordCheck = UsingTraits$$anon$2@880ec60
  apartmentApplication check  //> res0: String = Check Criminal Record... Checked Credit... Checked Application Detais...
```

通过 with 不同的 xxxCheck，我们可以组合出不同的 check 流程出来。

特质中，使用super来调用方法将会触发延迟绑定（late binding）。这不是对基类方法的调用。相反，调用将会被转发到混入该特质的类中。如果混入了多个特质，那么调用将会被转发到混入链中的下一个特质中，更加靠近混入这些特质的类。在这两个调用中，最右边的特质充当了第一个处理器，响应了对check()方法的调用。然后，它们调用了super.check()方法，并将调用转发到了它们左侧的特质。最终，最左侧的特质将会在实际的实例上调用check()方法。

