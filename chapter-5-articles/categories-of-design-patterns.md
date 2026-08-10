# Categories of Design Patterns

## Learning Objectives

1. Sort the twenty-three Gang of Four patterns into creational, structural, and behavioral, and state the question each family answers.
2. Read a pattern's name and place it in its family, so you know what problem it is going to attack before you read its description.
3. Keep the three confusion pairs apart, Adapter and Facade and Proxy, Strategy and State, Factory Method and Abstract Factory, by the problem each one answers.

## Introduction

The twenty-three patterns are not a flat list. The Gang of Four grouped them into three families, and the grouping is the fastest way to hold the catalog in your head. Creational patterns answer how objects are made. Structural patterns answer how classes and objects are arranged. Behavioral patterns answer how objects talk to each other and split the work. That is the whole map, and once it is in your head, a pattern's family tells you what it is attacking before you read a word of its description.

The families also tell you where the pain is coming from. If the trouble is "this code constructs the wrong concrete class all over the place," the fix lives in the creational family. If the trouble is "the class is tangled with a third-party API," the fix lives in the structural family. If the trouble is "objects can't cooperate without knowing too much about each other," the fix lives in the behavioral family.

## Problem Statement

An engineer without the map treats the catalog as a flat pile of names and memorizes them as unrelated facts. Adapter, Command, Singleton, Bridge, Memento, they all blur, and the blur is not a memory failure, it is a missing structure. Memorizing twenty-three disconnected names is impossible, and memorizing three families with their questions is not.

The blur has a real cost in a design discussion. Someone says "we need a Facade," and the engineer who cannot place Facade in the structural family cannot guess what it does, "a narrow door in front of a subsystem," and cannot say whether it fits. Someone says "this is a behavioral problem," and the engineer without the families cannot orient, cannot narrow the catalog to the eleven behavioral candidates, cannot start the selection. The categories are the index, and a catalog without an index is a pile.

## Core Concept

The creational family answers one question: how does an object get created, and who decides which concrete class? The point is decoupling the caller from the construction. If code says `new StripeGateway()`, that code is welded to `StripeGateway`, and every test and every future provider pays for the weld. The creational patterns move the decision elsewhere, behind a factory, a builder, or a method override.

The five creational patterns are Factory Method, Abstract Factory, Builder, Prototype, and Singleton. Factory Method lets a subclass decide the class of the object a method returns. Abstract Factory groups a family of related objects, a `UiFactory` that makes buttons and menus that match, so a theme stays consistent. Builder separates the construction of a complex object from its representation, so the same construction steps make different representations. Prototype creates new objects by cloning a prototype instead of by calling constructors. Singleton, the famous one, guarantees one instance and exposes it globally, and it is the pattern this handbook will tell you to avoid in most code.

The structural family answers a different question: how do classes and objects compose into larger structures? The point is assembling pieces without entangling them. When two pieces have interfaces that do not match, or one piece is too big to use directly, the structural patterns are the glue that keeps them separate.

The seven structural patterns are Adapter, Bridge, Composite, Decorator, Facade, Flyweight, and Proxy. Adapter translates one interface into another so a client can use a class it was not written for. Bridge splits an abstraction from its implementation so both can vary independently. Composite lets a client treat a tree of objects, a folder of files, as if each object were a single leaf. Decorator adds behavior to an object by wrapping it, so responsibilities can stack without subclassing. Facade gives a subsystem one simple door. Flyweight shares the common part of many fine-grained objects, the glyph in a text editor, so memory stays low. Proxy stands in for another object, controlling access, laziness, or interception, and it is the pattern frameworks use to wrap your beans.

The behavioral family answers the question that dominates real code: how do objects divide responsibility and communicate? The point is letting objects cooperate without knowing each other's internals. When the problem is "this object does too much and knows too much," the answer is usually behavioral.

