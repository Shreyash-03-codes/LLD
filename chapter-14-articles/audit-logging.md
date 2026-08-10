# Audit Logging

## Learning Objectives

- Distinguish an audit trail from a debug log: an audit record is evidence of who did what, when, and why, and it cannot be repaired after the fact.
- Decide what belongs in an audit event, who may write it, and where it lives separately from the application log.
- Argue for tamper-evidence and retention in terms a compliance person needs, not just what a developer finds convenient.

## Introduction

Audit logging serves a different reader than debugging logs and obeys a different contract. A debug log is for a developer who wants context about a failure. An audit log is evidence: who accessed, changed, or deleted what, with the actor and the moment. The two goals push engineering in opposite directions. Debug logs favor volume, context, and speed and trade longevity for convenience. Audit logs favor completeness, order, and non-repudiation; a missing audit record is indistinguishable from a coverup.

## Problem Statement

A customer dispute lands with support: an order was refunded, but the customer says they never requested it and the agent has no action on record. The team searches the debug logs. There is an INFO line about a refund with no actor id, no correlation to a session, and no outcome. Two people were logged in at that moment, but nothing says who performed the refund. The system's current state says refunded, but the history of how it got there is gone, because nobody recorded the causal trail. This is the exact case where an audit log is the entire answer, and where the debug log fails completely.

## Core Concept

An audit event is a small, complete, immutable record: who (the actor), what (the action), on what (the subject), when (a timestamp), and the outcome. The word immutable matters. You write the event once, append it, and never modify it. The audit log is a chronological, tamper-resistant history of the specific actions the business decided were auditable. It is not a larger debug log; it is a different data set with a different warranty. If a record is missing from a debug log, you shrug. If a record is missing from an audit log, you cannot answer the question it was meant to answer.

### Who writes an audit event

The audit record is written by the service that owns the domain state, in the same work unit as the state change. The habit of logging audit facts from a controller or a UI layer breaks this: the controller can confirm the request arrived but not that the state changed. The service that mutated the state is the only place that can guarantee "state changed implies audit recorded." That invariant is the single most important property of an audit trail, and the way to hold it is to put both writes in one transaction.

```java
@Service
public class RefundService {
    private final RefundRepository refunds;
    private final AuditRepository audit;

    @Transactional
    public void refund(RefundRequest req, String actor) {
        Refund refund = refunds.start(req.paymentId(), req.amount());
        audit.append(new AuditEvent(
            "refund.requested",
            actor,
            refund.getId(),
            Map.of("amount", req.amount(), "reason", req.reason(), "outcome", "PENDING_APPROVAL")
        ));
    }
}
```

Both writes commit or roll back together, so you never get a refund with no audit record or an audit record for a refund that never existed. The alternative, an out-of-band async writer, reopens the gap, and the gap is where the dispute lives.

### Tamper evidence, and why append-only alone is not enough

Append-only is not tamper-evident. A table with no UPDATE in application code is still rewritable by an operator who has database write access, and a truncate or a delete leaves no trace. An audit trail only earns the name when rewriting it is expensive or visible, and that comes from outside the normal application stack.

The options, in rough order of strength:

| Mechanism | What it does | Caveat |
|-----------|--------------|--------|
| Immutable store | write-once storage: an append-only file store, WORM, a dedicated audit service | needs an operator that cannot rewrite it |
| Hash chain | each record carries the hash of the previous | rewriting one record breaks every successor, but only if you verify against a kept integrity value |
| External watchdog | an independent collector copies events off the production path | requires a second system that can compare |
| Digital signature | the writer signs each record, key held elsewhere | strongest, costs key management and rotation |

The hash chain is the cheapest thing that works with existing storage. Each record stores the hash of the record that came before it, so altering event N requires rehashing everything after it, and a validator that keeps only the last hash can detect the break.

```java
public record AuditEvent(
    String id,
    String type,
    String actor,
    String subject,
    Instant at,
    Map<String, Object> data,
    String prevHash,
    String hash) { }
```

None of this replaces access control; it complements it. You still restrict who can write, and you make the write provable, so that a breach or a mistake is discoverable instead of erasable.

