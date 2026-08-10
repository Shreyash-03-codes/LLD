# Structural Patterns Overview

## Learning Objectives

- Understand that the seven structural patterns are all answers to one question: how do existing objects get arranged into a larger structure without making that arrangement fragile.
- Map each pattern to the specific shape problem it solves, so you can identify the pattern from the failure rather than from the diagram.
- See the single axis that separates them: what the wrapper or container *changes* about the objects it touches.

## Introduction

Structural patterns are the seven GoF patterns that deal with object composition: Adapter, Bridge, Composite, Decorator, Facade, Proxy, and Flyweight. The word "structural" is doing real work here. These patterns are not about how objects are built, that was the last chapter. They are not about how objects talk to each other, that is the next one. They are about the static shape of the object graph: what sits next to what, what wraps what, and what hides what.

Creational patterns put something else in charge of `new`. Structural patterns go one step further. They put something else in charge of the *relationships between the objects that already exist*.

## Problem Statement

The failure that all seven patterns attack is the same, and it is the failure of hardcoded structure. Consider a service that talks to a legacy library. The library's class has a method called `enqueueRaw()` that takes completely different arguments from what your service wants to call. Your callers depend on that odd method. Then the library is replaced, and every call site breaks. The structure of the object graph, who depends on whom, is written directly into the code, and there is no seam where the mismatch can be absorbed.

Now widen the picture. That is one mismatch. Real systems have dozens: a UI component that should render either a single widget or a whole panel, a request pipeline where every feature wants to add a step, a domain object that is expensive to load but the client needs it everywhere. If each of these shapes is baked into the callers, the codebase becomes a web of direct dependencies where changing any one shape breaks every consumer. Structural patterns exist to insert a layer at exactly those points, so the shape can change without the consumers noticing.

## Core Concept

The seven patterns are not interchangeable, and the way to keep them straight is to ask what each one *changes* about the objects it sits between.

| Pattern | What it changes | The shape it creates | Typical trigger |
|---------|-----------------|----------------------|-----------------|
| Adapter | The interface of a class | One object translated to look like another interface | A class whose API does not match what callers expect |
| Bridge | The way abstraction and implementation are tied | Two parallel hierarchies joined by a reference | Two dimensions of variation that both want to grow |
| Composite | Whether a client sees one object or a tree of them | A uniform tree of leaves and containers | Part-whole hierarchies that should be treated uniformly |
| Decorator | The behavior of an object | Layers of wrappers, each adding one responsibility | Feature combinations that would explode into subclasses |
| Facade | How much of a subsystem the client sees | One simplified front door over many classes | A subsystem whose internals the client should not navigate |
| Proxy | Who has direct access to the object | A stand-in that controls access or lifecycle | Lazy loading, access control, remote calls |
| Flyweight | How much state is duplicated per object | A shared pool of intrinsic state referenced by many | Many objects sharing the same expensive inner state |

That table is the whole chapter compressed, so it is worth pushing on three of the rows, because they are the ones that get conflated and the conflation produces real design damage.

Adapter, Facade, and Proxy all wrap something, which is why people blur them, but their intents could not be more different. Adapter wraps to *change the interface* so incompatible code can work together. Facade wraps to *simplify* a whole subsystem so the client deals with one method instead of ten classes. Proxy wraps to *control* access to one object: defer its creation, check permission, or reach across a network. Adapter makes broken code work. Facade makes messy code simple. Proxy makes unsafe or expensive code safe. If you remember nothing else from the overview, remember those three verbs.

Bridge and Adapter are the other pair people mix up, because both involve two things that do not naturally fit. Adapter deals with a fixed, already-broken interface mismatch, and it reshapes one side to match the other. Bridge is about a deliberate split made *before* anything is broken: you separate the abstraction from the implementation so both can vary independently. Adapter fixes a wound. Bridge is surgery performed in advance so the wound never happens.

The last useful way to group them is by how much machinery each adds. Composite, Decorator, and Proxy are the workhorses: they show up in almost every serious codebase, and two of them, Composite and Decorator, are the backbone of whole ecosystems, the UI component tree and the Java I/O stack. Facade is cheap and underrated. Flyweight is rare and often wrong. That distribution is the honest state of things, and later articles in this chapter will say exactly why.

### The common thread

Every structural pattern, without exception, inserts a layer of indirection between a client and something else. That is the shared DNA. What differs is what the layer does and what it is allowed to change.

