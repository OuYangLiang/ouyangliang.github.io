---
layout: post
title:  "MapReduce简介"
date:   2018-02-25 08:19:16 +0800
categories: Hadoop
keywords: Hadoop,MapReduce
description: MapReduce简介
---
Hadoop的框架最核心的设计就是HDFS和MapReduce。前面我们已经介绍过HDFS是一个分布式的文件系统，为海量的数据提供了存储能力；MapReduce建立在HDFS基础上，为海量的数据提供了一个计算框架。MapReduce是一种分布式计算模型，是Google提出的，主要用于搜索领域，解决海量数据的计算问题。MR由两个阶段组成：Map和Reduce，用户只需实现map()和reduce()两个函数，即可实现分布式计算。

### MapReduce基本概念

MapReduce背后的思想很简单，就是把一些数据通过map来归类，通过reducer来把同一类的数据进行处理，其原理如下所示：

![mapreduce principle]({{site.baseurl}}/pic/hadoop/6.svg)

Map阶段的输入来自HDFS中的文件，我们知道文件在HDFS中是被拆分为文件块分布存储的，默认情况下每一个文件块会由一个map任务来处理。Map任务的输出会作为Reduce的输入，Reduce是对数据集进行精简，然后得出相应结果。当然，MapReduce的整个执行过程比描述的会复杂很多，一般过程分为：Split, Map, Shuffle, Reduce, Output几个阶段，我们以一个单词计数的示例来看看各个阶段的作用。

假设我们有一个文件：

```text
hello java
hello c
hello php
hello javascript
hello java language
hello c language
hello php language
hello javascript language
```

一、Split阶段：得益于HDFS的特性，文件在HDFS中是分块存储的。假设每个文件块包括2行内容的话，一共四个文件块：

<div class="row">
<div class="col-sm-6">
<table class="table table-bordered table-condensed table-striped text-left">
<tr class="info"><td>文件块</td><td>内容</td></tr>
<tr><td>0</td><td>hello java<br/>hello c</td></tr>
<tr><td>1</td><td>hello php<br/>hello javascript</td></tr>
<tr><td>2</td><td>hello java language<br/>hello c language</td></tr>
<tr><td>3</td><td>hello php language<br/>hello javascript language</td></tr>
</table>
</div>
</div>

二、Map阶段：将各个文件块的内容转换成新的key-value对：

<div class="row">
<div class="col-sm-8">
<table class="table table-bordered table-condensed table-striped text-left">
<tr class="info">
  <td>文件块</td>
  <td>内容</td>
  <td>Map任务</td>
  <td>map函数输出</td>
</tr>
<tr>
  <td>0</td>
  <td>hello java<br/>hello c</td>
  <td>Map任务0</td>
  <td>[hello:1]<br/>[java:1]<br/>[hello:1]<br/>[c:1]</td>
</tr>
<tr>
  <td>1</td>
  <td>hello php<br/>hello javascript</td>
  <td>Map任务1</td>
  <td>[hello:1]<br/>[php:1]<br/>[hello:1]<br/>[javascript:1]</td>
</tr>
<tr>
  <td>2</td>
  <td>hello java language<br/>hello c language</td>
  <td>Map任务2</td>
  <td>[hello:1]<br/>[java:1]<br/>[language:1]<br/>[hello:1]<br/>[c:1]<br/>[language:1]</td>
</tr>
<tr>
  <td>3</td>
  <td>hello php language<br/>hello javascript language</td>
  <td>Map任务3</td>
  <td>[hello:1]<br/>[php:1]<br/>[language:1]<br/>[hello:1]<br/>[javascript:1]<br/>[language:1]</td>
</tr>
</table>
</div>
</div>

三、Shuffle阶段：这个阶段比较复杂，每个Mapper任务首先会根据Reducer的任务数量对key-value对进行分区，然后对每个分区的key进行排序和分组，执行Combiner，最后发送给Reducer任务。

