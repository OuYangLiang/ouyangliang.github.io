---
layout: post
title:  "MapReduce简介"
date:   2018-02-25 08:19:16 +0800
categories: Hadoop
keywords: Hadoop,MapReduce
description: MapReduce简介
---
Hadoop的框架最核心的设计就是HDFS和MapReduce。前面我们已经介绍过HDFS是一个分布式的文件系统，为海量的数据提供了存储能力；MapReduce建立在HDFS基础上，为海量的数据提供了一个计算框架。MapReduce是一种分布式计算模型，是Google提出的，主要用于搜索领域，解决海量数据的计算问题。MR由两个阶段组成：Map和Reduce，用户只需实现map()和reduce()两个函数，即可实现分布式计算。

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
<tr class="info"><td>文件块</td><td>内容</td><td>Map任务</td><td>map函数输出</td></tr>
<tr><td>0</td><td>hello java<br/>hello c</td><td>Map任务0</td><td>[hello:1]<br/>[java:1]<br/>[hello:1]<br/>[c:1]</td></tr>
<tr><td>1</td><td>hello php<br/>hello javascript</td><td>Map任务1</td><td>[hello:1]<br/>[php:1]<br/>[hello:1]<br/>[javascript:1]</td></tr>
<tr><td>2</td><td>hello java language<br/>hello c language</td><td>Map任务2</td><td>[hello:1]<br/>[java:1]<br/>[language:1]<br/>[hello:1]<br/>[c:1]<br/>[language:1]</td></tr>
<tr><td>3</td><td>hello php language<br/>hello javascript language</td><td>Map任务3</td><td>[hello:1]<br/>[php:1]<br/>[language:1]<br/>[hello:1]<br/>[javascript:1]<br/>[language:1]</td></tr>
</table>
</div>
</div>

三、Shuffle阶段：这个阶段比较复杂，每个Mapper任务首先会根据Reducer的任务数量对key-value对进行分区，然后对每个分区的key进行排序和分组，执行Combiner，最后发送给Reducer任务。

<div class="row">
<div class="col-sm-12">
<table class="table table-bordered table-condensed table-striped text-left">
<tr class="info"><td>Map任务</td><td>map函数输出</td><td>分区</td><td>排序</td><td>分组</td><td>Combiner</td></tr>
<tr><td>Map任务0</td><td>[hello:1]<br/>[java:1]<br/>[hello:1]<br/>[c:1]</td><td>分区0<br/>[hello:1],[java:1],[hello:1]<br/>分区1<br/>[c:1]</td><td>分区0<br/>[hello:1],[hello:1],[java:1]<br/>分区1<br/>[c:1]</td><td>分区0<br/>[hello:{1,1}],[java:1]<br/>分区1<br/>[c:1]</td><td>分区0<br/>[hello:2],[java:1]<br/>分区1<br/>[c:1]</td></tr>
<tr><td>Map任务1</td><td>[hello:1]<br/>[php:1]<br/>[hello:1]<br/>[javascript:1]</td><td>分区0<br/>[hello:1],[php:1],[hello:1]<br/>分区1<br/>[javascript:1]</td><td>分区0<br/>[hello:1],[hello:1],[php:1]<br/>分区1<br/>[javascript:1]</td><td>分区0<br/>[hello:{1,1}],[php:1]<br/>分区1<br/>[javascript:1]</td><td>分区0<br/>[hello:2],[php:1]<br/>分区1<br/>[javascript:1]</td></tr>
<tr><td>Map任务2</td><td>[hello:1]<br/>[java:1]<br/>[language:1]<br/>[hello:1]<br/>[c:1]<br/>[language:1]</td><td>分区0<br/>[hello:1],[java:1],[hello:1]<br/>分区1<br/>[language:1],[c:1]</td><td>分区0<br/>[hello:1],[hello:1],[java:1]<br/>分区1<br/>[c:1],[language:1]</td><td>分区0<br/>[hello:{1,1}],[java:1]<br/>分区1<br/>[c:1],[language:1]</td><td>分区0<br/>[hello:2],[java:1]<br/>分区1<br/>[c:1],[language:1]</td></tr>
<tr><td>Map任务3</td><td>[hello:1]<br/>[php:1]<br/>[language:1]<br/>[hello:1]<br/>[javascript:1]<br/>[language:1]</td><td>分区0<br/>[hello:1],[php:1],[hello:1]<br/>分区1<br/>[language:1],[javascript:1]</td><td>分区0<br/>[hello:1],[hello:1],[php:1]<br/>分区1<br/>[javascript:1],[language:1]</td><td>分区0<br/>[hello:{1,1}],[php:1]<br/>分区1<br/>[javascript:1],[language:1]</td><td>分区0<br/>[hello:2],[php:1]<br/>分区1<br/>[javascript:1],[language:1]</td></tr>
</table>
</div>
</div>

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
