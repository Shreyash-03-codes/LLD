# Common Concurrency Problems: Deadlock, Livelock, and Starvation

## Learning Objectives

- Explain the four necessary conditions for a deadlock and how to spot one.
- Tell a deadlock, a livelock, and a starvation apart by what each thread is doing.
- Name the simple habits that prevent a deadlock from forming in code.

## Introduction

The last few articles gave you the tools to coordinate. This one covers what happens when two threads coordinate each other into a corner. A deadlock parks both threads forever, each holding a lock the other wants. A livelock keeps them moving but never reaching the exit. Starvation quietly leaves one thread out while others run. These are not the crash bugs that scream; they are the hangs and the stalls that only show under load and that interviews love to test.

## Problem Statement

Two threads each need two locks. Thread A takes the lock on resource X then wants Y; Thread B takes Y then wants X. Both hold their first lock and wait for the second, and since neither will ever release, the pair is stuck for good. The process does not crash, the threads just idle, and on a server the whole pool can choke on one such pair. The other two failures, the threads spin or a low-priority thread never gets the CPU, look different but share the same theme of a wait that never ends.

## Core Concept

A deadlock needs four conditions at once, and a fix removes one of them.

- Mutual exclusion: the shared resource is held exclusively.
- Hold and wait: a thread holds what it has while asking for the rest.
- No preemption: a thread cannot be forced to release.
- Circular wait: A waits for B, B waits for A.

Break any one and the cycle is gone. The most buildable break is to avoid the circular wait by locking in a strict consistent order. If every thread acquires the two locks in the same order, the cycle cannot form, because nobody is waiting on the resource held by the lock taken later.

```java
synchronized (lockA) {
    synchronized (lockB) {
        // doWork
    }
}
```

A livelock is the same two threads but still moving: each changes its state while none of them makes progress. Two threads each try the same lock with `tryLock`, both give up at the same time, both retry, and repeat; each is working but the work is nothing. The tell is that the CPU is busy fighting.

Starvation is a thread that is waiting but the wait never ends because others keep overtaking it. A `ReentrantLock` with the default fairness lets a constant, small task barge ahead of a waiting writer. The fix is usually to acquire fairness, a fair lock that serves in the arrival order, or to break up the task so the slow caller fits.

## Real Production Usage

In real services a deadlock shows up as a thread dump where a pool spins and a request hangs, and the fix is a thread dump read, not a new lock. The classic production habits: lock in a consistent order, never hold one lock and call out to code that takes another, keep the critical sections short, and always release in a `finally`. A dump tool can spot the cycle, but the lasting skill is a consistent lock order. The honest note is that most deadlocks are not exotic; they are two nested locks applied in opposite directions in two places of the codebase.

## Common Mistakes

1. **Locking in opposite orders.** The exact circular wait in a different file, the hardest to spot by eye.
2. **Holding a lock across an I/O call.** A slow network read held inside a lock and the arriving thread waits on the edge.
3. **Using `tryLock` in a same cadence.** Every thread gives up and retries at the same beat, a livelock busy-wait.
4. **Blind `timeout` as the answer.** A `tryLock` timeout covers the symptom, not the order mistake.
5. **Ignoring the fairness flag.** Many readers barge ahead and a writer starves.

## Interview Perspective

The deadlock question is a classic because the answer needs a single consistent lock order. The strong answer names the four conditions, breaks one, and jumps the rest. Follow-up: "what is a deadlock versus a livelock" and "how do you detect one in a running service." The strong answer: a thread dump read and the waits are seen. The habit that closes the interview is locking in one collected order and never opening a second lock while one is held.

## Knowledge Check

- Name the four conditions that must all hold for a deadlock.
- Two threads retry a lock at the same cadence and never finish. Deadlock or livelock, and why?
- What single habit, applied everywhere, breaks a circular wait?

## Key Takeaways

- A deadlock is four conditions together: exclusive, hold-and-wait, no preemption, circular wait.
- Break the cycle with a consistent lock order; livelock and starvation are waiting, not blocked, and call their own fix.
- A thread dump is the detector for an interview and for a running service.

## What's Next

The wrong patterns are clear now, so the productive way is to build code that cannot have them. The next article is designing thread-safe classes, the layout that keeps a type safe, immutable, or narrowly locked by construction.

---

This article explains the three thread problems: deadlock as a circular wait on held locks, livelock as threads that keep moving without progress, and starvation as a wait that never ends. It argues that a deadlock breaks with a consistent lock order, and that the other two need fairness and a structure that stops the barge.