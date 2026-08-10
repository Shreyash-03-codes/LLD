# Synchronization and Locks

## Learning Objectives

- Explain why `synchronized` exists and what it actually protects against.
- Read a block and say what the lock is, what is covered, and what is left out.
- Know the difference between locking an object and locking a class, and between a method and a block.

## Introduction

The last article ran your tasks on a pool, which means many threads now call the same code at once. When two threads touch one counter, the writes interleave and the value comes out wrong. Synchronization is the tool that says a piece of code runs with exclusive access, so only one thread enters at a time. The keyword is `synchronized`, and the mechanism behind it is a monitor, an object lock that the JVM hands out by mutual exclusion.

## Problem Statement

Two threads check a bank account: read the balance, then update it. The statements look atomic but the CPU is a party of steps, and thread A reads 100 and thread B reads 100 before either writes, then both write 99 and the account dropped by one cent instead of two. Worse, without a memory barrier a thread may never see another thread's write at all because of caching. Lost updates and stale views are the two failures, and neither shows up as a crash, only as wrong totals.

## Core Concept

`synchronized` acts as both a lock and a memory barrier. The lock guarantees exclusive entry, and the barrier guarantees that a thread, after it leaves the block, sees everything the previous holder did inside it. Without the second part, even a "safe" sequence of writes can be invisible.

```java
class Account {
    private int balance;

    synchronized void debit(int amount) {
        this.balance = this.balance - amount;
    }
}
```

The block runs against the monitor of `this`. A static `synchronized` method locks on the `Class` object instead, because there is no instance. Use the two for the same data and you have two different locks guarding one field, which is a common and silent mistake.

You can scope a lock to a block, which lets you hold it for precisely the lines that need it and release it early:

```java
synchronized (lock) {
    counter++;
}
```

A method-wide lock is simple but lazy: it holds the monitor while doing work that does not need it, like a slow network read, and that blocks every other thread for no reason. The trade runs through the whole debate about coarse lock versus fine lock.

## Real Production Usage

The bank account is one of the hottest `synchronized` patterns, but in production it is often the wrong tool for the designers. Serializing on a single counter or a map update is fine. Locking a database read is not, because the database has its own control. In real services, `synchronized` shows up inside small primitives and concurrent collections, and the higher layers use `ConcurrentHashMap` and `Atomic` classes instead of a big method-wide lock, which keeps throughput high. A good rule: keep the lock short, because the cases that fail you are locks held while a thread waits on the network.

## Common Mistakes

1. **Locking on different objects.** Two threads sync on `this` and two on the class, so the field waits for nothing.
2. **Locking on a mutable field.** A `synchronized(list)`, then you reassign the list, and now the locks do not agree.
3. **Holding a lock through a slow I/O call.** Every other thread queues behind the network read, or worse, a deadlock.
4. **Locking only the write, not the read.** Protecting the write side while the read side goes unsynchronized is the classic staleness.

## Interview Perspective

Interviewers probe "what does synchronized do" to separate the people who recite the keyword from the ones who know it is both a lock and a barrier. The strong answer mentions the monitor, the memory barrier, and that it is reentrant, meaning the same thread can lock the same object again. Follow-ups: "where does the thread wait when the lock is taken" and "how is a static synchronized different." A good last line is that you want the smallest block that still covers every read and write, on the same object.

## Knowledge Check

- Two threads run `synchronized` on different objects that guard the same field. Why is the field still unprotected?
- Why does a thread exiting a `synchronized` block make its writes visible to the next thread to enter?
- When would a block-sized lock beat a method-wide lock?

## Key Takeaways

- `synchronized` is a monitor lock on an object, and it is the cheapest correctness tool you have.
- It is also a barrier: what one thread writes in the block is visible to the next.
- Lock the same object on every path, keep the block short, and don't lock on a mutable field.

## What's Next

Plain `synchronized` locks only one object and only one way. The next article is reentrant and read-write locks, which add explicit control, a way to wait without blocking on the monitor, and the ability to let many readers in at once instead of one at a time.

---

This article explains `synchronized` as both a mutual-exclusion lock and a memory barrier, and how a monitor held on one object guarantees only one thread sees a section at a time. It argues that lock conflicts live in the coverage, not the keyword, and that the smallest block on a stable object beats a whole method.