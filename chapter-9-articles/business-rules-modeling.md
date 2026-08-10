# Business Rules Modeling

## Learning Objectives

- Turn business rules into code that states them in the business's words, as methods on the object rather than scattered booleans.
- Split rules into invariants, constraints, and derivations, and place each kind where it can be enforced and held.
- Recognize when a rule is a guard on the object in contrast to a derivation that never needs a guard.

## Introduction

Business rules are the reason the model exists, and they are the part most projects get wrong. A business rule is what the business keeps saying: an order cannot ship before it is confirmed, a customer on the blacklist cannot place an order, a discount cannot make a total negative. This chapter is about where that rule lives and how it is stated, so the code says the rule the way the business says it.

The earlier articles build the objects; this one is about the behavior that makes them worth having, and about placing each rule where it cannot be bypassed.

## Problem Statement

The failure is the rule scattered across callers. The business says "an order cannot ship before it is confirmed," and the code implements that as an `if` in the shipping method, an `if` in a service, and an `if` in a batch job. One of the three callers forgets the check, so the rule holds for the two who remembered and fails for the one who did not.

That is a rule whose enforcement depends on memory. Every caller has to remember the rule, and the rule sits wherever someone happened to write it: next to the checkout, in the report, in the import. When the rule changes, "an order can cancel when it is confirmed or pre-paid," the developer has to find every call site, miss one, and the rule is wrong in the place they missed.

The deeper failure is a rule stated as a boolean and buried. The system holds the rule, but the rule is not a named thing; it is a condition duplicated across files. Nobody can ask the model "is this order shippable?" because shippable is not a method, it is an expression every caller re-types.

## Core Concept

A business rule models well when it is expressible, when it lives in one place, and when it guards the operation it describes. The first move is to name the rule and give it a method. When the rule is "an order can ship only when confirmed," the order gets that:

```java
public class Order {
    private OrderStatus status;

    public boolean canShip() {
        return status == OrderStatus.CONFIRMED;
    }

    public void ship() {
        if (!canShip()) {
            throw new IllegalStateException("cannot ship an unconfirmed order");
        }
        this.status = OrderStatus.SHIPPED;
    }
}
```

Now the rule is a method in the model's words, and because `ship()` is the only way the model changes state, the guard in `ship()` enforces it. The caller does not restate the rule; the caller calls `ship()`, and the model either honors the change or throws. That is what the previous chapters have argued: a rule enforced by the object cannot be bypassed.

### The kinds of rule and their home

Not every rule is the same shape, and flattening them into one is a common mistake. Separating three kinds reveals where each one belongs.

| Rule kind | What it says | Where it lives | Example |
|-----------|--------------|----------------|---------|
| Invariant | Holds at every moment | The object's every change | Balance is never negative |
| Constraint | Guards a transition | The method that transitions | Cannot ship while unconfirmed |
| Derivation | A value computed from facts | Computed on demand | Total is the sum of the lines |

An invariant must hold at all times, so the object enforces it at every point it changes. A constraint guards a transition, so it lives in the method that throws, like `canShip()` above. A derivation is never guarded; it is a method, `total()`, that recomputes from the fields and returns the result.

The sweeping consequence: the invariant, the derivation, and the guard all belong to the aggregate, and each is stated once. The service, the UI, the API call the object's methods, and none of them restates the rule. When the rule changes, one method changes, and every caller receives the new behavior.

An invariant reads the same way, with the guard on every mutation. "The balance cannot go negative" is not something each funding path must remember; it is the class refusing:

```java
public class Account {
    private Money balance = Money.zero();

    public void withdraw(Money amount) {
        Money next = balance.minus(amount);
        if (next.isNegative()) {
            throw new InsufficientFundsException();
        }
        this.balance = next;
    }
}
```

The `withdraw` method is the only place the balance changes, and it is the only place the invariant has to hold. A new caller cannot violate the rule by forgetting it, because to change the balance it must call `withdraw`, and `withdraw` refuses. That is the shape of every well-modeled rule: the rule lives beside the operation that could break it, so forgetting is impossible.

### The span of an invariant

