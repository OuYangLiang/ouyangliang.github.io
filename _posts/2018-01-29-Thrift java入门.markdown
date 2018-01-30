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

### 五、Thrift支持哪些数据传输协议

Thrift可以让你选择客户端与服务端之间数据的传输通信协议，但要保证客户端和服务端的协议一致。在传输协议上总体上划分为文本和二进制传输协议，为节约带宽和提高传输效率，一般情况下使用二进制类型的传输协议较多，但有时会还是会使用基于文本类型的协议，这需要根据实际需求来看。

* TBinaryProtocol

    二进制编码格式进行数据传输。

* TCompactProtocol

    这种协议非常有效的，使用Variable-Length Quantity (VLQ) 编码对数据进行压缩。

* TJSONProtocol

    使用JSON的数据编码协议进行数据传输。

* TSimpleJSONProtocol

    这种节约只提供JSON只写的协议，适用于通过脚本语言解析
* TDebugProtocol

    在开发的过程中帮助开发人员调试用的，以文本的形式展现方便阅读。

<br/>

### 六、Thrift在服务端提供了哪些选择

前面的示例中我们使用了`TSimpleServer`这个类，这是一个单线程阻塞式IO的服务实现，这点，通过`TSimpleServer`的源码也可以看出来

```java
public void serve() {
  //此处省略部分无关代码...
  while (!stopped_) {
    try {
      client = serverTransport_.accept();
      if (client != null) {
        processor = processorFactory_.getProcessor(client);
        inputTransport = inputTransportFactory_.getTransport(client);
        outputTransport = outputTransportFactory_.getTransport(client);
        inputProtocol = inputProtocolFactory_.getProtocol(inputTransport);
        outputProtocol = outputProtocolFactory_.getProtocol(outputTransport);
        if (eventHandler_ != null) {
          connectionContext = eventHandler_.createContext(inputProtocol, outputProtocol);
        }
        while (true) {
          if (eventHandler_ != null) {
            eventHandler_.processContext(connectionContext, inputTransport, outputTransport);
          }
          if(!processor.process(inputProtocol, outputProtocol)) {
            break;
          }
        }
      }
    }
    //此处省略部分无关代码...
  }
}
```

<br/>

Thrift在服务端提供了很多不同类型的选择：

* TSimpleServer

    TSimpleServer是单线程阻塞IO的实现，仅适用于demo。

* TThreadPoolServer

    顾名思义，TThreadPoolServer内部使用一个线程池来处理客户端的请求。它使用一个专门的线程来接收请求，一旦接收到请求就会放入ThreadPoolExecutor中的一个线程池处理，性能表现优异。

* TNonblockingServer

    单线程非阻塞IO的实现，通过java.nio.channels.Selector的select()接收连接请求，但是处理消息仍然是单线程，不可用于生产。

* THsHaServer

    THsHaServer继承了TNonblockingServer，不同之处在于THsHaServer内部使用了一个线程池来处理请求。THsHaServer相比较于TNonblockingServer类似TThreadPoolServer相比较于TSimpleServer。

    另外，当使用TNonblockingServer或者THsHaServer时，必须使用TFramedTransport来封装一下原始的transport。

* TThreadedSelectorServer

    是thrift 0.8引入的实现，处理请求也使用了线程池，比THsHaServer有更高的吞吐量和更低的时延。

<br/>

*最后附上示例源码地址*：[https://github.com/OuYangLiang/code-example/tree/master/thrift](https://github.com/OuYangLiang/code-example/tree/master/thrift)
