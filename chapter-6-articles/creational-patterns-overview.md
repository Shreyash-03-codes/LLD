# Creational Patterns Overview

## Learning Objectives

- Recognize what all creational patterns share: they take the decision of *how an object comes to exist* out of the code that just wants to use it.
- Map each of the five GoF creational patterns to the specific construction problem it solves, so you stop picking patterns by name recognition.
- Spot when a creational pattern is solving a real problem and when it is ceremony hiding an architectural mistake.

## Introduction

Creational patterns are the five patterns in the GoF catalog that deal with object creation: Singleton, Factory Method, Abstract Factory, Builder, and Prototype. That grouping is misleading. They do not share one technique and they are not interchangeable. What they share is a single motivation: the caller should not need to know the details of how an object is assembled, or even which concrete class it gets back.

In Java you create objects with `new`. It is fast, readable, and honest. The entire point of a creational pattern is to put something else in charge of that `new` in specific situations where the direct call is causing you pain. The five patterns are five different answers to five different kinds of pain.

## Problem Statement

Here is the failure mode. You write a payment service that constructs a gateway directly:

```java
public class CheckoutService {
    public void charge(Order order) {
        StripeGateway gateway = new StripeGateway(config.getStripeApiKey());
        gateway.charge(order.getTotal());
    }
}
```

This compiles, it works, and it is wrong in a way that shows up slowly. Every place that wants a `StripeGateway` must know its constructor arguments. When Stripe changes its SDK and the constructor takes a new timeout parameter, you touch every call site. When a second gateway appears, Paypal or Razorpay, you write a second branch somewhere, and then a third, and each branch is copy-pasted. When a test wants to run against a fake gateway, you cannot inject one because the concrete class is hardcoded.

The pain is not that you called `new`. The pain is that object construction is a decision, and decisions scattered across a codebase are decisions that get made inconsistently. A creational pattern centralizes that decision in one place, behind one name, so the rest of the system can depend on a contract instead of on a constructor.

## Core Concept

The five patterns differ along three questions: what are you building, how many variants are there, and how complex is the construction itself.

| Pattern | Problem it solves | What varies | Typical trigger |
|---------|-------------------|-------------|-----------------|
| Singleton | One instance must exist, exactly one, and it must be reachable everywhere | The number of instances | Shared resource, global config, connection pool |
| Factory Method | One object, chosen at runtime, behind a common interface | Which concrete class you get | New subtype added without editing callers |
| Abstract Factory | A family of related objects that must be used together | A whole set of related concrete classes | Themes, environments, product families |
| Builder | One object with many optional or interdependent attributes | How a complex object is assembled | Large constructors, immutable objects |
| Prototype | New objects created by copying an existing one | Construction cost or instantiation complexity | Expensive setup, object templates |

Let me push on that table, because it is where the real understanding lives. Factory Method and Abstract Factory both return objects behind interfaces, and people routinely conflate them. The difference is cardinality. Factory Method makes one decision about one object. Abstract Factory makes a batch decision about several objects that are supposed to stay consistent with each other. You do not reach for Abstract Factory because you have two factories. You reach for it because you have two *families* and you need to guarantee nobody builds a UI component from family A and sticks it into a frame from family B.

Builder and Factory Method also get blurred, and that blur is more damaging because the failure modes differ. A factory hides *which* concrete class you get. A builder hides *how* one concrete class gets assembled. When the class is fixed but the construction is long and error-prone, that is Builder territory. When the class itself is the thing that changes, that is factory territory. Later articles in this chapter hammer both of these separations, so I will not belabor them here.

Prototype is the one people understand least, because in Java it looks unnecessary. Why copy an object when `new` is cheap? The answer is that `new` is only cheap when the constructor does nothing. When constructing an object means loading config files, parsing templates, opening connections, or computing expensive derived state, `new` stops being cheap, and copying an already-built instance becomes the pragmatic move. Java's own runtime relies on this idea heavily, which is why `Object.clone()` exists at all and why it is such a sharp-edged API.

One more thing about the grouping before we move on. The GoF book treats these as a set, but in practice you will almost never use all five in one application, and you will reach for Factory Method and Singleton far more often than the other three. Builder shows up the moment your domain objects are immutable, which makes it extremely common in real codebases. Abstract Factory and Prototype are comparatively rare. That distribution is not a judgement on the patterns. It reflects the shape of typical application code.

### The common thread

Every creational pattern, without exception, is an example of deferring a decision. Without the pattern, the caller decides everything at the call site, at compile time, with its eyes closed. With the pattern, the caller says what it needs in abstract terms, and something else decides the concrete reality. This is the same idea as dependency injection, and it is why creational patterns and DI frameworks have such an intertwined history. Spring's container is, at its core, a very elaborate Abstract Factory. The patterns came first; the frameworks industrialized them.

