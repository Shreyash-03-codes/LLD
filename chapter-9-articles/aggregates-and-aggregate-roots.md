# Aggregates and Aggregate Roots

## Learning Objectives

- Define an aggregate as a cluster of domain objects that must stay consistent, with one root that is the only entry point.
- Apply the consistency rule: invariants are enforced inside the aggregate, and updates to an aggregate happen atomically.
- Reference other aggregates by id, never by object, and explain why that keeps aggregate boundaries real.

## Introduction

An aggregate is a group of domain objects that are one unit when it comes to consistency. An order and its line items are an aggregate: the lines belong to the order, the total depends on the lines, and a rule like "an order must have at least one line" spans both. The aggregate is the boundary of atomic change, and the aggregate root is the object at the boundary that everything else enters through.

This is the article where the chapter moves from classifying objects to drawing borders around them. The previous article split the model into entities and value objects. This one groups them and decides who is inside, who is outside, and how the inside and outside talk.

## Problem Statement

The failure that aggregates exist to fix is unconstrained cross-reference. Model everything as equal entities that hold references to each other, and the consistency of the system depends on every code path doing the right thing.

Consider an order with lines. The natural first model gives the order a list of lines, and gives the customer a reference to the order, and gives the line a reference to the product:

```java
public class Order {
    private List<OrderLine> lines = new ArrayList<>();
    private Customer customer;

    public void addLine(OrderLine line) {
        lines.add(line);
    }
}
```

That works until a rule needs to span the order and its lines, "an order cannot be submitted with an empty or missing line." Where does that check live? If it lives in the service, every caller of `addLine` and `submit` must remember it. If it lives on the order, the order must be able to inspect its lines, which it can. But the moment the check is on the order, the order must be allowed to enforce it, and the service must not be able to construct an order out of raw parts without going through the order.

The real failure is when invariants straddle objects that the code treats as independent. If a service updates a line directly, `line.setQuantity(0)`, the order's total and its "at least one line" rule are both silently violated, because nothing forced the change through the order. Every object being independently mutable is how an aggregate's invariants get broken by a code path the aggregate never sees.

## Core Concept

An aggregate is a cluster of objects that has three properties.

First, one root. The aggregate root is the only object inside the boundary that anything outside may hold a reference to. Everything else inside, the lines, the value objects, is reachable only through the root.

Second, the invariant boundary. All invariants of the cluster, rules that span more than one object inside, are enforced through the root. The root's methods are the only ways the inside can change, so the invariants cannot be bypassed.

Third, atomic change. An aggregate is loaded, changed, and stored as one unit. When an order gains a line, the order and its lines change together in one transaction, because the consistency of the cluster is a single fact.

The root enforces the rules the cluster needs:

```java
public class Order {
    private final OrderId id;
    private final List<OrderLine> lines = new ArrayList<>();
    private OrderStatus status = OrderStatus.DRAFT;

    public void addLine(ProductId product, Quantity qty, Money price) {
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("cannot add a line to a submitted order");
        }
        lines.add(new OrderLine(product, qty, price));
    }

    public Money total() {
        return lines.stream()
                .map(OrderLine::subtotal)
                .reduce(Money.zero(), Money::add);
    }

    public void submit() {
        if (lines.isEmpty()) {
            throw new IllegalStateException("cannot submit an empty order");
        }
        this.status = OrderStatus.SUBMITTED;
    }
}
```

`OrderLine` is inside the boundary. Nobody outside holds it, nobody mutates it directly. The order's methods are the only door, so "cannot add a line to a submitted order" and "cannot submit an empty order" are not checks a caller must remember, they are properties of the class. That is the whole point of the aggregate: invariants enforced at the root cannot be bypassed.

### The boundary decision

The hard part is not applying the rules; it is drawing the boundary. The decision is one question: which objects must change together for the invariants to hold?

Everything that must change together goes in the same aggregate. An order and its lines change together, so they are one aggregate. A customer and an order do not change together, an order can be submitted while the customer sits unchanged, so they are separate aggregates. A product catalog and an order do not change together, so they are separate too.

The rule of thumb that keeps boundaries sane is to start small. A tiny aggregate is a single entity with its value objects. Merge only when a real invariant spans the objects. Most teams do the reverse, merging everything reachable, and end up with one giant aggregate that locks the world on every change. An aggregate is a consistency unit, and the smaller the consistency unit, the fewer things you must lock and the more concurrent changes can proceed.

### Cross-aggregate references

Separate aggregates must still talk. An order belongs to a customer, and an order line references a product. The rule is that the reference is by id, not by object.

```java
public class Order {
    private CustomerId customerId;
}
```

The order holds a `CustomerId`, not a `Customer`. The line holds a `ProductId`, not a `Product`. Why it matters is the boundary: if the order held the actual `Customer` object, then changing the customer could change the order's view, and the two aggregates would be entangled through a shared reference. A reference by id keeps the aggregates separate, and whoever needs the customer's details loads it by id when it needs it, through its own aggregate.