The purpose of the boundary is that an aggregate's rule can rely on its whole aggregate. When the order's discount and its lines sit in the same boundary, a guard on `applyDiscount()` can read the updated total from the lines and refuse a discount that overshoots. If that discount lived in a separate aggregate that the order only referenced by id, the guard could still throw but it would be guarding a value, the sub-total loaded at a different time, and the two would drift. Keeping the rule and everything it depends on inside the same boundary is what makes the invariant hold.

### Rules that span the aggregate

A rule that reaches across the objects inside the same aggregate is a rule of that aggregate. A rule like "an applied discount cannot drop the total below zero" spans the line items and the discount, and the order's `applyDiscount()` enforces it as the single guard. That is an invariant living on the aggregate.

A rule that crosses an aggregate, say the total to be reported for a set of orders when no one object owns it, belongs in a domain service. That is where the earlier and this article meet: single-object rules live on the object, cross-object rules live in the domain service, and the same test, does the rule stay inside one thing, separates them.

## Real Production Usage

Most of the "business rules" that show up in a Java backend split into the three kinds above, and the discipline is placing them. Spring enforces a transaction guard at the service boundary, and the domain holds the invariant. JPA validations, `@NotNull`, `@Min`, are constraints the framework checks, and the domain keeps the ones the framework cannot express, the cross-field relationships a single annotation cannot say.

The realistic lesson is that the rules that are mere validations, whether a field is present and whether it is in range, the framework handles, and the rules that are not, such as shipping allowed only when confirmed, a guard that depends on state and on two objects together, the model holds as a method. Knowing which kind each rule is decides who owns it, the validator or the model.

## Common Mistakes

**Stating the rule as a scattered boolean.** The shippable check typed again at each call site is the favorite failure. If the rule lives outside the object, it is a condition a developer can forget, and the forgotten caller is the one that ships an unconfirmed order.

**Enforcing the rule everywhere except at the change.** A service that checks the rule and a state that mutates without one is a rule with a hole. The guard has to be the method that changes state, or a path exists that bypasses it.

**Guarding a derivation that needs no guard.** A derived total, the sum of its lines, is right by construction, so a guard on it can never fire. The guard belongs to a possibility of a bad value, not to a computed result.

## Interview Perspective

Interviewers ask about rules to hear whether you realize a rule has a place to live, not just that it exists. A weak answer says "a business rule is a business rule." A strong answer says the rule is an invariant, a constraint, or a derivation, that an invariant lives on the object, a constraint guards the object's transition, and a derivation is a method that computes.

The first follow-up is "where does 'cannot ship unconfirmed' live?" The strong answer: on the order, `ship()` refuses the transition, and the model states the rule in one method. The answer "in the service" is the weak one, because the service's guard depends on being remembered. The second is "is 'the order total' a rule or a computation?" and the answer is a derivation, it is computed from the lines, so it is a method and needs no guard of its own.

Common follow-ups:

- "An order's total is the sum of its lines. Does this rule need a guard?"
- "Our rule says quantity must be between one and sixty. Which kind is it and who enforces it?"

## Knowledge Check

1. "An order cannot cancel after it ships." Name the kind of rule, where it lives, and how the placement makes it impossible to bypass.
2. "The total is the sum of the line subtotals." Is it a guard or a derivation, and does it need a guard of its own?
3. "An applied discount cannot drop a total below zero." The discount is part of the same order aggregate. Where does the guard live, and how does a cross-aggregate version of the same rule behave?

## Key Takeaways

- Name the rule as a method on the object and guard the change that matters.
- Invariants hold at every change, constraints guard a transition, derivations compute on demand.
- A rule inside the object cannot be bypassed; a rule in a caller can.
- The aggregate enforces the rules that span its own objects.
- A single method changed, a single rule, and every caller complies.

## What's Next

The next article is about domain events. A rule living on an object answers how one object holds together, and the domain event is how the model learns something happened, "the order was submitted", and lets the other aggregates react. We will cover what happened, what records, and how raising the event keeps the order from knowing the consequences of its own submitted state.

---

This article explains business rules as three kinds, invariants, constraints, and derivations, each placed on the object that owns the fact. It argues that a rule with a guard on its operation cannot be bypassed, where a rule duplicated in callers is missed.