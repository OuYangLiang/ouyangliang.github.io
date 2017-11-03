---
layout: post
title:  "Thread Dump分析"
date:   2017-10-20 08:19:16 +0800
categories: JVM
keywords: 线程转储,thread, thread dump,java
description: 如何根据thread dump分析和定位问题
---
通过`jcmd -l`或`jps -l`命令得到进程的pid，再通过`jstack ${pid} > threaddump.txt`得到线程转储。

```
2017-10-20 23:57:49
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.111-b14 mixed mode):

"Attach Listener" #9 daemon prio=9 os_prio=31 tid=0x00007ff8e9836800 nid=0x3507 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Service Thread" #8 daemon prio=9 os_prio=31 tid=0x00007ff8e9806000 nid=0x4803 runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C1 CompilerThread2" #7 daemon prio=9 os_prio=31 tid=0x00007ff8e901b800 nid=0x4603 waiting on condition [0x0000000000000000]

"C2 CompilerThread1" #6 daemon prio=9 os_prio=31 tid=0x00007ff8e901b000 nid=0x4403 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread0" #5 daemon prio=9 os_prio=31 tid=0x00007ff8ea824800 nid=0x4203 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Signal Dispatcher" #4 daemon prio=9 os_prio=31 tid=0x00007ff8ea823800 nid=0x320f runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Finalizer" #3 daemon prio=8 os_prio=31 tid=0x00007ff8e9011000 nid=0x3003 in Object.wait() [0x000070000092e000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	- waiting on <0x0000000795588e98> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:143)
	- locked <0x0000000795588e98> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:164)
	at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:209)

"Reference Handler" #2 daemon prio=10 os_prio=31 tid=0x00007ff8e900e800 nid=0x2e03 in Object.wait() [0x000070000082b000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	- waiting on <0x0000000795586b40> (a java.lang.ref.Reference$Lock)
	at java.lang.Object.wait(Object.java:502)
	at java.lang.ref.Reference.tryHandlePending(Reference.java:191)
	- locked <0x0000000795586b40> (a java.lang.ref.Reference$Lock)
	at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:153)

"main" #1 prio=5 os_prio=31 tid=0x00007ff8e9800000 nid=0x1703 waiting on condition [0x0000700000219000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
	at java.lang.Thread.sleep(Native Method)
	at java.lang.Thread.sleep(Thread.java:340)
	at java.util.concurrent.TimeUnit.sleep(TimeUnit.java:386)
	at com.personal.oyl.kafka.test.App.main(App.java:16)

"VM Thread" os_prio=31 tid=0x00007ff8ea812000 nid=0x2c03 runnable

"GC task thread#0 (ParallelGC)" os_prio=31 tid=0x00007ff8ea805800 nid=0x2403 runnable

"GC task thread#1 (ParallelGC)" os_prio=31 tid=0x00007ff8ea806000 nid=0x2603 runnable

"GC task thread#2 (ParallelGC)" os_prio=31 tid=0x00007ff8ea806800 nid=0x2803 runnable

"GC task thread#3 (ParallelGC)" os_prio=31 tid=0x00007ff8ea807800 nid=0x2a03 runnable

"VM Periodic Task Thread" os_prio=31 tid=0x00007ff8ea040000 nid=0x4a03 waiting on condition

JNI global references: 6
```

<br/>

### Thread Dump文件中包括哪些内容

---

Thread Dump文件在逻辑上我们可以分为三个部分。第一个部分是文件的前两行，展示了Thread Dump文件生成的时间，以及当前JVM的简单信息，如下：

```
2017-10-20 23:57:49
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.111-b14 mixed mode):
```

<br/>

第二个部分是关于每个线程的详细描述信息，这些线程可能来自Java容器、第三方类库或应用程序本身。如：

```
"main" #1 prio=5 os_prio=31 tid=0x00007ff8e9800000 nid=0x1703 waiting on condition [0x0000700000219000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
	at java.lang.Thread.sleep(Native Method)
	at java.lang.Thread.sleep(Thread.java:340)
	at java.util.concurrent.TimeUnit.sleep(TimeUnit.java:386)
	at com.personal.oyl.kafka.test.App.main(App.java:16)
```

