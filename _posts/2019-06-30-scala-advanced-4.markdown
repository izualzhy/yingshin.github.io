---
title: "《Scala实用指南》读书笔记四：模式匹配和正则表达式"
date: 2019-06-30 00:09:09
tags: [scala]
---

## 1. 模式匹配

Scala的模式匹配非常灵活，可以匹配字面量和常量，以及使用通配符匹配任意的值、元组和列表，甚至还可以根据类型以及判定守卫来进行匹配。接下来我们就来逐个探索一下这些应用方式。

```scala
  // 匹配字面量和常量
  def activity(day: String): Unit = {
    day match {
      case "Sunday" => print("Eat, sleep, repeat... ")
      case "Saturday" => print("Hang out with friends... ")
      case "Monday" => print("...code for fun...")
      case "Friday" => print("...read a good book...")
    }
  }                                         //> activity: (day: String)Unit

  List("Monday", "Sunday", "Saturday").foreach { activity }
  //> ...code for fun...Eat, sleep, repeat... Hang out with friends...
  activity("Monday")                        //> ...code for fun...

  // 匹配通配符
  object DayOfWeek extends Enumeration {
    val SUNDAY: DayOfWeek.Value = Value("Sunday")
    val MONDAY: DayOfWeek.Value = Value("Monday")
    val TUESDAY: DayOfWeek.Value = Value("Tuesday")
    val WENDESDAY: DayOfWeek.Value = Value("Wendesday")
    val THURSDAY: DayOfWeek.Value = Value("Thurday")
    val FRIDAY: DayOfWeek.Value = Value("Friday")
    val SATURDAY: DayOfWeek.Value = Value("Saturday")
  }

  def activity2(day: DayOfWeek.Value): Unit = {
    day match {
      case DayOfWeek.SUNDAY => println("Eat, sleep, repeat...")
      case DayOfWeek.SATURDAY => println("Hang out with friends")
      case _ => println("... code for fun ...")
    }
  }                                         //> activity2: (day: PatternMatching.DayOfWeek.Value)Unit
  activity2(DayOfWeek.SATURDAY)             //> Hang out with friends
  activity2(DayOfWeek.MONDAY)               //> ... code for fun ...

  // 匹配元组和列表
  def processCoordinates(input: Any): Unit = {
    input match {
      case (lat, long) => printf("Processing (%d, %d)..", lat, long)
      case "done" => println("done")
      case _ => println("invalid input")
    }
  }                                         //> processCoordinates: (input: Any)Unit

  processCoordinates(39, -104)              //> Processing (39, -104)..
  processCoordinates("done")                //> done
  processCoordinates(0.123)                 //> invalid input

  def processItems(items: List[String]): Unit = {
    items match {
      case List("apple", "ibm") => println("Apples and IBMs")
      case List("red", "blue", "white") => println("Stars and Stripes..")
      case List("red", "blue", _*) => println("colors red, blue, ...")
      // 如果我们需要引用List中剩下的元素，可以在特殊的@符号￼之前放置一个变量名（如otherFruits）
      case List("apple", "orange", otherFruits @ _*) => println("appels, oranges, and " + otherFruits)
    }
  }                                         //> processItems: (items: List[String])Unit

  processItems(List("apple", "ibm"))        //> Apples and IBMs
  processItems(List("red", "blue", "green"))//> colors red, blue, ...
  processItems(List("red", "blue", "yellow"))
                                            //> colors red, blue, ...
  processItems(List("apple", "orange", "grapes", "dates"))
                                            //> appels, oranges, and List(grapes, dates)
  // 匹配类型和守卫
  def process(input: Any): Unit = {
    input match {
      case (_: Int, _: Int) => print("Processing (int, int)...")
      case (_: Double, _: Double) => print("Processing (double, double)...")
      case msg: Int if msg > 1000000 => println("Processing int > 1000000")
      case _: Int => print("Processing int...")
      case _: String => println("Processing string...")
      // 在编写多个case表达式时，它们的顺序很重要。Scala将会自上而下地对case表达式进行求值。￼如果把下一行提到更前，就会导致一个警告，以及一个不一样的结果，因为带有守卫的case语句永远也不会被执行。
      case _ => printf(s"Can't handle $input...")
    }
  }                                         //> process: (input: Any)Unit

  process(34.2, -159.3)                     //> Processing (double, double)...
  process(0)                                //> Processing int...
  process(1000001)                          //> Processing int > 1000000
  process(2.2)                              //> Can't handle 2.2...
  process("hello world")                    //> Processing string...
```

## 2. case表达式中的模式变量和常量

