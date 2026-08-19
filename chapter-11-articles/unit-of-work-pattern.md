# Unit of Work Pattern

## Learning Objectives

- Define a Unit of Work and state what it tracks while a transaction is alive.
- Explain why the JPA persistence context is a Unit of Work and where flush and commit differ.
- Predict when a mid-transaction flush happens and what it costs.

## Introduction

The Unit of Work sits between the object model and the database. It watches every entity that a transaction loads or mutates, remembers which are new, which are dirty, and which are marked gone, and at the flush points issues the statement set that brings the database back in line with the in-memory state. In Spring Data this illusion has a name you already met: the persistence context. The Unit of Work is the reason "save" feels instant and then one committed statement appears from nowhere.

## Problem Statement

Picture an admin form that edits three rows of a config. The service calls `save()` three times, and the author assumes that means three SQL writes, one per row, to the database in that order. It does not. Depending on what changed and what is dirty, the persistence context produces far fewer statements, sometimes a single `UPDATE` for the whole batch at the moment of commit. Worse, if the context had earlier loaded more entities than you touched, the flush at the end writes changes you never meant to touch because some state got dirty by association. The failure is that writes did not follow the shape of the `save()` calls. The Unit of Work decided otherwise, and nobody had accounted for it.

## Core Concept

A Unit of Work is an identity-first ledger plus a write manager. Every entity loaded inside a transaction is bound to its row. When you mutate a field, the unit records the change. At the decision point, the flush, it compares current state against the snapshot taken at load time and writes only the diff. In JPA this ledger is the persistence context, scoped by default to one transaction and backed by one `EntityManager`.

Consequences to internalize:

- Two reads of the same row within a transaction return the same Java object, courtesy of the identity map.
- `save()` registers an entity as managed; it does not immediately emit SQL.
- The actual write is deferred to the flush, which the framework decides, usually at commit, unless a query forces it earlier.
- A flush triggers an atomic batch: all net pending changes go out together.

```java
@Transactional
public void adjustInvoice(Long invoiceId, Long customerId) {
    Invoice invoice = invoices.find(invoiceId);
    Customer customer = customers.find(customerId);

    invoice.applyCredit(customer.getPoints());
    customer.markRewarded();

    // no SQL yet; save() was called inside the repository but no flush
    entityManager.flush();   // optional: force the write before continuing
}
```

The `flush()` here is the single point where tracked changes become SQL before commit, and it is there if you genuinely need a row's generated value before a later statement. If you do not need that, let the flush happen at commit and reap the batching. Forcing flushes is the classic way to kill a transaction's batching and scatter writes that could have been one batch.

### Flush, commit, and the real boundary

| What happens | When it triggers |
|--------------|------------------|
| Entity registered | when `save()` is called |
| Dirty state recorded | as you mutate managed entities |
| Flush (write) | automatic before commit, or when a query requires it |
| Commit (durable) | at the end of the transaction boundary |

The common failure is the question "did my row get written?" The honest answer is "at the flush, which is usually at commit, unless a query nudged it." Flush writes without committing, so a mid-transaction rollback still undoes the flushed rows at the session, and nothing is durable until the commit line.

### The identity map as the spine

What keeps the pile coherent is the identity map: one entity instance per row inside a session. Without it, loading the same row twice would hand back two objects, your updates would silently fight, and dirty checking would be meaningless. When you hit `clear()`, `detach()`, or `merge()` you are reaching into this same ledger. A cleared or detached entity loses its tracked status, and the next `save()` of a detached copy does a merge instead of the managed update you expected. Most "JPA didn't save my change" incidents trace back to an entity that left the context, not to SQL.

With Hibernate, the dial you manage under this pattern:

- `entityManager.clear()` empties the context so a long operation forgets its tracked set.
- `entityManager.detach(entity)` drops one entity from tracking.
- `entityManager.merge(entity)` reattaches a detached copy.

All of it exists so you can control how big and how honest the ledger is at any moment.

### Batching is the payoff

