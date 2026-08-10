# Association, Aggregation, and Composition

## Learning Objectives

1. Tell the three has-a relationships apart using the only axis that matters: who owns the lifetime.
2. Read and draw the UML notations, the plain line, the hollow diamond, and the filled diamond.
3. Model ownership correctly in Java, where the language does not enforce it for you.

## Introduction

Association, aggregation, and composition are three ways of saying "one object holds another," and the diagrams make them look like three flavors of the same thing. They are not. They are three different answers to one question: who owns the other object's lifetime?

Association is the weakest form. Two objects know about each other and interact, and neither controls the other's existence. A `Person` walks into a `Cinema`, they are related for the duration of the interaction, and neither owns the other. Aggregation adds a container. A `Team` holds `Player` objects, the players belong to the team, and the players also exist on their own; a player was a player before joining and stays a player after leaving. Composition is the strong form. A `House` owns its `Room` objects, and a room does not exist outside the house. The relationship is the room's existence, not just its connection.

## Problem Statement

The code that fails here is the code that never asked who owns what. A class holds references to other objects, creates some of them, receives others, and nobody can say where each one came from or who is responsible for cleaning it up.

Consider a `Session` that creates a `Transaction` in its constructor, stores it in a list, and also passes it to an event dispatcher so other parts of the system can see it. Later the session is finished and closed. The transaction, now held by the dispatcher and by whoever listened for the event, keeps running against a closed session. Code that believed the session owned the transaction calls methods on a dead object, and the only protection is a runtime check that nobody added. The model said composition, "the transaction lives inside the session," and the code leaked the child reference, turning it into aggregation with a broken lifetime promise.

Or the reverse. A `Report` holds a `Printer` that the application created at startup and intends to reuse. The report treats the printer as its own, closes it when the report is done, and the next report crashes. The model said aggregation, "the printer outlives any report," and the code assumed composition. Either way, the failure is the same: the ownership relationship was never decided, so the code invented one and got it wrong.

## Core Concept

The three relationships form a ladder, and the rungs are the strength of the lifetime tie.

Association. A plain relationship between independent objects. The arrow or plain line means "knows about," and nothing more. Neither object creates the other, neither destroys the other, and the relationship can be temporary or permanent without changing who lives and dies. A `Person` and a `Cinema`, a `Customer` and an `Order`, a `User` and a `Session`. The objects could each exist without the other, and they do. Association answers "are they connected" with yes and declines to say anything else.

Aggregation. A has-a relationship where the whole holds the parts and the parts still exist on their own. A `Team` has players, and a player is a person who existed before the team and will exist after. The diamond, hollow, sits at the container end. Aggregation says "this object holds those objects, and the held objects are not its property." The container can come and go and the parts persist, because the parts have their own existence.

Composition. A has-a relationship where the part exists only inside the whole. A `House` has rooms, and when the house is gone, the rooms are gone with it. The diamond is filled and sits at the owner end. Composition says "this object owns those objects, their lifetimes are tied to mine, and they do not exist independently." The part has no life outside the whole, and the whole is responsible for the part from creation to destruction.

The table is the whole article if you remember one column.

Relationship | Diamond | Does the part outlive the whole? | Example
--- | --- | --- | ---
Association | none | Yes, they are unrelated lives | Person and Cinema
Aggregation | hollow | Yes, the part exists on its own | Team and Player
Composition | filled | No, the part dies with the whole | House and Room

Diagram: the three relationship notations, with the ownership diamond on the container end.

