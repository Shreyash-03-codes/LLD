# Adapter vs Facade vs Proxy

## Learning Objectives

- Tell the three wrapper patterns apart by intent in one sentence each, and hold that distinction under follow-up pressure.
- Apply a decision procedure to any wrapper in real code and say which of the three it is, or whether it is none of them.
- Recognize that the patterns compose, and that a real class can be an adapter and a facade and a proxy at once without contradicting anything.

## Introduction

Adapter, Facade, and Proxy are the three structural patterns that wrap something, and they are the ones engineers most often confuse, because the diagrams are nearly identical: a box on the left, a box in the middle holding a reference, a box on the right. The structure will not tell you which pattern you are looking at. Only the intent will.

This article exists because the overview promised a clean comparison, and because "which of these is it" is a question that comes up in every code review and every interview where structural patterns appear. If you can answer it on sight, you have learned more from this chapter than the seven diagrams.

## Problem Statement

The confusion is expensive because the three patterns fail in opposite directions. Someone wraps a vendor class and calls it an adapter, but it is really a facade hiding a subsystem, and now the "adapter" is managing five classes instead of translating one interface. Someone wraps a service with a caching layer and calls it a proxy, but it is really a decorator adding behavior, and nobody can say whether the cache is access control or an added feature, so the wrapper's purpose is unnameable and nobody dares touch it.

The failure mode of all this is not the wrapper. It is the unnameable wrapper. When a wrapper's intent is unclear, its contract is unclear, and an unclear contract is how bugs get hidden. The whole point of naming a pattern is that the name is a promise about what the class does. If the name is wrong, the promise is wrong.

## Core Concept

Three verbs, memorized cold, and the confusion evaporates:

| | Adapter | Facade | Proxy |
|---|---|---|---|
| The verb | Change | Simplify | Control |
| What it wraps | One object | A whole subsystem | One object |
| Why it exists | To make incompatible interfaces work | To hide subsystem complexity | To defer, guard, or reach |
| The interface | Translates the target's interface | Presents its own front door | Keeps the subject's interface |
| Client knows about | The wrapped object, by its wrong name | The subsystem's existence, not its internals | The real object, if it exists |
| When it breaks | A new mismatch the adapter cannot translate | The facade starts deciding policy | The proxy leaks identity |

Adapter changes the interface. It sits between the client and an object whose API does not match what the client expects, and it speaks both languages. `InputStreamReader` adapts bytes to characters. The client wanted a `Reader`; the world offered an `InputStream`; the adapter makes them agree.

Facade simplifies a subsystem. It does not change interfaces to make incompatible things fit. The subsystem's classes are fine as they are. The facade gives the client a front door so it does not have to navigate three services and their ordering. `JdbcTemplate` simplifies JDBC. The client wanted a query; the subsystem had `Connection`, `Statement`, `ResultSet`; the facade reduces the navigation to one method.

Proxy controls access. It keeps the subject's interface and stands in front of it to defer its creation, check a permission, or forward a call across a boundary. Spring's transactional proxy controls when a transaction begins and ends. The client wanted the service; the proxy makes sure the transaction surrounds it.

The sentence that holds all three: **Adapter makes broken code work, Facade makes messy code simple, Proxy makes risky code safe.** Say it out loud once and the diagrams stop mattering.

### The decision procedure

Faced with an unknown wrapper, run these in order:

1. Does the wrapper preserve the wrapped interface, or translate it? If it translates, it is an Adapter. Stop.
2. Does the wrapper front more than one class? If it is one front door over several collaborating classes, it is a Facade. Stop.
3. Does the wrapper exist to gate when the real object is created, whether the caller is allowed, or where the call goes? If yes, it is a Proxy.
4. If none of those apply and the wrapper adds behavior while passing the interface through unchanged, it is a Decorator, which is not in this trio but is the fourth option you will meet.

Step 1 is the sharpest cut. Adapter is the only one of the three that changes the interface. Facade and Proxy both present the interface the client already knows, which is why they are the two people really blur. Step 2 then splits them: one front door over many classes is a facade, one stand-in for a single object is a proxy.

### The composition clause

These patterns are not mutually exclusive, and real classes are often two of them at once. An SLF4J logger binding is an adapter, it translates the SLF4J API to a backend's API, and it is part of a facade, because the logger interface is the front door to the whole backend. Spring's `@Transactional` proxy controls access to the transaction, and the method it protects is inside a service that other wrappers decorate. Trying to force a wrapper into exactly one bucket when it legitimately serves two intents is pedantry. What matters is that each intent is named, because each intent is a different contract the reader has to honor.

