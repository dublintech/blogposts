---
title: "Designing Data-Intensive Applications (2nd Edition), Part 3: Problems with Distributed Systems, and How Consensus Helps"
date: 2026-11-26
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

In [Part 2](../ddia-2e-part-2-replication-sharding-transactions/) I covered replication, sharding and transactions. This post covers the chapters on what goes wrong once your system spans more than one machine, and on consensus, the main tool we have for getting things right anyway. I'll be honest: some of this was new to me, and it's the part of the book I had to read slowly. It was worth it.

## The network promises nothing

### TCP does a lot, but not everything

TCP does a great deal of work for us. It detects dropped packets and retransmits them, puts packets back in the right order, and detects corrupted packets with checksums. It also does flow control and congestion control, slowing the sender down so it doesn't overwhelm the receiver or the network.

All that reliability comes at the cost of waiting. If a packet is lost, everything behind it waits for the retransmission. For latency-sensitive applications like VoIP or video calls, a late packet is useless; you'd rather drop a few milliseconds of audio than freeze the call. So they use UDP instead, which offers none of these guarantees.

What TCP *can't* tell you is why a response hasn't arrived. The request might have been lost, the remote node might have crashed, or it might just be slow. From the outside these all look the same, so the only tool you have is a timeout, and choosing a timeout is always a guess.

### Why can't networks just have a fixed delay?

The old telephone network does. A circuit-switched network like ISDN runs at a fixed rate of 4,000 frames per second, and when you make a call, it's allocated 16 bits in each frame, in each direction, for the duration of the call. There's no queuing, so the end-to-end latency is fixed. This is called a **bounded delay**.

Ethernet and IP are **packet-switched**. Packets from many senders share the same links and wait in queues at switches and routers, so delays are **unbounded**: a packet might arrive in a millisecond, or sit behind a burst of other traffic for much longer. This is a deliberate trade-off, not a design mistake. Packet switching is far better suited to bursty traffic. A circuit reserves bandwidth whether or not you use it, while a packet network lets a file transfer use whatever capacity is free right now. The price is that you can never promise how long anything will take.

## You can't trust the clock

Modern computers have at least two kinds of clock, and mixing them up causes real bugs.

- A **time-of-day clock** returns the current date and time, and is synchronised with the Network Time Protocol (NTP). If the local clock has drifted too far ahead, NTP can reset it, so the clock **jumps back in time**.
- A **monotonic clock** is guaranteed to always move forward. If NTP decides it's running fast or slow, it doesn't jump. It *slews*, slightly speeding up or slowing down the clock rate until it's back in line.

<p align="center">
  <img src="../../../images/ddia-clocks.svg" alt="Two charts of clock reading against real time. The time-of-day clock runs fast, then jumps backwards when NTP resets it. The monotonic clock also runs fast, but when NTP corrects it, it slows its rate rather than going backwards." />
</p>

The practical rule follows from this. Use the time-of-day clock for timestamps a human will read. Use the monotonic clock for measuring durations and timeouts. If you measure a timeout with a time-of-day clock and NTP steps it backwards, you can get a negative elapsed time. The absolute value of a monotonic clock is meaningless; it might be the number of nanoseconds since the machine booted. Only the difference between two readings on the *same* machine means anything, so you can't compare monotonic readings across machines.

## No shared memory, no easy answers

On a single machine, multi-threaded code has a well-stocked toolbox: mutexes, semaphores, atomic counters, lock-free data structures and blocking queues. All of them rely on shared memory and on the hardware guaranteeing what every thread sees.

None of that carries over to a distributed system. There's no shared memory, only messages sent over an unreliable network, between nodes whose clocks don't agree and that can pause at any moment. A node can't even be sure what it knows. It might believe it's still the leader when everyone else has moved on.

## Zombies and fencing tokens

That last point leads to my favourite idea in this part of the book.

A **zombie** is a former leader that still thinks it's in charge. Maybe it hit a long stop-the-world garbage collection pause, or was cut off by a network partition. Meanwhile its lease expired and a new leader took over. When the zombie wakes up, it carries on writing as if nothing happened. You now have two leaders, the classic **split brain**, and data gets corrupted.

The fix is a **fencing token**: a number that increases every time the lock or lease is granted. Every write carries the token, and the storage service rejects any write whose token is lower than one it has already seen.

<p align="center">
  <img src="../../../images/ddia-fencing-tokens.svg" alt="Timeline: node 1 gets a lease with token 33, then hits a long GC pause during which its lease expires. Node 2 gets a lease with token 34 and writes successfully. When node 1 wakes up and writes with token 33, storage rejects it because it has already seen token 34." />
