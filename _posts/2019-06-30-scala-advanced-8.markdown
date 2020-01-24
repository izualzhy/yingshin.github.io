---
title: "《Scala实用指南》读书笔记八：创建应用程序"
date: 2019-06-30 21:18:53
tags: [scala]
---

## 1. XML作为一等公民

Scala提供了一种类似于XPath的查询能力，它和XPath只有一点细微的差别。Scala不使用熟悉的XPath正斜杠`（/或者//）`来查询，而是使用反斜杠`（\和\\）`来作为分析和提取内容的方法。这种差别是必要的，因为Scala遵循Java的传统，使用两个正斜杠来进行注释，而单个正斜杠则是除法操作符。

```scala
  val xmlFragment =
      <symbols>
      <symbol ticker="AAPL"><units>200</units></symbol>
      <symbol ticker="IBM"><units>215</units></symbol>
    </symbols>  //> xmlFragment  : scala.xml.Elem = <symbols>
  // 天然支持 xml 的类
  println(xmlFragment.getClass)             //> class scala.xml.Elem

  // 取出 symbol 节点
  val symbolNodes = xmlFragment \ "symbol"  //> symbolNodes  : scala.xml.NodeSeq = NodeSeq(<symbol ticker="AAPL"><units>200</units></symbol>, <symbol ticker="IBM"><units>215</units></symbol>)
  // 逐个打印 symbol 节点
  symbolNodes foreach println
  //<symbol ticker="AAPL"><units>200</units></symbol>
  //<symbol ticker="IBM"><units>215</units></symbol>
  println(symbolNodes.getClass)             //> class scala.xml.NodeSeq$$anon$1

  // \() 方法只查找目标元素的直接子元素，如果要从目标元素开始的层次结构中搜索所有元素，应使用 \\()方法
  val unitsNodes = xmlFragment \\ "units"   //> unitsNodes  : scala.xml.NodeSeq = NodeSeq(<units>200</units>, <units>215</un
                                                  //| its>)

  unitsNodes foreach println                //> <units>200</units>
                                            //| <units>215</units>
  println(unitsNodes.getClass)              //> class scala.xml.NodeSeq$$anon$1
  println(unitsNodes.head.text)             //> 200
```