Diagram: the aggregate boundary

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 920 460" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#33475b"/>
    </marker>
  </defs>

  <rect x="340" y="30" width="280" height="380" rx="14" fill="#eef2f7" stroke="#33475b" stroke-width="1.5" stroke-dasharray="6 4"/>
  <text x="480" y="54" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">Order aggregate</text>

  <rect x="60" y="80" width="140" height="90" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="130" y="120" text-anchor="middle" font-size="12" fill="#1a2733">Customer</text>
  <text x="130" y="140" text-anchor="middle" font-size="11" fill="#5a6b7a">id: 91</text>

  <rect x="370" y="60" width="220" height="90" rx="8" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="480" y="104" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">Order (root)</text>
  <text x="480" y="126" text-anchor="middle" font-size="11" fill="#5a6b7a">id: 7841</text>

  <rect x="370" y="180" width="220" height="90" rx="8" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="480" y="222" text-anchor="middle" font-size="12" fill="#1a2733">OrderLine</text>
  <text x="480" y="244" text-anchor="middle" font-size="11" fill="#5a6b7a">value object</text>

  <rect x="370" y="300" width="220" height="90" rx="8" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="480" y="342" text-anchor="middle" font-size="12" fill="#1a2733">ShippingAddress</text>
  <text x="480" y="364" text-anchor="middle" font-size="11" fill="#5a6b7a">value object</text>

  <rect x="690" y="180" width="140" height="90" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="760" y="222" text-anchor="middle" font-size="12" fill="#1a2733">Product</text>
  <text x="760" y="244" text-anchor="middle" font-size="11" fill="#5a6b7a">id: 55</text>

  <line x1="480" y1="150" x2="480" y2="178" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="480" y1="270" x2="480" y2="298" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>

  <line x1="370" y1="125" x2="202" y2="125" stroke="#9aa7b4" stroke-width="1.5" stroke-dasharray="5 4" marker-end="url(#arrow)"/>
  <text x="286" y="110" text-anchor="middle" font-size="11" fill="#5a6b7a">CustomerId</text>

  <line x1="592" y1="225" x2="688" y2="225" stroke="#9aa7b4" stroke-width="1.5" stroke-dasharray="5 4" marker-end="url(#arrow)"/>
  <text x="640" y="205" text-anchor="middle" font-size="11" fill="#5a6b7a">ProductId</text>
</svg>
```

The dashed container is the aggregate boundary. The root is the only door in, the internal value objects sit inside, and the two outside aggregates are reached only by id. The dashed arrows are id references, not object references, and they cross the boundary without breaking it.

## Real Production Usage

Aggregates are the load-bearing unit in systems built around transactional consistency. The rule that has been proven in production, in event-sourced systems and in ORM-backed systems alike, is to keep the aggregate small and treat it as the unit of the transaction. When a command loads an order, changes it, and commits, the order aggregate is the scope of that transaction, and nothing outside it is locked.

Spring's transactional model maps naturally onto this. A `@Transactional` service method loads the root through a repository, invokes root methods, and commits; the aggregate is the consistency scope. What breaks the pattern in practice is lazy loading: with Hibernate, an aggregate boundary that loads half its members lazily has a consistency scope that depends on the session's state, and a rule that touches a not-yet-loaded member throws in production and passes in tests. The discipline that avoids it is loading the aggregate whole and eagerly within its boundary, so the root can see everything it must protect.

The other production lesson is the repository. Each aggregate root gets a repository, and the repository treats the aggregate as one loadable unit, not as a set of independent rows. Spring Data JPA repositories over an aggregate root are exactly this: `OrderRepository` loads an `Order`, and the lines come with it because they are inside the boundary.

## Common Mistakes

**One giant aggregate.** The merge-everything-reachable habit produces an aggregate that holds half the system, and every change locks half the system. The rule is to merge only when an invariant spans the objects, and to start small.

**Holding object references across aggregates.** An order holding a `Customer` instead of a `CustomerId` entangles the two aggregates. A change to the customer silently changes the order's view, and the boundary stops being real. Reference by id, load on demand.

**Direct mutation of members through lazy proxies or setters.** If a line can be mutated without going through the root, the root's invariants are advisory. The members must be reachable and changeable only through the root.

## Interview Perspective

Interviewers use aggregates to test whether you can draw boundaries under pressure. A weak answer defines an aggregate as "a root and its children." A strong answer says it is a consistency unit, that the root is the only entry point, that invariants spanning the cluster are enforced at the root and therefore cannot be bypassed, and that the aggregate is the unit of atomic change and of the transaction.

The follow-up that sorts people is "where do you draw the boundary between two aggregates?" The strong answer: where the invariants do not span the objects. Order and customer change independently, so they are separate aggregates; order and its lines change together, so they are one. The second follow-up is "how does an aggregate reference another one?" and the answer is by id, because an object reference would entangle the boundaries.

Common follow-ups:

- "Order and customer, one aggregate or two, and what decides it?"
- "A service needs the customer's email to send a confirmation. What does it hold, and what does it load?"

## Knowledge Check

1. "An order cannot be submitted with an empty list of lines." Trace the code paths that can violate this rule if `addLine` and `submit` are the only doors, versus if a service can mutate the lines directly.
2. The order holds a `CustomerId`. A service needs the customer's name. What does the service do, and what would be different if the order held a `Customer`?
3. Why is the aggregate the natural scope of a `@Transactional` method, and what breaks when a lazy-loaded member is outside the loaded scope?

## Key Takeaways

- An aggregate is a consistency unit with one root and an invariant boundary.
- The root is the only door in, so invariants enforced at the root cannot be bypassed.
- Objects that must change together are one aggregate; objects that change independently are separate.
- Cross-aggregate references are by id, never by object.
- The aggregate is the unit of the transaction, and a small aggregate keeps the world unlocked.

## What's Next

The next article is about domain services, which is where the chapter handles the logic that does not fit inside any aggregate. An aggregate holds its own rules, but some operations involve several aggregates, transfer money between two accounts, check an order against a blacklist, and that orchestration has to live somewhere that is not one object's job. We will cover what belongs in a service, what belongs on the entity, and the test that tells them apart.

---

This article explains the aggregate as a consistency unit, with one root and a boundary that enforces its invariants. It argues that aggregates stay small and separate, and that separate aggregates reference each other only by id.