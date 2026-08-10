# Domain Events

## Learning Objectives

- Define a domain event as a past-tense record of a fact, raised by the aggregate that caused the fact.
- Separate the fact from the reaction, so the aggregate records and never calls the consequence.
- Distinguish a domain event from a command and from an integration event, and keep each in its place.

## Introduction

A domain event is a record that something happened in the model. "The order was submitted," "the invoice was paid," "the customer was suspended." It is a fact, stated in the past tense, and the aggregate that caused the fact records it.

The reason the model records events is decoupling. The aggregate that raises an event does not need to know, or run, the consequences. When an order is submitted, the inventory, the notifier, and the auditor all want to react, and the order should not name any of them. The aggregate records the fact, and whoever cares reacts by listening.

## Problem Statement

The naive way for an order to trigger downstream effects is for the order to call them directly:

```java
public void submit() {
    if (lines.isEmpty()) {
        throw new IllegalStateException("empty order");
    }
    this.status = SUBMITTED;
    inventory.reserve(items);
    mailer.sendConfirmation(orderId);
    auditor.record(orderId);
}
```

That fails two ways.

First, the order now knows every downstream system. It holds `inventory`, `mailer`, and `auditor`, three collaborators that have nothing to do with the order's own rules. Every new downstream effect, a tax engine, a fulfillment center, a loyalty update, is a new method call and a new dependency on the aggregate. The order, which should be about its items and status, has become a hub that triggers half the company.

Second, the timing is wrong. The order cannot finish until `inventory.reserve` returns, and `mailer.sendConfirmation` blocks the whole operation on a network call. A rule that is really "record that the order is submitted" has become "do the reservation, the mail, the audit, all now, in this transaction." The moment the mailer is slow, order submission is slow.

## Core Concept

A domain event is a record, not a call. The aggregate raises the event and stops. It does not call anyone.

```java
public record OrderSubmitted(OrderId orderId, CustomerId customerId, Money total) {}
```

The event is an immutable record of the fact, and that is all it is. The aggregate records it onto a list and has no further concern with whoever reacts.

```java
public class Order {
    private final List<DomainEvent> domainEvents = new ArrayList<>();
    private OrderStatus status = DRAFT;

    public List<DomainEvent> unpublishedEvents() {
        return domainEvents;
    }

    public void submit() {
        if (lines.isEmpty()) {
            throw new IllegalStateException("empty order");
        }
        this.status = SUBMITTED;
        domainEvents.add(new OrderSubmitted(id, customerId, total()));
    }
}
```

The `submit()` method does its own rule, it refuses an empty order, it sets the status, and it records one fact. The reservation, the mailer, and the audit are none of the order's business. They subscribe to the event:

```java
public class InventoryReservationHandler {
    void on(OrderSubmitted event) {
        reserve(event.orderId());
    }
}
```

The handler reacts to the event in its own aggregate and its own transaction, some of them later and some of them async. The order aggregate and the inventory aggregate each protect their own invariants, and the event is the handoff, carrying the fact and nothing more.

Diagram: the event flow

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 920 420" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#33475b"/>
    </marker>
  </defs>

  <rect x="60" y="150" width="160" height="80" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="140" y="188" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">Order</text>
  <text x="140" y="209" text-anchor="middle" font-size="11" fill="#5a6b7a">aggregate</text>

  <line x1="222" y1="190" x2="366" y2="190" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <text x="294" y="176" text-anchor="middle" font-size="11" fill="#5a6b7a">raises</text>

  <rect x="368" y="150" width="230" height="80" rx="10" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="483" y="188" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">OrderSubmitted</text>
  <text x="483" y="209" text-anchor="middle" font-size="11" fill="#5a6b7a">record of a fact</text>

  <line x1="600" y1="190" x2="768" y2="78" stroke="#9aa7b4" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="600" y1="190" x2="768" y2="190" stroke="#9aa7b4" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="600" y1="190" x2="768" y2="310" stroke="#9aa7b4" stroke-width="1.5" marker-end="url(#arrow)"/>
  <text x="700" y="162" text-anchor="middle" font-size="11" fill="#5a6b7a">react</text>

  <rect x="770" y="40" width="120" height="76" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="830" y="73" text-anchor="middle" font-size="11" fill="#1a2733">Inventory</text>
  <text x="830" y="92" text-anchor="middle" font-size="11" fill="#5a6b7a">handler</text>

  <rect x="770" y="152" width="120" height="76" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="830" y="185" text-anchor="middle" font-size="11" fill="#1a2733">Notifier</text>
  <text x="830" y="204" text-anchor="middle" font-size="11" fill="#5a6b7a">handler</text>

  <rect x="770" y="264" width="120" height="76" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="830" y="297" text-anchor="middle" font-size="11" fill="#1a2733">Auditor</text>
  <text x="830" y="316" text-anchor="middle" font-size="11" fill="#5a6b7a">handler</text>
