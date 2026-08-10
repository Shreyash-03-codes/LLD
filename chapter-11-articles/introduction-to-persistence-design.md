# Introduction to Persistence Design

## Learning Objectives

- Explain why persistence is a design problem, not just a JDBC call, and enumerate the decisions a persistence design actually makes.
- Distinguish the storage engine from the data-access layer and see where your code's responsibilities begin and end.
- Map persistence choices (relational, key-value, document, columnar) to concrete workload shapes, not to hype or familiarity.

## Introduction

Persistence is where most designs quietly go to die. You can nail the domain model, the patterns, the clean concurrency, and then watch it all rot at the boundary where objects meet a database. The reason isn't that persistence is hard. It's that most engineers never treat it as a design problem. They treat it as a glue problem: "I need to save this object, so I'll write a repository and call it a day."

That framing is the whole mistake. Persistence design is a set of deliberate decisions about shape, ownership, integrity, and timing. Every one of those decisions has a cost, and most of them were made by default because the engineer copied a pattern from a previous project. This chapter is about making them deliberately.

## Problem Statement

Concrete failure: a team builds a service with an anemic `saveOrder(Order order)` method backed by a single generic JPA repository they wrote on day two. The application grows. Now a `Order` must be written only if a nested `LineItem` sum matches the total, only when the user isn't in a throttled state, and only in a way that doesn't clobber a concurrent refund. The team's answer to each new constraint is another raw query or another if-statement in the controller. Six months later the persistence layer is a pile of scattered conditionals, and nobody on the team can say whether a write is atomic, consistent, or even going to the same database.

That pile is the symptom of never deciding what persistence is supposed to do. The saving itself was correct; the design around it was absent.

## Core Concept

Persistence design splits into three layers that have very different rates of change.

| Layer | What lives here | Rate of change |
|-------|-----------------|----------------|
| Storage engine | The database itself: relational, key-value, document, columnar | Almost never |
| Access layer | Repositories, DAOs, ORM mappings, query definitions | When the data model changes |
| Domain layer | Entities, business rules, invariants | Frequently, with application logic |

Most bad persistence design is really a leak between these layers. Business rules show up inside a JPQL string. Storage-engine facts show up in a repository method name. That's the thing to fight.

The storage-engine decision comes before all the interesting patterns. There is no "best database"; there is only "best fit for the access pattern." The common mistake is to pick a relational database by default because it's the safe answer, then stretch it to support document-ish or event-ish shapes it fights. Match the store to how the data is actually read and written.

- Relational: data is normalized, highly interrelated, and queried in joins across many entity types. Transactions across tables matter. Most OLTP business systems land here.
- Key-value: you access everything by one key and the shape varies. Drop Redis or DynamoDB and you free yourself from schema migration for the hot paths.
- Document: you read or write whole logical chunks together, and you almost never want to stitch pieces across collections. Logs, profiles, JSON payloads.
- Columnar: you aggregate over huge volumes with a fixed column set. Analytics, not OLTP. Never for row-level inserts at scale.

Notice what's missing from that framing: "which has an ORM" is not a criterion. Whether Spring Data supports it is a footgun, not a feature requirement. The repository pattern from article two is meant to keep these decisions swappable, but swapping is the exception, not the norm. A real design picks a storage engine early and unapologetically.

### The mixed workload is the hard case

Most real systems are not one store. A single service often has a relational core for the facts that must transactionally join, a key-value store for the session and the short-lived lookup, and a cache in front of a hot render. The discipline is that each piece is a separate decision, each takes its own shape, and the boundary between them is explicit rather than accidental.

The failure worth naming is the instinct to put everything in one relational store because then it will be easy to find later. That instinct builds the cheapest schema and then the slowest one, jamming JSON blobs and event streams into tables that fight them. "One Postgres for everything" is not a relational decision; it is one store for every shape, which is the opposite.

### Reading a workload into a store

