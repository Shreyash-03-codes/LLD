# Singleton Pattern

## Learning Objectives

- Explain why the naive lazy Singleton is broken under concurrency, in terms of both race conditions and the memory visibility problems that follow.
- Implement a thread-safe Singleton three ways (synchronized accessor, double-checked locking, Bill Pugh holder) and defend the tradeoffs of each.
- Argue about the controversial parts: is the pattern a service locator, a global variable, or a legitimate tool, and what do the anti-pattern critics actually have right.

## Introduction

Singleton is the pattern that guarantees exactly one instance of a class exists and that everyone asking for it gets the same object. It is the first creational pattern most engineers learn, and it is also the one most of the industry implements wrong the first few times. The naive version has a race condition, the "safe" version pays for a lock on every call, and the actually-correct versions rely on either memory visibility subtleties that nobody explains or on a nested-class trick that looks like it should not work.

Before the code, the honest framing: Singleton exists because the shared state it exposes genuinely needs to be shared. Configuration, connection pools, caches, counters. If your object has no shared state and no identity, you probably want a static utility class or an injectable dependency, not a Singleton. If it has shared state but it does not need to be one instance process-wide, you probably want a dependency-injection-managed bean instead. The pattern is narrow. Respecting the narrowness is half of using it well.

## Problem Statement

The classic lazy Singleton fails silently. Consider the straightforward implementation that everyone writes the first time:

```java
public class Config {
    private static Config instance;

    public static Config getInstance() {
        if (instance == null) {
            instance = new Config();
        }
        return instance;
    }
}
```

Under a single thread this is fine. Under concurrency it is a lottery. Two threads call `getInstance()` together. Both read `instance == null` as true, both construct a `Config`, and the last write wins. You now have two instances, which defeats the entire point of the pattern, and no test will reliably catch it because it depends on thread scheduling. Worse, even a thread that reads `instance` after another thread wrote it is not guaranteed to see the fully constructed object. Java's memory model does not promise that the write to `instance` becomes visible to other threads immediately, and it does not promise that a reference seen by another thread points at a fully initialized object. The bug is a one-in-a-thousand crash that gets blamed on something else.

## Core Concept

### The three concerns of a correct Singleton

A correct Singleton has to satisfy three separate requirements, and the different implementations are really different compromises among them:

1. **Lazy vs eager construction.** Build the instance on first use, or build it when the class loads.
2. **Thread safety.** Concurrent callers must never observe two instances or a partially built one.
3. **Cost per call.** Every accessor call pays some price. The question is what that price is when the instance already exists.

The simplest safe fix is to synchronize the accessor:

```java
public class Config {
    private static Config instance;

    public static synchronized Config getInstance() {
        if (instance == null) {
            instance = new Config();
        }
        return instance;
    }
}
```

This is correct. It is also the version that makes engineers reach for alternatives, because `synchronized` on the whole method serializes every accessor call even after the instance exists. If `Config` is read from every request in a high-traffic path, that lock becomes a real cost. The next step is where the subtlety begins.

### Double-checked locking

Double-checked locking reads the field once without the lock, takes the lock only when it looks uninitialized, and re-checks inside the lock:

```java
public class Config {
    private static volatile Config instance;

    public static Config getInstance() {
        if (instance == null) {
            synchronized (Config.class) {
                if (instance == null) {
                    instance = new Config();
                }
            }
        }
        return instance;
    }
}
```

The second check matters because two threads can pass the first check together; only one of them should construct. The `volatile` is not a nicety, it is the whole ballgame. `new Config()` is not a single atomic step. It allocates memory, runs the constructor, and publishes the reference. On many architectures, and under the JMM, other threads can observe the reference before the constructor has finished running. `volatile` gives the write a happens-before relationship with later reads, so any thread that reads a non-null `instance` is guaranteed to see a fully constructed object. Remove `volatile` and double-checked locking is broken in a way that is essentially impossible to reproduce on your laptop and essentially guaranteed to bite in production.

