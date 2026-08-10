# Putting It Together: Full Domain Model Walkthrough

## Learning Objectives

- Assemble a full domain model from the pieces, entity, value object, aggregate, service, repository, rule, and event, in one system.
- Walk a real use case through the model and name which layer each step runs in.
- Read a complete model and identify the rich objects, the boundary, and the anemic trap.

## Introduction

This is the last article in the chapter, and it exists to prove the chapter fits together. All the pieces, entities, value objects, aggregates, domain services, repositories, business rules, and domain events, have been covered separately. This article builds one model out of them and walks a concrete flow through it, so the separate lessons read as a single design instead of a list.

The system: an online store. It has customers, orders, and a checkout that must place an order for a qualified customer, reserve the items, and record that it happened. It is small, and it is enough to touch every piece.

## Problem Statement

Without the discipline of this chapter, the same store gets built as a set of mutable tables with a fat service. The `Order` is fields and getters, the `Customer` is fields and getters, and one `CheckoutService` holds the qualification, the totals, the reserving, and the side effects, mixing the domain rules with the web plumbing.

That model is at risk in the three ways the chapter named. The rules have no home, so a suspended customer ships by the path that forgot to check. The entities are mutable bags, so any caller can rewrite state without a rule. And the domain is indistinguishable from the transport, so nothing in the codebase can say what the model is for. The walkthrough exists to show the same store built the way this chapter argues, and to make concrete what the pieces were for.

## Core Concept

### The building blocks

Start with the value objects. Money, and the quantity of an item, carry value-object correctness: immutable, structural equality, the `equals` over the fields. The order does not carry a raw number and a currency string, it carries a `Money` with `add`/`minus` and the unit guard.

The entities carry the lifecycle. An `Order` is an entity with identity; a `Customer` is an entity with identity. Each one is a root of an aggregate.

```java
public final class Money {
    private final BigDecimal amount;
    private final Currency currency;

    public Money add(Money other) {
        requireSameCurrency(other);
        return new Money(amount.add(other.amount), currency);
    }

    private void requireSameCurrency(Money other) {
        if (!currency.equals(other.currency)) {
            throw new IllegalArgumentException("currency mismatch");
        }
    }
}
```

### The aggregate

The `Order` aggregate owns its worth: the root is the only door, the lines and the address live inside, and the rules that span them live on the root.

```java
public class Order {
    private final OrderId id;
    private final List<OrderLine> lines = new ArrayList<>();
    private OrderStatus status = OrderStatus.DRAFT;

    public Money total() {
        return lines.stream().map(OrderLine::subtotal).reduce(Money.zero(), Money::add);
    }

    public void submit() {
        if (lines.isEmpty()) {
            throw new IllegalStateException("cannot submit an empty order");
        }
        this.status = OrderStatus.SUBMITTED;
    }
}
```

`submit()` refuses the empty order, and `total()` is the derivation computed from the lines. The other aggregate, `Customer`, holds the customer's own rules, the blacklist state and the qualification, and its reference to the order is by id, never by object.

The domain service is the transaction that spans the two aggregates. The checkout is not the order's job to reach into the customer and not the customer's job to reach into the order, so it is a service.

```java
class CheckoutService {
    private final OrderRepository orders;
    private final CustomerRepository customers;

    Order checkout(CustomerId customerId, Basket basket) {
        Customer customer = customers.find(customerId);
        customer.requireEligible();
        Order order = Order.create(customerId, basket);
        order.submit();
        orders.save(order);
        return order;
    }
}
```

`requireEligible` is the customer's own guard, `submit` is the order's own guard, and the service coordinates and persists.

### Raising the event

The checkout has a side effect: inventory should reserve the items, and the customer should be notified. The order records the fact, and the service publishes it.

```java
public record OrderSubmitted(OrderId orderId, CustomerId customerId, Money total) {}
```

The order's `submit()` could add the event to its `domainEvents` list, and the service, after committing, publishes it with `@TransactionalEventListener(phase = AFTER_COMMIT)`. The inventory handler and the notifier react, each in its own aggregate and its own transaction, and the order names neither.

### The shape

Put together, the model reads like the business. An `Order` root that refuses an empty submit, a `Money` that never mixes currencies, a customer that refuses an unqualified checkout, a service that coordinates, a repository that hides storage, and a `OrderSubmitted` record that hands the fact to the reactors. No single piece is novel; the domain is the sum.

