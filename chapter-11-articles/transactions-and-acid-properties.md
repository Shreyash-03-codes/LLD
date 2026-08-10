# Transactions and ACID Properties

## Learning Objectives

- Define a transaction as a bounded unit of work and say what each ACID letter does and does not guarantee.
- Read `@Transactional` in Spring and predict whether a transaction actually opens, merges, or silently no-ops.
- Distinguish isolation levels and name what each one still allows to confuse you.

## Introduction

A transaction is a bounded unit of work against a database: a set of changes that ends in a single commit (keep everything) or a single rollback (discard everything). ACID is the promise the store makes about that unit. It gets recited on auto-repeat and rarely unpacked. Most of what matters is in the corners nobody speaks the isolation levels and the strange ways Spring decides whether a transaction even exists.

## Problem Statement

Here's the concrete failure. Two `INSERT`s happen in one request but two separate transactions. The second fails, the service catches nothing, and the first row is committed to the database on its own. The business record is now split down the middle, and the customer sees the broken half. The reflex is to slap `@Transactional` on the method and call it solved. It sometimes works. Other times it attaches to a private method that never turned on, or a checked exception is swallowed and the rollback never fires, so the annotation has secretly done nothing and the bug moves to later in the day. Without a real definition of what a transaction is and where its boundary holds, you're adding annotations and hoping.

## Core Concept

Break the acronym into its actual jobs.

- **Atomicity** is all-or-nothing for the whole unit. No half state is ever visible.
- **Consistency** is the store moving from a valid state to a valid state, honoring the constraints declared.
- **Isolation** is concurrent transactions not seeing each other's in-flight partial writes.
- **Durability** is a committed write surviving a crash and restart.

The letter people trip on is C. Consistency in ACID is not "your business is logically sound." A database cannot enforce that your order total matches the sum of its line items unless you declare it. The database can only protect the invariants it knows about, foreign keys, unique constraints, checks, and not the ones that live only in your head. Application consistency is built by your code executing under the right boundary, and the durability at the bottom is what your transaction boundary claims.

### The Spring boundary

The real skill is reading `@Transactional` for what it is: a promise that says "if this method runs through the Spring proxy, scope a transaction around it." It does not begin a transaction. It invites one. That means a few quiet landmines.

- A private method annotation is ignored, because the proxy never sees a call into a private method.
- Calling `this.method()` from another method in the same bean bypasses the proxy and the annotation completely.
- Propagation decides whether the call joins an existing transaction or starts a new one.

```java
@Service
public class BookingService {
    private final BookingRepository bookings;
    private final PaymentGateway payments;

    @Transactional
    public void book(Booking booking) {
        bookings.save(booking);
        payments.charge(booking);
    }
}
```

With default propagation the two writes share one unit. If `payments.charge` throws after `bookings.save`, both are rolled back, and that is atomicity wearing a boundary. But the rollback only triggers when the exception propagates. Spring rolls back on unchecked exceptions by default; a checked exception or a swallowed catch turns your transaction into a no-op. The clean catch is to roll back explicitly.

### Isolation levels, strict to lazy

Isolation is about what one transaction sees of another's in-flight work.

| Level | Protects against | Still sees |
|-------|------------------|-----------|
| READ UNCOMMITTED | almost nothing | dirty reads, unrepeatable reads, phantoms |
| READ COMMITTED | dirty reads | non-repeatable reads, phantoms |
| REPEATABLE READ | non-repeatable reads | phantom rows |
| SERIALIZABLE | all of it | none, at full lock cost |

`READ COMMITTED` is the Postgres default and quietly good for most workloads. `SERIALIZABLE` is the safety you reach for when the invariant is worth losing concurrency, but most applications never need it. The trade-off you are really agreeing to is how much concurrency you give up to hold the stricter guarantee.

### Reaching below ACID

Naming the gap is the part interviews respect. A multi-step money move that must never half-commit belongs in a transaction. But an analytics row you can afford to lose once, or a huge event stream where any ACID store breaks the write lane, is a candidate for eventual consistency. Plenty of real systems accept "at least once" and de-dupe later. Knowing when ACID is the wrong answer, and admitting the store makes no such promise, is a senior call, not a bug.

### Setting the dial explicitly

