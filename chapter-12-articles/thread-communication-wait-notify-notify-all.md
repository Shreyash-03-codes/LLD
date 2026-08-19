# Thread Communication: wait, Notify, notifyAll

## Learning Objectives

- Explain what `wait` and `notify` handshake between two threads and what role the monitor plays.
- Write a correct wait that checks a condition with `while` and wakes with `notifyAll`.
- Know why this low-level tool sits below a `BlockingQueue` and when to use the higher one.

## Introduction

Locks decide who runs. `wait` and `notify` decide when a thread knows the wait is over. A producer fills a buffer; a consumer should stay parked until there is something to take. `Object.wait()` parks a thread and releases the monitor, and `notify` or `notifyAll` wakes parked threads. This is one of the least directly written threading primitives because it is sharp, but it is the base that every higher coordination tool is built on.

## Problem Statement

A consumer thread needs a value from a producer. The cheapest wrong way is to spin, looping on a flag and burning CPU while it checks the same boolean. The correct way is to park: the consumer calls `wait()` and the producer calls `notifyAll()`, so the consumer wakes only when there is real work. The trap in the middle is the missed and the spurious wake-up: the producer signals just before the consumer reaches `wait`, the signal is lost, and the consumer sleeps with no one to wake it.

## Core Concept

`wait` is the mirror image of the lock. A thread must hold the monitor to call `wait()`, and `wait()` then releases it and parks, so the producer can acquire it. On waking, the thread must reacquire the lock. `notify()` wakes one waiter at random; `notifyAll()` wakes every waiter. Because the method requires the monitor, the handshake always sits inside a `synchronized` block.

```java
synchronized (buffer) {
    while (buffer.isEmpty()) {
        buffer.wait();
    }
    return buffer.take();
}
```

The check is a `while`, not an `if`. Between the thread being woken and reacquiring the lock, another consumer can grab the one item, so a waiter must re-check the condition and wait again if it is still false. The producer pairs each fill with a signal under the same lock:

```java
synchronized (buffer) {
    buffer.add(item);
    buffer.notifyAll();
}
```

Two details tie it together. `wait()` releases the monitor while it parks and reacquires it on wake, which is what lets the producer's `notifyAll` into the same section. And `notifyAll` over `notify` is the safe default: with several waiters on a different condition, `notify` can wake the wrong one and leave the invariant, while `notifyAll` lets each waiter re-check its own condition and loop.

## Real Production Usage

Wait and notify are the block that a `BlockingQueue` hides. A `take()` is the wait and a `put()` is the notify, and the JDK wraps all of this precisely. Production code rarely writes the handshake by hand, because the brittle parts named above are exactly what the higher abstractions absorb, and a `ReentrantLock` `Condition` gives you named, separate wait sets when you need more than one. The low-level pair stays as the clock and the primitive, but a service thread rarely sits on it today. The lesson is to know the handshake and then use the layer above.

## Common Mistakes

1. **An `if` where the loop and the recheck should be.** A single `if` allows returning on a stale wake and taking from an empty buffer.
2. **A missed notify.** The producer signals before the consumer begins to wait, the signal is lost, and the consumer sleeps forever.
3. **Signal and consume on different monitors.** The wait and the notify touch two locks, so the wake never lands.
4. **Calling `wait` outside the lock.** It throws `IllegalMonitorStateException` unless the thread owns the monitor.
5. **Doing slow work under the `wait`.** Over-holding the monitor, so produces queue behind the wait that should race.

## Interview Perspective

The question is "why is the wait in a while, not an if," because it sifts people who have written the handshake. The strong answer: after a wake the thread reacquires the lock and re-checks the condition, because another consumer may have already taken the value. Follow-up: "`notify` vs `notifyAll`" and "what is lost after the signal." A good close: `notifyAll` is the default, the re-check is always a loop, and the `BlockingQueue` is what a build uses instead.

## Knowledge Check

- Why must a wait be a `while` re-check and not a single `if`?
- What exception does a `wait()` call throw if the thread does not hold the monitor?
- Why is `notifyAll` safer than `notify` when several threads wait on related conditions?

## Key Takeaways

- A `wait` thread parks on the monitor and must own the lock to call it.
- Every wait is a `while` re-check, and the preferred wake is `notifyAll`.
- The handshake is brittle, so a `BlockingQueue` or a lock `Condition` usually beats the low-level pair.

## What's Next

The wait can strand a thread when two waiters each need what the other holds. The next article is the problems two threads press into: deadlock, livelock, and starvation, and the ordering patterns that avoid them.

---

This article explains `wait`, `notify`, and `notifyAll` as a low-level handshake where a thread parks on a monitor and a producer wakes it. It argues that correctness lives in the loop and the `notifyAll`, and that these fragile pieces are why a `BlockingQueue` and a latch hide the same chore behind a safer face.