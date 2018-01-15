---
layout: post
title:  "cent os上安装java"
date:   2017-10-18 08:19:16 +0800
categories: linux
keywords: centos,linux
description: cent os上安装java
---

### 安装openjdk

使用yum安装java-1.8.0-openjdk与java-1.8.0-openjdk-devel:

```shell
sudo yum install java-1.8.0-openjdk.x86_64
sudo yum install java-1.8.0-openjdk-devel.x86_64
```

修改.bash_profile文件设置java_home与classpath

```shell
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.151-5.b12.el7_4.x86_64
export CLASSPATH=.:$JAVA_HOME/lib
PATH=$PATH:$JAVA_HOME/bin
```

<br/>

### 安装java

将java安装文件解压到/opt/local目录下面:

```shell
sudo tar -zxvf jdk-8u151-linux-x64.tar.gz -C /opt/local
```

centos使用`alternatives`来管理不同版本的java，这里需要把安装的java添加到`alternatives`中:

```shell
sudo alternatives --install /usr/bin/java java /opt/local/jdk1.8.0_151/bin/java 500
```

修改/etc/profile文件，设置java_home与classpath:

```shell
export JAVA_HOME=/opt/local/jdk1.8.0_151
export CLASSPATH=.:$JAVA_HOME/lib
PATH=$PATH:$JAVA_HOME/bin
```
