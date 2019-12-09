---
layout: post
title:  "Flink Windows"
date:   2019-12-08 08:00:00 +0800
categories: streaming
keywords: streaming
description: Flink Windows
commentId: 2019-12-08
---

Windows are at the heart of processing infinite streams. Windows split the stream into “buckets” of finite size, over which we can apply computations. This document focuses on how windowing is performed in Flink and how the programmer can benefit to the maximum from its offered functionality.

The general structure of a windowed Flink program is presented below. The first snippet refers to keyed streams, while the second to non-keyed ones. As one can see, the only difference is the `keyBy(...)` call for the keyed streams and the `window(...)` which becomes `windowAll(...)` for non-keyed streams.

**Keyed Windows**

```
stream
       .keyBy(...)               <-  keyed versus non-keyed windows
       .window(...)              <-  required: "assigner"
      [.trigger(...)]            <-  optional: "trigger" (else default trigger)
      [.evictor(...)]            <-  optional: "evictor" (else no evictor)
      [.allowedLateness(...)]    <-  optional: "lateness" (else zero)
      [.sideOutputLateData(...)] <-  optional: "output tag" (else no side output for late data)
       .reduce/aggregate/fold/apply()      <-  required: "function"
      [.getSideOutput(...)]      <-  optional: "output tag"
```

**Non-Keyed Windows**

```
stream
       .windowAll(...)           <-  required: "assigner"
      [.trigger(...)]            <-  optional: "trigger" (else default trigger)
      [.evictor(...)]            <-  optional: "evictor" (else no evictor)
      [.allowedLateness(...)]    <-  optional: "lateness" (else zero)
      [.sideOutputLateData(...)] <-  optional: "output tag" (else no side output for late data)
       .reduce/aggregate/fold/apply()      <-  required: "function"
      [.getSideOutput(...)]      <-  optional: "output tag"
```
In the above, the commands in square brackets ([…]) are optional. This reveals that Flink allows you to customize your windowing logic in many different ways so that it best fits your needs.

## Window Lifecycle

---

In a nutshell, a window is **created** as soon as the **first element** that should belong to this window arrives, and the window is completely **removed** when the time (event or processing time) passes its end timestamp plus the user-specified allowed lateness. Flink guarantees removal only for time-based windows and not for other types, e.g. global windows. For example, with an event-time-based windowing strategy that creates non-overlapping (or tumbling) windows every 5 minutes and has an allowed lateness of 1 min, Flink will create a new window for the interval between 12:00 and 12:05 when the first element with a timestamp that falls into this interval arrives, and it will remove it when the watermark passes the 12:06 timestamp.

In addition, each window will have a **Trigger** and a **Function** (ProcessWindowFunction, ReduceFunction, AggregateFunction or FoldFunction) attached to it. The function will contain the computation to be applied to the contents of the window, while the Trigger specifies the conditions under which the window is considered ready for the function to be applied. A triggering policy might be something like “when the number of elements in the window is more than 4”, or “when the watermark passes the end of the window”. A trigger can also decide to purge a window’s contents any time between its creation and removal. Purging in this case only refers to the elements in the window, and not the window metadata. This means that new data can still be added to that window.

Apart from the above, you can specify an **Evictor** which will be able to remove elements from the window after the trigger fires and before and/or after the function is applied.

## Keyed vs Non-Keyed Windows

---

The first thing to specify is whether your stream should be keyed or not. This has to be done before defining the window. Using the `keyBy(...)` will split your infinite stream into logical keyed streams. If `keyBy(...)` is not called, your stream is not keyed.

In the case of keyed streams, any attribute of your incoming events can be used as a key. Having a keyed stream will allow your windowed computation to be performed in parallel by multiple tasks, as each logical keyed stream can be processed independently from the rest. All elements referring to the same key will be sent to the same parallel task.

In case of non-keyed streams, your original stream will not be split into multiple logical streams and all the windowing logic will be performed by a single task, i.e. with parallelism of 1.

## Window Assigners

---

After specifying whether your stream is keyed or not, the next step is to define a window assigner. The window assigner defines how elements are assigned to windows. This is done by specifying the WindowAssigner of your choice in the `window(...)` (for keyed streams) or the `windowAll()` (for non-keyed streams) call.

