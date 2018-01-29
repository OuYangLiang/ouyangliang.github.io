---
layout: post
title:  "zookeeper集群部署"
date:   2017-08-15 08:19:16 +0800
categories: 中间件
keywords: zookeeper,linux
description: 在centos上部署zookeeper集群
---

**一、将zookeeper安装文件解压到/opt/local目录下面:**

```shell
sudo tar -zxvf zookeeper-3.4.10.tar.gz -C /opt/local
```

<br/>

**二、在zookeeper目录下面创建data目录，用于存放zookeeper日志文件和snapshot文件:**

```shell
cd /opt/local/zookeeper-3.4.10
sudo mkdir data
sudo mkdir data/snapshot
sudo mkdir data/logs
```

<br/>

**三、修改zookeeper配置文件conf/zoo.cfg:**

```shell
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just
# example sakes.
dataDir=/opt/local/zookeeper-3.4.10/data/snapshot
dataLogDir=/opt/local/zookeeper-3.4.10/data/logs
# the port at which the clients will connect
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1

server.0=192.168.0.161:2888:3888
server.1=192.168.0.162:2888:3888
server.2=192.168.0.163:2888:3888
```

文件中最后三行用于zookeeper集群配置，这里使用三台服务器来搭建zookeeper集群。3888端口用于领导者选举，2888端口用于leader向follower和observer同步数据，并且：

* 192.168.0.161的标识为0
* 192.168.0.162的标识为1
* 192.168.0.163的标识为2

所以我们需要分别在各个服务器中dataDir目录（也就是/opt/local/zookeeper-3.4.10/data/snapshot）下创建一个myid文件，文件内容是该服务器的标识。

<br/>

**四、修改/etc/profile文件，设置zookeeper path:**

```shell
export ZOOKEEPER_HOME=/opt/local/zookeeper-3.4.10
export PATH=$PATH:$ZOOKEEPER_HOME/bin
```

<br/>

**五、常用操作**

启动zookeeper

```shell
sudo zkServer.sh start
```

查看zookeeper状态

```shell
zkServer.sh status
```

现在可以通过下面命令登录zookeeper集群:

```shell
zkCli.sh -server 192.168.0.161:2181,192.168.0.162:2181,192.168.0.163:2181
```

使用四字母命令访问服务器，`nc`命令用于发送信息至指定IP的端口号:

```shell
echo stat | nc 192.168.0.161 2181 #查询zookeeper状态
echo conf | nc 192.168.0.161 2181 #查询zookeeper配置信息
echo ruok | nc 192.168.0.161 2181 #检查zookeeper是否运行正常
echo envi | nc 192.168.0.161 2181 #查询zookeeper详细信息，比stat命令返回更多信息
```

关闭zookeeper

```shell
sudo zkServer.sh stop
```
