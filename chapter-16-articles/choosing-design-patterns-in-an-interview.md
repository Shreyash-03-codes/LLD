# Choosing Design Patterns in an Interview

## Learning Objectives

- Learn to pick patterns from the problem's shape, not from a memorized menu, and to justify the choice in the language of the problem.
- Recognize the small set of patterns that actually recur in LLD case studies, and the larger set that are almost always a wrong answer in an interview.
- Practice the discipline this book has repeated in every case study: say "no pattern here" out loud when that is the honest answer.

## Introduction

Every candidate who reads about LLD walks in armed with the GoF catalog, and most of them are defeated by it. The reason is that they have learned the patterns as a menu and they are determined to order something. A parking lot gets an Observer because they like observers. A tic-tac-toe gets a State machine because it has turns. An e-commerce flow gets a Command pattern because there are buttons. The design becomes a parade of patterns, and the interviewer's one question, "why is that pattern here?", produces the answer that defines the whole interview: "because it's a classic use case." That is not a reason, and the interviewer knows it. The correct way to choose a pattern is the reverse of the way it is usually taught: start from the problem, find the place where something varies or something must be decoupled, and let the pattern be the answer to a specific question you actually asked.

## Problem Statement

Consider the concrete failure. A candidate is designing a notification system and adds a Command pattern, wrapping every notification type in a class, because they once saw a diagram. The interviewer asks "what does this buy you over a method call?" and the candidate cannot answer, because the notification types do not need to be queued, parametrized, or undone, which are the things the Command pattern buys. Meanwhile the same candidate skipped a Strategy interface on the channels, because that one was not in the diagram, and the interviewer's next question, "how do you add a WhatsApp channel?", has no clean answer. The candidate used the patterns wrong and missed the pattern they needed. That is not a knowledge failure, it is a selection failure, and selection is a skill with its own rules.

## Core Concept

The selection skill is three rules, and they are worth more than the entire GoF catalog.

**Rule one: name the variation first.** A pattern exists to absorb a specific kind of change. Before choosing any pattern, ask: what is going to vary in this system, and who is going to change it? In a parking lot, the pricing rule varies, so the Strategy at the pricing seam is justified, because "add a weekend rate" becomes a new implementation instead of an edit to the lot. In a notification system, the delivery mechanism varies, so the Channel hierarchy is justified. In a tic-tac-toe, nothing varies, so no pattern is justified. The variation question is the filter, and it works in both directions: it tells you which pattern to use and it tells you when to use none.

**Rule two: the pattern must be load-bearing, and you must be able to say what breaks without it.** This is the defense that survives the follow-up. "The pricing Strategy is load-bearing because without it, adding a rate requires editing `ParkingLot.exit`, which is the class every flow passes through." That is a complete answer: the variation, the pattern, and the cost of not having it. The candidate who cannot say what breaks without the pattern has a decoration, not a design decision. The interviewer will ask exactly this, so pre-answer it.

**Rule three: fewer patterns, better defended.** An LLD case study with one well-chosen pattern and a reason beats one with four patterns and a shrug. This book's case studies are consistent on this point: the parking lot has one seam, the elevator has one controller structure, the tic-tac-toe has none, the vending machine has a state machine because the temporal behavior is the product, the broker has a log structure that is not a GoF pattern at all. The pattern count is not a score. A design that needs no patterns and says so is showing better judgment than a design that manufactures them.

The menu that actually recurs in these case studies is short, and it is worth knowing it cold. The Strategy for interchangeable algorithms, the State machine for temporal behavior, the Observer for notification of change (the chat system's broker-subscriber shape), the Command for queued work (the job scheduler's jobs), the Facade for hiding an assembly of collaborators. That is roughly five patterns doing ninety percent of the legitimate work in this chapter. Everything else, the Abstract Factory, the Builder, the Proxy, the Flyweight, the Composite, is either a specialized fit or, in an interview, a way of spending time you do not have. When you catch yourself reaching for one of those, run rule one again and expect to find that no variation is being absorbed.

It is worth noticing how the five patterns map onto the case studies, because the mapping is the selection skill made visible. The split rules in Splitwise are a Strategy, and the elevator's dispatch policy would be one if there were two elevators. The vending machine and the booking lifecycle are State machines, because their behavior is genuinely temporal and an enum with guards is the honest size. The chat broker is Observer-shaped, the notification pipeline is Command-shaped when the job must be queued, and the library and the payment service are Facades. Every one of these choices was made in its article by running rule one, not by opening a catalog. If you can reproduce that reasoning on a new problem, you can reproduce it in the interview.

