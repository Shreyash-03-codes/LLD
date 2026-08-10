# How to Choose a Design Pattern

## Learning Objectives

1. Select a pattern by naming the instability in the code first, so the choice falls out of the problem instead of out of the catalog.
2. Use consequences, not resemblance, to break a tie between two candidate patterns.
3. Apply the YAGNI guard, so the pattern appears only when the problem is real, and you can say which change it is for.

## Introduction

The catalog is the easy part. Anyone can learn twenty-three names. The skill is choosing, and most engineers choose badly, because they browse the catalog instead of reading the problem. They see an interface with two implementations in their head and shout "Strategy," without asking what the code is actually struggling with.

The right way around is almost mechanical. Name the instability, the thing that changes and causes the pain. Name the constraint, what the change must not break. The pattern is usually the answer to those two sentences, and if no pattern falls out, the honest conclusion is that you do not need one yet.

## Problem Statement

A codebase has a fee calculation that grew an if-else chain. Every few months a new order type appears, and the chain grows by one branch.

```
public BigDecimal fee(Order order) {
    if (order.type() == OrderType.STANDARD) return standardFee(order);
    if (order.type() == OrderType.EXPRESS) return expressFee(order);
    if (order.type() == OrderType.BULK) return bulkFee(order);
    return minimumFee(order);
}
```

The engineer who owns it has read a blog post listing the GoF patterns and decides the code needs Strategy. The interface goes in, two classes come out, the callers are injected, and the diff is beautiful. Three weeks later the code review finds that `BULK` never actually varies at runtime, it has been one constant formula since day one, and the new interface is now the ceremony around a value that could have been a method. The pattern was applied to a problem that did not exist.

The failure is selection without diagnosis. The engineer saw the shape, an interface with implementations, and grabbed the pattern that matched the shape. The honest method goes the other way: ask what is actually changing, ask whether the change is real, and only then reach for the pattern. The if-else chain is not automatically a Strategy problem. It is a Strategy problem only if order types genuinely vary at runtime and the caller should not know which one is active.

## Core Concept

The selection method has three steps, and the order is the whole thing.

Name the instability. Find the code that changes and ask why it changes. The fee method changes because order types change. The gateway call changes because providers change. The notification wiring changes because new reactors appear. The instability is the sentence that starts with "every time." Every time we add an order type, the fee method grows. Every time we add a provider, the checkout changes. Every time we add a notification subscriber, the order code changes. That sentence is the problem, and the pattern is the answer to it.

Name the constraint. What must the change not break? When a provider is added, the checkout must not change. When a notification subscriber is added, the order code must not change. When a fee rule is added, the existing callers must not change. The constraint is what the pattern protects, and it is usually the same shape as the instability, something must be able to change without touching something else.

Let the pattern fall out. With the instability and the constraint named, the pattern is usually forced. "Adding a subscriber must not touch the order code" is Observer. "Adding a provider must not touch the checkout" is Strategy, the provider being the swapped behavior. "The caller should not build the object" is a factory. The pattern is not chosen from a list, it is named by the answer.

The worked example is the fee chain above. The instability: order types keep appearing. The constraint: adding a type must not modify the fee method and must not touch callers. The pattern that falls out: Strategy, a `FeeRule` interface, each order type supplying a rule, the calculator holding one and swapping it at runtime. But only if the variation is real. If the business says order types never change, the instability sentence is false, and the honest choice is to leave the if-else alone.

The problem to pattern map, read as "if the instability is this, the candidates are these":

| The instability | The sentence that names it | Candidate patterns |
| --- | --- | --- |
| Behavior or algorithm varies at runtime | "which rule is active depends on the request" | Strategy, State |
| Objects are created in many places, concrete classes hard-coded | "this code is welded to one class" | Factory Method, Abstract Factory |
| Many objects must react to one event | "adding a reactor must not touch the source" | Observer |
| Work must be queued, logged, retried, or undone | "a call must be deferred or reversed" | Command |
| A complex object has too many construction steps | "the constructor is unreadable" | Builder |
| A subsystem has too many parts for the caller | "the caller reaches into five classes" | Facade |
| An interface does not match what the client needs | "we translate the SDK everywhere" | Adapter |
| A shared algorithm differs in one step | "the methods are copy-paste except one line" | Template Method |
| An object's behavior changes with its internal state | "the if-else is on the status field" | State |

The map is a starting point, not a verdict. The map sends you to a shortlist, and the shortlist is broken by consequences. That is the step most engineers skip, and it is the step that separates a good choice from a defensible one. Two patterns can answer the same instability, and the consequences are what pick between them.

Strategy versus State is the classic tie. Both answer "behavior varies at runtime." The consequence decides: Strategy swaps an algorithm, and the context does not care which is active, a fee rule. State swaps behavior because the object's own condition changed, and the swap is driven by events, a vending machine. The question is whether the variation is a choice the caller makes or a condition the object is in. Same family, different consequence.

Factory Method versus Abstract Factory is the other common tie. Both answer "creation is welded." Factory Method creates one product, and a subclass picks the type. Abstract Factory creates a family that must stay consistent, buttons and menus that match. The consequence decides: do you need one object decoupled, or a matching set? If the answer is one, Factory Method. If the answer is a set, Abstract Factory.

