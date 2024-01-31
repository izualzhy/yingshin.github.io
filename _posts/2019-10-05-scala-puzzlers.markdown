---
title: "《Scala谜题》读书笔记"
date: 2019-10-05 20:07:25
tags: scala
---

有些问题还没有完全搞懂，不过先记录下来。

先放一张 scala 的类图：

![scala-class-relations.jpeg](/assets/images/scala-class-relations.jpeg)

笔记按照题目顺序整理。

```scala
//scalaVersion:2.12
package numericOps

import scala.collection.mutable

object ScalaPuzzlers extends App {
  /*
//  1. 占位符
//  Hi
//  Hi
//  List(2, 3)
//  常规匿名函数是从 => 一直到代码块结束的所有代码
  println(List(1, 2).map{i => println("Hi"); i + 1})
//  Hi
//  List(2, 3)
//  占位符_语法定义的匿名函数，只包括含有_的表达式
  println(List(1, 2).map{println("Hi"); _ + 1})

//  2. 初始化变量
  var MONTH = 12; var DAY = 24
//  scala 总认为首字母大写的变量为常量，因此赋值失败
//  match case 时也是如此
//  not found: value HOUR...MINUTE...SECOND
//  var (HOUR, MINUTE, SECOND) = (12, 0, 0)
//  3. 成员声明的位置
  trait A {
    val audience: String
    println("Hello " + audience)
  }

  class BMember(a: String = "World") extends A {
    val audience = a
    println("I repeat: Hello " + audience)
  }

  class BConstructor(val audience: String = "World") extends A {
    println("I repeat: Hello " + audience)
  }
//  构造器中声明的，跟A不同
//  Hello null
//  I repeat: Hello Readers
  new BMember("Readers")
//  构造体中声明的，跟A相同
//  Hello Readers
//  I repeat: Hello Readers
  new BConstructor("Readers")
//  4. 继承 ?
  trait A {
    val foo: Int
    val bar = 10
    println("In A: foo: " + foo + ", bar: " + bar)
  }

  class B extends A {
    val foo: Int = 25
    println("In B: foo: " + foo + ", bar: " + bar)
  }

  class C extends B {
    override val bar: Int = 99
    println("In c: foo: " + foo + ", bar: " + bar)
  }
//  超类会在子类之前初始化
//  按照声明顺序初始化
//  重载的变量，只初始化一次，在最大的子类里
//  A B 里 bar 值为0，因为 bar 只在 C 里初始化一次
//  In A: foo: 0, bar: 0
//  In B: foo: 25, bar: 0
//  In c: foo: 25, bar: 99
  new C()
//  如果改成 def bar，那么A B C对应的 bar 值都为 99
//  5. 集合操作
  def sumSizes(collections: Iterable[Iterable[_]]): Int =
  collections.map(_.size).sum
//  操作符一般会保持输入的集合类型不变,set去重后只有一个值
//  4
  println(sumSizes(List(Set(1, 2), List(3, 4))))
//  2
  println(sumSizes(Set(List(1, 2), Set(3, 4))))
//  6. 参数类型
  def applyMulti[T](n: Int)(arg: T, f: T => T) =
    (1 to n).foldLeft(arg) { (acc, _) => f(acc) }
  def applyNCurried[T](n: Int)(arg: T)(f: T => T) =
    (1 to n).foldLeft(arg) { (acc, _) => f(acc) }
  def nextInt(n: Int) = n * n + 1
  def nextNumber[N](n: N)(implicit numericOps: Numeric[N]) =
    numericOps.plus(numericOps.times(n, n), numericOps.one)
  println(applyMulti(3)(2, nextInt))
  println(applyNCurried(3)(2)(nextInt))
//  一个参数类型的信息，对同一个参数列表里参数的不可用。对随后的参数列表是可用的。
//  println(applyMulti(3)(2.0, nextNumber))
  println(applyNCurried(3)(2.0)(nextNumber))

//  7. 闭包
  import collection.mutable.Buffer
  val accessors1 = Buffer.empty[() => Int]
  val accessors2 = Buffer.empty[() => Int]

  val data = Seq(100, 110, 120)
  var j = 0
  for (i <- 0 until data.length) {
    accessors1 += (() => data(i))
    accessors2 += (() => data(j))
    j += 1
  }
//  在闭包里，val存储为常规的Int;而var变成一个scala.runtime.IntRef
//  因此避免在闭包里使用可变var或可变对象的自由变量
//  如果要用，先赋值给val，然后传到闭包里
//  100
//  110
//  120
  accessors1.foreach(a1 => println(a1()))
//  抛出异常：java.lang.IndexOutOfBoundsException: 3
  accessors2.foreach(a2 => println(a2()))

//  8. Map表达式
  val xs = Seq(Seq("a", "b", "c"), Seq("d", "e", "f"),
    Seq("g", "h"), Seq("i", "j", "k"))
  val ys = for (Seq(x, y, z) <- xs) yield x + y + z
//  for (pattern <- expr) yield fun
//  转换后为
//  expr withFilter {
//    case pattern => true
//    case _ => false
//  } map { case pattern => fun }
//  因此for会自动跳过不匹配的值
//  List(abc, def, ijk)
  println(ys)
//  scala.MatchError: List(g, h)
  val zs = xs map { case Seq(x, y, z) => x + y + z }
  println(zs)

//  9. 循环引用变量
  object XY {
    object X {
      val value: Int = Y.value + 1
    }
    object Y {
      val value: Int = X.value + 1
    }
  }
//  重新执行多次都是一种结果：或者 X:1 Y:2 或者 X:2 Y:1
//  实际代码尽量避免循环引用
  println(if (math.random > 0.5) XY.X.value else XY.Y.value)

//  10. 等式的例子
  import collection.immutable.HashSet
  trait TraceHashCode {
    override def hashCode(): Int = {
      println(s"TRACE: In hashCode for ${this}")
      super.hashCode()
    }
  }
  case class Country(isoCode: String)
  def newSwitzInst = new Country("CH") with TraceHashCode
  val countriesInst = HashSet(newSwitzInst)
//  true
  println(countriesInst.iterator contains newSwitzInst)
//  true
  println(countriesInst contains newSwitzInst)

  case class CountryWithTrace(isoCode: String) extends TraceHashCode
  def newSwitzDecl = new CountryWithTrace("CH")
  val countriesDecl = HashSet(newSwitzDecl)
//  true
  println(countriesDecl.iterator contains newSwitzDecl)
//  false
  println(countriesDecl contains newSwitzDecl)

//  new Country 的对象都有相同的 hashCode，相同的 equal
//  new CountryWithTrace 的对象都有不同的 hashCode，相同的 equal
//  case class 的 equal/hashCode 实现都是基于结构等式的：如果两个实例有相同的类型和相等的构造器参数
//  那么这两个实例就应该是相等的，这点跟普通class不同，普通class的两个实例一定是不相等的
//  override 之后，case class 就失去了这个特性
//  instx 有相同的 hashCode() 且 == 为 true
  val inst1 = new Country("CH") with TraceHashCode
  val inst2 = new Country("CH") with TraceHashCode
  def inst3 = new Country("CH") with TraceHashCode
  def inst4 = new Country("CH") with TraceHashCode
  //  declx 有不同的 hashCode() 且 == 为 true
  var decl1 = new CountryWithTrace("CH")
  var decl2 = new CountryWithTrace("CH")
  def decl3 = new CountryWithTrace("CH")
  def decl4 = new CountryWithTrace("CH")

//  11. lazy val
  var x = 0
  lazy val y = 1 / x
  try {
    println(s"1st $y")
  } catch {
    case _: Exception =>
      x = 1
      // lazy val 如果初始化失败的话，下次取值时还会尝试初始化，直到成功
      // 2nd 1
      println(s"2nd $y")
  }

//  12. 集合的迭代顺序
  case class RomanNumeral(symbol: String, value: Int)
  implicit object RomanOrdering extends Ordering[RomanNumeral] {
    def compare(a: RomanNumeral, b: RomanNumeral) =
      a.value compare b.value
  }
  import collection.immutable.SortedSet
  val numerals = SortedSet(
    RomanNumeral("M", 1000),
    RomanNumeral("C", 100),
    RomanNumeral("X", 10),
    RomanNumeral("I", 1),
    RomanNumeral("D", 500),
    RomanNumeral("L", 50),
    RomanNumeral("V", 5)
  )
  println("Roman numeral symbols for 1 5 10 50 100 500 1000")
//  I V X L C D M
//  C D I L M V X
  for (num <- numerals; sym = num.symbol) { print(s"${sym} ")}
  println()
//  转换会保持集合的类型不变，map后的集合仍然为set，按照 ASCII 顺序排序
//  如果想要保持原始顺序，toSeq 转为 sequence
  numerals map { _.symbol } foreach { sym => print(s"${sym} ") }
//  I V X L C D M
  numerals.toSeq map { _.symbol } foreach { sym => print(s"${sym} ") }

//  13. 自引用
//  显式给出类型的话，递归值定义是允许的，因此接下来这两行编译正常
  val s1: String = s1
//  当右值里有未初始化的变量时，赋缺省值，此处为 null
  val s2: String = s2 + s2
//  java.lang.NullPointerException
//  println(s1, s1.length)
//  (nullnull,8)
  println(s2, s2.length)
//  尽量避免自引用的形式
//  14. Return语句
val two = (x: Int) => { return x; 2 }
  def sumItUp: Int = {
    def one(x: Int): Int = { return x; 1 }
//    如果 two 直接在外部定义，则会报 return outside method definition 的错误
//    而这里之所以编译成功，是因为作为 sumItUp 的最终返回值返回了
//    因此 two 的参数是什么，返回值就是什么
    val two = (x: Int) => { return x; 2 }
    1 + one(2) + two(5)
  }
//  5
  println(sumItUp)
//  15. 偏函数中的_
  var x = 0
  def counter() = { x += 1; x}
  def add(a: Int)(b: Int) = a + b
//  adder1 扩展为 a => add(counter)(a)
//  执行 adder1 才会计算参数 counter，而且每次都会重新计算。所以等价于 def adder1 = ...
  var adder1 = add(counter)(_)
//  偏函数，adder2 扩展为
//  val adder2 = {
//    val fresh = counter()
//    a => add(fresh)(a)
//  }
//  因此 counter 会立即计算，并且之后 adder2 使用的是这个镜像，多次调用 adder2 不会重新计算 counter
  var adder2 = add(counter) _
//  def adder1
//  x = 1
  println("x = " + x)
//  12
  println(adder1(10))
//  x = 2
  println("x = " + x)
//  11
  println(adder2(10))
//  x = 2
  println("x = " + x)
//  12
  println(adder1(10))
//  x = 13
  println("x = " + x)

//  16. 多参数列表
  def invert(v3: Int)(v2: Int = 2, v1: Int = 1): Unit = {
    println(v1 + ", " + v2 + ", " + v3)
  }
  val invert3 = invert(3) _
//  not enough arguments for method apply: (v1: Int, v2: Int)Unit in trait Function2.
//  Unspecified value parameter v2.
//  invert3(v1 = 2)//1, 2, 3
  invert3(v1 = 2, v2 = 1)//1, 2, 3
  invert3(v2 = 2, v1 = 1)//2, 1, 3
  invert3(v1 = 1, v2 = 2)//2, 1, 3
//  invert3 扩展为
//  def invert3 = new Function2(Int, Int, Unit) {
//    override def apply(v1: Int, v2: Int): Unit = invert(3)(v1, v2)
//  }
//  因此后续调用 invert3(...)指定的参数名，都以扩展后的函数样式为准，比如指定的参数名
//  invert 的参数名这里只是混淆作用，invert3 指定的参数名跟 invert 的参数名是没有关系的

//  17. 隐式参数
  implicit var z1 = 2
  def add(x: Int)(y: Int)(implicit z: Int) = x + y + z
  // addTo 类型为 Int => Int = <function1>，只接收一个参数
  def addTo(n: Int) = add(n) _
  implicit val z2 = 3
  val addTo1 = addTo(1)
//  隐式参数是由编译器在方法 addTo 编译时解析的，此时只有 z1 在范围内，因此使用 z1 变量
//  5 = 1 + 2 + 2
  println(addTo1(2))
  z1 = 200
//  203 = 1 + 2 + 200
  println(addTo1(2))
//  Int does not take parameters
//  addTo1(2)(3)
//  如果声明 addTo1 时， z1 z2 都在范围内，那么确实会报 ambiguous implicit values

//  18. 重载
  object Oh {
    def overloadA(u: Unit) = "I accept a Unit"
    def overloadA(u: Unit, n: Nothing) = "I accept a Unit and Nothing"
    def overloadB(n: Unit) = "I accept a Unit"
    def overloadB(n: Nothing) = "I accept Nothing"
  }
//  代码可以运行，不过会报 Warning： a pure expression does nothing in statement position
//  I accept a Unit
//  overloadA 两个重载形状不同，编译器可以简单判断使用第一个
//  编译器应用了值抛弃，传入的参数修改为 { 99; () }，不过同样会导致上述 Warning
  println(Oh overloadA 99)
//  Error: overloaded method value overloadB with alternatives
//  由于 overloadB 两个重载形状相同，因此编译器需要判断使用哪个，overloadB 两种形式都无法把 Int 作为参数，因此编译报错
//  println(Oh overloadB 99)

//  19. 命名参数和缺省参数
  class SimpleAdder {
    def add(x: Int = 1, y: Int = 2) = x + y
  }
  class AdderWithBonus extends SimpleAdder {
//    override def add(y: Int, x: Int): Int = super.add(x, y) + 10
    override def add(y: Int = 3, x: Int = 4): Int = super.add(x, y) + 10
  }
  val adder:SimpleAdder = new AdderWithBonus
//  13
//  函数参数名字使用基类的，默认值则采用子类的，这里转换为 add(_, 0)
//  第一个参数默认值为3，因此值为 3 + 0 + 10
  println(adder add (y = 0))
//  14
//  这里转换为 add(0, _)
//  第二个参数默认值为 4，因此值为 0 + 4 + 10
  println(adder add 0)
//  即使在 C++ 里，在 override 的函数里修改默认值，也是非常不建议的

//  20. 正则表达式
  def traceIt[T <: Iterator[_]](it: T) = {
    println(s"TRACE: using iterator '${it}'")
    it
  }
  val msg = "I love Scala"
//  这里跟书里结论不同
//  TRACE: using iterator '<iterator>
//  First match index:9
//  First match index:9
  println("First match index:" + traceIt("a".r.findAllIn(msg)).start)
  println("First match index:" + "a".r.findAllIn(msg).start)

//  21. 填充
  implicit class Padder(val sb: StringBuilder) extends AnyVal {
    def pad2(width: Int) = {
      1 to width - sb.length foreach { sb += '*' }
      sb
    }
  }
//  length == 14
  val greeting = new StringBuilder("Hello, kitteh! ")
//  Hello, kitteh! *
  println(greeting pad2 20)
//  length == 9
  val farewell = new StringBuilder("U go now.")
//  java.lang.StringIndexOutOfBoundsException: index 10,length 10
//  println(farewell pad2 20)
  //  如果直接这么写代码 1 to 6 foreach { println("Hi") }
  //  编译报错:
  //  type mismatch;
  //  found: Unit
  //  required: Int => ?
  //  也就是说，必须把 1 to 6 的 Int 传入并且使用
  //  不过这样会编译成功
  //  println(1 to 6 foreach { greeting })，但是什么都不会输出。
  //  原因是默认调用了 StringBuilder 的 apply(index:Int):Char 方法
  //  所以 farewell 会报下标超限的错误
  //  而 greeting 的输出结果，跟第一个例子相关，因为 "sb += '*'" 只会执行一次。

//  22. 投影
  import collection.JavaConverters._
  def javaMap:java.util.Map[String, java.lang.Integer] = {
    val map =
      new java.util.HashMap[String, java.lang.Integer]()
    map.put("key", null)
    map
  }
  val scalaMap = javaMap.asScala
  val scalaTypesMap =
    scalaMap.asInstanceOf[scala.collection.Map[String, Int]]

  println(scalaTypesMap("key") == null)//true
  println(scalaTypesMap("key") == 0) //true
  println(scalaMap("key") == null) //true
  println(scalaMap("key") == 0) //false
  println(null.asInstanceOf[Int] == 0) //true
//  23. 构造器参数
  class Printer(prompter: => Unit) {
    def print(message: String, prompted: Boolean = false): Unit = {
      if (prompted) prompter
      println(message)
    }
  }
  def prompt(): Unit = {
    print("puzzler$ ")
  }
//  puzzler$ Puzzled yet?
  new Printer { prompt } print ( message = "Puzzled yet? ")
//  puzzler$ Puzzled yet?
  new Printer { prompt } print ( message = "Puzzled yet? ",
    prompted = true)
//  实际上，上述两个语句都是转为以下形式调用的
//  new Printer(()) { prompt } print { message = "Puzzled yet? "}
//  因此都会先调用 prompt
//  使用 () 替代 {} 后，输出就是正常的了
//  Puzzled yet?
  new Printer ( prompt ) print ( message = "Puzzled yet? ")
//  puzzler$ Puzzled yet?
  new Printer ( prompt ) print ( message = "Puzzled yet? ",
    prompted = true)
//  24. Double.NaN
  def printSorted(a: Array[Double]): Unit = {
    util.Sorting.stableSort(a)
    println(a.mkString(" "))
  }
//  1.23 4.56 7.89 NaN
  printSorted(Array(7.89, Double.NaN, 1.23, 4.56))
//  1.23 4.56 7.89 NaN
  printSorted(Array(7.89, 1.23, Double.NaN, 4.56))
//  这里也跟书里结果不同，所以我觉得NaN在跟 Double 比较时行为是无法预期的
//  属于 undefined behavior
  println(Seq(1.0, Double.NaN, 1.1).max)// 1.1
  println(Seq(1.0, 1.1, Double.NaN).max)// NaN
  println(Double.NaN > 100)// false
  println(Double.NaN < 100)// false
  println(Double.NaN == 100)// false
  println(Double.NaN != 100)// true
//  25. getOrElse
  val zippedLists = (List(1, 2, 3) , List(4, 5, 6)).zipped
//  scala.MatchError: 10 (of class java.lang.Integer)
  val (x, y) = zippedLists.find(_._1 > 10).getOrElse(10)
//  Option[A].getOrElse 不一定返回类型 A:
//  final def getOrElse[B >: A](default: => B): B
//  例如
//  val a = Some(1) //a: Some[Int] = Some(1)
//  a.getOrElse(2) //res0: Int = 1
//  a.getOrElse("ufo") //res1: Any = 1
//  当参数为 2 的时候返回 Int，当参数为 string 时，返回 Int 与 String 的最直接的子类 Any，而不会报错
//  类似的，val (x, y) = ... 编译期间也不会报错，而是展开为如下形式
//  val a$ = zippedLists.find(_._1 > 10).getOrElse(10) match {
//    case (b, c) => (b, c)
//  }
//  val x = a$._1
//  val y = a$._2
//  因此在运行时由于模式匹配失败报错了
//  自然的，如果这么修改之后返回的 x 类型为 Any
//  val x = zippedLists.find(_._1 > 10).getOrElse(10)
//  x: Any = 10
//  26. Any Args
  def prependIfLong(candidate: Any, elems: Any*):Seq[Any] = {
    if (candidate.toString.length > 1)
      candidate +: elems
    else
      elems
  }
//  love
  println(prependIfLong("I", "love", "Scala")(0))
  def prependIfLongRefac(candidate: Any)(elems: Any*):Seq[Any] = {
    if (candidate.toString.length > 1)
      candidate +: elems
    else
      elems
  }
//  ArrayBuffer((I,love,Scala), 0)
  println(prependIfLongRefac("I", "love", "Scala")(0))

//  编译器把 "I", "love", "Scala" 放到元组，作为第一个参数，0则作为第二个参数
//  跟32题，自动插入一个元组()有点像
//  可以使用 -Yno-adapted-args 直接编译失败，或者 -Ywarn-adapted-args 发出编译warning.
//  27. null
  def objFromJava: Object = "string"
  def stringFromJava: String = null

  def printLengthIfString(a: AnyRef):Unit = a match {
    case str:String =>
      println(s"String of length ${str.length}")
    case _ => println("Not a string")
  }

//  String of length 6
  printLengthIfString(objFromJava)
//  Not a string
  printLengthIfString(stringFromJava)
//  Scala 类关系图：![scala-class-relations](/assets/images/scala-class-relations.jpeg)
//  Null 是 Scala 中仅仅为了与 Java 兼容而存在的特殊类型，只有一个实例：null
//  Null 是所有引用类型的子类型，因此，参数如果为引用类型，那么都可以传入 null
//  但是 null 除了 Null 不是任何其他类型的实例
//  28. AnyVal
  trait NutritionalInfo {
    type T <: AnyVal
    var value:T = _
  }
  val containsSugar = new NutritionalInfo { type T = Boolean }

//  注：跟书里不同
//  false
  println(containsSugar.value)
//  true
  println(! containsSugar.value)
//  29. 隐式变量
  object Scanner {
    trait Console { def display(item: String) }
    trait AlarmHandler extends (() => Unit)

    def scanItem(item: String)(implicit c: Console): Unit = {
      c.display(item)
    }

    def hitAlarmButton()(implicit ah:AlarmHandler) { ah() }
  }
  object NormalMode {
    implicit val ConsoleRenderer = new Scanner.Console {
      def display(item: String) { println(s"Found a ${item}") }
    }
    implicit val DefaultAlarmHandler = new Scanner.AlarmHandler {
      def apply() { println("ALARM! ALARM!")}
    }
  }
  object TestMode {
    implicit val ConsoleRenderer = new Scanner.Console {
      def display(item: String) { println("Found a detonator") }
    }
    implicit val TestAlarmHandler = new Scanner.AlarmHandler {
      def apply() { println("Test successful. well done! ") }
    }
  }

  import NormalMode._
//  Found a knife
  Scanner scanItem "knife"
//  ALARM! ALARM!
  Scanner.hitAlarmButton()

  import TestMode._
//  跟书里结果不同，待搞明白后再补充解释
//  明确的点是尽量不要使用同名的隐式变量
//  Scanner scanItem "shoe"
//  Scanner.hitAlarmButton()
//  30. 显式声明类型?
  class QuietType {
    implicit val stringToInt = (_:String).toInt
    println("4" - 2)
  }
  class OutspokenType {
    implicit val stringToInt: String => Int = _.toInt
    println("4" - 2)
  }
//  2
  new QuietType()
//  java.lang.StackOverflowError
  new OutspokenType()
//  31. View
  val ints = Map("15" -> List(1, 2, 3, 4, 5))
  val intsIter1 = ints map { case (k, v) => (k, v.toIterator) }
  val intsIter2 = ints mapValues(_.toIterator)
  //1, 2
  println(intsIter1("15").next, intsIter1("15").next)
  //1, 1
  println(intsIter2("15").next, intsIter2("15").next)
//  32. toSet
//  def toSet[B >: A]: Set[B]
//  Set(1, 2, 3)
  println(List("1", "2").toSet + "3")
//  toSet 后的()，解释成 apply 方法: def apply(elem: A): Boolean，查看元素是否存在
//  而不传递参数值时，则认为是Unit值，一个()，例如这样：
//  def foo(x: Any): Unit = {
//    println(s"Hi ${x}")
//  }
//
//  foo(1)
//  foo("World")
//  foo()
//  false3
//  ()不存在于该 Set，因此返回 false
  println(List("1", "2").toSet() + "3")
//  true3
  println(List("1", "2").toSet("1") + "3")
//  33. 缺省值
//  这个例子的结果也和书上不同，不过表达的主题是一样的：withDefaultValue 不同实例使用的是同一个缺失值对象
//  这个例子会更直观一些
//  val a = mutable.HashMap[String, mutable.Map[Int, String]]()
//            .withDefaultValue(mutable.Map[Int, String]())
//  a("id1")(2) = "three"
//  //  three
//  println(a("id1")(2))
//  //  three
//  println(a("id2")(2))
  import collection.mutable
  import collection.mutable.Buffer

  val accBalances: mutable.Map[String, Int] =
    mutable.Map() withDefaultValue 100
  def transaction(accHolder: String,
                  amount: Int,
                  accounts: mutable.Map[String, Int]): Unit = {
    accounts += accHolder -> (accounts(accHolder) + amount)
  }

  def accBalancesWithHist: mutable.Map[String, Buffer[Int]] =
    mutable.Map() withDefaultValue Buffer(100)
  def transactionWithHist(accHolder: String,
                          amount: Int,
                          accounts: mutable.Map[String, Buffer[Int]]): Unit = {
    val newAmount = accounts(accHolder).head + amount
    accounts += accHolder ->
      (newAmount +=: accounts(accHolder))
  }
  transaction("Alice", 100, accBalances)
//  200
  println(accBalances("Alice"))
//  100
  println(accBalances("Bob"))

  transactionWithHist("Dave", 100, accBalancesWithHist)
  println(accBalancesWithHist)
  println(accBalancesWithHist("Dave"))
  println(accBalancesWithHist("Carol"))
  println(accBalancesWithHist("Dave"))

//  34. 关于Main
  class AirportDay {
    def tryCheckBag(weight: Int): String =
      "It's not a full flight, Your bag is OK."
  }
  class StartOfVacation extends AirportDay {
    override def tryCheckBag(weight: Int): String =
      if (weight > 25)
        "Your bag is too heavy.Please repack it."
      else
        "Your bag is OK."
  }
  def goToCheckIn(bagWeight: Int)(implicit ad:AirportDay): Unit = {
    println(s"The agent says:${ad tryCheckBag bagWeight}")
  }
  object AirportSim {
    def main(args: Array[String]) = {
      implicit val quietTuesday = new AirportDay
      goToCheckIn(26)
      implicit val busyMonday = new StartOfVacation
      goToCheckIn(26)
    }
  }
  object AirportSim2 extends App {
    implicit val quietTuesday = new AirportDay
    goToCheckIn(26)
    implicit val busyMonday = new StartOfVacation
    goToCheckIn(26)
  }

//  跟书里结论不同
//  The agent says:It's not a full flight, Your bag is OK.
//  The agent says:Your bag is too heavy.Please repack it.
//  The agent says:It's not a full flight, Your bag is OK.
//  The agent says:Your bag is too heavy.Please repack it.
  AirportSim main Array()
  AirportSim2 main Array()

//  35. 列表
  type Dollar = Int
  final val Dollar:Dollar = 1
  var x:List[Dollar] = List(1, 2, 3)
//  大括号内翻译为 (x:Int) => Dollar
//  语句里第一个 x 指列表，第二个 x 表示一个传递给 map 函数的参数
//  List(1, 1, 1)
  println(x map { x: Int => Dollar })
//  java.lang.IndexOutOfBoundsException: 3
//  小括号内翻译为 x: (Int => Dollar)
//  语句里两个 x 都表示列表，因此报range的错误
//  如果将 x 值修改为 List(0, 1, 2)，则"侥幸"运行通过，输出List(0, 1, 2)
//  println(x map ( x: Int => Dollar ))
//  如果没有传递参数类型，也可以运行输出 List(1, 1, 1)
  println(x map ( x => Dollar ))
//  如果使用圆括号传递一个参数的匿名函数，而这个匿名函数含有类型申明，那参数一定要放到圆括号里
//  println(x map ( (x: Int) => Dollar ))
//  36. 计算集合的大小
  import collection.mutable
  def howManyItems(lunchbox: mutable.Set[String],
                   itemToAdd: String): Int = (lunchbox + itemToAdd).size
  def howManyItemsRefac(lunchbox: mutable.Iterable[String],
                        itemToAdd: String): Int = (lunchbox + itemToAdd).size
  val lunchbox =
    mutable.Set("chocolate bar", "orange juice", "sandwich")
//  4
//  Set(sandwich, chocolate bar, orange juice, apple)
  println(howManyItems(lunchbox, "apple"))
  println(lunchbox + "apple")
//  Iterable 不支持 +，这里实际上是先通过 Predef.any2stringadd 转换为 string
//  47
//  ArrayBuffer(Set(sandwich, chocolate bar, orange juice))apple
  println(howManyItemsRefac(lunchbox, "apple"))
  println(mutable.Iterable(lunchbox) + "apple")
//  3
//  + 创建了一个新的 set，因此大小仍然是3。修改集合使用 +=
  println(lunchbox.size)
   */
}


```
