# DAO vs Repository Pattern

## Learning Objectives

- Separate the DAO's responsibility (physical access to storage) from the repository's responsibility (domain-centric collection-like access).
- Decide which to use by who owns the query and who owns the domain concept, not by which name sounds more modern.
- Explain why conflating the two creates a persistence layer that leaks storage details into the domain.

## Introduction

The DAO and the repository are not the same thing, and they get treated as interchangeable by most interviews and plenty of code. The DAO (Data Access Object) is about physical storage access: move rows in and out of a specific database. The repository is about domain-centric access: collect entities by concept, almost as if they lived in memory. They solve different problems, they live at different layers, and forcing the two into one component is where a lot of clean designs start to fray.

## Problem Statement

Concrete failure: a codebase with `OrderDao` exposes methods like `int insertOrder(Order o)` and `OrderRow findOrderRowByStatus(String status)`. The service layer calls `findOrderRowByStatus` directly. The value objects trickle upward, an `OrderRow` carrying a column name, a persistence flag, and an internal ordering, and then a controller maps them into a view model. The domain no longer has an `Order`; it has a thin rename of database rows. Change the schema and you recompile half the service. The DAO did its physical job, so why is the whole app fused to the storage shape? Because the repository's domain-layer purpose never existed; the DAO was doing a job two layers above.

## Core Concept

The separation centers on one question: does this API speak in domain semantics or in storage mechanics?

| Concern | DAO | Repository |
|---------|-----|------------|
| Primary role | Physical storage access | Domain-focused access |
| Speaks in | Table/row/persistence terms | Entity/business terms |
| Who defines the query | The DAO (SQL, mapping) | The caller or the mapping |
| Layer | Close to storage | Close to the domain |
| Returns | Value objects or rows | Aggregates / entities / domain types |

The DAO worries about the database being Postgres, about columns, about the driver. The repository worries about `Order`, about a collection of orders, about domain rules like "active orders." The repository usually hides a DAO or an ORM underneath, because physical access eventually needs someone to know about the storage. The repository is what the rest of the app talks to, and the DAO is the adapter at the bottom.

That mental model directly answers the real tension people bring: which do I put in my code? The answer is both, but at different heights.

```java
public interface OrderRepository {
    Order findById(Long id);
    List<Order> findByCustomer(String customer);
    void add(Order order);
}
```

The repository says nothing about the table, the SQL, or the transaction. That is someone else's job once it selects. There's no `insertOrder` here; there's `add(Order)`, a domain word. Down at the bottom sits the physical adapter:

```java
@Repository
public class JpaOrderEntity implements OrderRepository {
    private final OrderDao dao;

    public Order findById(Long id) {
        OrderEntity re = dao.findByPk(id);   // physical
        return OrderMapper.toDomain(re);     // domain
    }
}
```

Where the mapping boundary lives is the judgment call. Some teams put the mapping inside the DAO's return and never expose a raw row at all. Other teams are fine with a repository that directly wraps a Spring Data `JpaRepository` and maps inside. Both are defensible. What is not defensible is a single class pretending to be both a DAO and a repository, then praying no one notices.

### What to name

The pair of errors most teams make: the "lazy repository" and the "physical DAO that everyone talks to." In a clean world the DAO is a private implementation detail, not the public API. If your services inject a DAO, then the whole application is coupled to storage vocabulary, and your domain types leak storage gunk.

A strongly stable rule: the repository is what domain and services see; the DAO is what the repository sees. That keeps the storage engine behind the pattern from the introduction, and it keeps the domain agnostic about which database asked for a flush.

### The two smells that expose a missing split

Two smells give the layering away faster than any document. The first is a service that builds its own persistence object, calling a DAO constructor inside business logic. The second is a repository returning a type that only exists because of storage, a `Row`, a `Record`, a `Tuple`. Both say the same thing: the boundary is leaking because the domain stopped being the surface.

```java
public class OrderService {
    public void cancel(Long orderId, String reason) {
        OrderDao dao = new JdbcOrderDao();       // smell: service owns storage
        OrderRow row = dao.findRowByPk(orderId); // smell: service sees a row
        if ("SENT".equals(row.getStatus())) {
            throw new CancellationDenied("already shipped");
        }
        dao.updateStatus(orderId, "CANCELLED", reason);
    }
}
```

