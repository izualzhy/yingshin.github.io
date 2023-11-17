---
title:  "hadoop环境搭建笔记"
date: 2015-08-12 20:02:53
excerpt: "hadoop环境搭建笔记"
tags: mixed
---

关于如何搭建hadoop环境的文章网上有很多，大同小异。这里记录下自己搭建hadoop环境的过程。

<!--more-->

### Hadoop简介

Hadoop有两部分组成:分布式文件系统HDFS(Hadoop Distributed Filesystem)和MapReduce(Google MapReduce的开源实现)。

对于Hadoop的集群来讲，可以分成两大类角色：__Master__和__Salve__。一个HDFS集群是由一个__NameNode__和若干个__DataNode__组成的。其中NameNode作为主服务器，管理文件系统的命名空间和客户端对文件系统的访问操作；集群中的DataNode管理存储的数据。MapReduce框架是由一个单独运行在主节点上的__JobTracker__和运行在每个从节点的__TaskTracker__共同组成的。主节点负责调度构成一个作业的所有任 务，这些任务分布在不同的从节点上。主节点监控它们的执行情况，并且重新执行之前的失败任务；从节点仅负责由主节点指派的任务。当一个Job被提交时，JobTracker接收到提交作业和配置信息之后，就会将配置信息等分发给从节点，同时调度任务并监控TaskTracker的执行。
从上面的介绍可以看出，HDFS和MapReduce共同组成了Hadoop分布式系统体系结构的核心。__HDFS在集群上实现分布式文件系统，MapReduce在集群上实现了分布式计算和任务处理。HDFS在MapReduce任务处理过程中提供了文件操作和存储等支持，MapReduce在HDFS的基础上实现了任务的分发、跟踪、执行等工作，并收集结果，二者相互作用，完成了Hadoop分布式集群的主要任务。__


下面介绍下搭建过程：

### 机器准备
3台机器：1台Master + 2台Slave.  
Master机器是NameNode + JobTracker的角色，负责总管分布式数据和分解任务的执行。  
Slave机器是DataNode + TaskTracker的角色，负责分布式数据存储以及任务的执行。   

### 建立ssh信任关系
配置信任关系是为了可以无密码登陆  
执行`ssh-keygen -t rsa`一路回车，在~/.ssh下会生成:id\_rsa.pub,authorized\_keys文件。  
对每个机器执行该操作后，把所有机器的id\_rsa.pub合并为一个临时文件，将该临时文件追加到每个机器的authorized\_keys文件后就可以了。
#### 验证
从任意机器ssh到其他机器实验下，如果不需要输入密码，则成功。如果需要输入密码，检查下.ssh authorized\_keys的文件权限。

### JDK环境
#### 安装
下载jdk安装包, 安装jdk，解压jdk.xx.tar.gz文件即可。注意解压后的路径。  
#### 修改环境变量
修改~/.bashrc 文件:   

```
JAVA_HOME=刚才的解压路径
PATH=$JAVA_HOME/bin:$JAVA_HOME/jre/bin:$PATH
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JAVA_HOME/lib:$JAVA_HOME/jre/lib:$JAVA_HOME/jre/lib/rt.jar:$CLASSPATH
```

之后`source ~/.bashrc`生效。  

#### 验证
验证下是否安装成功：  

```
public class test {
    public static void main(String args[]) {
        System.out.println("A new jdk test!");
    }
}
```

将上述代码存为test.java，然后运行`javac test.java`会在当前目录下生成`test.class`文件，再执行`java test`如果能够输出`A new jdk test!`则表明java安装成功了。

在其他机器上重复该步骤，注意路径保持一致。

### Hadoop安装
#### 安装
下载hadoop安装包，我放了一个在[百度云](http://pan.baidu.com/s/1eQhMCfC)上，解压。   

#### 修改环境变量
修改~/.bashrc文件:

```
HADOOP_HOME=刚才的解压路径
PATH=$HADOOP_HOME/bin:$PATH
```
#### 配置
修改hadoop-env.sh   

位于conf目录下
修改`export JAVA_HOME=之前java的解压路径`

hadoop代码开发分为了core, hdfs, map/reduce三部分，对应的配置文件也被分成了三个:core-site.xml, hdfs-site.xml, mapred-site.xml.


修改core-site.xml  

```
<configuration>
<property>
<name>fs.default.name</name>
<value>hdfs://{master的IP}:9000</value>
<final>true</final>
</property>
<property>
<name>hadoop.tmp.dir</name>
<value>{hadoop的解压目录}/tmp</value>
<description> A base for other temporary directories</description>
</property>
</configuration>
```


修改 dfs-site.xml  

因为有两个slave，所以replication设置为2.

```
<configuration>
<property>
<name>dfs.name.dir</name>
<value>{hadoop的解压目录}/name</value>
<final>true</final>
</property>
<property>
<name>dfs.data.dir</name>
<value>{hadoop的解压目录}/data</value>
<final>true</final>
</property>
<property>
<name>dfs.replication</name>
<value>2</value>
<final>true</final>
</property>
</configuration>
```

修改 mapred-site.xml  

```
<configuration>
<property>
<name>mapred.job.tracker</name>
<value>{master的IP}:9001</value>
</property>
</configuration>
```

修改 masters, slaves   
内容分别修改为master主机名，所有的slave主机名。

#### 验证
执行命令`hadoop namenode -format`  
再执行`start-all.sh`

然后用jps查看，master上显示：  

```
16255 SecondaryNameNode
12143 Jps
16379 JobTracker
15941 NameNode
```

slave上显示：

```
15554 DataNode
28553 Jps
16125 TaskTracker
```

Hadoop就算安装成功了！
