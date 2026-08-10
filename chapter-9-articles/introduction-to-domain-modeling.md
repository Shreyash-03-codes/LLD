# Introduction to Domain Modeling

## Learning Objectives

- Define a domain model as the part of the codebase that mirrors the business vocabulary and rules, and explain what the mirror is for.
- Distinguish the domain model from the persistence, transport, and UI layers, and name the boundary where the model ends.
- Recognize the two failure modes this chapter exists to fix: the anemic model and the leaking model.

## Introduction

A domain model is the shape your code gives to the business it serves. When an order system uses classes called `Order`, `Customer`, `Shipment`, when a banking system uses `Account`, `Transaction`, `Limit`, that vocabulary is the domain model. It is not a schema and not a data structure. It is the business logic made into objects, holding both the data the business cares about and the rules that keep that data honest.

This chapter is about building that model deliberately instead of by accident. The first article frames what the model is and where it belongs, because every later decision, entity, value object, aggregate, repository, depends on the boundary this article draws.

## Problem Statement

Every codebase that touches real money or real workflows ends up with a class named after a business concept. That much is unavoidable. The problem is what that class is allowed to be.

The failure shows up in two ways, and you have seen both if you have worked on a backend for more than a year.

First, the data object. A class that is nothing but getters, setters, and an `equals` method, carrying fields between the database and the API, with no behavior anywhere near it. The business rules that should live on the object live in the service layer instead, so `transfer()` lives in a service that reads the account balance, checks a limit, and writes back, while `Account` itself does nothing. The object is a bag of state, and every rule about it is scattered across services that know its every field.

Second, the framework object. A class that is `@Entity`, `@Table`, `@Column` annotated top to bottom, with lazy proxies and Hibernate metadata woven through the business logic. The rules get tangled with the persistence concerns, and the model can only be reasoned about in the presence of a running database.

Both failures are the same root: the model is not owned by the business logic. It is owned by the database or the framework or the service layer, and the business rules have no home. This chapter exists to give the rules a home, and that starts with deciding what the model actually is.

## Core Concept

A domain model is a map, not a copy. It models the business, not the database and not the API. The map analogy carries: a map of a city leaves out the pipes and the wiring, and includes the roads and the districts because those are the things you navigate by. The domain model leaves out the persistence details and the transport details, and includes the business concepts and rules, because those are the things the business reasons about.

Three properties make a domain model a model instead of a dump.

### It speaks the business language

The class names and method names come from the business, not from the technology. An `Order` with a `submit()` method, a `Customer` with `isBlacklisted()`, a `Shipment` with `delay(Duration)`. When a product owner says "an order cannot ship before it is confirmed," there is a method somewhere that is literally the code form of that sentence. If the code speaks a different language, "the order object has a status field that is updated by the service", then the model is not a model of the business, it is a model of the database.

### It holds the invariants

An invariant is a truth the system must preserve. An account cannot go below its overdraft limit. An order in a `shipped` state cannot be `cancelled`. A customer on the blacklist cannot place a new order. The domain model is where invariants live, because they are business rules and they need a permanent home. When the invariant is enforced by a check in the controller, the rule can be bypassed by the next caller that forgets to check. When it is enforced by the object itself, it cannot be bypassed at all.

### It is persistence-ignorant

The model does not know about the database. It has no `@Entity`, no `@Column`, no session, no transaction. It is plain Java objects holding state and behavior. Persistence is an adapter that maps the model to the storage format, and it lives outside the model. This is the property that most backend developers find hardest, because the framework invites the opposite: the natural move is to let the ORM's annotations decorate the domain classes. The discipline of keeping the model clean is what the rest of this chapter builds on, and it is what makes the model testable without a database and readable without a debugger.

### The model boundary

The model is the center. Around it sit the adapters: the repository that stores it, the controller that exposes it, the DTOs that transport it, the mapper that converts it. The boundary rule is simple: the adapters may reference the model, the model must never reference the adapters. If the model imports a web framework, or a persistence annotation, or an HTTP class, the boundary has been crossed and the model is no longer a model.

