# Reentrant Lock and Read-Write Lock

## Learning Objectives

- Explain what a plain `synchronized` cannot do that an explicit `Lock` can.
- Use `ReentrantLock` with its `lock`, `unlock`, and `tryLock` while respecting the idiom.
- Know when a `ReentrantReadWriteLock` beats a plain lock, and when it makes things worse.

## Introduction

`synchronized` from the last article is a tool with no handles: you get in, you get out, and you cannot look before you lock. The `Lock` interface is the same idea with a dashboard. `ReentrantLock` gives you a try that does not block, a way to wait politely, and a lock you can acquire again in the same thread. The read-write variant splits the lock so many readers can enter while writers enter alone. Both make you buy what `synchronized` gave free: the compiler will not release the lock for you.

## Problem Statement

One writer holds `synchronized` on a cache and every reader queues behind it. The reads vastly outnumber the writes, so a single lock serializes dozens of readers who only want the same map. And when the lock is contended and you want a code path that gives up gracefully instead of waiting forever, `synchronized` gives no choice at all. The problem is a lock system that treats one reader the same as one writer, and cannot be told to give up.

## Core Concept

`ReentrantLock` sits in place of a monitor but hands control back to you. The counterpoint to remember is that it is reentrant: the same thread can call `lock()` twice and must call `unlock()` twice. Failing to unlock once leaves a lock held forever and the app stalls mysteriously. The idiom is always try-finally.

```java
Lock lock = new ReentrantLock();
lock.lock();
try {
    protectedCounter++;
} finally {
    lock.unlock();
}
```

The whole point of the explicit type is the extra verbs. `tryLock(duration)` returns a boolean instead of hanging, which is how a thread backs off instead of blocking forever. `lockInterruptibly()` lets a waiting thread notice a signal. `newCondition()` drops the monitor's implicit wait-notify and gives you a named condition to wait on. That set turns a binary in-or-out into workload choices.

`ReentrantReadWriteLock` splits the one lock into a read and a write. Any number of readers can hold the read lock together, but the write lock is exclusive and waits for all readers. The reads go wide when the data is read far more than it is written, and the write stays correct. The catch is that a reader that thinks it needs to write has no clean path, the upgrade is fragile, and it usually hurts when reads are short or the map is small, because the bookkeeping and cache-line fighting outweigh the concurrency gained.

See the flow:

```
ReadWriteLock:
 many readers  -> read lock  -> all parallel
 one writer    -> write lock -> waits for readers, then alone
```

## Real Production Usage

A `ReentrantLock` is what you reach for when `synchronized` is too blunt and you need `tryLock` or a timeout or you are building a cache that must not blow up one slow read for everyone else. The read-write lock shows up in hand-rolled caches and rate-limited stores, but the honest note is that most services use `ConcurrentHashMap`, which does its own reader-writer dance in hardware, and never ask for the read lock. When you do write one, keep the critical sections short, whatever the lock.

## Common Mistakes

1. **A lock leak on an early return.** Throwing or returning before `unlock` leaves the lock held.
2. **Reaching into the write lock for a read.** A reader that grabs the write lock serializes every other reader for nothing.
3. **Using a read-write lock where the writes dominate.** The extra machinery costs more than a plain lock.
4. **Forgetting a `tryFinally` always reaches.** The `finally` releases whatever path the method took.

## Interview Perspective

Interviewers ask "when do you reach for explicit Lock" to separate people who know the verbs from those who know the failure. Strong: "I use `tryLock` when a wait must not be fatal, and a read-write lock when I read far more than I write, and I remember the lock goes in a `finally`." Follow-up: "what could deadlock with two read-write locks" and "is read-write faster than a plain lock." A steady answer is that a read-write lock is often slower than a good `ConcurrentHashMap`, so the real decision is the data structure, not the lock.

## Knowledge Check

- What happens if a thread calls `lock()` twice and `unlock()` once on a `ReentrantLock`?
- A cache is read far more than written. What does a read-write lock give that a plain lock does not?
- Why is a `tryLock` a better fit than a plain `lock` when a request must not wait forever?

## Key Takeaways

- `ReentrantLock` is `synchronized` plus verbs, and it stays on if you forget the `finally`.
- `tryLock` and `lockInterruptibly` turn "wait forever" into "back off".
- A read-write lock wins only when reads dominate, and it rarely beats a concurrent collection. Choose the structure first.

## What's Next

Locks give a thread something to queue on, but the queues they make are silent. The next article is the semaphore and countdown latch, tools that put an explicit number of permits in front of a section, and a gate that waits until a set of parties appear before opening.

---

This article explains `ReentrantLock` as a manually controlled lock and `ReentrantReadWriteLock` as a lock that lets many readers in while writers wait. It argues that the extra verbs are bought with the duty to unlock, and that a read-write split only earns its overhead when reads truly dominate the writes.