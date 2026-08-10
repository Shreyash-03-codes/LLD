# Dependency Relationships

## Learning Objectives

1. Recognize a dependency, the thinnest relationship in the object model, and name the Java features that create one.
2. Read and draw the direction of a dependency arrow, and explain what direction means for a design.
3. Choose what to depend on so the dependency makes the system swappable instead of locked.

## Introduction

A dependency is the thinnest relationship between classes. It means "this class will not compile without that one." No field, no lifetime, no ownership, just a reference in the source. A `BillingService` that calls `new CreditCardProcessor()` depends on `CreditCardProcessor`. A `Report` whose method returns a `Summary` depends on `Summary`. A `Cache` whose method throws `CacheMissException` depends on `CacheMissException`. The class references the type, and the type must exist and be compatible, or nothing compiles.

The relationship to association, aggregation, and composition is one of degree. Those three hold objects, they say something about lifetime. A dependency says nothing about lifetime at all. It can be a held field, an aggregation, or it can be a method that constructs, uses, and discards the object in a single line. What all dependencies share is the direction: the arrow points from the class that needs, to the class it needs. Design is, to a shocking degree, the art of choosing which direction those arrows point.

## Problem Statement

A `BillingService` charges a credit card for an order.

```
public class BillingService {
    public Receipt charge(Order order) {
        CreditCardProcessor processor = new CreditCardProcessor();
        return processor.charge(order.getTotal());
    }
}
```

It works. Then a second payment method arrives, PayPal, and the team does the only thing the code permits: touch `BillingService`. `charge()` grows a parameter, or a branch, or a second method. The unit tests have a problem too, the test wants to avoid hitting a real card processor, and `BillingService` constructs its own, so the test cannot substitute anything. The test must either mock a concrete class, which every mocking library discourages, or skip the billing path entirely.

The failure is not the code's logic. The failure is the dependency direction. `BillingService` points at the concrete `CreditCardProcessor`, so every change in the processor's world is a change in `BillingService`'s world. The system is as flexible as its most concrete arrow, and here that arrow is welded to a specific class.

## Core Concept

Define the dependency precisely, because precision is where the design payoff lives. A class `A` depends on class `B` when `A`'s source references `B`, and that reference can take several forms: a method parameter, a return type, a local variable, a field, a static call, a superclass or interface in the `extends` or `implements` clause, a thrown or caught exception, an annotation. Anything that forces the compiler to load `B` to compile `A`. In UML, the dashed arrow points from `A` to `B`, the dependent to the dependency, usually labeled with the verb, "uses".

The first thing to notice is how cheap a dependency is and how little it claims. It claims no lifetime, the previous article's whole drama, ownership, does not apply. A dependency can be one line. This makes dependency the most common relationship in any real system and the one that is most often drawn wrong, because designers want to promote it into something grander. It is not grander. It is a compile-time fact.

The second thing is transitivity. If `A` depends on `B` and `B` depends on `C`, then changing `C` can break `A`, through `B`. The arrows chain, and the transitive reach of a change is exactly why dependency direction matters. A class whose dependencies are all concrete leaf classes is a class that must change whenever any leaf changes.

The third thing is the one that makes the whole article worth reading: the direction of a dependency is a design decision, not a fact of nature. `BillingService` pointing at `CreditCardProcessor` looks like the only option, and it is not. Introduce an abstraction:

```
public interface PaymentProcessor {
    Receipt charge(BigDecimal amount);
}
```

and let `BillingService` depend on that. Now the arrow from `BillingService` points at `PaymentProcessor`, the interface, and both `CreditCardProcessor` and a future `PayPalProcessor` point at the interface from the other side, implementing it. The service's dependency is now on a contract, and every implementation of the contract can be substituted at runtime, which is the polymorphism article's payoff realized in structure.

This is dependency inversion, and it deserves the word inversion because the arrows changed direction. Before, `BillingService` pointed down at the low-level card processor, a high-level policy depending on a low-level detail. After, both depend on the abstraction in the middle, and the low-level detail points up at the same abstraction. The dependency inversion principle is the article for it, in the next chapter, and this is the concrete move: depend on abstractions, and let the concrete classes depend on those abstractions too.

A useful companion rule is: depend on what is stable. An interface that represents a contract rarely changes. A concrete class that represents an integration point, a card processor, a database driver, a queue client, changes with every vendor update. Every class you point at a volatile concrete class is a class that will be edited when the vendor breathes. Point the volatile classes at abstractions, and the edits stay contained in one implementation file per vendor.

Dependency direction also organizes your layers. The classic layout is web at the top, domain in the middle, infrastructure at the bottom, with arrows pointing down from the top layers into the ones below. Domain depends on nothing below it, or depends on interfaces that infrastructure implements. That keeps the domain, the code with the actual business value, the most stable, because nothing volatile points at it. When you see a diagram where every layer points everywhere, you are looking at a system where one change cascades through everything.