That framing explains the one caution that applies to all seven. Indirection has a cost, and the cost is not abstract. Every wrapper is a class a junior engineer has to find, read, and mentally unwrap before they understand what a call does. `request.getBody()` on a request that has passed through three wrappers is a chain of delegation that is hard to trace. The discipline, exactly as with creational patterns, is to insert the layer where the shape genuinely varies, and to resist the urge to wrap because wrapping looks sophisticated. A facade over a subsystem that never changes is just an extra hop. A proxy around an object that is always cheap to create is wasted machinery. The patterns earn their indirection at the seams where the structure actually bends.

## Real Production Usage

Structural patterns are so embedded in the JDK that you stop noticing them, which is the best evidence they are real. Adapter lives in `InputStreamReader`, which adapts a byte `InputStream` to a character `Reader`, and in `Arrays.asList()`, which adapts an array to a `List`. Composite is `java.awt.Component` and `Container`, where a panel and a button are both components. Decorator is the entire I/O class hierarchy, where `BufferedInputStream` wraps a stream to add buffering and `Collections.synchronizedList()` wraps a list to add thread safety. Proxy is `java.lang.reflect.Proxy` and the lazy-loading stubs in ORMs. Flyweight is the string constant pool and the `Integer.valueOf()` cache. Spring adds its own layer on top: AOP proxies everywhere, transaction proxies, lazy beans. When you understand the patterns, the JDK stops being a list of classes and becomes a catalog of solutions you have already seen work.

## Common Mistakes

**Treating the seven as a menu.** The patterns solve different shape problems, and picking one because you like it, instead of because the failure matches, is how Adapter gets used where Bridge belonged and Proxy gets used where Decorator was correct. Identify the failure first, then the pattern.

**Wrapping without a reason.** Every wrapper costs a layer of indirection. A codebase that decorates, proxies, and facades everything in sight is harder to debug than the code it replaced. The seam must correspond to a real variation point, or it is ceremony.

**Confusing intent with structure.** Adapter, Facade, and Proxy look the same in a diagram and behave completely differently. The structure does not tell you which pattern you need. The intent does, and the intent is what the article will train you to name.

## Interview Perspective

Structural patterns are a fast filter in interviews because the same diagrams are recycled for three different intents. A candidate who draws a wrapper and calls it "Adapter" without checking what it changes has memorized shapes. A candidate who says "I need Adapter because this library's interface does not match what my callers already expect" has applied one.

The question interviewers reach for is the trio: Adapter versus Facade versus Proxy. It sorts people fast, which is why this chapter devotes a full article to it. The follow-up pattern is always the same, pick two patterns that look similar and force a distinction, so the skill is not knowing seven definitions but being able to make three clean comparisons.

Common follow-ups:

- "Adapter and Facade both wrap. Where does one end and the other begin?"
- "Decorator and Proxy both hold a reference to the wrapped object. What is the difference in intent?"

## Knowledge Check

1. A team has a `List` and needs every call to be thread-safe. Which structural pattern is `Collections.synchronizedList()` applying, and what does it change about the list?
2. Adapter, Facade, and Proxy all wrap. Give one real scenario for each where the other two would be the wrong choice.
3. A UI framework lets you compose a panel containing buttons and other panels. Name the pattern, and say what guarantee it gives the code that draws the tree.

## Key Takeaways

- Structural patterns shape the object graph: who wraps what, who hides what, who stands in for what.
- Adapter changes the interface, Facade simplifies a subsystem, Proxy controls access, and confusing those three intents is the most common structural mistake.
- Bridge separates two axes of variation that Adapter can only patch after the damage.
- Composite and Decorator are the workhorses and the backbone of the UI and I/O stacks; Flyweight is rare and easy to misuse.
- Indirection is the shared cost, so every wrapper needs a real variation point to justify itself.

## What's Next

The next article starts with the most common structural problem, the interface that does not fit. Adapter reshapes an existing class so it matches the interface your callers already depend on, and we will cover the object adapter, why the class adapter is mostly dead in Java, and where the JDK itself runs this pattern every day.

---

This article explains the seven structural patterns as different answers to the same question, how existing objects get arranged without breaking their consumers. Its central claim is that the patterns are distinguished by intent, not shape, and that Adapter, Facade, and Proxy are the three intents engineers most often blur.