### Retention is a designed trade, not a default

Debug logs have a short TTL and everyone is fine with that. Audit logs carry a statutory or business retention period, often years, and the policy is set by compliance, not by an engineer's storage preferences. The engineering tension is that long retention costs storage and makes the data painful to query. The common shape is a hot store that serves the last few months of interactive queries and a cold archive for the rest, with the archive still searchable through an export or a read path. The decision to be explicit: how long events live, where they live when hot, and how they are queried when archived. A retention period that is "we keep everything" is a budget you will be asked to justify.

### What to audit

Teams miss in both directions: audit everything until the store is a cost center, or audit so little that the same dispute is unanswerable. A practical starting list:

- Access boundary events: logins, session starts, password changes, lockouts.
- Privilege changes: promotions, demotions, suspensions, role grants and revocations.
- Money and durable effects: refunds, credits, payouts, transfers, cancellations.
- Reads the business cares about: account statements, data exports, dispute reviews.
- PII access: any read or export of personal data, because regulators ask.
- Admin actions: an admin who reads or edits a user's record is an event, even when the outcome is benign.

The gate is simple: audit the events a human will later be asked to explain. Sometimes the correct explanation is "nothing unusual happened," and the audit log is what proves it.

## Real Production Usage

Audit and authorization usually live near each other. In Java, Spring Security publishes authorization events that a listener can turn into audit records for access failures, and the database is the natural home for the records because the same transaction can carry both. Many teams write audit events to an append-only Kafka topic, which gives ordering, retention, and replay, but a topic is not tamper-evident by itself because anyone with partition write access can rewrite history, so the hash chain or the external copy is still required. A database audit table with INSERT-only grants plus a periodic hash-check job is the pragmatic middle. Frameworks like Axon and dedicated audit products exist, but for most services the discipline is: same-transaction write, a structure, a hash, and a retention policy.

## Common Mistakes

1. **Writing audit records at the controller or filter layer.** The controller sees the request, not the state change. The audit belongs in the transaction that changes state; otherwise you record intent and miss the effect.
2. **Believing append-only means tamper-proof.** Append-only resists the application layer, not an operator with database privileges. Tamper evidence is a separate mechanism, hash chain, signature, or external copy.
3. **Treating retention as a default.** "Keep everything" is not a policy; it is an unfunded budget. A deliberate hot/cold split with an explicit retention period is the design.

## Interview Perspective

The question is "how do you design an audit trail?" The weak answer is "we log everything." The strong answer: "I write the audit event in the same transaction as the state change, with an immutable event that carries actor, subject, action, timestamp, and outcome, store it in a place that cannot be silently rewritten, hash-chain it or keep an external copy, and set an explicit retention period." Interviewers check that you know who the actor is, that you hold the same-transaction invariant, and that you do not mistake append-only for tamper-evidence.

Follow-ups: "how do you make it tamper-evident" wants the hash chain plus a validator, and "who can write to it" wants the answer "not the same identity that runs the domain code."

## Knowledge Check

1. An operator has direct database write access and can edit user rows. What do you do, beyond restricting privileges, so that an edit is detectable?
2. State change and audit record are in one transaction that rolls back. What is guaranteed, and what is the loss if the two were separate writes?
3. Compliance asks "did anyone export Client X's data in December?" What fields must a record carry, and what retention design makes the question answerable years later?

## Key Takeaways

- Audit events are written by the owner of the change in the same work unit as the change, so the invariant "state changed implies audit recorded" holds.
- Every event carries actor, action, subject, timestamp, and outcome.
- Append-only is not tamper-evident; the hash chain, signature, or external copy is. Retention is a designed hot/cold trade, not a default.

## What's Next

The audit record exists, but it is still passive: nobody is watching it until a question arrives. The next article turns records into signal, monitoring and metrics design, which is how you decide what to count, how to name the counts, and how to turn a threshold into a page before a customer notices.

---

This article explains audit logging as a tamper-resistant record of who did what and when, written in the same transaction as the state change. It argues that append-only alone is not tamper-evidence, and the same-transaction invariant is what makes a trail defensible.
