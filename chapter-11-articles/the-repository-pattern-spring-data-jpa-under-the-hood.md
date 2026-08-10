# The Repository Pattern, Spring Data JPA Under the Hood

## Learning Objectives

- Explain what a repository hides and why Spring Data JPA makes you define only an interface.
- Name the concrete runtime artifacts the framework creates from a repository interface.
- Recognize when the repository abstraction leaks and why that breaks the pattern.

## Introduction

A repository is a collection-oriented facade over your storage. It makes a database look like an in-memory collection of domain objects: you add one, you get one back, you search by some property, and you never talk about the database yourself. Spring Data JPA is the framework that, given only a Java interface, writes the implementation at runtime. Because it does most of the work for you, everyone can use a repository and almost nobody can explain what actually got generated behind the interface.

That gap matters. When things go wrong, it's rarely the pattern. It's the framework's quiet behavior the author didn't account for: the eager-ness of a fetch, the caching of an entity, the shape of a generated query.

## Problem Statement

Take the most common real failure: you declare `interface OrderRepository extends JpaRepository<Order, Long>` with a method `List<Order> findByCustomerId(Long id)`. You have no class implementing it, none of your code calls SQL. Then production shows you logs of a query that's semantically backwards, or a lazy-loading exception pops up on a page you ship, and you have nothing to inspect except the interface and a stack trace that points into `SimpleJpaRepository`. Without knowing what the framework actually generated, you're debugging a black box by trial and error. That is the concrete failure this article fixes: treating a one-line interface as a completed piece of work instead of a query contract you should read like code.

## Core Concept

Forget that it says "Spring Data JPA" for a moment. The pattern is: you declare intent in an interface; a mechanism generates the mechanics. Every call to `findAll()`, `save()`, or a custom `findByX` is a contract resolved by a runtime implementation, not by anything you wrote.

### What generates the class

Spring Data works through a proxy. At startup it scans your repository interfaces, inspects their generic type arguments, and inspects each method's name and return type, then builds a JDK proxy backed by `SimpleJpaRepository`. That base class already contains the CRUD implementation you inherit: `save`, `findById`, `findAll`, `count`, `delete`. Derived methods like `findByCustomer` are resolved at runtime from the method name using a query parser: the framework strips `find`, `By`, and then reads the remaining tokens as property paths against the entity.

```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByCustomer(String customer);
    Optional<Order> findByCustomerAndStatus(String customer, OrderStatus status);
    @Query("select o from Order o where o.customer = :customer and o.status = :status")
    List<Order> findExplicitly(@Param("customer") String customer, @Param("status") OrderStatus status);
}
```

Read that interface like the code it produces. `findByCustomer` becomes a JPQL query `select o from Order o where o.customer = :customer`. The `And` keyword joins predicates with AND. The explicit `@Query` version bypasses derivation entirely and hands you a named query; here the framework performs the exact statement you wrote, so the derivation logic is irrelevant to that method.

Relying purely on method-name derivation is comfortable and trappy. Rename a field and the "ambiguity" error at startup is your contract breaking loudly, which is good, but the naive spellings hide in a stringified name the moment you rename the property and forget the method, which is a findable-at-startup miss. Read every repo method by asking: what is the generated statement and where does it hold the state I didn't spell out.

### The caching you did not ask for

The biggest buried fact is the persistence context, the in-memory cache inside the transaction. JPA keeps snapshots of every entity you loaded. When you call `save()`, it does not force a SQL insert the way a direct DAO write would. It marks the entity managed. The actual flush happens later, at transaction commit or when a query forces it, which the framework decides. So `save()` is not "write to database." `save()` is "tell the persistence context that this entity is now managed," and the real write sits at the flush boundary.

That single distinction is the repository's whole gotcha. Two `save()` calls in the same transaction do not reliably mean two inserts; if the entity has the same identity, JPA merges them. A flush-triggering query in the middle can push pending writes earlier than you expected. When Spring Data's purpose-built read methods and flush behavior seem out of sync, the persistence context is usually the culprit.

### Read through the persistence context

This is worth spelling out because it changes how repository reads behave.

Diagram: reads pass through the JPA persistence context before hitting the database.