The naming discipline is the actual skill, because a wrapper's behavior is fully described by its verbs. If a class translates an interface and gates access, saying "adapter proxy" tells the next engineer everything the class will do before they read a line. Saying "proxy" leaves the translation unmentioned and the surprise for later. The patterns here are not categories to sort objects into; they are verbs that describe contracts, and a wrapper is a sentence. The reader should never have to reverse-engineer the sentence from the code.

## Real Production Usage

The JDK draws the lines cleanly, which makes it the best reference. `Arrays.asList()` is pure adapter: it changes the interface of an array to `List`. `JdbcTemplate` is pure facade: it fronts the whole JDBC subsystem. `Integer.valueOf()` is a flyweight-ish factory, but the caching it does is the proxy idea applied to creation, deferring nothing but reusing everything. Reading the JDK with the three verbs in hand is the fastest way to lock them in, because every class there has already made a clean choice.

The frameworks blur them, and the blur is instructive. Spring Security's `SecurityContextHolderAwareRequestWrapper` is a decorator in the servlet wrapper family and a facade over the security context at the same time. Naming both intents is what lets you predict its behavior. When a real framework hands you a wrapper, name the verbs, all of them, and the class stops being mysterious.

Run the procedure on real classes and it stops being an exercise. `InputStreamReader` translates an interface, it is an Adapter. `JdbcTemplate` is one front door over many JDBC classes, it is a Facade. A `@Transactional` proxy preserves the service's interface but controls the transaction, it is a Proxy. `SecurityContextHolderAwareRequestWrapper` adds convenience methods while staying an `HttpServletRequest`, it is a Decorator. Four classes, four one-word answers, zero diagrams required. The test of mastery is doing that on sight, and if you can, the pattern names are doing their job instead of just decorating a slide.

## Common Mistakes

**Naming the wrapper by its structure instead of its intent.** "It wraps, so it is a proxy" is how unnameable wrappers get born. Wrap a class, translate its interface, and call it a proxy, and the next engineer will assume access control that is not there.

**Reaching for Facade when you meant Adapter.** A subsystem that speaks a foreign interface needs an adapter at the boundary, then possibly a facade for the front door. Conflating the two produces an "adapter" that manages multiple classes and an "facade" that translates interfaces, each doing the other's job badly.

**Forcing a wrapper into one bucket when it serves two.** Real wrappers are often adapter-plus-facade or proxy-plus-decorator. Denying the second intent does not remove it, it just makes it unspoken, and unspoken contracts are where the bugs live.

## Interview Perspective

This is the comparison interviewers actually ask, and it is the one that sorts candidates fast, because the difference is all intent. A weak answer lists structural differences and fumbles the question "but they all wrap, so what is the real difference?" A strong answer has the three verbs and the decision procedure, and can classify any wrapper on sight.

The strongest candidates add the honesty: "some real wrappers are two of these at once, and I name both intents." That sentence demonstrates that the patterns were learned from production code, where the lines blur, rather than from diagrams, where they look clean.

Common follow-ups:

- "Draw me a wrapper that is both a Facade and an Adapter, and justify both labels."
- "Your teammate wrapped a class and called it a Proxy, but it never checks permission or defers anything. What do you say?"
- "A vendor SDK ships one class that hides three remote services behind a friendly API. Is that a Facade or an Adapter?"

## Knowledge Check

1. Classify each of these in one word, Adapter, Facade, or Proxy: `InputStreamReader`, `JdbcTemplate`, a `@Transactional` proxy, `SecurityContextHolderAwareRequestWrapper`.
2. A wrapper fronts a single service, changes none of its method signatures, and exists solely to check authorization before forwarding. Which pattern is it, and which verb applies?
3. Explain why two wrappers with identical structure, one holding a `List` and translating its API, the other holding a `List` and guarding its mutation, are different patterns.

## Key Takeaways

- Adapter changes the interface, Facade simplifies a subsystem, Proxy controls access, and those three verbs are the whole comparison.
- The decision procedure runs in order: interface translated, many classes fronted, access controlled.
- Facade and Proxy are the pair people actually blur; step 1, the interface test, is what separates them cleanly.
- Decorator is the fourth wrapper in the wild, and real classes are often two intents at once.
- Name every intent a wrapper serves, because each intent is a contract the next engineer will rely on.

## What's Next

The final article of this chapter takes everything so far to the real world. Spring Security is a structural pattern showroom: the `DelegatingFilterProxy` and `FilterChainProxy` are proxies, the filter chain wraps requests and responses, and the whole servlet pipeline is a chain of wrappers. We will trace a request from the container through the security filters to your controller, and name every pattern it passes through.

---

This article explains how to tell Adapter, Facade, and Proxy apart with three verbs, change, simplify, and control, instead of memorized diagrams. It argues that naming a wrapper's intent is the entire skill, and that real wrappers often serve two intents at once, each of which must be named.
