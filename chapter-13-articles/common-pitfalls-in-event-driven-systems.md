# Common Pitfalls in Event-Driven Systems

## Learning Objectives

- Name the failure that each of the earlier techniques invited and how to spot it early.
- Explain why ordering and idempotence fail close at scale even when they hold in tests.
- State the habit that catches most of these, tracking what the system actually did, over what it was going to.

## Introduction

Every technique in this chapter is a place to fall. A queue that looks solid, an event modeled clean, an aggregate that works on the demo, all of them hide a pitfall that only shows after the flow runs in production with many services and no one rehearsed the failure. This article collects the failures in one place: lost events, duplicated events, out-of-order deliveries, and the drift that never knows it is drift. Most are avoidable, and the blunt truth is that they are avoided by the same boring habits.

## Problem Statement

A system publishes a `PaymentCaptured`, and the ledger service, the search service, and the analytics service all react. Weeks pass before anyone notices that a rare late network retry also published the same payment twice, the ledger recorded it twice, and the search shows a number the audit disagrees with. The events fell in a different order than they were written, and one consumer rebuilt a state that matched nothing. No test caught it because ordering and duplication are properties of the running system, not the code path a unit test checks.

## Core Concept

The pitfalls cluster into a few families. Knowing them by name, and recognizing them as you design the flow, is the skill.

**The lost event.** A producer publishes and moves on because it assumes the broker is durable. If the publish is fire-and-recover, a failure between the business write and the publish drops the event, and no one misses it. The fix is the transactional outbox: the event is written in the producing service's own database, in the same transaction as the state change, and a relay publishes it from the outbox. The durable list is the contract.

**The duplicate event.** At-least-once redelivery means an event can arrive twice, after a rebalance or a retry. A consumer that treats a charge as `+= amount` every time applies it twice. The fix is exactly what the earlier articles named: make the fold idempotent, deduplicate on a stable event id, and treat a redelivery as a no-op.

**Ordering.** The log orders per key. A consumer that reads events for one key from different partitions can see `Shipped` before `Placed`. Fixes narrow it: key every event that mutates one entity, put the ordering-required events in one partition, or make the consumer re-order using the sequence carried in the event. The first, a keyed stream, is the default.

**The drifting view.** A projection is built from a fold of stale or missing events, so the view reports a value with no "as of" and no way to tell how far behind. The failure is invisible. The fix is the vocabulary of the earlier article: timestamp the view and rebuild from the log instead of editing by hand.

| Pitfall | What actually happens | The repair |
|---------|-----------------------|-----------|
| Lost event | publish skipped after the write | outbox in the same transaction |
| Duplicate handling | replay applies twice | idempotent fold, dedup key |
| Out of order | keys landed in two partitions | key by entity, sequence carried |
| Drifting view | projection stale and silent | mark as-of, rebuild from the log |

## Real Production Usage

The production failure list is the same storefront every year: "a duplicated payment," "the search was missing an entity," "messages arrived in the wrong order on a reshard." The interviewer stage starts with Kafka as the log, which is durable, but durability is not correctness: it stores the events, it does not ensure the consumer once or in order. Most production teams add the outbox, an idempotent consumer keyed on the event, and per-key routing, and the failures that remain are exactly the ones these three do not cover. A good incident always traces back to a missing one.

## Common Mistakes

1. **The producer assumes the event shipped because the broker wrote it.** A publish that is not ack'd silently drops a purchase, and the outbox is the fix from the first frame.
2. **"The test passed the once."** A local consumer, one message, one order, in order, tells you nothing about the rebalance of ten partitions. The failure state machine lives only in production.
3. **The fix overlays the stream.** You store the projection and then patch it by hand; you broke the single source. Do not both stream and manually repair the same number.

## Interview Perspective

The question is "what breaks an event-driven system" and they want the list bounded. Strong: lost events from a non-acked publish, duplicates from at-least-once, out-of-order from cross partition, and a view that goes stale without the mark. They follow up with "the idempotency key" and "async wiring" and you name: write the event in the same transaction or the outbox, key the consumer by a canonical event id, and keep the sequence with the event, per stream. The line that closes is: the log is durable, the consumer is never exactly-once by default.

## Knowledge Check

- A producer writes to its database, then publishes. Where is the lost event, and why does a durable broker not fix it alone?
- A consumer is at-least-once keys nothing. Which two of the pitfalls above can it hit, and how does one stable event id fix both?
- Events for one order reach a consumer through two partitions, so the view shows `Shipped` before `Placed`. Name two fixes that keep the stream correct.

## Key Takeaways

- Lost, duplicate, out-of-order, and drift are the four, and you plan for each the moment you look at a stream.
- The outbox puts the event in the same transaction as the state change, at the very start of the write.
- Guarantees come from the idempotent consumer and the commit, not the broker; on an order the check is per key.

## What's Next

The whole chapter has operated on events, but nothing yet promised the system keeps working when the software fails. The next chapter is Reliability and Observability, and it changes the subject from how the events flow to how you know they did: the handling of a dead middleware, the idempotency review, and the traces and metrics that show, when an event fails, where.

---

This article explains the common failure set of an event-driven system: lost events, duplicates, out-of-order deliveries, and a drifting read view. It argues that each has a concrete repair, an outbox, an idempotent consumer, a keyed stream, and a marked rebuild, and that most incidents trace to one missing repair, not to the broker.