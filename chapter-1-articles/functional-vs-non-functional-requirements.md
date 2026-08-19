# Functional vs Non-Functional Requirements

## Learning Objectives

- State the difference between functional requirements and non-functional requirements in one sentence each.
- Identify which requirements actually constrain your architecture and design choices.
- Turn a vague non-functional demand like "must be fast" into something you can design against.

## Introduction

A functional requirement is something the system does. A non-functional requirement is a constraint on how well it does that thing. "The user can transfer money between accounts" is functional. "A transfer settles within two hundred milliseconds under peak load" is non-functional. One tells you what to build, the other tells you how the build must behave while it works.

The asymmetry is that teams treat the functional list as the whole requirement and the non-functional list as an afterthought at the bottom of the doc. That is backwards. The functional requirements tell you what a version one system could look like. The non-functional requirements are what disqualify three-quarters of those designs. Most designs that die in review die on a non-functional requirement they never wrote down.

## Problem Statement

A team builds a chat application and writes a long, careful functional spec: create account, create room, send message, list history. All of it is precise and testable. Nothing appears about how many messages per second a room must sustain, or how fast a message must appear on another screen, or what happens to the read state when two devices sync at once.

The design review goes great. The build goes great. In production, five thousand people join one room, the per-user disconnect-and-replay loop collapses the server, and messages lag by minutes. The feature works. The system fails. Every green light from the functional spec was true and irrelevant, because the real constraint was latency and scale under load, and it was never written down. The team is not at fault for building it wrong, it built exactly what was specified.

## Core Concept

Functional requirements describe behavior: the inputs, the outputs, the operations, the rules. They are what a stakeholder can demonstrate by clicking. "Refund the customer within one click of the order page" is functional. They are cheap to collect and cheap to verify, which is why they dominate every backlog and every interview question about "requirements."

Functional: behavior the system performs | Non-functional: qualities of that behavior
--- | ---
"User can upload a file" | "Upload of 1GB completes in under 60 seconds"
"System computes a discount" | "Correct with 8 decimal digits of rounding"
"User can reserve a seat" | "Two users never reserve the same seat"
"User receives a notification" | "Notification arrives within 1 second of the event"

Non-functional requirements split into a few families that keep recurring. Performance and latency, the speed and throughput. Scalability, how the numbers hold up as load grows. Availability and reliability, how often it is up and how long it takes to recover. Security, who can do what and who can see what. And a quieter one that the others make possible: capacity and resource usage, memory, disk, connections.

The reason these decide your architecture is that functional requirements barely move the structure. You can meet "user places an order" with a monolith or an event-driven set of services, with a relational database or a log, with sync or async. The non-functional ones close those options. If the requirement is two thousand orders per second with sub-second acknowledgment, you cannot hand-roll a single-threaded code path. If it is "a two-user pilot," the simplest database is fine. The functional requirement stayed the same; the architecture was decided by the non-functional ones.

The most important move in this subject is turning qualities into numbers. "Fast," "reliable," "scalable," mean nothing until someone says how fast, how reliable, how scalable. Real requirements carry thresholds and worst cases, not adjectives. Doing this changes both design and the conversation: "fast" is undeveloped opinion, "p99 under 250ms" is a contract you can evaluate a design against. This is the difference between a requirement and a wish.

There is a tension you need to feel. Non-functional requirements are constraints, and designers hate constraints. But a constraint you know about is a gift compared to a constraint you discover under load. The latency budget, the scaling number, the consistency rule, every one tells you where to cut corners that you would otherwise cut in the wrong place. LLD in particular, concurrency, cache hierarchies, and state ownership choices, are almost entirely driven by how fast and how concurrent the non-functional requirement demands.

It also helps to sort the families into hard and soft, because the two behave differently in design. Hard non-functional requirements are binary and non-negotiable: two users must never reserve the same seat, a payment must never be duplicated. These become invariants, and the design has to buy them in the data model and the locking before anything else. Soft non-functional requirements are degrees: how fast, how available, how cheap. These are where the actual design negotiation happens, because you can trade one soft requirement against another. Most design arguments, the cache versus the database, the sync call versus the async event, are arguments between two soft requirements where one won. Naming a requirement hard or soft tells you whether the argument can even happen.

Not all five families apply everywhere, and you should not pad every doc with all of them. For a batch job that runs once a night, latency is a non-goal. For a stock trade, a duplicated execution is a bug and a seconds-late one is a different bug. Pick the three or four families that actually decide your design and nail those with numbers.