Diagram: double-checked locking control flow

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 640 656" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#444"/>
    </marker>
  </defs>

  <rect x="190" y="30" width="240" height="44" rx="8" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="310" y="58" text-anchor="middle" font-size="14" fill="#1a2733">getInstance() entered</text>

  <polygon points="310,116 390,156 310,196 230,156" fill="#fdf3e3" stroke="#b98a1f" stroke-width="1.5"/>
  <text x="310" y="161" text-anchor="middle" font-size="13" fill="#7a5a10">instance == null?</text>

  <rect x="190" y="256" width="240" height="44" rx="8" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="310" y="284" text-anchor="middle" font-size="14" fill="#1a2733">synchronized (lock)</text>

  <polygon points="310,342 390,382 310,422 230,382" fill="#fdf3e3" stroke="#b98a1f" stroke-width="1.5"/>
  <text x="310" y="387" text-anchor="middle" font-size="13" fill="#7a5a10">instance == null?</text>

  <rect x="190" y="482" width="240" height="44" rx="8" fill="#e9f5ee" stroke="#2e7d4f" stroke-width="1.5"/>
  <text x="310" y="510" text-anchor="middle" font-size="14" fill="#1a2733">instance = new Config()</text>

  <rect x="190" y="586" width="240" height="44" rx="8" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="310" y="614" text-anchor="middle" font-size="14" fill="#1a2733">return instance</text>

  <rect x="470" y="134" width="150" height="44" rx="8" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="545" y="162" text-anchor="middle" font-size="14" fill="#1a2733">return instance</text>

  <rect x="470" y="360" width="150" height="44" rx="8" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="545" y="388" text-anchor="middle" font-size="14" fill="#1a2733">return instance</text>

  <line x1="310" y1="74" x2="310" y2="114" stroke="#444" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="310" y1="196" x2="310" y2="254" stroke="#444" stroke-width="1.5" marker-end="url(#arrow)"/>
  <text x="326" y="232" font-size="12" fill="#7a5a10">yes</text>
  <line x1="390" y1="156" x2="468" y2="156" stroke="#444" stroke-width="1.5" marker-end="url(#arrow)"/>
  <text x="412" y="147" font-size="12" fill="#7a5a10">no</text>
  <line x1="310" y1="300" x2="310" y2="340" stroke="#444" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="310" y1="422" x2="310" y2="480" stroke="#444" stroke-width="1.5" marker-end="url(#arrow)"/>
  <text x="326" y="458" font-size="12" fill="#7a5a10">yes</text>
  <line x1="390" y1="382" x2="468" y2="382" stroke="#444" stroke-width="1.5" marker-end="url(#arrow)"/>
  <text x="412" y="373" font-size="12" fill="#7a5a10">no</text>
  <line x1="310" y1="526" x2="310" y2="584" stroke="#444" stroke-width="1.5" marker-end="url(#arrow)"/>
</svg>
```

The performance win is real but smaller than the diagram suggests. In steady state, most calls hit the first `if` and never touch the lock, which is the whole reason DCL exists. On modern JVMs with biased locking and lock-free fast paths, a `synchronized` accessor is rarely the bottleneck people fear, so DCL is more of a legacy maneuver than a modern necessity. Understand it, but do not assume you need it.

### Bill Pugh holder: the actual best answer

The Bill Pugh initialization-on-demand holder idiom is the cleanest lazy thread-safe Singleton in Java, and it exploits a classloading guarantee instead of locks:

```java
public class Config {
    private Config() {}

    private static final class Holder {
        private static final Config INSTANCE = new Config();
    }

    public static Config getInstance() {
        return Holder.INSTANCE;
    }
}
```

No `synchronized`, no `volatile`, no manual null check. The JVM guarantees that `Holder` is not initialized until `getInstance()` references it for the first time, and that a class's static initialization runs exactly once under an implicit lock. So you get laziness for free, thread safety for free, and a read of a final field on every call, which is the cheapest possible read. There is a reason every serious answer to "how do you write a Singleton in Java" lands here. Use it.

There is one more option worth knowing even though it is rarely the right default: an `enum` with a single constant.

```java
public enum Config {
    INSTANCE;

    private final Properties props;

    Config() {
        this.props = loadProps();
    }