A WindowAssigner is responsible for assigning each incoming element to one or more windows. Flink comes with pre-defined window assigners for the most common use cases, namely tumbling windows, sliding windows, session windows and global windows. You can also implement a custom window assigner by extending the `WindowAssigner` class. All built-in window assigners (except the global windows) assign elements to windows based on time, which can either be processing time or event time.

Time-based windows have a start timestamp (inclusive) and an end timestamp (exclusive) that together describe the size of the window. In code, Flink uses TimeWindow when working with time-based windows which has methods for querying the start- and end-timestamp and also an additional method `maxTimestamp()` that returns the largest allowed timestamp for a given windows.

In the following, we show how Flink’s pre-defined window assigners work and how they are used in a DataStream program. The following figures visualize the workings of each assigner. The purple circles represent elements of the stream, which are partitioned by some key (in this case user 1, user 2 and user 3). The x-axis shows the progress of time.

### Tumbling Windows

---

A tumbling windows assigner assigns each element to a window of a specified window size. Tumbling windows have a fixed size and do not overlap. For example, if you specify a tumbling window with a size of 5 minutes, the current window will be evaluated and a new window will be started every five minutes as illustrated by the following figure.

<center><img src="{{site.baseurl}}/pic/flink-windows/1.svg" width="80%"/></center>

The following code snippets show how to use tumbling windows.

```java
DataStream<T> input = ...;

// tumbling event-time windows
input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.seconds(5)))
    .<windowed transformation>(<window function>);

// tumbling processing-time windows
input
    .keyBy(<key selector>)
    .window(TumblingProcessingTimeWindows.of(Time.seconds(5)))
    .<windowed transformation>(<window function>);

// daily tumbling event-time windows offset by -8 hours.
input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.days(1), Time.hours(-8)))
    .<windowed transformation>(<window function>);
```

Time intervals can be specified by using one of `Time.milliseconds(x)`, `Time.seconds(x)`, `Time.minutes(x)`, and so on.

As shown in the last example, tumbling window assigners also take an optional offset parameter that can be used to change the alignment of windows. For example, without offsets hourly tumbling windows are aligned with epoch, that is you will get windows such as 1:00:00.000 - 1:59:59.999, 2:00:00.000 - 2:59:59.999 and so on. If you want to change that you can give an offset. With an offset of 15 minutes you would, for example, get 1:15:00.000 - 2:14:59.999, 2:15:00.000 - 3:14:59.999 etc. An important use case for offsets is to **adjust windows to timezones other than UTC-0**. For example, in China you would have to specify an offset of Time.hours(-8).

### Sliding Windows

---

The sliding windows assigner assigns elements to windows of fixed length. Similar to a tumbling windows assigner, the size of the windows is configured by the window size parameter. An additional window slide parameter controls how frequently a sliding window is started. Hence, sliding windows can be overlapping if the slide is smaller than the window size. In this case elements are assigned to multiple windows.

For example, you could have windows of size 10 minutes that slides by 5 minutes. With this you get every 5 minutes a window that contains the events that arrived during the last 10 minutes as depicted by the following figure.

<center><img src="{{site.baseurl}}/pic/flink-windows/2.svg" width="80%"/></center>

The following code snippets show how to use sliding windows.

```java
DataStream<T> input = ...;

// sliding event-time windows
input
    .keyBy(<key selector>)
    .window(SlidingEventTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .<windowed transformation>(<window function>);

// sliding processing-time windows
input
    .keyBy(<key selector>)
    .window(SlidingProcessingTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .<windowed transformation>(<window function>);

// sliding processing-time windows offset by -8 hours
input
    .keyBy(<key selector>)
    .window(SlidingProcessingTimeWindows.of(Time.hours(12), Time.hours(1), Time.hours(-8)))
    .<windowed transformation>(<window function>);
```

Time intervals can be specified by using one of `Time.milliseconds(x)`, `Time.seconds(x)`, `Time.minutes(x)`, and so on.

As shown in the last example, sliding window assigners also take an optional offset parameter that can be used to change the alignment of windows. For example, without offsets hourly windows sliding by 30 minutes are aligned with epoch, that is you will get windows such as 1:00:00.000 - 1:59:59.999, 1:30:00.000 - 2:29:59.999 and so on. If you want to change that you can give an offset. With an offset of 15 minutes you would, for example, get 1:15:00.000 - 2:14:59.999, 1:45:00.000 - 2:44:59.999 etc.

