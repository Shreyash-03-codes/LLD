# Identifying Entities and Value Objects

## Learning Objectives

- Separate the two object kinds by one test: does the object have identity, or is it interchangeable by its contents?
- Model value objects as immutable, structural-equality objects and stop pretending they are rows in a table.
- Apply the distinction in practice: what `equals` does, what gets persisted as a table, and what gets embedded.

## Introduction

Every domain model is made of two kinds of objects, and the difference between them drives more design decisions than any other single distinction in this chapter. The two kinds are entities and value objects.

An entity has identity. It is the same thing over time even when its contents change. An order is the same order at `confirmed`, `shipped`, and `delivered`, even though its status field changes, because it has an identity that outlives any particular value of its fields. A value object does not. A money amount of fifty dollars is not the same object as a different fifty dollars; they are interchangeable, because only the contents matter.

This distinction is not academic. It decides how you write `equals`, how you store the object, how you handle immutability, and whether a change to a field means a new thing or the same thing changed.

## Problem Statement

Most codebases model everything as an entity, and the cost shows up in four places.

The first cost is `equals`. When every object is an entity, the default identity equality works for the database-backed ones and silently breaks for everything else. An `Address` compared by reference, so two identical addresses are not equal, and a test that compares the address on an invoice to the address on a customer fails for no reason anyone can see. When everything is an entity, equality becomes either broken or a hand-written field comparison bolted onto a class that also has an id, which is a smell that the class is really two things.

The second cost is mutability. Value objects modeled as mutable entities get changed in place, and a change to a shared address mutates every object holding it. The invoice, the shipment, and the customer all reference the same `Address` instance, one `setStreet()` call later, and three documents have silently changed.

The third cost is persistence. An entity needs a table and an id, so a value object gets its own table and its own id, and the database fills with one-row tables that exist only to give an `Address` a primary key. The schema starts to mirror the ORM's requirements instead of the domain's shape.

The fourth cost is the domain logic itself. When a money amount is a mutable entity with a `setAmount()`, a price cannot be trusted to stay a price. Rules that assume values do not change, a total, a rate, a quantity, are defenseless against a `set` call that a code path makes by accident.

## Core Concept

The test that separates the two is one question: does the thing care who it is, or only what it is?

An entity cares who it is. `Order` cares that it is order 7841, and it remains order 7841 while its status, its total, and its address change. Its identity is stable and its state is mutable. Two orders are the same order if and only if they share identity, even when every field differs.

A value object cares only what it is. `Money(50, USD)` is indistinguishable from another `Money(50, USD)`, so the two are equal and one can replace the other. A value object is immutable, because a value that could change identity-defining content is a value that cannot be trusted. And its equality is structural: same fields, same value.

```java
public final class Money {
    private final BigDecimal amount;
    private final Currency currency;

    public Money(BigDecimal amount, Currency currency) {
        this.amount = amount;
        this.currency = currency;
    }

    public Money add(Money other) {
        if (!currency.equals(other.currency)) {
            throw new IllegalArgumentException("currency mismatch");
        }
        return new Money(amount.add(other.amount), currency);
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Money m)) return false;
        return amount.equals(m.amount) && currency.equals(m.currency);
    }

    @Override
    public int hashCode() {
        return Objects.hash(amount, currency);
    }
}
```

The `add` returns a new `Money`, never mutating `this`. The `equals` compares the fields, never an id. That is a value object done properly, and it is the template for every one in the system.

The entity form of the same idea:

```java
public class Order {
    private final OrderId id;
    private OrderStatus status;

    public void confirm() {
        this.status = OrderStatus.CONFIRMED;
    }
}
```

The `Order` keeps its identity (`id`), mutates its state (`status`), and its equality, if it ever needs one, is identity equality. It is the same order through state changes, and the model expresses that by never treating a field change as a new object.

### The two signals

Two signals usually settle a doubtful case.

The first signal is time. If the object's sameness survives changes to its fields, it is an entity. If changing a field destroys the object's meaning, it is a value. A `Price` that changes from fifty to sixty is not the same price; it is a different price. A `Product` that changes price is the same product.

The second signal is sharing. If two other objects each need to refer to the same thing and a change in one should be visible in the other, it is an entity. If a copy is just as good as the original, it is a value object. The street address on an order and the street address on an invoice, if they must stay identical, one shared entity or one immutable value, and if a copy is fine, a value.

### What the decision buys

Once the split is made, three decisions fall out mechanically.

`equals` and `hashCode` are structural for value objects and identity for entities. Immutability is mandatory for value objects and optional for entities. Persistence maps entities to tables and value objects to embedded columns, one row on the owning table rather than a table of their own.

