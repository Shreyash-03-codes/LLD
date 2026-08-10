# Repositories in Domain Design

## Learning Objectives

- Define a repository as the domain-facing collection of an aggregate, an interface that loads and saves whole aggregates without exposing the database.
- Explain the dependency direction: the domain depends on a repository interface, and the persistence adapter implements it.
- Keep repositories small and aggregate-shaped, and resist turning them into a leaky general-purpose data access layer.

## Introduction

A repository is how the domain asks for an aggregate and puts it back. The `TransferService` from the last article did not know how accounts were stored; it called `accountRepository.find(from)` and got an `Account`. That `AccountRepository` is the repository, and it is the boundary between the domain model and the database.

This article is about that boundary. A repository hides the storage so the domain never sees a `Session`, a `Connection`, a `ResultSet`. The domain talks in aggregates; the repository translates those aggregates to and from the database, in whatever storage the system chose.

## Problem Statement

The naive approach is to let the domain talk to the database directly. Every domain operation that needs an order calls the data layer, or worse, holds a `Session` and runs queries inline:

```java
public Money totalRevenue(LocalDate day) {
    EntityManager em = entityManager;
    Query q = em.createQuery("select o from Order o where o.createdDate = :day");
    // ...
}
```

That fails in three ways.

First, the domain learns the database. The `Order` class is no longer persistence-ignorant, the property that the introduction made the whole model stand on. Every operation that needs persisted data now depends on the query API, so the model cannot be tested, run, or reasoned about without a database.

Second, the queries scatter. The same "load an order with its lines" appears at every call site, custom query text every time, and a change to how orders are stored, renaming a column, changing the fetch strategy, forces a change in every caller.

Third, the storage shape leaks. If an order must be loaded with its lines fetched eagerly, the distinction between "load the aggregate" and "load part of the aggregate" is decided by whatever query the caller happened to write. The domain cannot know what it got, and a rule that touches the whole aggregate can be handed a half-loaded order in production.

## Core Concept

A repository is an interface in the domain that represents the aggregate as a whole. It is written in the language of the domain, loads whole aggregates, and knows nothing about how they are stored.

```java
public interface OrderRepository {
    Optional<Order> findById(OrderId id);
    List<Order> findByCustomer(CustomerId customer, LocalDate on);
    void save(Order order);
}
```

The domain codes against this interface. A service that needs an order by id calls `OrderRepository.findById` and gets an `Optional<Order>`. It never names a table, a column, a query. It does not know whether the order lives in a relational database, a document store, or a cache behind the database.

The concrete implementation lives on the persistence side and implements the interface; in production it is typically a Spring Data JPA repository, a hand-written one over the session, or a mapper over a cursor.

```java
@Repository
public class JpaOrderRepository implements OrderRepository {
    private final JpaRepository<OrderJpa, Long> jpa;

    @Override
    public Optional<Order> findById(OrderId id) {
        return jpa.findById(id.value())
                .map(OrderJpa::toDomain);
    }

    @Override
    public void save(Order order) {
        jpa.save(OrderJpa.from(order));
    }
}
```

The `JpaOrderRepository` implements the domain interface and translates to and from the persistence entities, `OrderJpa` here mapping to the `Order` domain object. The direction of the dependencies is the important part: the domain defines `OrderRepository` and knows nothing of `JpaOrderRepository`; the infrastructure knows both and implements the domain's contract.

Diagram: the repository inverts the dependency

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 920 440" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#33475b"/>
    </marker>
  </defs>

  <text x="120" y="30" font-size="12" font-weight="bold" fill="#1a2733">Domain</text>
  <rect x="40" y="45" width="220" height="80" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="150" y="79" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">Order</text>
  <text x="150" y="101" text-anchor="middle" font-size="11" fill="#5a6b7a">aggregate root</text>

  <rect x="420" y="45" width="220" height="80" rx="10" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="530" y="79" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">OrderRepository</text>
  <text x="530" y="101" text-anchor="middle" font-size="11" fill="#5a6b7a">interface</text>

  <text x="120" y="240" font-size="12" font-weight="bold" fill="#1a2733">Infrastructure</text>
  <rect x="420" y="200" width="220" height="80" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="530" y="234" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">JpaOrderRepository</text>
  <text x="530" y="256" text-anchor="middle" font-size="11" fill="#5a6b7a">implements</text>

  <rect x="420" y="330" width="220" height="70" rx="10" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="530" y="358" text-anchor="middle" font-size="11" fill="#1a2733">Database</text>
  <text x="530" y="380" text-anchor="middle" font-size="11" fill="#5a6b7a">SQL</text>

  <line x1="530" y1="125" x2="530" y2="196" stroke="#9aa7b4" stroke-width="1.5" stroke-dasharray="5 4"/>
  <text x="562" y="165" text-anchor="middle" font-size="10" fill="#5a6b7a">implements</text>

  <line x1="530" y1="280" x2="530" y2="326" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
