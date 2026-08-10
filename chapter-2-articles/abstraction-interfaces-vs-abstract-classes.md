# Abstraction: Interfaces vs Abstract Classes

## Learning Objectives

1. State what abstraction is, hiding the mechanism behind the contract, and name the two Java tools that express it.
2. Decide between an interface and an abstract class by asking whether you need shared state or just a contract.
3. Explain why interfaces multiply freely and abstract classes do not, and what that forces on your design.

## Introduction

Abstraction is hiding the mechanism and exposing the contract. A caller of `List.get(i)` does not know whether the list is an array-backed `ArrayList` or a linked structure, and it does not need to. The contract, get the element at this position, is enough. Java gives you two tools for writing that contract: interfaces and abstract classes. They are routinely lumped together in interview answers and routinely confused, and the confusion costs real design quality.

The honest version of the distinction is short. An interface is a pure contract. It says what an object can do, and it carries no state. An abstract class is a partial implementation. It says what an object can do, and it also provides some of how, including fields that subclasses inherit. Everything else is detail around those two facts.

## Problem Statement

A codebase defines an abstraction the wrong way and pays for it for years. The team needs every report generator to expose a `generate()` method, and they write an abstract class `ReportGenerator` with an abstract method. The first two subclasses are fine. The third needs to extend something else, a base `BatchJob` class that owns the scheduling and the retry logic. Java allows one superclass, so the third generator cannot be both a `ReportGenerator` and a `BatchJob`. The whole design bends, a workaround is bolted on, and the abstraction that was supposed to free the team now blocks them.

The mistake was choosing the wrong tool for the job. The requirement was a contract, every generator can generate, which is an interface. The abstract class added no shared implementation worth inheriting, it only added a leash. One wrong choice of the two tools, and the design that was supposed to decouple became a cage.

## Core Concept

Abstraction sits between the concrete class and its callers. The caller depends on the abstraction, not on the concrete class, and the concrete class is free to change as long as it honors the abstraction. That is the decoupling, and it is the same idea as encapsulation applied across classes: the caller does not reach into the implementation, it reaches into the contract.

The interface is the purest form. It declares method signatures, and in modern Java, default methods with a body, but it declares no fields except constants, and it holds no instance state. A class implements an interface and must provide the methods, and a class can implement many interfaces at once.

```java
public interface ReportGenerator {
    String generate();
}
```

The abstract class is the hybrid. It can declare abstract methods that subclasses must implement, and it can declare concrete methods with bodies that subclasses inherit, and it can declare fields that subclasses inherit as part of their own state. An abstract class cannot be instantiated directly, because it is incomplete by design. A subclass extends it, fills in the abstract methods, and inherits the rest.

Interface | Abstract class
--- | ---
Pure contract, no state | Can hold state, shared by subclasses
Multiple inheritance allowed | Single inheritance only
Every member public | Members can be private, protected, or public
A class can implement many | A class can extend exactly one

The decision rule that follows from the table. If the only thing the classes share is the contract, what they can do, use an interface. If they share implementation, state, or behavior that subclasses should inherit, use an abstract class. Ask the question directly: does the base need to hold state, or does it only need to promise behavior?

The multiple-inheritance constraint is the sharpest practical difference, and it decides more designs than any other factor. A class in Java has one superclass, so spending that one slot on an abstraction is expensive. If the abstraction is a contract, spend nothing, use an interface, and the class keeps its superclass slot for a real base. If the abstraction is genuinely shared state, spend the slot on the abstract class, because there is no other way to inherit state.

Default methods exist to blur the line, and you should know why. An interface can define a method with a body, a default method, so new capabilities can be added to an interface without breaking every implementer. This is how the standard library grew `List.sort` and `stream()` without breaking every class that implemented `List`. The trap is reaching for a default method when what you actually need is state. A default method cannot hold state, so if the shared thing is a field, a default method is the wrong tool and an abstract class is the right one.

There is also the matter of what each one says about identity. An interface names a capability, "this thing is sortable," and a class can have many capabilities, which is why interfaces multiply. An abstract class names a kind, "this thing is a vehicle," and a thing has one kind, which is why classes do not multiply. When you design, ask what you are expressing. A capability goes in an interface. A kind, with shared machinery, goes in an abstract class.

## Real Production Usage

