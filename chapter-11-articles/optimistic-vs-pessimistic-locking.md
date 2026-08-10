# Optimistic vs Pessimistic Locking

## Learning Objectives

- State the difference between optimistic and pessimistic locking in terms of when contention is paid for.
- Know how JPA expresses each, `@Version` versus `PESSIMISTIC_WRITE`, and what each actually locks.
- Choose between them for a concrete workload and defend the choice.

## Introduction

Two transactions can both read a row, both modify it, and the second write silently clobbers the first. Locking is the mechanism that decides who wins and who retries. Optimistic locking assumes collisions are rare, so it checks at write time and fails loudly. Pessimistic locking assumes collisions are the norm, so it takes the resource first and holds it. The word choice between them is really a bet about your traffic.

## Problem Statement

The classic race is the inventory counter. Two requests read `stock = 1`, both decrement, both write `0`, and the shop has oversold one unit. Add a second dimension: an edit screen where two admins load the same profile, change different fields, and save. Whichever saves second overwrites the first silently, and the lost update is someone else's day. The failure is not bad data; it is data that looked fine. Both scenarios need a mechanism, and the mechanism you pick determines whether you find out at write time or at contention time.

## Core Concept

The two strategies answer the same question at different moments: "was this row changed underneath me?"

Optimistic locking does not hold any lock during the read. It carries a version and checks it only when writing. JPA exposes this as a `@Version` field.

```java
@Entity
public class Product {
    @Id
    private Long id;

    @Version
    private Long version;

    private int stock;
}
```

Every `UPDATE` in Hibernate becomes `update Product set stock = ?, version = version + 1 where id = ? and version = ?`. If the where clause matches zero rows, the write affects nothing, and Hibernate throws `OptimisticLockException`. The loser retries, usually by reloading and replaying. Cheap when collisions are rare, because nobody waits on anything.

Pessimistic locking takes the row at read time with a database lock. JPA says `lockMode = LockModeType.PESSIMISTIC_WRITE` and issues something like `SELECT ... FOR UPDATE`. The second transaction blocks at the read until the first commits, then proceeds on the new data.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select p from Product p where p.id = :id")
Product findForUpdate(@Param("id") Long id);
```

The difference in experience is where you feel it. Optimistic fails late, at the commit, and the client must retry. Pessimistic waits early, at the lock, and the client blocks. That is the whole trade in one sentence.

### When each one wins

| Situation | Pick | Why |
|-----------|------|-----|
| Read-heavy, few writers, short write window | Optimistic | nobody waits on a read; collisions are rare |
| Long interactive edit over stale data | Optimistic | you cannot hold a DB lock while a human thinks |
| High-conflict hot row (balance, stock) | Pessimistic | retries loop forever under contention |
| Read-verify-write where the check is stale | Pessimistic | the lock covers the read and the write together |

The trap most engineers fall into is treating optimistic as strictly better because it is "lighter." It is lighter only when the retry rate stays low. Under genuine hot-row contention every retry pays for a failed transaction, and the aggregate cost beats the lock wait. Pessimistic is the correct answer exactly when the failed-attempt cost is higher than the wait cost.

### Where each one breaks

Optimistic breaks under two conditions. First, long-lived stale reads: a user holds a form open, the version advances in the background, and their save fails with a version mismatch they did not cause. The fix is not more versioning; it is deciding whether this interaction should be optimistic at all. Second, the retry loop: in a genuinely contended system, optimistic locking degrades into a retry storm, and each retry usually redoes work.

Pessimistic breaks on duration. A lock held for one statement is fine. A lock held while calling an external service, or while a human edits a form, serializes every other reader and burns your connection pool. If you need pessimistic for a long business process, the answer is almost always to restructure the process into short transactions, not to widen the lock.

### A note on the dirty read without any lock

There is a third option that is really the default, and you should name it in any discussion: no lock at all, last writer wins, accepting that a lost update is fine for the data in question. Counters that can tolerate drift, cached aggregates, logs, are all candidates. Choosing no locking on purpose beats pretending you have locking because you added an annotation somewhere.

### What the retry looks like in code

Whichever you pick, the loser has to do something, and the shape of that something is the practical half of the choice. With optimistic locking the loser catches the version exception and reloads the row before retrying. The reload is required, not optional: retrying with the old version simply fails again.

```java
public class ReservationService {
    private final ReservationRepository reservations;

