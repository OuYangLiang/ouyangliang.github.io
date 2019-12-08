---
layout: post
title:  "Flink Event Time"
date:   2019-12-07 08:00:00 +0800
categories: streaming
keywords: streaming
description: Flink Event Time
commentId: 2019-12-07
---
Flink supports different notions of time in streaming programs.

* **Processing time:** Processing time refers to the system time of the machine that is executing the respective operation.

    When a streaming program runs on processing time, all time-based operations (like time windows) will use the system clock of the machines that run the respective operator. An hourly processing time window will include all records that arrived at a specific operator between the times when the system clock indicated the full hour. For example, if an application begins running at 9:15am, the first hourly processing time window will include events processed between 9:15am and 10:00am, the next window will include events processed between 10:00am and 11:00am, and so on.

    Processing time is the simplest notion of time and requires no coordination between streams and machines. It provides the best performance and the lowest latency. However, in distributed and asynchronous environments processing time does not provide determinism, because it is susceptible to the speed at which records arrive in the system (for example from the message queue), to the speed at which the records flow between operators inside the system, and to outages (scheduled, or otherwise).

* **Event time:** Event time is the time that each individual event occurred on its producing device. This time is typically embedded within the records before they enter Flink, and that event timestamp can be extracted from each record. In event time, the progress of time depends on the data, not on any wall clocks. Event time programs must specify how to generate Event Time Watermarks, which is the mechanism that signals progress in event time.

    In a perfect world, event time processing would yield completely consistent and deterministic results, regardless of when events arrive, or their ordering. However, unless the events are known to arrive in-order (by timestamp), event time processing incurs some latency while waiting for out-of-order events. As it is only possible to wait for a finite period of time, this places a limit on how deterministic event time applications can be.

    Assuming all of the data has arrived, event time operations will behave as expected, and produce correct and consistent results even when working with out-of-order or late events, or when reprocessing historic data. For example, an hourly event time window will contain all records that carry an event timestamp that falls into that hour, regardless of the order in which they arrive, or when they are processed. (See the section on late events for more information.)

    Note that sometimes when event time programs are processing live data in real-time, they will use some processing time operations in order to guarantee that they are progressing in a timely fashion.

* **Ingestion time:** Ingestion time is the time that events enter Flink. At the source operator each record gets the source’s current time as a timestamp, and time-based operations (like time windows) refer to that timestamp.

    Ingestion time sits conceptually in between event time and processing time. Compared to processing time, it is slightly more expensive, but gives more predictable results. Because ingestion time uses stable timestamps (assigned once at the source), different window operations over the records will refer to the same timestamp, whereas in processing time each window operator may assign the record to a different window (based on the local system clock and any transport delay).

    Compared to event time, ingestion time programs cannot handle any out-of-order events or late data, but the programs don’t have to specify how to generate watermarks.

    Internally, ingestion time is treated much like event time, but with automatic timestamp assignment and automatic watermark generation.

<center><img src="{{site.baseurl}}/pic/flink-eventtime/1.svg" width="80%"/></center>

## Setting a Time Characteristic

---

The first part of a Flink DataStream program usually sets the base time characteristic. That setting defines how data stream sources behave (for example, whether they will assign timestamps), and what notion of time should be used by window operations like `KeyedStream.timeWindow(Time.seconds(30))`.

The following example shows a Flink program that aggregates events in hourly time windows. The behavior of the windows adapts with the time characteristic.

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime);

// alternatively:
// env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime);
// env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

DataStream<MyEvent> stream = env.addSource(new FlinkKafkaConsumer09<MyEvent>(topic, schema, props));

stream
    .keyBy( (event) -> event.getUser() )
    .timeWindow(Time.hours(1))
    .reduce( (a, b) -> a.add(b) )
    .addSink(...);
