---
layout: post
title:  "Flink Distributed Runtime"
date:   2019-09-10 08:00:00 +0800
categories: streaming
keywords: Flink,流式计算
description: Flink Distributed Runtime
commentId: 2019-09-10
---
The Flink runtime consists of two types of processes:

* The **JobManagers** (also called masters) coordinate the distributed execution. They schedule tasks, coordinate checkpoints, coordinate recovery on failures, etc.

* The **TaskManagers** (also called workers) execute the tasks (or more specifically, the subtasks) of a dataflow, and buffer and exchange the data streams.


The JobManagers and TaskManagers can be started in various ways: directly on the machines as a standalone cluster, in containers, or managed by resource frameworks like YARN or Mesos. TaskManagers connect to JobManagers, announcing themselves as available, and are assigned work.

The client is not part of the runtime and program execution, but is used to prepare and send a dataflow to the JobManager. After that, the client can disconnect, or stay connected to receive progress reports. The client runs either as part of the Java/Scala program that triggers the execution, or in the command line process ./bin/flink run ....

<center><img src="{{site.baseurl}}/pic/flink-parallelism/1.svg" width="80%"/></center>

<br/>

Each worker (TaskManager) is a JVM process, and contains one or more slots. Each task slot represents a fixed subset of resources of the TaskManager. The number of slots is typically proportional to the number of available CPU cores of each TaskManager.

A Flink program consists of one or more tasks, and each task is executed by one thread. However, tasks could be chained together within one thread for better performance so long as they are from the same job.

Task can be splitted into subtasks, Flink executes a program in parallel by splitting it into subtasks and scheduling these subtasks to processing slots.

<center><img src="{{site.baseurl}}/pic/flink-parallelism/2.svg" width="80%"/></center>
