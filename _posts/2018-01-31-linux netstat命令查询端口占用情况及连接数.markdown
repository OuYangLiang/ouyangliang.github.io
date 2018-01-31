---
layout: post
title:  "linux netstat命令查询端口占用情况及连接数"
date:   2018-01-31 08:19:16 +0800
categories: linux
keywords: netstat,linux
description:
---
Netstat命令用于显示各种网络相关信息，如网络连接，路由表，接口状态 (Interface Statistics)，masquerade 连接，多播成员 (Multicast Memberships) 等等。

```shell
[ouyangliang2@hadoop01 ~]$ netstat
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address               Foreign Address             State      
tcp        0      0 localhost:8247              localhost:eforward          ESTABLISHED
tcp        0      0 hadoop01.crm.t:zabbix-agent 10.40.191.115:18352         TIME_WAIT   
tcp        0      0 localhost:eforward          localhost:19210             ESTABLISHED
tcp        0      0 localhost:eforward          localhost:8247              ESTABLISHED
tcp        0      0 localhost:eforward          localhost:19218             ESTABLISHED

Active UNIX domain sockets (w/o servers)
Proto RefCnt Flags       Type       State         I-Node Path
unix  2      [ ]         DGRAM                    13028  @/org/kernel/udev/udevd
unix  4      [ ]         DGRAM                    2319386033 /dev/log
unix  3      [ ]         STREAM     CONNECTED     2857486718
unix  3      [ ]         STREAM     CONNECTED     2857486717
unix  2      [ ]         DGRAM                    2857486714
unix  2      [ ]         STREAM     CONNECTED     2780507856
```

从整体上看，netstat的输出结果可以分为两个部分：一个是Active Internet connections，称为有源TCP连接，其中"Recv-Q"和"Send-Q"指%0A的是接收队列和发送队列。另一个是Active UNIX domain sockets，称为有源Unix域套接口(和网络套接字一样，但是只能用于本机通信，性能可以提高一倍)。

<br/>

### 常用参数说明

* -a (all)显示所有选项，默认不显示LISTEN相关

* -t (tcp)仅显示tcp相关选项

* -u (udp)仅显示udp相关选项

* -n 拒绝显示别名，能显示数字的全部转化成数字。

* -l 仅列出有在 Listen (监听) 的服務状态

* -p 显示建立相关链接的程序名

* -r 显示路由信息，路由表

* -e 显示扩展信息，例如uid等

* -s 按各个协议进行统计

* -c 每隔一个固定时间，执行该netstat命令。

提示：LISTEN和LISTENING的状态只有用-a或者-l才能看到

<br/>

### 查询端口监听状态

```shell
netstat -nl | grep -iw 端口号
```

<br/>

### 查询端口连接数

```shell
netstat -ntu | grep -iw 2181 | grep ESTABLISHED | wc -l
```

或

```shell
netstat -na | grep -iw 2181 | grep ESTABLISHED | wc -l
```
