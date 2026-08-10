# DRY, KISS, and YAGNI Principles

## Learning Objectives

1. Define DRY as not duplicating knowledge, and distinguish real duplication from coincidental repetition.
2. Explain YAGNI as a bet against speculative cost, and say when the bet is right and when it is a cop-out.
3. Use KISS as a design constraint that cuts both ways, protecting the reader, not just the author.

## Introduction

DRY, KISS, and YAGNI are the restraint family. The change-oriented principles build seams and abstractions. These three are the brakes: do not repeat knowledge, keep it simple, do not build what you do not need. They are the least glamorous principles in design, the ones that get quoted in code reviews as "this is overengineered," and the ones most often misquoted to mean the opposite of what they say.

Each one is a bet against a specific cost. DRY bets that duplicated knowledge will have to change twice and diverge. YAGNI bets that speculative code will cost more to carry than it saves. KISS bets that the reader's time is the scarce resource. The three share a warning: the simplest thing is often the right thing, and the code that does not exist cannot be wrong.

## Problem Statement

The codebase has a `UserValidator` and an `OrderValidator`, and both contain the same email validation, copy-pasted with one subtle difference. The regex in the user validator allows the new domain, the order validator still uses the old one. A change request "relax email rules" lands, one validator gets fixed, and the team ships. Two weeks later the order validator rejects an email the product team promised the customer.

That is the DRY failure, divergence, duplicated knowledge that changed once. The two validators did not look duplicated when the copy was made, and they did not look related when the first one was edited. The cost of the duplication was invisible until the moment two truths disagreed.

Now the same codebase, other side of the ledger. A junior engineer, told to make the code DRY, extracts a `EmailUtil`, a `ValidationBase`, and a `SharedHttpClientWrapper`, because the code "repeats." The three abstractions are each used once, each a file that must be opened to understand a call, and the codebase is harder to read than the duplication was. The "fix" cost more than the problem, and that is the other failure: restraint applied as a law.

## Core Concept

DRY means Don't Repeat Yourself, and the precise statement is about knowledge, not text. Two places that know the same fact, the same rule, the same formula, are duplicated knowledge, because the fact can change and must change in both places. Two places that happen to look similar, but know different things, are not duplication, they are coincidence, and extracting them creates a false coupling that will hurt.

The email example is real duplication: both validators know the email rule, one fact, two homes. The fix is a single validator both call. The counterexample: two methods that both loop over a list and sum, but one sums prices and one sums counts. The text looks similar, and the knowledge is different. Extract a `sumDoubles` and you have created a coupling between two rules that will change at different times. Coincidental duplication should be left alone, and telling it apart from real duplication is the actual skill behind DRY.

The test for real duplication is the change question: when this fact changes, how many places must change in the same release? One fact, many homes that must all change together, that is duplication that will diverge. The `sum` loop changes with the pricing rule, the count loop changes with the reporting rule, different releases, no divergence risk. DRY is about the first kind.

YAGNI, You Aren't Gonna Need It, is the bet that the code you are about to write speculatively will not be used. The canonical shape is the configuration flag for a feature that does not exist, the generic parameter for a case that has not arrived, the abstraction for the second implementation that is still hypothetical. Each is code that must be read, tested, and carried, for a future that may not come.

The honest reading of YAGNI is a cost comparison, not a prohibition. The speculative code costs you now, in reading and carrying it. It saves you later, if the future arrives, in the time it would take to build it then. YAGNI says the now-cost is certain and the later-saving is a guess, and you should not pay a certain cost for a guess. The bet is wrong when the future is not a guess: a public API contract, a protocol, a payment integration, things that are expensive to retrofit and known to be coming. YAGNI applied to those is not restraint, it is the cop-out that produces the rewrite.

KISS, Keep It Simple, Stupid, is the protection of the reader. The simplest design is the one the next engineer can hold in their head, and the complexity budget is the reader's attention. The failure mode is the clever solution: the bit-twiddling hash, the operator-overloaded expression, the three-line trick that the author can explain and nobody else can maintain. KISS is the argument for the boring implementation that works.

KISS cuts both ways, and this is where most engineers get it wrong. It is not an argument against structure. A strategy pattern with three small classes is simpler to understand than one 200-line method with five boolean branches, even though it has more moving parts, because each part is individually understandable. KISS is about cognitive load, not file count. The simplest thing that works is the thing with the least load, and sometimes that is the interface, not the if-chain.