Compare that with the same rule expressed at the domain, where the status check is a domain question and the persistence is a detail the service can leave alone:

```java
public class OrderService {
    public void cancel(Long orderId, String reason) {
        Order order = orders.findById(orderId);   // a domain type
        order.cancel();                            // the domain rule lives here
        orders.save(order);                       // one write, whatever the store
    }
}
```

The second version compiles to the same database work, but the service never learns the table name, the status column, or the driver. That is the entire difference the pattern buys. The first version leaks storage into the domain; the second keeps them apart, and containment is exactly the write-path ownership this chapter keeps returning to.

There is a legitimate exception, and honesty about it keeps the pattern useful. A one-off script, a small migration tool, or a batch job that pushes rows around is often better off as a pure DAO, because there is no domain worth protecting. The repository earns its cost the moment the codebase has real business rules that must not depend on the table layout. If you cannot name a second consumer who depends on the domain shape, you don't need the second layer yet, and adding it "for architecture" is the same form of over-engineering the pattern exists to discourage.

## Real Production Usage

Spring Data JPA's `JpaRepository` is genuinely a repository-as-domain gate because the services hit it, the entities are domain-shaped, and the mechanism is abstracted. When people need a DAO they usually drop raw `JdbcTemplate` or `MyBatis` under a `@Repository` for full control over mapping and SQL. That is exactly the layered model: a repository on top, a DAO as the driver below. Nothing here is invented.

The point is that you do not have to choose between Spring Data and a DAO any more than you choose between a front door and its hinge. The repository door is what the app knocks on; the DAO hinge is where the storage hangs. A `JpaRepository` is a door with the hinge already welded, which is fine for a plain entity. The signal to split them out is any moment the storage reality, a dialect trick, a custom mapping, a stored procedure, starts to demand a shape the domain would never name. That pressure is the boundary trying to tell you the layers have fused.

## Common Mistakes

1. **Naming a DAO method after the SQL and using it in services.** `findUserByStatusAndCountry` returns a screaming storage-linked method that ties both sides together.
2. **Treating the two patterns as synonyms in the interview or in design.** Then you skip the boundary question and nobody owns the mapping.
3. **Letting an entity ship storage fields into the service** by returning a DAO row direct, which is a domain leak.

## Interview Perspective

Interviewers ask "DAO vs repository" to see if you understand layering, not to hear the jargon. The weak answer either says they're the same or reproduces the CRUD that doesn't move things. The strong answer names the layer each sits on, states what each signals, and commits to keeping the domain on top. Adding "the DAO survives behind the repository and the repository is what the rest of the app sees" is the differentiator.

Follow-up: "How does the split help you test?" The strong answer: the repository interface is the seam. A service that depends on the repository can be given a fake that returns `Order` domain objects, and the whole ordering rule is asserted without a database running; the physical DAO is what a real-database test covers. That is the practical payoff of the boundary, not a stylistic preference.

## Knowledge Check

1. Your service calls a method named `findByPkInsertCount`. What layer does that suggest and what break lies there?
2. Shrinking DAO into a `JpaRepository` and leaving the repository out. Where does the mapping go and what does that cost?
3. If the repository returns either an entity or a DTO, which correlates with which pattern?

## Key Takeaways

- A DAO reads databases, a repository reads domain collections, and they do not overlap.
- The repository is what services see; the DAO stays hidden.
- The real payoff of the split shows up the day you change the storage mapping.

## What's Next

That repository and the DAO both hide a transaction, and that is where the next article takes you: the mechanics of what a transaction is and what ACID actually means. The app and the storage are always talking in units of work, and until you know whether `@Transactional` commits your two operations as a single unit, the repository you just designed can still lie to you under load.

---

This article explains the difference between a DAO's physical storage access and a repository's domain-facing access, and argues the repository sits above the DAO while the DAO stays hidden. It argues that using the two as interchangeable terms is exactly how storage leaks into the entire domain.