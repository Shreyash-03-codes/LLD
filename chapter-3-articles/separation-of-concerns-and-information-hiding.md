# Separation of Concerns and Information Hiding

## Learning Objectives

1. Define separation of concerns as splitting code by what it is for, and information hiding as concealing the details that are likely to change.
2. Read a method or a module and name the concerns it is mixing, and the details it is leaking.
3. Explain why information hiding is the mechanism that makes separation safe, and where the two are confused.

## Introduction

Separation of concerns and information hiding are two ideas that travel together and get blurred into one. They are different tools. Separation of concerns says a program should be split by concern, by what each piece of code is for, the HTTP handling, the business rule, the persistence. Information hiding says a module should conceal the decisions that are likely to change, so that changing a decision changes one module, not the whole program.

The two reinforce each other. Separation gives you the boundaries, and information hiding makes the boundaries safe, by making sure the things on one side cannot see the things on the other. You can separate concerns and still leak, if the pieces reach into each other. You can hide information and still have a mess, if the hiding happens inside a blob that does everything. The pair is only complete together.

## Problem Statement

A `PaymentController` handles an HTTP request and, in one method, parses the request body, validates the payload, applies the business rules, talks to the payment gateway, writes to the database, and formats the error response. Every concern is in one method.

The failure shows up as the change problem. A schema change to the payment request touches the parsing. A new business rule touches the validation. A gateway migration touches the call. An error-format change touches the response. All of them touch the same method, so every change to any concern is a change to a method that also owns the other four. The method cannot be tested for one concern without exercising all of them, and a change to the error format is a risk to the gateway call.

The information-hiding failure is hiding underneath. The method does not conceal its decisions, it publishes them. The validation rules, the gateway's shape, the SQL, the response contract, all of it is visible in one place, to every future reader. Nothing is hidden, so nothing is free to change without being noticed everywhere.

## Core Concept

Start with the separation. A concern is a reason the code exists, and the classic list for a web application is the request and response shape, the business rules, the persistence, the infrastructure. Separation is the claim that these should not live in the same method or the same class, because they change for different reasons. The request shape changes when the API contract changes. The business rule changes when the product changes. The persistence changes when the database changes. Four reasons to change, four places, and the engineer who can find a reason to change that touches only one place has separation working.

The layering is the standard map of it, and it is worth drawing because the names confuse people. The web layer speaks HTTP and nothing else, it parses and formats. The service layer speaks the business, it applies rules and orchestrates. The repository speaks persistence, it saves and loads. Each layer depends on the one below it through a narrow contract, and no layer reaches past its neighbor. The payment controller does not write SQL, and the repository does not parse HTTP. The concerns are separated by the layer boundaries.

Information hiding is the second tool, and its best statement is still Parnas's. A module should hide the decisions that are most likely to change, so that a change to those decisions is a change to the module and nothing else. The classic example is a module that hides the representation of a data structure: callers use the module's operations, and the choice of array versus list is a private decision that the module can change freely.

The word that does the work is "likely to change." Information hiding is not secrecy, it is anticipation. The module hides the details that have a reason to change, and exposes the interface that is stable. The database vendor, the storage format, the serialization, the caching strategy, these are the volatile details, and they belong behind a boundary. The business rule, the contract, the operation, these are the stable surface, and they are what callers see.

This is where the two ideas connect, and the connection is the reason they are taught together. Separation splits the code by concern, and information hiding makes each split piece safe by hiding its volatile details. The repository exists because persistence is a separate concern, and the repository hides the SQL and the vendor because those are the details most likely to change. The service calls `repository.save(order)` and knows nothing of SQL, so a database migration is a change to the repository, not to the service. Separation created the boundary, and hiding made the boundary honest.

The failure modes of the pair are the mirror images, and both are everywhere. The first is separation without hiding: a layered codebase where every layer leaks, the controller builds SQL, the service returns a database entity to the controller, the repository knows the request shape. The boundaries exist on paper and are porous in code. The second is hiding without separation: a god class with private fields and private methods that hide the details, but one class owns five concerns, so the hiding protects a mess. The first has the seams without the walls. The second has the walls without the seams.

The judgment question is what counts as a concern and what counts as a detail, and the honest answer is the same test as every other principle in this chapter: what changes together, and what changes separately? The things that change together, the validation rules, belong together. The things that change separately, the request format and the business rule, belong apart. And the detail to hide is the thing that will change, the vendor, the format, the representation, and the interface to expose is the thing that will not, the operation, the contract.