The three principles interact, and the interaction is where the judgment lives. DRY wants an abstraction, YAGNI says there is no second user, KISS says the duplication reads fine. The resolution is the evidence: the abstraction earns its existence when the duplication proves it is real, when the second place actually changes in the same release. Until then, YAGNI and KISS win, and the duplicated code is the honest version.

## Real Production Usage

The divergence failure is the everyday production story. Two services each embed the same status-code mapping, a change to the mapping updates one, and the other returns the stale code for six months. Nobody needs a framework to explain it, the two copies were the design, and the design failed the day they diverged. The fix, one shared mapping, is DRY doing its job.

The speculative failure is just as common. An event enum with twenty cases, twelve of which have no handler, built for a roadmap that got cut. The dead cases are carried by every switch, every test, every serialization, and the team that built them defends them with "we planned it." The planning was a guess, and the guess became a tax. The event types that actually shipped could have been added in an afternoon each.

The famous projects are a KISS case study in both directions. The ones that shipped simple and stayed simple beat the ones that shipped a framework of abstractions. The principle is visible in the good ones: a core small enough to understand, and complexity added only where the domain demands it.

## Common Mistakes

The most common mistake is DRY applied to coincidence. Two methods that look alike are merged into one, and the merge couples two rules that change at different times. The next change to one is now an edit to a shared method with two callers, and the fix for one breaks the other. The change question, do these change in the same release, separates the real duplication from the lookalike.

The second mistake is YAGNI used as a veto on all design. "We do not need an interface" said to the payment gateway that is about to have a second provider is not restraint, it is the cop-out. The bet needs the evidence of a known-coming change, and the engineer who never builds for any future builds a codebase that cannot grow. YAGNI is a cost comparison, not a faith.

The third mistake is KISS used to justify unreadable code. "This is simpler" said about the clever one-liner that three people have to decode, or the god method that avoids the "complexity" of a second class. The complexity did not go away, it moved into the reader's head. KISS is measured in cognitive load, and the obvious implementation wins even when it has more lines.

## Interview Perspective

The question "what does DRY actually mean" filters candidates hard. The weak answer is "don't copy-paste." The strong answer is "don't duplicate knowledge. Two places that know the same rule are duplication, because they change together. Two places that look similar but know different rules are coincidence, and extracting them couples them. The test is what changes together." The candidate who can say that has actually read the principle.

The question "would you refactor this duplicated code" wants the decision run. The strong candidate asks the change question before touching anything: "Does this fact change in one place or would a change hit both? If both, extract. If it is a lookalike, leave it."

The sharper question: "isn't YAGNI just an excuse to under-engineer." The strong answer holds the line. "YAGNI is a cost comparison. I pay a certain cost now to carry speculative code, against a guessed saving later. When the future is known, a public API, a planned integration, YAGNI does not apply. When it is a guess, it does."

## Knowledge Check

1. Two classes each build a query string with the same five lines, but one sorts by date and the other by status. Is this real duplication or coincidence, and what change question decides it?

2. A team adds an abstraction for a second database provider that has no planned date and no committed vendor. Apply YAGNI to the decision, and state the evidence that would reverse your answer.

3. A method with five boolean flags controls three behaviors. A colleague argues for a strategy pattern; another argues KISS means keep the flags. Which side has the correct reading of KISS, and what is the measure?

## Key Takeaways

- DRY is about duplicated knowledge that changes together, not about similar-looking text.
- YAGNI is a cost comparison, not a faith, and it loses when the future is known-coming.
- KISS is measured in the reader's cognitive load, not in file count or cleverness.
- The three are the brakes on the abstraction the change-oriented principles are eager to build.

## What's Next

The restraint family covers what not to build. The next article, High Cohesion and Loose Coupling, turns to the structure of what you do build: the inside and the outside of a module, and why the coupling everyone measures is usually a cohesion problem in disguise. It covers how to read a class for both, and which one is actually the goal.

---

This article explains DRY, KISS, and YAGNI as the restraint family, three bets against the cost of overbuilding. Its strongest claim is that DRY is about duplicated knowledge that changes together rather than similar text, and that YAGNI is a cost comparison that loses when the future is known to be coming.
