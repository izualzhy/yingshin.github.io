---
title:  "jekyll-bootstrap折腾记"
date:   2014-3-4 16:26:50
excerpt: "记录使用jekyll-bootstrap搭建博客的过程"
tags: blog
---

想自己弄一个博客很久了，因为一直以来都在使用GitHub保存一些[文件](https://github.com/yingshin)。
偶然间看到了GitHub上搭建博客的文章介绍，就想自己实践一番。在此之前，可以说对html,css,javascript,liquid,markdown知道的很少。
这几天下来，是一边看w3scholl经常google一边动手的过程，html,javascript,简单的css样式都用几个小时的时间过了一遍，接下来
还想多了解下css,HTML DOM, JQuery。

废话不多说，说下搭建这个博客的过程，包括其中的一些错误，希望对偶尔来到这里的朋友有所帮助。所有的文件地址在[这里](https://github.com/yingshin/yingshin.github.io)。

最开始是看到了这篇[文章](http://www.ruanyifeng.com/blog/2012/08/blogging_with_jekyll.html)，不知道是不是等的时间太短的原因。
总之建立分支的办法不用这么麻烦，直接在GitHub上建立一个username.github.io的仓库即可。同时建议不要点击Settings里的
Automatic Page Generator。一个文件一个文件的自己创建，可以比较清楚对jekyll来讲，每个文件的作用。  

<!--more-->  

最简单的页面是在仓库下创建一个index.html文件，文件名不要弄错。写入  

```
Hello World!
```  

上传，等待10分钟左右（只有第一次push需要这样），然后到username.github.io就可以查看这个网页了。

jekyll需要的就是这个index.html,这个是最基本的。明白了之后，就可以看jekyll的[doc](http://jekyllrb.com/docs/home/)了。
注意QuickStart里的  

```
jekyll new myblog
```

会产生jekyll需要的一系列文件、文件夹(包括index.html),具体每个文件的作用可以参考上面的[链接](http://jekyllrb.com/docs/home/),花不了很多时间
就可以浏览并操作一遍了。  

之前我是没有做过前端的，在写index.html, default.html的过程中，接触到了[BootStrap](http://getbootstrap.com/css/),其中bootstrap也经历了先使用v2，后使用v3的过程，
虽然过程比较折腾，但是学到了很多。BootStrap提供了很多css样式，可以直接看它的[例子](http://getbootstrap.com/getting-started/#examples),遇到不明白的也可以单独查看
[css组件](http://getbootstrap.com/css/)。使用之后可以尽快的将一个简单的界面呈现出来。包括也找了几个博客页面，看了下css的写法，觉得不错的，就抄了过来。  

导航栏，侧边栏，翻页控件都一一加了上来，注意有很多实用的[变量](http://jekyllrb.com/docs/variables/),jekyll都预先有定义。
这里使用的是[Liquid语法](https://github.com/shopify/liquid/wiki/liquid-for-designers)，需要大概了解一下，用到了慢慢查，循环，赋值等操作在html里使用可以取得事半功倍的效果。

接下来就是正式发表了，文章在\_post文件夹下就行,文件名需要遵循一定的格式  

```
YEAR-MONTH-DAY-title.MARKUP
YEAR-MONTH-DAY-title.MARKDOWN
```  

同时需要了解下[MarkDown的语法](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet)，很有意思，可以很快的写出一篇文章来。
使用MarkdownPad可以实时的查看下是否达到了想要的效果。

当然过程里遇到了很多问题，google帮了大忙，包括stackoverflow上的一些问答，非常直接的找到了问题的答案。