</p>

What I like about this is that the zombie doesn't need to know it's been replaced. It can't be relied on to know, because that's the whole problem. The protection lives in the storage, which only has to compare two numbers.

### Who hands out the tokens?

The token issuer can't be a single machine, or it would be a single point of failure. In practice it's a coordination service such as **ZooKeeper** or **etcd**, each running as a cluster of typically three or five nodes. They agree on every token using a consensus algorithm, **Zab** for ZooKeeper and **Raft** for etcd. That leads nicely into the second half of this part of the book.

## Linearizability vs serializability

These two words sound alike, and I'd been guilty of using them loosely. The book makes the distinction clear with an example. Imagine a database with a leader and a replica, and two sports fans, Alice and Bob, both checking the score of a match on their phones.

| Time | What happens |
|---|---|
| 10:00:00.000 | The final score update commits on the leader |
| 10:00:00.050 | Alice's phone reads from the leader: **final score** |
| 10:00:00.100 | Bob's phone reads from the replica, which is 300 ms behind: **game in progress** |
| 10:00:00.300 | The replica receives the update |

<p align="center">
  <img src="../../../images/ddia-linearizability.svg" alt="Timeline: the score update commits on the leader at 10:00:00.000. Alice reads from the leader at .050 and sees the final score. Bob reads from the replica at .100 and sees the game still in progress. The replica only applies the update at .300." />
</p>

Bob got a stale read, even though he asked *after* Alice had already seen the final score.

Is this system **serializable**? It can be. Serializability is about transactions: the result must be the same as if the transactions had run one at a time, in *some* order. Nothing says that order has to match real time. Bob's read is fine as long as we put it before the update in that order.

Is it **linearizable**? No. Linearizability is a recency guarantee. It means the system behaves as if there were only one copy of the data, and once any read has returned the new value, every later read must return it too. Alice saw the final score, so Bob, reading afterwards, should never see the old one.

In short: serializability is about isolating transactions from each other; linearizability is about every read seeing the latest completed write. A database that provides both is called *strictly serializable*.

### Why isn't everything linearizable?

Because it's expensive, and because some replication designs can't provide it:

- **Single-leader** replication can be linearizable, if you read from the leader or from replicas that are updated synchronously.
- **Multi-leader** replication is not linearizable. Two leaders can accept conflicting writes at the same time.
- **Leaderless** replication is generally not linearizable either, even with quorums, because of the edge cases from [Part 2](../ddia-2e-part-2-replication-sharding-transactions/).

In practice, few systems are linearizable, and the reason is usually performance rather than fault tolerance. My favourite example from the book is the RAM in your own computer. On a multi-core CPU, a thread on one core can write a value, and a thread on another core can read the old value straight afterwards. Each core has its own caches and buffers, and writes reach main memory asynchronously. A memory barrier (`volatile` in Java) forces the issue, at a cost. If the machine on your desk doesn't give you linearizability by default, it's no surprise that distributed databases often don't either.

## Consensus: getting linearizability *and* fault tolerance

Linearizability on its own is easy if you're willing to give up fault tolerance. A single-node auto-incrementing counter is linearizable: every request gets the next number, in order. But if that node dies, so does your counter.

This is where **consensus** comes in. Consensus algorithms like Raft and Zab let a group of nodes agree on a value, such as who the leader is or what the next entry in the log is, even when some of them fail. That gives you fault-tolerant, linearizable replication:

- **Leader election**: if the leader fails, the remaining nodes elect a new one automatically, and the algorithm guarantees there's only ever one leader per term.
- **Majority quorums**: a write only counts as committed once a majority of nodes have it. Any future majority must overlap with that one, so a newly elected leader always has every committed write, and nothing is lost during failover.

The two halves of this post connect neatly. Each new leader in Raft gets a term number that only ever increases, and nodes reject messages from older terms. That's the fencing token idea again, built right into the algorithm. That's why ZooKeeper and etcd can hand out fencing tokens to everyone else.

## Wrapping up

This was the most humbling part of the book for me. Networks don't promise delivery times, clocks don't agree and can go backwards, and any node can pause at any moment without knowing it. The book doesn't pretend you can make these problems go away. Instead it shows you the small set of tools that let you build correct systems anyway: timeouts, monotonic clocks, fencing tokens and, underneath them all, consensus.

My takeaway: you shouldn't build your own consensus. You should, though, know when you're relying on it, and what happens to your system when the network or the clock misbehaves.

In the next post I'll move on to derived data: batch and stream processing.
