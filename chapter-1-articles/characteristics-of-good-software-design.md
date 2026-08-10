# Characteristics of Good Software Design

## Learning Objectives

- List the properties that separate a design that survives from one that stalls, and explain why each one matters.
- Tell a genuine quality like low coupling from a look-alike like "everything is a constant."
- Evaluate a design by its cost of change instead of by how neat it looks.

## Introduction

Nobody can point at a good design the way they can point at a working feature. But the absence of it is unmistakable: every change touches ten files, tests are a chore, and new engineers take weeks to find where a behavior actually lives. Whatever you call it, there is a real difference between the code you can still safely change and the code you have started to fear.

The word that names the difference is most often "quality," which is too fuzzy to be useful. Let me name the specific properties instead, in order of how much they cost you when they are missing: low coupling, high cohesion, simplicity, testability, and readability. These are not decorative virtues. Each one maps directly to how expensive your next change is going to be.

## Problem Statement

Two teams write the same system. Team A's version ships this quarter and nothing breaks. Team B's version shipped a year ago, and every feature since has required a two-week refactor that touches classes nobody expected. The product owner cannot see the difference, the feature lists are identical, and the customers cannot either. The difference is entirely internal, and it is entirely the difference between a design that could take the next change and a design that dreads it.

That is the real cost of a bad design, and it is invisible to anyone who is not the team maintaining it. Nobody gets fired for it in a quarter. The design cost shows up later, as slower delivery, as staff who leave, as the "legacy" label, and as the rewrite that everyone proposes and nobody can justify. The common thread is a design that was allowed to look reasonable and never had to survive the next change.

## Core Concept

The characteristics of a good design are best seen as failure pressures. Each is the antidote to a specific way a design rots, not a decoration.

Low coupling is a measure of how independent one module is from changes in another. Two modules are tightly coupled when a change to how one stores its data forces the other to change too. Coupling is a judgment, not a count, and the test is causal: if I change this class, how many unrelated classes have to change because of it? Low coupling means the answer is few.

Cohesion is the flip side, and it describes how much the things inside one module belong together. A class is highly cohesive when its fields and methods all conspire around one job, and every method is actually used by the class's purpose. A god object that holds config, does IO, and computes reports is low cohesion; each feature drags the whole object into it. Low coupling and high cohesion are usually taught together because a module with one clear job tends to couple little with its neighbors. They are the same idea enforced from two sides.

The next one is underrated because it is invisible in a demo: simplicity. Not "simplicity" as in small, but as in few distinct ideas to hold in your head at once. A design is simple when a reasonable person can trace what the code does without a notebook. Complexity is not the same as "lots of lines"; two hundred straightforward lines beat eighty lines of clever indirection. The test for simplicity is how much it hurts to add the next feature.

Testability is how easy it is to prove the design is still correct. A design is testable when its behavior can be exercised without standing up the whole world, which means dependencies can be faked and the interesting logic can run in isolation. This is not a testing article, but it is a design characteristic, because a structure you cannot test is a structure you cannot trust to change.

Readability is last, not because it is unimportant, but because it is downstream of the others. A readable design is one a stranger can follow without a guide. The real design decision is not the absence of cleverness, it is the discipline that the moment a class does more than one job, or a method needs a paragraph to explain itself, the design has failed and the explanation is treating the symptom.

Characteristic | It defeats | How you notice it
--- | --- | ---
Low coupling | A change rippling to unrelated code | Editing one class does not touch five others
High cohesion | The god object and the grab bag module | Each class has one job and uses all its fields
Simplicity | The design no one can trace | A new feature does not multiply the surface you must understand
Testability | The change you fear to make | The risky behavior runs in isolation with a fake boundary
Readability | The class that needs a novel to explain | A reader finds behavior where a reader looks for it

All of this collapses into one claim, and it is the strongest thing in this article. Most writing on design leads with "maintainability," but that word is a label, not a lever. It tells you the outcome and nothing about how to get it. The actual lever is coupling. A design that is highly cohesive and low coupling is a design that is maintainable, extensible, testable, and often readable, at once, because all of those follow from the same affordance: you can change one place without dragging the rest of the system with it.

