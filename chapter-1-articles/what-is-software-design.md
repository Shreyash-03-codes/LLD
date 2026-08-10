# What is Software Design?

## Learning Objectives

- Define software design in a way that separates it from coding and from requirements gathering.
- Explain why decisions made at design time are orders of magnitude more expensive to reverse than decisions made at edit time.
- Recognize a completed design even when no diagram or document is sitting in front of you.

## Introduction

Software design is the act of deciding how a system is built before, and while, it is built. You are making decisions about structure, behavior, and interaction, and those decisions constrain what the code can and cannot become. The code is the last thing that happens. Everything that ends up in the code typed out by a developer is a consequence of a design decision made earlier, whether that decision was deliberate or accidental.

The uncomfortable truth is that a codebase always has a design. Ignoring design does not free you from having one; it just means the wrong person made the decisions. Someone chose the framework, someone chose how classes talk to each other, someone chose whether state lives here or there. If nobody chose on purpose, the choices got made by whoever happened to be editing the file last. Your job when you design is to take that responsibility away from the invisible committee that has been running the project by accident.

## Problem Statement

There is a recurring, painfully specific failure: a team starts a modest feature, three developers plus a couple of dependencies, and by the second sprint the bug arrives. The team fixes it, and another one appears in the same place. It takes the fix a long time, and nobody connects the bugs to each other, even though they are all in a ten-line method. The team guesses the number is stale and cannot figure out where it was last assigned. There is no single object responsible for that number, no one rule for how it changes, and no boundary at all.

The code is small enough that everyone can read all of it. That is precisely the trap. Small systems fail at the design level because nobody is forced to engage with design. The ordering of operations, the ownership of data, the seam between two responsibilities, all of it stays buried, and everything still compiles. Nothing stops you until you are in production under load, and by then the cost is not a refactor, it is an outage.

## Core Concept

At its narrowest, software design is about two questions. What are the pieces, and how do they communicate. That sounds thin, but it is nearly the whole game. Everything else, naming, formatting, comments, is polishing a structure that is already right or already wrong.

What are the pieces. The moment you write a class, a module, a package, a microservice, you have committed to a boundary. That boundary says these responsibilities live together and these responsibilities live apart. A design is only as good as its boundaries. When data becomes inconsistent, trace it back to a boundary that was placed in the wrong spot. When a change ships in one place and breaks in a far away place, that is a boundary that lets a ripple pass through where it should have stopped.

How do they communicate. Interfaces, method signatures, messages, events, shared mutable state. The more loosely two pieces communicate, the more freedom each has to change. The more they reach into each other's internals, the more a change in one becomes a change in the other. This is the practical meaning of coupling, and it is the most useful single lens in all of software design.

Design also has a time dimension that people overlook. A design is not a snapshot; it is a trajectory. You make a decision that fits today, and you are also betting that it remains true in six months. Most design work is actually the handling of how much. You cannot predict the future, so you bias toward the structure that absorbs the likely changes and isolates the ones you cannot predict. That is why the same feature can be designed ten different ways that are all correct for today, and one is dramatically better in eighteen months.

Let a few categories organize your thinking. Structural design is the shape of the codebase, the modules, the layers, the public API. Behavioral design is how pieces coordinate, ordering, retries, timeouts, consistency rules. These two are the bulk of what any engineer, including you, does in a day.

Structural | Functional
--- | ---
Class and module boundaries | Request and response flow
Dependency direction | Retry and timeout behavior
Public API surface | State transitions and invariants
How data is organized and owned | Error handling and recovery

Now the hard part. Design is not the artifact, it is the decision. A UML diagram is not a design. It is a drawing of one. If the drawing and the code drift, the code is the real design and the drawing is a lie. Most companies have a drawer full of class diagrams that no longer resemble the system, and the system is still running, so something is functioning as the design. It is the code. That is a useful thing to keep in your head: the actual design is whatever the code enforces at runtime, not whatever you intended.

This is why reviewing and refactoring are design activities. A refactor is not cosmetic. Moving a method from one class to another redraws a boundary, which is a design change that happens to be expressed in code. When a senior engineer looks at your pull request and squashes a god object, or asks why a class depends on a repository directly instead of through an interface, they are reviewing your design in the disguise of reviewing your code.

