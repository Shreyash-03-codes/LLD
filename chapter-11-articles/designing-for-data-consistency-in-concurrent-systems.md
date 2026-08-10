# Designing for Data Consistency in Concurrent Systems

## Learning Objectives

- Name the consistency failure modes that appear once more than one writer exists, and which earlier tool stops each one.
- Distinguish a race a single database can still arbitrate from a race the database can no longer settle.
- Recognize the two distributed tools, idempotency and the transactional outbox, and say when each becomes necessary.

## Introduction

Everything in this chapter so far assumed one writer and one database, and that assumption was never stated. Real systems have more threads than one, more instances than one, more stores than one. The guarantee that held when a single transaction owned a row stops holding when two machines each think they wrote it. This last article is about where the lock still works, and where the tool stops being the database.

## Problem Statement

A single-instance service guards an inventory decrement with an optimistic version and it works. The service is scaled to two instances. The two instances still hit the same database, and the version check still holds, so the old tool survives. But a background job now adjusts the same count through a different route, bypassing the version, and a client that retried a call decrements twice. The system loses the single place that used to decide who wins. The lesson is not "lock harder"; it is that consistency stops being one tool and splits into several that depend on the boundary.

## Core Concept

Sort the failure modes by where arbitration lives.

| Failure | Boundary | Tool |
|---------|----------|------|
| Two transactions race one row | one database | transaction + lock (already covered) |
| Retried call applies twice | client to server | idempotency |
| Write succeeds, event is lost | write plus queue | transactional outbox |
| Two stores that both must commit | cross-store | saga, last resort |

The first row is the important misbelief to dismantle.

### The single database still arbitrates

When you scale from one instance to two, but both of them write the same single database, you have not introduced a new problem in the write path. Two instances running `SELECT ... FOR UPDATE` on the same row serialize against each other at the database. Two instances reading the `@Version` on the same table still get the atomic WHERE update. The database is the arbiter, and it already was. Scaling instances does not force you to invent a distributed lock, because the table is still the author of ordering.

What scaling does change is the client-facing retry. The operation is now worth retrying, and a retry against a decrement is a double decrement. The guard is a unique identifier on the write.

```java
public void decrementStock(ProductId orderId, int qty, String requestId) {
    rowJdbc.update("""
        update product
           set qty = qty - ?1
         where id = ?2
           and not exists (
             select 1 from applied where request_id = ?3
           )
        """, qty, orderId.value(), requestId);
}
```

The `requestId` makes the whole retried call idempotent. The second call that carries the same id matches a `not exists` and writes nothing. This is the smallest, cheapest distributed tool, and it changes nothing about the schema's shape, only its contract.

### When the database stops being the arbiter

The moment the consistency spans two stores, the database can no longer decide the ordering for both of them. Two problems appear.

Delivered twice. The event is consumed by two handlers or the same handler retried. The answer is idempotency again, de-dupe on the event's own id at the consumer.

Message lost. The writer commits the business row and the publish to the queue fails. You need the row and the event to commit together, which is the transactional outbox.

```java
@Transactional
public void createOrder(Order order) {
    orders.save(order);
    outbox.save(new OutboxRecord("order.created", order));
}
```

The outbox is a table in the same database, written in the same transaction. A relay reads un-sent outbox rows, publishes each to the queue, and marks it handled. Because the row and the event live in one commit, a crash never leaves the row without an event. That is the guarantee.

### The two facts, in plain terms

There are two facts worth naming, because interviewers fish for exactly one of them.

- Idempotency stops the "happened twice." It relies on a stable key on the operation.
- The outbox stops the "happened once, then disappeared." It relies on writing the event in the same commit as the row.

### The stored row as the boundary

The deepest change is philosophical more than mechanical. Earlier, the invariant belonged in the data, and you enforced it with a transaction. Now the invariant belongs at a boundary, the API, the repository edge, and the cost is carried by idempotency and the outbox, not by a single lock. Recognizing which boundary you are on is the whole skill.

### A story of the two flows

Diagram: the two distributed failure modes and their fixes.

