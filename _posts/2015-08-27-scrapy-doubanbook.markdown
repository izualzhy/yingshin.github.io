---
title:  "使用scrapy爬取豆瓣书籍"
date: 2015-08-27 14:35:51
excerpt: "使用scrapy爬取豆瓣书籍"
tags: [scrapy]
---

这次爬虫的目标是从豆瓣读书抓取书籍以及对应的标签，抓取后的数据存放到mysql。

<!--more-->


先看下抓取后的数据

```
mysql> select * from doubanbook where title = '三体  '\G
*************************** 1. row ***************************
        id: 113563
       url: http://book.douban.com/subject/2567698/
     title: 三体
      tags: 科幻|刘慈欣|三体|中国|科幻小说|小说|硬科幻|中国科幻
createtime: 2015-05-26 09:38:36
1 row in set (0.25 sec)
```

总的来说，豆瓣读书的页面对爬虫还是比较友好的。没有费很大功夫就爬到了数据，当然这只是一个用来练习的例子，一些“孤岛”书籍无法从爬取结果里筛选出链接来，也就没有抓取下来。


### 爬虫结构
__整个doubanbook爬虫的文件结构：__

```
|--doubanbook
|   |--__init__.py
|   |--pipelines.py
|   |--items.py
|   |--settings.py
|   |--spiders
|       |--__init__.py
|       |--DoubanBook.py
|   |--proxymiddlewares.py
|   |--sql_model.py
|   |--proxy.csv
|   |--uamiddlewares.py
|--scrapy.cfg
```

