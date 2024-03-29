---
title: "单线程的习惯 - 评《单核工作法图解》"
date: 2018-12-22 21:43:08
excerpt: "单线程的习惯 - 评《单核工作法图解》"
tags: read
---

《单核工作法图解》这本书的作者即《番茄工作法》的作者，在我看来是番茄的升级版。

在看这本书之前，我对这些效率一类的书籍一直不感兴趣，觉得连阅读这些书籍本身都是在浪费时间、降低效率。我一直秉持着专注即高效的理念，早上到公司，第一时间看订阅的 rss 文章，标记下感兴趣的文章，接着收邮件，过一眼几个监控列表。中午饭后开始阅读标记的文章。其余时间，努力做的就是专注本身，比如写代码、wiki、cr、讨论等等。不过今年下半年以来，琐事逐渐增多，有一个深刻的感觉就是：

**工作时间变得碎片化**

因此效率越来越低，这是开始读《单核工作法图解》这本书的背景。

感觉西方人还是很有意思的，这类书籍很多，条条框框能写出一大堆来，而按照中国传统的思想，这些都属于“心学”的范畴？可遇而不可求，难以书面化，主要依赖你自己去领悟。就像很多古老的技术，例如陶瓷，都是师傅徒弟口口相传，赶上几代徒弟都天资愚钝，这个手艺基本就没落了，因为我们重“道”而对一些奇技淫巧嗤之以鼻。而西方则讲究方法论，也就是“术”，重在如何工厂化，可实践可推广。

拖延症到底有什么原因？一个事情的优先级，严格定义起来居然也分为多种。披着紧急外衣的任务，怎么分辨是哪种紧急。接触一些新的概念(服务生效应、康奈尔笔记法、艾维-李方法、手边管理法等)，再对比自己平时的工作种种，受益良多。

我相信只有遇到了效率问题，或者读过这类书籍，才会翻开这本书。这篇笔记里，我尽量只讲清楚基本概念，不去罗列名词、方法和简单的摘抄，因为只有花上一上午，读完这本书，结合自己遇到的种种，融会贯通，然后在平时工作里实践，一定能收获很多。

这也是推荐这本书最主要的原因。

## 1. 什么是单核工作法？

五项基本概念：

1. 快捷清单：存放当前5项最重要的任务  
2. 单核时段：专心处理快捷清单上的一项任务  
3. 全景闹钟：分针的下一个位置，例如 9:00,9:30,10:00,10:30  
4. 全景时间段，思考‘拉金问题’：此时此刻，我的时间最好用来干什么  
5. 颠倒优先级：避免紧急任务排到重要任务之前  

这些概念连起来就是单核工作法：

**按照重要程度，把当前至多5项重要任务写到快捷清单上，然后基于当前时间设置下一个全景闹钟(至少间隔25分钟)，开启自己的单核时段，专注做清单上的一件事。**

![monotasking](/assets/images/monotasking/monotasking.jpeg)

接下来，就是把这些概念，分开了揉碎了翻来覆去的讲，分为6个章节。

## 2. 削减待办任务

1. 忙碌谬论：不是事情越多，人就越有价值。今年上半年，遇到任务很多，单枪匹马搞来搞去，时间久了，才发现根本忙不过来，到时反而成为别人项目的瓶颈。  
2. 快捷清单别超过5项  
3. 集草器清单：记录一些没有拒绝，但是不重要紧急的事情，例如有个模块挂了1个月了，突然找过来，基本就是不着急，列到集草器清单里，包括目标、利益关系人、进入清单的日期  
4. 除草：除草就是干掉集草器清单的任务，或者尽快干掉，或者加入到快捷清单，或者直接删掉。  

大部分工作，时间分为可支配时间 和 不可支配时间。可支配时间里，采用单核工作法。不可支配，那就尽量在日程里划出来，或者定一个固定时间，例如我厂hi上常常可见的(xx时间hi不回复，xx时间统一过单子，急事电话)(当然实际上是胡扯，不打电话根本不给过，效率极低)。

有效的说"不"，口是心非，害人害己，口头答应，反而让对方有了期待，到时间做不成，只会更不愉快。

该退出时退出，不要浪费时间，原因仅仅是为了让利益关系人满意，还是给自己一个心安理得的慰藉？

![monotasking](/assets/images/monotasking/done_todo.jpeg)

## 3. 现在专注一件事

戒绝通知，手机拿远点，手边只留跟接下来要做的事情有关的物件。

无论你多牛逼，多任务一定会降低你的优先级。我的经验，例如跑个脚本确实要花很长时间，最好同时进行。那就对多个线程都在纸上记录下进度，避免混乱。

当然，尽量不要切换任务，因为一定会降低效率，保持单核工作。

专注分很多种：

