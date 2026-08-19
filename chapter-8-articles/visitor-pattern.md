# Visitor Pattern

## Learning Objectives

- Define the Visitor as an operation detached from the classes it runs over, dispatch by element type instead of by operation.
- Recognize double dispatch: the element's `accept` chooses the visitor, and the visitor's overloaded method chooses the element type.
- Argue the pattern's honest trade: it adds operations without editing the hierarchy, and it costs an `accept` on every class and a new visitor method for every new type.

## Introduction

Visitor is the pattern that separates an operation from the classes it visits. You have a family of classes, an order system with regular orders, gift cards, and discounts, and you want to attach operations to them, pricing, shipping, a tax calculation, a report, without bolting a method for every operation onto every class.

The pattern puts the operations into a visitor object. The classes stay small and accept a visitor. The visitor carries the operation and dispatches by element type, so each concrete element type gets its own method in the visitor.

## Problem Statement

The direct approach is to add one method per operation to each class. An order hierarchy wants to print, to calculate shipping, to generate a report:

```java
public abstract class Order {
    public abstract String render();
    public abstract double shipping();
    public abstract String report();
}

public class GiftCard extends Order {
    @Override public String render() { /* ... */ }
    @Override public double shipping() { return 0; }
    @Override public String report() { /* ... */ }
}
```

That compiles, and it grows badly. Each new operation, a tax engine, an export, a PDF generator, forces a new abstract method on `Order` and an implementation in every subclass. The operation logic, which belongs together in one place, is scattered across a class hierarchy. The class that should describe a gift card now also knows how to render, ship, tax, and export it. Every operation change touches every concrete class.

Worse, an operation that touches several types, computing a total across a list of mixed orders, is forced to type-check and cast its way through the list:

```java
double total = 0;
for (Order o : orders) {
    if (o instanceof GiftCard g) {
        total += g.balance();
    } else if (o instanceof StandardOrder s) {
        total += s.amount();
    }
}
```

Every place that computes a total re-implements this `instanceof` chain. The type knowledge is duplicated, the casts are unchecked, and adding a new order type means editing every chain in the codebase.

## Core Concept

Visitor reverses the ownership. The elements keep a single method, `accept`, and the operations live in a visitor class with one method per element type:

```java
public interface OrderVisitor {
    double price(GiftCard card);
    double price(StandardOrder order);
    double price(CouponOrder order);
}

public abstract class Order {
    public abstract double accept(OrderVisitor visitor);
}

public class GiftCard extends Order {
    @Override
    public double accept(OrderVisitor visitor) {
        return visitor.price(this);
    }
}

public class StandardOrder extends Order {
    @Override
    public double accept(OrderVisitor visitor) {
        return visitor.price(this);
    }
}
```

Now a new operation is one new class, not a change to the hierarchy:

```java
public class PricingVisitor implements OrderVisitor {
    @Override public double price(GiftCard card) { return card.balance(); }
    @Override public double price(StandardOrder order) { return order.amount(); }
    @Override public double price(CouponOrder order) { return order.discountedAmount(); }
}
```

The client writes one clean loop, no `instanceof` anywhere:

```java
OrderVisitor pricing = new PricingVisitor();
double total = 0;
for (Order o : orders) {
    total += o.accept(pricing);
}
```

### Double dispatch

The key is double dispatch, and it is worth seeing clearly. In a normal polymorphic call, `order.price()`, one virtual call picks the method by the runtime type of the receiver. Visitor needs two types to line up: the element type and the visitor type.

The first dispatch happens in the loop: `o.accept(pricing)` calls the concrete element's `accept`, so `GiftCard.accept` runs for a gift card. The second dispatch happens inside that `accept`: `visitor.price(this)`, where `this` has the concrete element type, picks the correct overload. The element chooses the visitor, and the visitor chooses the element type. Two virtual calls, and neither side ever does a cast.

That is why the pattern is called Visitor and not "instanceof mover": the type dispatch that the direct approach spreads across the codebase is centralized into one per-class `accept` method.

### The cost of the pattern

The trade is honest, so it deserves its own section. Adding a new operation is free, one visitor class. But adding a new element type is expensive: a new `visit` method in every existing visitor, and an `accept` in the new class. The pattern fits a hierarchy that is stable and operations that grow. It fights a hierarchy that grows, because every new type breaks every visitor.

The pattern also pushes the element's internals to the visitor. `price(GiftCard)` reads `card.balance()`, which means the card exposes enough state for the visitor to work. That is the same coupling the operation methods had, just relocated, and a visitor that needs many private fields ends up forcing the element to expose them.

### The siblings

