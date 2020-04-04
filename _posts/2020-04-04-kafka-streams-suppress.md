---
layout: post
title: Kafka Streams suppress feature
categories: what-i-learned
tags: java kafka streams
author: karsten
---
[Kafka Streams](https://kafka.apache.org/documentation/streams/) is a Java streaming framework, that tightly integrates with [Apacha Kafka](https://kafka.apache.org/). In contrast to [Apache Spark](https://spark.apache.org/) of [Apache Flink](https://flink.apache.org/) it does not require a separate cluster runtime. Instead it consist of a Java library to be fully integrated into any Java application. In that way it can augment the capabilities of that application by adding stream processing of Kafka topics. For a full feature description of Kafka Streams check out the ocumentation on the [Apache project page](https://kafka.apache.org/documentation/streams/) or from [Confluent](https://docs.confluent.io/current/streams/index.html).

Kafka Streams provides [its own DSL](https://kafka.apache.org/24/documentation/streams/developer-guide/dsl-api.html) to describe the stream processing topology. It is build upon two primitives: KStream and KTable. KStream is an abstraction of one or more Kafka topics. It supports message processing one message at a time. Aggregating multiple messages yields a KTable. KTables can be transformed back to a KStream consisting of the changelog of the KTable. The Kafka Streams documentation from Confluent contains a nice description of this [stream-table-duality](https://docs.confluent.io/current/streams/concepts.html#duality-of-streams-and-tables).

Generating the changlog out of a KTable will generate at least one message for each aggregated message in the resulting KStream. Therefore, downstream processors will see as many messages as were contained in the original KStream. For some applications, it would be benefitial to have a downsampling of the message frequency at the cost of accuracy or latency. This is, what the suppress feature of Kafka Streams provides. It allows to _suppress_ messages on the changelog stream according to configurable conditions.

To understand the availbable suppressions, it is necessary to know the aggregations Kafka Streams supports. Messages will always be aggregated by their respective Kafka message key. This aggregation can be unbound or windowed. Kafka Streams supports different kinds of time windows and also a notion of session windows. A detailed description of windowed aggregations is contained in the official documentation.

Building on these aggregations, there are two modes of suppression, that are supported by Kafka Streams. All KTables support a time window suppression. That means changelog messages are only forwarded downstream after a certain duration has passed after the first change to the KTable was made. This requires a buffer, whose size can be configured in bytes or number of messages. If the buffer is exceeded before the duration is reached, the changelog will be emitted regardless.

```java
public void suppressWithBoundedBuffer(StreamsBuilder builder) {
    builder.stream("events")
        .groupByKey() // group events by key
        .count()      // use event count as easy aggregation
        .suppress(
            Suppressed.untilTimeLimit(
                Duration.ofSeconds(2),       // emit count every 2 seconds
                BufferConfig.maxRecords(100) // collect at most 100 keys
            )
        );
}
```

The other mode is only supported for windowed aggregations. In this case all changelog messages can be suppressed until the window is closed. For time windowed aggregations, it is required to configure a grace period, to allow for late messages. It can be set to 0, but it needs to be explicitly set, otherwise no message will be forwarded. Since the closing the window can occur well after any buffer size is exceeded, no messages will be forwarded if this limit is reached. Instead Kafka Streams will try to shutdown the application.

```java
public void suppressWithUnboundedBuffer(StreamsBuilder builder) {
    builder.stream("events")
        .groupByKey() // group events by key
        .windowedBy(
            TimeWindows
                .of(Duration.ofSeconds(2))    // time windows of 2 seconds
                .grace(Duration.ofSeconds(1)) // allow 1 second lateness
        )
        .count() // use event count as easy aggregation
        .suppress(
            Suppressed.untilWindowCloses(
                BufferConfig.unbounded() // do not restrict buffer
            )
        );
}
```

There is a nice blog post by John Roesler on [Kafka Streams take on Watermarks and Triggers](https://www.confluent.io/blog/kafka-streams-take-on-watermarks-and-triggers/), that describes the suppress feature in great detail.