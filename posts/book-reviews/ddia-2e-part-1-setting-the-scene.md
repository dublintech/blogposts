---
title: "Designing Data-Intensive Applications (2nd Edition), Part 1: Setting the Scene"
date: 2026-11-22
author: "Alex Staveley"
tags:
  - book-review
  - ddia
  - data
  - architecture
  - distributed-systems
---

<p align="center">
  <img src="../../../images/ddia-2e-cover.png" alt="Cover of Designing Data-Intensive Applications, 2nd Edition, by Martin Kleppmann and Chris Riccomini" width="300" />
</p>

I've just finished 600 pages of the second edition of *Designing Data-Intensive Applications*. It's an excellent book, with lots of super material. One review would run too long.  So I'm splitting my thoughts into multiple posts. This first post covers the opening chapters, where the book lays its foundations: how to think about performance and reliability, how to choose a data model, how storage engines work underneath, and how to change your data over time without breaking everything.

If you've read the first edition, a lot of this will feel familiar, but it's been updated to fit how we build systems now. Rather than summarize each chapter, I've pulled out the ideas that stuck with me, the ones I expect to use in design discussions.

<p align="center">
  <img src="../../../images/martin-kleppmann.jpg" alt="Martin Kleppmann holding a copy of the first edition of Designing Data-Intensive Applications" width="280" /><br />
  <em>Martin Kleppmann in 2017 with the first edition. Photo: Martin Kleppmann, via <a href="https://commons.wikimedia.org/wiki/File:Martin_kleppmann_in_2017.jpg">Wikimedia Commons</a>, <a href="https://creativecommons.org/licenses/by/3.0/">CC BY 3.0</a>.</em>
</p>

## Averages hide the users who matter

Early on, the book argues that you should look at response time percentiles, not averages. What I liked was the explanation of why the high percentiles matter, using Amazon as the example.

Amazon describes its response-time requirements at the 99.9th percentile, which means caring about the slowest one request in a thousand. It might sound excessive caring about the 1 in a 1,000, until you ask who those slow requests belong to. Often it's the customers with the most data on their accounts, and they have the most data because they've bought the most. So the slowest requests tend to go to your most valuable customers. If you optimize for the median, you're optimizing for the wrong people.

A related idea is **tail latency amplification**. A single user-facing request often fans out to several backend calls in parallel, and the user waits for all of them. If even one of those calls is slow, the whole request is slow. 

## Faults aren't failures

This is a small distinction in wording that helps a lot. A **fault** is when one part of the system stops working: a disk dies, a node crashes, a network link drops. A **failure** is when the system as a whole stops providing its service to users.

The point is that you can't prevent faults; at any real scale they happen all the time. What you can do is design so that faults don't become failures. When someone says "we need to make this more reliable," it's worth asking which of the two they mean.

## A good abstraction is worth a lot

A good abstraction is one of the best tools we have for managing complexity. A well-designed abstraction hides a large amount of messy detail behind a clean interface. SQL is the obvious example: you say what data you want, and the query planner works out how to get it. 

## Schema on read vs schema on write

One of the clearest framings in the data models chapter is about where the schema is enforced.

Relational databases are **schema on write**. The database checks every write against the schema and rejects anything that doesn't fit.

Document databases are often called "schemaless," but that's misleading. There is still a schema; it just lives in the code that reads the data. That's **schema on read**. The application assumes the data has a certain shape and has to deal with the cases where it doesn't.

This is a bit like static versus dynamic typing in programming languages. Neither is simply better. Schema on read is useful when your data is truly varied, or when its structure is decided by something you don't control. Schema on write gives you guarantees and, as a side effect, documentation.

## Locality cuts both ways

Document databases get good data locality. A document and everything nested inside it is stored together as one contiguous encoded blob (JSON or a binary variant such as BSON). If you usually need the whole thing at once, say a user profile with all its positions and education history, you get it in a single read with no joins.

The flip side is that the database usually loads the whole document even when you need only a small part of it. Writes work the same way: changing one field typically means rewriting the entire document. If your documents are large and you mostly touch small pieces of them, locality starts to work against you. A relational database gives you finer-grained control over what you read and write.

So the advice is to keep documents fairly small and avoid writes that keep growing them.

## Let the relationships decide

The kind of relationships in your data is an important factor in choosing a data model.

- If your data is mostly **one-to-many**, a tree where each thing contains other things, the document model fits well.
- If lots of records refer to shared entities (**many-to-one**), you want joins, which is the relational model's strength and historically a weakness of document databases.
- Relational databases handle **many-to-many** fine with join tables. But when everything connects to everything, and your queries follow paths of unknown length, a graph database lets you write the query the way you think about it.


The book's graph example made this click for me. Think about representing where people were born and where they live. Places nest inside other places, but differently in every country. In the US you might have city, then state, then country. In France it's city, then département, then région, then country. In the UK it's different again.

Trying to fit that into fixed columns like `city`, `state` and `country` gets awkward quickly. In a graph you don't need to. Every place is a vertex, each vertex has a type, and they're linked by a single relationship, `WITHIN`. People link to places with `BORN_IN` or `LIVES_IN`. The graph doesn't need to know what a département is; it only knows that one place is within another.

<p align="center">
  <img src="../../../images/ddia-graph-people-places.svg" alt="Graph of places linked by WITHIN edges at different depths: Idaho within United States within North America; Beaune within Côte-d'Or within Bourgogne-Franche-Comté within France within Europe; London within United Kingdom within Europe. Lucy is BORN_IN Idaho and LIVES_IN London; Alain is BORN_IN Beaune and LIVES_IN London." />
