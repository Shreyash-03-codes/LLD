# Lazy Loading vs Eager Loading and the N+1 Problem

## Learning Objectives

- Explain lazy vs eager loading as fetch-time and fetch-shape choices the ORM makes for relationships.
- Reproduce the N+1 problem from a loop that touches a relationship and tie it back to lazy defaults.
- Apply the two per-query fixes, a join fetch for scalars and an entity graph for collections, and say when each is right.

## Introduction

When an entity holds a relationship to another entity, the ORM has to pick which parts of the object graph to pull and when. Eager means bring it with the parent in one go. Lazy means fetch it on first touch. The choice sounds like a tuning knob, but the lazy default is what turns a normal-looking loop into the N+1 query problem, the most expensive ORM mistake in normal use.

## Problem Statement

A `findAll()` returns a list of orders. The code then walks the list and, for each order, reads the customer's name. Under lazy defaults each name read becomes its own query. A list of 100 orders turns into 1 query for the list plus 100 for the customers. One trivial page is suddenly 101 round trips to the database. On a busy endpoint that number scales with traffic, and the database fills with identical tiny queries that each pay the full latency of the trip for one row. The endpoint is slow, the connection pool is strained, and nothing in the method looked expensive.

## Core Concept

Fetching a relationship is really two decisions bundled: which parts of the graph you load and at what point.

The two strategies describe the timing.

| | Eager | Lazy |
|---|---|---|
| When it loads | with the parent query | on first access |
| What it issues | a join in the parent query, or a second select | one separate select on touch |
| Best for | small children you always walk | children that are big or rarely used |
| Risk | pulls children you never touch | every touch is a new query |

The hidden trap is which is the default. The JPA provider maps `@OneToMany` and `@ManyToMany` as lazy, while `@ManyToOne` and `@OneToOne` often come out eager, depending on the provider. That means your most expensive relationships are exactly the lazy ones, and the small always-attached ones are eager. It is the opposite of what most people intuit, and it is the reason the N+1 shows up precisely where you did not expect it.

### The N+1, named

The name counts the queries. N rows, then one more for each, so N plus 1 total. Below is the shape that triggers it.

```java
List<Order> orders = orderRepository.findAll();
for (Order order : orders) {
    order.getCustomer().getName();   // one SELECT per order
}
```

If `customer` was not loaded with the orders, every `getCustomer()` touch triggers a separate SELECT. Some providers even log a count warning so you can catch it. The remedies, the join fetch and the entity graph, are what comes next.

### The per-query fixes

The naive fix is to make the relationship eager globally. That is mostly wrong: it moves the cost onto every load of the parent, including list pages that never touch the child. The real tools act per query.

For a single association the join fetch is the direct fix.

```java
@Query("select o from Order o join fetch o.customer")
List<Order> findAllWithCustomer();
```

One query, the customer rides along in the same round trip. But when the relationship is a collection, a plain fetch join is a trap. The parent row replicates once per child, and you multiply the result and have to de-duplicate the objects. The controlled tool for collections is an entity graph.

```java
@EntityGraph(attributePaths = {"customer", "customer.orders"})
@Query("select o from Order o")
List<Order> findAllWithGraph();
```

The `@EntityGraph` is eager loading scoped to this one query: attach these paths, in this depth, and leave the default intact everywhere else. It is the tool to reach for when the relationship tree has more than one level.

### When lazy is the honest default

Abandoning lazy loading as a whole is a mistake too. Most relationship paths are never touched by most code, so lazy is the correct default for the general case. The discipline is to choose a fetch plan per access type, then make the database do it in one. A flat list is solved with a join; a deep detail view with a graph. The answer is a per-query behavior, not the flip of a global flag.

### Batch size as the safety net

There is a third mitigation that lands even when per-query planning missed: batch size. Hibernate can fetch the children of many parents in one pass, a `SELECT ... WHERE parent_id IN (...)`, instead of one query per child. `@BatchSize` declares how many children share a round trip.

```java
@Entity
public class Order {
    @OneToMany(mappedBy = "order")
    @BatchSize(size = 25)
    private List<LineItem> items;
}
```

Batch loading is not a substitute for a deliberate join, because it still issues a separate statement for the relation, but it collapses N into N/25. That is often the difference between a page that dims under load and one that does not. Treat it as the safety net that makes an unplanned touch survivable, not as the first tool you reach for.

### The lazy edge case: the session already closed

A lazy relationship can only be loaded while the persistence context is open. The moment an entity is handed to a view layer after the transaction has committed, touching an unfetched collection throws, in Hibernate the `LazyInitializationException`. It shows up most often exactly where you would guess: a page renders an order summary and reaches into a lazy list after the context has closed.

```java
public OrderDto summary(Long id) {
    Order order = orders.findById(id);   // entity loaded, then tx ends
    int items = order.getItems().size(); // throws if items were not fetched
    return new OrderDto(order, items);
}
```

The fix is never to widen the transaction or keep the session alive just to satisfy a fetch. It is to decide what the page needs and load exactly that, with a join or a graph, before the entity leaves the boundary. The two failure modes, the N+1 explosion and a page that throws, are the same oversight wearing two faces: the fetch was left to chance instead of declared.

## Real Production Usage

Spring Data JPA exposes the controls as `@Query` join fetches and `@EntityGraph` on repository methods, and Hibernate will tell you about the count if you turn on query logging. The production instinct is a small number of purpose-built fetch queries for the hot paths, kept explicit, with the rest of the graph left lazy. Real teams ship a handcrafted graph for the heavy detail endpoint and let everything else ride on the default.

## Common Mistakes

1. **Flipping the relationship to `EAGER` to stop N+1.** It moves the cost everywhere, including the frequent page paths that never want the children.
2. **Using a fetch join on a collection without thinking about cardinality.** The rows multiply and you pay to de-duplicate.
3. **Believing the default is uniformly lazy.** When the provider ships `@ManyToOne` as eager, you get eager loading exactly where the impact is invisible.

## Interview Perspective

Interviewers want you to name the pattern and then fix it against a specific graph, not recite the term. Weak: "the fix is to not use lazy loading." Strong: "keep the default lazy, and for the hot read use a join fetch for a single association and an entity graph for a collection, scoped to that one query." The probe that separates them is "what changes when the association is a collection?" A solid answer cites the row multiplication and reaches for the graph instead of a fetch join.

Follow-up: "Would you make everything eager for a small read?" They want you to refuse and point at over-fetching on every path, not just this one.

## Knowledge Check

1. A `findAll` list has 40 orders, each with a lazy `customer`. Your loop reads all 40 names. How many distinct queries hit the database, and what is the shape of the count?
2. Convert that loop to a join fetch on `customer`. Where does the extra per-order fetch go, and why is the collection case different?
3. You set the `items` collection to eager globally. What breaks on the frequent page paths that never read `items`?

## Key Takeaways

- Lazy defers the fetch, eager pulls it now, and the JPA defaults put lazy on exactly the relationships that cost the most.
- N+1 is N parents plus one child query each, and the fix is a per-query fetch, not a global expectation.
- A fetch join for a direct attribute, an entity graph for a collection, and lazy everywhere else.

## What's Next

Even a well-fetched single join still pays a full database round trip per request for data that barely changes. The next article moves the frontier one step closer to the app: caching fundamentals, the second store that sits between the service and the database, and the freshness it trades for speed.

---

This article explains lazy and eager loading and traces the N+1 query explosion to JPA's per-relationship defaults. It argues that a per-query fetch join or entity graph, not a global eager flag, is the real fix, and that the misread defaults are what hide the source of the query storm.