The eleven behavioral patterns are Chain of Responsibility, Command, Interpreter, Iterator, Mediator, Memento, Observer, State, Strategy, Template Method, and Visitor. Chain of Responsibility passes a request along a chain until one handler takes it. Command turns a request into an object, so it can be queued, logged, or undone. Interpreter defines a grammar and evaluates sentences of that grammar, the calculator that parses an expression and returns a value. Iterator gives sequential access to a collection without exposing its structure. Mediator centralizes the communication between many objects so they do not reference each other. Memento captures and restores an object's state. Observer lets many objects react to an event without the source knowing them. State changes an object's behavior when its state changes. Strategy swaps an algorithm at runtime. Template Method fixes an algorithm's skeleton and lets subclasses supply steps. Visitor adds operations to a class hierarchy without changing the classes.

The full map, with the count and the single idea each family stands for:

| Family | Question it answers | Patterns | Java-native example |
| --- | --- | --- | --- |
| Creational | Who creates the object, and which concrete class? | Factory Method, Abstract Factory, Builder, Prototype, Singleton | `List.of(...)`, `new StringBuilder(...)` |
| Structural | How do pieces compose without entangling? | Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy | `BufferedInputStream`, `Collections.unmodifiableList` |
| Behavioral | How do objects split work and communicate? | Chain of Responsibility, Command, Interpreter, Iterator, Mediator, Memento, Observer, State, Strategy, Template Method, Visitor | `Comparator`, `Runnable`, `Stream` pipeline |

The three families are visible in the JDK without reading a pattern book, and it is worth seeing them in code because the families map to real method calls.

```
// Creational: the factory family, absorbed into java.util
List<String> tags = List.of("urgent", "archived");
Set<String> unique = Set.copyOf(tags);

// Behavioral: Strategy as a lambda, the algorithm injected at the call site
list.sort(Comparator.comparing(Order::getTotal));

// Behavioral: Command as a unit of work, queued for the executor
executor.submit(() -> archive(order));
```

`List.of` is the factory idea, static creation decoupled from the caller. The `Comparator` passed to sort is Strategy, the sort algorithm staying fixed while the ordering strategy changes. The `Runnable` submitted to the executor is Command, a request turned into an object that the executor can queue and run. The families are not a museum; they are three names for three kinds of flexibility, and the JDK ships all three kinds.

The confusion pairs are where the categories do their real work, because each pair sounds the same and means something different.

Adapter versus Facade versus Proxy. All three wrap something, and the family is the same, but the problem differs. Adapter translates an interface, the client calls `send()` and the wrapper calls the SDK's `postMessage()`. Facade simplifies a subsystem, one `placeOrder()` that drives five internal calls. Proxy controls access to one object, lazy loading, security, interception. The test: is the problem mismatched interfaces, too many moving parts, or controlling access to a single target?

Strategy versus State. Both are an interface with interchangeable implementations, and both are behavioral. The difference is the intent. Strategy swaps an algorithm, and the context does not care which one is active. State swaps behavior in response to the object's own state, and the swap is driven by events. A payment calculator using Strategy picks a fee rule. A vending machine using State changes what happens when you press a button. The question that separates them: is the variation an algorithm choice, or is it the object's condition?

Factory Method versus Abstract Factory. Both create objects, and both are creational. Factory Method creates one object, and a subclass decides which one. Abstract Factory creates a family of related objects, and the factory decides which family. One product versus a matching set. The test: are you deciding one type, or a group of types that must stay consistent?

The families also predict consequences. Creational patterns pay for decoupled construction with indirection and often with more classes. Structural patterns pay for clean composition with wrapper layers and indirection. Behavioral patterns pay for loose coupling between collaborators with either more classes, more indirection, or more plumbing. The pattern families are not a way to avoid cost, they are three different places to spend it, and knowing which family you are in tells you what the bill will look like.

## Real Production Usage

Spring is the production exhibit for all three families at once. The creational family is the `ApplicationContext` itself, a factory that constructs and wires beans, and the singleton scope it manages by default. The structural family is the proxying, the `@Transactional` beans wrapped in a Proxy that opens and commits the transaction around your method. The behavioral family is the event machinery, the `ApplicationEventPublisher` that implements Observer, letting modules react to events without the publisher knowing them.

