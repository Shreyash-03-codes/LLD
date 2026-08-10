# Domain Services

## Learning Objectives

- Recognize the operation that belongs to no single aggregate and put it in a domain service instead of forcing it onto an entity.
- Argue the deciding test for service membership, does the rule describe one object or a transaction across objects, without sliding into anemic entities.
- Keep domain services stateless and named for the operation, so a stateless class full of verbs is a service, not a thing.

## Introduction

A domain service is an operation that belongs to the domain but to no single aggregate. Transferring money between two accounts touches two aggregates. Checking an order against a customer blacklist touches two aggregates. Recommending a route touches many. These operations have real business meaning, and at the same time they are not the job of any one object, because they cross object boundaries.

This article is where the chapter admits that a model built only of entities and value objects cannot hold every rule. Some rules are about several aggregates at once, and those rules live in a domain service.

## Problem Statement

The temptation for an operation that touches two aggregates is to force it onto one of them. So a transfer ends up as a method on the account:

```java
public class Account {
    private Money balance;

    public void transfer(Account target, Money amount) {
        if (balance.compareTo(amount) < 0) {
            throw new InsufficientFundsException();
        }
        this.balance = this.balance.subtract(amount);
        target.balance = target.balance.add(amount);
    }
}
```

That works, and it quietly breaks the model. The `transfer` method reaches into the other account's `balance` field directly, which is the cross-aggregate reference we said to avoid. The receiving account is changed without going through any of its own operations, and the aggregate boundary exists in theory and nowhere else.

The other tempting home is the application layer next to the controller. The transfer transaction, the withdrawal limits, the currency conversion, all the real business rules, end up in a method that also handles HTTP and transactions. The rules are now in the wrong layer, mixed with plumbing, and they cannot be tested without a request.

Both placements fail the same way. A cross-aggregate rule is put somewhere it does not belong, on an entity that should not reach into another aggregate, or in a framework-adjacent layer that should not hold domain rules at all.

## Core Concept

A domain service is the way a transaction that no single object owns is represented. It is a class with real business rules, named for the operation, and it takes the aggregates it needs as parameters.

```java
class TransferService {
    private final AccountRepository accounts;

    TransferService(AccountRepository accounts) {
        this.accounts = accounts;
    }

    public void transfer(AccountId from, AccountId to, Money amount) {
        if (amount.isNegative() || amount.isZero()) {
            throw new IllegalArgumentException("transfer amount must be positive");
        }
        Account debit = accounts.find(from);
        Account credit = accounts.find(to);
        if (debit.balance().compareTo(amount) < 0) {
            throw new InsufficientFundsException();
        }
        debit.debit(amount);
        credit.credit(amount);
    }
}
```

Note what changed from the broken version. The service holds the orchestration and the validation that spans the two accounts. Each account is still the only class that mutates itself, through its own methods, `debit()` and `credit()`, `withdraw()` and `deposit()`. The service owns the coordination, the entities own their integrity, and neither account ever reads the other's balance.

### The deciding test

The test for whether a rule belongs on an entity or in a domain service is one question: does this rule describe the life of a single object, or a transaction between objects?

If the rule is about the life of one object, its method belongs on that object. `withdraw()` is an account rule. `isOverdrawn()` is an account rule. Overdraft fees and daily limits belong there too, because they are each the account's own concern. If the rule is about several objects cooperating, a transfer, a settlement, a matching, it belongs in a domain service.

The smell that says a method has moved past its own boundary is when it reaches into another entity's internals. The moment `Account.transfer` reads `target.balance`, the rule has crossed from a single-object concern to a cross-object concern, and it needs to move to a domain service.

### Don't abdicate the entities

The domain service is easy to reach for as an escape hatch. The lazy path is to move every rule to a service and leave the entities as bags of fields. That path produces the anemic model this chapter critiques later. The guard is the deciding test above. A service that duplicates a rule an entity already owns is a sign the entity was emptied. The benchmark: every rule in the system has exactly one home, and most rules live on entities, with domain services holding only the cross-aggregate few.

### Naming and shape

Domain services are named for what they do, in the business's words, `TransferService`, `FraudCheckService`, `RouteCalculator`. They are stateless, so one instance serves many ops, and they can be injected like any collaborator. A service is a set of verbs. If a domain service starts exposing getters over a quantity of held state, or a method that reads like it owns a record, it has drifted toward being an application service or an entity, and it is in the wrong place.