<svg width="900" height="340" viewBox="0 0 900 340" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="arr3" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto">
      <path d="M0,0 L7,3 L0,6 Z" fill="#222"/>
    </marker>
  </defs>

  <text x="40" y="34" font-family="Arial" font-size="15" font-weight="bold" fill="#222">Retried call</text>

  <rect x="40" y="52" width="120" height="48" rx="8" fill="#eef2f7" stroke="#333" stroke-width="2"/>
  <text x="100" y="70" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Client retry</text>
  <text x="100" y="88" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">same requestId</text>

  <rect x="240" y="52" width="120" height="48" rx="8" fill="#fbf0f0" stroke="#962828" stroke-width="2"/>
  <text x="300" y="70" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">decrement x2</text>
  <text x="300" y="88" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">double spent</text>

  <rect x="460" y="52" width="130" height="48" rx="8" fill="#eef7ee" stroke="#2f6f3e" stroke-width="2"/>
  <text x="525" y="70" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">requestId gate</text>
  <text x="525" y="88" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">second is a no-op</text>

  <path d="M 162 130 C 185 140 205 140 232 130" fill="none" stroke="#222" stroke-width="2" marker-end="url(#arr3)"/>
  <path d="M 362 130 C 385 140 405 140 452 130" fill="none" stroke="#222" stroke-width="2" marker-end="url(#arr3)"/>

  <text x="40" y="186" font-family="Arial" font-size="15" font-weight="bold" fill="#222">Publish lost</text>

  <rect x="40" y="204" width="120" height="48" rx="8" fill="#eef2f7" stroke="#333" stroke-width="2"/>
  <text x="100" y="222" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Commit row</text>
  <text x="100" y="240" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">then queue fails</text>

  <rect x="240" y="204" width="120" height="48" rx="8" fill="#fbf0f0" stroke="#962828" stroke-width="2"/>
  <text x="300" y="222" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">row saved</text>
  <text x="300" y="240" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">event dropped</text>

  <rect x="460" y="204" width="130" height="48" rx="8" fill="#eef7ee" stroke="#2f6f3e" stroke-width="2"/>
  <text x="525" y="222" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">outbox</text>
  <text x="525" y="240" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">relay retries</text>

  <path d="M 162 204 L 232 204" fill="none" stroke="#222" stroke-width="2" marker-end="url(#arr3)"/>
  <path d="M 362 204 L 452 204" fill="none" stroke="#222" stroke-width="2" marker-end="url(#arr3)"/>
  <text x="320" y="142" text-anchor="middle" font-family="Arial" font-size="12" fill="#555">both share one commit</text>
  <text x="320" y="158" text-anchor="middle" font-family="Arial" font-size="12" fill="#555">so a failure cannot split them</text>
</svg>

### The two guarantees, restated

For the "applied twice" failure the tool is a stable key, and the database still decides. For the "row and event" divergence the tool is the same transaction extended by the outbox. Neither needs a second datastore.

## Real Production Usage

Kafka consumers de-duplicate on an event id in practice, which is idempotency at the consumer. Outboxes show up wherever you must "write a business row and also emit an event," with a relay that publishes the rows and marks them sent. Spring pairs a JPA transaction with `@TransactionalEventListener` so the event is fired after the commit. The only new structure is the outbox table, and it is a deliberate extra store, not a surprise third database.

## Common Mistakes

1. **Buying a distributed lock for two instances over one database.** The table already serializes the row; the lock server is a distributed cost solving a problem that did not exist here.
2. **Skipping the request id on a retryable write.** A network retry becomes a double apply and the version check that used to guard becomes a version that no longer sees the second copy.
3. **Emitting the event outside the same transaction as the write.** It races the row-count, and a crash leaves the two worlds unequal, which is exactly the outbox failure being asked about.

## Interview Perspective

Interviewers probe whether you can say where a tool stops. Weak: "make everything idempotent and you're done" or "use a saga." Strong: "separate the cases: one DB still arbitrates with its own lock, a retried call needs a request id, a row plus an event needs the outbox shared in one commit." They want you to avoid over- and under-engineering, so your answer names the boundary.

Follow-up: "Does scaling to two instances change your concurrency solution?" with the honest one: no for the database row write, yes for the retried call.

## Knowledge Check

1. Two instances decrement the same row via `FOR UPDATE`. Are they a new concurrency problem, and where does the justification live?
2. A client retries a decrement. Which single identifier turns the second attempt into a no-op?
3. Why must the outbox row and the business `create` share one commit, and what exact failure does that protect?

## Key Takeaways

- Scaling instances does not change a shared-row write; the database still arbitrates it.
- Idempotency turns a retried call into a no-op by keying the writes to a stable request id.
- The outbox shares one commit between the business row and the event, so neither can be lost alone.

## What's Next

Persistence, on one node and then across a few stores, is now behind you. The next chapter is Concurrency and Multithreading, where the same shape appears inside a single machine: threads stepping on a shared `ConcurrentHashMap`, a mutex guarding a shared balance, and an executor pool starving under a poorly sized bound. The vocabulary you built here, lock, invariant, retry, and owner, carries straight over, but now the contention is on the JVM heap rather than a row in a database.

---

This article explains how to tell the consistency problems one database still arbitrates from the ones a distributed tool must fix, and names idempotency and the transactional outbox as the two reliable answers. It argues that a single database already orders a shared row, and that a distributed lock is usually invented precisely where the database stopped deciding.