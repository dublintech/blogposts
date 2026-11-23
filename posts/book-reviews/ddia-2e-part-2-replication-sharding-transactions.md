---
title: "Designing Data-Intensive Applications (2nd Edition), Part 2: Replication, Sharding and Transactions"
date: 2026-11-23
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

In [Part 1](../ddia-2e-part-1-setting-the-scene/) I covered the foundations. This post moves on to chapters 6, 7 and 8: replication, sharding and transactions. These are topics most of us use every day without thinking too hard about what's going on underneath. Here are the nuggets that made me think a little deeper, and broadened my appreciation for just how much work goes into keeping data correct once it lives on more than one machine.

## Replication

### Replication is not a backup

Replication can look like a backup: there's another copy of your data somewhere else. But there's a key difference. A backup is something you might want to *revert to*. A replica faithfully follows the leader, so if someone accidentally deletes a table, the replicas delete it too, within milliseconds. Replication protects you from a machine dying. It doesn't protect you from yourself.

### Single leader is still king

The most common approach is single-leader replication. All writes go to one node, the leader, and it streams changes out to followers. Most relational databases work this way, and so do MongoDB and DynamoDB. It's simple to reason about, and that counts for a lot.

### Three ways to ship the log

The interesting part is *what* the leader actually sends to its followers. The book walks through three approaches, and each one fixes a problem with the one before it.

| Approach | What gets sent | Example | Catch |
|---|---|---|---|
| **Statement-based** | Each SQL statement, verbatim, re-executed on every follower | MySQL before 5.1 | Breaks with nondeterministic functions like `NOW()` or `RAND()`: each follower gets a different answer |
| **Write-ahead log shipping** | The storage engine's low-level log: which bytes changed in which disk blocks | PostgreSQL streaming replication, Oracle | Leader and followers are tied to the same storage format, usually the same version, which makes zero-downtime upgrades hard |
| **Logical (row-based)** | Row-level changes: "insert this row", "update row 42's columns to these values", "delete row 42" | MySQL binlog (row-based), PostgreSQL logical decoding | Fewer catches. It's independent of the storage engine, so mixed versions work |

The last row is the one I find most interesting. Because a logical log is decoupled from the storage engine, *other systems* can read it too. That's the basis of change data capture (CDC): your data warehouse, search index or cache can subscribe to the same stream of changes the followers get.

### Replication lag and what users see

With asynchronous followers, a follower can be a little behind the leader. Usually it's milliseconds, but under load it can be seconds or more. The book describes three guarantees that make lag less painful.

**Read-your-writes consistency.** If I update my profile and then reload the page, I should see my change. If my read happens to hit a follower that hasn't caught up yet, it looks like my update was lost.

<p align="center">
  <img src="../../../images/ddia-read-your-writes.svg" alt="Two sequence diagrams. Left: a user updates their bio on the leader, then reads from a lagging follower and gets the old bio. Right: the user's read after a write is routed to the leader and returns the new bio." />
</p>

The fix is to route a user's reads to the leader for a while after they write. Not every database does this for you, so it often ends up in the application layer: something has to remember that this user just wrote and send their next read to the leader.

**Monotonic reads** are about time: one reader's successive reads never go backwards. Without it, you can refresh a page and see an older state than you saw a second ago, because the second read went to a replica further behind.

**Consistent prefix reads** are about order: what you see respects the order in which related writes happened. Without it, you can see the answer to a question before the question itself.

### When multi-leader makes sense

Multi-leader setups usually don't make sense within a single region; the conflict handling costs more than it buys you. Across regions, though, a case can be made: users write to a nearby leader, and you can survive losing a whole region.

The other place multi-leader fits is **offline-first** applications, which need to keep working when disconnected from the internet. Every device is effectively a leader, syncing when it reconnects. **Local-first** goes one step further: the application should keep working even if the online service behind it shuts down for good. Git is a good example. Your repository is complete on your own machine; GitHub is a convenience, not a dependency.

### Merging conflicting edits

If two leaders accept writes to the same data, you need a way to merge them. The book compares two families of algorithms, both used for collaborative editing.

- **Operational transformation (OT)** relies on a central server. When two operations race, the server transforms each incoming operation against the ones it raced with, so that both can be applied. Google Docs works this way.
- **CRDTs** (conflict-free replicated data types) don't need a central server. For text, each character gets a unique, stable ID, and an insert is described relative to those IDs rather than a numeric position like "index 5". Positions shift as people type; IDs don't. That's what lets replicas apply the same operations in any order and still end up with the same document.

### Leaderless and quorums

