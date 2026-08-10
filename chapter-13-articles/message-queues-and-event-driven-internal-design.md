# Message Queues: Concepts and Internal Structure

## Learning Objectives

- Name the pieces of a log-based queue: partition, offset, and consumer group.
- Explain the trade a partitioned log makes between per-key ordering and parallelism.
- State what the offset commit decides and what a consumer owes for at-least-once.

## Introduction

A topic needs a home, and the home is a log. This article is the internal layer of the broker, the parts you inherit the moment you pick Kafka or RabbitMQ. The honest framing is a story about a partitioned, keyed log: a fixed number of ordered append logs, an offset that says how far a consumer has read, and a consumer group that hands out parallel slices. Read a queue as a log and the forward-looking pieces of an event-driven system stop being dark magic.

## Problem Statement

A catalog service writes hundreds of thousands of events per second, and a consumer draining a single ordered list is far too slow. You want parallel readers, but two parallel readers must not both consume the same order's events out of sequence. Then a broker node dies and a consumer has to know where its reading stands. Ordering, parallelism, and replay are three demands pulling in different directions, and without a clear model the queue is where they collide.

## Core Concept

A message queue is not one list. It is a fixed number of ordered, append-only logs called partitions. An event goes to whichever partition its key hashes to, and inside a partition the events are ordered by arrival. Every event gains an offset, its position in that partition. The ordering guarantee is deliberately narrow: it is per partition and per key, never across the whole topic.

```java
ProducerRecord<String, OrderPlaced> record =
    new ProducerRecord<>("orders", order.getId(), event);
producer.send(record);  // the key picks the partition
```

The consumer group is how parallelism is reconciled with order. A group is a set of members that together read the topic. Each partition is assigned to one member at a time, so all the events for one key arrive in one partition and are drained in order by one member. The group sets the ceiling on parallelism: with three partitions you can run three busy members, and a fourth consumer adds nothing.

```java
consumer.subscribe("orders"); // the group shares the offsets for the topic
```

The last piece is the commit. A consumer reads up to offset 5, does its work, then commits that it reached offset 6. If the consumer dies, the group restarts from the last committed offset. Where the commit sits decides the delivery semantics:

- At-least-once: do the work, then commit. A crash before the commit replays the event, so duplicates can happen and the consumer must be idempotent.
- At-most-once: commit before the work. A crash can skip an event, no duplicates, best for a miss you can absorb.
- Exactly-once: runs on transactions and deduplication, it is a hard feature of the product, never the default of a plain queue.

Diagram: a topic splits into three partitions, and each worker in the consumer group reads exactly one.

<svg width="920" height="430" viewBox="0 0 920 430" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="qm" markerWidth="10" markerHeight="10" refX="8" refY="4" orient="auto">
      <path d="M0,0 L8,4 L0,8 Z" fill="#333"/>
    </marker>
  </defs>

  <rect x="40" y="40" width="300" height="150" rx="8" fill="#fdf6e3" stroke="#8a6d00" stroke-width="2"/>
  <text x="190" y="64" text-anchor="middle" font-family="Arial" font-size="14" fill="#222">Topic: orders</text>

  <rect x="60" y="90" width="260" height="40" rx="4" fill="#eef2f7" stroke="#555" stroke-width="1.5"/>
  <text x="190" y="115" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Partition 0 (key)</text>
  <rect x="60" y="152" width="260" height="40" rx="4" fill="#eef2f7" stroke="#555" stroke-width="1.5"/>
  <text x="190" y="177" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Partition 1 (key)</text>
  <rect x="60" y="214" width="260" height="40" rx="4" fill="#eef2f7" stroke="#555" stroke-width="1.5"/>
  <text x="190" y="239" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Partition 2 (key)</text>

  <path d="M 320 110 L 560 110" stroke="#333" stroke-width="2" fill="none" marker-end="url(#qm)"/>
  <path d="M 320 172 L 560 172" stroke="#333" stroke-width="2" fill="none" marker-end="url(#qm)"/>
  <path d="M 320 234 L 560 234" stroke="#333" stroke-width="2" fill="none" marker-end="url(#qm)"/>

  <rect x="560" y="40" width="300" height="150" rx="8" fill="#eef7ee" stroke="#2f6f3e" stroke-width="2"/>
  <text x="710" y="64" text-anchor="middle" font-family="Arial" font-size="14" fill="#222">Consumer group</text>

  <rect x="580" y="90" width="260" height="40" rx="4" fill="#ffffff" stroke="#2f6f3e" stroke-width="1.5"/>
  <text x="710" y="115" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Member A</text>
  <rect x="580" y="152" width="260" height="40" rx="4" fill="#ffffff" stroke="#2f6f3e" stroke-width="1.5"/>
  <text x="710" y="177" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Member B</text>
  <rect x="580" y="214" width="260" height="40" rx="4" fill="#ffffff" stroke="#2f6f3e" stroke-width="1.5"/>
  <text x="710" y="239" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Member C</text>

  <rect x="40" y="330" width="400" height="54" rx="6" fill="#f8f4e8" stroke="#8a6d00" stroke-width="1.5"/>
  <text x="240" y="354" text-anchor="middle" font-family="Arial" font-size="12" fill="#222">offset 0..N</text>
  <text x="240" y="372" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">committed after the work</text>