### Session Windows

---

The session windows assigner groups elements by sessions of activity. Session windows do not overlap and do not have a fixed start and end time, in contrast to tumbling windows and sliding windows. Instead a session window closes when it does not receive elements for a certain period of time, i.e., when a gap of inactivity occurred. A session window assigner can be configured with either a static session gap or with a session gap extractor function which defines how long the period of inactivity is. When this period expires, the current session closes and subsequent elements are assigned to a new session window.

<center><img src="{{site.baseurl}}/pic/flink-windows/3.svg" width="80%"/></center>

The following code snippets show how to use session windows.

```java
DataStream<T> input = ...;

// event-time session windows with static gap
input
    .keyBy(<key selector>)
    .window(EventTimeSessionWindows.withGap(Time.minutes(10)))
    .<windowed transformation>(<window function>);

// event-time session windows with dynamic gap
input
    .keyBy(<key selector>)
    .window(EventTimeSessionWindows.withDynamicGap((element) -> {
        // determine and return session gap
    }))
    .<windowed transformation>(<window function>);

// processing-time session windows with static gap
input
    .keyBy(<key selector>)
    .window(ProcessingTimeSessionWindows.withGap(Time.minutes(10)))
    .<windowed transformation>(<window function>);

// processing-time session windows with dynamic gap
input
    .keyBy(<key selector>)
    .window(ProcessingTimeSessionWindows.withDynamicGap((element) -> {
        // determine and return session gap
    }))
    .<windowed transformation>(<window function>);
```

Static gaps can be specified by using one of `Time.milliseconds(x)`, `Time.seconds(x)`, `Time.minutes(x)`, and so on.

Dynamic gaps are specified by implementing the `SessionWindowTimeGapExtractor` interface.

<kbd>Attention</kbd> Since session windows do not have a fixed start and end, they are evaluated differently than tumbling and sliding windows. Internally, a session window operator creates a new window for each arriving record and merges windows together if they are closer to each other than the defined gap. In order to be mergeable, a session window operator requires a merging Trigger and a merging Window Function, such as ReduceFunction, AggregateFunction, or ProcessWindowFunction (FoldFunction cannot merge.)

### Global Windows

---

A global windows assigner assigns all elements with the same key to the same single global window. This windowing scheme is only useful if you also specify a custom trigger. Otherwise, no computation will be performed, as the global window does not have a natural end at which we could process the aggregated elements.

<center><img src="{{site.baseurl}}/pic/flink-windows/4.svg" width="80%"/></center>

The following code snippets show how to use a global window.

```java
Stream<T> input = ...;

input
    .keyBy(<key selector>)
    .window(GlobalWindows.create())
    .<windowed transformation>(<window function>);
```

## Window Functions

---

After defining the window assigner, we need to specify the computation that we want to perform on each of these windows. This is the responsibility of the window function, which is used to process the elements of each (possibly keyed) window once the system determines that a window is ready for processing.

The window function can be one of ReduceFunction, AggregateFunction, FoldFunction or ProcessWindowFunction. The first three can be executed more efficiently (see State Size section) because Flink can incrementally aggregate the elements for each window as they arrive. A ProcessWindowFunction gets an Iterable for all the elements contained in a window and additional meta information about the window to which the elements belong.

A windowed transformation with a ProcessWindowFunction cannot be executed as efficiently as the other cases because Flink has to buffer all elements for a window internally before invoking the function. This can be mitigated by combining a ProcessWindowFunction with a ReduceFunction, AggregateFunction, or FoldFunction to get both incremental aggregation of window elements and the additional window metadata that the ProcessWindowFunction receives.

### ReduceFunction

---

A ReduceFunction specifies how two elements from the input are combined to produce an output element of the same type. Flink uses a ReduceFunction to incrementally aggregate the elements of a window.

A ReduceFunction can be defined and used like this:

```java
DataStream<Tuple2<String, Long>> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .reduce(new ReduceFunction<Tuple2<String, Long>> {
      public Tuple2<String, Long> reduce(Tuple2<String, Long> v1, Tuple2<String, Long> v2) {
        return new Tuple2<>(v1.f0, v1.f1 + v2.f1);
      }
    });
```

The above example sums up the second fields of the tuples for all elements in a window.

### AggregateFunction

---

