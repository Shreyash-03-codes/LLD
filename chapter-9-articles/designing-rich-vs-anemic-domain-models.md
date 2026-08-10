# Designing Rich vs Anemic Domain Models

## Learning Objectives

- Recognize an anemic domain model by its shape: entities that are only data and getters, and services that hold every rule.
- Build a rich model by moving rules onto the objects that own the mutated data, with the change methods as the only way the data changes.
- Argue the honest boundary: an anemic shape is a smell for a domain object and harmless for a DTO or read model.

## Introduction

An anemic domain model is a domain model with no behavior. The objects exist, an `Order`, an `Account`, a `Customer`, and each is a bag of fields with getters and setters. The real rules live in a separate service layer, and the objects are empty vessels the services push through.

A rich domain model is the opposite: the object that owns the data also enforces the rules that govern it. This article is the chapter's verdict on where behavior lives. Every earlier article built toward it, and together they narrow to one question every backend developer has answered one way or the other: does the logic live with the data, or beside it?

## Problem Statement

The shape a great many codebases default to is the bag. The entity is fields and accessors:

```java
public class Account {
    private Money balance;
    private boolean suspended;

    public Money getBalance() { return balance; }
    public void setBalance(Money value) { this.balance = value; }
    public boolean isSuspended() { return suspended; }
    public void setSuspended(boolean value) { this.suspended = value; }
}
```

The transfer logic, the overdraft check, the suspension check, all of it, lives in a service:

```java
public void transfer(Account from, Account to, Money amount) {
    if (from.isSuspended() || to.isSuspended()) {
        throw new AccountSuspendedException();
    }
    if (from.getBalance().compareTo(amount) < 0) {
        throw new InsufficientFundsException();
    }
    from.setBalance(from.getBalance().minus(amount));
    to.setBalance(to.getBalance().add(amount));
}
```

It works, the code runs, and the tests pass. The cost is what you cannot see yet.

The first cost is that the rules have no home. The suspended check lives in the transfer service, and next week a withdrawal service is written that has to remember the same check. The rule is duplicated, and every caller must remember it, which is exactly the scattered-boolean failure from the business rules article, rebuilt with getters and setters.

The second cost is that any caller can mutate the object with no guard. `setBalance` is public, so a service can write the balance in any order, and the balance never passes through a rule. The invariants this chapter placed on the object live nowhere. The model is a shared mutable bag, and every invariant depends on the services not to misuse it.

The third cost is that the object becomes indistinguishable from a DTO. An `Account` that is only fields is the same kind of thing as the `AccountResponse` sent over HTTP, and the codebase stops being able to tell its domain model from a transport shape.

## Core Concept

The rich model moves the behavior onto the data. The account that holds the balance is the object that enforces the rules on the balance:

```java
public class Account {
    private Money balance = Money.zero();
    private boolean suspended;

    public Money balance() {
        return balance;
    }

    public boolean isSuspended() {
        return suspended;
    }

    public void debit(Money amount) {
        requireActive();
        if (balance.compareTo(amount) < 0) {
            throw new InsufficientFundsException();
        }
        this.balance = balance.minus(amount);
    }

    public void credit(Money amount) {
        requireActive();
        this.balance = balance.plus(amount);
    }

    private void requireActive() {
        if (suspended) {
            throw new AccountSuspendedException();
        }
    }
}
```

Two things are the whole difference.

First, the methods that change the data, `debit` and `credit`, are the only ways the balance changes, and they enforce the rules independently of any service. The transfer still needs a coordinating service, cross-aggregate work is legitimately a service, but the service now calls `debit` and `credit`, and the suspended check is not something it remembers, it is what the account does.

Second, there are no setters. There is a `balance()` read and no `setBalance`, so there is no sequence of calls in which the balance is overwritten without a rule. The shape of the class is the enforcement. A rule that has no home in the class is a rule the class cannot hold, and the class should carry it.

### The test for an anemic object

The test is one question: can you change the object's data without the object noticing? If a service can `setBalance` and no check fires, the model is anemic. If the only way to change the balance is through a method that guards it, the model is rich.

The second tell is the method names. Anemic objects have methods named `getBalance`, `setBalance`, `getStatus`, a vocabulary of storage. Rich models have `debit`, `credit`, `submit`, `ship`, a vocabulary of the business. A method named for an operation rather than a field is the sign the object carries behavior.

### The honest limit

