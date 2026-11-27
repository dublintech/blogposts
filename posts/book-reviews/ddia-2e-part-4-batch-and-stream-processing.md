---
title: "Designing Data-Intensive Applications (2nd Edition), Part 4: Handling Batches and Streams"
date: 2026-11-27
author: "Alex Staveley"
tags:
  - book-review
  - ddia
  - data
  - architecture
  - distributed-systems
---

<p align="center">
  <img src="../../../images/ddia-2e-cover.png" alt="Cover of Designing Data-Intensive Applications, 2nd Edition, by Martin Kleppmann and Chris Riccomini" width="220" />
</p>

In [Part 3](../ddia-2e-part-3-distributed-trouble-and-consensus/) I covered what goes wrong in distributed systems and how consensus helps. This post covers the batch processing and stream processing chapters. They're about *derived data*: taking data that already exists somewhere and turning it into something else, like a search index, a report, a recommendation model or an enriched event.

The difference between the two comes down to one word. A batch job's input is **bounded**: it has a known, finite size, and the job finishes. A stream's input is **unbounded**: it never ends, so the processing never finishes either.

## Batch processing

### From MapReduce to dataflow engines

Batch processing was shaped by **MapReduce**, published by Google in 2004 and popularised by Hadoop. It was a big step forward: it let you process enormous datasets across clusters of cheap commodity machines.

It has since been superseded by **dataflow engines** such as Spark, for two main reasons.

First, MapReduce writes too much to disk. A real workflow is usually a chain of MapReduce jobs, and each job writes its full output to the distributed filesystem, replicated across several machines, before the next job can start reading it. Dataflow engines treat the whole workflow as one job and keep intermediate data in memory or on local disk.

Second, dataflow engines are far more expressive. Joins, groupings and filters are built-in operators, rather than something you have to squeeze into a series of map and reduce stages.

<p align="center">
  <img src="../../../images/ddia-mapreduce-vs-dataflow.svg" alt="Top: a MapReduce workflow where each of three jobs writes its output to HDFS, replicated three times, before the next job starts. Bottom: a dataflow engine running filter, join, group by and aggregate operators as one job, passing data between them in memory." />
</p>

### Three approaches to fault tolerance

If a machine dies halfway through a job, each system recovers differently:

- **MapReduce** makes everything durable. Each job's output is written to HDFS, so a failed task can simply be rerun from durable input.
- **Spark** tracks the *lineage* of every dataset, meaning how each one was produced from its inputs. If a partition is lost, Spark recomputes just that partition.
- **Flink** takes periodic *checkpoints* of its operators' state and, after a failure, restarts from the most recent one.

There's a subtlety with lineage-based recovery: recomputing only works if the computation is deterministic. If an operator uses random numbers or the current time, the recomputed partition might not match the original.

### The line between warehouse and batch is blurring

Cloud data warehouses like **BigQuery** and **Snowflake** have moved into territory that used to belong to batch processing frameworks. They store data on cheap cloud object storage, separate storage from compute, and handle semi-structured (messier) data like JSON, not just neat relational tables. They also offer programmatic interfaces alongside SQL, such as Snowflake's Snowpark for Python, Java and Scala, and BigQuery DataFrames for Python.

The same thing is happening from the other direction. Spark has added data warehouse features, such as Spark SQL and support for table formats with transactions. So the distinction is becoming less about which tool you use and more about the interface you use it through.

## Where the data lives

### Distributed filesystems: shared nothing

Traditional shared storage, such as NAS and SAN, is a **shared-disk** architecture. It needs specialised hardware and network infrastructure. A distributed filesystem like HDFS is **shared-nothing**: it needs no special hardware, just ordinary computers connected by a normal datacentre network. Files are split into large blocks, which are spread across the machines in the cluster and replicated in case one dies.

### Object storage is not a filesystem

Object stores like S3 have largely replaced HDFS as the place batch jobs read from and write to, but they behave differently from a filesystem. A stored object is immutable. You can't edit part of it in place, so even a small change means rewriting the entire object. This is one reason batch outputs are usually written as new files rather than updated.

### Avro and Parquet

The two formats you'll most often see batch jobs reading are **Avro** and **Parquet**. Avro came out of the Hadoop ecosystem. Parquet was developed at Twitter and Cloudera, inspired by Google's Dremel.

They have a lot in common. Both always have a schema. Parquet always embeds it in the file. Avro can embed it in a file's header, or reference it, for example by an ID in a schema registry when Avro messages are sent through Kafka. Either way, the schema is stated once rather than repeating field names in every record, so the data takes less space. This is the schema idea from [Part 1](../ddia-2e-part-1-setting-the-scene/) paying off again.

The most important difference is the layout. **Avro is row-oriented**: one whole record after another. **Parquet is column-oriented**: all the values for one column are stored together.

