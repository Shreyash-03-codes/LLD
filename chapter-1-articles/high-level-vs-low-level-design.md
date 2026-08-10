# High-Level vs Low-Level Design: Drawing the Boundary

## Learning Objectives

- Separate system-level decisions from class-level decisions, and decide which level a given decision belongs to.
- Catch yourself jumping to low-level details before high-level choices are settled.
- Recognize when the boundary between the two moves because your context changed.

## Introduction

High-level design and low-level design are not two kinds of documentation. They are two scopes of decision. High-level design is about the pieces of a system and how they fit together: which services exist, which data flows between them, what technology each piece uses. Low-level design is about the inside of a single piece: the classes, the responsibilities, the methods, the state, the invariants that hold while it runs.

The practical difference is the question you are answering. HLD answers "what parts does this system have, and how do they talk?" LLD answers "what does this one part do internally, and how does it stay correct?" You can hold both answers in your head at once. Most of the skill is knowing, at any given moment, which question you are supposed to be answering, because answering the wrong one wastes everyone's time.

## Problem Statement

The design review where a senior engineer asks "what happens when the retry queue is full" and the junior, mid-whiteboard, starts naming classes and method signatures.

That exchange is a mismatch, not a question of who is smarter. One person is designing how the system absorbs load, which is a high-level decision about the retry queue, its position, its bounds. The other is designing the internals of a class that barely exists yet. The meeting bogs down because the two are talking past each other, and the decision actually needed, the high-level one, never gets made.

The sharper failure is the reverse and it is more common in big teams. The architecture is designed at a high level, a decision about service boundaries and event topics, and nobody ever carries it down to the classes. The service diagram is beautiful. Underneath, each service is a giant module with no internal structure, because the LLD step was skipped. High-level coherence hides a low-level mess. Both directions of the mix-up cost real money.

## Core Concept

Think of the system as a stack of decisions. At the top you decide the system exists and what shape it takes. Each step down you add detail about a smaller part. The HLD versus LLD boundary falls between deciding how the pieces are arranged and deciding how one piece behaves internally.

Decisions that live above the line:

- Which components or services exist, and which responsibilities they own.
- How components communicate, request-response, async, events.
- Where data lives and who reads and writes it.
- Technology choices: database, message broker, deployment shape.
- Failure, scaling, and resource boundaries.

Decisions that live below the line:

- The classes inside one component and their responsibilities.
- Method and interface signatures.
- The state held by each object and how it changes.
- Algorithms and data structures for a single operation.
- Concurrency and locking inside one component.

The line is a scope boundary, not an approval gate. It is not "write the HLD doc, get it approved, then write the LLD doc." It is "these questions are about the whole system, those questions are about one class within it." The moment the question becomes about a single class, you are doing the low-level design, even if no component-level question is resolved yet.

Here is the diagram of where the boundary sits.

Diagram: where the high-level and low-level boundary falls in the design stack.

```
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 860 560" font-family="Arial, Helvetica, sans-serif">
  <defs>
    <marker id="arrowhead" markerWidth="10" markerHeight="8" refX="9" refY="4" orient="auto">
      <polygon points="0 0, 10 4, 0 8" fill="#57606a"/>
    </marker>
  </defs>

  <text x="140" y="195" font-size="13" fill="#6e7781" text-anchor="end" font-style="italic">system level</text>
  <text x="140" y="345" font-size="13" fill="#6e7781" text-anchor="end" font-style="italic">class level</text>

  <rect x="250" y="40" width="400" height="70" rx="8" fill="#f0f0f0" stroke="#57606a" stroke-width="2"/>
  <text x="450" y="70" font-size="16" font-weight="bold" fill="#24292f" text-anchor="middle">REQUIREMENTS</text>
  <text x="450" y="95" font-size="13" fill="#444c56" text-anchor="middle">what the system must do</text>

  <line x1="450" y1="120" x2="450" y2="140" stroke="#57606a" stroke-width="2" marker-end="url(#arrowhead)"/>

  <rect x="250" y="150" width="400" height="100" rx="8" fill="#e8f0fb" stroke="#1f6feb" stroke-width="2"/>
  <text x="450" y="180" font-size="16" font-weight="bold" fill="#0a3069" text-anchor="middle">HIGH-LEVEL DESIGN</text>
  <text x="450" y="205" font-size="13" fill="#1f4484" text-anchor="middle">components, data flow, tech stack</text>
  <text x="450" y="225" font-size="13" fill="#1f4484" text-anchor="middle">failure and scaling</text>

  <line x1="170" y1="272" x2="370" y2="272" stroke="#d1242f" stroke-width="2" stroke-dasharray="6,6"/>
  <line x1="510" y1="272" x2="700" y2="272" stroke="#d1242f" stroke-width="2" stroke-dasharray="6,6"/>
  <rect x="380" y="259" width="140" height="26" rx="13" fill="#ffffff" stroke="#d1242f" stroke-width="2"/>
  <text x="450" y="276" font-size="13" font-weight="bold" fill="#d1242f" text-anchor="middle">THE BOUNDARY</text>

  <rect x="250" y="300" width="400" height="100" rx="8" fill="#e6f6ec" stroke="#1a7f37" stroke-width="2"/>
  <text x="450" y="330" font-size="16" font-weight="bold" fill="#033d16" text-anchor="middle">LOW-LEVEL DESIGN</text>
  <text x="450" y="355" font-size="13" fill="#166534" text-anchor="middle">classes, methods, interfaces, state</text>
  <text x="450" y="375" font-size="13" fill="#166534" text-anchor="middle">concurrency and invariants</text>

  <line x1="450" y1="410" x2="450" y2="440" stroke="#57606a" stroke-width="2" marker-end="url(#arrowhead)"/>

  <rect x="250" y="450" width="400" height="70" rx="8" fill="#f0f0f0" stroke="#57606a" stroke-width="2"/>
  <text x="450" y="480" font-size="16" font-weight="bold" fill="#24292f" text-anchor="middle">CODE</text>
  <text x="450" y="503" font-size="13" fill="#444c56" text-anchor="middle">the implementation</text>
</svg>
```

