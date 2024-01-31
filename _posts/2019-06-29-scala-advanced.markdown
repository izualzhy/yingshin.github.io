---
title: "《Scala实用指南》读书笔记一：处理对象与善用类型"
date: 2019-06-29 09:50:51
tags: scala
---

## 1. 处理对象

### 1.1. 单例

可以选择将一个单例关联到一个类。这样的单例，其名字和对应类的名字一致，一次你被称为伴生对象(companion object)。相应的类被称为伴生类：

```scala
object singleton extends App {
  import scala.collection._
  
  class Marker private (val color: String) {
    println(s"Creating ${this}")
    override def toString = s"markier color $color"
  }
  object Marker {
    private val markers = mutable.Map(
      "red" -> new Marker("red"),
      "blue" -> new Marker("blue"),
      "yellow" -> new Marker("yellow")
    )
    
    def getMarker(color: String): Marker =
      markers.getOrElseUpdate(color, new Marker(color))
  }


  println(Marker getMarker "blue")
  println(Marker getMarker "blue")
  println(Marker getMarker "red")
  println(Marker getMarker "red")
  println(Marker getMarker "green")
  //error: constructor Marker in class Marker cannot be accessed in object singleton
  //val marker = new Marker("not allowed")
}
```

Marker 的构造器被声明为 private;然而，它的伴生对象可以访问它。因此，我们可以在伴生对象中创建 Marker 的实例。如果试着在类或者伴生对象之外创建 Marker 的实例，就会收到错误提示。

### 1.2. static

Scala 没有 static 关键字(未来或许会支持 @static 修饰)。对上一节的 Marker 对象，如果我们想获取所有支持的颜色，这个方法按理应该是一个类级别的方法。可以在`object Marker`里增加方法：

```scala
def supportedColors: Iterable[String] = markers.keys
```

就可以调用`Marker.supportedColors`获取支持的全部颜色:

```scala
println(s"Supported colors are : ${Marker.supportedColors}")
```

同时，定义`apply`替换`getMarker`方法，可以使 Marker 更加简洁

```
    def apply(color: String): Marker =
      markers.getOrElseUpdate(color, new Marker(color))
```

我们可以直接使用`println(Marker("blue"))`来获取伴生对象 Marker 的实例，特殊的 apply() 方法是达到这种效果的关键。当我们调用`Marker("blue")`时，实际上在调用`Maker.apply("blue")`。这是一种创建或者获得实例的轻量级语法。

### 1.3. 枚举

要在 scala 中创建枚举，要先从创建对象开始，这和创建一个单例的语法特别相像。例如创建一个货币的枚举：

```scala
// Currency.scala
object Currency extends Enumeration{
  type Currency = Value
  val CNY, GBP, INR, JPY, NOK, PLN, SEK, USD = Value
}
```

以及类`Money`使用`Currency`作为一个参数

```scala
// Money.scala
import Currency._

class Money(val amount: Int, val currency: Currency) {
  override def toString = s"$amount $currency"
}
```

可以用 values 这个属性遍历枚举的所有值：

```scala
// UseCurrency.scala
object UseCurrency extends App {
  Currency.values.foreach { currency ⇒ println(currency) }
  // 创建一个 Money 实例
  println(s"${new Money(2, Currency.USD)}")
}
```

## 2. 善用类型

Scala 的关键优点之一便是 Scala 是静态类型的。通过静态类型，编译器充当了抵御错误的第一道防线。它们可以验证当前的对象是否就是想要的类型。

### 2.1. Nothing && Any

Nothing 是所有类型的子类型，Any 是所有类型的基础类型。

⁨![Nothing_Any](/assets/images/scala/Nothing_Any.png)

Scala 将跑出异常的表达式的返回类型推断为 Nothing。Nothing 是抽象的，因此在运行时永远都不会得到一个真正的 Nothing 实例。它是一个纯粹的辅助类型，用于类型判断以及类型验证。

```scala
  //Scala将抛出异常的表达式的返回类型推断为Nothing
  def madMethod() = { throw new IllegalArgumentException() }
                                                  //> madMethod: ()Nothing
```

### 2.2. Option

函数实现里，根据参数不同，有的分支返回某种类型，有的则什么都不返回。`Option[T]`用于解决这个问题，例如：

```scala
  //commentOnPractice 方法返回的是 Some[T] 的实例或者 None，而不是 String 的实例。
  // 这两个类都继承自 Option[T] 类
  def commentOnPractice(input: String) = {
    if (input == "test") Some("good") else None
  }                                         //> commentOnPractice: (input: String)Option[String]

  for (input <- Set("test", "hack")) {
    val comment = commentOnPractice(input)
    // Option[T]的getOrElse()方法
    val commentDisplay = comment.getOrElse("Found no comments")
    println(s"input: $input comment: $commentDisplay")
  }                                         //> input: test comment: good
                                                  //| input: hack comment: Found no comments
```

