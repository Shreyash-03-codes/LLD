# Future and CompletableFuture for Async Design

## Learning Objectives

- Explain what a `Future` is and why a bare `get()` blocks the calling thread.
- Build a chain of work with `CompletableFuture` instead of nesting callbacks.
- Run independent async calls and combine them so the latency is the slowest, not the sum.

## Introduction

A `Runnable` runs and returns nothing. A `Callable` returns a value, but the value is produced on another thread, so it comes back wrapped in a `Future`, a handle that fills in later. The plain form has one sharp edge: `future.get()` blocks the calling thread until the value is there, so you give back the wall-clock gain the moment you call it. `CompletableFuture` keeps the promise and adds verbs that say what to do with the value when it arrives, without the calling thread sitting and waiting.

## Problem Statement

A report endpoint needs three numbers, one from each of three services. The naive version calls them one at a time: a ninety-millisecond call, then another, then a third, for a report that takes over two hundred milliseconds even though the three calls could run at once. You can run them on a pool and collect the futures, but the `get` and the join all block at the sync point, so the report thread still waits for the slowest, one after another, and the three parallel reads become serial again. A bare `Future` runs the call but blocks when you read it; `CompletableFuture` turns that wait into a chain of callbacks.

## Core Concept

A `Future<T>` is a handle to a `T` that may or may not be present yet. The pool returns future, and `Callable` fills them in.

```java
Future<Integer> f = pool.submit(() -> compute());
Integer v = f.get();  // blocks until ready
```

The distinguishing move is `CompletableFuture` also being a `Future`, but adding combinators. The consumer says what should happen when the value arrives and the work proceeds as a callback, so no thread sits at a sync point.

```java
CompletableFuture.supplyAsync(() -> readOrders())
    .thenApply(orders -> orders.size())
    .thenAccept(n -> log("orders={}", n));
```

The active parts are the supply and the combinators. `thenApply` maps a value, `thenCompose` chains a stage that itself returns a future and flattens it, `thenCombine` joins two futures, and `exceptionally` catches a failure. None of these block the calling thread; the pool carries the work, and the chain continues as pieces complete.

The one habit to break is calling `get` or `join` on a handler thread. It parks the worker pool for the duration of the call, which is exactly the threading gain thrown away. The design goal is to hand the chain the reply and never wait for it yourself.

## Real Production Usage

Web requests run their async work off a service and onto a bounded task pool, and the handler returns the chain rather than blocking on it. For the three-service report, the correct pattern is to launch the three reads together and `thenCombine` the results, so the report returns when the slowest call does, not after the sum of all three. `CompletableFuture` is what sits behind Spring's `@Async` and a good deal of web client code. The honest note is that most of your time is `supplyAsync`, `thenApply`, `thenCompose`, and `exceptionally`; the rest is rarely the happy path.

## Common Mistakes

1. **Blocking in the handler.** Calling `get` or `join` on a request thread and giving back the win you launched the async for.
2. **Letting a failure go untouched.** A `CompletableFuture` that fails and is never observed loses the error quietly.
3. **Nesting a future inside a `thenApply`.** A stage that returns a future needs a `thenCompose` to flatten it, not a `thenApply`.
4. **Sharing one pool for everything.** A long `get` in one chain can crowd the happy path for every other chain on the common pool.
5. **Not setting a timeout.** By default a future that never resolves waits forever and the request hangs instead of failing.

## Interview Perspective

The question is "Future versus CompletableFuture," a probe for whether a candidate understands blocking. Strong: "a plain `Future` calls the call and `get()` blocks the thread; `CompletableFuture` composes callbacks so no thread waits, and the chain continues when the value lands." Follow-ups are "how do you fan out three calls" and "how does `thenCompose` differ from `thenApply`, and the answer must name flattening." A good closing line: parallel waits end up on the slowest call, not the sum, so compose, don't hold.

## Knowledge Check

- What does `future.get()` do to the calling thread, and when does that undo the benefit of the future?
- Why does a stage that returns a future need `thenCompose` instead of `thenApply`?
- What happens to a `CompletableFuture` whose exception no one observes?

## Key Takeaways

- A `Future` holds a value whose `get()` blocks; a `CompletableFuture` describes the flow so nothing waits.
- Use `thenApply` to map, `thenCompose` to flatten nested futures, and `exceptionally` for the failure path.
- Run the independent futures together and combine, so the latency is the slowest, not the sum.
- Do not call `get` or `join` on a request thread; hand off the chain.

## What's Next

The futures and the chains talk about values arriving, but between two threads there is a lower-level handshake. The last low-level tools are `wait`, `notify`, and `notifyAll`, a signal where a thread parks itself and a partner wakes it.

---

This article explains `Future` as a handle for a value that completes later and `CompletableFuture` as the same idea with callbacks so no thread blocks on `get`. It argues that the gain comes from composing stages rather than holding, and that a timeout and an exception path are part of the design, not an afterthought.