The skill has a tight shape. Ask first how the data is actually consumed. If reads join and aggregate across entity types and the writes must be atomic across tables, relational. If every access is one key and the writes can be lossy under pressure, key-value, with a cache in front. If a logical chunk is written and read whole, document. If you sweep enormous fixed-shape columns, columnar. The table of the store follows the answer to "what does a single read look like and how does it relate to other reads."

### Ownership of the write

The clearest mindset shift in persistence design is treating writes as needing ownership. When two pieces of code both write the same rows under different assumptions, the invariant that made sense to one author breaks the other. You end up with a half-saved cart, a status field that skipped two states, or a refund and a charge that both think they won.

The fix isn't a clever database feature. It's deciding, explicitly, who owns the consistency rule.

```java
public class OrderService {
    private final OrderRepository orders;
    private final AccountRepository accounts;

    @Transactional
    public void placeOrder(Order order, Account account) {
        order.place();                       // rule: order only moves to PLACED if payable
        account.debit(order.total(), "order-" + order.id());
        orders.save(order);
        accounts.save(account);
    }
}
```

In this snippet the transactional boundary and the repository calls are visible in prose with the added words "// rule:". Notice what matters: the invariant is written in one service method, in one transaction, so an exception in either save rolls both back. That is the persistence contract. The two separate `save` calls only look independent. The service is the owner, and the transaction guarantees the pair acts as a unit.

## Real Production Usage

You can see the storage-shape decision in the real world everywhere. Kafka is chosen for streams of events that are append-heavy and partitionable by key, not for relational lookups. Redis is a key-value store that people abuse as a source of truth when they should use it as a cache, which creates a whole class of consistency bugs covered in the caching articles. Hibernate via Spring Data JPA is a relational ORM that many real services rely on, and the reason it gets a whole chapter is that its defaults (lazy loading, dirty checking) actively change the behavior you did not ask for.

The first skill those real systems share is that they name the storage engine for its actual shape, then let the access layer follow.

## Common Mistakes

1. **Treating persistence as "the ORM."** Because you use Spring Data JPA, the incorrect assumption is you did persistence design. You did not. You chose a driver. The design decisions around transactions, locking, and caching are still yours.
2. **Using relational stores for unrelated shapes.** Every graph of data, a JSON blob, an event stream, all jammed into tables with a giant `metadata` column. It works at ten rows and breaks the join model forever.
3. **Putting read logic in the entity.** An entity that calls the repository inside itself now mixes a UX-level fetch rule into the domain. The rule should live in the service.

## Interview Perspective

Interviewers usually zoom in on persistence because it's where abstract answers stop being enough. The weak answer to "how do you decide between a relational store and a key-value store?" is a shrug or "depends." The strong answer names the workload: "If reads aggregate and join across entity types, relational; if every read is a single-key lookup and writes can be lossy-under-pressure, a KV store, and my cache sits in front." They want shape leadership plus the readiness to trade durability for speed.

Common follow-ups: "What does a transaction mean in a KV store?" and "If I tell you the cache invalidation is honest, does the design change?" You can lean on the ownership-of-writes idea from the core concept here.

## Knowledge Check

1. Classify "user profile that is read forty times per read and edited rarely" into a store and defend the trade.
2. A team writes a business rule (an order can only be cancelled while `status == HOLD`). Where does that rule belong: entity, repository, service?
3. Look at the `placeOrder` snippet. What would the difference be if the two saves were in two transactions?

## Key Takeaways

- Persistence is decisions about storage shape, write ownership, and timing; choosing a driver is not the same as designing persistence.
- Match the storage engine to how the data is actually read and written.
- Keep business rules out of the query layer and out of the entity.

## What's Next

This chapter is the first chapter about storage. The next article goes straight into the repository pattern and what Spring Data JPA actually does when it creates a repository for you. That article is where the "save" method you kept hands-waving about stops being a magic incantation and becomes a contract you can reason about, including where the JPA provider quietly inserts caching, dirty checking, and query generation around your interface.

---

This article explains persistence as three layers, storage engine, access layer, and domain logic, and why most bad persistence is a leak between them. It argues that calling the database's shape is the real design decision, not the framework you used to get there.