    private static Properties loadProps() {
        Properties p = new Properties();
        // load config from the classpath
        return p;
    }
}
```

The enum gives you the JVM's serialization guarantee for free and is immune to reflection-based construction. Its real problem is that an enum is a poor home for mutable behavior and awkward to inherit or extend, so it shines mostly for genuinely stateless-enough singletons. Fine to know, rarely the tool of choice for real application singletons.

## Real Production Usage

The JVM and the standard library are full of genuine Singletons, which settles the "is this a real pattern" question decisively. `Runtime.getRuntime()` returns one runtime object per process. `Desktop.getDesktop()`, `Toolkit.getDefaultToolkit()`, and `java.util.logging.LogManager.getLogManager()` all enforce single instances behind static accessors.

The more contentious ground is Spring. Spring beans default to singleton scope, but that is a container-managed single instance, not the GoF pattern. The container creates the bean once and hands it to anyone who asks, and that arrangement gives you all the single-instance benefits without any of the pattern's liabilities: the class is a plain class, there is no static state, and tests can construct fresh instances freely. The lesson worth taking is that when you control the wiring, dependency injection is usually a better answer than a hand-rolled Singleton, and the pattern survives mainly in codebases that cannot or will not use a container.

## Common Mistakes

**Making every "global thing" a Singleton.** A logger, a config, a metrics sink, a cache, an executor, a database pool, a feature flag service. Each individually defensible, collectively a global state soup that is impossible to test and impossible to reason about. The pattern does not make globals respectable; it makes them centralized. Prefer passing an instance down or letting a container hand you one.

**Implementing lazy Singleton with the naive null check and shipping it.** This is the single most common production Singleton bug in Java. It works in development, fails under real concurrency, and fails nondeterministically. If you cannot explain why the naive version is broken, you have no business writing a Singleton from scratch.

**Synchronizing everything to avoid thinking.** The fully synchronized accessor is correct, and under low contention it is fine. Reaching for it reflexively because you do not want to reason about `volatile` is acceptable. Reaching for it and then claiming it is fast is not. Say what you are paying for.

## Interview Perspective

Interviewers ask about Singleton for two reasons. First, it is a fast way to see whether you can reason about concurrency and the JMM. Second, it is a fast way to see whether you have an opinion. Candidates who recite the pattern without a stance look like they memorized a book. Candidates who say "I use the Bill Pugh holder, and I reach for it rarely, because Spring already gives me single instances without the global state" demonstrate that they have actually hit this problem at work.

A weak answer: "Singleton ensures one instance, you make the constructor private and return the static instance." A strong answer: "Singleton centralizes shared state into one instance, the hard part is doing that safely under concurrency, I use the holder idiom for laziness and thread safety, and I treat it as a last resort because it couples callers to global state that DI would handle better."

Common follow-ups:

- "What happens to a Singleton in a multi-classloader or distributed deployment?"
- "How does the Bill Pugh holder manage to be both lazy and thread-safe without a lock?"

## Knowledge Check

1. Two threads race through the naive `getInstance()`. Describe exactly how both can construct the instance, and why the fix is not merely "make it volatile."
2. A colleague removes the `volatile` from your DCL Singleton "because modern CPUs are cache coherent." Walk through which guarantee you actually lose and what goes wrong in the JMM, not in the hardware.
3. Why does the Bill Pugh holder never construct `Config` when nothing calls `getInstance()`, even though the class is loaded? What does that say about when static initialization runs?

## Key Takeaways

- The naive lazy Singleton has a race condition, and it is the most commonly shipped Singleton bug in Java.
- A correct Singleton is correct about laziness, thread safety, and call cost, and you cannot have all three with every implementation.
- Double-checked locking requires `volatile`, or it is broken in ways that only show up under load.
- The Bill Pugh holder is the best Java answer: lazy, thread-safe, and lock-free by relying on classloading guarantees.
- Singleton centralizes shared state; prefer DI-managed beans, and treat the hand-rolled pattern as a fallback, not a default.

## What's Next

The next article is Factory Method, which sits on the opposite side of the creational spectrum. Singleton answers "how do I get the one instance," Factory Method answers "which concrete class should the caller get," and getting that distinction clean is where the real skill shows. We will cover the standard shape, why it survives the arrival of a second variant without touching callers, and the common trap of turning it into a glorified switch statement.

---

This article explains what a Singleton must actually guarantee under concurrency and walks through the naive, synchronized, double-checked-locking, and Bill Pugh implementations. Its central claim is that the Bill Pugh holder idiom is the only Java answer that gets laziness, thread safety, and low call cost all at once, and that the pattern itself should be a fallback rather than a default.