![thread dump线程信息解读]({{site.baseurl}}/pic/threaddump/1.svg)

<br/>

第三个部分是一些特殊的线程，如JVM内部的一些GC线程、编译线程等：

```
"VM Thread" os_prio=31 tid=0x00007ff8ea812000 nid=0x2c03 runnable

"GC task thread#0 (ParallelGC)" os_prio=31 tid=0x00007ff8ea805800 nid=0x2403 runnable

"GC task thread#1 (ParallelGC)" os_prio=31 tid=0x00007ff8ea806000 nid=0x2603 runnable

"GC task thread#2 (ParallelGC)" os_prio=31 tid=0x00007ff8ea806800 nid=0x2803 runnable

"GC task thread#3 (ParallelGC)" os_prio=31 tid=0x00007ff8ea807800 nid=0x2a03 runnable

"VM Periodic Task Thread" os_prio=31 tid=0x00007ff8ea040000 nid=0x4a03 waiting on condition

JNI global references: 6
```

<br/>

### 读懂Thread Dump

---

我们在分析Thread Dump文件时，第二部分的内容是我们分析的重点。而这部分内容中的重点是理解线程的状态和分析线程的stacktrace信息。

<br/>

#### 线程状态

* **runnable**

    线程正在执行，具体在做什么需要看堆栈信息。处于runnable状态的线程不一定会消耗CPU，像socket IO操作，线程正在从网络上读取数据，尽管线程状态runnable，但实际上却在等待网络io，线程绝大多数时间是被挂起的，只有当数据到达后，线程才会被唤起。挂起发生在本地代码（native）中，虚拟机无法知道，不像显式的调用sleep和wait方法，虚拟机才能知道线程的真正状态，但在本地代码中的挂起，虚拟机无法知道真正的线程状态，因此一概显示为runnable。

* **waiting on condition**

    该状态表示线程在等待某个条件的发生，具体是什么原因，需要根据stacktrace来分析。最常见的情况是线程在等待网络的读写，如果网络数据没准备好，线程就等待在那里。如果发现有大量的线程都在处在waiting on condition，从线程stacktrace看，正等待网络读写，这可能是一个网络瓶颈的征兆，因为网络阻塞导致线程无法执行。可能是网络非常忙，几乎消耗了所有的带宽，仍然有大量数据等待网络读写；也可能是网络空闲，但由于路由等问题，导致包无法正常的到达。

    另外一种常见情况是该线程在sleep，等待sleep的时间到了时候，将被唤醒。

* **Waiting for monitor entry与in Object.wait()**

    这两种情况在多线程的情况下经常出现，Java是通过Monitor来实现线程互斥和协作。可以把monitor理解成一个锁，每个对象实例都有且仅有一个。某个时刻monitor只能被一个线程占有，该线程就是Active Thread，而其它线程都是Waiting Thread，Waiting Thread分布在entry set和wait set两个集合中，所下图所示：

    ![monitor]({{site.baseurl}}/pic/threaddump/2.svg)

    当一个线程尝试对一个对象加锁时（比如synchronized），它会进入到entry set。获取对象锁以后，它变为锁的owner，成为Active thread继续执行任务。当线程获得锁后，又发现其它一些条件不满足，这时它可以主动释放锁并等待（调用锁对象的`wait`方法），并进入wait set。当线程在entry set中时，它们的状态就是**Waiting for monitor entry**；而当线程在wait set中时，它们的状态是**in Object.wait()**。

<br/>

#### 一个简单的例子

```java
public static void main(String[] args) {
    Object lock = new Object();
    BlockingQueue<String> queue = new LinkedBlockingQueue<>();

    new Thread(() ->  {
        synchronized(lock) {
            while (true) {
                System.out.println("do something...");
            }
        }
    }, "thread-1").start();

    new Thread(() ->  {
        synchronized(lock) {
            while (true) {
                System.out.println("do something else...");
            }
        }
    }, "thread-2").start();

    new Thread(() ->  {
        try {
            String s = queue.take();
            System.out.println(s);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }, "thread-3").start();

    try {
        TimeUnit.SECONDS.sleep(60);
        System.exit(0);
    } catch (InterruptedException e) {}
}
```