</svg>
```

The order raises the event; the event is a record; and three handlers react, each in its own corner. The fan-out is the whole point: the order names none of them, and a fourth handler, a tax engine, joins the diagram without the order changing at all.

### The rules of events

Two rules keep the pattern from rotting.

The first rule is that the aggregate records the event; it never calls the consequence. An aggregate that raises an event and then immediately invokes the reaction is still coupled, just wearing a long step of ceremony in between. If the reaction must be immediate and synchronous, the "event" is not earning its name.

The second rule is naming. An event is a past-tense record of what happened, `OrderSubmitted`, `InvoicePaid`. A command is a future-tense request, `SubmitOrder`, `SendInvoice`. When the two are confused, the model starts recording intent instead of fact, and the event handling becomes a command queue under another name. The past tense is the boundary: the model records facts, and commands ask for them to happen.

### What the event carries

The event carries the identities and the figures the reactors need, captured at the moment the fact happened. `OrderSubmitted` carries the order id, the customer id, and the total. It does not carry a live reference to the order's mutable lines, because the event is a value, immutable, and its fields are stable. The reactor may read the event long after the order moved on, and the event still describes what happened.

That is why the event is a record and its fields are value objects, a snapshot of the fact, not a window into the aggregate's current state.

## Real Production Usage

Spring's `ApplicationEventPublisher` and `@EventListener` are the natural home for domain events inside one service. The aggregate records the event, the service collects them from the aggregate, and the publisher delivers them to the `@EventListener` handlers. The `@TransactionalEventListener(phase = AFTER_COMMIT)` variant runs the handlers only after the transaction commits, which is the correct timing: a handler should not react to a fact that the transaction later rolls back.

Beyond one service, the event stops being a domain event. When a separate service must react, the fact is translated to an integration event and published to a broker such as Kafka. The two kinds are different layers. A domain event decouples aggregates within one service; an integration event decouples services across a broker. The anti-pattern is blurring them, publishing every domain event to Kafka and thereby reconnecting the model to a message bus, which re-tangles the very things the event was meant to separate.

## Common Mistakes

**Calling the downstream from the aggregate.** The order that invokes `inventory`, `mailer`, and `auditor` is an aggregate that knows too much. Every new reaction is a new import and a new coupling, and the aggregate stops being its own.

**Naming the event as a command.** A record named `SubmitOrder` is a request, not a fact. The past tense is the tell: events record, commands ask. A model that raises `SubmitOrder` has turned its event mechanism into a command bus.

**Handing out the live aggregate through the event.** An event that carries the order's own mutable list gives the reactor a hand into the order's current state, and the reaction reads state that has moved on. The event carries stable values, never the aggregate's internals.

## Interview Perspective

Interviewers ask about domain events to test whether you separate the fact from the consequence. A weak answer says an event is "a notification the model sends." A strong answer says an event is a past-tense record the aggregate owns, that the aggregate records and never calls the consequence, that the reaction belongs to a subscriber, and that the event carries a value snapshot of the fact, not the aggregate's live state.

The follow-up that sorts candidates is "who reacts to `OrderSubmitted`?" The strong answer: the handlers, each doing its own work in its own aggregate and transaction, and the order has depended on none of them. The candidate who answers by naming the listeners the order holds has re-coupled the design.

The second follow-up is "`OrderSubmitted` versus `SubmitOrder`." The strong answer names the past-tense fact against the future-tense request, and says the model raises the first and never the second. The weaker answer treats them as interchangeable, which is the sign the event bus has become a command queue.

Common follow-ups:

- "The mailer must react after the order transaction commits. Which annotation and why?"
- "The order raised the event and then nothing ran because the transaction rolled back. What is missing?"

## Knowledge Check

1. In the `submit()` above, why is recording the event the last thing the order does, with no `inventory.reserve` or `mailer` call in sight? Give the rule that decides.
2. `OrderSubmitted` versus `SubmitOrder`: which is a domain event, and why does the model raise the past-tense fact and rarely the command form?
3. The order's event is published after the transaction commits. What problem does that timing avoid, and what would a reactor see if the event fired before commit?

## Key Takeaways

- A domain event is a past-tense record of a fact, raised by the aggregate that caused it.
- The aggregate records the event and never calls the consequence.
- The event carries immutable key facts, ids and totals, not the aggregate's live state.
- `@TransactionalEventListener(phase = AFTER_COMMIT)` keeps reactions from responding to a fact that rolled back.
- A domain event decouples aggregates; an integration event decouples services, and blurring the two is an anti-pattern.

## What's Next

The next article is about the rich model versus the anemic model, which is the chapter's verdict on how much behavior the objects should carry. The events showed the aggregate recording and delegating; the next question is how rich the object's own behavior is. We will cover the getter and setter model where the service does the thinking, versus the rich model where the object does, and the signals that tell one from the other.

---

This article explains the domain event as a past-tense record of a fact, raised by the aggregate that caused it, with the consequence left to a listener. It argues that the aggregate records and never calls the reaction, and the event carries stable values.