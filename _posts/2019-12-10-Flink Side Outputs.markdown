---
layout: post
title:  "Flink Side Outputs"
date:   2019-12-10 08:00:00 +0800
categories: streaming
keywords: streaming
description: Flink Side Outputs
commentId: 2019-12-10
---

In addition to the main stream that results from DataStream operations, you can also produce any number of additional side output result streams. The type of data in the result streams does not have to match the type of data in the main stream and the types of the different side outputs can also differ. This operation can be useful when you want to split a stream of data where you would normally have to replicate the stream and then filter out from each stream the data that you don’t want to have.

When using side outputs, you first need to define an OutputTag that will be used to identify a side output stream:

```java
// this needs to be an anonymous inner class, so that we can analyze the type
OutputTag<String> outputTag = new OutputTag<String>("side-output") {};
```

Notice how the OutputTag is typed according to the type of elements that the side output stream contains.

Emitting data to a side output is possible from the following functions:

* ProcessFunction
* KeyedProcessFunction
* CoProcessFunction
* KeyedCoProcessFunction
* ProcessWindowFunction
* ProcessAllWindowFunction

You can use the Context parameter, which is exposed to users in the above functions, to emit data to a side output identified by an OutputTag. Here is an example of emitting side output data from a ProcessFunction:

```java
DataStream<Integer> input = ...;

final OutputTag<String> outputTag = new OutputTag<String>("side-output"){};

SingleOutputStreamOperator<Integer> mainDataStream = input
  .process(new ProcessFunction<Integer, Integer>() {

      @Override
      public void processElement(
          Integer value,
          Context ctx,
          Collector<Integer> out) throws Exception {
        // emit data to regular output
        out.collect(value);

        // emit data to side output
        ctx.output(outputTag, "sideout-" + String.valueOf(value));
      }
    });
```

For retrieving the side output stream you use `getSideOutput(OutputTag)` on the result of the DataStream operation. This will give you a DataStream that is typed to the result of the side output stream:

```java
final OutputTag<String> outputTag = new OutputTag<String>("side-output"){};

SingleOutputStreamOperator<Integer> mainDataStream = ...;

DataStream<String> sideOutputStream = mainDataStream.getSideOutput(outputTag);
```