Because frameworks hide the boundary, the deliberate act is writing the isolation and propagation where you actually mean them. An optimistic reconsideration of a `readOnly` flag is the cheapest win: on a method that only reads, you mark it `@Transactional(readOnly = true)` to tell the provider it can skip dirty-tracking and reduce flush work, and it signals to the next reader that this method must not write.

```java
@Service
public class AnalyticsQueryService {
    @Transactional(readOnly = true, isolation = Isolation.READ_COMMITTED)
    public RevenueReport daily(int days) {
        return repository.aggregate(days);
    }
}
```

The same screen is where the propagation lives. `Propagation.REQUIRED` (default) joins an existing transaction; `Propagation.REQUIRES_NEW` suspends the calling one and starts its own, so an inner write commits even if the outer rolls back. The honest rule is that `REQUIRES_NEW` is a deliberate "I want this to survive separately from my caller," and reaching for it casually is how nested actions stop respecting the outer boundary.

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void audit(String event) { ... }
```

An audit record is the classic example, because you want it written even when the main transaction fails, and that is the silence of the trade. Naming the dial, not letting the framework pick it, is the behavior that turns the annotation from a checkbox into a decision.

### Where people actually stick the annotation

The most common deployment mistake is `@Transactional` on a controller or a web-layer boundary. It looks harmless, but now a single HTTP request owns one giant transaction that wraps serialization of the response and any network work the controller did. Under load that is a held connection and locks held far longer than the business needed. The boundary rule: start the transaction at the service method that owns the business operation, and keep the web layer and presentation out of it.

A second favorite spot is the repository `save()`. If the repository method carries `@Transactional`, every `save` becomes its own transaction, and a service method that wanted one unit across two saves gets two. That is the opposite of what the earlier `transfer` wanted, and it fails silently because nothing throws. The boundary belongs at the service, not at each repository write. A team that cannot point at one place and say "the transaction lives here" has no transaction ownership at all.

## Real Production Usage

Spring and Hibernate do this for you: the proxy intercepts the annotation, `JpaRepository.save` participates in the open transaction, and Hibernate flushes at commit, which is durability done on the store. The gap you have to manage is between the framework default and the real database isolation, because JPA can flush ahead of the commit and turn a read inside the same transaction into something different than you expect. The fix is knowing which is which on a given database.

## Common Mistakes

1. **`@Transactional` on a private method.** It silently does nothing, and the eventual sign is a half-commit nobody can defend.
2. **Treating database consistency as your business truth.** It is not. The DB will only protect the constraints it can see; your invariants are on you.
3. **Catching and swallowing inside the boundary.** The rollback never fires, which is worse than no annotation because you now believe you have a guarantee.

## Interview Perspective

The interviewer wants you to restate the letters and then say which one actually gutter under a specific scenario. The weak answer recites the four letters. The strong one says "under READ COMMITTED a dirty read is impossible, but a repeated range query can still surface phantom rows." Expect the probe "what does consistency really mean here" that separates the person who read a blog from the one who watched the DB only enforce what it could declare.

Follow-up: "What happens if you call `book()` twice from the same thread?" That opens propagation and proxy behavior.

## Knowledge Check

1. Is "inventory can't go negative" a database consistency guarantee or an application invariant? Defend the answer.
2. Postgres defaults to READ COMMITTED. Two transactions both read, then both write, the same row. What happens to the second write? Be exact.
3. Your `@Transactional` method calls another of its own methods directly, no proxy involved. How many transactions actually open?

## Key Takeaways

- Atomicity is decided at commit, isolation is the concurrency window, and consistency only covers what the database declared.
- `@Transactional` is a boundary, not a spell: proxy calls, propagation, and exception flow decide whether it does anything.
- Every isolation level trades throughput for a guarantee, so pick by the invariant you refuse to break.

## What's Next

Atomicity, flush timing, and the persistence context point at one question: who decides when a batch of writes gets pushed to the database at all? The clean answer is the Unit of Work pattern, the idea that sits between the commit boundary and the count of SQL statements. ACID tells you the promise; Unit of Work tells you who assembles the list and hands it over.

---

This article explains what each letter of ACID actually guarantees and when a Spring @Transactional boundary silently does nothing. It argues that database consistency only ever protects declared constraints and never your business rules, and that a swallowed exception quietly voids the transaction's whole warranty.