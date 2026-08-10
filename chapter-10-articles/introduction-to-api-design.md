# Introduction to API Design

## Learning Objectives

- Define the API as a contract with real consumers, and name the two users a design must satisfy.
- Separate the concerns the API owns, the contract and the transport, from the ones it does not, the implementation.
- Recognize the two classic failure modes: the overloaded endpoint and the mirror-of-the-database endpoint.

## Introduction

An API is a contract. It is the surface your application shows the world, and the moment it ships, clients start talking to it. You can change the code behind it freely, but the contract is a different thing. Your callers have compiled against it, mocked it, cached it, and written tests around it. Everything else in this chapter leans on that one fact: an API is a contract, and you want to get it right before it goes live, because revising a live contract is slow and expensive.

This is not the same exercise as defining a well-named method. An API is the whole deal at once: the resources, the verbs allowed on them, the shape of every request and response, how failures are reported, and what a caller can expect from a retry. This first article frames what good API design actually is, so the rest of the chapter can hand you the specific rules without you losing the point.

## Problem Statement

There are two archetypal failures, and you have both if you have warmed a backend for more than five years.

The first is the overloaded endpoint. A single POST `/orders` that takes a request able to hold anything. The caller can create an order, draft one, replace one, or mark one paid, all driven by a `type` or `status` field somewhere in the JSON. The endpoint accreted options one at a time until it stopped being one thing. Now it is a case statement that nobody can reason about, and every caller goes through the same door with different intent. Touching it breaks someone.

The second is the mirror-of-the-database endpoint. GET `/users` returns the user row straight out of the ORM. Ids, timestamps, foreign keys, a lazy loading proxy, all of it, serialized as the JSON. The endpoint is a wide portal to your tables, not a designed interface. Callers reach into your internals, your storage schema becomes part of your public shape, and you cannot rename a column without breaking a caller who never should have seen it.

Both failures are the same mistake wearing different clothes. The API was treated as a thin alias for the database or for the service, not as its own artifact with its own shape and opinions. The fix is to design the API deliberately, which is what the whole chapter does.

## Core Concept

Before an endpoint gets written, an API design has to settle a few questions.

### The contract outlasts the code

The defining property of an API is the asymmetry of time. Rewrite the internals of a service, change storage, refactor the domain, and no caller notices. The client saw the same contract, so the change is invisible. The reverse is not true. Change the contract, rename a field, drop a parameter, change a status code, and every caller that was built against the old shape has a problem, often at a time you do not control.

This is why API design work pays off out of proportion to its size. It is one of the only places in the codebase where a mistake you make is multiplied by everyone downstream of you. And once consumers are live, you cannot quietly undo it in the way you can with a refactor.

The practical consequence is to prefer the shape that is easy to extend later over the one that is leanest today. Additive changes, a new field with a default, a new endpoint next to an existing one, are cheap. Removing, renaming, or changing the meaning of an existing field is where the pain lives. A lot of chapter is advice on keeping your changes on the additive side of that line.

### Two users

Every API has two audiences, and a design that serves only one is broken.

The reader is a person, usually the next engineer at the next desk. They want the surface to be guessable: resources as nouns, verbs that match the operation, error shapes consistent across the whole API. The first time they read it, they should not have to consult the docs to see what a call does. The reader's need is legibility.

The second audience is the machine, frequently a rigid one you cannot control. An SDK that is pinned to an old version, a partner integration built on a frozen contract, a batch job that retries blindly. That consumer cannot ask a question. It only does what the code does. It demands strictness, explicitness, and a shape that does not leave one field open to interpretation.

These two pull in opposite directions, which is why the work is a discipline and not a checklist. The legibility the person wants and the strictness the machine needs can happily be the same contract, and a good chunk of this chapter is the rules that get you there for both at once.

### The API is a boundary layer, not the application

A clean way to think about it: the API is a layer in front of your application, not another name for it. The HTTP boundary, controllers, DTOs, marshalling, exists to carry a contract. It should be thin by design. The minute a controller starts doing the real business logic, or a query, or computing a total, the layering has come down and the API has become the application.

The tell when reading code is whether a request is being converted or executed. A controller that converts a DTO, calls a service, and maps a result back, is the boundary earning its living. A controller that embeds a query, invokes a second service, or mutates a couple of aggregates before returning, has left the boundary and is now the implementation. The API layer converts; it does not do the work.

