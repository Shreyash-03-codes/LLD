# Semaphore and CountDownLatch

## Learning Objectives

- Explain what a semaphore counts and when a permit is not a lock.
- Use `Semaphore` to cap how many threads enter a section.
- Use a `CountDownLatch` to make a coordinator wait until a set of threads finish a phase.

## Introduction

Locks are binary: one thread owns the section or none does. A semaphore is a ticket system. You hand out a fixed number of permits, threads take one to enter, and release it when they leave, so up to a chosen count can run at once. The countdown latch is a gate counting the other direction: it waits for a number of events to happen before it opens. Both are concurrency primitives that express capacity and coordination instead of plain mutual exclusion.

## Problem Statement

A service spins up threads that each call an external rate-limited store. The limit is ten calls per second, but nothing counts. A hundred threads run at once and the provider refuses the request. You need "at most ten active", which a lock cannot express because a lock is an all-or-nothing gate. Separately, a startup depends on three caches and a config being ready, and the main thread must wait until all four tasks complete before serving. Two timing problems, each needing a door over a count.

## Core Concept

A `Semaphore` holds a number of permits. `acquire()` takes one or blocks until one frees, and `release()` returns one. A semaphore of ten limits ten threads inside the guarded call. A semaphore of one behaves almost exactly like a lock, which is the tell that they are the same family.

```java
Semaphore limit = new Semaphore(10);
try {
    limit.acquire();
    store.call();
} finally {
    limit.release();
}
```

`tryAcquire()` does not block; it returns false the moment the count is exhausted, so a caller can fail fast or back off instead of queuing. And a semaphore can be given to several threads, unlike a lock that is owned by one, which is why the fair flag matters when you care who waits.

A `CountDownLatch` counts down, not up. You start it at N and each `countDown()` drops it by one; `await()` blocks until it reaches zero. The constructor argument is a number of events, and the gate opens once all of them have arrived, then it stays open forever. It cannot be reused, which is why a `CyclicBarrier` exists for the resetting version.

```java
CountDownLatch ready = new CountDownLatch(4);
new Thread(() -> { freshCache(); ready.countDown(); }).start();
ready.await();
serve();
```

The latch is one-way: the threads that `countDown` are workers, and the one thread on `await` is the coordinator waiting for them.

## Real Production Usage

Semaphores are the throttle in front of connection pools, ratio-limited APIs, and test driver steps: a packed set of task slots shared across a pool, not owned by one thread. The latch is the standard "wait for setup" pattern at boot and in distributed load tests. Both are simple to write and honestly rare in the hot path, because the pools and queues in the executor already did most of the capacity work. You add a semaphore when the limit is a real external bill, and a latch when startup order matters.

## Common Mistakes

1. **Acquiring but never releasing.** A crashed `try` that misses the `finally` slowly leaks permits until no one can pass.
2. **Choosing a wrong N for the latch.** Counting inline operations, not complete events, means the door opens a step too early or too late.
3. **Swallowing `InterruptedException`.** Help the calling code and you lose the interruptible instead of passing it on.
4. **Reusing a latch after it hits zero.** It stays open and the coordination silently does nothing.

## Interview Perspective

The question is usually "semaphore versus mutex" or "semaphore versus latch." The strong answer: a semaphore counts any number of permits and release need not be on the acquiring thread; a mutex is a single owner and reentrant. For the latch, say you start at N, workers call `countDown`, and the coordinator `await`s until it hits zero. Follow-up: "semaphore of a million, what does it control" and "what would you use to repeat a countdown." The trap to name is that a small semaphore looks like a lock, and people over-answer instead of noting it counts down.

## Knowledge Check

- A semaphore of one behaves almost like what other tool?
- When does `tryAcquire()` return instead of blocking, and what does a caller do with it?
- A latch opens and cannot be reset. What tool covers the case where the countdown must begin again?

## Key Takeaways

- A `Semaphore` lets a fixed N threads in and counts down with each permit. A single permit is nearly a lock.
- `tryAcquire` is the non-blocking knob; keep `release` in a `finally`.
- A `CountDownLatch` waits down to zero once and cannot be reset, which is the sign that a cyclic barrier waits for restart.

## What's Next

The collection classes never once appeared in the countdown, but concurrency mainly runs through them. The next article is the concurrent collections, the maps, lists, and queues that carry the locks under the hood and save you from writing one.

---

This article explains the `Semaphore` as a fixed-count of permits that limits how many threads can be inside a section, and the `CountDownLatch` as a gate that opens when a set of events has drained to zero. It argues that both replace a personal lock with a shared count, and that each leaks guarantees when you mishandle the permits or the count.