<div class="row">
<div class="col-sm-12">
<table class="table table-bordered table-condensed table-striped text-left">
<tr class="info">
  <td>Map任务</td>
  <td>map函数输出</td>
  <td>分区</td>
  <td>排序</td>
  <td>分组</td>
  <td>Combiner</td>
</tr>
<tr>
  <td>Map任务0</td>
  <td>[hello:1]<br/>[java:1]<br/>[hello:1]<br/>[c:1]</td>
  <td>分区0<br/>[hello:1],[java:1],[hello:1]<br/>分区1<br/>[c:1]</td>
  <td>分区0<br/>[hello:1],[hello:1],[java:1]<br/>分区1<br/>[c:1]</td>
  <td>分区0<br/>[hello:{1,1}],[java:1]<br/>分区1<br/>[c:1]</td>
  <td>分区0<br/>[hello:2],[java:1]<br/>分区1<br/>[c:1]</td>
</tr>
<tr>
  <td>Map任务1</td>
  <td>[hello:1]<br/>[php:1]<br/>[hello:1]<br/>[javascript:1]</td>
  <td>分区0<br/>[hello:1],[php:1],[hello:1]<br/>分区1<br/>[javascript:1]</td>
  <td>分区0<br/>[hello:1],[hello:1],[php:1]<br/>分区1<br/>[javascript:1]</td>
  <td>分区0<br/>[hello:{1,1}],[php:1]<br/>分区1<br/>[javascript:1]</td>
  <td>分区0<br/>[hello:2],[php:1]<br/>分区1<br/>[javascript:1]</td>
</tr>
<tr>
  <td>Map任务2</td>
  <td>[hello:1]<br/>[java:1]<br/>[language:1]<br/>[hello:1]<br/>[c:1]<br/>[language:1]</td>
  <td>分区0<br/>[hello:1],[java:1],[hello:1]<br/>分区1<br/>[language:1],[c:1],[language:1]</td>
  <td>分区0<br/>[hello:1],[hello:1],[java:1]<br/>分区1<br/>[c:1],[language:1],[language:1]</td>
  <td>分区0<br/>[hello:{1,1}],[java:1]<br/>分区1<br/>[c:1],[language:{1,1}]</td>
  <td>分区0<br/>[hello:2],[java:1]<br/>分区1<br/>[c:1],[language:2]</td>
</tr>
<tr>
  <td>Map任务3</td>
  <td>[hello:1]<br/>[php:1]<br/>[language:1]<br/>[hello:1]<br/>[javascript:1]<br/>[language:1]</td>
  <td>分区0<br/>[hello:1],[php:1],[hello:1]<br/>分区1<br/>[language:1],[javascript:1],[language:1]</td>
  <td>分区0<br/>[hello:1],[hello:1],[php:1]<br/>分区1<br/>[javascript:1],[language:1],[language:1]</td>
  <td>分区0<br/>[hello:{1,1}],[php:1]<br/>分区1<br/>[javascript:1],[language:{1,1}]</td>
  <td>分区0<br/>[hello:2],[php:1]<br/>分区1<br/>[javascript:1],[language:2]</td>
</tr>
</table>
</div>
</div>

四、Reduce阶段：Shuffle结束后，同一分区的数据会传送给同一个Reducer任务。Reducer任务接收到key-value对后会先根据key进行排序和分组，最后执行Reducer函数输出结果。

<div class="row">
<div class="col-sm-12">
<table class="table table-bordered table-condensed table-striped text-left">
<tr class="info">
  <td>Reduce任务</td>
  <td>输入</td>
  <td>排序</td>
  <td>分组</td>
  <td>输出</td>