## Real Production Usage

Every live API you have used is a working example of contract design, and they all converge on the same lesson. Stripe exposes clean noun resources and does not let the surface drift. GitHub has carefully versioned an enormous API. The pattern is not framework-specific: it is the behavior of a stable, guessable contract that does not change under its consumers.

This is the Java habit that matters most, precisely because it is easy to miss. Spring's `@RestController` gives a fast path to annotate a domain object and hand it back as JSON. That is the quickest route to the mirror-of-the-database endpoint. Nothing in the framework stops you, so the discipline has to come from the design, not from the tooling. The tooling produces; the contract is your job.

### What follows your answer

Every endpoint is a short decision tree, and designing the whole tree is the work. For each resource and verb you should be able to say:

- The path, which resource and whether it is a collection or a single item.
- The verb, what the operation does in a word.
- The request shape, what the caller must send, and what is allowed to default.
- The response shape, what the caller can rely on getting back.
- The error set, what happens when the call is malformed, forbidden, or missing its target.
- The pagination when the collection is large, how a caller walks all of it.

Most API surfaces are not bad because any single answer was wrong. They are bad because the same question gets a different answer at every endpoint. Consistency is the cheapest virtue in API design, and it is the one that decides whether the surface feels designed or patched together.

## Common Mistakes

**An endpoint for each service method.** `POST /api/orders-create`, `POST /api/orders-cancel`, a folder of RPC-shaped URLs, is the fastest way to inherit the volatility of your service layer. The endpoint vocabulary changes, the caller's coupling to the internal method names is total, and every internal refactor shows up in the contract.

**Designing only the success path.** Documenting the happy JSON and leaving the callers to guess at the errors, the status codes, the pagination limits is a contract with holes in it. Integration bugs live disproportionately in exactly those holes. Specify failures as carefully as the success case.

**One endpoint, several contracts.** Reusing a single endpoint and flipping behavior with a field, or answering differently for different clients, is how a contract becomes undocumentable. If the shape is genuinely different, give it different name and a different endpoint. A flag on the same endpoint that changes the meaning is a function that no client can reason about.

## Interview Perspective

Interviewers probing API design are testing whether you treat the API as a product rather than an afterthought of the controller. A weak answer describes the mechanics, how a request maps to a method. A strong answer says the API is a contract with consumers, the design must serve a human reader and a rigid runtime, and the common failures, the overloaded endpoint and the mirror-of-the-database endpoint are about mistaking the API for a layer over the system.

The follow-up that separates people is "what is the hardest thing to change in a live API?" The strong answer is anything already shipped: a removed or renamed field, a resource you promised. Extensions are cheap, restatements are not. If the candidate talks about changing buttons or the implementation, they have never had to keep somebody else's client working.

Common follow-ups:

- "A client depends on a field you are about to remove. What are your options?"
- "Is business logic allowed in a controller, and where is the line?"

## Knowledge Check

1. An endpoint returns your domain entity as JSON, including ORM lazily-loaded relationships. The day you rename the underlying table, what is the concrete cost, and who pays it?
2. You have an interface that is easy to change now but vaguely specified, and one that is strict but heavier to use. Which do you protect for a public, long-lived contract, and why?
3. Name two audiences of an API that pull in different directions, and give one design decision that serves both at once.

## Key Takeaways

- The API is a contract, and it outlives the implementation behind it.
- Additive changes are cheap; the breaking changes are the ones to avoid.
- Design the strict shape, a human can read it and give a deterministic answer to the machine.
- The API layer is a converter at the boundary, not the object named by the caller.
- Consistency across the surface is the cheapest thing that makes the API feel designed.

## What's Next

That is the framing; the chapter is about the rules. The next article is REST and resource modeling, where the abstract contract turns into concrete nouns and verbs. It covers how to decompose a problem into resources, what a collection and an item look like in the path, and how to map the standard verbs to the operation a caller actually wants to express. That is the next place to anchor the theory.

---

This article explains what an API is, a contract with consumers you do not control, and frames its two failure modes: the overloaded endpoint and the mirror-of-the-database endpoint. It argues that the interface outlives the code and that additive changes are the only cheap ones.