Scala 有强大的[模式匹配能力](https://izualzhy.cn/scala-advanced-4)。Scala 也将这一能力扩展到了匹配 XML 片段中：

```scala
  unitsNodes.head match {
    case <units>{ numberOfUnits}</units> => println(s"Units: $numberOfUnits")
  }                                         //> Units: 200
```

通过使用_*通配符，我们要求将`<symbols>和</symbols>`元素之间的所有内容都读到了占位符变量 symbolNodes里。

```scala
  xmlFragment match {
    case <symbols>{ symbolNodes @ _* }</symbols> =>
      for (symbolNode @ <symbol>{ _* }</symbol> <- symbolNodes) {
        println("%-7s %s".format(
          symbolNode \ "@ticker", (symbolNode \ "units").text))
      }
  }                                         //> AAPL    200
                                            //| IBM     215
```

同样，也可以对 node 像列表一样执行map方法：

```scala
  def updateUnitsAndCreateXML(element: (String, Int)) = {
    val (ticker, units) = element
    <symbol ticker={ ticker }>
      <units>{ units + 1 }</units>
    </symbol>
  }    //> updateUnitsAndCreateXML: (element: (String, Int))scala.xml.Elem
  val updatedStocksAndUnitsXML =
    <symbols>
      { stocksAndUnitsMap map updateUnitsAndCreateXML }
    </symbols>  
```

## 2. 从 Web 获取股票价格

本地文件 stocks.xml 记录了股票代码的列表及持有的数量，同时，我们记录了公司最近一段时间的股价，例如[GOOG的股价](https://raw.githubusercontent.com/ReactivePlatform/Pragmatic-Scala-StaticResources/master/src/main/resources/stocks/daily/daily_GOOG.csv)，要获得最新的收盘价，可以取第二行的数据。

因此，我们首先定义`getLatestClosingPrice`，该方法接收公司名作为参数，返回其时间及对应的收盘价(Record).

```scala
  import scala.io.Source

  case class Record(year: Int, month: Int, date: Int, closePrice: BigDecimal)

  def getLatestClosingPrice(symbol: String): BigDecimal = {
      // 访问有时超时，git clone 到本地，同时+了sleep 1s来代替网络延时
      /*
      val url = "https://raw.githubusercontent.com/ReactivePlatform/" +
        s"Pragmatic-Scala-StaticResources/master/src/main/resources" +
        s"/stocks/daily/daily_${symbol}.csv"
      */
      Thread.sleep(1000)
      val url = s"http://0.0.0.0:8000/" +
        s"Pragmatic-Scala-StaticResources/src/main/resources/stocks/daily/daily_${symbol}.csv"
      val data = Source.fromURL(url).mkString
      // timestamp ,open     ,high     ,low      ,close    ,volume
      val latestClosePrize = data.split("\n")
        .slice(1, 2)
        .map(record => {
          val Array(timestamp, open, high, low, close, volume) = record.split(",")
          val Array(year, month, date) = timestamp.split("-")
          Record(year.toInt, month.toInt, date.toInt, BigDecimal(close.trim))
        })
        .map(_.closePrice)
        .head

      latestClosePrize
  }                                               //> getLatestClosingPrice: (symbol: String)BigDecimal
```

接着定义读取本地xml的方法，获取持有的股票名称及数量：

```scala
  def getTickersAndUnits: Map[String, Int] = {
      val stocksAndUnitsXML = scala.xml.XML.load("./stocks.xml")
      (Map[String, Int]() /: (stocksAndUnitsXML \ "symbol")) {
        (map, symbolNode) =>
          val ticker = (symbolNode \ "@ticker").toString
          val units = (symbolNode \ "units").text.toInt
          map + (ticker -> units)
      }
  }                                               //> getTickersAndUnits: => Map[String,Int]
  
  val symbolsAndUnits = getTickersAndUnits        //> symbolsAndUnits  : Map[String,Int] = Map(MSFT -> 190, AAPL -> 200, AMD -> 150, HPQ -> 225, ORCL -> 200, INTC -> 160, IBM -> 215, ALU -> 150, VRSN -> 200, CSCO -> 250, TXN -> 190, ADBE -> 125, NSM -> 200, SYMC -> 230, XRX -> 240)
```

最后将两者的结果结合，并且按照股票名称排序：

```scala
  println("Ticker Units Closing Prics($) Total Value($)")
          //> Ticker Units Closing Prics($) Total Value($)

  val startTime = System.nanoTime()
  // 串行
  // val valuesAndWorth = symbolsAndUnits.keys.map { symbol =>
  // 并行
  val valuesAndWorth = symbolsAndUnits.keys.par.map { symbol =>
      val units = symbolsAndUnits(symbol)
      val latestClosingPrice = getLatestClosingPrice(symbol)
      val value = units * latestClosingPrice

      (symbol, units, latestClosingPrice, value)
  }

  val newWorth = ((BigDecimal(0.0)) /: valuesAndWorth) { (worth, valueAndWorth) =>
      val (_, _, _, value) = valueAndWorth
      worth + value
  }                                               //> newWorth  : scala.math.BigDecimal = 212071.3000

  val endTime = System.nanoTime()                 //> endTime  : Long = 63624493516148

  valuesAndWorth.toList.sortBy { _._1 } foreach { valueAndWorth =>
        println(valueAndWorth)
      }

  println(f"$$$newWorth%.2f")                     //> $212071.30
  println(f"taken ${(endTime - startTime)/1.0e9}%.2f seconds")
                                                  //> taken 5.54 seconds
```
