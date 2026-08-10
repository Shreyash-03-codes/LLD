# High Cohesion and Loose Coupling

## Learning Objectives

1. Define cohesion as what belongs inside a module and coupling as what a module reaches outside itself.
2. Read a class for cohesion and a dependency graph for coupling, and say which one each change symptom points at.
3. Explain why loose coupling is a means and high cohesion is the goal, and where the two trade off.

## Introduction

High cohesion and loose coupling are the oldest pairing in design, older than SOLID, and they are almost always taught as a single phrase that nobody unpacks. They are two different measurements. Cohesion measures the inside of a module: do the things in here belong together? Coupling measures the outside: how much does this module depend on the others, and how much do the others depend on it?

A good module is both. The things inside it are tightly related, and the module leans on the outside world as little as possible, through stable, narrow contracts. When a design goes bad, the two symptoms arrive together, because a class with unrelated jobs is usually also a class that reaches into a lot of other classes to do them.

## Problem Statement

A `BookingFlow` class has grown a second career. It validates the booking, applies the pricing, writes to the database, sends the confirmation, and updates the analytics. The methods inside it are unrelated in the "why" sense, and to do all of them it reaches into the `UserRepository`, the `PaymentGateway`, the `EmailClient`, the `AnalyticsTracker`, and the `InventoryService`.

The symptom that shows up in every review is coupling: "this class depends on five things." And it is true. But the coupling is not the disease, it is the symptom. The class depends on five things because it is doing five jobs. Split the jobs, and the five dependencies separate along with the jobs. The `BookingFlow` that only books has one or two dependencies. The coupling measurement went down because the cohesion went up. The engineer who only sees the coupling and tries to "reduce dependencies" by wrapping them in a facade has moved the coupling around without fixing it.

That is the insight that makes this article worth writing: most coupling complaints are cohesion problems wearing a different label, and treating the coupling without the cohesion is rearranging the disease.

## Core Concept

Cohesion is the answer to one question about the inside of a module: do the things here change together, for one reason, at one time? The answer is measured in the same currency as the Single Responsibility Principle, and this is not a coincidence, SRP is the operational form of cohesion. A class where every method touches the same state, the same rule, the same concern, is highly cohesive. A class where half the methods touch the booking state and half touch the email template is not.

The classic cohesion ladder is still the clearest way to see the range. The best is functional cohesion, where every method contributes to one function, an algorithm, a rule, a flow. The worst is coincidental cohesion, where the methods are grouped because they happened to be in the same file. In between are the accidental groupings, methods that share state but not purpose, or share a module because they are convenient. Functional cohesion is the target, and everything below it is debt.

Coupling is measured between modules, and it comes in two flavors that matter differently. Content coupling, reaching into another class's internals, a public field, a getter that returns the internal list, is the worst, and it is the encapsulation violation from an earlier chapter. The good kind is message coupling: a module calls another's public method and knows nothing else about it. The spectrum is how much one module knows about another's internals, and the goal is the minimum that still lets the module do its job.

The two measurements interact in a way that confuses the usual advice. Loose coupling is not achieved by wrapping dependencies in a facade, a gateway, a coordinator. A `BookingFacade` that hides the five dependencies behind one object still has a class doing five jobs, it just does them behind a door. The coupling count is unchanged, the cohesion is unchanged, and the codebase gained a layer. Coupling drops when the dependency graph gets thinner, when modules stop reaching into each other, and that happens when the jobs are separated, not when the reach is hidden.

The other direction of the interaction is the trade that is always present. Coupling is not something you minimize to zero. A module that depends on nothing, a pure function, is trivially decoupled, and it is also useless, because it cannot touch the database, the network, or any other module's state. The real target is that a module's dependencies be few and stable, so that when a dependency changes, the blast radius is small. A `BookingService` that depends on one repository interface and one pricing interface is loosely coupled even though it depends on things, because each dependency is narrow and stable.

The stability of the dependency matters as much as its count. A module that depends on three stable interfaces is better off than a module that depends on one class that changes every week. The coupling that hurts is the coupling to the volatile, the concrete class, the package that moves, and this is the same direction the dependency inversion principle was pushing. Loose coupling and DIP are the same argument: point your dependencies at the stable seams, not at the moving parts.