## Real Production Usage

The Spring web stack is separation made into a framework, and you can read the concerns in the class names. The `@RestController` owns the HTTP contract. The `@Service` owns the business rules. The `@Repository` owns persistence. A controller change is an HTTP concern and it does not touch the service. A schema change is a persistence concern and it does not touch the controller. The framework's conventions are the separation, and the interface boundaries are the hiding.

The repository pattern is information hiding in its cleanest production form. The service depends on an `OrderRepository` interface, and the JPA implementation hides the entity manager, the transactions, the SQL, the vendor. The service cannot see any of it, so any of it can change. Swap the JPA implementation for a mock in a test, or a different persistence in another environment, and the service does not notice. The decision, how orders are stored, is hidden, which is exactly what the pattern exists to do.

The messaging side shows the pair at the system boundary. A service that publishes an `OrderPlaced` event depends on a publisher interface, and the Kafka implementation hides the topic, the serialization, the broker. The business code expresses "an order was placed," and the Kafka details are hidden behind the boundary. The concern, business events, is separated from the concern, broker transport, and the volatile transport details are hidden from the business code.

## Common Mistakes

The most common mistake is separating by accident instead of by concern. The layers exist, and the concerns are smeared across them: the controller formats the error, the service builds part of the HTTP response, the repository does validation. The boundaries are drawn, and the concerns crossed them anyway, so a change to a concern still touches every layer. The rule to hold: each layer owns its concern entirely, and a concern that appears in two layers is a concern that was not separated.

The second mistake is hiding nothing. A module whose every internal is public, whose fields have getters and setters, whose decisions are visible in its signature, has no anticipation built in, so the first change to a hidden detail is a change to every caller. The detail that was going to change was not hidden, and the blast radius is the whole module's callers.

The third mistake is over-layering, the reflex that wraps every class in a service, every service in a facade, until the separation is a museum and the concerns are buried under the ceremony. The layering exists to separate concerns that change separately, and a layer added for a concern that does not exist is the speculative abstraction from the DRY and YAGNI article. The concern earns its layer when a change to it actually needs to be isolated.

## Interview Perspective

The question "what is the difference between separation of concerns and information hiding" is the filter. The weak answer blurs them. The strong answer separates them precisely. "Separation is splitting the code by concern, what each piece is for, HTTP, business, persistence. Information hiding is hiding the details that are likely to change, the vendor, the format, behind a stable interface. Separation creates the boundaries, and hiding makes them safe."

The follow-up "give me an example of each" wants the pair shown working. "A repository is both: persistence is separated from the business rule, and the SQL and vendor are hidden from the service. The separation is the boundary, the hiding is why a database migration does not touch the business code."

The sharper question: "isn't information hiding just encapsulation." The strong answer places them. "Encapsulation is the object-level wall around a class's state. Information hiding is the module-level wall around the decisions that change, which can be bigger than state, a vendor, a format, an algorithm. Encapsulation is one tool; information hiding is the broader rule about what belongs behind any wall."

## Knowledge Check

1. A controller method parses the request, calls a gateway, and formats the response, all in one method. Name the concerns and the layer each one belongs in, and what changes would each one cause independently.

2. A repository exposes `getOrderById` and returns the JPA entity to the service. Which detail did the repository fail to hide, and what would a change to the entity shape now touch?

3. A team adds a facade layer between every service and every controller. Apply the concern test to the decision and say whether the layer earns its existence, and what evidence would prove it.

## Key Takeaways

- Separation of concerns splits code by what it is for; information hiding conceals the details that change.
- Separation creates the boundaries, and hiding makes them honest, the pair is only complete together.
- The repository is both at once, persistence separated from the business rule, vendor and SQL hidden from the service.
- The concern test is what changes together and what changes apart, and a concern in two layers was never separated.

## What's Next

Every principle in this chapter has claimed the same payoff: a change that is cheap, contained, and safe. The last article makes that claim measurable. Designing for Testability argues that the ability to test a class in isolation is the only evidence that the loose coupling and the hidden information actually worked. It covers the seams that make a design testable and the ones that do not.

---

This article explains separation of concerns as splitting code by what it is for, and information hiding as concealing the details that change. Its strongest claim is that separation creates the boundaries and hiding makes them honest, and that the repository is both at once, the vendor hidden from the business code.
