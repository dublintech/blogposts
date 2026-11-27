---
title: "Designing Data-Intensive Applications (2nd Edition), Part 5: Unbundling the Database, and Doing the Right Thing"
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

In [Part 4](../ddia-2e-part-4-batch-and-stream-processing/) I covered batch and stream processing. The book's last two chapters step back from individual techniques. The first asks what happens if you take everything a database does for you and rebuild it across many systems, tied together with logs. The second is about engineering ethics. Both are more philosophical than the rest of the book, so I'll keep this post short and focus on the ideas I expect to actually use.

## Unbundling the database

A single database quietly does a lot for you. It stores data, keeps secondary indexes in sync, maintains materialised views, and keeps a replication log so followers apply changes in the same order as the leader. All of that is bundled together and kept consistent for you.

Real systems rarely live in one database, though. A way that helped me picture this was an online pet store. It might have Postgres for orders, Elasticsearch for search, Redis as a cache and a warehouse for analytics. Together they act as one big database, but nothing is keeping them consistent with each other. The book's argument is that we should rebuild what the database does internally, but across systems: an ordered log as the source of truth, with each system deriving its state from that log. You'll sometimes hear this called "turning the database inside out".

## The dual-writes problem

The clearest reason to do this is a bug that almost every team writes at some point. The application updates the database and *then* updates the search index:

```java
db.update(pet);
searchIndex.update(pet);
```

It looks harmless. The book illustrates the problem with two clients writing at the same moment. To make it concrete for myself, I put it in pet store terms: two staff members change the price of the same pet at the same time.

| Step | Database | Search index |
|---|---|---|
| 1 | Alice sets price to £100 | |
| 2 | Bob sets price to £120 | |
| 3 | | Bob's update arrives: £120 |
| 4 | | Alice's update arrives: £100 |

The database says £120 and the search index says £100, and they'll stay that way. Nothing failed, no error was logged, and nothing will ever correct it. The two writes simply arrived in a different order at each system. On top of that, if the second write fails because the search index is down for a moment, the systems disagree again.

The fix is to stop writing to both. Write once, to the database, and let change data capture turn its changes into a log. The search index, cache and warehouse all consume that log **in the same order**, so they all apply Alice's change and then Bob's, and they all end up agreeing with the database.

<p align="center">
  <img src="../../../images/ddia-dual-writes.svg" alt="Left: Alice and Bob each write a price to both the database and the search index; the writes arrive in a different order, so the database ends at £120 and the search index at £100. Right: Alice and Bob write only to the database, whose ordered changelog of £100 then £120 is consumed by the search index, cache and warehouse, all ending at £120." />
</p>

This is the same idea as a database's internal replication log from [Part 2](../ddia-2e-part-2-replication-sharding-transactions/), and the stream–table join from [Part 4](../ddia-2e-part-4-batch-and-stream-processing/), now applied across a whole architecture.

## Integrity matters more than timeliness

The idea I found most useful in this part of the book is splitting "consistency" into two separate concerns:

- **Timeliness** means users see up-to-date data. When it fails, the effect is temporary: wait a moment and the replica catches up.
- **Integrity** means data isn't lost, corrupted or contradictory. When it fails, the effect is permanent: no amount of waiting will bring back a lost payment.

We usually treat these as one thing and pay for both, with distributed transactions and linearizability everywhere, which [Part 3](../ddia-2e-part-3-distributed-trouble-and-consensus/) showed are expensive. But many businesses already accept weak timeliness as long as integrity holds. Your bank shows "pending" transactions that settle later. Airlines overbook flights and deal with it at the gate. Going back to my pet store: if two customers manage to buy the last puppy at the same moment, the store can apologise to one of them and offer an alternative. That's annoying, but no money or data has been lost.

So how do you guarantee integrity without expensive coordination? The book's answer is the **end-to-end argument**. Imagine a customer double-clicks "Pay". TCP's deduplication won't help, and neither will database transactions, because these are two separate, valid requests. The only reliable fix is for the client to generate a **request ID** once and send it through every step, so each system can recognise the duplicate and process the payment only once. In the database, that can be as simple as a uniqueness constraint on the request ID.

<p align="center">
  <img src="../../../images/ddia-request-id.svg" alt="A customer double-clicks Pay, sending two requests that carry the same request ID, 7f3a. The payment service has a unique constraint on request_id, so the first request charges £100 and the second is recognised as a duplicate and skipped." />
</p>

Combine that with logs that are processed in order, deterministically, and you get strong integrity guarantees without making every component wait for every other.

## Doing the right thing

The final chapter is about ethics, and I'll keep this brief. Two points stood out to me.

First, predictive systems used for loans, hiring or insurance learn from past decisions, so they learn the biases in those decisions and then apply them at scale, with an air of objectivity. The people affected often have no way to find out why the system said no, let alone appeal.

Second, the book suggests a thought experiment: replace the word "data" with "surveillance" and see whether the sentence still sounds acceptable. "We're a surveillance-driven organisation" lands rather differently. The practical takeaway for engineers is to collect and keep only what you need. Stored personal data is a liability as well as an asset.

## Verdict

Overall this book is quite simply a 10/10, and one of the best technical books I have ever read. It's for anyone interested in distributed systems that handle a lot of data. Its greatest strength is its rigour and technical detail. It's not the type of book you read once and give back to the library. You put it on your shelf and pick it up again whenever you're trying to ground some complex, abstract idea or a hard problem in a distributed system.