Leaderless databases like Cassandra let clients write to several replicas directly. With *N* replicas, if every write is confirmed by *W* nodes and every read queries *R* nodes, then as long as **R + W > N** the sets of nodes you wrote to and read from must overlap. At least one node you read from has the latest value.

The book is careful to say this is weaker than it sounds. Concurrent writes, failed writes that partially succeeded, and nodes being replaced can all produce edge cases where you still read a stale value. Quorums make stale reads unlikely, not impossible.

## Sharding

### Key range vs hash

When you have too much data, or too much load, for one machine, you split it across several with sharding. The two common approaches pull in opposite directions:

- **Key-range sharding** keeps keys in sorted order, so range queries are efficient. The risk is hot spots: if your key is a timestamp, today's writes all land on one shard.
- **Hash sharding** spreads keys evenly, which is good for distributing load, but neighbouring keys end up on different shards, so range queries have to ask every shard.

### Compound keys: a bit of both

The compound primary key pattern tries to get both. Picture an `Orders` table in DynamoDB. The **partition key** is `userId`, which is hashed, so users are spread evenly across shards and all of one user's orders live together. The **sort key** is the order date, so within that user's shard the orders are sorted and "this user's orders from last month" is a cheap range scan. Cassandra does the same thing with its partition and clustering keys.

### Secondary indexes: pay on reads or pay on writes

Sharding by primary key is the easy part. Secondary indexes are where it gets interesting, because the index doesn't line up with the shards.

<p align="center">
  <img src="../../../images/ddia-secondary-indexes.svg" alt="Local secondary index: each of three shards indexes only its own products, so a write touches one shard but a query for red products must ask all three. Global secondary index: the index is itself sharded by colour, so a query for red goes to one index shard, but a write must update both the data shard and the index shard." />
</p>

- A **local secondary index** is stored per shard, covering only that shard's rows. A write only has to touch its own shard. A read has to ask every shard and combine the answers, often called scatter/gather.
- A **global secondary index** is sharded by the indexed value itself. A read goes to the one index shard that holds that value. A write, though, may have to update several shards: the data shard plus whichever index shards cover the new values.

Same trade-off as LSM-trees and B-trees in Part 1: you choose where you'd rather pay.

## Transactions

### Read committed isn't enough

Read committed is the default isolation level in many databases, and it gets you a lot: no dirty reads, no dirty writes. But it doesn't prevent **read skew**.

The book's example: Alice has $500 in each of two accounts, and a transaction transfers $100 from one to the other. If Alice reads account 1 *before* the transfer commits and account 2 *after* it, she sees $500 and $400. $100 has apparently vanished. Each individual read saw committed data. They just saw it at different moments.

### Snapshot isolation and MVCC

The fix is **snapshot isolation**. Confusingly, some databases, PostgreSQL and MySQL among them, call it *repeatable read*. Each transaction sees the database as it was at one point in time.

What I liked was how cheap this is. The snapshot is a logical view of the whole database frozen at a point in time, but physically nothing is copied. The database records which transactions were in progress and which had committed when your transaction started. Each row version records which transaction created it and which transaction deleted or overwrote it. When you read, the database shows you only the versions that were committed as of your snapshot. This is **multi-version concurrency control (MVCC)**.

### Write skew: the double booking

Snapshot isolation also stops phantoms from showing up in your reads. If someone inserts a new row that matches your query, you won't see it, because it isn't in your snapshot. That sounds like a pure win, but it's exactly what enables the next problem.

<p align="center">
  <img src="../../../images/ddia-write-skew.svg" alt="Timeline of two concurrent transactions. Alice and Bob each check whether room 101 is free at 2pm, each sees zero bookings in their snapshot, each inserts a booking and commits, leaving the room double-booked." />
</p>

Alice and Bob both try to book the same meeting room at the same time. Each transaction checks whether the room is free, sees zero bookings in its snapshot, inserts a booking and commits. The room is now double-booked. Neither transaction overwrote the other's row, so there's no conflict for snapshot isolation to detect. Each one simply acted on a premise, "the room is free," that the other one had just made false.

This is **write skew**. You can't fix it by locking the rows you read, because the row that causes the problem didn't exist yet when you looked. Your options are serializable isolation, or turning the conflict into something lockable, such as a constraint or a pre-populated table of time slots.

## Wrapping up

The thread through all three chapters is that every guarantee has a price, and the book is honest about who pays it. Read-your-writes costs leader load. Global indexes make writes expensive. Snapshot isolation is cheap, but it leaves write skew open. None of it is magic, and once you see the mechanics you start asking better questions about the databases you already use.

In the next post I'll get into what happens when the network and clocks stop behaving, and how distributed systems manage to agree on anything at all.