The reason the Unit of Work exists is that it can collect a pile of small writes and fire them in groups the database can accept at once. That is the difference between a loop of 10,000 inserts and 10,000 separate round trips. The unit collects the inserts, and the flush boundary is the batch boundary.

Spring Data expresses this as `saveAll()` rather than `save()` in a loop: the latter keeps one statement per iteration, the former stages the set. The rule for the boundary is simple. If a page saves forty children besides the parent, use a method that signals intent to write a set and let the unit turn it into as few statements as the store allows.

### The size and the end of the unit

The shortcut that stops the ledger from quietly becoming a liability is flushing at the right moment. A unit held open and growing holds a connection and a lock, so a well-behaved write path is short: load only what the operation needs, mutate, flush at commit. The difference between a smooth write and a slow one is rarely the SQL speed. It is how long the unit sat open and how much it tracked.

There is one case where forcing the flush early is genuinely correct, and it is worth separating from the panic flush: when a later statement in the same transaction needs a value the database will produce, usually a generated primary key or a unique constraint conflict. You flush and catch, or you flush to materialize id, rather than leaving the pending write hostage to the commit.

```java
@Transactional
public void importShelf(List<Book> books) {
    for (Book b : books) {
        books.persist(b);

        if (b.requiresIsbn()) {          // needs the id now
            entityManager.flush();        // write it, see the failure here
            b.setIsbn("isbn-" + b.getId());
        }
    }
}
```

That measured flush, on the rare path that actually needs the id, is the legitimate use. Flushing at the top "just in case," on every iteration, is the anti-pattern, because it tears the batch apart for nothing.

## Real Production Usage

Every Hibernate surface you touch, `.clear()`, `.detach()`, `.merge()`, and the `hibernate.jdbc.batch_size` knob is an intervention in the Unit of Work. Batching is the framework deliberately collecting writes and submitting them in groups. The N+1 spike, the "why is my page taking long to commit", and the ghost changes you never asked for are all right there in the ledger behavior. Real services tune the batch size and bound the context length rather than fight the pattern.

## Common Mistakes

1. **Calling `save()` in a loop and assuming it writes per iteration.** It registers, then flushes; the eventual SQL count is almost never the count of `save()` calls.
2. **Scoping the context to a whole request.** It holds the connection, the lock, and the tracked set, then flushes a huge batch at the end.
3. **Forcing a flush everywhere to satisfy one early read.** You lose batching and scatter writes that a single committed batch would have handled.

## Interview Perspective

Weak: "the repository writes the entity into the database." Strong: "the persistence context tracks managed entities and writes the diff at flush, which usually sits at the commit; save is registration." Interviewers want you to declare the gap between save and commit, and they often press on "call save in a loop" expecting the diff and identity-map answer rather than an ORM shrug.

Follow-up: "If the method mutates a man charged entity then calls a query, what happens?" They expect you to say the flush can be triggered by the query rather than held to commit.

## Knowledge Check

1. A transaction loads the same row through two code paths. Are they the same object of two, and which mechanism decides?
2. Does `save()` cause SQL immediately or at commit? Name the thing that can force a mid-transaction write.
3. Your unit holds an entity, your method mutates it, and then a native query runs. What does the unit do when the query runs, and why does the timing matter to you?

## Key Takeaways

- A Unit of Work tracks loaded and mutated entities and writes only the difference at commit.
- `save()` is a registration, flush is the write, and commit is the durable boundary.
- The identity map holds one entity per row, and anything calling it off the context is what breaks the trick.

## What's Next

The Unit of Work shows both sides waiting to write the same row, and the next article settles who wins. Optimistic versus pessimistic locking is where a transaction picks its traffic rules, and it is the gateway to the concurrency thinking that closes out this chapter.

---

This article explains the Unit of Work behind JPA's persistence context, tracking changed entities and writing the diff at commit. It argues that a loop of save() calls is never a loop of SQL, and that the flush boundary, not the save, is the boundary nobody remembers.