<p align="center">
  <img src="../../../images/ddia-row-vs-columnar.svg" alt="A small table with user_id, country, date and amount columns, shown in two on-disk layouts. Avro stores each record's four values one after another. Parquet stores all user_ids, then all countries, then all dates, then all amounts, so similar values sit together and compress well." />
</p>

That's why Avro suits messages and events, where you read and write whole records, and Parquet suits analytics, where a query reads a few columns of a huge number of rows. With Parquet you only read the columns you need, and long runs of similar values compress very well.

### The shuffle

Batch jobs often need to bring together all the records with the same key, for example to group or join them. When the dataset is petabytes in size, that can't happen on one machine. The data has to be redistributed across the cluster so that all the records for a given key end up on the same machine. This is called the **shuffle**.

In MapReduce, the shuffle is also a distributed sort. Each mapper partitions its output by key and sorts it, and each reducer merges the sorted files it receives from all the mappers. Sorting at that scale is one of the most expensive steps in a batch job, which is why dataflow engines try to avoid sorting when an operator doesn't need it.

### DataFrames, but distributed

Spark's DataFrames (and, to a degree, Flink's Table API) both offer DataFrame-style APIs, and it's tempting to think of them as Pandas on a cluster. They're not quite the same:

- A local DataFrame, as in Pandas, has an **index**, so you can look up rows by label. Distributed DataFrames don't have an index like that; they're spread across many machines.
- Pandas operations **run immediately**. Distributed DataFrames are **lazy**: each operation adds a step to a plan, and nothing runs until you ask for a result, such as writing output or collecting rows. That gives the engine the chance to optimise the whole plan before it executes anything.

## Stream processing

### Log-based message brokers

Traditional message brokers delete a message once it's been delivered and acknowledged. **Log-based message brokers**, Kafka being the best-known, take a different approach. They combine the durable storage of a database with the low latency of messaging.

Every message is appended to a log on disk. Consumers keep track of how far they've read using an **offset**, which is just a position in that log. The broker doesn't need to track which messages each consumer has acknowledged. All it needs to remember is one offset per partition for each consumer group. Because the log is retained, a consumer can also rewind and reprocess old messages, which a traditional broker can't do.

### Partitions and consumer groups

A Kafka topic is split into **partitions**, and each partition is its own ordered log. When a consumer group subscribes to a topic, each partition is assigned to exactly one consumer in the group. So each message goes to one consumer in the group, while a different consumer group gets its own full copy of the stream.

<p align="center">
  <img src="../../../images/ddia-kafka-partitions.svg" alt="A producer writes to a topic with three partitions, choosing a partition by hashing the message key. Each partition is an ordered log with numbered offsets and a marker showing the consumer group's current offset. Consumer group A has two consumers: one reads partitions 0 and 1, the other reads partition 2." />
</p>

Ordering follows from this. Order is guaranteed **within a partition**, but there's **no ordering guarantee across partitions**. If some messages have to stay in order, say every event for the same order, you give them the same **partition key**. The producer hashes the key to choose a partition, so those events always go to the same partition, and from there to the same consumer.

### Three kinds of stream join

Stream processors bring in an extra layer of sophistication, and joins are where it shows. The book describes three patterns.

<p align="center">
  <img src="../../../images/ddia-stream-joins.svg" alt="Three panels. Stream-stream join: search events and click events are matched within a time window. Stream-table join: activity events are joined against a local user profiles table that is kept up to date from a changelog, producing enriched events. Table-table join: tweet and follow changelogs maintain two tables, whose join is a materialised view of timelines." />
</p>

- **Stream–stream join**: match related events from two streams that occur within a window of time. The book's example is a search stream and a click stream. Join each search with any clicks on its results within, say, an hour, and you can measure click-through rates.
- **Stream–table join**: one input is a database changelog, captured with change data capture, which the stream processor uses to keep a local copy of a table up to date. Events from the other stream are then enriched by looking them up in that table, for example adding user profile details to each activity event, without querying the database for every event.
- **Table–table join**: both inputs are database changelogs, each keeping a table up to date. The output is a **materialised view** of the join between them, updated continuously as either side changes. The book's example is a Twitter-style home timeline cache, maintained from a stream of tweets and a stream of follow and unfollow events.

The stream–table join ties back to [Part 2](../ddia-2e-part-2-replication-sharding-transactions/), where logical replication logs made change data capture possible. This is where that pays off: the database's changelog becomes just another stream you can build on.

## Wrapping up

The thread through these chapters is that batch and stream processing are two sides of the same idea: derive new data from existing data, with the original kept as the source of truth. Batch works on a bounded snapshot and finishes. Streaming works on an unbounded log and keeps the derived data continuously up to date. The tools have changed a lot since MapReduce, but the core ideas, like partitioning by key, logs, immutable inputs and recomputing derived data, haven't.

In the next post I'll cover the book's remaining chapters and give my overall verdict.