    public void correct(Long seatId, String holder, int attempts) {
        for (int i = 0; i < attempts; i++) {
            Reservation r = reservations.findById(seatId);
            if (r.getHolder().equals(holder)) {
                throw new AlreadyReserved(seatId);
            }
            try {
                reservations.save(r.withHolder(holder)); // version bumped at flush
                return;
            } catch (OptimisticLockException e) {
                r = reservations.findById(seatId);       // reload the latest
            }
        }
        throw new RetryExhausted(seatId);
    }
}
```

The pessimistic version needs none of that scaffolding, because the second transaction blocks at the `FOR UPDATE` read and then proceeds on fresh data. That is the whole readability difference the choice buys: optimistic pushes retry logic into the application, pessimistic hides it in the database's wait. Your job is to decide which cost, a retry loop in your code or a blocked wait in the database, your traffic can afford.

## Real Production Usage

Spring Data JPA gives you both dials. `@Version` is the field-level optimistic default you get by adding one long to an entity. `@Lock(LockModeType.PESSIMISTIC_WRITE)` on a repository method produces a `SELECT ... FOR UPDATE` underneath. Real services use versioned writes for most domain rows and reach for pessimistic reads on the few rows where the retry cost is unacceptable, usually inventory or wallet-style rows. The pattern of pairing optimistic by default with pessimistic on the hot row is the production norm.

### The version is not a magic number

A `@Version` field still lets two coordinated reads both see the old value, because the version guards the write, not the read. That is why optimistic locking under `REPEATABLE READ` can miss a conflict the developer was sure it would catch: the second transaction re-reads its own snapshot, and a snapshot read is not blocked, so the write proceeds unless the version number itself is tested. Carry the rule with you: optimistic locking detects a lost update at the moment of write, and it does nothing to stop a read of stale value.

None of that changes the arithmetic. If ten users race for the last seat, optimistic locking loads ten versions and exactly one commits, and nine retry. Pessimistic queues all ten at the table row and serves them sequentially. Both end serial, but the ten losers wait in different places, retry work versus block time.

## Common Mistakes

1. **Using optimistic locking on a hot, heavily contested row.** Every write fails and retries, and the retry storm costs more than the lock ever would.
2. **Holding a pessimistic lock across a slow operation.** The lock covers a remote call or a user's think time, and the whole table effectively serializes.
3. **Believing optimistic locking stops reads from seeing stale data.** It does not. It only refuses to let a stale writer commit.

## Interview Perspective

Interviewers want you to phrase locking as a bet about contention. Weak: "optimistic doesn't lock and pessimistic does." Strong: "optimistic pays the check at write time and fails late with a retry; pessimistic pays at read time and blocks early, and I pick by whether a failed attempt is cheap enough to afford." The follow-up is almost always "what happens to the loser?" and the strong answer says retry with reload, versus blocked-and-proceed.

Follow-up: "Which do you put on a reservation system?" They want you to walk to the hot-row argument, not to a memorized slogan.

## Knowledge Check

1. Two requests run the same `decrementStock` with a `@Version` field. Describe exactly what the second one sees and what the caller must do.
2. You hold a pessimistic lock while calling a payment gateway inside the transaction. What breaks, and under what load?
3. A product has `@Version`. Why does that not stop a second transaction from reading the old `stock` value?

## Key Takeaways

- Optimistic checks at write and fails late; pessimistic claims at read and blocks early. That is the whole trade.
- Choose by contention: rare collisions means optimistic, hot rows and stale interactive edits mean pessimistic.
- Locking duration, not the annotation, is what ruins a pessimistic design.

## What's Next

The flip side of writing safely is reading efficiently, and the very common cost of reading through an ORM is next. The next article is about lazy versus eager loading and the N+1 problem, the query explosion that happens when your persistence context quietly loads every child on demand.

---

This article explains optimistic locking, which checks a version at write time and fails late with a retry, and pessimistic locking, which takes the row at read time and blocks early. It argues that choosing between them is a bet on contention rate, and that a retry storm over a hot row is worse than the wait it avoided.