```
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 800 430" font-family="Arial, Helvetica, sans-serif">
  <text x="90" y="86" font-size="13" font-weight="bold" fill="#1f2328">Association</text>
  <rect x="90" y="100" width="160" height="60" rx="6" fill="#f6f8fa" stroke="#57606a" stroke-width="2"/>
  <text x="170" y="138" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">House</text>
  <rect x="520" y="100" width="160" height="60" rx="6" fill="#f6f8fa" stroke="#57606a" stroke-width="2"/>
  <text x="600" y="138" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">Person</text>
  <line x1="250" y1="130" x2="520" y2="130" stroke="#57606a" stroke-width="2"/>
  <text x="385" y="115" font-size="12" fill="#57606a" text-anchor="middle">Neither object owns the other</text>

  <text x="90" y="206" font-size="13" font-weight="bold" fill="#1f2328">Aggregation</text>
  <rect x="90" y="220" width="160" height="60" rx="6" fill="#f6f8fa" stroke="#57606a" stroke-width="2"/>
  <text x="170" y="258" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">Team</text>
  <rect x="520" y="220" width="160" height="60" rx="6" fill="#f6f8fa" stroke="#57606a" stroke-width="2"/>
  <text x="600" y="258" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">Player</text>
  <polygon points="256,250 286,250 271,238 271,262" fill="#ffffff" stroke="#57606a" stroke-width="2"/>
  <line x1="286" y1="250" x2="520" y2="250" stroke="#57606a" stroke-width="2"/>
  <text x="385" y="235" font-size="12" fill="#57606a" text-anchor="middle">Part can outlive the whole</text>

  <text x="90" y="326" font-size="13" font-weight="bold" fill="#1f2328">Composition</text>
  <rect x="90" y="340" width="160" height="60" rx="6" fill="#f6f8fa" stroke="#57606a" stroke-width="2"/>
  <text x="170" y="378" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">House</text>
  <rect x="520" y="340" width="160" height="60" rx="6" fill="#f6f8fa" stroke="#57606a" stroke-width="2"/>
  <text x="600" y="378" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">Room</text>
  <polygon points="256,370 286,370 271,358 271,382" fill="#1f2328" stroke="#1f2328" stroke-width="2"/>
  <line x1="286" y1="370" x2="520" y2="370" stroke="#57606a" stroke-width="2"/>
  <text x="385" y="355" font-size="12" fill="#57606a" text-anchor="middle">Part dies with the whole</text>
</svg>
```

Now the uncomfortable part for Java developers. Java has a garbage collector and no destructors, so nothing in the language enforces any of this. A composed object can be stored in a static field and live forever. An aggregated object can be created only inside the owner and never escape, which is composition in everything but the label. Ownership in Java is a discipline expressed through code structure, and the mechanics of that discipline matter.

Composition in Java means the child is created by the parent, kept private, and never handed out. The parent's constructor builds the parts, the parent holds them in private fields, and no method returns the child or passes it to another object. Because the child cannot escape, it cannot outlive the parent except by leaking through the garbage collector, and in practice it dies when the parent dies. If a part must be shared, it was never composition, it was aggregation.

Aggregation in Java means the parent receives the part from outside, as a constructor parameter, a setter, or a factory result, and stores a reference. The part's lifetime is managed by whatever created it, and the parent is a tenant, not a landlord. This is where dependency injection lands. When Spring hands a service its repository, that is aggregation with the container as the true owner, and the service must not close or dispose what it does not own.

UML gives you the notation so diagrams say what they mean. A plain line, no diamond. A hollow diamond on the container's end for aggregation. A filled diamond for composition. When you see the diamond, read it as "this end owns the lifetime," and when you see the plain line, read "these objects know each other and that is all."

## Real Production Usage

Ownership in real Java shows up where resources end, and the cleanest production example is the `Transaction` in JDBC. A transaction is created by a `Connection`, and it is meaningful only while the connection is active. Commit, rollback, it dies with the connection's work. That is composition. Nobody passes a transaction to another subsystem to reuse later, because it would be meaningless, and the model reflects reality.

The opposite end is the shared service. An application creates one `ExecutorService` at startup and hands it to every job that needs threads. Jobs come and go, the executor lives for the whole process, and it must be shut down exactly once, by whoever created it, at application shutdown. That is aggregation, the executor's lifetime is independent of any single job. The bug in the problem statement, a job shutting down the executor it does not own, is exactly the ownership error.

