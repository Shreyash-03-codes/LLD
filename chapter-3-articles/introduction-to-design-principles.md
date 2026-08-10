# Introduction to Design Principles

## Learning Objectives

1. Treat design principles as heuristics with a price, not as laws to obey.
2. Weigh two principles against each other when they pull in opposite directions.
3. Recognize when a principle is being quoted in place of an argument.

## Introduction

A design principle is a rule of thumb distilled from other people's failures. Someone shipped a god class and watched every feature branch touch it, so the Single Responsibility Principle got written down. Someone added a payment provider and had to edit twelve files, so the Open-Closed Principle got written down. The principles in this chapter are not a checklist you apply and collect. They are the scar tissue of the industry, compressed into one-line rules.

The reason they survive is that they all aim at the same target: making the cost of changing the system predictable and low. Coupling, duplication, hidden decisions, these are what make a change expensive. The principles are different attacks on the same enemy.

## Problem Statement

A design review is about to end. The senior engineer on the other side of the table says, "This violates the Single Responsibility Principle," and the room goes quiet. Nobody asks what the change actually costs. The principle is used as a veto, the objection wins by citation, and a class that was doing one coherent job gets split into three that all change together.

That is the failure mode this chapter exists to warn about. A principle applied as a law is a way to stop thinking. The same reviewer would be equally happy citing "you can't have a 500-line class," which is not a principle at all, it is a superstition with a line count. The system does not get better because a rule was honored. It gets better when the change cost goes down, and the principles are the instruments for measuring that, not the goal.

## Core Concept

Start with what a principle is not. It is not a fact. The Open-Closed Principle is not true the way "every object in Java inherits from Object" is true. It is a bet. "If you structure the code this way, the next change will be cheaper." The bet pays off under one condition: the change you were preparing for is the change that actually arrives. When the change that arrives is different, the structure you built to be "open" is just indirection you carry for no reason.

So the real skill is not memorizing the principles. It is holding the cost model in your head. Every principle buys you something and charges you something, and the question you ask before applying any of them is: what change am I paying to make cheaper?

Take DRY and YAGNI, which will get their own article later in this chapter. DRY says do not duplicate knowledge, because duplicated knowledge changes twice. YAGNI says do not build what you do not need yet, because speculative code is dead weight. They pull against each other. Abstract shared logic now, and you might be abstracting a coincidence. Copy the logic twice, and you might be creating a second source of truth. There is no rule that settles which is right. There is only the change that is actually coming, and the honest assessment of which bet you can afford to lose.

The principles group into families, and seeing the grouping helps you use them.

The first family is about change. SRP, OCP, cohesion and coupling, separation of concerns. They all say the same thing from different angles: the cost of a change is proportional to how much of the system must be touched, so structure the code so that things that change together live together and things that change separately are walled off from each other.

The second family is about contracts. Liskov Substitution, Interface Segregation, Law of Demeter. They are about what you promise the outside world and how little of your internals the outside world is allowed to know. They protect you from the explosion that happens when every object in a system knows the shape of every other object.

The third family is about restraint. DRY, KISS, YAGNI. They are the brakes on the first two families, because the change-oriented principles, applied without restraint, produce a system of speculative abstractions that nobody asked for.

Two of the families fighting each other is normal. The change family wants seams and indirection. The restraint family wants you to not build seams until a change shows up to use them. A design principle, applied well, is the resolution of that fight at a specific seam: here, and not before, we abstract. The engineers who apply principles well do not apply them uniformly. They apply them at the seams where change actually lands, and they leave the rest alone.

Principles also change their meaning with scale, and this is a source of confusion nobody warns you about. Apply the same advice to a method and it means one thing; apply it to a service and it means another. Splitting a method that mixes validation and pricing is a small refactor. Splitting a module that mixes HTTP and persistence is an architectural decision with a deployment cost. Engineers who carry a principle from the method level to the system level without re-weighting the cost are the ones who build a class-per-method codebase at one end and a monolith with no boundaries at the other. A design that is beautifully principle-clean at the class level can be catastrophically layered at the system level, and the fix is always the same: re-run the cost model at the new scale. The principle is the same; the scale changes the price.

