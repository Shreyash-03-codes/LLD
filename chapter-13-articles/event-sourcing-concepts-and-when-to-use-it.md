# Event Sourcing: Concepts and When to Use It

## Learning Objectives

- Explain how event sourcing makes the event log the source of truth and the state derived.
- Reconstruct an aggregate's state by folding its event stream.
- State the costs, replay, versioning, and two stores, that make it a design choice and not a default.

## Introduction

The last article rebuilt a projection from an event log. Event sourcing takes that and promotes it to the whole application: you store the events, not the current state, and the current state is whatever folding the history produces. An account is not a balance field with a value; it is the sequence of `Deposited` and `Withdrawn` events, and the balance is a function of them. The radical claim is that the store keeps the history and the state is a derived artifact. That idea buys you auditing and time travel at the price of extra machinery.

## Problem Statement

A system stores only the current balance. A support agent asks for an order's state six months ago and there is nothing to show, only the number that stands now. An auditor wants to know why a charge was voided, and the voided entry is gone. When an account error surfaces, the fix is a bare `UPDATE balance = balance - 10` that no one can ever explain again. The design keeps the current value and throws away the history, and for a business that owns the history that is storage of the answer, not the data.

## Core Concept

Event sourcing captures every change to an aggregate as an append-only event. You never update an account; you append `Withdrawn(amount)`. State is a fold of the log, and you can reproduce it at any point in the past by replaying the events up to that point.

```java
public class Account {
    private final List<Event> changes = new ArrayList<>();
    private long balance;

    private Account(List<Event> history) {
        history.forEach(this::apply);
    }

    public static Account replay(List<Event> history) {
        return new Account(history);
    }

    public void withdraw(long amount) {
        if (balance < amount) throw new InsufficientFunds();
        append(new Withdrawn(amount));
    }

    public void apply(Event e) {
        if (e instanceof Withdrawn w) balance -= w.amount();
        if (e instanceof Deposited d) balance += d.amount();
    }

    private void append(Event e) {
        apply(e);
        changes.add(e);
    }
}
```

The engine of event sourcing is the read, not the write, staying split. A command arrives at the aggregate, which folds its history to know the current state, validates, appends the new event, and appends that event to the durable log. The state is never the primary record; it is materialized in memory only when the aggregate runs.

On the read side, a projection folds the same log into whatever view the system serves. That projection is rebuildable, it is just a fold, which already proved itself in the previous article. The two arrows, append and fold, are the whole model.

Diagram: a command validates on the aggregate, appends to the event log, and a projection folds the log into a read model.

<svg width="920" height="340" viewBox="0 0 920 340" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="es" markerWidth="10" markerHeight="10" refX="8" refY="4" orient="auto">
      <path d="M0,0 L8,4 L0,8 Z" fill="#333"/>
    </marker>
  </defs>

  <rect x="40" y="120" width="150" height="60" rx="6" fill="#eef2f7" stroke="#333" stroke-width="2"/>
  <text x="115" y="148" text-anchor="middle" font-family="Arial" font-size="14" fill="#222">Command</text>
  <text x="115" y="168" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">place order</text>

  <path d="M 190 150 L 262 150" stroke="#333" stroke-width="2" fill="none" marker-end="url(#es)"/>
  <text x="226" y="140" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">validate</text>

  <rect x="262" y="110" width="150" height="80" rx="6" fill="#fdf6e3" stroke="#8a6d00" stroke-width="2"/>
  <text x="347" y="146" text-anchor="middle" font-family="Arial" font-size="14" fill="#222">Aggregate</text>
  <text x="347" y="168" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">state in memory</text>

  <path d="M 442 150 L 514 150" stroke="#333" stroke-width="2" fill="none" marker-end="url(#es)"/>
  <text x="478" y="140" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">append</text>

  <rect x="514" y="40" width="180" height="110" rx="8" fill="#fdf6e3" stroke="#8a6d00" stroke-width="2"/>
  <text x="604" y="76" text-anchor="middle" font-family="Arial" font-size="14" fill="#222">Event log</text>
  <text x="604" y="100" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">append only</text>
  <text x="604" y="120" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">source of truth</text>
  <text x="604" y="140" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">history kept</text>

  <path d="M 604 205 L 604 246" stroke="#8a6d00" stroke-width="2" fill="none" marker-end="url(#es)"/>
  <text x="628" y="230" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">fold</text>

  <rect x="514" y="246" width="180" height="60" rx="8" fill="#eef7ee" stroke="#2f6f3e" stroke-width="2"/>
  <text x="604" y="276" text-anchor="middle" font-family="Arial" font-size="14" fill="#222">Read model</text>
  <text x="604" y="296" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">projection</text>
</svg>

The strengths are real. You have a complete audit, every change, with who and when. You can answer "what was the balance in March" by folding the log to that point. And the read model can be a fresh fold that any new consumer builds from scratch by replaying from the start.

## What it costs

The first bill is two stores. The log and the projection are separate, so a write to the log is not instant in a read model. The projection lags and the eventual-consistency machinery of the last article is required. The second bill is replay. Restarting an aggregate, or a new projection, replays the history, and a long-lived system replays a great many events on boot. The third bill is schema forever. An event is written once and read for all time, so every consumer keeps reading v1 forever; event schemas do not migrate cleanly, they need versioning and careful compatibility.

## Real Production Usage

Event sourcing is a surgical fit for a ledger, an audit trail, a balance and history as the product: banking, an order workflow that must answer "what happened and in what order from a regulator," Axon, and Kafka with an outbox, are how teams in Java ship it. The line that decides is whether history is the product. If the system must reproduce exactly what happened and why, the design pays. If the front end edits a few fields of a record and the current value is all that matters, event sourcing is a tax you pay for an audit nobody asked for, and a plain store is right.

## Interview Perspective

The question "event sourcing, what does it cost" is whether you know the trade, not the buzzword. Strong: "event sourcing makes the log the truth and the state derived, buys an audit and temporal queries, and it costs a projection that lags, a two-store system, an event-schema never changes quietly, and a replay cost." Follow-up: "when would you not use it" and the strong answer: "when the current state is the whole claim, when replay is heavy, or when you need no history, skip the machinery." They are checking that you reach for it, because the audit is the point.

## Knowledge Check

- The `Account` above holds only the latest balance. How do you answer "what was the balance after the third deposit," from the log alone?
- A store replays twenty thousand events to map one aggregate. Why must that be expected, and what options make the boot cheaper?
- Projection lags after a write to the event log. Is that a bug or the design, and what chapter's idea keeps it from being a crash?

## Key Takeaways

- Event sourcing stores the events and derives the state by folding, giving an audit and any past point free.
- It pays two stores, a replay on boot, and a schema that versions forever.
- Use it when history and the audit are the product; skip it when the current value-alone store is the answer.

## What's Next

Event sourcing splits the writer, the log append, from the reader, the projection, on purpose. The next article draws that boundary by design. CQRS, command query responsibility segregation, asks why both sides need one shape at all, and pairs with event sourcing to serve each read from its own model.

---

This article explains event sourcing as making the event log the source of truth and the current state a derived fold of it, which buys a complete audit and temporal queries. It argues that the pattern is right for a ledger where history is the product, and a genuine tax where the current state is all that matters.