## Real Production Usage

Databases and caches are the clearest encoding of a non-functional trade-off. A relational database with transactions satisfies the "two users cannot reserve the same seat" constraint, strong consistency, at the cost of faster read scaling. A cache like a read-heavy distributed cache satisfies the "reads are fast" requirement but risks serving stale data, a price you pay in exchange for eventual consistency. The whole design of a caching layer, invalidate on write, a write-through, a TTL, is the product of which non-functional requirement you favored. Open any system design conversation and this is exactly what you will spend it on.

Kafka is the same thing wearing a costume. The reason teams pick an event broker is usually a non-functional requirement: replay ability, decoupling producers from consumers, order across a topic, the ability to add consumers without redeploying producers. These are not features, they are qualities. Ask "what functionality does Kafka provide" and you will misread what it is for. Read "how well does it let two systems proceed independently" and the picture clears.

In the Java world, the choice between `synchronized`, concurrent collections, and `volatile` with atomics is a non-functional decision about concurrency. All three make the code behave "correctly" under locking, but which one you choose depends on the throughput and latency and the safety-at-the-cost-of-speed you require. `ConcurrentHashMap` versus `Collections.synchronizedMap` is not a functional decision; the method names are the same. The difference is entirely about the performance and blocking of a shared structure under load.

## Common Mistakes

The biggest mistake is leaving non-functional requirements unnumbered. The doc says "must scale," the team builds for a guess, and the guess is wrong either way, too much cost or too little performance. As soon as someone writes a number, the requirement becomes a conversation you can have, and before that, it is a wish.

The reverse mistake is tuning every requirement into a number and pretending all of them matter. Not every family applies to every system. A daily batch job that writes a latency budget is waste. Rank the non-functional requirements your system actually collides with, and let the rest be not-a-goal stated in one line, which clears the design table.

The cut trap is the one that gets systems killed: you finish the functional work, the feature works, and you ship without re-checking the non-functional side under real load. The functional green means nothing if the non-functional corner was the whole requirement. "It works" is the acceptance test only when you never wrote down how well it had to work.

## Interview Perspective

Interviewers on an LLD or design problem will watch how you handle requirements before you write a class. The weak candidate takes the functional list and starts naming classes immediately. The strong candidate first asks which non-functional constraints bind, latency or throughput or consistency, and lets that steer the whole design.

A strong answer on the parking lot often is: "Before I place the spot, I need to know the concurrency. If a thousand cars check in during rush hour, the availability count is a concurrent write, and if two cars could race for the last spot I need atomic reservation, not a plain read." That is non-functional thinking made design. The weak version counts lanes and ignores the race.

Expected follow-ups: "which is more important for this feature, consistency or availability, and how would you satisfy the other?" and "turn that requirement into a number one would design against." Give exact numbers: a median and a p99, not a single average.

## Knowledge Check

1. The same feature, "let a user pay for an order," can be designed as a monolith with one database or as services with an event log. Identify the non-functional requirements that would force one over the other, and what numbers you would want before choosing.

2. A product owns "our system is reliable." Convert it into two specific non-functional requirements with numbers, and explain how each would change the design of an error-handling layer.

3. A team ships a feature whose functional tests all pass but that produces duplicate orders under a flash of traffic. Is the team's understanding of requirements incomplete, or their design wrong? Connect the answer to the functional/non-functional pair.

## Key Takeaways

- Functional requirements say what the system does; non-functional say how well, and the second is what decides your architecture.
- Unquantified non-functional requirements are wishes; numbers turn them into contracts.
- Your picks of database, cache, and concurrency mechanism are non-functional requirements made concrete.
- Not every non-functional goal binds every system; listing them all when only a few decide the design is waste.

## What's Next

You now know which requirements bind your design, but knowing what to design and producing it are different muscles. The next article, The Software Design Process and Design Thinking, walks the actual journey from a raw idea and its requirements to a working shape, and shows how the design thinking loop iterates over both functional and non-functional constraints without pretending the first pass is the answer.

---

This article explains the distinction between functional requirements, what a system does, and non-functional requirements, the standard it meets while doing it, and why the latter almost always decides the architecture. Its strongest claim is that a requirement without a number is a wish, and that "it works" is a test failure when the real constraint was never written down.