```

Note that in order to run this example in event time, the program needs to either use sources that directly define event time for the data and emit watermarks themselves, or the program must inject a Timestamp Assigner & Watermark Generator after the sources. Those functions describe how to access the event timestamps, and what degree of out-of-orderness the event stream exhibits.

## Event Time and Watermarks

---

A stream processor that supports event time needs a way to measure the progress of event time. For example, a window operator that builds hourly windows needs to be notified when event time has passed beyond the end of an hour, so that the operator can close the window in progress.

Event time can progress independently of processing time (measured by wall clocks). For example, in one program the current event time of an operator may trail slightly behind the processing time (accounting for a delay in receiving the events), while both proceed at the same speed. On the other hand, another streaming program might progress through weeks of event time with only a few seconds of processing, by fast-forwarding through some historic data already buffered in a Kafka topic (or another message queue).

The mechanism in Flink to measure progress in event time is **watermarks**. Watermarks flow as part of the data stream and carry a timestamp t. A Watermark(t) declares that event time has reached time t in that stream, meaning that there should be no more elements from the stream with a timestamp t’ <= t (i.e. events with timestamps older or equal to the watermark).

The figure below shows a stream of events with (logical) timestamps, and watermarks flowing inline. In this example the events are in order (with respect to their timestamps), meaning that the watermarks are simply periodic markers in the stream.

<center><img src="{{site.baseurl}}/pic/flink-eventtime/2.svg" width="80%"/></center>

Watermarks are crucial for out-of-order streams, as illustrated below, where the events are not ordered by their timestamps. In general a watermark is a declaration that by that point in the stream, all events up to a certain timestamp should have arrived. Once a watermark reaches an operator, the operator can advance its internal event time clock to the value of the watermark.

<center><img src="{{site.baseurl}}/pic/flink-eventtime/3.svg" width="80%"/></center>

Note that event time is inherited by a freshly created stream element (or elements) from either the event that produced them or from watermark that triggered creation of those elements.

## Watermarks in Parallel Streams

---

Watermarks are generated at, or directly after, source functions. Each parallel subtask of a source function usually generates its watermarks independently. These watermarks define the event time at that particular parallel source.

As the watermarks flow through the streaming program, they advance the event time at the operators where they arrive. Whenever an operator advances its event time, it generates a new watermark downstream for its successor operators.

Some operators consume multiple input streams; a union, for example, or operators following a `keyBy(…)` or `partition(…)` function. Such an operator’s current event time is the minimum of its input streams’ event times. As its input streams update their event times, so does the operator.

The figure below shows an example of events and watermarks flowing through parallel streams, and operators tracking event time.

<center><img src="{{site.baseurl}}/pic/flink-eventtime/4.svg" width="80%"/></center>

## Late Elements

---

It is possible that certain elements will violate the watermark condition, meaning that even after the Watermark(t) has occurred, more elements with timestamp t’ <= t will occur. In fact, in many real world setups, certain elements can be arbitrarily delayed, making it impossible to specify a time by which all elements of a certain event timestamp will have occurred. Furthermore, even if the lateness can be bounded, delaying the watermarks by too much is often not desirable, because it causes too much delay in the evaluation of event time windows.

For this reason, streaming programs may explicitly expect some late elements. Late elements are elements that arrive after the system’s event time clock (as signaled by the watermarks) has already passed the time of the late element’s timestamp.

By default, late elements are **dropped** when the watermark is past the end of the window. However, Flink allows to specify a maximum allowed lateness for window operators. Allowed lateness specifies by how much time elements can be late before they are dropped, and its default value is 0. Elements that arrive after the watermark has passed the end of the window but before it passes the end of the window plus the allowed lateness, are still added to the window. Depending on the trigger used, a late but not dropped element may cause the window to fire again. This is the case for the EventTimeTrigger.

In order to make this work, Flink keeps the state of windows until their allowed lateness expires. Once this happens, Flink removes the window and deletes its state.

Default By default, the allowed lateness is set to 0. That is, elements that arrive behind the watermark will be dropped. You can specify an allowed lateness like this:

```java
DataStream<T> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .allowedLateness(<time>)
    .<windowed transformation>(<window function>);