<svg width="740" height="340" viewBox="0 0 740 340" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="arr" markerWidth="10" markerHeight="10" refX="8" refY="4" orient="auto">
      <path d="M0,0 L8,4 L0,8 Z" fill="#222"/>
    </marker>
  </defs>

  <rect x="40" y="60" width="150" height="70" rx="8" fill="#eef2f7" stroke="#333" stroke-width="2"/>
  <text x="115" y="90" text-anchor="middle" font-family="Arial" font-size="14" fill="#222">Call</text>
  <text x="115" y="112" text-anchor="middle" font-family="Arial" font-size="13" fill="#333">findByCustomer</text>

  <rect x="290" y="60" width="180" height="70" rx="8" fill="#fdf6e3" stroke="#8a6d00" stroke-width="2"/>
  <text x="380" y="88" text-anchor="middle" font-family="Arial" font-size="14" fill="#222">Persistence</text>
  <text x="380" y="108" text-anchor="middle" font-family="Arial" font-size="14" fill="#222">Context</text>

  <rect x="550" y="60" width="150" height="70" rx="8" fill="#eef7ee" stroke="#2f6f3e" stroke-width="2"/>
  <text x="625" y="96" text-anchor="middle" font-family="Arial" font-size="14" fill="#222">Database</text>

  <path d="M 192 95 L 286 95" stroke="#222" stroke-width="2" fill="none" marker-end="url(#arr)"/>
  <text x="239" y="84" text-anchor="middle" font-family="Arial" font-size="12" fill="#555">check identity map</text>

  <path d="M 472 95 L 545 95" stroke="#222" stroke-width="2" fill="none" marker-end="url(#arr)"/>
  <text x="508" y="84" text-anchor="middle" font-family="Arial" font-size="12" fill="#555">miss</text>

  <path d="M 472 95 C 470 140 115 140 113 95" fill="none" stroke="#7a5e00" stroke-width="2" marker-end="url(#arr)"/>
  <text x="290" y="152" text-anchor="middle" font-family="Arial" font-size="12" fill="#555">hit, return cached instance</text>
</svg>

The point of that diagram isn't cleverness. It's the reason repeated `findByCustomer` calls in one transaction can return the same object before flush without SQL. Production logic that expects "one call, one SELECT" breaks here, and knowing the identity map kills the surprise.

### Two boundaries worth naming

Inside the generated implementation two configuration facts change behavior. First, Spring Data exposes a flush mode. In the default `AUTO`, the provider flushes before a query that touches the flushed type, so a read finds what a `save()` just wrote but may also emit pending writes early. In `COMMIT`, no query flush happens, so reads inside the transaction see only flushed data, which is faster but can surprise someone expecting their own pending write to show up.

Second, `@Modifying` queries with `clearAutomatically = true` trade the unit of work's cache for predictable state. A bulk UPDATE bypassing the persistence context leaves the entities stale, so clearing the context afterwards is the deliberate choice between fewer statements and what the business code sees. These two dials are the seams where a repository method stops behaving like the collection it promises to be.

## Real Production Usage

Real Spring Data services win by leaning on derived queries for reads that match the entity's natural shape, and they sharply control the persistence context with explicit `@Transactional` boundaries and occasional `flush()` and `clear()` to cap the cache's lifetime and grab predictable SQL. The dominant mistake is letting a transaction run too long so the context quietly holds hundreds of managed entities, then a flush at the end issues a spike of writes that appeared to never have touched the DB.

## Common Mistakes

1. **Reading the interface as the contract, never the generated query.** A `findByCustomerAndStatus` with an argument ordering that's wrong still compiles and fails days later.
2. **Assuming `save()` means immediate write.** It registers into the persistence context; the flush is elsewhere. Expecting the row to exist before commit is how delayed flush bugs arrive.
3. **Letting method-name derivation drive naming.** Property renames silently flip the runtime generating a different statement.

## Interview Perspective

Good interviewees know a repository interface implies a generated implementation and can say what methods that produces and where flush happens. The weak answer goes one of two ways: they either call the repository all magic, or they recite the CRUD methods without touching the persistence context. Interviewers probe the context because it's the piece people skip. Try "why would two calls to `save()` produce one SQL insert?" and notice whether they can say "managed entity, same identity, one flush at commit" or if they stall.

Follow-up "how many queries does the framework run when you materialize a lazily loaded collection?" exists to see if persistence-context awareness is real or only a term they memorized.

## Knowledge Check

1. Your repository method's JPQL is derived from the method name. A field rename silently changes a query. If you want the compiler to catch it, which alternative do you reach for?
2. Session A calls `repository.save(order)` and, in the same transaction, `repository.findById(order.getId())`. How many SQL inserts and selects does reality likely run, and why?
3. Contrast the explicit `@Query` in this file with the derived one. Which breaks first and what's the failure mode difference?

## Key Takeaways

- A repository interface is a contract for a runtime-generated implementation; read every method like the query it writes.
- `save()` registers an entity into the persistence context, not into the database. Flush and commit own the write.
- The persistence context caches identity, so reads in one transaction can skip SQL entirely, and you must account for it.

## What's Next

The obvious follow-up question after meeting the repository is whether it's the only data-access shape you have. The next article draws the line between the repository and the DAO, because the two get conflated constantly and the difference is about who owns the transaction and the persistence context. That article pins down which abstraction you are really standing on and when reaching for a DAO is the right call or a confession.

---

This article explains that repository is a contract for a framework-generated proxy and that Save clones the persistence context rather than SQL writes. It argues that reading a repository method like the query it emits is what keeps Spring Data from becoming a black box.