An AggregateFunction is a generalized version of a ReduceFunction that has three types: an input type (IN), accumulator type (ACC), and an output type (OUT). The input type is the type of elements in the input stream and the AggregateFunction has a method for adding one input element to an accumulator. The interface also has methods for creating an initial accumulator, for merging two accumulators into one accumulator and for extracting an output (of type OUT) from an accumulator. We will see how this works in the example below.

Same as with ReduceFunction, Flink will incrementally aggregate input elements of a window as they arrive.

An AggregateFunction can be defined and used like this:

```java
/**
 * The accumulator is used to keep a running sum and a count. The {@code getResult} method
 * computes the average.
 */
private static class AverageAggregate
    implements AggregateFunction<Tuple2<String, Long>, Tuple2<Long, Long>, Double> {
  @Override
  public Tuple2<Long, Long> createAccumulator() {
    return new Tuple2<>(0L, 0L);
  }

  @Override
  public Tuple2<Long, Long> add(Tuple2<String, Long> value, Tuple2<Long, Long> accumulator) {
    return new Tuple2<>(accumulator.f0 + value.f1, accumulator.f1 + 1L);
  }

  @Override
  public Double getResult(Tuple2<Long, Long> accumulator) {
    return ((double) accumulator.f0) / accumulator.f1;
  }

  @Override
  public Tuple2<Long, Long> merge(Tuple2<Long, Long> a, Tuple2<Long, Long> b) {
    return new Tuple2<>(a.f0 + b.f0, a.f1 + b.f1);
  }
}

DataStream<Tuple2<String, Long>> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .aggregate(new AverageAggregate());
```

The above example computes the average of the second field of the elements in the window.

### FoldFunction

---

A FoldFunction specifies how an input element of the window is combined with an element of the output type. The FoldFunction is incrementally called for each element that is added to the window and the current output value. The first element is combined with a pre-defined initial value of the output type.

A FoldFunction can be defined and used like this:

```java
DataStream<Tuple2<String, Long>> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .fold("", new FoldFunction<Tuple2<String, Long>, String>> {
       public String fold(String acc, Tuple2<String, Long> value) {
         return acc + value.f1;
       }
    });
```

The above example appends all input Long values to an initially empty String.

<kbd>Attention</kbd> fold() cannot be used with session windows or other mergeable windows.

### ProcessWindowFunction

---

A ProcessWindowFunction gets an Iterable containing all the elements of the window, and a Context object with access to time and state information, which enables it to provide more flexibility than other window functions. This comes at the cost of performance and resource consumption, because elements cannot be incrementally aggregated but instead need to be buffered internally until the window is considered ready for processing.

The signature of ProcessWindowFunction looks as follows:

```java
public abstract class ProcessWindowFunction<IN, OUT, KEY, W extends Window> implements Function {

    /**
     * Evaluates the window and outputs none or several elements.
     *
     * @param key The key for which this window is evaluated.
     * @param context The context in which the window is being evaluated.
     * @param elements The elements in the window being evaluated.
     * @param out A collector for emitting elements.
     *
     * @throws Exception The function may throw exceptions to fail the program and trigger recovery.
     */
    public abstract void process(
            KEY key,
            Context context,
            Iterable<IN> elements,
            Collector<OUT> out) throws Exception;

    /**
     * The context holding window metadata.
     */
    public abstract class Context implements java.io.Serializable {
        /**
         * Returns the window that is being evaluated.
         */
        public abstract W window();

        /** Returns the current processing time. */
        public abstract long currentProcessingTime();

        /** Returns the current event-time watermark. */
        public abstract long currentWatermark();

        /**
         * State accessor for per-key and per-window state.
         *
         * <p><b>NOTE:</b>If you use per-window state you have to ensure that you clean it up
         * by implementing {@link ProcessWindowFunction#clear(Context)}.
         */
        public abstract KeyedStateStore windowState();

        /**
         * State accessor for per-key global state.
         */
        public abstract KeyedStateStore globalState();
    }

}
```

<kbd>Note</kbd> The key parameter is the key that is extracted via the KeySelector that was specified for the keyBy() invocation. In case of tuple-index keys or string-field references this key type is always Tuple and you have to manually cast it to a tuple of the correct size to extract the key fields.

A ProcessWindowFunction can be defined and used like this:

```java
DataStream<Tuple2<String, Long>> input = ...;

input
  .keyBy(t -> t.f0)
  .timeWindow(Time.minutes(5))
  .process(new MyProcessWindowFunction());

/* ... */

public class MyProcessWindowFunction
    extends ProcessWindowFunction<Tuple2<String, Long>, String, String, TimeWindow> {

  @Override
  public void process(String key, Context context, Iterable<Tuple2<String, Long>> input, Collector<String> out) {
    long count = 0;
    for (Tuple2<String, Long> in: input) {
      count++;
    }
    out.collect("Window: " + context.window() + "count: " + count);
  }
}
```

The example shows a ProcessWindowFunction that counts the elements in a window. In addition, the window function adds information about the window to the output.

<kbd>Attention</kbd> Note that using ProcessWindowFunction for simple aggregates such as count is quite inefficient.

### ProcessWindowFunction with Incremental Aggregation

---

A ProcessWindowFunction can be combined with either a ReduceFunction, an AggregateFunction, or a FoldFunction to incrementally aggregate elements as they arrive in the window. When the window is closed, the ProcessWindowFunction will be provided with the aggregated result. This allows it to incrementally compute windows while having access to the additional window meta information of the ProcessWindowFunction.

#### Incremental Window Aggregation with ReduceFunction

The following example shows how an incremental ReduceFunction can be combined with a ProcessWindowFunction to return the smallest event in a window along with the start time of the window.

```java
DataStream<SensorReading> input = ...;

input
  .keyBy(<key selector>)
  .timeWindow(<duration>)
  .reduce(new MyReduceFunction(), new MyProcessWindowFunction());

// Function definitions

private static class MyReduceFunction implements ReduceFunction<SensorReading> {

  public SensorReading reduce(SensorReading r1, SensorReading r2) {
      return r1.value() > r2.value() ? r2 : r1;
  }
}

private static class MyProcessWindowFunction
    extends ProcessWindowFunction<SensorReading, Tuple2<Long, SensorReading>, String, TimeWindow> {

  public void process(String key,
                    Context context,
                    Iterable<SensorReading> minReadings,
                    Collector<Tuple2<Long, SensorReading>> out) {
      SensorReading min = minReadings.iterator().next();
      out.collect(new Tuple2<Long, SensorReading>(context.window().getStart(), min));
  }
}
```

#### Incremental Window Aggregation with AggregateFunction

The following example shows how an incremental AggregateFunction can be combined with a ProcessWindowFunction to compute the average and also emit the key and window along with the average.

```java
DataStream<Tuple2<String, Long>> input = ...;

input
  .keyBy(<key selector>)
  .timeWindow(<duration>)
  .aggregate(new AverageAggregate(), new MyProcessWindowFunction());

// Function definitions

/**
 * The accumulator is used to keep a running sum and a count. The {@code getResult} method
 * computes the average.
 */
private static class AverageAggregate
    implements AggregateFunction<Tuple2<String, Long>, Tuple2<Long, Long>, Double> {
  @Override
  public Tuple2<Long, Long> createAccumulator() {
    return new Tuple2<>(0L, 0L);
  }

  @Override
  public Tuple2<Long, Long> add(Tuple2<String, Long> value, Tuple2<Long, Long> accumulator) {
    return new Tuple2<>(accumulator.f0 + value.f1, accumulator.f1 + 1L);
  }

  @Override
  public Double getResult(Tuple2<Long, Long> accumulator) {
    return ((double) accumulator.f0) / accumulator.f1;
  }

  @Override
  public Tuple2<Long, Long> merge(Tuple2<Long, Long> a, Tuple2<Long, Long> b) {
    return new Tuple2<>(a.f0 + b.f0, a.f1 + b.f1);
  }
}

private static class MyProcessWindowFunction
    extends ProcessWindowFunction<Double, Tuple2<String, Double>, String, TimeWindow> {

  public void process(String key,
                    Context context,
                    Iterable<Double> averages,
                    Collector<Tuple2<String, Double>> out) {
      Double average = averages.iterator().next();
      out.collect(new Tuple2<>(key, average));
  }
}
```

#### Incremental Window Aggregation with FoldFunction

The following example shows how an incremental FoldFunction can be combined with a ProcessWindowFunction to extract the number of events in the window and return also the key and end time of the window.