</svg>

The commit box sits at the bottom on purpose. The offset is the cursor, and the commit is the decision about where the cursor is published. The important companion fact is that the log retains the data for a configured window, so replay is a normal consumer action, not a reconstruction of something lost.

## Real Production Usage

Kafka is the log, and the group is an event-handling service. The offset is persisted by the consumer, so the work resumes on a rebalance from the last committed offset, and the commit is placed after the side-effect write, wrapped in a transaction or made idempotent. spring-kafka hides the listener, the group, and the ack behind a template, but the model on this page is the one underneath. The rule of thumb I keep: make the consumer idempotent and treat every redelivery as a no-op, because in the long run you will be redelivered.

## Common Mistakes

1. **Adding a consumer past the partition count.** Extra threads sit idle, because the throughput is set by the partition count and the group size, not by the number of workers you threw at it.
2. **Committing the offset before the side-effect.** A crash skips the event and the reply never fires; choose where the commit sits and prove it.
3. **Presuming order across the topic.** Two partitions, two members, and the events for two orders can be applied in a different order than they were written. Order on the key, not on the stream.

## Interview Perspective

The interviewer wants the tension: how a topic is both parallel and ordered. Weak: "a message is consumed by one reader." Strong: "a topic splits into partitions; order is per partition, the consumer group assigns each partition to one member, and the offset carries the state, so each key is one serial member while several keys run in parallel." Follow-up: "what does a rebalance do" and "how does at-least-once duplicate." Both hinge on the same commit: a rebalance hands a partition away and the new member resumes from the last committed offset, so a crash between the work and the commit replays the event.

## Knowledge Check

- A consumer applies an event, updates its read model, then commits. If it crashes between the update and the commit, what is replayed and what delivery mode is that?
- You need all events for one order consumed in order while the topic stays fast overall. Which of the partition, the producer key, the group size, and the commit make that true?
- A topic has four partitions and six consumers in a group. What is the ceiling on busy consumers, and what are the two idle ones doing?

## Key Takeaways

- A topic is partitioned into order per partition and per key, never per whole topic.
- The consumer group is the only place parallelism meets order, one member per partition.
- The commit position is the policy, at-least-once or at-most-once, and the consumer is idempotent for the predictable.
- The whole log is the record; replay is reading a history, not a special case.

## What's Next

The queue creates the lag between the publish and the consumer's view, and the next article is about living with it. Eventual consistency and what it means for object design speaks to the fact that a consumer reads a stale view and rebuilds it, which the offset commit and the replay above make the daily reality.

---

This article explains the internals of a message queue as a partitioned keyed log with an offset and a consumer group, the pieces that give ordering per key, parallelism across members, and at-least-once replay. It argues that the position of the commit is the entire delivery policy, and that an idempotent consumer is the non-negotiable price of at-least-once.