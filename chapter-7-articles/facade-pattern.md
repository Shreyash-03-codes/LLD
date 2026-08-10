# Facade Pattern

## Learning Objectives

- Build a facade that gives a subsystem one front door, and explain the difference between simplifying a subsystem and hiding it.
- Keep the facade thin and delegate the work, which is what separates it from a god class.
- Recognize the real facades in the ecosystem, Spring's `JdbcTemplate` and SLF4J's `Logger` being the two clearest.

## Introduction

Facade provides a unified interface to a set of interfaces in a subsystem, so the client deals with one method instead of ten classes. Where Decorator adds layers, Facade removes awareness. The client stops knowing that inventory, payment, and shipping are three separate services with three separate quirks, and starts knowing only "place an order."

The facade is not a manager, and it is not a bus. It is a front door. Everything the client needs goes through it, and the facade translates each request into the subsystem calls that make it happen.

## Problem Statement

The failure is entanglement. A checkout flow touches three services, each with its own ordering requirement and its own error convention:

```java
public class CheckoutController {
    private final InventoryService inventory;
    private final PaymentService payment;
    private final ShippingService shipping;

    public void checkout(Cart cart) {
        if (inventory.reserve(cart.getItems())) {
            boolean paid = payment.charge(cart.getTotal());
            if (paid) {
                shipping.schedule(cart.getAddress());
            }
        }
    }
}
```

Now count the ways this rots. Every caller of `checkout` must repeat the sequence, so the second controller, the API gateway, and the batch job each reimplement reserve-then-charge-then-schedule, each with their own slightly wrong ordering. Every caller must know the three service classes by name and wire them together. Every caller must know that `reserve` returns a boolean but `charge` does not, and what each failure means. The workflow, which is a single business operation, has been scattered across the whole codebase, and the ordering logic, the part that is genuinely one decision, exists in as many copies as there are callers.

The failure is not the three services. They are fine. The failure is that the client is forced to know the subsystem's internals, the sequence, the error conventions, and the wiring, when all the client actually wants is a completed order.

## Core Concept

Facade gives that operation a home. One class, `OrderFacade`, owns the sequence and the wiring, and the client depends on it:

```java
public class OrderFacade {
    private final InventoryService inventory;
    private final PaymentService payment;
    private final ShippingService shipping;

    public OrderFacade(InventoryService inventory,
                       PaymentService payment,
                       ShippingService shipping) {
        this.inventory = inventory;
        this.payment = payment;
        this.shipping = shipping;
    }

    public void placeOrder(Cart cart) {
        if (inventory.reserve(cart.getItems())) {
            if (payment.charge(cart.getTotal())) {
                shipping.schedule(cart.getAddress());
            }
        }
    }
}
```

The client collapses to what it always should have been:

```java
public class CheckoutController {
    private final OrderFacade orders;

    public CheckoutController(OrderFacade orders) {
        this.orders = orders;
    }

    public void checkout(Cart cart) {
        orders.placeOrder(cart);
    }
}
```

Every caller of `placeOrder` now gets the sequence for free, correctly ordered, in exactly one place. A new consumer of the checkout flow depends on one class and one method. When the ordering changes, say the charge moves before the reserve, one file changes. This is the entire value of the facade: the workflow becomes a named, single-owner operation instead of a convention that every caller has to remember.

Diagram: facade in front of a subsystem

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1000 380" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#444"/>
    </marker>
  </defs>

  <rect x="40" y="150" width="200" height="80" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="140" y="172" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">Client</text>
  <text x="140" y="198" text-anchor="middle" font-size="12" fill="#1a2733">+placeOrder()</text>

  <rect x="380" y="140" width="240" height="100" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="500" y="164" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">OrderFacade</text>
  <text x="500" y="192" text-anchor="middle" font-size="12" fill="#1a2733">+placeOrder()</text>
  <text x="500" y="216" text-anchor="middle" font-size="12" fill="#1a2733">+getStatus()</text>

  <rect x="700" y="60" width="260" height="80" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="830" y="82" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">InventoryService</text>
  <text x="830" y="108" text-anchor="middle" font-size="12" fill="#1a2733">+checkStock()</text>

  <rect x="700" y="160" width="260" height="80" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="830" y="182" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">PaymentService</text>
  <text x="830" y="208" text-anchor="middle" font-size="12" fill="#1a2733">+charge()</text>

  <rect x="700" y="260" width="260" height="80" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="830" y="282" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">ShippingService</text>
  <text x="830" y="308" text-anchor="middle" font-size="12" fill="#1a2733">+schedule()</text>

  <line x1="240" y1="190" x2="378" y2="190" stroke="#6b7a89" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
  <line x1="620" y1="170" x2="698" y2="110" stroke="#6b7a89" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
  <line x1="620" y1="190" x2="698" y2="200" stroke="#6b7a89" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
  <line x1="620" y1="210" x2="698" y2="290" stroke="#6b7a89" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