上面这个简单的程序，启动了两个线程（thread-1与thread-2）向控制台打印信息，注意，它们都尝试对`lock`对象加锁，不出意外的话应该是线程thread-1加锁成功，thread-2会一直处于等待状态（即Waiting for monitor entry）。线程thread-3尝试从一个空的阻塞队列里面取出元素，主线程main会sleep60秒（状态应该是waiting on condition），然后中止程序的执行，下面是这段程序的Thread Dump：

```
2017-10-23 17:05:57
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.144-b01 mixed mode):
"thread-3" #11 prio=5 os_prio=31 tid=0x00007fd60d832000 nid=0x4f03 waiting on condition [0x000000011e18f000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x000000074000a178> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
	at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
	at com.personal.oyl.test.App.lambda$2(App.java:34)
	at com.personal.oyl.test.App$$Lambda$3/1044036744.run(Unknown Source)
	at java.lang.Thread.run(Thread.java:748)

"thread-2" #10 prio=5 os_prio=31 tid=0x00007feb9c147800 nid=0x4d03 waiting for monitor entry [0x00000001271e5000]
   java.lang.Thread.State: BLOCKED (on object monitor)
	at com.personal.oyl.test.App.lambda$1(App.java:24)
	- waiting to lock <0x0000000740006918> (a java.lang.Object)
	at com.personal.oyl.test.App$$Lambda$2/1418481495.run(Unknown Source)
	at java.lang.Thread.run(Thread.java:748)

"thread-1" #9 prio=5 os_prio=31 tid=0x00007feb9c022000 nid=0x4b03 runnable [0x00000001270e2000]
   java.lang.Thread.State: RUNNABLE
	at java.io.FileOutputStream.writeBytes(Native Method)
	at java.io.FileOutputStream.write(FileOutputStream.java:326)
	at java.io.BufferedOutputStream.flushBuffer(BufferedOutputStream.java:82)
	at java.io.BufferedOutputStream.flush(BufferedOutputStream.java:140)
	- locked <0x000000074001d380> (a java.io.BufferedOutputStream)
	at java.io.PrintStream.write(PrintStream.java:482)
	- locked <0x00000007400044b0> (a java.io.PrintStream)
	at sun.nio.cs.StreamEncoder.writeBytes(StreamEncoder.java:221)
	at sun.nio.cs.StreamEncoder.implFlushBuffer(StreamEncoder.java:291)
	at sun.nio.cs.StreamEncoder.flushBuffer(StreamEncoder.java:104)
	- locked <0x0000000740004468> (a java.io.OutputStreamWriter)
	at java.io.OutputStreamWriter.flushBuffer(OutputStreamWriter.java:185)
	at java.io.PrintStream.write(PrintStream.java:527)
	- eliminated <0x00000007400044b0> (a java.io.PrintStream)
	at java.io.PrintStream.print(PrintStream.java:669)
	at java.io.PrintStream.println(PrintStream.java:806)
	- locked <0x00000007400044b0> (a java.io.PrintStream)
	at com.personal.oyl.test.App.lambda$0(App.java:16)
	- locked <0x0000000740006918> (a java.lang.Object)
	at com.personal.oyl.test.App$$Lambda$1/471910020.run(Unknown Source)
	at java.lang.Thread.run(Thread.java:748)

"main" #1 prio=5 os_prio=31 tid=0x00007feb9c008800 nid=0xd03 waiting on condition [0x000000010e7c6000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
	at java.lang.Thread.sleep(Native Method)
	at java.lang.Thread.sleep(Thread.java:340)
	at java.util.concurrent.TimeUnit.sleep(TimeUnit.java:386)
	at com.personal.oyl.test.App.main(App.java:30)
```

可以看到，Thread Dump的信息和我们的预期是一样的。线程thread-1成功加锁`locked <0x0000000740006918> (a java.lang.Object)`，并不断写控制台写，注意看stacktrace信息；thread-2尝试对同一对象进行加锁`waiting to lock <0x0000000740006918> (a java.lang.Object)`，因为锁对象已经被thread-1持有，所以thread-2处于等待状态Waiting for monitor entry。线程thread-3尝试从一个空的阻塞队列里面获取元素，会一直阻塞到队列有元素返回为止，所以状态是waiting on condition；主线程main主动调用sleep方法，处于waiting on condition状态。