</tr>
<tr>
  <td>Reduce任务0</td>
  <td>[hello:2]<br/>[java:1]<br/>[hello:2]<br/>[php:1]<br/>[hello:2]<br/>[java:1]<br/>[hello:2]<br/>[php:1]</td>
  <td>[hello:2]<br/>[hello:2]<br/>[hello:2]<br/>[hello:2]<br/>[java:1]<br/>[java:1]<br/>[php:1]<br/>[php:1]</td>
  <td>[hello:{2,2,2,2}]<br/>[java:{1,1}]<br/>[php:{1,1}]</td>
  <td>[hello:8]<br/>[java:2]<br/>[php:2]</td>
</tr>
<tr>
  <td>Reduce任务1</td>
  <td>[c:1]<br/>[javascript:1]<br/>[c:1]<br/>[language:2]<br/>[javascript:1]<br/>[language:2]</td>
  <td>[c:1]<br/>[c:1]<br/>[javascript:1]<br/>[javascript:1]<br/>[language:2]<br/>[language:2]</td>
  <td>[c:{1,1}]<br/>[javascript:{1,1}]<br/>[language:{2,2}]</td>
  <td>[c:2]<br/>[javascript:2]<br/>[language:4]</td>
</tr>
</table>
</div>
</div>

### Yarn简介

(待续...)

### Yarn环境搭建

在[《HDFS简介》]({{site.baseurl}}/2018/02/HDFS简介.html)中我们已经搭建了一个HDFS集群，现在我们接着在上面搭建一个Yarn环境。

<br/>

**一、修改mapred-site.xml文件（三台服务器一样）**

```xml
<property>
  <!--指定Mapreduce运行在yarn上-->
  <name>mapreduce.framework.name</name>
  <value>yarn</value>
</property>
<property>
  <!--Larger resource limit for maps-->
  <name>mapreduce.map.memory.mb</name>
  <value>1024</value>
</property>
<property>
  <!--Larger resource limit for reduces.-->
  <name>mapreduce.reduce.memory.mb</name>
  <value>1024</value>
</property>
<property>
  <!--Higher memory-limit while sorting data for efficiency.-->
  <name>mapreduce.task.io.sort.mb</name>
  <value>100</value>
</property>
<property>
  <name>yarn.app.mapreduce.am.resource.mb</name>
  <value>1536</value>
</property>
```
<br/>

**二、修改yarn-site.xml文件（三台服务器一样）**

```xml
<property>
  <name>yarn.resourcemanager.hostname</name>
  <value>192.168.0.161</value>
</property>
<property>
  <!--For ResourceManager Only-->
  <!--ResourceManager对客户端暴露的地址。客户端通过该地址向RM提交应用程序，杀死应用程序等-->
  <name>yarn.resourcemanager.address</name>
  <value>192.168.0.161:8032</value>
</property>
<property>
  <!--For ResourceManager Only-->
  <!--ResourceManager对ApplicationMaster暴露的访问地址。ApplicationMaster通过该地址向RM申请资源、释放资源等-->
  <name>yarn.resourcemanager.scheduler.address</name>
  <value>192.168.0.161:8030</value>
</property>
<property>
  <!--For ResourceManager Only-->
  <!--ResourceManager对NodeManager暴露的地址.。NodeManager通过该地址向RM汇报心跳，领取任务等-->
  <name>yarn.resourcemanager.resource-tracker.address</name>
  <value>192.168.0.161:8031</value>
</property>
<property>
  <!--For ResourceManager Only-->
  <!--ResourceManager对管理员暴露的访问地址。管理员通过该地址向RM发送管理命令等-->
  <name>yarn.resourcemanager.admin.address</name>
  <value>192.168.0.161:8033</value>
</property>
<property>
  <!--For ResourceManager Only-->
  <!--ResourceManager对外Web UI地址。用户可通过该地址在浏览器中查看集群各类信息-->
  <name>yarn.resourcemanager.webapp.address</name>
  <value>192.168.0.161:8088</value>
</property>
<property>
  <!--For ResourceManager Only-->
  <!--单个可申请的最小内存资源量-->
  <name>yarn.scheduler.minimum-allocation-mb</name>
  <value>1024</value>
</property>
<property>
  <!--For ResourceManager Only-->
  <!--整个集群的可又支配内存总量-->
  <name>yarn.scheduler.maximum-allocation-mb</name>
  <value>8192</value>
</property>
<property>
  <!--For NodeManager Only-->
  <!--NodeManager上运行的附属服务。需配置成mapreduce_shuffle，才可运行MapReduce程序-->
  <name>yarn.nodemanager.aux-services</name>
  <value>mapreduce_shuffle</value>
</property>
<property>
  <!--For NodeManager Only-->
  <!--中间结果存放位置，类似于1.0中的mapred.local.dir。注意，这个参数通常会配置多个目录，已分摊磁盘IO负载-->
  <name>yarn.nodemanager.local-dirs</name>
  <value>/opt/local/bigdata/nm-local-dir</value>
</property>
<property>
  <!--For NodeManager Only-->
  <!--日志存放地址（可配置多个目录）-->
  <name>yarn.nodemanager.log-dirs</name>
  <value>/opt/local/bigdata/nm-log-dir</value>
</property>
<property>
  <!--For NodeManager Only-->
  <!--NodeManager总的可用物理内存-->
  <name>yarn.nodemanager.resource.memory-mb</name>
  <value>4096</value>
</property>
```