```scala
  // case表达式中的模式变量和常量
  class Sample {
    val max = 100

    def process(input: Int): Unit = {
      input match {
        case max => println(s"You matched max $max")
        // case this.max => println(s"You matched max $max")
        // case `max` => println(s"You matched max $max")
      }
    }
  }

  val sample = new Sample                   //> sample  : PatternMatching.Sample = PatternMatching$Sample$1@5442a311
  try {
    sample.process(0)
  } catch {
    case ex: Throwable => println(ex)
  }                                         //> You matched max 0
  sample.process(100)                       //> You matched max 100
```

上面代码的原意，是希望只处理参数为 100 的情况，不过可以看到 max 并没有作为一个匹配的值(100)。

对应的修改方式是其中注释的两行任意一行即可，可以观察到捕获异常：

```
//> scala.MatchError: 0 (of class java.lang.Integer)
```

不过最好的修改方式是用 Max 替换 max，即常量的名称使用大写字母。

## 3. 使用 case 类进行模式匹配

case 类轻量级，易于创建，可以使用 case 表达式来进行模式匹配。所有的参数都公开为值。

```scala
  // 使用case类进行模式匹配
  trait Trade
  case class Sell(stockSymbol: String, quantity: Int) extends Trade
  case class Buy(stockSymbol: String, quantity: Int) extends Trade
  case class Hedge(stockSymbol: String, quantity: Int) extends Trade
  
  object TradeProcessor {
    def processTransaction(request: Trade): Unit = {
      request match {
        case Sell(stock, 1000) => println(s"Selling 1000-units of $stock")
        case Sell(stock, quantity) => println(s"Selling $quantity units of $stock")
        case Buy(stock, quantity) if quantity > 2000 => println(s"Buying $quantity (large) units of $stock")
        case Buy(stock, quantity) => println(s"Buying $quantity units of $stock")
      }
    }
  }
  
  TradeProcessor.processTransaction(Sell("GOOG", 500))
                                                  //> Selling 500 units of GOOG
  TradeProcessor.processTransaction(Buy("GOOG", 200))
                                                  //> Buying 200 units of GOOG
  TradeProcessor.processTransaction(Sell("GOOG", 1000))
                                                  //> Selling 1000-units of GOOG
  TradeProcessor.processTransaction(Buy("GOOG", 3000))
                                                  //> Buying 3000 (large) units of GOOG
```

如果 case 类没有参数时，一定记得加一个()，表名接受一个空的参数列表:

```scala
  case class Apple()
  case class Orange()
  case class Book()

  object ThingsAcceptor {
    def acceptStuff(thing: Any): Unit = {
      thing match {
        case Apple() => println("Thanks for the Apple")
        case Orange() => println("Thanks for the Orange")
        case Book() => println("Thanks for the Book")
        case _ => println(s"$thing ?")
      }
    }
  }
  ThingsAcceptor.acceptStuff(Apple())       //> Thanks for the Apple
  ThingsAcceptor.acceptStuff(Book())        //> Thanks for the Book
  ThingsAcceptor.acceptStuff(Apple)         //> Apple ?
```

## 4. 提取器

通过使用 Scala 的提取器来匹配任意模式，可以将模式匹配提升到下一个等级。顾名思义，提取器将从输入中提取匹配的部分。看一个例子，`StockService/ReceiveStockPrice/Symbol`如何通过 match 来实现提取 GOOG/IBM 的价格。

```scala
  // 提取器和正则表达式
  object Symbol {
    def unapply(symbol: String): Boolean = {
      symbol == "GOOG" || symbol == "IBM"
    }
  }
  object ReceiveStockPrice {
    def unapply(input: String): Option[(String, Double)] = {
      try {
        if (input contains ":") {
          val splitQuote = input split ":"
          Some((splitQuote(0), splitQuote(1).toDouble))
        } else {
          None
        }
      } catch {
        case _: NumberFormatException => None
      }
    }
  }
  object StockService {
    def process(input: String): Unit = {
      input match {
        // 当 case Symbol()=> 被执行的时候，match 表达式将自动将 input 作为参数发送给 unapply() 方法
        case Symbol() => println(s"Look up price for valid symbol $input")
        case ReceiveStockPrice(symbol @ Symbol(), price) =>
          println(s"Received price $$$price for symbol $symbol")
        case _ => println(s"Invalid input $input")
      }
    }
  }
  StockService process "GOOG"               //> Look up price for valid symbol GOOG
  StockService process "IBM"                //> Look up price for valid symbol IBM
  StockService process "ERR"                //> Invalid input ERR
  StockService process "GOOG:310.84"        //> Received price $310.84 for symbol GOOG
  StockService process "GOOG:BUY"           //> Invalid input GOOG:BUY
  StockService process "ERR:12.3"           //> Invalid input ERR:12.3
```

