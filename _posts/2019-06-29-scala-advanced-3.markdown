---
title: "《Scala实用指南》读书笔记三：集合"
date: 2019-06-29 23:47:57
tags: [scala]
---

## 1. 集合

Scala标准库包含了一组丰富的集合类，以及用于组合、遍历和提取元素的强大操作。在创建Scala应用程序时，会经常用到这些集合。如果想要在使用Scala时更加具有生产力，彻底地学习这些集合是很有必要的。

在[scala 30分钟极简入门
](https://izualzhy.cn/scala-beginner#8-%E9%9B%86%E5%90%88)也简单介绍过集合。

例如:

```scala
  val colors = Set("Bule", "Green", "Red")        //> colors  : scala.collection.immutable.Set[String] = Set(Bule, Green, Red)
  colors.getClass                                 //> res0: Class[?0] = class scala.collection.immutable.Set$Set3
  colors getClass                                 //> res1: Class[?0] = class scala.collection.immutable.Set$Set3
```

根据所提供的参数，Scala发现我们需要的是一个Set[String]。同样地，如果是Set(1，2，3)，那么我们将会得到一个Set[Int]。因为特殊的apply()方法（也被称为工厂方法），所以才得以创建一个对象而又不用使用new关键字。类似于X(...)这样的语句，其中X是一个类的名称或者一个实例的引用，将会被看作是X.apply(...)。

提到 scala，不会不介绍集合，而且方法大都见名知意，因此为了简洁，这里直接贴一下代码：

```scala
  val feeds1 = Set("blog.toolshed.com", "pragdave.me", "blog.agiledeveloper.com")
  //> feeds1  : scala.collection.immutable.Set[String] = Set(blog.toolshed.com, pragdave.me, blog.agiledeveloper.com)
  val feeds2 = Set("blog.toolshed.com", "martinfowler.com/bliki")
  //> feeds2  : scala.collection.immutable.Set[String] = Set(blog.toolshed.com, martinfowler.com/bliki)

  feeds1 filter(_ contains "blog")  //> res2: scala.collection.immutable.Set[String] = Set(blog.toolshed.com, blog.agiledeveloper.com)
  feeds1 ++ feeds2                  //> res3: scala.collection.immutable.Set[String] = Set(blog.toolshed.com, pragdave.me, blog.agiledeveloper.com, martinfowler.com/bliki)
  feeds1 & feeds2                   //> res4: scala.collection.immutable.Set[String] = Set(blog.toolshed.com)

  feeds1 map ("http://" + _)                //> res5: scala.collection.immutable.Set[String] = Set(http://blog.toolshed.com,  http://pragdave.me, http://blog.agiledeveloper.com)

  //关联映射
  val feeds = Map(
    "Andy Hunt" -> "blog.toolshed.com",
    "Dave Thomas" -> "pragdave.me",
    "NFJS" -> "nofluffjuststuff.com/blog")
  //> feeds  : scala.collection.immutable.Map[String,String] = Map(Andy Hunt -> blog.toolshed.com, Dave Thomas -> pragdave.me, NFJS -> nofluffjuststuff.com/blog)

  feeds filterKeys (_ startsWith "D")       //> res6: scala.collection.immutable.Map[String,String] = Map(Dave Thomas -> pragdave.me)

  val filterNameStartWithDAndPraprogInFeed = feeds filter { element =>
    val (key, value) = element
    (key startsWith "D") && (value contains "pragdave")
    //> filterNameStartWithDAndPraprogInFeed  : scala.collection.immutable.Map[String,String] = Map(Dave Thomas -> pragdave.me)
  }

  //Some[T]
  feeds.get("Andy Hunt")                    //> res7: Option[String] = Some(blog.toolshed.com)
  //None
  feeds.get("?")                            //> res8: Option[String] = None
  
  feeds("Andy Hunt")                        //> res9: String = blog.toolshed.com
  // feeds("?")
  val newFeeds1 = feeds.updated("Venkat Subramaniam", "blog.agiledeveloper.com")
  //> newFeeds1  : scala.collection.immutable.Map[String,String] = Map(Andy Hunt -> blog.toolshed.com, Dave Thomas -> pragdave.me, NFJS -> nofluffjuststuff.com/blog, Venkat Subramaniam -> blog.agiledeveloper.com)

  // 不可变列表
  val lists = List("blog.toolshed.com", "pragdave.me", "blog.agiledeveloper.com")
  //> lists  : List[String] = List(blog.toolshed.com, pragdave.me, blog.agiledeveloper.com)
  // 如果我们想要前插一个元素，即将一个元素放在当前List的前面，我们可以使用特殊的：:()方法。
  // a :: list读作“将a前插到list”。虽然list跟在这个操作符之后，但它是list上的一个方法。
  "prepend" :: lists                        //> res10: List[String] = List(prepend, blog.toolshed.com, pragdave.me, blog.agiledeveloper.com)
  // 假设我们想要追加一个列表到另外一个列表，例如，将listA追加到另外一个列表list。那么我们可以使用：::()方法将list实际上前插到listA。
  // 因此，代码应该是list ::: listA，并读作“将list前插到listA”。
  // 因为List是不可变的，所以我们不会影响前面的任何一个列表。我们只是使用这两个列表中的元素创建了一个新列表。
  lists ::: List("another", "list")         //> res11: List[String] = List(blog.toolshed.com, pragdave.me, blog.agiledeveloper.com, another, list)
  lists.filter(_ contains "blog").mkString(" ,")
  //> res12: String = blog.toolshed.com ,blog.agiledeveloper.com
  lists.forall(_ contains "com")            //> res13: Boolean = false
  lists.exists(_ contains "dave")           //> res14: Boolean = true
  lists.exists(_ contains "?")              //> res15: Boolean = false
  
  lists.map(_.length)                       //> res16: List[Int] = List(17, 11, 23)
  
  // foldLeft 越来越简洁，/:()方法等价于foldLeft()方法，而\:()方法等价于foldRight()方法。
  lists.foldLeft(0){ (total, feed) => total + feed.length }
                                                  //> res17: Int = 51
  (0 /: lists){ (total, feed) => total + feed.length }
                                                  //> res18: Int = 51
  (0 /: lists){ _ + _.length }              //> res19: Int = 51
```

### 1.1. 方法名约定

如果要前插一个值到列表中，可以编写value :: list。即使它读起来好像是“将value前插到list中”，但是，该方法的目标实际上是list，而value作为参数，即list.::(value)。

如果方法名以冒号（:）结尾，那么调用的目标是该操作符后面的实例。Scala不允许使用字母作为操作符的名称，除非使用下划线对该操作符增加前缀。因此，一个名为jumpOver:()的方法是被拒绝的，但是jumpOver_:()则会被接受。

例如：

```scala
  class Cow {
    def ^(moon: Moon): Unit = println("Cow jumped over the moon")
  }
  
  class Moon {
    def ^:(cow: Cow): Unit = println("This cow jumped over the moon too")
  }
  
  val c = new Cow                           //> c  : Collections.Cow = Collections$Cow$1@5702b3b1
  val m = new Moon                          //> m  : Collections.Moon = Collections$Moon$1@69ea3742
  c ^ m //调用 c                              //> Cow jumped over the moon
  c ^: m //调用 m                             //> This cow jumped over the moon too
  m.^:(c)                                   //> This cow jumped over the moon too
```

^()方法是一个定义在Cow类上的方法，而^:()方法是独立定义在Moon类上的一个方法。对这两个方法的调用看起来几乎是完全一样的，cow都在操作符的左边，而moon都在操作符的右边。但是，第一个调用发生在cow上，而第二个调用发生在moon上，这一区别相当微妙。

`+-!~`这些操作符也是跟随着他们后面的实例：

```scala
  class Sample {
    def unary_+(): Unit = println("Called unary +")
    def unary_-(): Unit = println("Called unary -")
    def unary_!(): Unit = println("Called unary !")
    def unary_~(): Unit = println("Called unary ~")
  }
  
  val sample = new Sample                   //> sample  : Collections.Sample = Collections$Sample$1@4b952a2d
  +sample                                   //> Called unary +
  -sample                                   //> Called unary -
  !sample                                   //> Called unary !
  ~sample                                   //> Called unary ~
```

### 1.2. for 表达式

```scala
  for (i <- 1 to 3) {println(s"ho ${i}")}   //> ho 1
                                            //| ho 2
                                            //| ho 3
  val results1 = for (i <- 1 to 10)
    yield i * 2                       //> results1  : scala.collection.immutable.IndexedSeq[Int] = Vector(2, 4, 6, 8, 10, 12, 14, 16, 18, 20)
  val results2 = (1 to 10).map(_ * 2)       //> results2  : scala.collection.immutable.IndexedSeq[Int] = Vector(2, 4, 6, 8, 10, 12, 14, 16, 18, 20)
  // 列表推导： list comprehension
  val doubleEven1 = for (i <- 1 to 10; if i % 2 == 0)
    yield i * 2                       //> doubleEven1  : scala.collection.immutable.IndexedSeq[Int] = Vector(4, 8, 12, 16, 20)
  for {
    i <- 1 to 10
    if i % 2 == 0
  } yield i * 2                             //> res20: scala.collection.immutable.IndexedSeq[Int] = Vector(4, 8, 12, 16, 20)

  // 多个生成器的话，每个生成器都将形成一个内部循环
  // 最右边的生成器控制最里面的循环
  for (i <- 1 to 3; j <- 4 to 6) {
    println(s"($i, $j) ")             //> (1, 4)
                                      //| (1, 5)
                                      //| (1, 6)
                                      //| (2, 4)
                                      //| (2, 5)
                                      //| (2, 6)
                                      //| (3, 4)
                                      //| (3, 5)
                                      //| (3, 6)
```
