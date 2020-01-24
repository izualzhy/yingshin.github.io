---
title: "我是一个 op 之 supervisor"
date: 2019-05-26 10:09:01
tags: op supervisor
---

## 1. 简介

平时在开发机上，有一些进程比较稳定，基本不用考虑退出的场景，例如`python -m SimpleHTTPServer`，我习惯直接使用 nohup 放到后台执行。nohup 使用简单，但是如果进程需要从 stdin 读取输入，例如[agedu -w](https://www.chiark.greenend.org.uk/~sgtatham/agedu/)(显示磁盘使用容量)，无法使用 nohup 放到后台执行。

后台服务进程需要保证 7*24，因此对稳定性要求较高。进程无论是主动还是意外退出时，都需要一个自动拉起的机制，保证服务可用。supervisor 就是这样一个工具。

更进一步，supervisor 提供了非常方便的管理进程的能力，使得我们可以专注于实现后台服务功能本身，而不用关心如何管理服务进程。

## 2. 安装及配置

supervisor 是用 python 写的，可以直接`pip install supervisor`安装

安装完成后会生成3个工具：

1. `supervisord`负责管理用户进程，其本身以守护进程 + server 的形式存在  
2. `supervisorctl`作为 client 进程，能够与`supervisord`交互，执行查看/停止/启动进程等操作  
3. `echo_supervisord_conf`生成`supervisord`需要的默认配置  

同时在 /etc 目录下生成一个 supervisord.conf 和 supervisor.d 目录。

supervisord.conf 基本是开箱即用，主要是修改几个文件路径，确保拥有对应的写权限，如果修改不对，启动时也会给出明确的提示，按照提示修改即可。主要有

```
[unix_http_server]
file=/tmp/supervisord/supervisor.sock   ; (the path to the socket file)

[inet_http_server]         ; inet (TCP) server disabled by default
port=*:8013        ; (ip_address:port specifier, *:port for all iface)

[supervisord]
logfile=/tmp/supervisord/log/supervisord.log ; (main log file;default $CWD/supervisord.log)
pidfile=/tmp/supervisord/supervisord.pid ; (supervisord pidfile;default supervisord.pid)
childlogdir=/tmp/supervisord/log/childlogdir ; ('AUTO' child log dir, default $TEMP)

[supervisorctl]
serverurl=unix:///tmp/supervisord/supervisor.sock ; use a unix:// URL  for a unix socket
```

配置里`;`开头表示注释.

需要启动的进程在 supervisor.d 目录下，默认为 .ini 文件。

```
[include]
files = /etc/supervisor.d/*.ini
```

可以参考`[program:theprogramname]`的写法。

## 3. 示例

我们先编写一个测试用的程序

```cpp
#include <stdio.h>
#include <unistd.h>

bool running = true;

void sig_term_handler(int sig) {
    printf("recv SIGTERM, set running to false\n");
    running = false;
}

int main() {
    printf("enter main.\n");

    signal(SIGINT, sig_int_handler);
    signal(SIGTERM, sig_term_handler);

    while (running) {
        printf("running:%d\n", running);
        fflush(stdout);
        sleep(1);
    }

    printf("leave main.\n");

    return 0;
}
```

进程一直运行，每秒打印一条日志到 stdout，直到收到 SIGTERM 的信号，编译生成的二进制名为`test_test`。

对应的`test_test.ini`内容如下：

```
[program:test_test]
command=/home/users/izualzhy/Training/test/test_test
user=izualzhy
autostart=true
autorestart=true
startsecs=60
stopsignal=TERM
stopasgroup=true
ikillasgroup=true
startretries=1
redirect_stderr=true
stdout_logfile=/tmp/supervisord/test_test/run.log
stdout_logfile_maxbytes=1MB
stdout_logfile_backups=10
```

这里主要是配置 supervisor 启动的命令行(最好是绝对路径)，例子里没有打印日志而是使用标准输出，这里都重定向到了`run.log`。

`redirect_stderr=true`类似于`2 &> 1`的效果，因此标准错误输出也是相同路径，

准备就绪后使用`supervisord -c /etc/supervisord.conf`启动，可以观察到`test_test`作为`supervisord`的子进程存在。

```
$ ps -eo pid,ppid,command | grep test
22195 11475 /home/users/izualzhy/Training/test/test_test
$ ps -eo pid,ppid,command | grep supervisor
11475     1 /bin/supervisord -c supervisord.conf
```

观察`run.log`也可以确认`test_test`进程的状态。

## 4. supervisorctl

`supervisord`启动后，可以使用 supervisorctl 控制，例如更新配置、重启进程等，使用 help 查看完整命令，例如我们可以重启`test_test`。

```
$ supervisorctl
test_test                        RUNNING   pid 22195, uptime 1:36:19
supervisor> restart test_test
test_test: stopped
test_test: started
```

`run.log`可以观察到对应的输出:

```
running:1
recv SIGTERM, set running to false
leave main.
enter main.
running:1
```

这里也可以验证我们前面设置的`stopsignal`，作用上为`kill -SIGTERM`

## 5. Web Server

Web Server 提供了一个可视化简版的 supervisorctl，对应`inet_http_server`的配置，supervisord 启动后也会默认生成一个支持浏览器访问的 http 服务。

![starting](/assets/images/supervisor/starting.png)

![running](/assets/images/supervisor/running.png)

## 6. superlance

supervisor 提供了进程管理的功能，当一个进程退出时能够被重新拉起。

而当一个进程非预期退出时，最好也能够同时邮件或者短信出来。或者当端口异常、内存过高时，也能够通知出来。superlance 通过一系列的工具集提供了这些功能。

superlance 同样可以使用`pip install superlance`安装。

修改 supervisord.conf，增加 EventListener

```
[eventlistener:crashmail]
command=crashmail -a -s "sendEmail -f xxx@xxx.com -t xxx@xxx.com -u warning-subject -s xxx" -m xxx@xxx.com
events=PROCESS_STATE_EXITED
```

重新 reload，新增了 crashmail 进程

```
$ supervisorctl
test_test                        RUNNING   pid 16358, uptime 0:47:48
supervisor> reload
Really restart the remote supervisord process y/N? y
Restarted supervisord
supervisor> status
crashmail                        RUNNING   pid 12205, uptime 0:00:01
test_test                        STARTING
```

kill test_test 后，就会收到程序退出的邮件。

## TIPS

1. supervisor 不支持后台进程，`command`需要阻塞在前台启动。  

## 参考资料：

1. [Supervisor: A Process Control System
](http://www.supervisord.org/index.html)  
2. [DAEMON-IZE YOUR PROCESSES ON THE CHEAP, PART TWO: SUPERVISOR](https://ryanmckern.com/2013/01/daemon-ize-your-processes-on-the-cheap-part-two-supervisor/)  
3. [crashmail Documentation
](https://superlance.readthedocs.io/en/latest/crashmail.html)  
4. [supervisor管理进程 superlance对进程状态报警](https://www.cnblogs.com/binglansky/p/9246780.html)