The JDK covers the families the same way. `java.io` is the structural exhibit, a stack of Decorators, the `BufferedInputStream` and `GZIPInputStream` wrapping each other. `java.util` is the creational exhibit, the static factory methods on the collection interfaces, and the behavioral exhibit at once, `Comparator` as Strategy and the enhanced for loop as Iterator made syntax. An engineer who wants to see a family done well does not need a textbook, the standard library is the textbook.

The family that gets the least respect in production is Flyweight, because its classic example, sharing glyphs in a text editor, feels distant. It shows up when the JVM interns strings, when `Integer.valueOf(5)` returns a cached instance, when a pool of objects is shared instead of rebuilt. The family survives even when the pattern's textbook example does not, which is the same lesson as the history: the idea persists, the example dates.

## Common Mistakes

The first mistake is memorizing the patterns and skipping the families. The families are the memory, and the patterns hang off them. An engineer who can say "that is a behavioral problem, so it is one of eleven, and the symptoms say it is Observer" has done the selection work in two sentences. An engineer who memorized the list of names has a pile.

The second mistake is treating the families as airtight. A pattern can have a foot in two families in practice, a Proxy is structural, and a remote Proxy is also about control, a Command is behavioral and also a way to defer work, which smells creational. The families are an index, not a law. The question they answer is still the fastest way to orient, and the overlap does not weaken the index.

The third mistake is using the family name as a decision. "It is a structural problem, so we use the structural patterns" is not a design, it is a hint. The family narrows the catalog, and the specific problem narrows it to one. The engineer who stops at the family has narrowed the search and not finished it.

## Interview Perspective

The question "walk me through the categories of design patterns" is a filter for whether the candidate holds the catalog or has only seen it. The weak answer is a recitation, "creational, structural, behavioral," with nothing after the words. The strong answer gives the map and the question each family answers. "Creational is who creates the object and which concrete class. Structural is how pieces compose without entangling. Behavioral is how objects split the work and communicate. And the JDK shows all three: factories in `java.util`, decorators in `java.io`, strategies in `Comparator`."

The follow-up that tests the map is "which family would a facade belong to, and why." The weak answer guesses "behavioral." The strong answer places it. "Structural. The facade is about composition, giving a subsystem a narrow door so the caller is not entangled with the five classes behind it. The problem it solves is arrangement, not creation and not communication."

The sharper follow-up is the confusion pair, "what is the difference between Strategy and State." The strong answer states the intent. "Both are behavioral and both are an interface with interchangeable implementations. Strategy is an algorithm choice, the context does not care which fee rule is active. State is the object's own condition, the events drive the swap, and the object changes what it does because of where it is in its life."

## Knowledge Check

1. You inherit code that wraps a third-party SDK client so the rest of the team can call `send()` instead of the SDK's `postMessage()`. Name the family and the pattern, and state which of its cousins, Facade or Proxy, the problem is not.

2. A payment system needs a fee rule chosen at runtime and a checkout that reacts to order events. Name the family and the pattern for each need, and state the single question that separates the two patterns you are using.

3. Given the patterns Builder, Observer, and Decorator, sort them into families and, for each one, name the real Java construct that is its closest native cousin.

## Key Takeaways

- The three families are three questions: who creates the object, how do pieces compose, how do objects communicate.
- The families are the index to the catalog, and a pattern's family tells you its problem before you read its description.
- Adapter, Facade, and Proxy all wrap, and the mismatched interface, the too-many-parts, and the access-control problems keep them apart.
- Every family pays for its flexibility, and knowing which family you are in tells you what the bill will look like.

## What's Next

The map is complete, the families and the confusion pairs are in place. The next article is where the map does its work: how to choose a pattern, starting from the problem instead of the catalog, and how to let the instability in your code point at the one pattern that fits. Selection is the skill the map exists to support.

---

This article explains the three families of the twenty-three patterns, creational, structural, and behavioral, and the question each family answers. Its strongest claim is that the families are the index to the catalog, and that seeing the pattern in its family tells you its problem before you read its description.
