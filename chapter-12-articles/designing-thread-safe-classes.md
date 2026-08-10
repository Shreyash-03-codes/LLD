# Designing Thread-Safe Classes

## Learning Objectives

- Explain the three ways to make a class safe to share across threads.
- Know when a class should be immutable, confined, or internally synchronized.
- Audit a small class for a shared mutable field that the design did not cover.

## Introduction

Every lock and primitive so far exists so a class can be shared. The design question is where the safety lives. A class is thread-safe when its internal state stays consistent no matter which threads call it. There are three honest answers to where that safety lives: the state never changes, the state is confined to one thread, or the state is guarded by a lock inside the class. Pick one deliberately, because a class that half-fits none of the three is a race you have not met yet.

## Problem Statement

A `UserSession` class keeps a `Map` and a `lastSeen` timestamp. One thread updates the map while a reporting thread reads it, and the report shows a half-updated account or a null. The bug is not in either method; it is in the assumption that a class of two mutable fields is safe because "nobody writes a lock." To fix a class you name the safety strategy first. Thread safety is an interface-level contract, not a happy accident of the fields being private.

## Core Concept

The three strategies in one line: immutable, thread-confined, or guarded by a lock.

**Immutable.** All fields are `final`, and the internal state is built once. Sharing becomes free, because no thread can see a different value. The read path never locks and is always correct. The caveat is that a class that must change cannot be immutable, so you re-build the value and share the new one.

```java
final class Price {
    private final double amount;
    Price(double amount) { this.amount = amount; }
    double value() { return amount; }
}
```

**Thread-confined.** The state is private and only ever touched by one thread, often the one that owns it. A `ThreadLocal` or an explicitly owned executor are the markers. The safety holds as long as the object never escapes to another thread, and the class is only as safe as the discipline that keeps the value local.

**Guarded by a lock.** The state is mutable and shared, so every access path, the reads included, goes through the same lock or the same concurrency primitive. A `synchronized` map, a `ReentrantLock`, or an atomic field are the guards. The classic failure is guarding only the writes and leaving a read unlocked.

The audit for a new class: find every mutable field, ask which strategy guards it, and if the answer is "none," the design is unfinished. A useful extra rule is to hold the lock for the smallest region that covers both the check and the update, so a read-then-write is not split.

## Real Production Usage

Real classes mostly live on one side of the line. Value objects are immutable by contract, a service worker's scratch state is confined to the thread, and a counter or a cache is guarded, usually with a concurrent collection that does the locking. The famous pattern in the JDK is `String` and the primitive wrappers, immutable by design, while `ConcurrentHashMap` and `AtomicInteger` are the guarded end. The honest note is that production code rarely rolls its own lock, but it names the strategy at every class boundary, so the next maintainer knows which guarantees hold.

## Common Mistakes

1. **Half-immutable.** Making the fields final but returning a reference to a mutable inner map, so the state changes through the getter.
2. **Confined in theory, shared in practice.** A `ThreadLocal` handed to another thread and the confinement silently gone.
3. **Guarding writes only.** A getter with no lock reads a field while the writer moves it.
4. **Check then act outside the guard.** Reading a flag, then acting without the lock, a race in the gap.
5. **Mutating a "immutable" value in place.** A class with `final` fields and a method that changes them breaks the contract.

## Interview Perspective

The prompt is "how do you make a class thread-safe" and the interviewer listens for the three strategies. The strong answer names immutable, confinement, and locking, then says which one the class uses and why. Follow-up: "a getter that returns the list" and "what if two threads must both change a counter." The strong answer: return a copy or an unmodifiable view, and use an atomic or a lock for the counter. The line that closes is naming the strategy per class, because thread safety is a contract, not a property.

## Knowledge Check

- A class has `final` fields but returns a mutable inner map from a getter. Which strategy does it actually violate?
- What makes a `ThreadLocal` unsafe the moment it escapes to another thread?
- Why does guarding only the writes leave a read still racy?

## Key Takeaways

- Pick the strategy first: immutable, thread-confined, or guarded by a lock.
- Every mutable field must have a named guard, reads included.
- Audit a class for the shared mutable field before trusting it, and prefer an existing concurrent type to a hand-rolled lock.

## What's Next

Safe classes one by one still do not make a system that scales cleanly. The last article in this chapter is concurrency design best practices, pulling the tools together into the habits that keep a whole service steady under load.

---

This article explains that a thread-safe class is a contract backed by one of three strategies: immutable state, thread-confined state, or state guarded by a lock. It argues that the design must name the strategy before the code, so a shared mutable field without a guard is a known bug and not a surprise.