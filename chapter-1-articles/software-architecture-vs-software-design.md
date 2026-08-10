# Software Architecture vs Software Design

## Learning Objectives

- Separate architecture from design by reversibility and scope, not by which diagram someone drew.
- Decide whether a new decision is architectural (costly to reverse) or a design decision (cheap to reverse).
- Explain why architecture is an emergent property of decisions, not a top layer you bolt on.

## Introduction

Software architecture and software design get used as synonyms, and that blur costs more than the naming pedantry suggests. When you conflate them you assign the wrong person, the wrong process, and the wrong cost to a decision.

Architecture is the set of decisions that are expensive to change later. It is the shape of the whole system at a level where redoing means reworking many parts: the split between services, the choice of database versus event log, the decision that a domain rule lives in one service and not another. Design is everything you can still change cheaply: the classes inside a service, the methods, the way one module calls another. Same activity family, very different stakes. The first change is a project; the second change is a pull request.

## Problem Statement

Picture the decision that gets made at the wrong altitude. A team building a payments flow chooses "we will use a document database for transactions" without deciding whether the transactions are the system of record or a projection of an event log. That one choice quietly commits them to an architecture: every consumer of the data has to read from that document store, and changing to an event-driven model later means rewriting half the system. Nobody called it an architecture decision. They treated it as a storage choice and moved on.

The opposite happens too. A team puts "we use microservices" at the top of every doc, calls it the architecture, and then never decides anything else. Service boundaries exist, but there is no design inside the services. That is architecture theater. Naming the pattern is not making the decision. The architecture is the specific shape your system takes, the boundaries, the data flow, not the fashion label you attach to it.

Either way, the same thing goes wrong: a decision with architectural stakes was made with the attention span of a design decision, or a design conversation got promoted to architecture and stuck there.

## Core Concept

Two tests separate architecture from design. Reversibility and scope.

Reversibility is the stronger test. A decision is architectural if reversing it later is expensive, in money or in rework across many parts. Choosing the programming language is architectural. Choosing a logging library is not, you can swap it in an afternoon. Choosing that orders and inventory live in the same database is architectural, because every transaction between them now has a fast local path that you will rebuild if you split them. Reversibility does not care how "big" the decision feels. A storage engine choice can be architectural; a module naming scheme is not.

Scope is the second test. Architecture concerns the whole system or its largest pieces and the relationships between them. Design concerns the inside of one piece. This overlaps the high-level/low-level line you already know, which is why the words get tangled. The clean way to think: all architecture decisions are high-level, but not all high-level decisions are architecture. Choosing the message format between two services is high level but cheap to change. Choosing the event broker itself is high level and expensive to change. One is design, the other is architecture.

Architecture | Design
--- | ---
Whole system or between its largest pieces | Inside one piece
Expensive and risky to reverse | Cheap and safe to reverse
Sets constraints everything else must respect | Works within those constraints
Decided by a few people, infrequently | Decided by everyone, constantly

There is a third property that most writing on this topic skips: architecture is emergent. You rarely sit down one morning and "do architecture." You make a string of decisions, a cache here, a contract there, a rule about who owns a piece of state, and the architecture is what those decisions add up to. That is why a codebase can have an architecture nobody drew: a hundred local choices congealed into a shape, and that shape now constrains everything. You can only design around it, at the architectural level, when you notice it.

This changes what "architect" means. An architect is not the person who draws the top diagram. It is the person who recognizes which of today's decisions are going to be expensive to reverse and refuses to let those slide by quietly. That is the real job, and it has nothing to do with seniority. A junior who flags "if we put this rule in the report service, changing it later means touching three consumers" is doing architecture. The title is earned by catching the high-stakes decisions, not by owning the box drawing.

One more distinction worth naming, because it comes up constantly in discussions of the two words. A pattern is not an architecture. Layered, hexagonal, event-driven, those are templates. Your architecture is what you actually chose when the template met your system, including all the places you departed from it. Saying "we are hexagonal" is a claim about intent. The reality is on the whiteboard and in the code.

There is a practical place where the reversibility test bites hardest, and it deserves its own name: the fracture plane. An architectural split is only worth doing where the system has a natural seam, a place where two parts change for different reasons and at different rates. Order processing and inventory reporting change for different reasons, so the seam between them is a candidate for a boundary. Splitting a system along a seam that does not exist is the most expensive way to manufacture an architecture that buys you nothing, because the two halves still change together and now they change across a network. Most bad microservice architectures are bad exactly this way: the splits follow no fracture plane, so the coupling the design was supposed to remove was just moved to a wire. The reversibility test tells you where the fracture planes are, because a seam that survives many independent changes is the one a split would have been cheap to leave reversible, and a seam that couples tightly is the one a split cannot fix.