There is one more thing a principle can do that is more important than its content. A principle gives you a vocabulary for arguing about design. When you say "this class has three reasons to change," you are not just describing the class, you are opening a specific question: which change is coming, and what does it cost to touch this file. The value of the vocabulary is that it makes the real question askable. The danger is when the vocabulary replaces the question.

## Real Production Usage

Watch a Spring service that has absorbed two jobs over a year. It starts as an `OrderService` that places orders. Then it starts validating payment details. Then it starts sending confirmation emails. The file grows, and the tell is the test class: every new behavior adds tests to the same test file, and every pull request touches the same service. Nothing about the framework forced this. The seams the framework gives you, `@Service`, `@Repository`, separate classes per job, were all available on day one. The absorption happened because nobody asked the cost question when a second job got glued to the first.

The inverse is just as instructive. A codebase that applies a principle reflexively, one interface per class, a "manager" for every feature, has the same symptom in a different costume: indirection that nobody can justify. Both failures are failures of judgment, not of principle. The code that ages well is the code where the seams match the changes.

## Common Mistakes

The most common mistake is using a principle as a veto word. "This violates OCP," said without a specific change in mind, is not an argument, it is a performance. The response that ends the meeting: "What change is this seam supposed to make cheaper, and what is the evidence it will come?" If nobody can answer, the principle was decoration.

The second mistake is applying a principle to the wrong scope. A throwaway migration script does not need an abstraction layer, and a core pricing engine does not get a pass on cohesion because it is "urgent." Principles are bets about future changes, and the size of the bet should match the size of the change you are betting on.

The third mistake is treating the principles as a single rule book. The families pull against each other, and the engineer who never feels the tension has probably been applying one favorite principle everywhere and ignoring the ones it conflicts with. Feeling the tension is the job.

## Interview Perspective

The question "what are SOLID" is a filter for whether you can name things, and most candidates pass it. The question that separates candidates is "give me an example where you would not apply a principle." The weak answer says you always apply them. The strong answer names the cost. "I would not introduce an interface for a class with one implementation and no second consumer, because the abstraction charges me indirection for a change that has not shown up."

The follow-up that usually comes is "what is the point of design principles, anyway." The strong answer is one sentence: they exist to keep the cost of change predictable, and every principle is a bet that a specific kind of change will come. Candidates who can answer that are describing the rest of this chapter without having read it.

## Knowledge Check

1. DRY says extract shared logic. YAGNI says do not build it yet. Two engineers on one codebase take opposite sides. Which questions would you ask to break the tie, and why do the answers settle it?

2. A reviewer blocks a pull request citing the Open-Closed Principle. The change adds a new payment method to a calculator that already has three. State the specific change the reviewer should be preparing for, and the evidence you would demand before accepting the abstraction.

3. All the principles in this chapter reduce to one goal. State it, and then explain which of the three families is the brake on the others.

## Key Takeaways

- A design principle is a bet that a specific kind of change will come, not a law.
- Every principle charges you something and buys you something; the cost model is the real subject.
- The three families, change, contracts, restraint, pull against each other, and feeling that tension is the job.
- A principle quoted without a specific change in mind is not an argument, it is a performance.

## What's Next

The first family starts with the most abused principle in the set. The next article, the Single Responsibility Principle, is about the difference between "one reason to change" and "does one thing," and why the engineers who split classes by the second rule are the ones who make codebases worse. It opens with a god class and works backward to the question that actually decides where the seams go.

---

This article explains design principles as bets about the cost of change and organizes them into the change, contract, and restraint families. Its strongest claim is that a principle quoted without a specific change in mind is a performance, not an argument, and that the real skill is holding the cost model, not memorizing the rules.
