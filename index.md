---
layout: splash
title: "DublinTech"
excerpt: "A collection of Technical Articles by Alex Staveley."
header:
  overlay_color: "#1a1a2e"
  overlay_filter: 0.5
read_time: false
---

<div style="display:flex; align-items:center; gap:1.5em; margin:2em 0 2.5em;">
  <img src="{{ "/images/me.jpg" | relative_url }}" alt="Alex Staveley" style="border-radius:50%; width:90px; height:90px; object-fit:cover; flex-shrink:0;">
  <div>
    <strong>Alex Staveley</strong><br>
    Software architect with 25+ years of experience. Writing about AI Coding Tools, architectural governance, and engineering discipline in modern software delivery.
  </div>
</div>

---

## AI in the SDLC

- [Part 1 - Compare & Contrast ArchUnit and AI Coding Tool steering .md files](posts/aisdlc/ai-sdlc-part-1-archunit-vs-llm-md-files/) — Compares ArchUnit and AI Coding Tool steering files as two complementary approaches to encoding architectural principles.

- [Part 2 - Indicating Your Design Debt to AI Coding Tools](posts/aisdlc/ai-sdlc-part-2-archunit-design-debt/) — How to annotate known design debt so AI Coding Tools don't accidentally reproduce architectural anti-patterns.

- [Part 3 - Linking Design Debt to ArchUnit](posts/aisdlc/ai-sdlc-part-3-archunit-design-debt/) — Extends Part 2 by connecting design debt annotations to executable ArchUnit rules to prevent new violations automatically.

## Book Reviews

### Designing Data-Intensive Applications (2nd Edition)

- [Part 1 - Setting the Scene](posts/book-reviews/ddia-2e-part-1-setting-the-scene/) — The foundations: percentiles over averages, faults vs failures, choosing a data model, LSM-trees vs B-trees, and why data outlives code.

- [Part 2 - Replication, Sharding and Transactions](posts/book-reviews/ddia-2e-part-2-replication-sharding-transactions/) — Replication logs and lag, multi-leader and CRDTs, quorums, sharding and secondary indexes, snapshot isolation and write skew.

- [Part 3 - Problems with Distributed Systems, and How Consensus Helps](posts/book-reviews/ddia-2e-part-3-distributed-trouble-and-consensus/) — Unbounded network delays, unreliable clocks, zombies and fencing tokens, linearizability vs serializability, and consensus.