Visitor is sometimes confused with the Strategy pattern, and the difference is placement. Strategy moves one interchangeable algorithm out of a class, and the class holds the chosen one. Visitor moves a family of operations out of a class hierarchy entirely, and the hierarchy has no idea which operations exist. Strategy is an algorithm the object uses; Visitor is an operation the object accepts.

The composite and Visitor pair naturally. A composite tree accepts a visitor that descends the tree, and the visitor processes every node by its concrete type. The traversal can live in the composite and the work in the visitor, which is exactly how compilers walk an AST.

## Real Production Usage

The honest home of Visitor is a stable type hierarchy with growing operations, and no codebase fits that better than a compiler. An AST has a fixed set of node types, `Expr`, `Literal`, `BinaryOp`, `Variable`, and the compiler grows operations over it constantly: parsing, type-checking, code generation, optimization passes, pretty-printing. Each pass is a visitor over the same node types, and adding a new pass means adding a class, not editing every node. That is the pattern doing its best work.

The second production home is report and export generation over a domain hierarchy, where the domain types are stable and the exports multiply, HTML, JSON, CSV, PDF. Each export is a visitor. When the requirement is "a new format next quarter," Visitor turns a cross-cutting change into one new class.

The pattern is rarer in transactional domain logic, and the reason is the cost. If the domain hierarchy grows, if new order types arrive as often as new operations, Visitor breaks down, and pattern-matching approaches with sealed classes or explicit `instanceof` chains carry the same operation with less machinery. Choose Visitor where the type set is closed and the operation set is open.

## Common Mistakes

**Adding a new element type without touching the visitors.** The pattern's accepted trade is that a new concrete class needs a `visit` method in every visitor. Miss one and the client hits an `UnsupportedOperationException` or a default branch at runtime, when the failure is visible in the same commit. When the type set is likely to grow, say so up front and question the choice of Visitor.

**Making the visitor carry state where a parameter belongs.** A visitor that collects totals in a field and returns nothing works, and it is also the most common way to make a visitor non-reentrant. Two passes cannot share the instance, and the ordering of calls leaks into the visitor. Prefer a return value and stateless visitors.

**Forcing the pattern on an open hierarchy.** The whole point is a closed type set. If your team is adding concrete types every sprint, the visitor's per-type methods become a maintenance tax on every new type, and the pattern has inverted into a liability. Match the pattern to a stable hierarchy and you are fine; force it onto a moving one and you pay for it forever.

## Interview Perspective

Interviewers use Visitor to test whether you understand dispatch, and the weak answer says "it adds a method to the visitor." The strong answer names double dispatch, the element chooses the visitor and the visitor chooses the element type, and explains why the second dispatch needs the `accept` method and cannot be a plain virtual call.

The follow-up that sorts candidates is "what happens when you add a new type?" The strong answer: every visitor gains a method, which is why the pattern is meant for a stable hierarchy, and why a growing hierarchy pushes you toward sealed types or `instanceof` instead. The second follow-up is "visitor versus strategy," and the strong answer places Strategy as an interchangeable algorithm the object uses and Visitor as an operation the object accepts.

Common follow-ups:

- "Where exactly does the second dispatch happen, and why does `instanceof` not do it?"
- "Your team adds a new order type every sprint. Is Visitor the right call?"

## Knowledge Check

1. Trace the loop `o.accept(pricing)` for a `CouponOrder`. Which method runs first, and which method runs second?
2. Add a new operation, a PDF export, to the order system. Which files change, and which files stay untouched?
3. Add a new element type, a `PrepaidVoucher`, instead. What breaks, and why is that the pattern's known cost?

## Key Takeaways

- Visitor moves operations out of the hierarchy into a visitor class, so new operations add a class, not edits to every element.
- Double dispatch is the mechanism: the element's `accept` calls the visitor, and the visitor's overload chooses the element type.
- The cost is a new visitor method per new element type, which is why the pattern demands a stable hierarchy.
- Visitor fits compilers and export engines; a hierarchy that grows every sprint points away from the pattern.
- The visitor reads element internals, so the elements must expose the state the operations need.

## What's Next

The next article is a comparison, Strategy vs State, the two behavioral patterns that look the same at a glance and do not serve the same problem. Both swap behavior at runtime, both hide a field swap behind an interface, and both are the wrong choice when you want to learn the difference the hard way. We will read them side by side, pick the deciding question, and see the transition that tells them apart.

---

This article explains the Visitor pattern as a way to detach operations from a stable class hierarchy, using an order system as the example. It argues that the pattern earns its place when the type set stays closed while the operations keep growing.