> 以下是五种不同的注意力类型，难度依次升高。[14]
>  
>（1）集中性注意力：能够留意到周围环境中的声音或其他事件。  
（2）持续性注意力：专心处理一项且仅此一项任务，例如给客户写一封电子邮件。  
（3）选择性注意力：控制自己在一项任务上保持专注，不受无关想法和周围事情的影响。  
（4）转换性注意力：在两项相关的事情上快速切换任务，例如一边在课堂上听讲，一边对感兴趣的内容做笔记。  
（5）分散性注意力：同时做两件事，也叫多任务处理。想要成功做到的话，其中必须包括一项自动任务，例如走路。如果真的试图专注于两件无关的事情，等于还是在任务间转换。[15]  

单核工作法强调第3种。不要尝试做一些鸡毛蒜皮的小事，来获得多巴胺奖励，觉得自己有所作为，好像很忙碌，要专注在全景时间段判断出的重要任务上。

听点镇静型音乐，最好别带歌词的，有助于专注。

![focus](/assets/images/monotasking/focus.jpeg)

## 4. 永不拖延

我们评估自己的忙碌程度时，会认为近期超忙，远期不那么忙，这是思维的误区。

>造成拖延的原因多种多样。别人想让我做这件事，但是对我有什么好处？这个任务太庞大，我应该从哪里开始？我担心结果不够好，所以就不交付。我太累了，需要休息。我的事太多，分身乏术。


![never_delay](/assets/images/monotasking/never_delay.jpeg)

## 5. 循序渐进

任务太大，那就分解任务。分解任务之前，考虑好这件事的终极目的。

紧急会有很多种，我只列举下概念，不解释，你看下是否能自己解释出来：

失去机会的紧急、艰难成就的紧急、竞争的紧急、稀缺的紧急、时间用于的紧急。

侯世达定律宣城，完成任务时间永远比期待的更长，事先100%预估时间是不可能的。

>假设有四个商人，他们都同意工程A是顶级优先，B其次，C再次，最后是D；然而他们的分配方式却大相径庭。
※ 相对优先级：吉姆觉得应该同时进行全部四个工程，但是优先级更高的工程要投入更多的精力。
※ 满溢优先级：乔治觉得应该把所有精力投入到优先级最高的工程，一直到它完成。然后剩下的时间才能给次级优先的工程。
※ 均平优先级：杰克觉得所有工程要同时进行，而且要投入同样多的精力，除非工程之间出现冲突。一旦发生冲突，A比其他三个工程优先，以此类推。
※ 完成优先级：约翰觉得优先级问题只有根据工程完成的快慢来讨论才有意义。

我发现我经常陷入完成优先级，而实际上大佬们思考的总是满溢优先级。（读这本书之前，真的没有想过优先级可以这么细分，而且很有道理）

![step_by_step](/assets/images/monotasking/step_by_step.jpeg)

## 6. 简化协作

稀缺心态者以零和的思维模式对待生活。ta们相信，如果ta要赢，其他人必须输才行。跨组跨部门是这样，KPI会导致这样，防止工作时被忽悠了优先级，不知道OKR推广后会不会好一点。

协作避免不了沟通，在信息的丰富程度上，在白板前面对面沟通时信息程度最为丰富，发送文字信息是最慢的。

梯级顺序是这样的：

1. 照章办事  
2. 单向文本，例如电子邮件  
3. 即时消息  
4. 打电话  
5. 视频会议  
6. 面对面  
7. 面对面配合使用白板等绘图工具  

数字越大信息越丰富但是花的力气也越大，我厂的会议室经常订不到，可见大家现在做事情都接受不了信息太少。面对面的问题是有时候根本找不到对方，这个时候只能找管理层协助了。而管理层常犯的一个错误是：当看到这个问题时，找很多人到会议室，集中对清楚，case study等，会认为事情为什么没有这样早点解决。殊不知这种形式的沟通，普通一线员工想要这么做，花的力气很大。

![cooperation](/assets/images/monotasking/cooperation.jpeg)

## 7. 给创意充电

平时多吃素、增加工作之外的活动。

加班会有恶性循环的！

我之前总是想找到一种笔记方式，能够随时随地满足需求，但是发现很难，因为有时候需要发散去思考，有时候需要线性去总结。书里说我们的大脑喜欢在概念之间建立连接，可惜顺序结构的笔记和线性思维方式都做不到这一点。

![creativity](/assets/images/monotasking/creativity.jpeg)

## 8. 总结

这种书籍，我相信一定是见仁见智的，读的过程，难免会把自己代入，之前可能只是吐槽或者纳闷时间都去哪了，读完这本书后总能找到其中的一个相似场景，因此这篇笔记也是尽量列了自己有感悟的部分。

周末下午，泡上一杯红茶，阳台上搬个凳子，用几个小时的时间读完这本书，然后尝试应用，让自己接下来几十年的工作生涯受益，不亦乐乎？
