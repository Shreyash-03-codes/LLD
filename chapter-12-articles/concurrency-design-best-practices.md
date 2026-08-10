# Concurrency Design Best Practices

## Learning Objectives

- Name the handful of habits that keep a multithreaded service steady under load.
- Choose the smallest correct concurrency tool instead of the biggest lock.
- Recognize when a shared design is the problem and remove the sharing instead of patching it.

## Introduction

This chapter handed you threads, pools, locks, collections, and futures. The practices in this last article are the judgment that decides which of them to reach for, and in what order. Most concurrency bugs come not from a missing tool but from choosing the wrong level of the stack: a big lock where a collection would do, a shared mutable field where a copy would do, a hand-built wait where a queue would do. The best design is the one where the concurrency is a decision you made, not a consequence you met.

## Problem Statement

A service gets slower and slower under load, and the team finds a pool stuck, a cache that misses constantly, and a `synchronized` block that covers a network call. None of it is wrong in a single method; all of it is a pile of locks and waiting that should not exist. The failure is systemic: too much sharing, too-wide locking, and no single owner for the concurrency decisions. Fixing it means rethinking the design with the tools ordered by cost.

## Core Concept

The practices, in the order a designer should apply them.

**Prefer no sharing.** The cheapest concurrency is the one that is not there. Copy the data, keep the state in one thread, or make the value immutable, and the locks vanish. The hardest problems are designed in when a mutable object is passed between threads for no reason.

**Choose the narrowest tool.** An atomic field for a counter, a concurrent map for a shared cache, a pool for a workload, and a lock only when the collection cannot express the rule. Most code that reaches for `synchronized` or a `ReentrantLock` is guarding a check-then-act that a `compute` or an atomic already folds.

**Keep critical sections short.** If a lock is needed, hold it for the fewest instructions, and never across an I/O call. A lock held through a network read makes every other thread wait on the network, the classic hidden serialization.

**Lock in a consistent order.** With two or more locks, order them so a cycle cannot form. The deadlock chapter's fix lives here.

**Use the executor, not a thread.** Ask a pool for work instead of building a `Thread`. The executor brings the bound, the queue, and the shutdown, for free.

**Time the waits.** Every blocking call gets a timeout and a failure path. A future without a timeout, a lock without a `tryLock`, and a wait without a `whenComplete` each hang on the same spot: a condition that never arrives.

**Name the strategy at the boundary.** A class shared across threads states whether it is immutable, confined, or guarded, so the next maintainer knows what holds.

```java
ExecutorService pool = newFixedThreadPool(4);   // bound the workers
pool.submit(() -> cache.computeIfAbsent(key, load()));  // narrow tool
future.get(2, TimeUnit.SECONDS);                // time the wait
```

## Real Production Usage

The companies that run large concurrent services converged on the same playbook: isolate state by request, share only through concurrent collections and message queues, bound every pool, and never let one slow dependency block the whole worker set. The honest note is that the playbook is not exotic. It is these practices applied steadily. When a production issue is a thread dump of a blocked pool, the fix is rarely a new concurrency primitive; it is one of these habits restored, a copy that removes the sharing or a timeout that lets a stalled call fail.

## Common Mistakes

1. **Sharing mutable state by default.** Passing a live object to a pool and adding locks later instead of passing a copy now.
2. **A coarse lock over a fine problem.** Wrapping a whole map in `synchronized` where a concurrent map reads in parallel.
3. **A lock across I/O.** Every thread waits on one network call, a serialization hidden behind a method name.
4. **A thread per task.** No pool, no bound, and a spike of work becomes a spike of threads.
5. **An untimed wait.** A blocking call with no timeout that holds a request forever.

## Interview Perspective

The design questions are "how would you fix a slow pool" and "what makes your system safe under load." The strong answer orders the tools: no sharing first, then the narrowest concurrent type, then a lock only when needed, then a timeout. Follow-up: "the lock and the network call" and "why an atomic over a synchronized." The answer to close on: the goal is to remove the sharing or the wait, and a lock is the last resort, not the first line.

## Knowledge Check

- What is cheaper than any lock, and what design move achieves it?
- Why should a lock never be held across a network call?
- Name the three knobs a blocking wait should carry in a production design.

## Key Takeaways

- No sharing beats good sharing; copy, confine, or make the value immutable.
- Reach for the narrowest tool and hold a lock for the shortest section.
- Bound every pool, order every lock, and time every wait.

## What's Next

The tools of this chapter are all about threads that run side by side and share. The next chapter is event-driven design, which changes the material: instead of threads blocking and coordinating on shared state, an event loop reacts to messages with no threads to lock or pool, and the coordination moves into the ordering of events. The questions become what to emit, when, and how the consumers keep up, a different shape from the lock and the pool.

---

This article explains the concurrency practices that keep a multithreaded system steady: avoid sharing, use the narrowest tool, keep critical sections short, and time every wait. It argues that most concurrency bugs come from choosing the wrong level of the stack, and that a lock is the last resort, not the first line.