## Real Production Usage

Spring exists to manage dependency direction at industrial scale. A service declares its dependencies in its constructor, and the container supplies concrete implementations at startup, wired by the `@Service` and `@Repository` annotations. The service's code depends on interfaces, the container depends on the bean definitions, and nothing in the service's file names a concrete implementation. This is exactly the fix from the problem statement, institutionalized: the concrete arrows are drawn once, in configuration, instead of scattered through every class that constructs what it uses.

The test pyramid depends on it too. A unit test constructs the class under test and hands it fakes, mocks, or stubs, which requires the class to accept its dependencies rather than build them. `new BillingService(paymentProcessor)` is testable; `new BillingService()` that internally does `new CreditCardProcessor()` is not. The moment a class can be handed its dependencies, its tests get cheap and its deployment swaps get cheap, and both come from the same arrow.

The database layer shows the trade in the real world. A `UserRepository` as an interface with a `JpaUserRepository` implementation means a test can pass an in-memory fake, a stage environment can pass a Postgres implementation, and production can pass the same JPA implementation, all against the same `UserRepository` type. The rest of the application never knows which one it got, because it never named one. That is the concrete payoff of drawing the arrow at the abstraction.

## Common Mistakes

The most common mistake is constructing concrete dependencies inside the class, the `new CreditCardProcessor()` inside the method from the problem statement. It compiles, it runs, and it welds the class to its dependency so nothing can be swapped and no test can inject. The fix is always the same: accept the dependency in the constructor, and let the caller decide what arrives.

The second mistake is abstraction without a second implementation. An interface with one concrete class and no test fake and no planned alternative is ceremony, a file that exists to satisfy a principle and makes the code harder to read. The dependency inversion move pays only when something can be substituted. When in doubt, keep the concrete dependency and add the abstraction the day a second implementation appears.

The third mistake is circular dependencies. `A` depends on `B`, `B` depends on `A`, and the two cannot be constructed, tested, or understood in isolation. The fix is almost always to extract the shared piece into a third type both can depend on, which also fixes the direction. A cycle in the dependency graph is a code smell with a definite cure.

The fourth mistake is depending on the wrong layer, high-level policy code that imports the low-level driver directly. The result is that a vendor change ripples into the policy code, which is the code that must not change. The direction rule catches it: the policy should point at an abstraction, and the driver should point at the same abstraction.

## Interview Perspective

The question "what is the difference between association and dependency" wants a precise, one-line answer. "Association is a structural relationship between objects that know each other. Dependency is any use of a type that makes the source depend on it, a parameter, a return type, a local variable, and it says nothing about lifetime." The candidate who can say that in one breath has the relationship taxonomy.

The design question "how do you reduce coupling" wants the direction move. "Depend on abstractions instead of concrete classes, accept dependencies in the constructor, and let the caller supply them." The follow-up "why interfaces and not abstract classes" routes back to the abstraction article: interfaces are the pure contract with no state to drag into the dependents.

The trickiest version is "does dependency injection mean dependency inversion." The answer is no, and being able to say why is a differentiator. "Injection is a mechanism, the container supplies the instance. Inversion is a principle, the arrows point at abstractions. Injection is a way to implement inversion, and you can invert without a container, by wiring in the constructor of the composition root." Two words, two levels, and the candidate who holds them apart is rare.

## Knowledge Check

1. List every Java feature that can create a dependency from one class on another, and identify which of them create lifetime ties and which create none.

2. A `ReportService` constructs `PdfRenderer` inside its method, and a second renderer, `HtmlRenderer`, is required. Describe the exact change, and name which article's principle the change uses.

3. `A` depends on `B`, `B` depends on `C`, and `C` changes its constructor. Trace the transitive blast radius, and then restructure the three so that only one file needs to change.

## Key Takeaways

- A dependency is any use of a type that makes the source compile against it, and it says nothing about lifetime.
- The direction of the dependency arrow is a design choice, and good design points at abstractions, not concrete classes.
- Constructors should receive dependencies; classes should not build the things they need.
- Depend on what is stable, keep the domain clean, and add the abstraction when a real second implementation exists.

## What's Next

Dependencies carry types across their arrows, and the next article, Generics and Type Safety, is about what a type can and cannot be when it moves: how one method can work for every type without losing the compiler's checks, and why the erasure underneath the syntax explains most generics surprises.

---

This article explains the dependency, the thinnest relationship in the object model, a compile-time fact that carries no lifetime meaning, and shows that its direction is a design choice. Its strongest claim is that depending on abstractions instead of concrete classes, with dependencies supplied through constructors, is the move that makes a system swappable and a test injection possible.