<br/>

**三、创建临时目录与日志目录（三台服务器一样）:**

```shell
mkdir /opt/local/bigdata/nm-log-dir
mkdir /opt/local/bigdata/nm-local-dir
```

<br/>

**四、在192.168.0.161服务器启动resourcemanager进程:**

```shell
/opt/local/hadoop-2.6.5/sbin/yarn-daemon.sh start resourcemanager
```

<br/>

**五、在192.168.0.162与192.168.0.163服务器启动nodemanager进程:**

```shell
/opt/local/hadoop-2.6.5/sbin/yarn-daemon.sh start nodemanager
```

<br/>

**六、在192.168.0.161服务器启动作业日志服务:**

```shell
/opt/local/hadoop-2.6.5/sbin/mr-jobhistory-daemon.sh start historyserver
```

<br/>

现在我们可以通过http://192.168.0.161:8088访问Yarn控制平台了，http://192.168.0.161:19888访问MR作业日志服务了

也可能像启动HDFS集群那样，使用<kbd>start-yarn.sh</kbd>命令启动Yarn集群。

### 测试

首先，将测试的文件上传到HDFS中：

```shell
./bin/hdfs dfs -put ~/inputfile /hadoop/documents/
./bin/hdfs dfs -cat /hadoop/input/inputfile
hello java
hello c
hello php
hello javascript
hello java language
hello c language
hello php language
hello javascript language
```

执行

```shell
./bin//hadoop jar ~/hadoop-0.0.1-SNAPSHOT.jar com.personal.oyl.code.example.hadoop.WordCount /hadoop/input /hadoop/output
```

查看结果

```shell
[john@server-1 hadoop-2.6.5]$ ./bin/hdfs dfs -cat /hadoop/output/part-r-00000
c	2
hello	8
java	2
javascript	2
language	4
php	2
```

和预期一样!

https://www.zhihu.com/question/23345991
http://www.cnblogs.com/duguguiyu/archive/2009/02/28/1400278.html
http://www.cnblogs.com/MitiskySean/p/3320451.html
http://hadoop.apache.org/docs/current/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html
http://blog.sina.com.cn/s/blog_7581a4c30102veem.html
http://blog.csdn.net/cnbird2008/article/details/23788233
http://flyingdutchman.iteye.com/blog/1878775
http://www.cnblogs.com/dandingyy/archive/2013/03/08/2950703.html
http://blog.csdn.net/post_yuan/article/details/54631446
http://www.cnblogs.com/ahu-lichang/p/6665242.html
