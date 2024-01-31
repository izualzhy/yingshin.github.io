---
title: "《Scala实用指南》读书笔记六：并行集合"
date: 2019-06-30 11:10:00
tags: scala
---

如果惰性是提高效率之道路，那么并行性则可以被认为是提高效率之航线。如果两个或者多个任务可以按任意顺序序列执行，而又不会对结果的正确性产生任何影响，那么这些任务就可以并行执行。Scala为此提供了多种方式，其中最简单的方式是并行地处理集合中的元素。

这篇笔记是书中一个例子：给定一些城市的名字，从 web 获取对应的天气状况(xml格式)，按照城市名字排序展示。因此会用到 url/xml 操作，不过重点我们看下，如何释放一个集合的并行能力。

## 1. 顺序集合

首先定义一个方法，参数为城市名，返回这个城市的天气状况：

```scala
  import scala.io.Source
  import scala.xml._
  
  def getWeatherData(city: String) = {
    val response = Source.fromURL(
      s"https://raw.githubusercontent.com/ReactivePlatform/" +
      s"Pragmatic-Scala-StaticResources/master/src/main/resources/" +
      s"weathers/$city.xml")
    val xmlResponse = XML.loadString(response.mkString)
    val cityName = (xmlResponse \\ "city" \ "@name").text
    val temperature = (xmlResponse \\ "temperature" \ "@value").text
    val condition = (xmlResponse \\ "weather" \ "@value").text
    (cityName, temperature, condition)
  }                                         //> getWeatherData: (city: String)(String, String, String)
  // getWeatherData("Houston,us")              //> res0: (String, String, String) = (Houston,61,clear sky)
```

主要就是`fromURL`获取 xml 格式数据，然后解析的过程。`getWeatherData("Houston,us")`是测试用的例子，打开注释可以看下执行的效果。

然后定义一个方法，打印该返回值：

```scala
  def printWeatherData(weatherData: (String, String, String)): Unit = {
    val (cityName, temperature, condition) = weatherData
    println(f"$cityName%-15s $temperature%-6s $condition")
  }           //> printWeatherData: (weatherData: (String, String, String))Unit
  printWeatherData(getWeatherData("Sydney,australia"))
              //> Sydney          68     broken clouds
```

最后一个方法，定义城市名字的列表，逐个获取各个城市天气的情况并打印，记录整个耗时。其中参数是`getData`方法。该方法，接收一个城市名字列表，返回对应的一个三元组列表。

```scala
  def timeSample(getData: List[String] => List[(String, String, String)]): Unit = {
    val cities = List("Bangalore,india",
      "Berlin,germany",
      "Boston,us",
      "Brussels,belgium",
      "Chicago,us",
      "Houston,us",
      "Krakow,poland",
      "London,uk",
      "Minneapolis,us",
      "Oslo,norway",
      "Reykjavik,iceland",
      "Rome,italy",
      "Stockholm,sweden",
      "Sydney,australia",
      "Tromso,norway")
    val start = System.nanoTime
    getData(cities) sortBy{_._1} foreach printWeatherData
    val end = System.nanoTime
    println(s"Time taken: ${(end -start)/1.0e9} s")
  }
```

运行一下：

```scala
  timeSample(cities => cities map getWeatherData)
            //> Bangalore       88.57  few clouds
            //| Berlin          48.2   mist
            //| Boston          45.93  mist
            //| Brussels        49.21  clear sky
            //| Chicago         31.59  overcast clouds
            //| Houston         61     clear sky
            //| Krakow          55.4   broken clouds
            //| London          50     broken clouds
            //| Minneapolis     29.55  clear sky
            //| Oslo            42.8   fog
            //| Reykjavik       48.06  overcast clouds
            //| Rome            54.88  few clouds
            //| Stockholm       38.97  clear sky
            //| Sydney          68     broken clouds
            //| Tromso          35.6   clear sky
            //| Time taken: 44.019762313 s
```

这个理解起来不复杂，就是顺序获取，一共花了 44s。

## 2. 并行集合加速

前面的例子有两个部分：慢的部分——对于每个城市，我们都通过网络获取并收集天气信息，快的部分——我们对数据进行排序，并显示它们。非常简单，因为慢的部分被封装到了作为参数传递给timeSample()函数的函数值中。因此，我们只需要更换那部分代码来提高速度即可，而其余的部分则可以保持不变。

如果能够并行对 cities 执行 getWeatherData 方法，毫无疑问性能会得到提升。在 Scala 中，这并不复杂：

```scala
  timeSample(cities => (cities.par map getWeatherData).toList)
            ...
            //| Time taken: 3.225210903 s
```

对于许多顺序集合，Scala都拥有其并行版本。￼例如，ParArray是Array对应的并行版本，同样的，ParHashMap、ParHashSet和ParVector分别对应于HashMap、HashSet和Vector。我们可以使用par()和seq()方法来在顺序集合及其并行版本之间进行相互转换。

这本书看到这里，对于大数据的处理计算，Scala 真的开始让人惊艳了。