```

## Idling sources

---

Currently, with pure event time watermarks generators, watermarks can not progress if there are no elements to be processed. That means in case of gap in the incoming data, event time will not progress and for example the window operator will not be triggered and thus existing windows will not be able to produce any output data.

To circumvent this one can use periodic watermark assigners that don’t only assign based on element timestamps. An example solution could be an assigner that switches to using current processing time as the time basis after not observing new events for a while.

Sources can be marked as idle using `SourceFunction.SourceContext#markAsTemporarilyIdle`. For details please refer to the Javadoc of this method as well as StreamStatus.

## How operators are processing watermarks

---

As a general rule, operators are required to completely process a given watermark before forwarding it downstream. For example, WindowOperator will first evaluate which windows should be fired, and only after producing all of the output triggered by the watermark will the watermark itself be sent downstream. In other words, all elements produced due to occurrence of a watermark will be emitted before the watermark.

The same rule applies to TwoInputStreamOperator. However, in this case the current watermark of the operator is defined as the minimum of both of its inputs.

## Assigning Timestamps

---

In order to work with event time, Flink needs to know the events’ timestamps, meaning each element in the stream needs to have its event timestamp assigned. This is usually done by accessing/extracting the timestamp from some field in the element.

Timestamp assignment goes hand-in-hand with generating watermarks, which tell the system about progress in event time. There are two ways to assign timestamps and generate watermarks:

1. Directly in the data stream source
1. Via a timestamp assigner / watermark generator: in Flink, timestamp assigners also define the watermarks to be emitted

### Source Functions with Timestamps and Watermarks

---

Stream sources can directly assign timestamps to the elements they produce, and they can also emit watermarks. When this is done, no timestamp assigner is needed. Note that if a timestamp assigner is used, any timestamps and watermarks provided by the source will be overwritten.

To assign a timestamp to an element in the source directly, the source must use the `collectWithTimestamp(...)` method on the SourceContext. To generate watermarks, the source must call the `emitWatermark(Watermark)` function.

Below is a simple example of a (non-checkpointed) source that assigns timestamps and generates watermarks:

```java
@Override
public void run(SourceContext<MyType> ctx) throws Exception {
	while (/* condition */) {
		MyType next = getNext();
		ctx.collectWithTimestamp(next, next.getEventTimestamp());

		if (next.hasWatermarkTime()) {
			ctx.emitWatermark(new Watermark(next.getWatermarkTime()));
		}
	}
}
```

### Timestamp Assigners / Watermark Generators

---

Timestamp assigners take a stream and produce a new stream with timestamped elements and watermarks. If the original stream had timestamps and/or watermarks already, the timestamp assigner overwrites them.

Timestamp assigners are usually specified immediately after the data source, but it is not strictly required to do so. A common pattern, for example, is to parse (MapFunction) and filter (FilterFunction) before the timestamp assigner. In any case, the timestamp assigner needs to be specified before the first operation on event time (such as the first window operation). As a special case, when using Kafka as the source of a streaming job, Flink allows the specification of a timestamp assigner / watermark emitter inside the source (or consumer) itself. More information on how to do so can be found in the Kafka Connector documentation.

#### With Periodic Watermarks

----

AssignerWithPeriodicWatermarks assigns timestamps and generates watermarks periodically (possibly depending on the stream elements, or purely based on processing time).

The interval (every n milliseconds) in which the watermark will be generated is defined via `ExecutionConfig.setAutoWatermarkInterval(...)`. The assigner’s `getCurrentWatermark()` method will be called each time, and a new watermark will be emitted if the returned watermark is non-null and larger than the previous watermark.

Here we show two simple examples of timestamp assigners that use periodic watermark generation.

