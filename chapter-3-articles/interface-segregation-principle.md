# Interface Segregation Principle

## Learning Objectives

1. State ISP as "no client should depend on methods it does not use," and explain what that coupling costs.
2. Split a fat interface along consumer lines, role interfaces, and know where the split line belongs.
3. Recognize the fat-interface tell: an implementer that throws `UnsupportedOperationException`.

## Introduction

The Interface Segregation Principle says no client should be forced to depend on methods it does not use. A client that depends on an interface is coupled to every method in that interface, whether it calls them or not. Change a method, and every dependent compiles against the new shape. Delete a method, and every dependent breaks. The interface is a contract with everyone who holds it, and the fat interface signs that contract on behalf of clients who never asked for the parts they are bound to.

The fix is not to make interfaces small for the sake of smallness. It is to make the interface match what the client actually needs. Split the interface along the seams where clients differ, so each client depends only on the methods it uses.

## Problem Statement

A `ReportRepository` interface starts with three methods and grows.

```
public interface ReportRepository {
    Report findById(long id);
    List<Report> findRecent(int days);
    void save(Report report);
    void delete(long id);
    void purgeOlderThan(int days);
}
```

The `ReportViewer`, a read-only screen that only ever calls `findById` and `findRecent`, is made to depend on the whole thing. The class is not the problem, the dependency is. When the team changes `save` to take an author argument, the change ripples into the viewer, which does not save anything. When `purgeOlderThan` is added, every class that implements the interface, including the one in the test fake, must add a method it will never call. The viewer's file changes, the test file changes, the build breaks, all because of a method the viewer does not use.

The specific cost is the one engineers forget. Every method in the interface is a reason for the interface to change, and every change to the interface is a change to every dependent, whether the dependent uses the changed method or not. A fat interface is a broadcast: one method changes, everyone recompiles, everyone retests, everyone is at risk. The read-only client carries the risk of the delete-and-purge feature it has nothing to do with.

## Core Concept

The principle is the client-side version of SRP. SRP says a class should have one reason to change. ISP says the same thing from the other side: an interface should not impose more than one reason to change on its clients. The `ReportRepository` has two reasons to change, the reading contract and the writing contract, and it forces every client to care about both. Split it:

```
public interface ReportReader {
    Report findById(long id);
    List<Report> findRecent(int days);
}

public interface ReportWriter {
    void save(Report report);
    void delete(long id);
    void purgeOlderThan(int days);
}
```

The viewer depends on `ReportReader`. The admin panel depends on both. The test fake implements only the interface the test needs. A change to the writing contract touches the writer and its users, and the viewer is untouched. The split did not reduce the total amount of code, it reduced the blast radius of change, and that is the entire point.

The split line is the subject, because that is where the principle is either used or abused. The interface should be split along the seams where the clients differ, not along an arbitrary taxonomy. If every client uses every method, the interface is cohesive and splitting it is ceremony. If one set of clients uses one group of methods and another set uses another, that is the seam, and the interface is fat. The question is always: who depends on this interface, and which methods does each one actually call.

This is also where ISP connects to the dependency inversion principle coming later in this chapter. The client does not consume an interface that someone else designed. The client defines the interface it needs, and the implementation conforms to the client. The `ReportViewer` defines "I need to read reports," and the repository implements the reading contract. The interface belongs to the consumer, not to the producer. That ownership is the engine behind both ISP and DIP, and it is the reason role interfaces are named after the role, `ReportReader`, not after the implementer, `ReportRepository`.

The fat-interface tell is concrete and worth memorizing: an implementer that throws `UnsupportedOperationException`. The moment a class implements an interface and cannot honestly implement one of its methods, the interface is fat for that implementer. The exception is the implementation saying "this method exists in the contract I was forced to sign, and I cannot honor it." The interface should be split until no implementer has to throw.

The one thing that does not fix a fat interface is a default method. A default implementation in the interface makes the throw optional, which hides the fatness instead of removing it. The implementer that does not care about `purgeOlderThan` silently inherits a default that does nothing, and the coupling is still there, just quieter. The viewer still recompiles when the interface changes. Defaults treat the symptom; splitting removes the cause.

There is a limit to segregation that keeps it honest. Splitting an interface into one method per client produces an explosion of interfaces that all change together, which is the "does one thing" mistake from the SRP article wearing a different shirt. The methods that always change together belong together, because they represent one contract with one reason to change. The split is correct when the parts change for different reasons, and wrong when they only look different.

