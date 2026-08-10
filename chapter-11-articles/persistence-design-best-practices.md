# Persistence Design Best Practices

## Learning Objectives

- Apply a working checklist to a real backend schema and catch the recurring failure categories before they hit production.
- Resolve the three tensions that dominate persistence work: consistency versus throughput, one source of truth versus several, and a deliberate decision versus a framework default.
- Enforce a business invariant in the layer that can actually guarantee it.

## Introduction

This chapter has handed you a set of focused tools: storage engines, the repository, transactions, the unit of work, locking, fetch strategies, and caches. Best practices is what those tools look like when they are actually holding a load. This isn't a tutorial on any one of them. It's the set of judgment calls that decide whether the persistence layer survives traffic or quietly betrays the domain.

## Problem Statement

The concrete failure is the chapter's tools used at their installed defaults. A service ships with lazy loading left on everywhere, a cache whose eviction was never wired, writes with no version guard, and a `@Transactional` hung across a controller. Alone, none of it throws. They all look correct in isolation. Then one request hits the cache miss during a spike, one transaction waits on a remote call, and one unguarded write races a refund, and the layer that "worked all along" turns out to have been drifting the whole time. The habits below exist to replace that drift with a decision.

## Core Concept

The first rule worth stating plainly: the framework default is a decision, it just was made by somebody who never met your data. If you took every suggestion in this chapter at its default, you would get a lazy everything, eager where the cost is invisible, a cache whose eviction nobody wired, and rows updated without a version. It would mostly work, until the one request that pays for all those defaults at once.

So the practice is to turn each tension into an explicit criterion, then pick. Three tensions keep coming back.

| Decision | The default you would drift into | The deliberate version |
|----------|---------------------------------|------------------------|
| Relationship fetch | lazy everywhere because it was default | declared per query shape |
| Write concurrency | no version, last-writer-wins | versioned, or locked on the hot row |
| Cache writes | update the cache gently | delete the key, invalidate |
| Transaction scope | web request boundary | smallest unit that owns the operation |
| Schema shape | as-written first draft | a reason behind each non-obvious column |

The habits later in this article are the "deliberate" column made actionable. A review is fastest when framed as: for each row here, have you slipped to the default?

### 1. Consistency is owned, not assumed

The number one rule for any write-heavy table: some single place must answer the question "who guarantees this invariant, and at what moment?" An inventory row cannot go negative. That is not a database fact, as established in the transactions article; it is a fact of your code. If the answer is "the service method," then the transaction boundary and the lock both live in that method, and any second writer through a separate path is a bug you should be able to point at. A consistency rule with no owner is a consistency rule that will be broken by the first clever idea someone else has later.

### 2. Transactions are short and named

A transaction that outlives its work is a connection held and a lock held on a unit that should already be done. The rule of thumb: make the transaction the smallest unit that contains the whole business operation, and keep remote calls and human work out of it. A database transaction is a tool of correctness, not a wrapper for a network call.

```java
@Transactional
public void transfer(Long from, Long to, BigDecimal amount) {
    Account a = accounts.lockForUpdate(from);  // hot row, guarded
    Account b = accounts.lockForUpdate(to);
    a.debit(amount);
    b.credit(amount);
}
```

Short, single-purpose, and every remote call is outside this boundary. Nothing about the transfer waits on anything except the two rows it must move.

### 3. One source of truth on the write path

Every write path needs a single owner for consistency, and every read is a projection of it. The moment two writers update two copies and no one reconciles, you have a system that serves whichever copy the reader happened to hit. The cache-aside default from the caching chapter already commits to this: the database owns the truth, the cache is a projection, writes invalidate. Keep that single-owner rule even as you add layers.

### 4. The schema earns its shape

A denormalized column, a cascade delete, a renamed table all hard-code a business decision into the database, and the application must mirror it forever. That is fine when the decision was deliberate and cheap to renew. The failure is a schema whose shape was chosen as the default when you wrote the first draft, then silently enforced for years. Every non-obvious schema decision deserves a reason and a place where it breaks loudly if it drifts, not a place where it fails silently.