That is the property to keep. When you are reviewing a design and you can't decide whether it is good, ask not "is it clean" but "when this feature changes, how many files do I have to touch, and do any of them belong to someone else's concern?" A good design is precisely the one where the answer is "one file, its collaborators." Everything else, the pretty abstraction, the clever extension point, is decoration that earns its keep only when it makes that answer cheaper.

## Real Production Usage

The standard library is one of the best running examples of high cohesion and low coupling. Java's `Collections` and the `java.util.concurrent` package separate a data structure's interface from its implementation and from its threading semantics. When you use a `ConcurrentHashMap`, you touch the container, not the internals of why the buckets are thread-safe. The design holds that boundary, and you are able to change the structure you use without changing the code around it, because coupling is low.

Spring's dependency injection is a coupling lever turned into a framework. When Spring hands a bean to a class instead of letting the class construct it, it removes the construction coupling. Your class no longer needs to know about the concrete collaborator's setup, only about its interface. That is coupling reduction made a framework, and it buys testability for free, because a fake collaborator slides in where the real one would.

A negative example, one you have seen in a real codebase, is the concurrency answer that couples every method call: a single shared lock that serializes all access to a structure because the design never decided who owns what. It is "correct" in the narrow sense and it is a design that collapses under load. The quality that is missing is not memory, it is clear state ownership, which is coupling applied to state.

## Common Mistakes

The most common mistake is rating a design by its neatness instead of its cost of change. A design can look beautiful, layered, and abstract and still be the worst machine to change, because the abstraction sits at the wrong place. When you review a design, calling it beautiful is fine, but you must call it good because the next change was cheap, not because it looks tidy.

The second mistake is mistaking reduced repetition for reduced coupling. Extracting a shared method is not decoupling anything if the thing you extracted has to be dragged along by every caller. Shared code is real coupling, which can be worth it, but you need to know you paid for it. DRY is not a substitute for a measured boundary.

The third mistake is making testability the reason for the design rather than the symptom. Hacking a class to expose internals just so the test can reach it is not a testable design, it is a leaky one. Real testability is the ease of running the behavior in isolation, and the honest route is to lower coupling and raise cohesion, not to widen the class's API for the test.

## Interview Perspective

Interviewers who ask "what makes a good design" are not looking for a slide list of principles. They want to see whether the candidate can take a design and press it with a single criterion. The candidate who says "low coupling and high cohesion" and then cannot name a concrete change that would be easy or painful has memorized the words.

The strong answer picks a change in the problem and runs it through the design. "When I add a spot type with a different pricing rule, does that touch the billing class or does it stay inside the lot class?" That is coupling applied to the candidate's own design, and it beats a recitation of principles. The weak answer defends the design with "it's clean" or "it follows best practices," neither of which is a test.

Expected follow-ups: "what is the change you most want this design to survive, and what does it cost today?" and "where is the weakest point in this design, and why did you put it there?" Both want the candidate to name the part of the design that carries the coupling, which is a sign they understand it as a risk.

## Knowledge Check

1. Two classes both depend on a shared configuration class. Is that coupling bad by default? Describe the condition under which sharing that config class is a good trade and the condition under which it is a trap that spreads half the codebase.

2. A design keeps all its logic in one god class but the code compiles and passes tests. Which characteristic is failing, how would you detect it, and why does it not show up in any green test?

3. A feature is delivered quickly but its next two changes are slow. Team A calls this bad luck. Team B says the design is the problem. Which diagnosis points at a cause you can act on, and what is that cause?

## Key Takeaways

- The core question of a good design is the cost of change, not how it looks.
- Low coupling and high cohesion are the same idea from two sides: change stays inside its own unit.
- Testability and readability mostly follow from good boundaries; you do not need a separate effort, and you must not widen the API for the test.
- Beauty is decoration; it counts only when it lowers the cost of change.

## What's Next

You now know what good looks like, which makes the opposite easier to name. The next article, Common Software Design Mistakes, catalogs the specific, repeated ways designs go bad, from the god object to premature abstraction, and you will see that nearly all of them are a single root cause: a coupling created at the wrong altitude. Knowing the failure by name makes it catchable at review time.

---

This article defines the characteristics of good software design through the lens of how cheap change is, led by coupling and cohesion, not how neat the code looks. Its strongest claim is that maintainability is a label for one real lever, and the honest test of a design is "how many files do I touch when behavior changes."