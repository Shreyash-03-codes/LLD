# Event Modeling: Designing Good Events

## Learning Objectives

- Name the fields an event must carry so a subscriber never has to guess or query back.
- Explain why an event is a past-tense delta, not a snapshot of an entity.
- Version and evolve an event without breaking the consumers on the old shape.

## Introduction

An event bus is only as good as the events on it. Publishing is the easy half; the real work is designing the event so a downstream service can act on it without reaching back into the source. A good event is a public contract: it announces a fact, carries the detail to react to it, and names its version so it can change without shattering the subscribers. Get the event right and the subscribers stay boring.

## Problem Statement

A catalog service publishes `OrderUpdated` and a subscriber wants to know the new total. The event contains only an order id, so the subscriber reads the id, makes its own call back to the catalog for the total, and if that call is slow or down, the whole flow stalls. Then a team renames a field and publishes, and half the subscribers read a null or work off a stale value, because the event exposed the innards of the entity instead of a stable shape. The failure is both: the event does not carry what the consumer needs, and it is not versioned, so an internal change leaks out to everyone.

## Core Concept

Design an event around the fact, not the object it came from. Four properties do the heavy lifting.

**A past-tense fact.** The event is `OrderPlaced`, `PaymentCaptured`, never `Order`, never `CurrentState`. It tells you what already happened, and it is immutable, you publish a fact and never edit it. A later fact is a new event.

**Identity and the payload.** The event carries the key and the data a consumer needs to react without a second query. `OrderPlaced` carries the order id, the customer id, and the total, so the email service builds its message and the analytics service counts the sale, and neither calls back into the source.

```java
public final class OrderPlaced {
    private final String orderId;
    private final String customerId;
    private final Money total;
    private final Instant occurredAt;
}
```

**A timestamp and an ordering key.** Every event carries a time it happened and a key the consumers group on. The key (`orderId`) is what routes all events for one order to the same place, and order guarantee happens per key, not globally. Without the key, the events for one order scatter and the sequence is lost.

**A version.** An event is published by one side and read by many who are not in the same code review. There is no rename-without-a-grievance. Keep the publisher `schemaVersion` on the event, add fields in a backward-compatible way, and if a field must split or change meaning, introduce a new event type or explicitly bump the major and let old readers either drop it or be migrated.

The rule that binds everything: publish the change, not the state. When a product price changes, publish `PriceSet(price)`, not a full `Product` object. A snapshot in the event looks convenient and becomes brittle, because a consumer that trusts it as the full truth drifts the moment a later event arrives out of order. A delta plus the key lets a consumer rebuild the view itself, and rebuilding a view from deltas is where the event-sourcing article later comes from.

## Real Production Usage

The rule I keep is from a system that models money movements: a debit and a credit are two events, not one `amount`, because that way every consumer and the audit reads the exact volume. The delta and the keyed stream let a new consumer rebuild its whole state by replaying the topic from its saved offset. Kafka's schema registry is the version holder, the contract that rejects an incompatible publish before it reaches a reader. The discipline is a consumer that reads an event and acts from the payload, no extra call to the source. That self-sufficiency is the whole reason to bother modeling the event correctly.

## Common Mistakes

1. **Publishing an id only.** The consumer must call the source to learn the detail, which is exactly the synchronous dependency event design is trying to remove.
2. **Publishing a snapshot of state.** A consumer trusts the whole object and goes stale the moment the next event arrives out of order; a delta plus a key avoids the confusion.
3. **Renaming or deleting a field like a class refactor.** It is not your property once two teams read it. Add compatibly or version and deprecate, never "rename it everywhere".

## Interview Perspective

A strong answer names the payload, the key, the timestamp, and the version, and then says the event is a public contract, not the entity. The follow-up that separates candidates is "you must change an event and consumers you do not control, what do you do," and the strong answer is: the event stays immutable, you publish the new fact with a version and either a compatible field or a new event type. They also love "why not just publish the whole product object," answered by the delta-and-key rule and the drift it prevents.

## Knowledge Check

- An event carries only the resource id. Describe the synchronous dependency that this brings back to the subscriber.
- A publisher sends a full snapshot of the product on every change. What fails about that contract on an out-of-order event, and how does a delta-plus-key change it?
- You must remove a field your own team uses. Describe the sequence of versions you would ship so no consumer breaks.

## Key Takeaways

- An event is a fact that is timestamped, immutable, carries a key and a delta.
- Carry the payload a consumer needs; an id forces a second call and a snapshot invites drift.
- Treat the event as a versioned public contract, not an entity you refactor in place.

## What's Next

With the event designed, the next article is the middle of the flow: message queues and their internal design. It is the model that stores, order, and redelivers the events you publish, and its internals, the partition, the offset, the group, are inherited the moment you pick a broker.

---

This article explains how to model an event as a timestamped, immutable, keyed delta that carries what the consumer needs and never a reference or a full snapshot. It argues that the most important test of an event is that a subscriber can rebuild its own view from the payload alone, and that events evolve by version or a new type, never by editing the old fact.