## Real Production Usage

Kafka again, because it is the cleanest. When a team decides to publish "every order event" to a topic and let consumers rebuild their own projections, that is an architecture decision. It reorganizes the whole company's view of data. When they decide that one consumer offsets to a service whose view is a single window of recent orders, that is design. Same product, different altitudes.

Spring gives the inverse example. The fact that your web layer, service layer, and repository layer exist is a design decision made once inside an application. It constrains day-to-day coding, but changing a layer's internal responsibilities is cheap. Meanwhile the decision to run three Spring applications instead of one, and how they talk, is architecture. The framework itself is an opinion about design that got encoded, which is why it feels like an architecture until you try to reverse the actual architectural choices on top of it.

Real codebases show the emergent property constantly. Some shops have a documented architecture and drift from it, some have none and keep a working one by accident. The famous examples of architecture pain, the monolith that needs a multi-month split, or the distributed monolith that pretends to be microservices, are all just reversibility bites: decisions made when they were cheap and reversed when they were not.

## Common Mistakes

The most common mistake is calling any high-level decision architecture and any low-level one design, then treating the boundary as fixed by rank or by document type. A decision is architectural because of what it would cost to reverse. Store that test and apply it per decision.

The second mistake is deciding architecture in the design phase's energy. Teams schedule a two-hour design session, and in the first ten minutes someone says "we'll just use Postgres for everything" and nobody pushes back on the reversibility of that choice. The rest of the session is design, method by method, while the most expensive decision of the afternoon rode in unexamined. Watch for the high-stakes decision that got smuggled into a low-stakes conversation.

The third mistake is confusing the pattern with the reality. Teams claim an architecture ("we are event-driven") and keep building request-response with a broker bolted on. The architecture is the real thing, not the label. You want to state decisions, not decorate them.

## Interview Perspective

In interviews, the two words are a trap because candidates can sound fluent while meaning nothing. The interviewer asks what the difference is, and the candidate says "architecture is the big picture, design is the small picture," which is true and useless. The candidate who separates them by reversibility, and can name a decision at each level and what reversing each would cost, is the one who gets the nod.

A strong answer on a design problem flags architecture decisions explicitly and does not let them slip. "Before I design classes, I need to settle whether this is one service or two. That is an architecture decision for us because the split affects deployment and data ownership, and it would be expensive to reverse later." A weak answer dives straight into class diagrams and treats the service boundary as decoration.

Expected follow-ups: "when you chose the database in this design, why was that architectural, and what would redoing it cost?" and "in the last system you worked on, which decision turned out to be architectural and you did not know it at the time?" Both want the same thing, evidence that you can smell a high-stakes decision mid-conversation.

## Knowledge Check

1. Which of these are architecture decisions, and which are design decisions: choosing Spring's web framework for a service, choosing to split order processing and inventory into two services, choosing that a service exposes its data via REST rather than direct database reads, choosing a field name in a response JSON. Defend each with the reversibility test.

2. A team documents "we are a microservices architecture" but every feature ships as one deployment with internal modules. Is the team's architecture microservices? Justify using the emergent property.

3. Two engineers debate whether moving the retry policy from the client to the server is architecture or design. Under what conditions is each answer correct, and what question would you ask to settle it?

## Key Takeaways

- Architecture is the set of expensive-to-reverse decisions; design is the cheap-to-reverse ones.
- Reversibility and scope are the two tests; scope alone is not enough.
- Architecture emerges from the sum of decisions, so it can exist without any diagram.
- Patterns are templates, not architectures; the actual shape you chose is the architecture.

## What's Next

You now know what the two words mean, but neither design nor architecture gets built in a vacuum. Both respond to something that comes before them: the requirements. The next article, Functional vs Non-Functional Requirements, explains why the functional list gets all the attention while the non-functional ones decide your architecture, and why "it works" is the wrong test for the requirements that actually shape the system.

---

This article explains that architecture and design are separated by reversibility and scope, not by diagram type, and that architecture emerges from the sum of decisions rather than from a top-level drawing. Its strongest claim is that an architect is whoever catches the high-stakes decision in a low-stakes conversation, not whoever owns the box diagram.