Read the diagram as a single question. "Which of my pieces does it affect?" If the answer is several pieces, it is high-level design. If the answer is one class or one method, it is low-level. That single test resolves most of the confusion faster than any definition.

There is a wrinkle that trips people up. The boundary is not fixed. It moves when your context moves. In one company a "service" is a two-hundred-line class and the decision to split it is a low-level decision. In another company a "service" is deployed separately and the decision to split it is a high-level one, because it changes deployment and operations. The same noun, different side of the line. When someone asks "is this HLD or LLD", the honest answer is "depending on what the piece means at your scale."

So treat high-level and low-level not as two checkpoints on a release calendar but as two resolutions of the same lens. You zoom out, decide the pieces, zoom in, decide each piece. Good design processes do many passes at each resolution, not one pass at each and done.

## Real Production Usage

The frameworks you use encode both levels. Spring's layered layout, controller, service, repository, is a high-level decision within an application about which responsibilities live in which tier. When you design a feature in Spring, you first decide which layer owns what, that is the high-level part, and then you decide the methods and fields per class, the low-level part. You do not need framework lore to see it; the layering is visible in any Spring shop's package structure.

Kafka is a clean example of the boundary moving. Deciding how many partitions a topic has, and how consumers are grouped, is a high-level decision. It determines system throughput and ordering, and it touches producers and consumers on both sides. Deciding how the consumer loop's internals iterate, handle a decode, and resume is a low-level decision inside one consumer class. Two questions living on opposite sides, and teams use both every day without naming them.

Design documents in real companies usually split the same way. The architecture artifact lists services and the REST or event contracts between them, high level. The detail design for one service lists its classes and the public API each exposes, low level. The boundary is the same one you draw on a whiteboard, just pinned to a document.

## Common Mistakes

The most common mistake is answering a low-level question during a high-level conversation. Someone asks where the retry policy lives and the response names a specific class and its fields. That response is not a flaw in knowledge, it is a flaw in framing. You anchor the whole team downward and the system questions never get decided. When you catch yourself naming methods, stop and ask what the pieces are.

The opposite mistake is designing boxes forever and never descending. The team nails the service diagram, signs off, and then each service is one giant router with no internal design. The high-level is coherent, the low-level is a wall of code. Both errors are the same error seen from two ends: treating the boundary as a wall instead of a step you are supposed to climb over.

The subtler one: freezing the boundary based on a technology rather than a scope. You label "we don't do HLD here, we're just a library" or "this is all HLD, it's a distributed system." The library still has classes that need LLD, and the distributed system still has per-service internals that need it. The line tracks the level of the decision, not the size of the company or the buzz.

## Interview Perspective

Interviewers are short on time, but they watch one thing: can the candidate answer the question that is actually being asked instead of re-scoping it downward. The problem is high-level, and they want to see whether you notice and answer at that level, or drop to a class diagram.

A strong answer makes the level explicit. "Before I pick a structure, I need to decide whether this is one service or several, and how they talk. That decision is high level. Once that's fixed, I'll tell you the classes inside." Weak answers talk about the HLD and LLD documents as though they are serialized paperwork, "first we make the HLD, then the LLD," and cannot point to a single decision at either level.

Common follow-ups: give one example of a high-level decision and one low-level decision in this parking lot or elevator you just designed, and "where would you reconsider the boundary if this system grew to scale?" Both probe whether the boundary is a rule you hold or a reflex you can move.

## Knowledge Check

1. In a parking lot system, sort these into high-level vs low-level: choosing to store spot availability in a centralized service, choosing to model the parking rates as a separate class, deciding the entrance gates and exit gates communicate over a message queue, and deciding the PaymentService uses a two-phase payment state. Justify each.

2. A team splits one large service into two based on a load problem. Is that a high-level or a low-level decision, and what specific fact about your system would change your answer?

3. Two engineers are arguing about where a retry policy should live. One escalates it to a design meeting about a whole cluster write-back-through queue. The other reserves it quietly inside one class. Are they disagreeing about the answer, or about the question? Explain.

## Key Takeaways

- High-level design is choosing the pieces and their communication; low-level design is choosing what each piece does internally.
- The single deciding question is "does this touch several pieces or one piece?"
- The boundary is relative; the same decision crosses the line depending on what a "service" means at your scale.
- The line is a lens resolution you pass through repeatedly, not a phase you complete once.

## What's Next

The boundary you just drew is real but messy, because high-level and low-level both live under a wider name. What most engineers call "architecture" and "design" overlap the boundary in a way that muddles the distinction further. The next article, Software Architecture vs Software Design, takes the line you now hold and separates those two terms from each other, and shows why confusing them sends decision and ownership to the wrong person.

---

This article explains that high-level and low-level design are not documents but two scopes of decision, split by whether a choice touches the whole system or a single component. Its strongest claim is that the boundary is not fixed and moves with your scale, so the same question can be high-level in one organization and low-level in another.