Then the YAGNI guard, and it runs last, not first. The guard asks one question: is the instability real, and does it need to be handled now? A codebase that has added one order type in three years does not have a Strategy instability, it has a method. A codebase that has added six providers in two quarters has one. The pattern earns its indirection only when the instability sentence is true and the change is near enough that the pattern pays for itself. Choosing a pattern for a change that may never come is speculative abstraction, and it is the same disease the DRY and YAGNI article spent its length on.

The last part of choosing is naming what you are buying. Every pattern buys a kind of flexibility and pays for it. Strategy buys swappable behavior and pays in indirection and classes. Observer buys decoupled reaction and pays in indirection and harder-to-trace flow. Builder buys readable construction and pays in a second class. An engineer who can say "I am buying X and paying Y" has made a decision. An engineer who can only say "the book does it this way" has made a citation.

## Real Production Usage

Selection in production is mostly a review activity, and the pattern names are how it happens. "This is a Strategy, the if-else should be a rule interface" is a review that names the instability and the answer in one line. "Do not make this an Observer, the source should know its reactor" is a review that applied the constraint test. The names are the selection vocabulary, and a team that has them selects faster than a team that has to describe the shape in prose.

The framework does the selection for you more often than you think. When you declare a Spring bean and the container injects it, the container chose a pattern, the factory and the singleton registry, on your behalf. When you use `List.of` or a stream collector, the JDK chose the factory family for you. A large share of production "pattern selection" is deciding when to let the framework do it and when to write the pattern yourself. The honest default is to let the framework do it, and to write the pattern only when the framework does not cover the instability.

The JDK is also the model of consequence-aware selection. The collection factory methods chose the factory family and documented the consequences, immutable, null-hostile, unpredictable iteration order for sets. `Optional` chose the monadic shape and documented its consequences, no nulls, no direct access. The standard library is the best working example of naming the instability, choosing the shape, and stating the trade.

## Common Mistakes

The first mistake is browsing the catalog. Engineers who open a list of patterns and look for one whose diagram matches their code are selecting by resemblance, and resemblance is the weakest signal. Every pattern in the structural family looks like boxes and arrows. The honest method starts from the instability sentence, and the pattern is the answer, not the start.

The second mistake is the golden hammer. An engineer who learned Strategy deeply sees Strategy problems everywhere, and one who learned Singleton sees one-instance problems everywhere. The hammer is the self-flattering version of the catalog browse, and it produces the pattern applied to the non-problem, the `BULK` case that never varied. The guard is the instability sentence, and it refuses the hammer.

The third mistake is ignoring consequences. The two-pattern tie is broken by the trade, and an engineer who cannot name the trade is picking by taste. "Strategy feels right" is not a decision. "Strategy buys swappable behavior, and the fee rules genuinely vary, so the indirection is worth it" is a decision. The consequences are the part of the pattern that is actually about your code.

## Interview Perspective

The selection skill is what interviewers probe when they ask "design a payment system and tell me why you chose that shape." The weak answer names patterns without the diagnosis. "We could use Strategy here, and maybe an Observer, and a Factory for the gateway." The strong answer runs the method. "The instability is providers, every new provider must not touch the checkout, so the gateway is an interface and the checkout holds one, that is Strategy. The instability is order events, adding a subscriber must not touch the order code, so I publish events, that is Observer."

The follow-up that separates them is "why Strategy and not State." The weak answer cannot say. The strong answer uses the consequence. "Strategy swaps an algorithm, and the caller chooses which one, a fee rule picked per request. State swaps behavior because the object's condition changed, and events drive the swap. Here the variation is a choice, not a condition, so Strategy, not State."

The sharper follow-up is the YAGNI probe, "could you build this without any patterns." The weak answer panics and adds patterns to defend against the question. The strong answer holds the line. "Yes, and I would. The providers do not vary yet, so the interface is a seam, not a necessity. I would build the simple version, and the seam is where Strategy plugs in the day the second provider arrives." The interviewer is not testing pattern fluency. They are testing whether the pattern is a decision or a reflex.

## Knowledge Check

1. A report service has a method that differs between PDF and CSV output by one step, and the two versions are copy-paste. Name the instability, state the pattern the method points at, and state the consequence you are paying.

2. Two teammates argue for Strategy and State on the same class, a shipping calculator whose behavior depends on the shipment's status. State which is correct and the question that settles the argument.

3. An engineer proposes a Factory for a single class that has been constructed the same way for three years. Run the three-step method on the proposal: what is the instability sentence, is it true, and what does the YAGNI guard say?

## Key Takeaways

- Selection starts from the instability sentence, and the pattern is the answer to it, not the start.
- The problem to pattern map builds a shortlist, and consequences, not resemblance, break the tie.
- Every pattern buys a kind of flexibility and pays for it, and naming both is what makes the choice a decision.
- The YAGNI guard runs last: if the instability is not real, the pattern is ceremony, and the seam is enough.

## What's Next

Choosing well is one half of the skill, and choosing badly is the other half worth studying. The next article is the negative image, the anti-patterns and the misuses, the God Object, the Singleton abuse, the pattern spam, and the honest replacement for each one. It is the article that keeps the selection method from being undone by its own enthusiasm.

---

This article explains how to choose a design pattern by naming the instability first, building a shortlist, and breaking ties with consequences. Its strongest claim is that a pattern is selected by the problem sentence, never by matching the diagram, and that the YAGNI guard runs last, refusing the pattern when the instability is not real.
