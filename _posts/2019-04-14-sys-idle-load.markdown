---
title: "扯扯 cpu idle 与 load average"
date: 2019-04-14 19:06:03
tags: linux
---

最近值周遇到一个系统负载的问题，回溯了下问题大概已经持续了两个多月，这篇笔记记录下对于系统 CPU 负载的理解，不定期根据线上经历更新。

## 1. idle 与 load

idle 表示 cpu 的闲置程度，数值越大表示 cpu 负载越低。对应的，load average 表示 CPU 的利用率，系统记录了过去 1min/5min/15min 的记录。

通过 top 可以看到这两个数值：

![top](/assets/images/top-1.png)

更细的，可以看到每个 cpu 的 idle：

![top-each-cpu](/assets/images/top-each-cpu.png)

严格来讲，load average 是指运行队列的平均长度，也就是等待 cpu 的进程数。这些进程是系统中的活动进程，也就是处于 TASK_RUNNING or TASK_UNINTERRUPTIBLE 的进程数。

系统的进程状态有这么几种：

![process status](https://idea.popcount.org/2012-12-11-linux-process-states/76a49594323247f21c9b3a69945445ee.svg)

## 2. 飙高 load average 但 idle 正常

除了 TASK_RUNNING，我们可以使用 TASK_UNINTERRUPTIBLE 来模拟 load average 高的情况，写了一段测试代码：

```cpp
#include <stdio.h>
#include <string>
#include <vector>
#include <thread>

void foo(const std::string& args) {
    std::string contents;
    for (int i = 0; i < 1024; ++i) {
        contents.append("hello, world\n");
    }


    std::string file_name = "test_io";
    file_name.append(args);
    file_name.append(".data");
    while (1) {
        FILE* fp = fopen(file_name.data(), "w");
        fwrite(contents.data(), contents.size(), 1, fp);

        fclose(fp);
    }
}

int main() {
    std::vector<std::thread> threads;
    const int cores_count = 1000000;

    for (int i = 0; i < cores_count; ++i) {
        threads.push_back(std::thread(foo, std::to_string(i)));
    }

    for (int i = 0; i < cores_count; ++i) {
        threads[i].join();
    }

    return 0;
}
```

开启N多线程，都尝试去写文件，由于读写 hang 住时并不会消耗 cpu，当然调度太多线程也会使得 cpu 飙高。因此我们能够观察到 cpu 不那么高，但是 load average 飙到 2000+ 的场景：

![high-average-normal-idle](/assets/images/high-average-normal-idle.png)

因此，如果碰到 load average 飙高但 idle 正常的情况，注意观察是否有大量 IO 线程。

假设刚才编译出的二进制为 test_io

```
ps -eTo stat,pid,tid,ppid,comm | grep test_io | awk '{print $1}' | sort | uniq -c
   2380 Dl+
      5 Rl+
    289 Sl+
```

可以看到有很多 D 进程.

## 3. Zombie 进程是否导致异常

线上情况当时有一个干扰因素，是存在大量的Z进程，但是按照第一节的解释，Z进程不会导致系统负载的两个指标异常，例如我们构造一个产生5000+Z进程的场景：

```
#include "unistd.h"
#include "stdio.h"

int main() {
    for (int i = 0; i < 5000; ++i) {
        pid_t child = fork();

        if( child == -1  ) { //error
            perror("\nfork child error.");
        } else if(child == 0){
            return 0;
        }
    }

    sleep(1000);

    return 0;
}
```

![zombie](assets/images/zombie.png)

虽然存在 5000 zombie 进程，但系统负载和 idle 没有明显异常

## 4. average 和 idle 都很高的场景

[这里](https://yq.aliyun.com/articles/99312?t=t1)记录了一个 load average 和 cpu 都很高的场景，跟我们理解的常识确实是相反的。

发现原因是所有进程都调度到了单独一个 cpu 上。

此时可以用 vmstat 确认下:

![vmstat](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/9a0c040b24699d4128bbecae1af08b1d.png)

例如我当时看到的情况，比较明确是队列太多了

![vmstat-my](assets/images/vmstat-my.png)