Cohesion is the goal, and coupling is the means, and the reason is a claim about change. A highly cohesive module has one reason to change, so a change to that reason touches one module. Loose coupling makes that change cheap, because the module that changed does not drag its neighbors along. The two work as a pair, but the causality runs one way: fix the cohesion, and the coupling follows. Fix the coupling with a facade, and the cohesion is still broken.

## Real Production Usage

The layered architecture is the standard picture of both working together. The service layer holds cohesive units, one service per use case, each depending on a repository interface and maybe a collaborator. The repository is a narrow seam to the persistence, and the service knows nothing of the SQL. A schema change touches the repository, and the service is untouched, because the coupling is to the interface, not to the database. That is high cohesion in each layer and loose coupling between them.

Spring makes the boundaries visible in the bean graph. The dependency injection container draws the coupling for you: a service with two constructor arguments depends on two beans, and the container resolves them. A service with seven constructor arguments is a smell you can read at a glance, and the count is the coupling measurement made visible. The bean with one narrow dependency and one job is the well-shaped unit.

The hexagonal architecture is the same idea drawn as a whole. The application core holds the cohesive domain rules, and the adapters, HTTP, persistence, messaging, hang off the core through ports. The core depends on the ports, the narrow contracts it owns, and the adapters implement them. The coupling between the domain and the outside world is reduced to the port width, and the cohesion of the core is protected from the outside world's noise.

## Common Mistakes

The most common mistake is treating the coupling symptom without the cohesion cause. A class with five dependencies gets a facade, and the five dependencies are now hidden behind one, and everyone declares victory. The class still does five jobs, the five jobs still change at different times, and the facade is a layer that must be maintained to hide the problem. The fix is to split the jobs, and the dependencies split with them.

The second mistake is minimizing coupling to zero. A module that calls nothing and is called by nothing is perfectly decoupled and useless. The target is not no dependencies, it is few and stable dependencies, pointed at seams that do not move. The engineer who treats every dependency as a defect ends up with a design of pure functions that cannot touch the real world.

The third mistake is measuring coupling by file count or class count. The `BookingFacade` added a class and the coupling measurement, in the sense of what knows about what, did not change. The measure that matters is how many modules a change touches and how much of each module's internals the others know. Count the blast radius, not the classes.

## Interview Perspective

The question "what is the difference between cohesion and coupling" is usually answered with definitions, and the interviewer pushes. The strong answer adds the relationship. "Cohesion is what belongs inside a module, and coupling is what it reaches outside. They interact, a class doing five jobs reaches into five things, and the coupling is the symptom of the broken cohesion. Fix the cohesion and the coupling follows; wrap the coupling in a facade and the cohesion is still broken."

The follow-up "is loose coupling always good" wants the nuance. "No. The target is few, stable dependencies, not zero. A module that depends on nothing is useless, and a module that depends on three stable interfaces is better than one that depends on one volatile class. What hurts is coupling to the things that change."

The sharper question: "how do you measure coupling." The strong answer names the blast radius. "By what a change touches. When a repository changes, the service is untouched, that is loose. When a change to one class edits four others, that is tight. The count is the reach, not the number of classes."

## Knowledge Check

1. A `ReportService` has five dependencies: the query builder, the formatter, the storage, the emailer, and the scheduler. Which single change to the code most reduces its coupling, and why does it also raise its cohesion?

2. Two modules both depend on the same stable interface. One also depends on a concrete class that changes every sprint. Which is more loosely coupled in the sense that matters, and what is the principle that states the rule?

3. A team adds a facade around a god class to "reduce coupling." Explain why the coupling measurement did not change, and what the facade actually added.

## Key Takeaways

- Cohesion is what belongs inside a module; coupling is what it reaches outside, and most coupling complaints are cohesion problems.
- Functional cohesion is the target, and splitting the jobs splits the dependencies with them.
- Loose coupling means few, stable dependencies at seams that do not move, not zero dependencies.
- The coupling that matters is the blast radius: how many modules a change touches.

## What's Next

Cohesion and coupling govern the shape of a module and its reach. The Law of Demeter is a precise rule about that reach: what a method is allowed to touch. The next article covers the train wreck, the friends-versus-strangers line, and why the dot count is a heuristic for a much deeper violation of encapsulation.

---

This article explains high cohesion and loose coupling as the inside and the outside of a module, and how the two interact when a class does too many jobs. Its strongest claim is that most coupling complaints are cohesion problems in disguise, and that the coupling that matters is the blast radius of a change, not the number of classes.