The web layer gives a sharper one. An `HttpSession` owns its attributes, they exist only while the session exists, and clearing or expiring the session destroys them. Any attribute is invalid after the session ends, and code that cached a session attribute outside the session is holding a reference to a dead composition. Frameworks reinforce this with `@SessionScoped` beans that the container destroys when the session dies. Composition is not only modeled, it is enforced.

The rule that runs through all three: whoever creates a thing owns its lifetime, and whoever owns the lifetime is the only one who closes it. State it once, in every review, and the relationship bugs in the problem statement stop happening.

## Common Mistakes

The first mistake is treating aggregation as composition, usually by calling the relationship composition because it feels stronger. A `Team` and its `Player` objects is aggregation, because players have independent lives, and modeling it as composition forces you to invent a rule that a player cannot exist outside a team, which is false. The distinguishing question is always "does the part exist without the whole," and when the answer is yes, the filled diamond is a lie.

The second mistake is modeling composition and then leaking the child. The parent creates the part, stores it privately, and one method returns it "so the caller can see it," which turns the composition into aggregation, which the caller does not know. The next person reads the class, sees a filled diamond in the head, and cleans up under the wrong assumption. Composition and a public child reference cannot coexist.

The third mistake is over-modeling. A class diagram with an aggregation arrow between every pair of classes is not a design, it is noise. Most fields are simple references, association, and drawing ownership diamonds on them implies a lifetime contract that was never intended. Apply the notation when the lifetime question has a real answer, and leave the plain line everywhere else.

The fourth mistake is forgetting which end gets the diamond. The diamond goes on the container, the whole, the owner. `House` has a filled diamond on the house end, `Room` gets none, and drawing it backwards communicates the exact opposite of the intended ownership.

## Interview Perspective

The question is usually phrased as "difference between association, aggregation, and composition," and the candidate who lists definitions gets partial credit. The candidate who answers with the lifetime test gets the job. "Aggregation means the part can exist without the whole, a team and its players. Composition means the part only exists inside the whole, a house and its rooms. The diamond is filled for composition, hollow for aggregation, and absent for association."

The sharper interviewers push on Java. "How do you express composition in Java, given there are no destructors?" The answer is structure, not syntax: the part is created in the parent's constructor, stored privately, and never exposed, so it cannot outlive the parent. "What happens when you hand the child to another object?" Then it is not composition anymore, it is aggregation, because the child's lifetime is now shared.

Another common probe: "Where does Spring sit?" The answer is aggregation with the container as owner. The service does not create the repository and does not close it; the container manages both lives, and the service that closes what it was given is committing the ownership error from this article.

## Knowledge Check

1. A `Department` holds a list of `Employee` objects, and employees join and leave the company while the department persists. Name the relationship and the correct UML notation.

2. A `Connection` creates a `Transaction` in its constructor, keeps it in a private field, and passes it to a `commit()` method that closes the session's work. Name the relationship and explain what Java enforces about it.

3. A `Report` stores the application-wide `Printer` it was given and closes it in `close()`. Identify the ownership error and the correct relationship, then describe what changes.

## Key Takeaways

- Association, aggregation, and composition are answers to one question: who owns the lifetime.
- Aggregation parts outlive the whole; composition parts die with the whole; the diamond is filled for composition and hollow for aggregation.
- Java enforces nothing, so composition is the discipline of private fields, parent-created children, and no escaping references.
- Whoever creates a resource owns it, and only the owner closes it.

## What's Next

The has-a relationships are about who holds and owns, and the remaining relationship type is thinner: a class can depend on another without holding it at all. The next article, Dependency Relationships, covers the direction of the arrow, why a dependency is a relationship in its own right, and how inverting it is the first move of clean design.

---

This article explains the three has-a relationships, association, aggregation, and composition, and reduces them to one question: who owns the other object's lifetime. Its strongest claim is that Java enforces none of it, so composition is a discipline of private fields and parent-created children, and that whoever creates a resource is the only one allowed to close it.