```java
DataStream<SensorReading> input = ...;

input
  .keyBy(<key selector>)
  .timeWindow(<duration>)
  .fold(new Tuple3<String, Long, Integer>("",0L, 0), new MyFoldFunction(), new MyProcessWindowFunction())

// Function definitions

private static class MyFoldFunction
    implements FoldFunction<SensorReading, Tuple3<String, Long, Integer> > {

  public Tuple3<String, Long, Integer> fold(Tuple3<String, Long, Integer> acc, SensorReading s) {
      Integer cur = acc.getField(2);
      acc.setField(cur + 1, 2);
      return acc;
  }
}

private static class MyProcessWindowFunction
    extends ProcessWindowFunction<Tuple3<String, Long, Integer>, Tuple3<String, Long, Integer>, String, TimeWindow> {

  public void process(String key,
                    Context context,
                    Iterable<Tuple3<String, Long, Integer>> counts,
                    Collector<Tuple3<String, Long, Integer>> out) {
    Integer count = counts.iterator().next().getField(2);
    out.collect(new Tuple3<String, Long, Integer>(key, context.window().getEnd(),count));
  }
}
```

### Using per-window state in ProcessWindowFunction

---

In addition to accessing keyed state (as any rich function can) a ProcessWindowFunction can also use keyed state that is scoped to the window that the function is currently processing. In this context it is important to understand what the window that per-window state is referring to is. There are different “windows” involved:

* The window that was defined when specifying the windowed operation: This might be tumbling windows of 1 hour or sliding windows of 2 hours that slide by 1 hour.

* An actual instance of a defined window for a given key: This might be time window from 12:00 to 13:00 for user-id xyz. This is based on the window definition and there will be many windows based on the number of keys that the job is currently processing and based on what time slots the events fall into.

Per-window state is tied to the latter of those two. Meaning that if we process events for 1000 different keys and events for all of them currently fall into the [12:00, 13:00) time window then there will be 1000 window instances that each have their own keyed per-window state.

There are two methods on the Context object that a process() invocation receives that allow access to the two types of state:

* `globalState()`, which allows access to keyed state that is not scoped to a window

* `windowState()`, which allows access to keyed state that is also scoped to the window

This feature is helpful if you anticipate multiple firing for the same window, as can happen when you have late firings for data that arrives late or when you have a custom trigger that does speculative early firings. In such a case you would store information about previous firings or the number of firings in per-window state.

When using windowed state it is important to also clean up that state when a window is cleared. This should happen in the `clear()` method.

## Triggers

---

A Trigger determines when a window (as formed by the window assigner) is ready to be processed by the window function. Each WindowAssigner comes with a default Trigger. If the default trigger does not fit your needs, you can specify a custom trigger using `trigger(...)`.

The trigger interface has five methods that allow a Trigger to react to different events:

* The `onElement()` method is called for each element that is added to a window.
* The `onEventTime()` method is called when a registered event-time timer fires.
* The `onProcessingTime()` method is called when a registered processing-time timer fires.
* The `onMerge()` method is relevant for stateful triggers and merges the states of two triggers when their corresponding windows merge, e.g. when using session windows.
* Finally the `clear()` method performs any action needed upon removal of the corresponding window.

Two things to notice about the above methods are:

1) The first three decide how to act on their invocation event by returning a TriggerResult. The action can be one of the following:

* CONTINUE: do nothing,
* FIRE: trigger the computation,
* PURGE: clear the elements in the window, and
* FIRE_AND_PURGE: trigger the computation and clear the elements in the window afterwards.

2) Any of these methods can be used to register processing- or event-time timers for future actions.

### Fire and Purge

---

Once a trigger determines that a window is ready for processing, it fires, i.e., it returns FIRE or FIRE_AND_PURGE. This is the signal for the window operator to emit the result of the current window. Given a window with a ProcessWindowFunction all elements are passed to the ProcessWindowFunction (possibly after passing them to an evictor). Windows with ReduceFunction, AggregateFunction, or FoldFunction simply emit their eagerly aggregated result.

When a trigger fires, it can either FIRE or FIRE_AND_PURGE. While FIRE keeps the contents of the window, FIRE_AND_PURGE removes its content. By default, the pre-implemented triggers simply FIRE without purging the window state.

Attention Purging will simply remove the contents of the window and will leave any potential meta-information about the window and any trigger state intact.

