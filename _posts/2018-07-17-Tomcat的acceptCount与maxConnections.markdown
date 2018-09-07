---
layout: post
title:  "Tomcat的acceptCount与maxConnections"
date:   2018-07-17 08:00:00 +0800
categories: 其它
keywords: acceptCount,maxConnections,全链接,半链接
description: Tomcat的acceptCount与maxConnections
commentId: 2018-07-17
---

在使用tomcat时，经常会遇到连接数、线程数之类的配置问题，也偶尔遇到如connection timeout、read Timeout、connection reset by peer等错误，我们知道肯定是链接出了问题，但是为什么出问题？这些错误之间有什么不同呢？Tomcat中涉及并发的参数，如maxThreads、maxConnections、acceptCount等，他们代表什么意思呢？

<br/>

要解释上面这些问题，我们首先要了解TCP，对于Client端的一个请求，流程是这样的：Client首先发送SYN给服务端，内核会把连接信息放到syn队列中，同时返回一个SYN+ACK包给客户端。之后Client再次发送ACK包给服务端，内核会把连接从syn队列中取出，再把这个连接放到accept队列中。最后应用服务器调用accept系统调用从accept队列中获取已经建立成功的连接套接字，如下图所示：

<center><img src="{{site.baseurl}}/pic/tomcat/1.svg"/></center>

Tomcat的acceptor线程则负责从accept队列中取出该Connection，然后交给工作线程去处理（读取请求参数、处理逻辑、返回响应等等。如果该连接不是keep alived的话，则关闭该连接，然后该工作线程释放回线程池；如果是keep alived的话，则等待下一个数据包的到来直到keepAliveTimeout，然后关闭该连接释放回线程池），然后自己接着去accept队列取Connection。

如果accept队列满了之后，即使Client继续向server发送ACK的包，服务端会通过/proc/sys/net/ipv4/tcp_abort_on_overflow来决定如何响应，0表示直接丢丢弃该ACK，服务端过一段时间再次发送SYN+ACK给Client（也就是重新走握手的第二步），如果Client超时等待比较短，就会返回read timeout；1表示发送RST通知Client，Client端会返回connection reset by peer。如果是syn队列满了，不开启syncookies的时候，服务端会丢弃新来的SYN包，而Client在多次重发SYN包得不到响应而返回connection timeout错误。但是，当Server端开启了syncookies=1，那么SYN半连接队列就没有逻辑上的最大值了。

通过上面的分析，我们了解到TCP层面有两个队列：半连接队列（syn队列）与完全连接队列（accept队列）。syn队列的大小取决于max(64, /proc/sys/net/ipv4/tcp_max_syn_backlog)；accept队列的大小取决于min(backlog, somaxconn)，somaxconn是一个os级别的系统参数，而backlog的值可以由我们的应用程序（tomcat）去定义。

<br/>

### Tomcat参数

---

经过上面的分析，就可以很容易的掌握这些参数的意义和配置技巧了。

* **acceptCount**

    acceptCount参数用来指定accept队列的大小，即TCP完全链接队列的大小。

    acceptCount的大小要视情况而定。如果设得太小，请求量上来的时候Client会收到read timeout或connection reset by peer；如果设得太大，请求会积压在accept队列得不到及时的处理，同样会因为等待超时导致Client端返回read timeout。

* **maxThreads**

    maxThreads指定Tomcat最大并发线程数量，即同一时刻Tomcat最多maxThreads个线程在处理客户端的请求。

    maxThreads的设置与应用的特点有关，也与服务器的CPU核心数量有关。对于计算密集型应用，maxThreads应该尽可能设得小一些，让线程占用更多的CPU时间去处理逻辑；而对于IO密集型应用，maxThreads可以设得稍大些，让应用可以处理更大的并发。

* **maxConnections**

    maxConnections表示有多少个socket连接到tomcat上。对于BIO模式，一个线程只能处理一个链接，一般maxConnections取值与maxThreads相同，否则Client的socket即使连接上了，但是都在tomcat的task queue里头，等待worker线程处理返回响应；对于NIO模式，一个线程同时处理多个链接，maxConnections应该配置得比maxThreads要大的多，默认情况下是10000。
