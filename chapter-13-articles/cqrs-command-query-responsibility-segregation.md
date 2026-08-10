# CQRS: Command Query Responsibility Segregation

## Learning Objectives

- Define the split CQRS names: a write model that changes state and a read model that answers queries.
- Explain when the shape of the two should differ and why one object for both is a bias toward whichever side it favors.
- State the storage options, the same store split into a read replica, distinct read stores, and when the extra is warranted.

## Introduction

CQRS is the short name for command query responsibility segregation: the code that writes state and the code that reads it stop being one class. It gets paired with event sourcing all the time, but it is its own idea and you can run it without a single event. The claim underneath is plain. A single `Order` object used to update stock, render a page, and feed a report assumes that reads and writes want the same shape, and that assumption is worth questioning.

## Problem Statement

One `Order` class is the object the service loads, the object the front end renders, and the object a reporting job scans. Three users with three demands. The write path wants invariants and a transactional guard. The read path wants no business logic and the cheapest page, ideally a flat view. The report wants high throughput across many rows. Every concern hits the same shape, so the model is a compromise that fits none of them, and a read that grows joins slows the same write that shares the object. The failure is one class being asked to be a command, a query, and a projection.

## Core Concept

CQRS splits the model in two. The write side speaks in commands, `PlaceOrder`, which carry the intent and land on an aggregate that enforces the rules and changes state. The read side speaks in queries, `GetOrderView`, and returns a projection shaped for a screen or a report. The two do not share a class and are not required to share a store.

```java
// the write model: a command describes an action, an aggregate holds the rules
public record ChargeCommand(String accountId, Money amount) {}

// the read model: a view shaped for a screen, with no behavior
public record AccountStatementView(String accountId, Money balance, List<ChargeView> rows) {}
```

The names tell the point. `ChargeCommand` is an instruction carrying what it takes to act. `AccountStatementView` is a shape whose wording answers the UI. CQRS is the rule that these two never become a single `Account` that mutates while it renders.

The payoff is in the design, not the JSON. Once the write and the read are two classes, each takes its own shape: the write model protects the invariant, and the read model is free to be a dumb join that serves the page. The read that used to slow the write because they shared a table now keeps its own projection, and the write transaction stays short.

## Real Production Usage

CQRS and event sourcing pair well, because the projection from the event log already provides a read shape and the aggregate gives a clean write. But CQRS alone appears in a simpler jacket: a read replica, the same database where reads go to a secondary and writes go to the primary. Reads scale by adding replicas, and the writes are never slowed by the read load. When the read and the write push in opposite directions, a duplicate read model is worth the price.

The level of separation is a dial. Some systems split only the class and the transaction boundary; others split the schema and keep a read store that folds events. The dial turns up as read pressure climbs and drops back near zero when the application is a form that edits a couple of fields.

## Common Mistakes

1. **Adding CQRS to a CRUD service that already fits.** Four reads and two writes gain a projection, two stores, and a staleness window the single-form user never asked for. The machinery does not pay.
2. **Splitting names, not models.** If the write and the read still share one class and you only rename the methods, you have CQRS without the separation. Behavior never a working, the model did.
3. **Letting the read mutate the write.** A query that writes back to the write model grows a second authority, and you have the very same shape you removed.

The telltale anti-pattern to watch for is an ORM entity passed up to the renderer and down to the writer at the same time. That is the shared shape returning, and the fix is the same line: a command into the aggregate and a separate projection to the query, with the view built by folding.

## Interview Perspective

The interviewer wants to hear that you know CQRS is a real separation, not a rename. Weak: "CQRS is separate read and write." Strong: "CQRS runs a command into the write side and a query into the read side, so the shape matches the job, the reads get a replica or a projection, and the write keeps its invariant." Follow-up: "does CQRS need event sourcing," and the strong answer: "the projection is a natural pair, but CQRS needs only a separate read, a replica works, so the two are close but independent."

A second follow-up is "when do you reach for it," and the answer is when the read and the write genuinely differ in scale or shape, or when the read must not be coupled to the write's transaction.

## Knowledge Check

- A single object is loaded, changed, and returned by the same code path. Which two pressures fight on that object, and what does each side of CQRS give its own?
- The read side can use a separate shape. Name the two storage arrangements that satisfy it.
- Does CQRS require event sourcing, and what is the minimal version? Explain why the two are often paired but independent.

## Key Takeaways

- The write model is a command plus an aggregate; the read model is a view plus a projection, and they are two shapes.
- CQRS earns its keep when the read pressure outweighs the write; it is not a default, and the CRUD that fits one shape is fine as it is.
- A separate read can reference a replica or a fold, which is what lets the read scale apart from the write.

## What's Next

The split models of the chapter are in place, and every one of them is a place to fail quietly. The next article gathers those failure habits, the common pitfalls of event-driven systems, and reads back through the chapter to name the mistakes that make each technique fail in a running system.

---

This article explains CQRS as separating the write model, a command and an aggregate, from the read model, a view and a projection, so each takes the shape its work demands. It argues that CQRS is justified by a reads-and-writes load that diverges, and that both a read replica and a source projection are valid reads, so CQRS does not depend on event sourcing.