### Default Triggers of WindowAssigners

---

The default Trigger of a WindowAssigner is appropriate for many use cases. For example, all the event-time window assigners have an EventTimeTrigger as default trigger. This trigger simply fires once the watermark passes the end of a window.

<kbd>Attention</kbd> The default trigger of the GlobalWindow is the NeverTrigger which does never fire. Consequently, you always have to define a custom trigger when using a GlobalWindow.

<kbd>Attention</kbd> By specifying a trigger using `trigger()` you are overwriting the default trigger of a WindowAssigner. For example, if you specify a CountTrigger for TumblingEventTimeWindows you will no longer get window firings based on the progress of time but only by count. Right now, you have to write your own custom trigger if you want to react based on both time and count.

### Built-in and Custom Triggers

---

Flink comes with a few built-in triggers.

* The (already mentioned) EventTimeTrigger fires based on the progress of event-time as measured by watermarks.
* The ProcessingTimeTrigger fires based on processing time.
* The CountTrigger fires once the number of elements in a window exceeds the given limit.
* The PurgingTrigger takes as argument another trigger and transforms it into a purging one.

## Evictors

---

Flink’s windowing model allows specifying an optional Evictor in addition to the WindowAssigner and the Trigger. This can be done using the `evictor(...)` method. The evictor has the ability to remove elements from a window after the trigger fires and before and/or after the window function is applied. To do so, the Evictor interface has two methods:

```java
/**
 * Optionally evicts elements. Called before windowing function.
 *
 * @param elements The elements currently in the pane.
 * @param size The current number of elements in the pane.
 * @param window The {@link Window}
 * @param evictorContext The context for the Evictor
 */
void evictBefore(Iterable<TimestampedValue<T>> elements, int size, W window, EvictorContext evictorContext);

/**
 * Optionally evicts elements. Called after windowing function.
 *
 * @param elements The elements currently in the pane.
 * @param size The current number of elements in the pane.
 * @param window The {@link Window}
 * @param evictorContext The context for the Evictor
 */
void evictAfter(Iterable<TimestampedValue<T>> elements, int size, W window, EvictorContext evictorContext);
```

The `evictBefore()` contains the eviction logic to be applied before the window function, while the `evictAfter()` contains the one to be applied after the window function. Elements evicted before the application of the window function will not be processed by it.

Flink comes with three pre-implemented evictors. These are:

* CountEvictor: keeps up to a user-specified number of elements from the window and discards the remaining ones from the beginning of the window buffer.
* DeltaEvictor: takes a DeltaFunction and a threshold, computes the delta between the last element in the window buffer and each of the remaining ones, and removes the ones with a delta greater or equal to the threshold.
* TimeEvictor: takes as argument an interval in milliseconds and for a given window, it finds the maximum timestamp max_ts among its elements and removes all the elements with timestamps smaller than max_ts - interval.

<kbd>Default</kbd> By default, all the pre-implemented evictors apply their logic before the window function.

<kbd>Attention</kbd> Specifying an evictor prevents any pre-aggregation, as all the elements of a window have to be passed to the evictor before applying the computation.

<kbd>Attention</kbd> Flink provides no guarantees about the order of the elements within a window. This implies that although an evictor may remove elements from the beginning of the window, these are not necessarily the ones that arrive first or last.

## Allowed Lateness

---

When working with event-time windowing, it can happen that elements arrive late, i.e. the watermark that Flink uses to keep track of the progress of event-time is already past the end timestamp of a window to which an element belongs.

By default, late elements are dropped when the watermark is past the end of the window. However, Flink allows to specify a maximum allowed lateness for window operators. Allowed lateness specifies by how much time elements can be late before they are dropped, and its default value is 0. Elements that arrive after the watermark has passed the end of the window but before it passes the end of the window plus the allowed lateness, are still added to the window. Depending on the trigger used, a late but not dropped element may cause the window to fire again. This is the case for the EventTimeTrigger.

In order to make this work, Flink keeps the state of windows until their allowed lateness expires. Once this happens, Flink removes the window and deletes its state.

Default By default, the allowed lateness is set to 0. That is, elements that arrive behind the watermark will be dropped.

You can specify an allowed lateness like this:

```java
DataStream<T> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .allowedLateness(<time>)
    .<windowed transformation>(<window function>);
```

