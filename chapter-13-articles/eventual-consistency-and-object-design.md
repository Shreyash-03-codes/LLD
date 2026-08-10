# Eventual Consistency and What It Means for Object Design

## Learning Objectives

- State what eventual consistency is and why an event-driven system forces it.
- Explain what changes for an object when a read can trail the source of truth.
- Apply the design that makes staleness tolerable, a projection rebuilt from the event stream.

## Introduction

Every model so far held a promise: after a write, a read sees the result. Event-driven design sets that promise aside. Eventual consistency means the write lands and every reader sees it only at some later moment, not at the instant of the write. It is not a failure and not a bug, it is the honest price of decoupling the write from the read. The hard part is not the storage; it is the object. You design a service and a class that can serve a slightly old view without treating it as a crash.

## Problem Statement

A payment service publishes `OrderShipped`. A billing service consumes it and updates its copy, but a customer opens the order page a fraction of a second before the billing service catches up. The page shows pending while the shipping team already marked it shipped. Two reads of the same order, moments apart, disagree. The order page has no single copy, each part of the flow is its own projection, and the front end is showing one that has not caught up. The failure is not the event pipeline. The failure is an object that served a stale view as though it were the truth.

## Core Concept

Eventual consistency is the property that the writes reach every reader, eventually, with no promise that they all agree at once. The alternatives, a strongly consistent store that answers every read from one synchronized view, is slower and couples every reader to that view. Event-driven systems choose eventual because the write and the readers live in separate stores. Each consumer keeps its own read model, and that model converges only after the event has been consumed.

The consequences reshape the object you design:

- **A read model may be behind.** A projection's contract is not "the truth," it is "true as of a time." It must carry that time.
- **There is no single canonical copy.** Until the consumer catches up, several copies of "order state" exist, each a different moment in the fold.
- **The read model is derived, not stored.** It is a function of the events, so stale is not corrupt, it is behind.

That last point is the idea worth keeping. A read model is `state = fold(events)`. The billing view is the result of folding `OrderPlaced` and `PaymentCaptured` in the order they arrived. When a consumer lags, the view is not wrong, it is a projection over fewer events. You repair it by replaying, not by patching data.

```java
public class OrderProjection {
    private final Map<String, OrderSummary> byId = new HashMap<>();

    public void apply(OrderEvent event) {
        if (event instanceof OrderPlaced o) {
            byId.put(o.orderId(), new OrderSummary(o.placedAt()));
        } else if (event instanceof PaymentCaptured p) {
            byId.get(p.orderId()).recordPaid(p.amount());
        }
    }
}
```

The object-design lessons that fall out:

- Treat the events as the record and the read model as a derived view. Do not hand the projection out as the source of truth.
- Make the fold idempotent. At-least-once delivery replays events, and replaying a charge that adds to a running amount double counts. A `recordPaid` that compares against a carried marker is safe to call twice.
- The projection can be rebuilt any time from the log, which means you can let it lag and catch it up at your own pace instead of fearing the lag.

The practical tension is ordering. Events can arrive out of order from different keys, so a projection often waits, or its consumers producers key the stream per entity. The object must not assume the events arrive in the order the story was told.

## Real Production Usage

A projection is the engine of real reads. A search index and a reporting store are projections, each folding the same event stream differently. A bank-style balance is an "as of" number, and the authoritative ledger is the log that produced it. Reconciliation works because the projection is derived and the log is the source: when the two disagree, the fix is to replay the projection, not to edit the store by hand. And the whole value of eventually consistent reads is real: reads that can be a little stale are reads that can cache, scale, and be served by a parallel store, which is the actual speed of the design.

## Common Mistakes

1. **Serving a stale projection as the truth.** A view that does not say "as of 2s ago" surfaces the drift to the user as a bug.
2. **A non-idempotent fold.** At-least-once replays a `PaymentCaptured` and the ledger double counts. Every apply must be safe to run twice.
3. **Trusting the projection as the source.** Once the projection is edited outside the fold, the log and the view diverge, and nothing can repair the drift as well as that.

## Interview Perspective

Interviewers ask "eventual consistency, how do you live with it" to separate the phrase from the mechanism. Strong: "the event is the source and the read is a projection rebuilt idempotently from the stream; the view marks its own as-of time, and I repair drift by replaying." Weak: "it is always eventually and the system pays." Follow-up: "when do you need strong read" the answer names both: a balance and a transfer force strong, a feed and a report admit eventual. The note to say is that you pick which window matters, because eventual is a choice, not a surprise.

## Knowledge Check

- A page shows `OrderShipped` as still pending, a moment after publish. Is that a bug in the code, and what marks the difference between a stale view and a corrupt one?
- A projection folds `PaymentCaptured` twice on a replay and ends with two charges. What changes, and how does an idempotent `apply` fix the replay?
- A search index and a ledger consume the same events. Why is a rebuild a normal operation for both, and what does that say about where the truth lives?

## Key Takeaways

- Eventual consistency means a read is a view of a moment in the stream, and the view must carry its own as-of time.
- A read model is a function of the event log, folded idempotently and rebuilt by replay.
- Keep the event log as the record and the projection as a derived view; chosen per-answer, not a global bug.

## What's Next

Event sourcing makes that fold the primary record. The next article keeps state as the event log, the source of truth, and reconstructs the aggregate by folding, which is the formal version of the projection idea this article built by hand.

---

This article explains eventual consistency as the moment between an event and the projections that consume it, and shows how to design the read side so that lag is safe. It argues that the read model must be an idempotent, rebuildable fold over the event log, marked with an as-of time, rather than a snapshot that pretends to be the truth.