There is a cost, and the overview would be dishonest without stating it. Every layer of indirection between you and the object makes the code harder to follow. When you open a file and see `Gateway gateway = gatewayFactory.create()`, you cannot tell from that line alone what you got. A junior engineer (and plenty of seniors) will go hunting through factory implementations when a simple `new StripeGateway()` would have been self-evident. The discipline is to add indirection only where the decision genuinely varies. If there is exactly one gateway in your system and you are certain there will be exactly one for the next two years, a factory is not foresight. It is friction.

## Real Production Usage

The JVM ecosystem is saturated with creational patterns, and the most honest place to see them is not the GoF diagrams but the standard library.

- **Factory Method**: `java.util.concurrent.Executors.newFixedThreadPool(...)` returns an `ExecutorService` whose concrete type you never see. `Collections.singletonList(...)`, `List.of(...)`, and the rest of the static factory methods on collections types are Factory Method in daily use.
- **Builder**: `StringBuilder` is the canonical builder, and the fluent builders in Spring (e.g. `UriComponentsBuilder`, `SpringApplicationBuilder`), Guava (`ImmutableMap.builder()`), and the JPA Criteria API are all the same shape.
- **Abstract Factory**: `javax.xml.parsers.DocumentBuilderFactory` is a textbook Abstract Factory. `SAXParserFactory` and `TransformerFactory` too. You ask for a parser factory, you get a concrete factory, and the concrete factory gives you concrete parsers that all play together.
- **Prototype**: harder to spot in the JDK, but the idea is embedded in `Object.clone()` and in frameworks that cache bean definitions. Prototype-scoped beans in Spring are arguably closer to a factory than a true prototype, which is worth knowing because it shows how blurred these lines get in practice.
- **Singleton**: `Runtime.getRuntime()`, `Desktop.getDesktop()`, `Toolkit.getDefaultToolkit()` in Swing. All of these enforce one instance behind a static accessor.

## Common Mistakes

**Treating every creational pattern as a candidate for every object.** Walk a codebase and you will find a factory for a class that has exactly one implementation and one constructor. That factory is not a pattern, it is a tax. Add indirection when you have a demonstrated second variant, not when you predict one.

**Assuming Singleton means "static utility class."** A Singleton is a class that controls its own instantiation so only one instance exists. A class with only static methods is not a Singleton, it has no instances at all. The confusion matters because static utilities are not interchangeable with Singletons when it comes to interface implementation and mocking.

**Reaching for Abstract Factory when one factory would do.** The pattern is for coordinated families, not for "I have two factories." If the objects you create never need to be consistent with each other, Abstract Factory is overhead you did not earn.

## Interview Perspective

Interviewers ask about creational patterns not because they care about the diagrams but because the patterns are a reliable probe for how you think about coupling. A weak answer describes what each pattern does. A strong answer explains what each pattern *changes about the relationship between caller and object*, and can say which problems it introduces rather than only which problems it solves.

The strongest signal is usually the comparison questions. "Factory vs Builder" and "Singleton vs static class" sort candidates fast. Anyone who can say "Builder assembles one known class, Factory decides which unknown class you get" in one sentence has actually used these patterns. Anyone who reaches for the definition off the diagram has memorized them.

Common follow-ups:

- "How do you decide between Factory Method and Abstract Factory?"
- "Why is Singleton controversial, and when would you still use one?"

## Knowledge Check

1. A codebase has one `DatabaseConnection` class with a complex constructor, and a second team now needs connections to a read replica with different settings. Would you introduce a Builder or a Factory, and why? Which one fails to help here?
2. Describe a system where Abstract Factory is the correct pattern and Factory Method genuinely cannot do the job, no matter how you arrange the factories.
3. "Singleton guarantees one instance." Is that a statement about the class, about the JVM, or about the classloader? What breaks the guarantee?

## Key Takeaways

- Creational patterns all defer one decision: how an object comes to exist.
- The five patterns solve five different problems; picking by name instead of by problem is how they get misused.
- Factory Method decides which class. Builder decides how a class is assembled. Abstract Factory decides a whole family.
- Indirection costs readability. Add it only when the construction decision actually varies.
- The JVM standard library is full of these patterns, which is the best place to study them as used, not as drawn.

## What's Next

The next article takes the pattern that most people reach for first and that most people implement wrong: Singleton. We will go through why the naive lazy version is broken under concurrency, how double-checked locking actually works and why `volatile` is the non-negotiable piece, and why the Bill Pugh holder class beats all of it. If you have ever written a synchronized getter "just to be safe," that article is for you.

---

This article explains what the five GoF creational patterns share, why direct `new` becomes a liability once construction decisions vary, and how to match each pattern to the specific construction problem it solves. It argues that creational patterns are deferred decisions, not decoration, and that most misuse comes from adding indirection before a second variant actually exists.
