---
layout: post
title:  "CompletableFuture"
date:   2023-08-02 08:00:00 +0800
categories: java
keywords: java
description: java
commentId: 2023-08-02
---
CompletableFuture是JDK 1.8引入的实现类，该类实现了Future和CompletionStage两个接口。该类的实例作为一个异步任务，可以在自己异步执行完成之后触发一些其他的异步任务，从而达到异步回调的效果。

Future接口大家已经非常熟悉了，接下来介绍一下CompletionStage接口。CompletionStage代表异步计算过程中的某一个阶段，一个阶段完成以后可能会进入另一个阶段。一个阶段可以理解为一个子任务，每一个子任务会包装一个Java函数式接口实例，表示该子任务所要执行的操作。

<center><img src="{{site.baseurl}}/pic/completablefuture/1.svg" width="40%"/></center>

CompletionStage代表某个同步或者异步计算的一个阶段，或者一系列异步任务中的一个子任务（或者阶段性任务）。每个CompletionStage子任务所包装的可以是一个Function、Consumer或者Runnable函数式接口实例。

多个CompletionStage构成了一条任务流水线，一个环节执行完成了可以将结果移交给下一个环节（子任务）。多个CompletionStage子任务之间可以使用链式调用。下面是一个简单的例子：

```java
    CompletableFuture<Integer> f = CompletableFuture.completedFuture(10);
    f.thenApply(x -> x + x)
     .thenAccept(System.out::println)
     .thenRun(() -> System.out.println("Over"));
```

CompletionStage代表异步计算过程中的某一个阶段，一个阶段完成以后可能会触发另一个阶段。虽然一个子任务可以触发其他子任务，但是并不能保证后续子任务的执行顺序。

CompletionStage子任务的创建是通过CompletableFuture完成的。CompletableFuture类提供了非常强大的Future的扩展功能来帮助我们简化异步编程的复杂性，提供了函数式编程的能力来帮助我们通过回调的方式处理计算结果，也提供了转换和组合CompletionStage()的方法。

```java
    CompletableFuture<Integer> f = CompletableFuture.completedFuture(10);
    CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "hello");
    CompletableFuture<Void> f3 = CompletableFuture.runAsync(() -> {});
```

## 异步任务的串行执行

如果两个异步任务需要串行（一个任务依赖另一个任务）执行，可以通过CompletionStage接口的thenApply()、thenAccept()、thenRun()和thenCompose()四个方法来实现。

```java
    CompletableFuture<Integer> f = CompletableFuture.completedFuture(10);
    CompletableFuture<Integer> f2 = CompletableFuture.completedFuture(20);
    f.thenCompose(x -> f2.thenApply(y -> x + y)).thenAccept(System.out::println);
    f.join();
```

**关于thenApply与thenCompose，两个方法的返回值都是CompletionStage**

* 对于thenApply，fn函数是一个对一个已完成的stage或者说CompletableFuture的返回值进行计算、操作；

* 对于thenCompose，fn函数是对另一个CompletableFuture进行计算、操作。


## 异步任务的合并执行

如果某个任务同时依赖另外两个异步任务的执行结果，就需要对另外两个异步任务进行合并。以泡茶喝为例，“泡茶喝”任务需要对“烧水”任务与“清洗”任务进行合并。对两个异步任务的合并可以通过CompletionStage接口的thenCombine()、runAfterBoth()、thenAcceptBoth()三个方法来实现。

```java
    CompletableFuture<Integer> f = CompletableFuture.completedFuture(10);
    CompletableFuture<Long> f2 = CompletableFuture.completedFuture(20L);

    f.thenCombine(f2, (x, y) -> x + y).thenAccept(System.out::println);
    f.thenAcceptBoth(f2, (x, y) -> System.out.println(x + y));
    f.runAfterBoth(f2, () -> System.out.println("done"));
```

CompletionStage接口的allOf()会等待所有的任务结束，以合并所有的任务。

```java
    List<Integer> src = Stream.of(1,2,3,4,5).collect(Collectors.toList());

    CompletableFuture<?>[] futures = src.stream().map(i -> 
            CompletableFuture.supplyAsync(() -> {
            try {
                TimeUnit.SECONDS.sleep(i);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return i;
        })
    ).toArray(CompletableFuture[]::new);

    CompletableFuture<Void> all = CompletableFuture.allOf(futures);
    all.join();
```

## 异步任务的选择执行

CompletableFuture对异步任务的选择执行不是按照某种条件进行选择的，而是按照执行速度进行选择的：前面两个并行任务，谁的结果返回速度快，谁的结果将作为第三步任务的输入。对两个异步任务的选择可以通过CompletionStage接口的applyToEither()、runAfterEither()和acceptEither()三个方法来实现。

```java
    CompletableFuture<Integer> f = CompletableFuture.completedFuture(10);
    CompletableFuture<Integer> f2 = CompletableFuture.completedFuture(20);

    f.applyToEither(f2, x -> x + 1).thenAccept(System.out::println);
    f.acceptEither(f2, System.out::println);
    f.runAfterEither(f2, () -> System.out.println("done"));
```