</p>

Querying is where this pays off. To find everyone born in the US who now lives in Europe, a graph query language like Cypher lets you write:

```cypher
MATCH
  (person)-[:BORN_IN]->()-[:WITHIN*0..]->(:Location {name: 'United States'}),
  (person)-[:LIVES_IN]->()-[:WITHIN*0..]->(:Location {name: 'Europe'})
RETURN person.name
```

The `*0..` means "follow `WITHIN` edges as many times as needed." It doesn't matter how many levels lie between a city and its continent. The query follows the shape of the question, not the shape of the schema. The relational equivalent is possible, but it needs recursive common table expressions and gets complicated quickly. The book's SQL version is several times longer and much harder to read.

## DataFrames: reshaping data

I was glad to see DataFrames covered in the second edition, since anyone working near data science uses them heavily. My first instinct was to think of them as a table you manipulate in code, which is true but doesn't explain why you'd choose one over SQL.

Where they really help is reshaping data. SQL needs to know its output columns when you write the query. A DataFrame doesn't, so operations where the data determines the columns become easy: a dynamic pivot, where the values in a column become new columns, or a transpose, where rows and columns swap.

This makes DataFrames a natural bridge between relational data and machine learning. ML libraries want numeric matrices, not tables of records. Converting a `country` column into one-hot encoded `country_UK`, `country_US` and `country_FR` columns is a single call on a DataFrame and awkward in SQL.

## Two ways to store data

The storage chapter compares two families of storage engines. I think every architect / developer should understand both, even if you never build one.

**Log-structured storage**, such as SSTables and the LSM-trees built from those files, never overwrites data on disk. New writes go into a sorted in-memory buffer called the memtable. When it fills up, it's flushed to disk as an immutable sorted file, an SSTable. In the background, compaction merges these files and discards outdated values. Every write is a fast sequential append, so writes are cheap. The cost shows up on reads: a key might be in any of several files, so a lookup may need to check the memtable and a few SSTables. Bloom filters and compaction keep that under control. It is called log structured because each SSTable is immutable and subsequent SSTables are just appended like a log.

**Update-in-place storage**, the B-tree, takes the opposite approach. Data lives in one sorted tree of fixed-size pages, and a write changes the right page directly on disk. A read is one predictable walk from the root to a leaf, usually three or four levels even for very large datasets, so reads are fast and consistent. The cost shows up on writes: each one is a random page rewrite, plus a write-ahead log entry for crash safety, and sometimes a page split.

<p align="center">
  <img src="../../../images/ddia-lsm-vs-btree.svg" alt="Side-by-side comparison. LSM-tree: writes go to an in-memory sorted memtable, which is flushed to immutable SSTables on disk, and compaction merges older SSTables. B-tree: a tree of fixed-size pages where reads and writes walk from root to leaf, with every write first recorded in a write-ahead log." />
</p>

Put simply, LSM-trees make writing cheap and pay later when reading and compacting, while B-trees pay up front on every write so reads stay cheap. The book is careful to call this a rule of thumb, and I agree. Real performance depends on your workload, so benchmark your own access patterns.

## Data outlives code

The encoding chapter contains what I think is the most practically useful idea in this part of the book: your data lives much longer than the code that wrote it.

During a rolling deployment, or while mobile clients are slowly upgrading, old and new versions of your code run side by side and read each other's data. So you need compatibility in both directions:

- **Backward compatibility**: new code can read data written by old code.
- **Forward compatibility**: old code can read data written by new code.

Forward compatibility is the harder of the two, because old code has to tolerate fields it has never seen.  For example, old code reads a record, doesn't recognize a new field, writes the record back, and silently deletes that field.

Nothing crashes and no alert fires; the data just disappears.

## Why schemas help

This leads to the case for schema-based formats such as Protocol Buffers and Avro, as opposed to plain JSON.

Picture a JSON message on a Kafka topic with several teams consuming it. The producing team renames `userName` to `username` as a harmless-looking tidy-up. Nothing checks the change, nothing fails at build time, and it ships. Every consumer now looks for `userName`, finds nothing, and either crashes or quietly stores blank names. You find out from bug reports.

With Protocol Buffers, what goes over the wire is a tag number, not the field name:

```protobuf
message UserSignedUp {
  string user_name = 1;
  string email     = 2;
}
```

Renaming `user_name` changes nothing on the wire, because tag 1 is still tag 1. Adding a field means adding a new tag, and old consumers skip tags they don't recognize. The one rule is that you never reuse a tag number.

I'd add one caveat that came out of my own thinking rather than the book. Much of the benefit comes from having a schema at all, ideally kept in a schema registry that checks compatibility before you deploy. You can get some of that with JSON Schema. Binary formats go further, because the schema is required to decode the data and their evolution rules are simple and well-defined. Avro, for example, resolves the writer's schema against the reader's, so each consumer gets data in the shape it expects, with defaults filled in for missing fields.

## Wrapping up

These opening chapters do what foundations should: they give you a vocabulary and a set of trade-offs that the rest of the book builds on. The theme I noticed throughout is that the book almost never says "use X." It says "X makes this cheap and that expensive; here's how to tell which matters to you."

In the next post I'll move into distributed data: replication, sharding, transactions, and what happens when the network and clocks stop behaving. That's where the book gets most interesting, and most humbling.
