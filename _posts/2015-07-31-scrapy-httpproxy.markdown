---
title:  "scrapy笔记一proxy的抓取"
date: 2015-07-31 23:03:31
excerpt: "scrapy笔记一proxy的抓取"
tags: [scrapy]
---

这篇笔记介绍使用scrapy写一个入门爬虫的示例，该爬虫从网站爬取一些ip/port数据。主要目的是先把一个爬虫启动起来，后续再逐步讲其中的一些细节。

<!--more-->

### 安装
scrapy的安装如果有root权限是比较方便的，没有的话需要手动编译安装一些库的依赖，以及使用pip/easy\_install安装一些包的依赖，具体不再细说，无非是修改PATH,LD\_LIBRARY\_PATH,C\_INCLUDE\_PATH等。

### 背景
想要做一个简单的爬虫，不可避免要遇到代理的问题。看到[快代理](http://www.kuaidaili.com/proxylist/)这里有一些可以用于爬虫的代理ip和port，如何能够使用scrapy把这些代理自动抓取下来呢？方案有很多，这篇笔记介绍下如何使用scrapy快速的实现这个目标。

### 实现
#### 目录结构
通过执行  
`scrapy startproject httpproxy`  
`scrapy genspider proxy_spider kuaidaili.com/proxylist/`  
可以获得一个最基本的爬虫目录结构   

```
|~~httpproxy
|~~|~~__init__.py
|~~|~~pipelines.py
|~~|~~items.py
|~~|~~spiders
|~~|~~|~~__init__.py
|~~|~~|~~proxy_spider.py
|~~|~~settings.py
|~~scrapy.cfg
```
修改几个文件，就可以达到我们爬虫的目标了。

#### 要抓取的对象
爬取的目标是结构化的数据，在scrapy中使用scrapy.Item来定义，修改items.py。  
我们需要爬取的是地址和端口，定义如下：

```
class HttpproxyItem(scrapy.Item):
    # define the fields for your item here like:
    ip = scrapy.Field()
    port = scrapy.Field()
```

#### 定义爬取的规则
从页面中提取需要的元素，可以使用xpath。
具体可以参考这个[链接](http://www.w3.org/TR/xpath/)，推荐chrome安装一个xpath helper，以及使用chrome里的copy XPath也可以。也可以使用`scrapy shell`这个命令来调试下是否该xpath能够生效。  
我们需要3个xpath的路径：地址、端口以及跳转导航条，这里直接贴出来。  
分别为  

```
//*[@id="list"]/table/tbody/tr/td[1]/text()
//*[@id="list"]/table/tbody/tr/td[2]/text()
//div[@id="listnav"]/ul/li/a/@href
```  
跳转导航条的作用是通过该页获取其他所有存储地址、端口的页面。  
修改后的proxy\_spider.py文件内容为：  

```
# -*- coding: utf-8 -*-
import scrapy
from httpproxy.items import HttpproxyItem


class ProxySpiderSpider(scrapy.Spider):
    name = "proxy_spider"
    # allowed_domains = ["http://www.kuaidaili.com/proxylist/"]
    start_urls = (
        'http://www.kuaidaili.com/proxylist/',
    )

    def parse(self, response):
        sel = scrapy.Selector(response)
        ip_list = sel.xpath('//*[@id="list"]/table/tbody/tr/td[1]/text()').\
            extract()
        port_list = sel.xpath('//*[@id="list"]/table/tbody/tr/td[2]/text()').\
            extract()
        for ip, port in zip(ip_list, port_list):
            yield HttpproxyItem(ip=ip, port=port)
        url_list = sel.xpath('//div[@id="listnav"]/ul/li/a/@href').extract()
        for url in url_list:
            yield scrapy.Request(url='http://www.kuaidaili.com'+url,
                                 callback=self.parse)

```

### 运行
执行命令  
`scrapy crawl proxy_spider -t json -o json.out`
得到文件json.out就是我们的爬取结果了。是不是很简单？

有任何问题，请给我留言。

### 参考资料
[http://scrapy-chs.readthedocs.org/zh_CN/latest/intro/overview.html](http://scrapy-chs.readthedocs.org/zh_CN/latest/intro/overview.html)