The standard library is the textbook on this exact choice, and you can read the decision in its design. `List`, `Set`, `Map` are interfaces, pure contracts, and that is why `ArrayList`, `LinkedList`, `HashMap`, and dozens of implementations all coexist, each free to be whatever it is internally as long as it honors the contract. Then `AbstractList` and `AbstractMap` are abstract classes that implement most of the interface, leaving subclasses to fill in a couple of methods. The pattern is deliberate: contract in the interface, shared machinery in the abstract class, concrete behavior in the implementations.

The library also shows the capability idiom. `Comparable` and `Serializable` are interfaces that name a capability with no behavior to share. `Serializable` has no methods at all, it is a marker, and its whole job is to say "this thing can be serialized." You cannot express that with an abstract class without forcing every serializable class into a single superclass, which would be absurd. The interface is the only tool that fits.

Spring leans on interfaces for exactly the decoupling reason. A `Repository` or a `Service` interface lets the container hand callers a contract and lets the implementation be swapped, mocked, or replaced without touching the caller. The dependency injection model works because callers depend on the interface, not on the concrete bean. When the contract is all the caller needs, and Spring's whole model assumes it, an interface is the default choice.

## Common Mistakes

The most common mistake is writing an abstract class when the requirement is only a contract. The abstract class works at first, and it quietly burns the superclass slot, so the first subclass that needs to extend something else hits the wall. If the base would hold no state and provide no implementation worth inheriting, write an interface. You are not losing anything, you are keeping the class's one superclass slot free.

The second mistake is reaching for a default method when the real need is state. A default method looks like shared implementation, and it is, until you try to give it a field. The interface cannot hold instance state, and the workaround, static maps and global registries, is worse than the problem. If subclasses need to share state, that is an abstract class and no amount of interface cleverness changes it.

The third mistake is abstracting before there is anything to abstract. One concrete class does not justify an interface or an abstract class, and the "we might need a second one" interface is premature abstraction, the same disease from the design mistakes article. The contract earns its existence when a second consumer or a second implementation arrives, and it is cheap to add then.

## Interview Perspective

The question "interface vs abstract class" is a fixture, and it separates the candidate who recites the syntax from the candidate who can decide. The weak answer lists what each can contain. The strong answer gives the decision rule: interface when the classes share only a contract, abstract class when they share state or implementation, and it notes that the single-superclass constraint makes the choice expensive to get wrong.

The stronger answer reads the standard library as evidence. "`List` is the interface, `AbstractList` is the shared implementation, and `ArrayList` is one concrete case, which is why there can be many implementations of one interface but only one superclass." That answer shows the model, not just the rules.

Expected follow-ups: can an interface have fields, and can an abstract class be instantiated? Both are answered by the state question. Fields in an interface are constants only, no state, and an abstract class cannot be instantiated because it is incomplete by design. The candidate who answers both from the state-and-contract distinction has the actual understanding.

## Knowledge Check

1. A `List` interface and an `AbstractList` abstract class both exist in the JDK. What does each one provide, and what would be lost if `AbstractList` were removed and every concrete list had to implement the interface from scratch?

2. Two payment providers both need a `charge()` method and share a retry policy with a counter and a backoff schedule. Should the shared thing be an interface or an abstract class, and what fact about the retry policy decides the answer?

3. A class needs to be both a `ReportGenerator` and a `BatchJob`. Explain why `ReportGenerator` being an interface keeps that class legal, and why the same class would be impossible if both were abstract classes.

## Key Takeaways

- Abstraction hides the mechanism behind the contract, and the two Java tools express it differently.
- Interface for a pure contract, abstract class for a contract plus shared state and implementation.
- The single-superclass rule makes the choice expensive, so default to an interface until shared state forces an abstract class.
- Capabilities multiply, so they go in interfaces; a kind has shared machinery, so it goes in an abstract class.

## What's Next

Interfaces and abstract classes are both ways to define what a type is, and a type in Java often gets its identity from a hierarchy. The next article, Inheritance and Its Types, covers that hierarchy: how a class extends another, the different shapes inheritance takes, and the rules that decide whether a subclass is really a kind of its parent or just a copy.

---

This article explains abstraction as hiding mechanism behind contract and separates the two Java tools, the stateless interface and the state-holding abstract class, by what each one can carry. Its strongest claim is that the single-superclass rule makes the choice expensive, so you should default to an interface until shared state genuinely forces an abstract class.