Diagram: the parts that make up the model

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 980 460" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#33475b"/>
    </marker>
  </defs>

  <rect x="20" y="130" width="310" height="230" rx="14" fill="#eef2f7" stroke="#33475b" stroke-width="1.5" stroke-dasharray="6 4"/>
  <text x="175" y="158" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">Order aggregate</text>

  <rect x="60" y="180" width="210" height="120" rx="8" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="165" y="210" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">Order (root)</text>
  <text x="165" y="236" text-anchor="middle" font-size="11" fill="#5a6b7a">submit()</text>
  <text x="165" y="258" text-anchor="middle" font-size="11" fill="#5a6b7a">lines</text>

  <rect x="360" y="180" width="230" height="120" rx="8" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="475" y="214" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">CheckoutService</text>
  <text x="475" y="240" text-anchor="middle" font-size="11" fill="#5a6b7a">domain service</text>
  <text x="475" y="262" text-anchor="middle" font-size="11" fill="#5a6b7a">coordinates</text>

  <rect x="650" y="180" width="250" height="120" rx="8" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="775" y="214" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">Customer</text>
  <text x="775" y="240" text-anchor="middle" font-size="11" fill="#5a6b7a">aggregate</text>
  <text x="775" y="262" text-anchor="middle" font-size="11" fill="#5a6b7a">id: 91</text>

  <line x1="300" y1="240" x2="358" y2="240" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>

  <line x1="590" y1="240" x2="648" y2="240" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
</svg>
```

The dashed boundary is the order aggregate. The service in the middle uses each aggregate, the order through the root and the customer by id, and no arrow reaches into the other aggregate's internals. The shape is the chapter.

### The anemic check

Run the two tests from the previous article against this model. Can a service rewrite the order's status without going through the root? No, `submit()` is the only door. Do the method names read business verbs? `submit()`, `total()`, `requireEligible()`, yes. That is the rich model, and it is not because anyone forced it, it falls out of placing each rule at the object that owns it.

## Real Production Usage

This is the shape of a domain running under Spring. The `Order` and `Customer` are plain objects, `@Entity` free, and the persistence is a layer that maps them. `OrderRepository` and `CustomerRepository` are interfaces in the domain and the Spring Data implementations bind at the edge. The `CheckoutService` is a `@Service`, and the checkout runs in a `@Transactional` method that loads the aggregates through the repositories, changes them through their roots, saves, and then the event is published after the commit.

The piece worth stating is the discipline. This codebase has no `setStatus` and no getter soup; everything the business says, "cannot submit an empty order", "cannot check out a suspended customer", "cannot add a currency and a fee", appears once, on the object that owns the facts. When a rule changes, a rule changes in one method, and when a new rule lands, it lands in the object that holds it.

## Common Mistakes

**Building the same model a second time at the storage layer.** Defining the aggregates in the domain and then re-creating parallel classes in a persistence package for the mapping is a common source of the two-models smell. When the domain changes, the storage must change in two places, by hand and in sync. The discipline is a small mapping layer, and the watch is the drift that the copy silently makes.

**Letting the integration drift into the domain.** A domain event that must reach another service is translated to a message at the boundary, not published by every aggregate method. The model in which every command publishes to the broker has reconnected the very units the event was meant to separate.

**Giving up the rich boundary at one seam.** One eager service that mutates an entity field to "just fix it" re-introduces the anemic seam into an otherwise rich model. The trigger test is the method names; the day a `set` appears in the domain, the guard is gone.

## Interview Perspective

The walkthrough is the shape interviewers like to hear in a domain-modeling round. A weak answer lists the pieces, "here is the entity, the service, the repository, the value object." A strong answer walks the flow and names the behavior at each stop: the order refuses the empty submit, the money refuses the mismatch, the service coordinates the customer and the order, the repository persists, and the event publishes the fact.

The follow-up that decides it fast: "who owns the rule that an order cannot submit empty?" The strong answer points at `Order.submit()`. And "who owns the rule that a suspended customer cannot check out?" The strong answer is the customer's `requireEligible()`. The answer the interviewer is listening for is that the rule lives with the object that owns the fact, not with the coordinator that happens to call it.

## Knowledge Check

1. In the `CheckoutService` above, `order.submit()` and `customer.requireEligible()` are each called. Where does each rule live, and what would be different if the service reimplemented the suspended check itself?
2. The order records an `OrderSubmitted` event and the service publishes it after commit. Trace who orchestrates, who records the fact, and who runs the side effect, and confirm that the order depends on none of the reactors.
3. The model has no `setStatus` and no setter on the order. Which rich-anemic check does that pass, and what would the tell be if the order grew a setter?

## Key Takeaways

- The value objects, the aggregate, the domain service, the repository, the rules, and the event all cooperate in one model.
- The root enforces the order's rules, the customer's own guard, the service coordinates across, and the repository persists.
- The rich model falls out when each rule lives at the object that owns the fact.
- The boundary holds: no aggregate reaches into another's internals, and references pass by id.
- The one place to change a rule is the one object that owns it, and every caller obeys.

## What's Next

This is the last article of the chapter, and it closes the domain. The next chapter is about API and Interface Design, and the change is the direction the model faces. Everything so far was the internal shape of the domain, the aggregate holding its invariants, the object carrying the rules, the event handing off the fact. The next chapter turns the model outward, to the boundary it presents to callers: the contract of the interface, the shape of the request and response, the versioning of the boundary, and what the outside world is allowed to see and do. The objects are already built; the next problem is the boundary they offer to the world.

---

This article explains one full model, an order aggregate, a customer, a domain service, a repository, and the submitted event, and walks a checkout through them. It argues the model holds because each rule lives on the object that owns it, and the service coordinates.