### 5. Test the boundary, not the syntax

The persistence test that matters uses a real database and asserts a real property: the transaction rolled back the whole order, the fetch did not become N+1, the `@CacheEvict` actually ran when the write happened. A test that only proves a repository method exists proves nothing about persistence. The boundary between the code and the store, commit, rollback, flush, lock, is exactly where these problems live, and it is where a lone database-free test goes quiet.

### 6. The constraint belongs at the data

The endgame of ownership is not only code, it is the database constraint. A unique index, a check, a foreign key, is the one place a rule lives that no application error at runtime can bypass. The order of attacks for an invariant is: enforce it in the database, so the store refuses bad rows; then own it in the service so the error message is human; then let the ORM reflect it.

```java
public class AccountBookEntry {
    @OneToMany(mappedBy = "account")
    private List<Entry> entries;

    public void post(Entry e) {
        if (BigDecimal.ZERO.compareTo(balance().add(e.amount())) > 0)
            throw new InsufficientBalance();     // friendly branch
        entries.add(e);                          // the DB also blocks below zero
    }
}
```

The database constraint is the last line, and the domain check is the one that produces a sensible error. As a junction, the invariant is enforced twice, but it is owned once. This is how a real schema protects itself and still speaks to a user.

## A concrete audit

If you have five minutes on a real schema, run it past these in order. Who owns each write and could two writers both claim it? How long is the longest transaction and what is it waiting on? Which relationships are lazy and which were meant to be, and what would a query log say? Does the cache evict the day a write, and does a test ever watch it do so? And where does the schema reflect a business rule that no constraint protects? Running those five questions over a schema in code review catches most of the chapter's proposed drift long before it reaches production.

## Real Production Usage

The production shape is smaller than the theory implies. A read-mostly service keeps a cache of a stable render, an inventory row is guarded by its lock and by the constraint behind it, and the repository exposes the one fetch query the path actually needs. What fails in practice, almost every time, is the service that spreads consistency responsibility across several methods and a cache update that races a write. The discipline is to compress the ownership to one place and to keep the transaction short.

## Common Mistakes

1. **Spreading the invariant across three writers.** Each route checks a "stop condition" its own way, and none is the owner, so the invariant fails exactly when two routes run at once.
2. **Letting the transaction wait on a remote call.** A lock is held across a network round trip, and the service's throughput collapses to the latency of that call.
3. **Keeping every framework default.** An eager fetch or an updated cache or a version-less write each looks fine in isolation and only fails together under the load.

## Interview Perspective

An interviewer finishes a persistence design and asks "what would you change if the load doubled." The weak answer is a list of literals. The strong answer is criteria: "I would audit the write path for a single atomic owner, the transaction for a remote call, the cache for whether the eviction runs on the write, and the fetch for whether it is declared." They want to see you treat persistence as a system under tension, not as a pile of repository classes.

Follow-up: "Which default would you disable first?" They want you to single out one, an eager fetch or a version-less write, whose blast radius is biggest for the workload at hand.

## Knowledge Check

1. Two methods, both validators, both write the same inventory row. Which one owns the invariant, and what races does the split create?
2. A transaction holds two locks and calls a payment gateway inside it. What burns as load scales and what does that say about a boundary?
3. Why is caching an update a mental mismatch, while an invalidate is a coherent rule, given the one-truth requirement?

## Key Takeaways

- Persistence decisions are often made by defaulting; replace each default with a deliberate criterion against the access pattern.
- Enforce each invariant with a single owner and a short, named transaction.
- Remove the remote call that a transaction is trying to wrap.

## What's Next

Every practice so far assumed one database and one node at a time. The last article widens the lens to the reality that is bigger: designing for data consistency in concurrent systems covers the distributed race, the cached copy, and where the rules above hold and break in production.

---

This article explains the chapter's tools as five persistence habits, every invariant owned, short transactions, one truth on the write path, and a schema that earns its shape. It argues that the framework default is a decision made for you, and that a layer survives production only when its choices were deliberate.