## Real Interview Context

The pattern discussion is where interviewers distinguish the candidate who learned patterns as a list from the candidate who understands them as answers. The question is almost always "why did you choose this pattern?" and the grading is not about whether the choice matches a canonical diagram. It is about whether the justification is in the language of the problem. "I made the channel a Strategy because the providers vary and I want to add one without touching the dispatcher" is a problem-language answer. "The Strategy pattern decouples algorithm from context" is a textbook-language answer. The first one demonstrates understanding; the second one demonstrates memorization. And when the candidate says "no pattern here, and here is why nothing varies," the interviewer hears something even more valuable: a candidate who will not manufacture complexity to look smart.

## Common Mistakes

The most common mistake is the pattern-first approach. The candidate picks a pattern they like and hunts for a place to put it, and the design contorts to fit the pattern. The giveaway is that the justification arrives after the pattern, "I used a Builder here, and, uh, it gives you a fluent interface," with the variation nowhere in sight. Pattern-first is how every over-engineered design in this book gets born.

The second mistake is the pattern name as the explanation. "I used the Observer pattern" as a complete sentence. The name is not the justification, the problem it solves is, and the candidate who offers the name has handed the interviewer an opening: "and what problem does it solve in this system?" The name followed by silence is a pattern shaped hole in the answer.

The third mistake is applying an application-code pattern to a storage problem. The pub-sub broker is the recurring victim: candidates try to shape the append-only log with a Visitor or a Composite, when the honest answer is that storage engines are shaped by disk physics, not by the GoF catalog. The candidate who says "there are no GoF patterns here, there is a log structure" is not admitting a gap, they are displaying senior understanding.

## Interview Perspective

A weak pattern answer is a design covered in patterns with no variation statements anywhere. The interviewer removes one pattern and asks what breaks, and the answer is "well, you lose the abstraction," which is not a thing that breaks. The interview degrades into a pattern quiz the candidate is losing.

A strong pattern answer is a design with one or two patterns, each introduced with a variation statement and defended with a breakage statement. When the interviewer proposes a different pattern, the strong candidate compares honestly: "a State machine would work here, but the enum switch is smaller and the transitions are all guarded, so I would only upgrade if the states grew past three." Follow-ups in a strong pattern interview are collaborations, not interrogations, because the candidate's justification is testable and the interviewer is testing it in good faith.

One last distinction is worth carrying into the room: a pattern discussion is not a spot-check of your recall, so do not treat the interviewer's proposed pattern as a correct answer you must agree with. The interviewer who says "would an Observer work here?" is usually testing whether you can evaluate it, not whether you can accept it. The strong response is the evaluation: "an Observer works if the display genuinely needs to react to changes from multiple places; here the return value is enough, so I would not pay for the indirection." Agreeing with every suggestion is folding, and disagreeing without a reason is rigidity. The evaluation, with the condition stated, is the third option, and it is the only one that reads as senior.

## Knowledge Check

1. A candidate uses a Factory to create vehicles in a parking lot. Run rule one: what varies, who changes it, and what is the honest answer about whether the Factory is justified?
2. Explain the difference between "I used the Strategy pattern because the pricing algorithm can change" and "I used the Strategy pattern to decouple the algorithm from the context," and which one is the interview answer.
3. A system has a single payment provider and a single currency. The candidate adds a payment Strategy interface anyway. Defend the addition or recommend removing it, using the load-bearing test.

## Key Takeaways

- Name the variation first; the pattern is the answer to a question you actually asked.
- Every pattern must be load-bearing, which means you can say what breaks without it.
- Five patterns do most of the legitimate work in LLD: Strategy, State, Observer, Command, Facade.
- "No pattern here" is a complete and often the correct answer, when nothing varies.
- Justify in the language of the problem, never in the language of the textbook.

## What's Next

Choosing the pattern is half the battle; defending the choice is the other half, and it is the subject of the next article. Discussing trade-offs is the skill the senior article called the second upgrade, and it deserves its own treatment because it is the one skill that appears in every interview, every code review, and every meeting you will ever sit in.

---

This article explains how to choose design patterns in an interview by naming the variation before the pattern and keeping only what is load-bearing. Its point is that saying no pattern at all is often the strongest answer available.