The rich model is not the absence of services. Cross-aggregate rules still belong in a domain service, and a rich model still has plenty. The difference is what the service contains. In the rich model, the service coordinates, it calls `account.debit(from)` and `account.credit(to)`, and it does the transfer-specific validation that spans the two accounts, while the account keeps its own rule. In the anemic model, the service does the entity's rules, `debit` does not exist, and the service reads the balance, compares, and writes it back.

That is the boundary the whole article points at: a rich service calls the entity's methods; an anemic service reimplements them. The service that coordinates is legitimate. The service that has been handed every entity's judgment is the anemic model wearing the words "logic layer."

### When anemic is fine

The word "anemic" is a verdict, and it deserves a nuance. Some models are pure data and always will be. A DTO, a projection, a read model, a payload boundary, these are bags, and an anemic shape there is not just fine, it is correct. The judgment applies to the domain object, the model that should hold the rule. When a transport shape is a bag and a domain model is a bag, the difference is whether the object was supposed to carry a rule and does not.

## Real Production Usage

Most production Java sits somewhere in the middle, and the honest note is where. JPA entities tend to drift anemic because the framework rewards the bag: annotate the fields, give out the getters, and the service runs the logic. The teams that keep the domain round are the ones that keep the aggregate's own truths on the aggregate and let the service coordinate, a distinction this whole chapter has repeated.

Where production Java is genuinely rich, it is often at the value object: a `Money` with `add`, `minus`, and unit checks, an `EmailAddress` that rejects malformed input at construction, `LocalDate` and `BigDecimal` from the JDK. The value objects carry behavior and hold their invariant. Where it is anemic, it is entity getters leaking and a service with a check a mile long. The team that ships a domain with rules on the object and services that cooperate is the team with a model left to maintain.

## Common Mistakes

**Entities as field bags, services as the owner.** The transfer reads and writes through `getBalance` and `setBalance` in a service, and the account has no behavior. The tell is the method name: `get` and `set` versus the business verb.

**Letting the service own a single entity's rule.** The suspended check belongs on the account, and a service that re-checks it for every operation has turned a rule into a reminder. When the account's own `debit` throws for a suspended account, the caller cannot forget.

**Confusing the DTO with the domain.** An anemic model is a smell precisely when the anemic object is a domain object that should hold a rule. When the shape is a DTO or a read model, anemic is the correct shape, and forcing behavior onto it is a different mistake.

## Interview Perspective

Interviewers ask about rich domain models to hear your verdict on where logic lives. A weak answer says "the rich model has methods." A strong answer says the rich model places the rule with the object that owns the data, that the change methods are the only doors, and that a rich service coordinates while the entity enforces its own invariants.

The first follow-up, "where do the transfer rules live?," is answered strongly as a domain service that coordinates while `debit` and `credit` enforce the entity's rules. The weak answer reads and writes balances in the service. The interviewer is listening for the division: the service coordinates, the entity enforces. And "is an anemic model always wrong?" the strong answer is no, a DTO is anemic and correct, and the smell is the domain object dressed as one.

Common follow-ups:

- "Your account has getters and no behavior. Where do its rules live and who remembered them?"
- "A `Money` with `add` that rejects mismatched currencies, why is that the rich side of the line?"

## Knowledge Check

1. `Account` exposes `getBalance` and `setBalance`, and a service transfers by reading and writing those. Name the shape, and trace the difference between the rule in the service and the guard on the account.
2. The account now has `debit` and `credit` with no setters. What changed about the balance, and why can it no longer be overwritten without the suspended rule running?
3. A class `AccountResponse` used only to ship data to the client has only getters. Is it anemic, and why is that fine, where a domain `Account` with only getters is a smell?

## Key Takeaways

- The rich model places the rule with the object that owns the data and the change methods that mutate it.
- The test: if a service can rewrite the data without a guard, the model is anemic.
- The method names are the tell: `get` and `set` versus `debit` and `submit`.
- A rich service calls the entity's methods; an anemic service reimplements the entity's rules.
- The anemic shape is a smell for a domain object and the right shape for a DTO or read model.

## What's Next

The next article closes the chapter by building one full model. All the pieces, entities, value objects, aggregates, services, repositories, rules, and events, and now the rich versus the anemic boundary, are put into a single walkthrough of one system. The final article assembles them so the separate lessons read as one coherent design instead of a list.

---

This article explains the rich model, which places rules on the object owning the data, against the anemic model with rules pushed into the service. It argues that method names are the tell, and that anemic is a smell only for the domain object.