```java
/**
 * This generator generates watermarks assuming that elements arrive out of order,
 * but only to a certain degree. The latest elements for a certain timestamp t will arrive
 * at most n milliseconds after the earliest elements for timestamp t.
 */
public class BoundedOutOfOrdernessGenerator implements AssignerWithPeriodicWatermarks<MyEvent> {

    private final long maxOutOfOrderness = 3500; // 3.5 seconds

    private long currentMaxTimestamp;

    @Override
    public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
        long timestamp = element.getCreationTime();
        currentMaxTimestamp = Math.max(timestamp, currentMaxTimestamp);
        return timestamp;
    }

    @Override
    public Watermark getCurrentWatermark() {
        // return the watermark as current highest timestamp minus the out-of-orderness bound
        return new Watermark(currentMaxTimestamp - maxOutOfOrderness);
    }
}

/**
 * This generator generates watermarks that are lagging behind processing time by a fixed amount.
 * It assumes that elements arrive in Flink after a bounded delay.
 */
public class TimeLagWatermarkGenerator implements AssignerWithPeriodicWatermarks<MyEvent> {

	private final long maxTimeLag = 5000; // 5 seconds

	@Override
	public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
		return element.getCreationTime();
	}

	@Override
	public Watermark getCurrentWatermark() {
		// return the watermark as current time minus the maximum time lag
		return new Watermark(System.currentTimeMillis() - maxTimeLag);
	}
}
```

<br/>

#### With Punctuated Watermarks

---

To generate watermarks whenever a certain event indicates that a new watermark might be generated, use AssignerWithPunctuatedWatermarks. For this class Flink will first call the `extractTimestamp(...)` method to assign the element a timestamp, and then immediately call the `checkAndGetNextWatermark(...)` method on that element.

The `checkAndGetNextWatermark(...)` method is passed the timestamp that was assigned in the `extractTimestamp(...)` method, and can decide whether it wants to generate a watermark. Whenever the `checkAndGetNextWatermark(...)` method returns a non-null watermark, and that watermark is larger than the latest previous watermark, that new watermark will be emitted.

```java
public class PunctuatedAssigner implements AssignerWithPunctuatedWatermarks<MyEvent> {

	@Override
	public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
		return element.getCreationTime();
	}

	@Override
	public Watermark checkAndGetNextWatermark(MyEvent lastElement, long extractedTimestamp) {
		return lastElement.hasWatermarkMarker() ? new Watermark(extractedTimestamp) : null;
	}
}
```

Note: It is possible to generate a watermark on every single event. However, because each watermark causes some computation downstream, an excessive number of watermarks degrades performance.

## Timestamps per Kafka Partition

---

When using Apache Kafka as a data source, each Kafka partition may have a simple event time pattern (ascending timestamps or bounded out-of-orderness). However, when consuming streams from Kafka, multiple partitions often get consumed in parallel, interleaving the events from the partitions and destroying the per-partition patterns (this is inherent in how Kafka’s consumer clients work).

In that case, you can use Flink’s Kafka-partition-aware watermark generation. Using that feature, watermarks are generated inside the Kafka consumer, per Kafka partition, and the per-partition watermarks are merged in the same way as watermarks are merged on stream shuffles.

For example, if event timestamps are strictly ascending per Kafka partition, generating per-partition watermarks with the ascending timestamps watermark generator will result in perfect overall watermarks.

The illustrations below show how to use the per-Kafka-partition watermark generation, and how watermarks propagate through the streaming dataflow in that case.

```java
FlinkKafkaConsumer<MyType> kafkaSource = new FlinkKafkaConsumer<>("myTopic", schema, props);
kafkaSource.assignTimestampsAndWatermarks(new AscendingTimestampExtractor<MyType>() {

    @Override
    public long extractAscendingTimestamp(MyType element) {
        return element.eventTimestamp();
    }
});

DataStream<MyType> stream = env.addSource(kafkaSource);
```

<center><img src="{{site.baseurl}}/pic/flink-eventtime/5.svg" width="80%"/></center>