### 2.3. Either

Either 用于解决返回两种类型的情况

```scala
  //当一个函数调用的结果可能存在也可能不存在时，Option类型很有用
  //有时候，你可能希望从一个函数中返回两种不同类型的值之一。
  //这个时候，Scala的Either类型就排上用场了
  //Either类型有两种值：左值(通常被认为是错误)和右值(通常被认为是正确的或者符合预期的值)
  def compute(input: Int) =
    if (input > 0)
      Right(math.sqrt(input))
    else
      Left("Error computing, invalid input")
                                                  //> compute: (input: Int)scala.util.Either[String,Double]

  def displayResult(result: Either[String, Double]): Unit = {
    println(s"Raw: $result")
    result match {
      case Right(value) => println(s"result $value")
      case Left(err) => println(s"Error: $err")
    }
  }                                         //> displayResult: (result: Either[String,Double])Unit

  displayResult(compute(4))                 //> Raw: Right(2.0)
                                                  //| result 2.0
  displayResult(compute(-4))                //> Raw: Left(Error computing, invalid input)
                                                  //| Error: Error computing, invalid input
```

### 2.4. 返回值类型推断

只有当你使用等号(=)将方法的声明和方法的主体部分区分开时，Scala 的返回值类型推断才会生效。否则，该方法将会被视为返回一个 Unit，等效于 Java 的 Void.

例如:

```scala
  // 不使用=，返回Unit
  def function1 { Math.sqrt(4) }            //> function1: => Unit
  def function2 = { Math.sqrt(4) }          //> function2: => Double
  // 如果一个函数的主题是一个简单的表达式或者复合表达式，那么就可以删除大括号
  def function3 = Math.sqrt(4)              //> function3: => Double
  def function4: Double = { Math.sqrt(4) }  //> function4: => Double
```

### 2.5. 参数化类型的型变

在期望接收一个基类实例的集合的地方，能够使用一个子类实例的集合的能力叫做协变(covariance)。而在期望接收一个子类实例的集合的地方，能够使用一个超类实例的集合的能力叫做逆变(contravariance)。在默认的情况下，Scala 都不允许(即不变)。

#### 2.5.1. 协变

我们定义了两个类，其中 Dog 类扩展了 Pet 类。我们有一个方法 workWithPets，它接受一个 Pet 的数组，但是实际上什么也没做。创建一个 Dog 的数组 dogs ，如果把 dogs 传递给 workWithPets 方法，会得到一个编译错误：

```scala
// 协变
  class Pet(val name: String) {
    override def toString: String = name
  }
  class Dog(override val name: String) extends Pet(name)
  def workWithPets(pets: Array[Pet]): Unit = {}

  val dogs = Array(new Dog("Rover"), new Dog("Comet"))
  // type mismatch;
  // found   : Array[learn_type.Dog]
  // required: Array[learn_type.Pet]
  // Note: learn_type.Dog <: learn_type.Pet, but class Array is invariant in type T.
  // You may wish to investigate a wildcard type such as `_ <: learn_type.Pet`. (SLS 3.2.10)
  workWithPets(dogs)
```

scala 抱怨对 workWithPets() 方法的调用--我们不能将一个包含 Dog 的数组发送给一个接受 Pet 的数组的方法。但是，这个方法是无害的。例如:

```scala
  def workWithPets[T <: Pet](pets: Array[T]): Unit =
    println("Playing with pets: " + pets.mkString(", "))
```

`T<:Pet`表名由 T 表示的类派生自 Pet 类。这个语法用于定义一个上界（如果可视化这个类的层次结构，那么 Pet 将会是类型 T 的上界），T 可以是任何类型的 Pet，也可以是在该类型层次结构中低于 Pet 的类型。通过指定上界，我们告诉 Scala 数组参数的类型参数 T 必须至少是一个 Pet 的数组，但是也可以是任何派生自 Pet 类型的类的实例数组。

#### 2.5.2. 逆变

逆变则对应了`Base>:Derived`的场景，例如定一个 copy() 的方法，用于 dogs copy 到`Array[Pet]`

```scala
  // 逆变
  def copyPets[S, D >: S](fromPets: Array[S], toPets: Array[D]): Unit = {
    println("from:" + fromPets.mkString(", ") + " to:" + toPets.mkString(", "))
  }                                         //> copyPets: [S, D >: S](fromPets: Array[S], toPets: Array[D])Unit
  val pets = new Array[Pet](10)             //> pets  : Array[learn_type.Pet] = Array(null, null, null, null, null, null, null, null, null, null)
  copyPets(dogs, pets)                      //> from:Rover, Comet to:null, null, null, null, null, null, null, null, null, null
```

