---
layout: post
title: "my hadoop2 notebook"
date: 2018-03-20 21:00:00
categories: 基础
tags: [fromSRE,Hadoop]
---

# Hadoop2 学习笔记
## 1.概述
### Hadoop的两个核心组成部分：
1）分布式文件系统-HDFS；  
2）分布式数据处理架构-MapReduce。MR功能实现了将单个任务打碎，并将碎片任务（Map）发送到多个节点上，之后再以单个数据集的形式加载（Reduce）到数据仓库。

#### 1.HDFS 两个版本的差异
+ 在一版本中，一个集群中仅有一个NameNode。一个集群中仅有一个SecondaryNameNode定期（checkpoint）保存NameNode数据，即SecondaryNameNode中存储的是上一次checkpoint之后的NameNode。checkpoint node默认在主节点，也可以放到其他节点上，但还是归主节点管理。对于单点NameNode坏了的话，可以通过SecondaryNameNode来恢复上一次checkpoint时候的数据，或者是写NameNode的时候就往第三方上写一份，用于备份恢复。  
+ 在二版本中，NameNode节点可以有备NameNode，也可以没有。

#### 2.MapReduce  
占用20%-30%磁盘空间  
Map：将任务分解为多个子任务执行  
Reduce：将这些子任务的处理结果进行汇总  
![Alt text](http://ww4.sinaimg.cn/large/0060lm7Tly1fpo91ssghjj30ld07675l.jpg
 "")

#### 3.Apache Hadoop 与 CDH
Hadoop1对应CDH3   
Hadoop2对应CDH4

#### 另，
与Spark相比，中间结果存储在磁盘中，而spark的中间结果存储在内存中，因此，spark更快

## 2.Hadoop1 架构
Hadoop1集群中的节点类型分为：
![Alt text](http://ww2.sinaimg.cn/large/0060lm7Tly1fpo96uut5lj30wo0fqmy9.jpg
"")
1）HDFS：
+ NameNode记录元数据（记录文件如何分割成数据块，以及存储数据块的数据节点的信息；对内存和I/O进行集中管理）
+ DataNode负责将HDFS数据块读写到本地文件系统
+ 当client端有读写请求时，NameNode会告诉client去哪个DataNode进行操作，<font color=#0099ff>client会直接与这个DataNode进行读写访问</font>

2）MR：
+ JobTracker负责为所执行任务（Map任务和Reduce任务）分配指定的节点主机，监控所允许的任务并进行重启。
+ TaskTracker需要与JobTracker进行定时心跳，若JobTracker不能准时获取TaskTracker的信息，则认为该TaskTracker节点坏了，JobTracker会将任务重新分配给其他从节点
+ <font color=#0099ff>client端和JobTracker通信，不直接和TaskTracker连接 </font>   

### 1.HDFS 结构
#### 1.1 NameNode
![Alt text](http://ww2.sinaimg.cn/large/0060lm7Tly1fpo9e1soinj30cb077mx7.jpg
"")
+ 在checkpoint的时候，edit文件内容会合并到fsimage中。在Hadoop1中，SecondaryNameNode负责checkpoint工作。checkpoint的具体过程如下：   

![Alt text](http://ww3.sinaimg.cn/large/0060lm7Tly1fpo9evs7n1j30fm0fcn03.jpg
"")
+ fsimage和edits文件都是经过序列化的，在NameNode启动的时候，它会将fsimage文件中的内容加载到内存中，之后再执行edits文件中的各项操作，使得内存中的元数据和实际的同步，存在内存中的元数据支持客户端的读操作。   
+ edits预写入式的日志，并且会接受执行成功或失败的结果。edits只记录对于元数据的改变的操作，读操作不记录。

#### 1.2 DataNode
HDFS机制将每个文件分割成一个或多个大小相等的数据块（Block，默认64M），每个Block会存储在一个或多个DataNode上，默认复制因子为3。如果某个节点坏了，NameNode发现数据块的拷贝数低于设定值，会增加数据块；当节点恢复，NameNode发现数据块的拷贝数高于设定值，会删除多余的。

#### 1.3 读HDFS操作
![](http://ww2.sinaimg.cn/large/0060lm7Tly1fpo9htoeu7j30l50faq45.jpg
)
#### 1.4 写HDFS操作
同步写，必须等3、4成功，才算写成功
![](http://ww4.sinaimg.cn/large/0060lm7Tly1fpo9igb9oqj30l60fejt2.jpg
)

### 2.MR 结构
#### 2.1 NameNode
+ TaskTracker定期向JobTracker发送心跳信号，这部分的数据流量非常小
+ 客户端应用主要与JobTracker和HDFS通讯，但不直接和TaskTracker进行通讯
+ MapReduce的主要网络数据流量是Shuffle阶段的TaskTracker产生的  