None of this requires a diagram. It requires judgment. You can design well sitting at a keyboard, and you can design badly with the finest architecture slide show known to humans. The artifact is optional. The decision is not.

## Real Production Usage

The frameworks you use every day are design opinions enforced at scale. Spring's dependency injection is a design decision that says classes should not construct their own collaborators, because constructing them couples you to their shape and their readiness. Hibernate is a design decision about where the object relational boundary sits: you work with plain objects and it manages the mapping, and the cost is a boundary you must respect or it reimburses you in performance and lazy-loading surprises. Kafka is a design decision about how pieces communicate: producers and consumers share no connection and talk only through a log, which loosens coupling hard and trades consistency for it.

Read them that way. Every framework you grow up on is a settled argument about structure and communication. When you learn Spring, you are not just learning annotations; you are learning that a container should own your object lifecycle. That is the largest design decision possible, and it was decided for you before you wrote a single class. Knowing which parts of a framework are design opinions and which are interface details is the difference between using a tool and understanding it.

## Common Mistakes

Most engineers get this wrong by assuming design is something you do only at the start, with a blank screen and a whiteboard, and then move on. They draw boxes for two hours, then code freely. The files that survive years are rarely designed that way; they are designed in a long series of small, correct turns. Treating design as a single grand gesture instead of a continuous activity is how designs rot in place while the code evolves past them.

Another common failure is designing in response to the code you already have. You write something, then rationalize its shape as a deliberate design. That is a description of the design, not the design itself. It means you tend to be defensive about a structure that was mostly an accident. It reverses the order. The shape should lead the code, and when it does you notice that the code is failing you.

The worst habit is designing for today's happiness. You choose the structure that makes this one feature the simplest, ignoring the limited changes that would break it. Every decision is a bet, and betting everything on short-term convenience produces a system that collapses the day one real requirement touches it.

## Interview Perspective

In interview settings, ask yourself what you actually want to know when you look at candidates on this topic. You want to know whether the person distinguishes a decision from a diagram, and whether they can commit to a structure and then defend each trade-off. A strong answer names a concrete constraint. "I put the state machine behind an interface because the source of the truth it reads from may change," beats "I separated concerns so it's clean."

The weak candidate describes design as how the code is organized and cannot give a reason two modules are split. The same candidate redesigns the answer whenever the interviewer probes with a new requirement, which quietly admits they had no spine to their earlier answer. The candidate that holds his or her structural decision and explains at what price holds the stronger position.

Follow-up questions you should expect and be ready for: what happens to this design if you have ten times the data? And where in this system is the one change that worries you the most, and why did you not design around it? Both are really the same request: show me that you know what costs your design will eventually surface.

## Knowledge Check

1. You join a project where the code works but the class diagram is two years old and drifted badly. Which document is the actual design, the diagram or the code, and should the diagram be updated to match the code, or the code refactored to match the diagram? Defend the priority.
2. A developer suggests changing the data flow so one module holds the source of truth for a value, when a higher module currently reads or writes it from three places. Is this a design decision or an implementation decision, and how would you argue about the trade-off?
3. A team says they "just started coding" for a feature, using no design step and no diagram. Is that feature designed, or undesigned? Justify your answer using the idea that a codebase always has a design.

## Key Takeaways

- Design is the set of structural and behavioral decisions; the diagram is optional and often a lie.
- The two questions that cover the most ground are "what are the pieces" and "how do they communicate."
- A codebase always has a design; if you do not set it on purpose, someone sets it on accident.
- The real test of a design is not how it looks today but how cheap the anticipated changes turn out to be.

## What's Next

From here the natural question is scope. This article argued that design lives in decisions, but those decisions do not all live at the same level. That is the guard you cross when you move from designing the modules and their roles in a two-hour brainstorm to designing the behavior of a single class on a whiteboard. The next article, High-Level vs Low-Level Design, draws that exact boundary and shows you which failure each level is responsible for catching, because addressing the wrong level is one of the most expensive design mistakes you can make.

---

This article explains that software design is the set of decisions about structure and communication, not the diagrams that describe them, and that a codebase always carries a design whether anyone chose one or not. Its strongest claim is that the real design is whatever the code enforces at runtime, making the drawing optional and the decision mandatory.