### 2.6. 隐式类型转换

在使用日期和时间操作时，如果能编写下面的代码，那将会非常方便，并且具有更好的可读性：

```scala
2 days ago
5 days from_now
```

这看起来更像是数据输入，而不是代码--DSL的特性之一。通过隐式类型转换，scala 可以实现这样的魔法。

如果想要使用隐式转换函数，那么Scala将会要求导入scala.language.implicit Conversions。这将有助于提醒阅读代码的人代码中即将进行类型转换。

#### 2.6.1. 隐式函数

真正的乐趣在于，在一个Int上调用days()方法，并让Scala静默地将Int转换为一个DateHelper的实例，这样就可以调用这个方法了。Scala只需在一个简单的函数前面加上implicit关键字即可使用启用这个技巧的特性。

让我们创建这个隐式函数:

```scala
import scala.language.implicitConversions
import java.time.LocalDate

class DateHelper(offset: Int) {
  def days(when: String): LocalDate = {
    val today = LocalDate.now
    when match {
      case "ago" ⇒ today.minusDays(offset)
      case "from_now" ⇒ today.plusDays(offset)
      case _ ⇒ today
    }
  }
}

object DateHelper {
  val ago = "ago"
  val from_now = "from_now"
  implicit def convertInt2DateHelper(offset: Int): DateHelper = new DateHelper(offset)
}
```

如果一个函数被标记为implicit，且在当前作用域中存在这个函数（通过当前的import语句导入，或者存在于当前文件中），那么Scala都将会自动使用这个函数。看一个使用了我们编写在`DateHelper`伴生对象中的隐式转换的例子：

```scala
import DateHelper._

object DayDSL extends App {
  val past = 2 days ago
  val appointment = 5 days from_now
  
  println(past)
  println(appointment)
}
```

#### 2.6.2. 隐式类

相对于创建一个常规类和一个单独的隐式转换方法，你可以告诉Scala，某个类的唯一目的就是作为一种适配器或者转换器。为此，可以将一个类标记为implicit类。

例如：

```scala
object DateUtil {
  val ago = "ago"
  val from_now = "from_now"

  implicit class DateHelper(val offset: Int) {
    import java.time.LocalDate
    def days(when: String): LocalDate = {
      val today = LocalDate.now
      when match {
        case "ago" ⇒ today.minusDays(offset)
        case "from_now" ⇒ today.plusDays(offset)
        case _ ⇒ today
      }
    }
  }
}
```

使用上跟隐式函数几乎是一样的：

```scala
object DayDSL extends App {
  import DateUtil._
  val past = 2 days ago
  val appointment = 5 days from_now
  
  println(past)
  println(appointment)
}
```

为了提供流利性、易用性并使用领域特定方法对现有类进行扩展，我们更倾向于使用隐式类，而不是隐式方法——隐式类表意更加清晰明确，并且比任意的隐式转换方法更容易定位。

### 2.7. 使用隐式转换

看一个使用字符串插值器来创建一个隐式转换的实际例子。

```scala
  import MyInterpolator._

  val ssn = "123-45-6789"                   //> ssn  : String = 123-45-6789
  val account = "0123456789"                //> account  : String = 0123456789
  val balance = 20145.23                    //> balance  : Double = 20145.23

  mask"Account: $account Social Security Number: $ssn Balance: $$^$balance Thanks for you business."
  //| res12: StringBuilder = Account: ...0123456789 Social Security Number: ...123-45-6789 Balance: $20145.23 Thanks for you business.
```

其中`MyInterpolator.mask`就是我们自定义的插值器，mask 这一行实际上转换为：

```
new StringContext("Account: ", "Social Security Number: ", "Balane: ", "$^", "Thanks for you business.").mask(account, ssn, balance)
```

对比这一行，再来看`MyInterpolator`的实现就比较容易了：

```scala

object MyInterpolator {
  implicit class Interpolator(val context: StringContext) extends AnyVal {
    def mask(args: Any*): StringBuilder = {
      val processed = context.parts.zip(args).map { item ⇒
        val (text, expression) = item
        // $^输出为$，接expression
        if (text.endsWith("^"))
          s"${text.split('^')(0)}$expression"
        // 其他原样输出，...接expression
        else
          s"$text...${expression}"
      }.mkString

      // 补全结尾数据
      new StringBuilder(processed).append(context.parts.last)
    }
  }
}
```