MapReduce按以下步骤执行应用程序：  
1）客户端向JobTracker提交一个应用程序  
2）JobTracker确定整个应用程序所需的处理资源：从NameNode获取所需的文件名和数据块的位置，计算处理所有数据所需的Map和Reduce任务数  
3）JobTracker查看所有从节点的状态；并将要执行的Map和Reduce任务排队等待  
4）当从节点有可用Slot时，Map任务将被部署到这个从节点。Map任务使用的数据是存储同一个从节点的  
5）JobTracker监控任务的进度；如果任务失败或者节点出错，任务会在下一个可用的Slot上重启；如果同一任务失败4次（默认值），该Job就失败了  
6）Map任务完成后，Reduce任务处理Map产生的中间结果  
7）Reduce任务将结果返回客户端  

对于第4点，HDFS以数据块形式存储数据，不关心数据块的内容。这样如果Map处理的数据记录跨越了两个数据块，如何处理？为此，HDFS引入了Input Split概念，以对数据块中的数据进行逻辑表示。如下：
![](http://ww2.sinaimg.cn/large/0060lm7Tly1fppcgfyktej30m708xjs1.jpg
)
如果数据记录位于一个数据块中，Input Split可表示完整的数据记录集；若数据记录跨两个数据块，Input Split中会包含第二个数据块的位置以及所需完整数据的偏移量。  
Reduce是处理Map后的结果，具体在哪个节点（a/b/c/.....）上处理，是由MR分资源确定的，对我是透明的  

## 3.Hadoop1 环境搭建
两台为例，
![](http://ww1.sinaimg.cn/large/0060lm7Tly1fppcif9nwsj30kt0dz767.jpg
)  

ip | hostname | 角色 | 进程
----|------|----|----
10.221.155.43 | vm-kvm10094-app  | master  | Namenode SecondaryNamenode JobTracker
10.221.155.44 | vm-kvm10095-app  | slave  | Datanode TaskTracker  

安装hadoop，hadoop-1.2.1，解压安装在/opt/app/hadoop。
安装jdk1.7
两台机器配互信
hadoop的配置文件在/opt/app/hadoop/conf中，有，
![](http://ww4.sinaimg.cn/large/0060lm7Tly1fppcn6t3qbj306x09wt8q.jpg
)  
集群参数的配置主要是通过几个配置文件来完成，这些配置文件位于conf目录：
+ hadoop-env.sh：用来定义Hadoop环境变量的Shell脚本
+ core-site.xml：定义所有Hadoop进程和客户端相关的参数
+ hdfs-site.xml：定义所有HDFS进程和客户端相关的参数
+ mapred-site.xml：定义与MapReduce进程和客户端相关的参数
+ log4j.properties：Java属性文件，包含所有日志配置信息
+ hadoop-policy.xml：定义有权限向MapReduce提交作业的用户或组
+ mapred-queue-acls.xml：对不同的队列实现不同用户的提交权限  

另有几个可选的配置文件：
+ master：列出由换行符分隔的、运行SecondaryNameNode的机器名
+ slaves：列出由换行符分隔的、运行DataNode/TaskTracker进程的机器名
+ fair-scheduler.xml：定义MapReduce插件Fair Scheduler任务调度的资源池和设置
+ capacity-scheduler.xml：定义MapReduce插件Capacity Scheduler任务调度的队列和设置
+ dfs.include：列出由换行符分隔的、允许连接NameNode的机器名
+ dfs.exclude：列出由换行符分隔的、禁止连接NameNode的机器名  

修改hadoop-env.sh中的export JAVA_HOME=/opt/app/jdk。  
修改conf目录中以下配置文件：  
+ core-site.xml：核心配置文件  
+ hdfs-site.xml：HDFS配置文件  
+ mapred-site.xml：MapReduce配置文件    

配置Hadoop — core-site.xml  
vi conf/core-site.xml
``` javascript
<configuration>
    <property>
       <name>hadoop.tmp.dir</name>
       <value>/home/hadoop/hadoop1/tmp</value>
    </property>
    <property>
       <name>fs.default.name</name>
       <value>hdfs://vm-kvm10094-app:8020</value>   也就是listen的端口，用于slave节点、client来连接
    </property>
</configuration>
```
其中使用了一个不存在的目录，可以使用mkdir命令事先创建该目录。

配置Hadoop — hdfs-site.xml  
vi hdfs-site.xml
``` javascript
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>3</value>   复制因子，需要小于实际slave节点数
    </property>
</configuration>
```

配置Hadoop — mapred-site.xml  
vi conf/mapred-site.xml
``` javascript
<configuration>
    <property>
        <name>mapred.job.tracker</name>
        <value>vm-kvm10094-app:8021</value>  也就是listen的端口，用于slave节点、client来连接
    </property>
</configuration>
```

vi masters  
vm-kvm10094-app

vi slaves  
vm-kvm10095-app

启动，  
在master节点上，HDFS使用前，需要执行格式化NameNode（仅第一次的时候需要），hadoop namenode -format。格式化的目的是在NameNode上创建初始的原数据，创建空的fsimage和edits文件，并为DataNode随机产生storgeID。当DataNode第一次连接NameNode时会接受这个storageID，之后他会拒绝与其他NameNode连接。如需重新格式化NameNode，必须将DataNode的数据和StorageID全部删除。或者是可以尝试将之前的name目录下所有文件cp到新的name目录下。接下来看下面的，也是关于格式化NameNode的，
+ 格式化操作会在NameNode数据目录（即dfs.name.dir指定的本地系统路径）的