This one-way dependency is not a nicety, it is what keeps the rules testable. A domain model with no imports from the framework can be unit-tested with zero setup, because nothing outside the test needs to exist. That is the property every later article in this chapter assumes.

## Real Production Usage

The pattern that production Java has converged on is the layered architecture with the domain model in the middle. Spring Boot projects that survive past their first year tend to settle into: controllers that translate HTTP, services that orchestrate, a domain model that holds the rules, and repositories that persist. The well-run ones keep the domain model free of `@Entity` annotations and treat the ORM mapping as a separate layer.

Spring's own guidance and the conventions in mature codebases point the same way: the transaction boundary is a service method, the domain model is plain Java, and the repository is an interface the model depends on only through abstraction. Frameworks like Spring Data JPA support this by letting you define the repository as an interface while the domain class stays clean. The framework is not the enemy of the model; the enemy is letting the framework own the model.

The honest version of this, from real codebases, is that most teams do not keep the model perfectly pure. There is a spectrum from "annotations only on the mapping layer" to "annotations all over the domain objects," and the teams that succeed long-term keep the business rules away from the persistence metadata. This chapter argues for the cleaner end of the spectrum, and the walkthrough article at the end shows what that looks like in one system.

## Common Mistakes

**Letting the database define the model.** The most common origin of a bad domain model is designing the classes from the tables. The tables are a storage decision; the model is a business decision. When the two disagree, the model should win and the mapping should adapt. Most engineers get this backwards because the database is visible and the model is not.

**Putting rules in the service layer because it is easier.** It is always easier in the moment to write the check in the service. The cost is paid later, when the same rule has to be checked in three places and one of them forgets. The rule belongs in the object, once, and the services call it.

**Decorating the domain with the framework.** Adding `@Entity` and `@Column` to the domain classes feels like the framework's intended use. It is the intended use for a persistence model, not for a domain model. When the business logic is entangled with lazy loading and session proxies, the model cannot be reasoned about alone.

## Interview Perspective

Interviewers asking about domain modeling are testing whether you can reason about where logic lives, not whether you can recite a definition. A weak answer defines a domain model as "the classes that represent the business data." A strong answer says it is the business rules made into objects, that it speaks the business language, holds the invariants, and is persistence-ignorant, and that the adapters depend on it rather than the reverse.

The follow-up that sorts people is "where does a business rule live, the service or the entity?" The strong answer: on the entity, because an invariant enforced by the object cannot be bypassed, while a rule checked in a service is a rule that every future caller must remember. The second follow-up is "should the domain class have `@Entity` on it?" and the strong answer is no, the mapping belongs to a persistence layer, so the model stays testable and the rules stay clean.

Common follow-ups:

- "An order cannot ship before confirmation. Where does that check live?"
- "The domain model and the database schema disagree on the shape of an order. Which one changes?"

## Knowledge Check

1. A class named `Order` has fields, getters, and setters, and every rule about orders lives in a service. Name the failure and the fix this chapter proposes.
2. An invariant is "an account cannot go below its overdraft limit." If it lives in the controller, what can go wrong, and what is the alternative home?
3. The domain model must be persistence-ignorant. What breaks, concretely, the day someone puts `@Entity` on the domain classes?

## Key Takeaways

- The domain model is a map of the business, not a copy of the database or the API.
- It speaks the business language, holds the invariants, and is persistence-ignorant.
- The adapters depend on the model; the model never depends on the adapters.
- Rules that live on the object cannot be bypassed; rules that live in services must be remembered.
- The rest of this chapter builds on the boundary this article draws.

## What's Next

The next article is about the two kinds of objects that make up a domain model: entities and value objects. The distinction between them, one has identity and one is interchangeable, drives everything from how you define `equals` to how you decide what belongs in a table. We will cover the tests that separate the two and the common case where teams model everything as an entity.

---

This article explains what a domain model is, a map of the business rather than a copy of the database, and why the model owns the rules. It argues that a rule enforced by the object itself is the only one that cannot be bypassed.