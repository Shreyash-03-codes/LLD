# Concurrent Collections: ConcurrentHashMap and Friends

## Learning Objectives

- Explain why `HashMap` and `ArrayList` are not safe across threads and what that means in practice.
- Read a `ConcurrentHashMap` and say how it stays fast under writers, and what reads may miss.
- Pick a concurrent collection for a short pattern instead of a synchronized wrapper.

## Introduction

The previous articles handled a lock object. Most real code never touches a lock, because the work belongs to a map or a queue that someone already made thread-safe. The `java.util.concurrent` collections wrap serious correctness into ordinary method calls, so a shared map can be pounded by readers and writers without you writing a single lock. The price is a precise honesty: they trade away a global, instant snapshot, and claiming otherwise is where bugs appear.

## Problem Statement

A billing service keeps a `HashMap` of account to pending invoice, and two worker threads call `put` at once. The map rebalances and one thread steps through a stale structure and throws or corrupts the table. You could wrap it in `Collections.synchronizedMap`, but then every read locks the whole map, and a cache that is read sixty times for every write spends most of its time blocked. You want a map that is safe to share without pausing all readers, and that is exactly what the concurrent map gives in exchange for a weaker view of itself.

## Core Concept

`ConcurrentHashMap` does not lock the whole table for every access. It splits the table into many slots and each thread claims only one at a time, so reads look at whatever the current bucket holds and do not wait. The write updates one and the read, which may be a different one, proceeds. This keeps high throughput, but it changes what you are allowed to assume.

The honest part is the `size()`. Because updates happen in parallel lock short buckets, the count is approximate. There are field-based helpers that encourage the doubt: the `putIfAbsent`, `compute`, and `merge` fold their check and update into one atomic action, so the code never reads the old value and writes it back separately.

```java
ConcurrentHashMap<String, Long> byCountry = new ConcurrentHashMap<>();

byCountry.compute("IN", (k, before) -> before == null ? 1L : before + 1L);
```

The bare version that reads then writes is a lost update, even though the map is "thread-safe." "Thread-safe" here means the stored bytes never corrupt, not that your read-then-write sequence is atomic. Concurrent queues and copy-on-write lists pick up where a map stops. `CopyOnWriteArrayList` copies the backing array on every write, so reads are never blocked; that is right for the read-dominant, subscription-style lists. `ConcurrentLinkedQueue` gives lock-free queue behavior. `BlockingQueue` adds wait-for-an-element and capacity limits.

## Real Production Usage

`ConcurrentHashMap` is the workhorse of the average service, the default pick for a shared map, from request counters to caches hit thousands of times a second. The honest truth is it is right for "many reads, some writes, wants to be fast", and the mistakes come from assuming the stale-read behaviors are bugs. The queues and the copy-on-write lists cover the rest. `Hashtable` and `Collections.synchronizedMap` are the legacy of whole-map locking.

## Common Mistakes

1. **"If the map is safe, my composition is safe."** A check-then-act, `if contains then put`, defeats the map and still races.
2. **Relying on `size` for a threshold.** The count is approximate by design; a gate on it can exceed your guard.
3. **`CopyOnWriteArrayList` with huge writes.** Copies the whole array for every write; write-heavy code pays for it.
4. **Modifying a map while a stream runs over it.** You get a source that changed under the stream.
5. **Believing null keys and values are allowed.** They are not in the concurrent map, a surprise from `HashMap`.

## Interview Perspective

Interviewers ask "how is a ConcurrentHashMap different from HashMap" to check real understanding of the concurrency. The strong answer: it splits the table into segments and does no global lock, so many readers parallel, writes lock one slot, and it trades a strong `size` for the weak one. Follow-up: "is a `get` always correct" and "why not just use it everywhere." An honest answer: it is a safe cache, often the right call, but the moment the logic reads then writes a derived value, the map does not save you.

## Knowledge Check

- Why does a check-then-act fail on a `ConcurrentHashMap` even though the map is thread-safe?
- What does `computeIfAbsent` fold into one step that a `get` then `put` does not?
- Why is an exact `size()` a trade rather than an accident, and what does it cost?

## Key Takeaways

- `ConcurrentHashMap` is safe but not atomic for your own check-then-act; the `compute*` helpers are the atomics.
- The concurrent map traded the exact `size` for throughput, so read patterns that gate on count are approximate.
- Pick a `BlockingQueue` or a copy-on-write list when the access is a queue or an event broadcast, not a muddy guard.

## What's Next

Collections carry the data, but a single background call still blocks the answer. The next article is `Future` and `CompletableFuture`, staging a value that arrives later and wiring it into a pipeline without a thread sitting and waiting.

---

This article explains how the concurrent collections, led by `ConcurrentHashMap`, deliver thread safety by locking small and letting reads run in parallel. It argues that the trade for speed is a weaker global view, so compound read-then-write still needs the atomic helpers, and the map does not make your own composition correct.