其中,  
__items.py__:抓取的结构化数据      
__pipelines.py__:item在spider中被收集后，会传递到Item PipeLine，在这里被存储进数据库     
__setttings.py__:定制Scrpy组件的设置，包括core, extension, pipleline及spider组件    
__DoubanBook.py__:爬虫代码，解析出抓取地址集，以及需要的结构化数据  
__proxymiddlewares.py__: 设置proxy的中间件  
__sql_model.py__:写入数据库的orm model，使用的是sqlalchemy  
__proxy.csv__:proxy的列表，通过[这篇笔记](http://izualzhy.cn/scrapy-httpproxy/)获取    
__uamiddlewares.py__:设置User-Agent的中间件    


### 爬取什么

我们还是以[三体](http://book.douban.com/subject/2567698/)这个链接为例，在这个页面我们需要获取两部分内容：  
1. 结构化数据，包括书名、标签   
2. 新的爬取地址  

具体xpath的查找方式略过，这里直接贴下结论：  
对内容1   

```
# 书名
//*[@id="wrapper"]/h1/span/text()
# 标签
//*[@id="db-tags-section"]/div/span/a/text()  
```

对内容2  
可以看到所有图书的链接有这样的形式:`.*book.douban.com/subject/\d+`，我们获取所有链接标签的内容，通过正则筛选出来。  

如上所述，DoubanBook.py的代码为： 

```
# -*- coding: utf-8 -*-
import scrapy
from doubanbook.items import DoubanbookItem


class DoubanbookSpider(scrapy.Spider):
    name = "DoubanBook"
    # allowed_domains = ["read.douban.com/ebook/9580262"]
    start_urls = (
        'http://book.douban.com',
    )

    def parse(self, response):
        sel = scrapy.Selector(response)
        url = response.url
        title = sel.xpath('//*[@id="wrapper"]/h1/span/text()').extract()
        tags = sel.xpath('//*[@id="db-tags-section"]/div/span/a/text()').extract()
        if title:
            yield DoubanbookItem(url=url, title=title[0], tags=tags)

        new_books = sel.xpath('//a/@href').re('.*book.douban.com/subject/\d+')
        for book in new_books:
            yield scrapy.Request(url=book, callback=self.parse)
```

### 如何爬取

上述代码在爬取少量地址时是没有问题的，但是在需要爬取大量数据时爬虫面临被禁止的问题，比如可能会碰到这个[机器人页面](http://www.douban.com/misc/sorry?original-url=http%3A%2F%2Fbook.douban.com%2Fsubject%2F1986291%2F)，这个时候就需要换代理、UA等方法了，这里也主要介绍下如何在爬取时随机选取代理、以及使用User-Agent。

首先在settings.py里指定：

```
# -*- coding: utf-8 -*-

# Scrapy settings for doubanbook project
#
# For simplicity, this file contains only the most important settings by
# default. All the other settings are documented here:
#
#     http://doc.scrapy.org/en/latest/topics/settings.html
#

BOT_NAME = 'doubanbook'

SPIDER_MODULES = ['doubanbook.spiders']
NEWSPIDER_MODULE = 'doubanbook.spiders'

ITEM_PIPELINES = {
    'doubanbook.pipelines.DoubanbookPipeline': 999,
    'scrapy.contrib.downloadermiddleware.httpproxy.HttpProxyMiddleware': 110,
    'doubanbook.proxymiddlewares.ProxyMiddleWare': 100,
    'doubanbook.uamiddlewares.UserAgentMiddleware': 400,
    'scrapy.contrib.downloadermiddleware.useragent.UserAgentMiddleware': None,
}

DOWNLOAD_DELAY = 0.5
RANDOMIZE_DOWNLOAD_DELAY = True
COOKIES_ENABLED = False
USER_AGENT = 'Mozilla/5.0 (Windows NT 6.1; WOW64)\
    AppleWebKit/537.36 (KHTML, like Gecko) Chrome/42.0.2311.152 Safari/537.36'
PROXY_LIST = './doubanbook/proxy.csv'
UA_LIST = [
    "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.1 "
    "(KHTML, like Gecko) Chrome/22.0.1207.1 Safari/537.1",
    "Mozilla/5.0 (X11; CrOS i686 2268.111.0) AppleWebKit/536.11 "
    "(KHTML, like Gecko) Chrome/20.0.1132.57 Safari/536.11",
    "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/536.6 "
    "(KHTML, like Gecko) Chrome/20.0.1092.0 Safari/536.6",
    "Mozilla/5.0 (Windows NT 6.2) AppleWebKit/536.6 "
    "(KHTML, like Gecko) Chrome/20.0.1090.0 Safari/536.6",
    "Mozilla/5.0 (Windows NT 6.2; WOW64) AppleWebKit/537.1 "
    "(KHTML, like Gecko) Chrome/19.77.34.5 Safari/537.1",
    "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/536.5 "
    "(KHTML, like Gecko) Chrome/19.0.1084.9 Safari/536.5",
    "Mozilla/5.0 (Windows NT 6.0) AppleWebKit/536.5 "
    "(KHTML, like Gecko) Chrome/19.0.1084.36 Safari/536.5",
    "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/536.3 "
    "(KHTML, like Gecko) Chrome/19.0.1063.0 Safari/536.3",
    "Mozilla/5.0 (Windows NT 5.1) AppleWebKit/536.3 "
    "(KHTML, like Gecko) Chrome/19.0.1063.0 Safari/536.3",
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_8_0) AppleWebKit/536.3 "
    "(KHTML, like Gecko) Chrome/19.0.1063.0 Safari/536.3",
    "Mozilla/5.0 (Windows NT 6.2) AppleWebKit/536.3 "
    "(KHTML, like Gecko) Chrome/19.0.1062.0 Safari/536.3",
    "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/536.3 "
    "(KHTML, like Gecko) Chrome/19.0.1062.0 Safari/536.3",
    "Mozilla/5.0 (Windows NT 6.2) AppleWebKit/536.3 "
    "(KHTML, like Gecko) Chrome/19.0.1061.1 Safari/536.3",
    "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/536.3 "
    "(KHTML, like Gecko) Chrome/19.0.1061.1 Safari/536.3",
    "Mozilla/5.0 (Windows NT 6.1) AppleWebKit/536.3 "
    "(KHTML, like Gecko) Chrome/19.0.1061.1 Safari/536.3",
    "Mozilla/5.0 (Windows NT 6.2) AppleWebKit/536.3 "
    "(KHTML, like Gecko) Chrome/19.0.1061.0 Safari/536.3",
    "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/535.24 "
    "(KHTML, like Gecko) Chrome/19.0.1055.1 Safari/535.24",
    "Mozilla/5.0 (Windows NT 6.2; WOW64) AppleWebKit/535.24 "
    "(KHTML, like Gecko) Chrome/19.0.1055.1 Safari/535.24"
    ]
```

这里使用的是中间件的方法，下载器中间件是介于Scrapy的request/response处理的钩子框架。 是用于全局修改Scrapy request和response的一个轻量、底层的系统。更多信息可以参考[这里](http://scrapy-chs.readthedocs.org/zh_CN/0.24/topics/downloader-middleware.html)。其中我们需要编写的是

_doubanbook.proxymiddlewares.ProxyMiddleWare_: 随机选取代理  
_doubanbook.uamiddlewares.UserAgentMiddleware_:  随机选取UA  
_doubanbook.pipelines.DoubanbookPipeline_: 存储抓取到的目标数据  

注意settings.py还有很多其他设置，感兴趣的可以一一了解下，注意在这里也可以定义我们自己的设置，比如PROXY_LIST就是存放代理的文件，生成该文件的方法参考另外一篇[文章](http://izualzhy.cn/scrapy-httpproxy/)。

#### 代理中间件
设置代理的代码也很简单，从PROXY\_LIST里读取所有代理并随机选择一个：  

```
#!/usr/bin/env python
import csv
import random
from doubanbook.settings import PROXY_LIST


content = csv.reader(file(PROXY_LIST, 'rb'))
proxies = []
for proxy in content:
    proxies.append(proxy)


class ProxyMiddleWare(object):
    def process_request(self, request, spider):
        proxy = random.choice(proxies)
        proxy_str = 'http://' + proxy[1] + ':' + proxy[2]
        request.meta['proxy'] = proxy_str
```

#### UA中间件

设置UA的想法也是类似，从settings.py里读取所有UA，并随机选择一个

```
#!/usr/bin/env python

import random
from doubanbook.settings import UA_LIST


class UserAgentMiddleware(object):
    def process_request(self, request, spider):
        ua = random.choice(UA_LIST)
        if ua:
            request.headers.setdefault('User-Agent', ua)
```

### 如何存储
当Item在Spider中被收集之后，它将会被传递到Item Pipeline，一些组件会按照一定的顺序执行对Item的处理。

每个item pipeline组件(有时称之为“Item Pipeline”)是实现了简单方法的Python类。他们接收到Item并通过它执行一些行为，同时也决定此Item是否继续通过pipeline，或是被丢弃而不再进行处理。

以下是item pipeline的一些典型应用：

1. 清理HTML数据
2. 验证爬取的数据(检查item包含某些字段)
3. 查重(并丢弃)
4. 将爬取结果保存到数据库中

因此可以在这里数据落盘，我选择的是存储进mysql的方式，选用的是sqlalchemy来快速实现这点，先看下__pipelines.py__文件的内容：

```
# -*- coding: utf-8 -*-
from sql_model import get_session, DoubanBook

# Define your item pipelines here
#
# Don't forget to add your pipeline to the ITEM_PIPELINES setting
# See: http://doc.scrapy.org/en/latest/topics/item-pipeline.html


class DoubanbookPipeline(object):
    tag_map = {}

    def process_item(self, item, spider):
        with get_session() as session:
            doubanbook = DoubanBook()
            doubanbook.url = item['url']
            doubanbook.title = item['title'].encode('utf8')
            doubanbook.tags = '|'.join(item['tags']).encode('utf8')
            session.add(doubanbook)
            session.commit()
        return item
```

 session在sqlalchemy中类似于transcation的概念，具体看下sql_model.py这个文件：

```
#!/usr/bin/env python
# -*- coding: utf-8 -*-


from sqlalchemy import create_engine
from sqlalchemy import Column
from sqlalchemy.orm import sessionmaker
from sqlalchemy.sql import text
from sqlalchemy.types import CHAR, Integer, TIMESTAMP
from sqlalchemy.ext.declarative import declarative_base

BaseModel = declarative_base()
# DB连接配置
DB_CONNECT_STRING = 'mysql+mysqldb://root@127.0.0.1:3306/spider'
engine = None
DB_Session = None


def init_db_session():
    global engine
    engine = create_engine(DB_CONNECT_STRING, echo=False)
    global DB_Session
    DB_Session = sessionmaker(bind=engine)


def get_session():
    return MySession()


class MySession(object):
    def __init__(self, *arg, **warg):
        if not DB_Session:
            init_db_session()
            BaseModel.metadata.create_all(engine)
        self.session = DB_Session()

    def __enter__(self):
        return self.session

    def __exit__(self, *arg, **warg):
        self.session.close()


# 数据表对应ORM模型
class DoubanBook(BaseModel):
    __tablename__ = 'doubanbook'
    __table_args__ = {
        'mysql_engine': 'InnoDB'
    }

    id = Column(Integer, primary_key=True)
    url = Column(CHAR(256))
    title = Column(CHAR(256))
    tags = Column(CHAR(512))
    createtime = Column(TIMESTAMP, server_default=text('CURRENT_TIMESTAMP'),
                        nullable=False)
```

主要是`DB_CONNECT_STRING`这里改成自己的数据库就可以了，关于_sqlalchemy_以后可能会专门写下自己的心得笔记，这里使用非常清晰，不再赘述。


以上就是整个爬取的全部代码介绍了，接下来运行`scrapy crawl DoubanBook`，数据就会不断的爬取到数据库了。


因为本人也是现学现用现记录自己的使用心得，如果有其他问题，也都欢迎路过的朋友 留言说明。

### 参考资料
1. [scrapy官方文档](http://scrapy-chs.readthedocs.org/zh_CN/0.24/intro/overview.html)