<kbd>Note</kbd> When using the GlobalWindows window assigner no data is ever considered late because the end timestamp of the global window is `Long.MAX_VALUE`.

### Getting late data as a side output

---

Using Flink’s side output feature you can get a stream of the data that was discarded as late.

You first need to specify that you want to get late data using `sideOutputLateData(OutputTag)` on the windowed stream. Then, you can get the side-output stream on the result of the windowed operation:

```java
final OutputTag<T> lateOutputTag = new OutputTag<T>("late-data"){};

DataStream<T> input = ...;

SingleOutputStreamOperator<T> result = input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .allowedLateness(<time>)
    .sideOutputLateData(lateOutputTag)
    .<windowed transformation>(<window function>);

DataStream<T> lateStream = result.getSideOutput(lateOutputTag);
```

### Late elements considerations

---

When specifying an allowed lateness greater than 0, the window along with its content is kept after the watermark passes the end of the window. In these cases, when a late but not dropped element arrives, it could trigger another firing for the window. These firings are called late firings, as they are triggered by late events and in contrast to the main firing which is the first firing of the window. In case of session windows, late firings can further lead to merging of windows, as they may “bridge” the gap between two pre-existing, unmerged windows.

<kbd>Attention</kbd> You should be aware that the elements emitted by a late firing should be treated as updated results of a previous computation, i.e., your data stream will contain multiple results for the same computation. Depending on your application, you need to take these duplicated results into account or deduplicate them.

## Working with window results

---

The result of a windowed operation is again a DataStream, no information about the windowed operations is retained in the result elements so if you want to keep meta-information about the window you have to manually encode that information in the result elements in your ProcessWindowFunction. The only relevant information that is set on the result elements is the element timestamp. This is set to the maximum allowed timestamp of the processed window, which is end timestamp - 1, since the window-end timestamp is exclusive. Note that this is true for both event-time windows and processing-time windows. i.e. after a windowed operations elements always have a timestamp, but this can be an event-time timestamp or a processing-time timestamp. For processing-time windows this has no special implications but for event-time windows this together with how watermarks interact with windows enables consecutive windowed operations with the same window sizes. We will cover this after taking a look how watermarks interact with windows.

### Interaction of watermarks and windows

---

Before continuing in this section you might want to take a look at our section about event time and watermarks.

When watermarks arrive at the window operator this triggers two things:

* the watermark triggers computation of all windows where the maximum timestamp (which is end-timestamp - 1) is smaller than the new watermark
* the watermark is forwarded (as is) to downstream operations

Intuitively, a watermark “flushes” out any windows that would be considered late in downstream operations once they receive that watermark.

### Consecutive windowed operations

---

As mentioned before, the way the timestamp of windowed results is computed and how watermarks interact with windows allows stringing together consecutive windowed operations. This can be useful when you want to do two consecutive windowed operations where you want to use different keys but still want elements from the same upstream window to end up in the same downstream window. Consider this example:

```java
DataStream<Integer> input = ...;

DataStream<Integer> resultsPerKey = input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.seconds(5)))
    .reduce(new Summer());

DataStream<Integer> globalResults = resultsPerKey
    .windowAll(TumblingEventTimeWindows.of(Time.seconds(5)))
    .process(new TopKWindowFunction());
```

In this example, the results for time window [0, 5) from the first operation will also end up in time window [0, 5) in the subsequent windowed operation. This allows calculating a sum per key and then calculating the top-k elements within the same window in the second operation.

## Useful state size considerations

---

Windows can be defined over long periods of time (such as days, weeks, or months) and therefore accumulate very large state. There are a couple of rules to keep in mind when estimating the storage requirements of your windowing computation:

1. Flink creates one copy of each element per window to which it belongs. Given this, tumbling windows keep one copy of each element (an element belongs to exactly one window unless it is dropped late). In contrast, sliding windows create several of each element, as explained in the Window Assigners section. Hence, a sliding window of size 1 day and slide 1 second might not be a good idea.

2. ReduceFunction, AggregateFunction, and FoldFunction can significantly reduce the storage requirements, as they eagerly aggregate elements and store only one value per window. In contrast, just using a ProcessWindowFunction requires accumulating all elements.

3. Using an Evictor prevents any pre-aggregation, as all the elements of a window have to be passed through the evictor before applying the computation (see Evictors).
