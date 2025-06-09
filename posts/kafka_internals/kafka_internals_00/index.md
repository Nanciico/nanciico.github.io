# The Fundamentals of Apache Kafka Architecture


[Apache Kafka Architecture Deep Dive](https://developer.confluent.io/courses/architecture/get-started/)

---

## Overview of Apache Kafka Architecture

{{< image src="/images/kafka_internals/00_01.jpg" caption="Overview of Apache Kafka Architecture" title="Overview of Apache Kafka Architecture" >}}

The storage layer is designed to store data efficiently and is a distributed system such that if your storage needs grow over time you can easily scale out the system to accommodate the growth.

The compute layer consists of four core components—the producer, consumer, Kafka Streams, and Connect APIs, which allow Kafka to scale applications across distributed systems.

---

## What Is an Event?

An event is a record of something that happened that also provides information about what happened.

Examples of events are customer orders, payments, clicks on a website, or sensor readings.

{{< image src="/images/kafka_internals/00_02.jpg" caption="Kafka Event" title="Kafka Event" >}}

In a Kafka-based architecture, an event record consists of a timestamp, a key, a value, and optional headers.

---

## Kafka Topics

{{< image src="/images/kafka_internals/00_03.jpg" caption="Kafka Topics" title="Kafka Topics" >}}

Topics are append-only, immutable logs of events. Typically, events of the same type, or events that are in some way related, would go into the same topic.

### Kafka Topic Partitions

In order to distribute the storage and processing of events in a topic, Kafka uses the concept of partitions. A topic is made up of one or more partitions and these partitions can reside on different nodes in the Kafka cluster.

{{< image src="/images/kafka_internals/00_04.jpg" caption="Kafka Topic Partitions" title="Kafka Topic Partitions" >}}

Within the partition, each event is given a unique identifier called an offset. The offset for a given partition will continue to grow as events are added, and offsets are never reused.

---

## Scala 快速入门

[Scala for Java Programmers](https://docs.scala-lang.org/tutorials/scala-for-java-programmers.html)

---

