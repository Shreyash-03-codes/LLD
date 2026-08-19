# The Executor Framework and Thread Pools

## Learning Objectives

- Explain what the executor framework hides, a mechanism that runs work on a thread without you building a thread for each job.
- Read a `ThreadPoolExecutor` and say what `corePoolSize`, `maxPoolSize`, and the queue actually control.
- Pick a pool type for a workload and name the two failure modes a pool has.

## Introduction

The executor framework is the answer to "where should my work run?" A thread is expensive to spin up and hand off, so spawned work should run on a pool of existing threads rather than a fresh one each time. The pool is a fixed set of worker threads that pull tasks, and a queue that lets work wait when all of them are busy. You, the caller, only hand over a `Runnable` or `Callable`; the pool decides which thread picks it up.

## Problem Statement

You launch one background thread per e-mail. On a quiet day it is two threads, fine. On a spike it is nine hundred. Each thread holds a stack and a handful of resources, so the JVM burns, and then the machine stalls under a pile of ten thousand threads that are mostly idle or blocked. Every burst re-creates the same threads, and throws them away when it ends. The thread-per-task model is the whole bankruptcy: threads are born and die again for work a fixed set of workers could serve. The executor fixes it with reuse.

## Core Concept

The heart is `ThreadPoolExecutor`, and you build one with three dials plus a queue.

- `corePoolSize`: the number of threads kept alive even when idle.
- `maxPoolSize`: the most threads allowed at once.
- `workQueue`: where submitted work sits when every thread is busy.

There is one fact that unlocks reading a `ThreadPoolExecutor`: the interplay of the queue. If the queue is unbounded, the executor never goes above `corePoolSize` no matter how high you set the max, because extra threads only appear when the queue is full and it never is. That is why a `LinkedBlockingQueue` with a huge max is still a fixed pool, and a `SynchronousQueue`, which does not buffer at all, is why the pool goes straight to `maxPoolSize` when the threads are busy. Read those two together.

```java
ExecutorService pool = new ThreadPoolExecutor(
        4,          // core
        8,          // max
        30, TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(100),
        new ThreadPoolExecutor.CallerRunsPolicy()   // policy when rejected
);
```

When a thread is idle for longer than the keep-alive, the pool shrinks back toward core. When the queue and max are all exhausted, a rejection policy runs, and the policy is a decision, not a default. `AbortPolicy` throws, `CallerRunsPolicy` runs the task on the submitting thread, and `DiscardPolicy` throws it away.

Diagram: a task entering a pool, waiting in the queue, then running on a worker.

<svg width="860" height="320" viewBox="0 0 860 320" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="arrow" markerWidth="10" markerHeight="10" refX="8" refY="4" orient="auto">
      <path d="M0,0 L8,4 L0,8 Z" fill="#222"/>
    </marker>
  </defs>

  <rect x="40" y="130" width="120" height="60" rx="8" fill="#eef2f7" stroke="#333" stroke-width="2"/>
  <text x="100" y="158" text-anchor="middle" font-family="Arial" font-size="14" fill="#222">Tasks</text>
  <text x="100" y="178" text-anchor="middle" font-family="Arial" font-size="12" fill="#555">submit many</text>

  <path d="M 222 160 L 366 160" stroke="#222" stroke-width="2" fill="none" marker-end="url(#arrow)"/>
  <text x="294" y="150" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">hand to pool</text>

  <rect x="370" y="130" width="170" height="60" rx="8" fill="#fdf6e3" stroke="#8a6d00" stroke-width="2"/>
  <text x="455" y="158" text-anchor="middle" font-family="Arial" font-size="14" fill="#222">Blocking queue</text>
  <text x="455" y="178" text-anchor="middle" font-family="Arial" font-size="12" fill="#555">pending tasks</text>

  <path d="M 540 160 C 560 160 560 160 606 160" stroke="#222" stroke-width="2" fill="none" marker-end="url(#arrow)"/>
  <text x="572" y="150" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">worker takes</text>

  <rect x="40" y="250" width="170" height="50" rx="6" fill="#f8dede" stroke="#962828" stroke-width="2"/>
  <text x="105" y="268" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Full, reject</text>
  <text x="105" y="288" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">policy runs</text>

  <path d="M 455 192 L 455 224 L 225 224 L 212 246" stroke="#8a6d00" stroke-width="2" fill="none" marker-end="url(#arrow)"/>
  <text x="306" y="214" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">queue full, pool maximum</text>

  <rect x="230" y="250" width="170" height="50" rx="6" fill="#eef7ee" stroke="#2f6f3e" stroke-width="2"/>
  <text x="315" y="278" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Worker 1</text>
  <rect x="420" y="250" width="170" height="50" rx="6" fill="#eef7ee" stroke="#2f6f3e" stroke-width="2"/>
  <text x="505" y="278" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Worker 2</text>
  <rect x="610" y="250" width="170" height="50" rx="6" fill="#eef7ee" stroke="#2f6f3e" stroke-width="2"/>
  <text x="695" y="278" text-anchor="middle" font-family="Arial" font-size="13" fill="#222">Worker 3</text>

  <path d="M 540 275 L 606 275" stroke="#222" stroke-width="2" fill="none" marker-end="url(#arrow)"/>