</svg>
```

The top is the domain: the `Order` aggregate and the `OrderRepository` interface it depends on. The bottom is the infrastructure: `JpaOrderRepository` implements the interface (dashed line) and talks SQL to the database. The critical arrow is the dashed one, and its direction, up, is what makes the dependency inverted: the infrastructure depends on the domain's interface; the domain does not depend on the infrastructure. The domain does not even know `JpaOrderRepository` exists.

The rule that follows: the domain references the interface, the infrastructure implements it, and the two are bound by a framework at the composition boundary. In Spring this is autowiring the concrete bean behind the interface; the domain sees the interface and the container hands it the implementation.

### What the repository is

A repository is a conceptual collection of an aggregate. It is not a general-purpose data access class. It exposes operations that mirror what the domain needs, `findById`, `save`, maybe `findByCustomerAndDate`, and nothing more. The moment a repository exposes the persistence concept, returning a `Page`, a `Query`, a detached proxy, it has stopped being a domain repository and started being a data access layer wearing the name.

### Two properties to honor

First, the repository returns whole aggregates. Because an aggregate is loaded as a consistency unit, `findById` returns an `Order` with its lines and value objects intact, ready for its rules to run. The repository does not offer a way to load a single line.

Second, the repository knows nothing about the domain's rules and the domain knows nothing about the storage. The repository asks the persistence layer for the rows and translates them; it never reimplements a domain rule, and the domain never issues a query. That translation is the only responsibility.

## Real Production Usage

Spring Data JPA is the pattern's production home. Defining an interface such as `interface OrderRepository extends JpaRepository<Order, Long>` gives the aggregate a Spring-managed persistence, and the framework derives query methods from the method names. In well-arranged systems the domain `OrderRepository` is the interface in the domain package and the concrete Spring bean implements it on the other side of the boundary, which is the dependency inversion in code.

The production lesson, from real codebases, is about fidelity to the aggregate. When a repository returns a lazily-loaded entity and the session closes before the rule runs, the aggregate rule throws `LazyInitializationException` in production but passes in tests. The discipline is the repository loads the whole aggregate eagerly, and the boundary exposes that completeness, so every rule has access to everything inside its aggregate.

The second production lesson is the query that no longer queries. The repository hides the storage, and in practice the same interface can move from PostgreSQL to a cache-backed implementation, or behind a write-through cache, without the domain changing. That is the payoff: the domain is indifferent to the database, and the database can be changed underneath it.

## Common Mistakes

**Letting the domain call the persistence.** Inline `EntityManager` in a domain operation, or a service importing a `Session`, defeats the boundary. The domain must hold an interface, and the framework wires the concrete repo at the edge.

**Turning the repository into a general-purpose data layer.** A repository method that accepts a query, returns a detached entity, or exposes the raw `Query` leaks persistence into the callers. The repository belongs to an aggregate, and its methods belong to that aggregate.

**Loading part of the aggregate.** A rule that needs the lines must be satisfied by a repository call that returns the lines, or the lazy loading breaks the root and throws. The repository is aggregate-shaped and returns the whole for any find.

## Interview Perspective

Interviewers ask about repositories to test whether you understand dependency inversion and the boundary. A weak answer says "the repository is the class that talks to the database." A strong answer says the repository is an interface in the domain that represents the aggregate as a collection, the infrastructure implements it, and the direction of the dependency is what keeps the domain from knowing the database.

The follow-up that sorts people is "can the domain talk to the database directly?" The strong answer says no, the database is behind the repository, the domain depends on the interface, and that is what makes the model persistence-ignorant and testable. A second follow-up is "what happens when the repository returns a lazily loaded aggregate?" and the strong answer names the proxy and the hidden coupling in the real order operation.

Common follow-ups:

- "Which side knows which: does the domain know the repository name, or the repository the domain?"
- "Your `OrderRepository` exposes a method that runs any query. What have you created?"

### Why repositories exist

It is worth being blunt about the reason. A repository is not a nicer name for a DAO. A DAO is a data-access class that knows how to read and write rows; a repository is an aggregate's collection that the domain calls, with the storage hidden. When a change to the storage, the schema, or the cache forces a change in the domain classes, the boundary is absent. The domain carries its own invariants and the repository carries the storage, and the two never bleed into each other.

## Knowledge Check

1. The domain's `OrderRepository` is an interface, and `JpaOrderRepository` implements it. Which side knows about which, and what does that direction make possible?
2. A rule needs an order's lines to be present so its rules hold. What does the order require from its loader, and what breaks if the loader returns a proxy over a closed session?
3. A new requirement moves the orders behind a distributed cache. What must change, the domain or the `JpaOrderRepository`, and what stays unchanged?

## Key Takeaways

- The repository is an interface in the domain that represents the aggregate as a collection, and the infrastructure implements it.
- The domain depends on the interface; the infrastructure depends on the domain's contract, so the dependency is inverted.
- Repositories are aggregate-shaped: find methods return whole aggregates and accept no raw queries.
- The domain never names a table, a query, or a session.
- A change of storage leaves the domain unchanged, and the domain is mocked across the interface.

## What's Next

The next article is about modeling business rules. This chapter has built the skeletons, entities, value objects, aggregates, services, and repositories, and the next one is where those objects start carrying the rules that make the model worth having. We will cover where a rule lives, how much to state it as code, and how the business says them in the model.

---

This article explains the repository as an interface in the domain representing an aggregate as a collection, with the persistence implementation behind it. It argues that the domain depends on the interface and never the storage, and stays testable without a database.