该提取器具有一个名为unapply()的方法，它接受我们想要匹配的值。当case Symbol()=>被执行的时候，match表达式将自动将input作为参数发送给unapply()方法。

由于方法名奇怪，unapply()方法可能会让你感到吃惊。对于提取器，你可能会预期类似于evaluate()这样的方法。提取器有这样的方法名的原因是：提取器也可能会有一个可选的apply()方法。这两个方法，即apply()和unapply()，执行的是相反的操作。unapply()方法将对象分解成匹配模式的部分，而apply()方法则倾向于将它们再次合并到一起。

注意`process`实现里，有个`symbol @ Symbol()`的语法：第一个结果（symbol）上，我们进一步应用了Symbol提取器来验证股票代码。我们可以使用一个模式变量，然后在其后面跟上@符号，在该股票代码从一个提取器到另外一个提取器的过程中对股票代码进行拦截。类似前面`otherFruits @ _*`.

## 5. 正则表达式

看个使用正则的例子：

```scala
  //正则表达式
  val pattern = "(S|s)cala".r               //> pattern  : scala.util.matching.Regex = (S|s)cala
  val str = "Scala is scalable and cool"    //> str  : String = Scala is scalable and cool
  println(pattern findFirstIn str)          //> Some(Scala)
  println((pattern findAllIn str).mkString(", "))
                                            //> Scala, scala
  println("cool".r replaceFirstIn (str, "awesome"))
                                            //> Scala is scalable and awesome
  println("(S|s)cala".r replaceAllIn (str, "haha"))
                                            //> haha is hahable and cool
```

正则表达式可以作为一个提取器，Scala将放置在括号中的每个匹配项看作是一个模式变量。因此，例如"(Sls)cala".r的unapply()方法将会保存返回`Option[String]`，"(Sls)(cala)".r的unapply()方法将会返回`Option[(String, String)]`。

在 Scala 中，正则表达式和模式匹配密不可分，我们通过正则来提取股价：

```scala
  def processRegex2(input: String): Unit = {
    val MatchStock = """^(.+):(\d*\.\d+)""".r
    input match {
      case MatchStock("GOOG", price) => println(s"We got GOOG at $$$price")
      case MatchStock("IBM", price) => println(s"IBM's trading at $$$price")
      case MatchStock(symbol, price) => println(s"Price of $symbol is $$$price")
      case _ => println(s"not processing $input")
    }
  }                                         //> processRegex2: (input: String)Unit
  processRegex2("GOOG:310.1")               //> We got GOOG at $310.1
  processRegex2("IBM:98.0")                 //> IBM's trading at $98.0
  processRegex2("GE:31.1")                  //> Price of GE is $31.1
```

## 6. 无处不在的下划线字符

+ 作为包引入的通配符。例如，在Scala中import java.util._等同于Java中的import java.util.*。  
+ 作为元组索引的前缀。对于给定的一个元组val names = ("Tom", "Jerry")，可以使用names._1和names._2来分别索引这两个值。  
+ 作为函数值的隐式参数。代码片段list.map { _ * 2 }和list.map { e => e* 2 }是等价的。同样，代码片段list.reduce { _ + _ }和list.reduce { (a, b) => a + b }也是等价的。  
+ 用于用默认值初始化变量。例如，var min : Int = _将使用0初始化变量min，而var msg : String = _将使用null初始化变量msg。  
+ 用于在函数名中混合操作符。你应该还记得，在Scala中，操作符被定义为方法。例如，用于将元素前插到一个列表中的：:()方法。Scala不允许直接使用字母和数字字符的操作符。例如，foo：是不允许的，但是可以通过使用下划线来绕过这个限制，如foo_:。  
+ 在进行模式匹配时作为通配符。`case _`将会匹配任意给定的值，而`case _:Int`将匹配任何整数。此外，`case <people>{_*}</people>`将会匹配名为people的XML元素，其具有0个或者多个子元素。  
+ 在处理异常时，在catch代码块中和case联用。
+ 作为分解操作的一部分。例如，max(arg: _*)在将数组或者列表参数传递给接受可变长度参数的函数前，将其分解为离散的值。  
+ 用于部分应用一个函数。例如，在代码片段val square = Math.pow(_: Int, 2)中，我们部分应用了pow()方法来创建了一个square()函数。

_符号的目的是为了使代码更加简洁和富有表现力。开发人员应该根据自己的判断来决定何时使用该符号。只在代码真的变得更加简洁的时候才使用它，也就是说，代码是透明的，而且易于理解和维护。当你觉得代码变得生硬、难以理解或者晦涩时，就避免使用它。