</svg>

### Picking a pool by task shape

The two common `Executors` tools are `newFixedThreadPool(n)` and `newCachedThreadPool()`. The fixed pool keeps n and a queue, right for steady work. The cached pool grows without bound as overload grows, which is exactly the "thread per spike" hazard that set this article up, so it is reserved for the short, rare, and not a default you want under a real flood. The `newScheduledThreadPool` covers the delayed and periodic. The real question is task shape: if the work is CPU-bound, a fixed pool sized near the core count wins, and adding threads past cores only causes context switches.

### Shutting a pool down

Applications that do not shut down leak executor threads. `shutdown()` stops accepting new work and lets in-flight finish; `shutdownNow()` interrupts the running. Not calling either leaves the pool threads alive, and a batch program that forgot the shutdown simply hangs after its last task. Web servers keep pools alive by design, but a batch or CLI must call it.

## Real Production Usage

Spring's `@Async` calls run through a shared executor, Tomcat serves requests with a fixed thread pool, and `CompletableFuture` pipelines share `commonPool()`, a fixed pool tuned to the core count. The real signal is that production work of this shape always flows through a named pool, never a bare `new Thread`. Routing a task through a pool earns throttling, a capped worker count, and a clean shutdown point, all of which a raw thread gives up.

## Common Mistakes

1. **A queue it will never use.** Setting `maxPoolSize` high but handing it a `LinkedBlockingQueue` and expecting growth, which never comes because the queue never fills and workers stay at core.
2. **A CPU-bound pool in "more" mode.** Oversizing a CPU pool means more context switches than progress.
3. **Forgetting the shutdown.** Pool threads stay alive and a long-read batch program just hangs at the end.

## Interview Perspective

Interviewers ask "fixed versus cached" to see pool reasoning, not recall. Weak: "cached grows". The strong answer: "fixed keeps n workers and a queue and stays stable; cached grows with load, so I use it only for short tasks and not as a default under a spike." They often follow up on "an unbounded queue, what happens to the max" and "when does the pool reject a task," which are both answered by reading the queue and the pool size together.

Follow-up: "what does shutdownNow do" and "which pool for a web gateway."

## Knowledge Check

- You set `maxPoolSize` high with a `LinkedBlockingQueue`. Say why the pool never grows.
- Which queue type makes the pool reach its maximum, and what does that cost the producer?
- What happens to a submitted task when the queue and the pool are both full?

## Key Takeaways

- Executors decouple the work from the thread that runs it.
- `corePoolSize` plus a queue that never fills means you stay at core, and `SynchronousQueue` is the only easy way to reach max.
- Fixed pool for steady, cached pool only for the transient, and shut it down.

## What's Next

The pool gives you several threads calling one piece of code, which is exactly when threads start interfering. The next article is synchronization and locks, the tool that makes two threads coordinate on a shared object instead of clobbering it.

---

This article explains the executor framework and how a thread pool turns a flood of tasks into a bounded set of workers and a waiting queue. It argues that the queue is the control that decides whether a pool ever grows, and that the size of a CPU pool is how many cores you have been willing to use.