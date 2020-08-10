ksqlDB is an event streaming database specifically built for Apache Kafka. Although ksqlDB aims to give you a higher-level set of primitives than Kafka has, it's inevitable that it can't (and shouldn't) be entirely abstracted over. This section aims to describe the minimal number of Kafka concepts that you need to use ksqlDB effectively. For more information, consult the official [Apache Kafka documentation](https://kafka.apache.org/documentation/).

## Records

The primary unit of data in Kafka is the event. An event models something that happened in the world at a point in time. In Kafka, you work with events using a construct known as a record. Records carry a few different kinds of data within them: key, value, timestamp, topic, partition, offset, and headers.

The key of a record is an arbitrary piece of data that denotes the identity of the event. If the events are clicks on a web page, a suitable key might be the user ID who did the clicking. The value is also an arbitrary piece of data that represents the data of interest. The value of a click event probably contains the page that it happened on, the DOM element that was clicked, and other interesting tidbits of information.

The timestamp denotes when the event happened. There are a few different “kinds” of time that can be tracked. These aren’t discussed here, but they’re useful to learn about nonetheless.

The topic and partition describe which larger collection of events this event belongs to, and the offset describes its exact position within that larger collection (more on that below).

Finally, the headers carry arbitrary, user-supplied metadata about the record.

ksqlDB abstracts over some of these pieces of information so you don’t need to think about them. Others are exposed directly and are an integral part of the programming model. For example, the fundamental unit of data in ksqlDB is the _row_. A row is a helpful abstraction over a Kafka record. Rows have columns of two kinds, key columns and value columns. They also carry pseudocolumns for metadata, like a timestamp.

In general, ksqlDB avoids raising up Kafka-level implementation details that don’t contribute to a high-level programming model.

## Topics

Topics are named collections of records. Their function is to let you hold events of mutual interest together. A series of click records might get stored in a “clicks” topic so that you can access them all in one spot. Topics are append-only. Once you add a record to a topic, you can’t change or delete it individually.

There are no rules for what kinds of records can be placed into topics. They need not conform to the same structure, relate to the same situation, or anything like that. The way you manage publication to topics is entirely a matter of user convention and enforcement.

ksqlDB provides a higher-level abstraction over topics called streams and tables. A stream or table is just a Kafka topic w
ith a registered schema. The schema controls the shapes of records that are allowed to be stored in the topic. This kind of static typing makes it easier to understand what sort of rows are in your topic, and generally helps you make fewer mistakes in your programs that process them.

## Partitions

When a record is placed into a topic, it is placed into a particular partition. A partition is a totally ordered sequence of records. Topics have multiple partitions to make storage and processing more scalable. When you create a topic, you choose how many partitions it has.

When you append a record to a topic, a partitioning strategy chooses which partition it lands in. There are many partitioning strategies. The most common one is to hash the contents of the record’s key against the total number of partitions. This has the effect of placing all records with the same identity into the same partition, which is useful because of the strong ordering guarantees.

The order of the records is tracked by a piece of data known as an offset, which is set when the record is appended. A record with offset 10 happened earlier than a record with offset 20.

Many of the mechanics here are automatically handled by ksqlDB on your behalf. When you create a stream or table, you choose the number of partitions for the underlying topic so that you can have control over its scalability. When you declare a schema, you choose which columns are part of the key and which are part of the value. Beyond that, you need not think about individual partitions or offsets. Here are some examples of that.

When a record is appended, its keys content is automatically rolled up and hashed to consistently send it to the same partition. When records are processed, they follow the correct offset order, even in the presence of failures or faults. When a stream’s key content changes because of how a query wants to process the rows (via `GROUP BY` or `PARTITION BY`), the underlying records keys are recalculated, and the records are sent to a new partition set to perform the computation.

## Producers and consumers

- Clients that publish and subscribe to records.
- Low level, need to build messages, send them places, manage what you send where

- ksqlDB is both. Continuously producing & consuming on your behalf

## Brokers

- Where the data is stored. Servers to the clients. Manage access to data.

- ksqlDB just points to the brokers like any other application

## Serializers

- How data is transported and stored.
- Kafka just speaks bytes
- Separate for key and value
- Great because it doesnt need to keep up with formats
- Roll up at producer, store on broker, unroll at client
- Errors can happen if producer + consumer don't talk correctly.

- ksqlDB up-levels this by supporting a limited number of serializers out of the box
- Maintains metadata about serializers so that you can't mess this up
- Just say what format you want things in, and it just works.

## Schemas

- Kafka is just bytes, but the data has rhyme and reason
- Helpful to make the schema explicit
- Record explicit schemas somewhere
- Confluent Schema Registry helps you do this. Not Apache, but used lots of places.

- ksqlDB speaks with SR behind the scenes to automatically store and retrieve schemas based on your stream/table schemas
- Can also extract schemas automatically for existing topics
- Makes your ksqlDB schemas centrally available to other programs

## Consumer groups

- Allows you to load balance many partitions over many consumers
- Works dynamically. As you gain/lose consumers, load balancing happens automatically

- ksqlDB builds on this. When you deploy a persistent query, it loads balances the work using a consumer group
- Looks at how many source partitions there are and divides them by the number of available servers.
- e.g. 10 partitions, 2 servers = 5 parts per server. Lose a server, and the other one gets all 10. Add 5 servers, and they all have 2.

## Retention

- How long records stay in a topic's partition.
- Affects how long you can "replay" a topic from the earliest information.
- Retention is a nuanced topic, best to read about it in the docs.
- Still important to call out and know what it is.

## Compaction

- Ditch all but the latest record per key
- Used for tables + state maintainence
- Set automatically for you
- Affects things like joins where they are timestamp dependent