Diagram: identity vs value equality

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 340" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#33475b"/>
    </marker>
  </defs>

  <text x="225" y="28" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">Entity: identity</text>
  <text x="675" y="28" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">Value Object: contents</text>

  <rect x="90" y="60" width="180" height="52" rx="6" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="180" y="84" text-anchor="middle" font-size="12" fill="#1a2733">Order 7841</text>
  <text x="180" y="102" text-anchor="middle" font-size="11" fill="#5a6b7a">status: shipped</text>

  <rect x="90" y="180" width="180" height="52" rx="6" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="180" y="204" text-anchor="middle" font-size="12" fill="#1a2733">Order 7841</text>
  <text x="180" y="222" text-anchor="middle" font-size="11" fill="#5a6b7a">status: confirmed</text>

  <rect x="540" y="60" width="180" height="52" rx="6" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="630" y="84" text-anchor="middle" font-size="12" fill="#1a2733">Money 50 USD</text>
  <text x="630" y="102" text-anchor="middle" font-size="11" fill="#5a6b7a">immutable</text>

  <rect x="540" y="180" width="180" height="52" rx="6" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="630" y="204" text-anchor="middle" font-size="12" fill="#1a2733">Money 50 USD</text>
  <text x="630" y="222" text-anchor="middle" font-size="11" fill="#5a6b7a">immutable</text>

  <text x="180" y="260" text-anchor="middle" font-size="12" fill="#1a2733">same id, same object</text>
  <line x1="180" y1="132" x2="180" y2="178" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <text x="180" y="290" text-anchor="middle" font-size="12" fill="#33475b">equal by identity</text>

  <text x="630" y="260" text-anchor="middle" font-size="12" fill="#1a2733">same fields, interchangeable</text>
  <line x1="630" y1="132" x2="630" y2="178" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <text x="630" y="290" text-anchor="middle" font-size="12" fill="#33475b">equal by value</text>

  <line x1="450" y1="40" x2="450" y2="320" stroke="#9aa7b4" stroke-width="1" stroke-dasharray="4 4"/>
</svg>
```

Left column, the same order id across two states is still one entity, equal by identity even though the state differs. Right column, two identical money values are two instances of the same value, equal by contents and interchangeable. The dashed divider marks the two categories; the labels under each column are the takeaway.

## Real Production Usage

Java's own libraries are the closest thing to a domain-model best practice on display. `java.time` models `LocalDate`, `Duration`, and `Period` as immutable value objects with structural equality, and `Money` is the same idea in the domain layer. `BigDecimal` is the canonical value object. When the JDK ships a whole API where every value is immutable and equality is by content, that is the convention to copy.

Spring Data JPA supports the split directly: a `@Embeddable` value object is stored as columns on the owning table instead of a table of its own, which is exactly the persistence decision this article argues for. The `@Entity` classes are the entities, and the embeddables are the value objects, and a mature codebase keeps the two visibly different at the schema level.

The production habit that follows: make value objects `final`, with `final` fields, no setters, a constructor that validates, and `equals` and `hashCode` over the fields. Copy-on-write methods like the `add` above. Never let a value object leak a mutable field to the outside, because the leak turns an immutable value into a shared mutable trap.

## Common Mistakes

**Modeling every value as an entity.** The `Address` table with its own id is the classic. The schema starts mirroring the ORM instead of the domain. When a class has an id you never use and equality that compares every field except the id, you have a value object dressed as an entity.

**Mutating value objects.** A money object with a `setAmount()` is a value object that lied about being a value. Rules that depend on a total staying put are defenseless. The rule is copy-on-write: `add` returns a new `Money`, and the class has no setters to begin with.

**Letting value objects escape immutable.** Returning the internal `List` or the internal `Date` from a getter hands the caller a way to mutate the value. Return a copy or an unmodifiable view, or the immutability is cosmetic and the sharing bug comes back.

## Interview Perspective

Interviewers use the entity/value split to check whether you can classify under time pressure. A weak answer says "an entity has an id and a value object doesn't." A strong answer gives the one test, does the object care who it is or what it is, then names the consequences: value objects are immutable with structural equality, entities have identity with mutable state, and the persistence decision follows the classification.

The follow-up that sorts people is "is an address an entity or a value object?" The strong answer: it depends, which is the point. If two orders sharing an address must reflect a change to that address, it is an entity; if a copy is fine, it is a value object. The second follow-up is "what does `equals` do for each?" and the answer is structural for values and identity for entities, and anyone who hesitates has not felt the bug yet.

Common follow-ups:

- "The address on an invoice must stay identical to the address on a shipment. Which kind is it?"
- "Your `Money` class has a `setAmount`. What have you broken?"

## Knowledge Check

1. An order keeps its identity across status changes, but a price that changes is a different price. Which is the entity and which is the value object, and which property decides?
2. `Address` needs `equals` and `hashCode` that compare its fields. `Order` needs equality by id. Explain why the two classes need different equality, and what breaks if both use field-based equality.
3. A value object's `getter` returns the internal mutable `List`. A customer calls `add()` on it. What is the resulting bug and what is the fix?

## Key Takeaways

- One test decides: the object cares who it is (entity) or what it is (value object).
- Value objects are immutable, structural-equality, copy-on-write, and stored embedded.
- Entities have identity, mutable state, and identity equality.
- Modeling every value as an entity fills the schema with meaningless ids and breaks `equals`.
- The classification decides `equals`, immutability, and persistence without further debate.

## What's Next

The next article is about aggregates and aggregate roots, which is where entities and value objects start being grouped. The chapter has so far been about classifying objects; aggregates are about drawing the boundary around a cluster of them, deciding what is one consistent unit and what is a separate unit that only references by id. We will cover the root, the boundary, and why consistency inside an aggregate is the rule that holds it together.

---

This article explains the split between entities and value objects by one test, which cares who it is or what it is. It argues that value objects are immutable with structural equality, and modeling every value as an entity is a mistake.