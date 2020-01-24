---
title:  "使用python获取IP代理列表"
date:   2014-04-16 15:33:03
excerpt: "Fetch active proxy list using python."
tags: proxy  python
---

无聊写点小例子玩玩，非常简单。  
抓取的网站地址为：`http://activeproxy.net/en/`   

主要就是urlopen获取内容，使用SGMLParser筛选代理数据，存储。  

最后的存储文件片断:  

```
 checktime                  country             proxyip      anonymous     speed      port   connect
     02:57                  Germany      94.249.168.231             no      1181      3128     0,006
     03:14                  Germany       93.180.156.21             no       807      3128     0,009
     18:05                  Germany      188.138.115.15             no       744      3128     0,009
     06:31                  Germany       91.250.83.172             no       578      3128     0,011
     00:30            United States          5.153.4.90             no       606      3128     0,013
     14:25                   France       91.121.58.235             no       490      3128     0,014
```

抓取用的python脚本:

```
#!/usr/bin/python

import os
import urllib2
from sgmllib import SGMLParser

url = 'http://activeproxy.net/en/%d/'

class ProxyParser(SGMLParser):
    def __init__(self):
        SGMLParser.__init__(self)
        self.proxies = []
        self.update()

    def update(self):
        self.tr = False
        self.td = False
        self.proxy = {}
        self.cur_attr = ''
        self.count = 0

    def start_tr(self, attr):
        self.tr = True 

    def end_tr(self):
        if (self.proxy):
            self.proxies.append(self.proxy)
            self.count += 1
        self.tr = False
        self.proxy = {}

    def start_td(self, attr):
        self.td = True 
        if self.tr:
            self.cur_attr = attr[0][1][4:]

    def end_td(self):
        self.td = False
        self.cur_attr = ''

    def handle_data(self, text):
        if self.tr and self.td and not text.startswith('\n'):
            self.proxy[self.cur_attr] = text


i = 1
proxyParser = ProxyParser()
while True:
    proxyParser.update()
    print 'proceeding...', url % (i)
    content = urllib2.urlopen(url % (i)).read()
    content = content.decode('gb2312').encode('utf-8')
    proxyParser.feed(content)
    print proxyParser.count, 'fetched.'
    i += 1
    if i > 100 or proxyParser.count != 20:
        break;


f = open('active_proxy', 'w')
str_format = '%10s%25s%20s%15s%10s%10s%10s' + os.linesep
if proxyParser.proxies:
    f.write(str_format % tuple(proxyParser.proxies[0].keys()))

for item in proxyParser.proxies:
    if len(item.values()) == 7:
        f.write(str_format % tuple(item.values()))

f.close()
```