## Real Production Usage

Domain services show up wherever a transaction is genuinely cross-aggregate. The transfer is the canonical case, and payment and settlement systems run on domain services that move money between aggregates while each aggregate guards itself. A pricing engine that totals an order across the order, the discounts, and the product is also a candidate, because the total is a computation over several things.

In a Spring backend, a domain service is a `@Service` bean with no HTTP concern, no `@Controller`, no `@RequestMapping`. It is injected with the repository interfaces it needs, and it is called from the application service after the request boundary. This distinction is worth being explicit about. An application service owns the use case, the transaction, and the plumbing; a domain service owns the business rule. Teams that collapse the two end up with the orchestration rules tangled in the controller and the domain rules untestable.

The testing here falls out. Because the service takes the repository by interface and deals only in domain objects, a unit test can drive it with a fake repository and assert that the money moved, the limits held, and the invalid amount threw, all with no database and no web. That is the same testability the chapter has been building on, and the domain service keeps it by not importing the framework.

## Common Mistakes

**Putting a cross-object rule on an entity because it is easy.** The transfer that mutates the target directly is the standard example. If the code reaches into another aggregate's internals, the rule belongs in a domain service, not on the entity.

**Making the service fat by moving every entity rule out.** The anemic model is the other extreme, and the test that catches it is counting where the rules live. If the entities are empty and the services carry everything, the model is a data layer wearing a domain costume.

**Writing the service against the web layer.** A `TransferService` with `@PostMapping` and the session inside is not a domain service, it is an application service that swallowed the domain rules. Keep the domain service and the application service in distinct layers, so the rules stay testable.

## Interview Perspective

Interviewers ask about domain services to check whether you place a rule by reasoned decision. A weak answer says a service is "a class with methods." A strong answer gives the deciding test: if a rule describes one entity it belongs on that entity, and if it describes a transaction across entities it belongs in a domain service. And the first signal is a method reaching into another object's fields.

The follow-up that sorts candidates is "where do the transfer-between-two-accounts rules belong?" The strong answer is a domain service that holds the coordination and the amount check, with each account mutating itself through its own methods. The specific that not everyone states is that neither account ever reads the other's balance.

A second follow-up is "between a domain service and an application service?" The strong answer is the split of responsibility: an application service owns the use case and the transaction and the request, a domain service owns the business rule and the names. A team that collapses the two loses the ability to run the rules without the framework.

### Why the boundary stays slippery

Do not over-weigh what is easy to place. The transfer is classic, and the boundary, what stays on a single aggregate and what spans two, is the same test you have been drawing since the aggregate article: what must change together, and who is allowed to reach into whose insides. A rule that no debit may leave an account negative stays inside the entity. A rule that a transfer may not reverse an earlier transfer must look at both sides and judge, and that belongs in a domain service.

### How the design should read

Once the transfer sits in the domain service, the code reads like the business rule. The "cannot transfer a negative" is a sentence the domain can say in the business vocabulary, and the accounts stay narrow and self-contained. That is the design marker: the placement of rules is a test of where the model holds the behavior. A model where every rule went through the deciding test reads as coherent, and a model where rules pile up near controllers reads as something you will be moving later.

## Knowledge Check

1. "A transfer between two accounts touches two aggregates." Does its rule belong on an account entity or in a domain service, and which part of the deciding test tells you?
2. A rule "an account cannot withdraw more than the balance available" is about the life of one account in the same way a transfer is about two. Where does the withdrawal rule live, and why is it a different home than the transfer rule?
3. A newly introduced class called `TransferService` contains `@PostMapping` and a session object. What has this class become, and what should the real `TransferService` hold?

## Key Takeaways

- A domain service is the home of a rule that touches more than one aggregate.
- The deciding test: the rule describes one entity, or a transaction across objects.
- Keep the same majority of rules on entities and reserve services for the cross-object few.
- The reach into another entity's internals is the upfront signal of the boundary being crossed.
- Keep the domain service free of web and persistence wiring, so the rule stays testable in a unit test.

## What's Next

The next article is about repositories. The domain service in this article reached through a repository to load the two accounts it coordinated, and the repository is how the model asks for its aggregates without knowing about the database. We will cover the interface the domain depends on, the adapter that stores and loads, and the query methods a repository should and should not expose.

---

This article explains when a business rule belongs in a domain service using a money transfer across two accounts. It argues that the test is whether the rule describes one object or a transaction between objects.