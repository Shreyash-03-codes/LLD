# Thread Class vs Runnable vs Callable

## Learning Objectives

- State the actual difference between `Thread`, `Runnable`, and `Callable` and why only one is a thread.
- Know `Callable` returns a value and can throw, and exactly what an `ExecutorService` needs for each.
- Explain why extending `Thread` is almost never the answer and what you lose by doing it.

## Introduction

There are three ways a Java program hands work to a thread, and beginners treat them as three interchangeable flavours of the same thing. They are not. `Thread` is a class you would extend, `Runnable` is an interface that returns nothing, and `Callable` is an interface that returns a result and may throw. The difference sounds cosmetic and it is not, it controls whether your work can return a value, fail cleanly, and be handed to a thread pool. Most of a junior's confusion is really a confusion about which one a given API accepts.

## Problem Statement

You need to run a call to a payment service in the background and get its outcome back. You reach for `new Thread(() -> { ... })` because that is the first thing you learned. The thread runs and finishes, and the result sets nowhere, because a `Runnable.run()` returns `void`. To get the result back you invent a mutable wrapper shared between the worker and the caller, which is now a data race you did not plan. Or you want to retry on a checked exception, and `run()` cannot declare any checked exception at all, so you swallow it. Both problems are the same mistake: using the tool whose contract cannot carry either a return value or a failure. The answer is `Callable`, which exists exactly because a real job produces a value and can go wrong.

## Core Concept

The three options describe what shape of work the thread can carry.

| Shape | Return | can throw | How it runs |
|-------|--------|-----------|-------------|
| `Thread` (extend) | via subclassed fields | limited | `start()` the subclass |
| `Runnable` | `void` | unchecked only | wrap in a Thread, or submit to a pool |
| `Callable` | a value | `Exception` | submit to an `ExecutorService` |

Those rows collapse to a sentence that matters: if your unit of work returns something, use `Callable`; if it returns nothing, use `Runnable`; and reserve `Thread` for code that also overrides thread behavior, which is almost never. Java itself split the "here is some work" from "here is a thread" on purpose, because work and a worker are different responsibilities. A `Runnable` is what you run. A `Thread` is who runs it.

```java
public class VoidTask implements Runnable {
    public void run() {
        log("void work done");
    }
}
```

```java
public class ValuedTask implements Callable<String> {
    public String call() throws Exception {
        return payments.charge();   // may return and may fail
    }
}
```

The `Thread` approach, straight to its grave, looks like this and is the one to avoid:

```java
class CounterThread extends Thread {
    private int count;
    public void run() {
        count = 42;
    }
    public int getCount() { return count; }
}
```

Extending `Thread` bakes your work into the actor itself. You can no longer reuse that work on a pool thread, you cannot hand the work to anyone who only wants a `Runnable`, and you inherit `Thread`'s whole surface for no reason. Extending a concrete class is the least flexible of the three and the one this chapter rightly warns against.

### The Executor picks the shape

The framework in the next article wants a specific interface. `ExecutorService.submit(Runnable)` returns a `Future` that resolves to `null` when done, because a `Runnable` has no value. `submit(Callable)` returns a `Future` that carries the computed value and surfaces the thrown `Exception` through `get()`. So the real answer to "how do I collect the result of a background call" is: submit a `Callable` and read the `Future`, and the framework that article is about supplies the `Future`.

### The checked exception is the giveaway

A method that can fail with a checked exception cannot be a `Runnable`, because `run()` is declared to throw nothing. The compiler enforces this and leaves you two ugly exits: catch inside the runnable and swallow, or lift the exception into an unchecked one. A `Callable<T>.call()` declares `throws Exception`, so the failure flows to the `Future` and is thrown when you call `get()`, honest and catchable. When you see a codebase pre-`Callable` futzing with a `Runnable` to smuggle an exception, that is the smell the interface exists to fix.

## Real Production Usage

This is the most real part of the article. Spring `@Async` returns a `Future` or `CompletableFuture`. Kafka consumers run a `Runnable` in a loop on one thread, and produce results through a `KafkaTemplate`. Every `ExecutorService` in java.util.concurrent accepts all three, submit unifying Runnable and Callable under a `Future`. The names the framework authors chose tell you the rule: `execute()` takes `Runnable` only; `submit()` takes both. When you want a value, the API is `Callable` and `Future`, not a shared field.

## Common Mistakes

1. **Extending `Thread` and hiding the result in a field.** You lose reuse and you share a mutable field, which is exactly the race this chapter keeps warning about.
2. **Using `Runnable` for work that needs a result.** The `void` return forces you into shared state. Reach for `Callable`.
3. **Swallowing a checked exception in a `Runnable`.** The failure disappears, and the next layer has no idea. A `Callable` keeps it.

## Interview Perspective

Interviewers ask this to separate "knows the API" from "knows the reasoning behind it." The weak answer lists the three. The strong says "Runnable returns nothing and can't throw, Callable returns a value and can, and I reach for them when I want an ExecutorService to hand me a Future rather than a raw Thread." The payoff moment is when they ask "how do you get a future back" and you say "submit a Callable."

Follow-up: "what's wrong with extending Thread" and "what does the Executor return for a Runnable versus a Callable."

## Knowledge Check

- A background job must send a total to a phone and retry when the API throws. Which of the three carries that and why not the other two?
- Why does a `Future.get()` after you `submit(Runnable)` return `null`, and what does that say about the shape you asked for?
- You wrote `class Worker extends Thread`. Which two capabilities do you lose that either interface gives you back?

## Key Takeaways

- `Runnable` is void and can't throw; `Callable` returns a value and can throw; `Thread` is for extending only as the rare case.
- Reach for `Callable` when the work has a result and submit it to a pool for a `Future`.
- `run()` cannot declare, so a failing unit of work must be a `Callable`.

## What's Next

The interfaces exist to be handed to something, and the next article is the something. The executor framework and thread pools is where your `Callable` stops being a hand-rolled `Thread` and becomes a unit of work the pool schedules for you, which changes how many threads you burn and who waits for what.

---

This article explains the difference between `Thread`, `Runnable`, and `Callable`, and which one carries a value or an exception back. It argues that work with a result must be a `Callable`, that `run()` returns `void` and cannot throw, and that extending `Thread` is the least flexible shape of the three.