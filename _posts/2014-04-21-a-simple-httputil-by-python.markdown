---
title:  "一个简易的GET/POST工具"
date: 2014-04-21 11:03:18
excerpt: "一个简易的GET/POST工具，可以设置request参数，类型，cookie等，使用python。"
tags: python
---

经常需要测试某个服务器的API，无外乎以下几种需求：    
1. request type: Get/Post   
2. 设置request params或者queries   
3. 设置cookie   

因此自己拿python写了一个简易版的，平时用来开发测试：   
![simple httputil using python](/assets/images/simple_httputil_python.jpg)

<!--more-->

用到的库也非常常见    
网络请求用的是`urllib2`    

```
#!/usr/bin/python

import urllib,urllib2
import cookielib
import Cookie

class HttpUtil(object):
    def __init__(self):
        self.data = {}
        self.cookie = Cookie.SimpleCookie()
        self.url = ''
        self.method = 'GET'
        self.content = ''

    def set_url(self, url):
        self.url = url
    def set_param(self, key, value):
        self.data[key] = value
    def set_method(self, method):
        self.method = method
    def set_cookie(self, key, value):
        self.cookie[key] = value

    def set_params(self, text):
        params = text.split('&')
        print params;
        for item in params:
            param_pair = item.split('=')
            if len(param_pair) == 2:
                self.set_param(param_pair[0], param_pair[1])

    def set_cookies(self, text):
        cookies = text.split(';')
        for item in cookies:
            cookie_pair = item.split('=')
            if len(cookie_pair) == 2:
                self.set_cookie(cookie_pair[0], cookie_pair[1])

    def do_request(self):
        params = urllib.urlencode(self.data)
        if self.cookie:
            h = urllib2.HTTPHandler(debuglevel = 1)
            opener = urllib2.build_opener(h)
            for key in self.cookie:
                opener.addheaders.append(('Cookie', '%s=%s' % (key, self.cookie[key])))
            urllib2.install_opener(opener)
        if self.method == 'GET':
            new_url = self.url + '?' + params 
            print new_url
            print self.cookie
            self.content = urllib2.urlopen(new_url).read()
        elif self.method == 'POST':
            req = urllib2.Request(self.url, params)
            self.content = urllib2.urlopen(req).read()


            
```

图形界面用的是`Tkinter`    

```
#!/usr/bin/python
from Tkinter import *
import HttpUtil

#a simple util to set webreqeust and get result visiually.
#author: izualzhy.cn/com
class MainWindow(object):
    def __init__(self):
        self.httpUtil = HttpUtil.HttpUtil()

        root = Tk()
        self.url_label = Label(root, text = 'URL: ')
        self.url_label.grid(row=0, column=0)
        self.url_entry = Entry(root)
        self.url_entry.grid(row=0, column=1, columnspan=3)

        self.param_label = Label(root, text = "PARAM: ")
        self.param_label.grid(row=1, column=0)
        self.param_key_entry = Entry(root)
        self.param_key_entry.grid(row=1, column=1)
        self.param_value_entry = Entry(root)
        self.param_value_entry.grid(row=1, column=2)
        self.param_add_button = Button(root, text = "Add Param", command = self.AddParamCallBack)
        self.param_add_button.grid(row=1, column=3)

        self.cookie_label = Label(root, text = "COOKIE: ")
        self.cookie_label.grid(row=2, column=0)
        self.cookie_key_entry = Entry(root)
        self.cookie_key_entry.grid(row=2, column=1)
        self.cookie_value_entry = Entry(root)
        self.cookie_value_entry.grid(row=2, column=2)
        self.cookie_add_button = Button(root, text = "Add Cookie", command = self.AddCookieCallBack)
        self.cookie_add_button.grid(row=2, column=3)

        self.param_text = Text(root)
        self.param_text.grid(row = 3, column = 0, rowspan = 1, columnspan = 4)
        self.cookie_text = Text(root)
        self.cookie_text.grid(row = 4, column = 0, rowspan = 1, columnspan = 4)

        self.get_button = Button(root, text = "GET", command = self.GetCallBack)
        self.get_button.grid(row = 5, column = 0)
        self.post_button = Button(root, text = "POST", command = self.PostCallBack)
        self.post_button.grid(row = 5, column = 1)
        self.clear_param_button = Button(root, text = "ClearParam", command = self.ClearParamCallBack)
        self.clear_param_button.grid(row = 5, column = 2)
        self.clear_cookie_button = Button(root, text = "ClearCookie", command = self.ClearCookieCallBack)
        self.clear_cookie_button.grid(row = 5, column = 3)

        self.result_text = Text(root)
        self.result_text.grid(row = 6, column = 0, rowspan = 5, columnspan = 4)
        root.mainloop()

    def AddParamCallBack(self):
        if (self.param_text.get(0.0, END)):
            self.param_text.insert(INSERT, '&')
            self.param_text.insert(INSERT, '%s=%s' %(self.param_key_entry.get(), self.param_value_entry.get()))

    def AddCookieCallBack(self):
        if (self.cookie_text.get(0.0, END)):
            self.cookie_text.insert(INSERT, ';')
            self.cookie_text.insert(INSERT, '%s=%s' %(self.cookie_key_entry.get(), self.cookie_value_entry.get()))

    def ClearParamCallBack(self):
        self.param_text.delete(0.0, END)

    def ClearCookieCallBack(self):
        self.cookie_text.delete(0.0, END)

    def SetParmas(self):
        self.httpUtil.set_url(self.url_entry.get())
        self.httpUtil.set_params(self.param_text.get(0.0, END))
        self.httpUtil.set_cookies(self.cookie_text.get(0.0, END))

    def GetCallBack(self):
        self.SetParmas()
        self.httpUtil.set_method("GET")
        self.httpUtil.do_request()
        self.result_text.delete(0.0, END)
        self.result_text.insert(INSERT, self.httpUtil.content)
        print self.httpUtil.content

    def PostCallBack(self):
        self.SetParmas()
        self.httpUtil.set_method("POST")
        self.httpUtil.do_request()
        self.result_text.delete(0.0, END)
        self.result_text.insert(INSERT, self.httpUtil.content)
        print self.httpUtil.content

if __name__ == '__main__':
    mainWindow = MainWindow()
```