## Real Production Usage

The JDK is the best demonstration because you can read the split in the names. Reading and writing are separate contracts: `Reader`, `Writer`, `Readable`, `Appendable`. A `BufferedReader` implements `Readable` and `Closeable`, it does not implement `Appendable`, because reading is a different role from writing and the JDK does not force one client to carry the other's methods. The `java.util.function` package is segregation as a design policy: `Function`, `Predicate`, `Consumer`, `Supplier` are separate single-method contracts, because a lambda consumer of a `Predicate` should not carry the methods of a `Function`.

Spring's `*Aware` interfaces are the same idea at the framework level. `ApplicationContextAware`, `BeanNameAware`, `BeanFactoryAware`, each a tiny contract for one capability, and a bean implements only the ones it needs. A bean that wants the application context does not implement the whole container interface, it implements one focused interface. The framework is segregated by consumer, exactly as the principle demands.

The read/write repository split shows up in real domain models, and it is worth the cost when the reading side is a public reporting surface and the writing side is internal. A reporting client depends on the reader and stays untouched when the write path changes shape. The two contracts are separated because they change for different reasons and are consumed by different audiences.

## Common Mistakes

The most common mistake is keeping the fat interface because "the class can do all this." The class can, and that is irrelevant. The question is whether the client should be coupled to all of it. The class implementing both `ReportReader` and `ReportWriter` is fine, a concrete class can have many roles. The mistake is the single fat interface that forces every client to depend on both roles at once.

The second mistake is segregation by structure instead of by consumer. Splitting a repository into `ReportFindByIdRepository`, `ReportFindRecentRepository`, and `ReportSaveRepository` because the methods "are different" produces three interfaces that every client implements and every change touches. The seam is the consumer, not the method. If the same client calls all three, they are one contract.

The third mistake is reaching for a default method to paper over the split. The default method does not remove the coupling, it hides it, and the interface keeps broadcasting its changes to every dependent. The `UnsupportedOperationException` disappears and the fatness stays.

## Interview Perspective

The question "explain the Interface Segregation Principle" is usually answered with the slogan, and the follow-up is where the interview happens. "Give me a fat interface and show me the split." The weak answer describes splitting in the abstract. The strong answer is concrete: "A `ReportRepository` with read and write methods forces a read-only viewer to depend on `save` and `delete`. Split it into `ReportReader` and `ReportWriter`, and the viewer depends on the reader and is untouched when the write contract changes."

The follow-up that filters is "what is the tell." The strong answer names the `UnsupportedOperationException`. "When an implementer has to throw on a method it does not support, the interface is fat for that implementer, and the interface should be split until no implementer has to throw."

The sharper question: "don't default methods solve this." The strong answer refuses the false fix. "No. A default method hides the coupling, it does not remove it. The dependent still recompiles when the interface changes, and the implementer that does nothing is still bound to the method. The split is the only real fix."

## Knowledge Check

1. A `UserRepository` has `findById`, `findByEmail`, `save`, and `delete`. A password-reset service uses only `findByEmail` and `save`. Draw the split that ISP demands, and say which service depends on which interface.

2. An implementer of `Printer` throws `UnsupportedOperationException` on `printColor`. Classify what the throw tells you, and name the two ways the design could respond.

3. A team splits a `PaymentProcessor` into `ChargeProcessor` and `RefundProcessor`, and every change to charging also changes refunding. What does the change history say about the split, and what does it imply about the seam they chose?

## Key Takeaways

- ISP is the client-side SRP: a client should not be coupled to methods it does not use.
- Split the interface where consumers differ, role interfaces, so each client carries only the contract it needs.
- The `UnsupportedOperationException` throw is the fat-interface tell, and default methods hide it, they do not fix it.
- The interface belongs to the consumer, which is also the engine behind the Dependency Inversion Principle.

## What's Next

ISP made the client own its interface. The Dependency Inversion Principle is that ownership taken to its conclusion: high-level policy should not depend on low-level details at all, both should depend on the abstraction, and the abstraction should be defined by the side that does not change. The next article covers the direction of the arrows and the diagram that makes it visible.

---

This article explains the Interface Segregation Principle as the client-side version of SRP, and how a fat interface couples every dependent to methods it does not use. Its strongest claim is that an implementer throwing UnsupportedOperationException is the tell, and that default methods hide the fatness instead of removing it.
