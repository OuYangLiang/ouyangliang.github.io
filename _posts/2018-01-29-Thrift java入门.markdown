---
layout: post
title:  "Thrift java入门"
date:   2018-01-29 08:19:16 +0800
categories: 中间件
keywords: thrift,rpc
description: thrift java学习笔记
---
Thrift是一个RPC框架，由facebook开发，07年四月开放源码，08年5月进入Apache孵化器。它支持可扩展且跨语言的服务的开发，它结合了功能强大的软件堆栈和代码生成引擎，以构建在C++、Java、Python、PHP、Ruby、Erlang、Perl、Haskell、C#、Cocoa、JavaScript、Node.js、Smalltalk and OCaml等等编程语言间无缝结合的、高效的服务。Thrift允许你定义一个简单的定义文件中的数据类型和服务接口。以作为输入文件，编译器生成代码用来方便地生成RPC客户端和服务器通信的无缝跨编程语言。

<br/>

### 一、Thrift基本结构

![thrift network structure]({{site.baseurl}}/pic/thrift/1.svg)

所图所示，thrift在设计上分了四个层次，从下到上分别是：

* Transport层：抽象了数据在网络中的传输。

* Protocol层：定义了数据的序列化、反序列化方式。常用的格式有：二进制、压缩格式和json格式。

* Processor层：Thrift中最关键的一层，它包括thrift文件生成的接口，以及这些接口应对的实现。

* Server层：将所有这些（Transport、Protocol与Processor）封装在一起，对外提供服务。

<br/>

### 二、在Mac OS上安装Thrift

使用thrift最关键的一环就是使用[IDL语言](http://thrift.apache.org/docs/idl)来编写thrift文件，在thrift文件中我们可以定义如javabean、enum、interface等常用语言的无素。最后通过thrift-compiler编译thrift文件生成目标语言。在mac上使用homebrew安装thrift非常简单：

```shell
brew install thrift
```

之后就可以使用<kbd>thrift</kbd>命令编译thrift文件了，如：

```shell
$ thrift --gen java hello.thrift
```

上述命令执行后，会在当前目前下多一个`gen-java`目录，生成的文件就在里面：

```shell
$ find .
.
./gen-java
./gen-java/com
./gen-java/com/personal
./gen-java/com/personal/oyl
./gen-java/com/personal/oyl/code
./gen-java/com/personal/oyl/code/example
./gen-java/com/personal/oyl/code/example/thrift
./gen-java/com/personal/oyl/code/example/thrift/HelloService.java
./hello.thrift
```

<br/>

### 三、使用Thrift的基本步骤

1. 使用Thrift的[IDL语言](http://thrift.apache.org/docs/idl)编写thrift文件，定义接口。

2. 使用thrift-compiler编译thrift文件，生成接口文件。

**服务端的开发**

1. 实现接口文件中定义的接口

2. 创建processor、transport与protocol

3. 创建TServer

4. 启动Server

**客户端的开发**

1. 创建transport与protocol

2. 创建Client

3. 调用Client的相应方法

<br/>

### 四、一个简单的示例

首先，我们编写一个thrift文件，命名为hello.thrift

```thrift
namespace java com.personal.oyl.code.example.thrift

service HelloService {
  i32 add(1:i32 num1, 2:i32 num2)
}
```

hello.thrift文件中定义了一个名为HelloService的service，对应于java来说它会生成一个interface，其中有一个add方法，它接收两个int类型的参数并返回一个int值，namespace用于说明这个interface所在的包名。

<br/>

接着我们使用thrift-compiler编译hello.thrift文件，这会在当前目录下生成接口文件，即HelloService.java文件，如下所示：

```shell
$ thrift --gen java hello.thrift
$ find .
.
./gen-java
./gen-java/com
./gen-java/com/personal
./gen-java/com/personal/oyl
./gen-java/com/personal/oyl/code
./gen-java/com/personal/oyl/code/example
./gen-java/com/personal/oyl/code/example/thrift
./gen-java/com/personal/oyl/code/example/thrift/HelloService.java
./hello.thrift
```

<br/>

查看HelloService.java文件的内容会发现，我们定义在hello.thrift文件中的方法实际上是封装在Iface接口中，这意味着在服务端我们需要实现的是HelloService.Iface而不是HelloService。

```java
package com.personal.oyl.code.example.thrift;

public class HelloService {
  public interface Iface {
    public int add(int num1, int num2) throws org.apache.thrift.TException;
  }
  ...
}
```

好了，现在我们已经生成了接口文件，接下来我们继续编写服务端的代码。

<br/>

接下来需要引入thrift的类库，如果使用maven的话，可以在pom.xml文件中增加libthrift的依赖：

```xml
<dependency>
  <groupId>org.apache.thrift</groupId>
  <artifactId>libthrift</artifactId>
  <version>0.11.0</version>
</dependency>
```

**（注：libthrift的版本最好与thrift-compiler的版本保持一致）**

<br/>

服务端的开发，我们首先需要实现HelloService.Iface接口：

```java
public class HelloServiceImpl implements HelloService.Iface{
    @Override
    public int add(int num1, int num2) throws TException {
        return num1 + num2;
    }
}
```

<br/>

最后通过libthrift提供的类库，将我们的服务爆露出去：

```java
public class HelloServer {
    public static void main(String[] args) {
        try {
            HelloService.Processor<HelloServiceImpl> processor =
                    new HelloService.Processor<>(new HelloServiceImpl());

            TServerTransport serverTransport = new TServerSocket(8080);
            TServer.Args param = new TServer.Args(serverTransport);
            param.processor(processor);
            param.protocolFactory(new TBinaryProtocol.Factory());

            TServer server = new TSimpleServer(param);
            System.out.println("Starting Thrift Server......");
            server.serve();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

HelloService类中包括一个内部类Processor，借助于它和我们的实现类HelloServiceImpl可以得到了一个processor；再通过TServerSocket类与TBinaryProtocol类实例化得到transport对象与protocol对象；一起封装打包到TServer.Args对象中，作为参数构建TSimpleServer对象，之后就可以启动服务了。

<br/>

客户端代码相对简单，注意需要保证protocol与服务器端一致。

```java
public class HelloClient {

    public static void main(String[] args) {
        try {
            TTransport transport = new TSocket("localhost", 8080);
            transport.open();

            TProtocol protocol = new TBinaryProtocol(transport);
            HelloService.Client client = new HelloService.Client(protocol);

            System.out.println(client.add(100, 200));

            transport.close();
        } catch (TException e) {
            e.printStackTrace();
        }
    }
}
```

<br/>

至此，Thrift的入门介绍算是结束了，但是Thrift提供了不同类型的Server、Protocol及Transport，针对不同的场景、要求，他们表现有很大的不同，后面一篇博客我们会介绍它们之间的优略!!!