</svg>
```

The client sees one box. The facade sees three. The arrow count tells the story: one dependency on the left, three on the right, and the right-hand couplings live entirely inside the facade.

### What a facade does not do

Three constraints keep a facade honest, and violating any of them turns it into the very thing it was meant to prevent.

First, the facade does not hide the subsystem. It simplifies it. Nothing stops a caller from reaching past the facade and using `PaymentService` directly when it needs a raw capability. Trying to enforce total encapsulation by making the subsystem private turns the facade into a gatekeeper and forces every capability through one bottleneck. The facade is a convenience, not a wall.

Second, the facade does not decide. It delegates. If `OrderFacade` starts implementing pricing, discount rules, and fraud checks, it is a god class wearing a facade costume. The facade's job is orchestration: the sequence and the wiring. The decisions belong to the services it calls.

Third, the facade does not leak its internals through its return values. If `placeOrder` returns a `PaymentResult` type that only `PaymentService` defines, callers are back to knowing the subsystem. Facade boundaries that return subsystem types are facades in name only.

## Real Production Usage

Spring's `JdbcTemplate` is the best large facade in the ecosystem. Raw JDBC is a subsystem: `Connection`, `Statement`, `ResultSet`, exception translation, resource closing. `JdbcTemplate` fronts all of it with methods like `queryForObject` and `update`, and the client never sees a `ResultSet`. The same shape appears as `NamedParameterJdbcTemplate` and in Hibernate's `Session`, which fronts JDBC, the first-level cache, dirty checking, and connection management behind a small set of operations.

SLF4J's `Logger` is a facade in the purest form, a logging front door whose backends, logback, log4j, java.util.logging, are interchangeable behind it. `Logger.info(...)` hides which backend exists, which is the entire point of the library. When you meet a class whose name ends in `Template`, `Session`, `Manager`, or `Facade` and whose methods are noticeably simpler than the machinery beneath, you are probably looking at this pattern.

A subsystem can have more than one facade, and usually should. The order-placing client wants different operations than the admin client, and one `OrderFacade` serving both tends to grow until it serves neither well. Multiple facades, each tuned to a client, keep every facade thin and every client on its own front door. This is the difference between a facade and a wall: a facade is per-client simplification, not a single bottleneck every caller is forced to share.

The facade also earns its keep at architectural boundaries. A batch job, a CLI tool, and a REST controller can each depend on the same `OrderFacade`, and none of them needs to know the three services behind it. The boundary becomes the facade, the subsystem can be replaced without touching consumers, and the consumers stay stable. That is the facade doing architectural work, not just tidying up a controller.

## Common Mistakes

**Letting the facade grow until it is a god class.** The facade accumulates orchestration, then policy, then the business rules, until it is the largest class in the system and every feature touches it. The discipline is that the facade orchestrates and delegates. The moment it starts deciding, split the decision out.

**Hiding the subsystem so thoroughly that raw capabilities are unreachable.** A facade that walls off the services forces the special case to go through the general door, which is how you end up with a facade that takes eleven parameters to handle the case its designer did not think of.

**Returning subsystem types from facade methods.** The facade's contract should be its own vocabulary. If callers must import `PaymentService` types to use `OrderFacade`, the facade has not actually reduced what the client knows.

## Interview Perspective

Facade is the pattern interviewers use to test restraint. It is easy to define and easy to overuse, so the strong answer is about the boundaries. A weak answer says "it's a class that wraps other classes." A strong answer says "a facade gives the subsystem one front door, it delegates rather than decides, it does not wall off the subsystem, and if it grows decisions it has become a god class."

The follow-up usually probes the difference between simplification and encapsulation, or between a facade and an adapter. "Your facade returns a subsystem type. What is wrong?" is the kind of question that separates people who have maintained one from people who have read about one.

Common follow-ups:

- "How is a Facade different from an Adapter?"
- "When does a facade stop being helpful and start being a bottleneck?"

## Knowledge Check

1. A new team wants to use `InventoryService` directly for an admin restock tool. Does the facade forbid it, and should it?
2. `OrderFacade` has grown a `applyDiscounts()` and a `checkFraud()` method that implement real logic. What has the facade become, and where does the logic belong?
3. `JdbcTemplate` hides `ResultSet` entirely from its callers. Name one thing a client of `JdbcTemplate` cannot do anymore, and one thing it never has to worry about again.

## Key Takeaways

- Facade gives a subsystem one front door and makes a workflow a single, single-owner operation.
- The facade delegates and orchestrates; the moment it decides policy it has become a god class.
- A facade simplifies, it does not hide, and it must not leak subsystem types through its contract.
- Spring's `JdbcTemplate` and SLF4J's `Logger` are the clearest production facades in the Java world.
- One dependency in, three couplings out, all three contained in one class, is the whole shape.

## What's Next

The next article is Proxy, the last of the three wrapper patterns, and the one that is easy to misplace next to Facade. Where Facade simplifies a whole subsystem for a client, Proxy controls access to a single object: lazy loading, permission checks, or a remote call behind a local face. We will cover the three real kinds of proxy and where the JDK and Spring build them for you.

---

This article explains the Facade pattern as a single front door over a multi-class subsystem, with an order placement workflow as the running example. It argues that a facade must orchestrate and delegate without deciding policy, and that a facade which starts hiding its subsystem or returning subsystem types has stopped being one.
