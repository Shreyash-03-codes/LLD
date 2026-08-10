# Design Pattern Interview Strategy

## Learning Objectives

1. Bring a pattern up without name-dropping, by leading with the instability and the constraint and letting the pattern fall out.
2. Run the selection method out loud in a design session, so the interviewer sees the reasoning and not the flashcard.
3. Say "no pattern" without looking like you do not know any, using the YAGNI guard as the punchline.

## Introduction

The interview is the one place where the full catalog matters. Production code leans on a handful of patterns, and the rest of the GoF catalog is trivia until the day a specific problem arrives. The interview inverts that: a reviewer at work grades your code, an interviewer grades your design narrative. The pattern question in an interview is not "do you know the catalog," it is "can you produce a shape when the problem needs one, and can you leave it out when it does not."

The strategy this article teaches is a single sentence: talk about the problem until the pattern is the only thing left to say. The candidate who says "I would use Strategy here" has answered in six words. The candidate who says "the two payment methods vary, and adding one must not touch the order flow, so I would introduce an abstraction the flow depends on" has given the interviewer a design to react to. The pattern name is a footnote on the second answer. It is the whole of the first.

## Problem Statement

The design round gives a familiar prompt: build an ordering system, handle payments, maybe a discount engine, a notification flow. The candidate who has studied the catalog front to back starts to allocate. Order goes in a Facade. Discounts are a Strategy. Notifications are an Observer. The drawing board fills with named boxes before a single requirement is restated.

The failure is not the pattern choices. It is the order of operations. The candidate selected the shapes before the problem was understood, so every follow-up question knocks a box off the board. The interviewer says "now add a subscription that charges monthly" and the monthly charge does not fit the Strategy the candidate drew, and the candidate is now defending a diagram instead of solving a problem.

The stronger candidate reads the same prompt and does the opposite. They restate the requirements. They name the two things that vary and the one thing that is stable. They ask the clarifying question about who can change what. And then, only then, they say the word. The pattern was the last thing they reached for and the first thing the interviewer noticed, which is exactly why it landed.

## Core Concept

The strategy rests on the selection method from the How to Choose article, converted into a spoken rhythm. Three beats, repeated out loud, and the pattern falls out:

Beat one, name the instability. "The thing that changes here is the discount rule, and it changes without the order flow changing." This is the sentence the interviewer wants to hear first, because it shows you found the axis of change before you found the shape.

Beat two, name the constraint. "Adding a new rule must not modify the order code, so the rule set and the consumer have to be decoupled." The constraint is what the pattern is for. Without it, the abstraction is decoration.

Beat three, let the pattern fall out, and name it only now. "Decoupled variation that the consumer sees through one type, that is Strategy, and in Java it is just a functional interface the flow depends on."

The pattern emerges from the problem. The spoken version of the method, done right, is short:

"Two things vary here: the payment provider and the discount rule. The order flow is stable and adding a provider or a rule must not touch it, so I would give each axis an interface the flow depends on. The flow takes a `DiscountRule`, the rule is a functional interface, and the providers are injected. The pattern name for the rule is Strategy, and the injection is just the dependency-injection discipline from the earlier chapter. If the rule count stays at one, I would skip the interface and keep a method."

The spoken design has a code shape, and the shape is small:

```
@FunctionalInterface
interface DiscountRule {
    Money apply(Order order);
}

class OrderService {
    private final DiscountRule rule;
    OrderService(DiscountRule rule) {
        this.rule = rule;
    }
    Money total(Order order) {
        Money subtotal = order.subtotal();
        return subtotal.minus(rule.apply(order));
    }
}
```

The interview value of this shape is that it is a normal Java class, not a diagram. The interviewer can read it, ask about it, and propose a change to it. The Strategy name is mentioned once. The code is the argument, the name is the footnote.

The "no pattern" answer has the same rhythm and a different ending. "The requirement has one discount rule today and no sign of a second. I would write the method. The interface is the YAGNI guard talking, and I would add the abstraction when the second rule is real, because the refactor from method to Strategy is small and safe." The YAGNI guard is not an admission of ignorance, it is the punchline of the previous article, and a candidate who can name the cost of the abstraction and defer it reads as someone who has shipped.

The table for the spoken rhythm is the same map as the selection article, converted to sentences:

| The instability | The sentence to say | The pattern that falls out |
| --- | --- | --- |
| One of many variations, chosen by a key | "The strategy varies by type and the consumer must not know the types" | Factory Method |
| Variations that the flow must not know | "The flow depends on the abstraction, providers are injected" | Strategy |
| A stable algorithm with steps that vary | "The steps vary, the order of them is fixed" | Template Method |
| Objects built from many parts, construction is noisy | "Construction varies with the pieces, not the order" | Builder |
| A pipeline of handlers, each one deciding | "Each handler decides whether to handle or pass on" | Chain of Responsibility |
| One object is the gate to a subsystem | "The callers talk to one door, the subsystem is behind it" | Facade |

The meta-rule of the whole strategy is that the interviewer is not grading the pattern, they are grading the reasoning. The candidate who names Strategy in beat three has done all three beats. The candidate who names it in the first sentence has done none. The pattern is the conclusion of the argument, and the argument is the answer.

## Real Production Usage

The interview strategy is not only for interviews. It is the review skill from the anti-pattern article, spoken in a meeting instead of written in a comment. The engineer who says "the module varies on this axis and the consumer must not know it, so I am introducing the interface" has written a design doc in one sentence, and the team can react to the reasoning instead of the shape.

The pattern-first phrasing is the anti-pattern in meetings, the same way it is in code. "Let's add a Strategy here" closes the discussion, because the pattern name smuggles in every assumption the name carries. "The two sources vary, and the pipeline must not know them" opens the discussion, because the team can agree with the instability and argue with the constraint, which is the part that actually matters.

The best production demonstration of the rhythm is a codebase review where a pattern is removed. A team removes an Observer wiring that had one reactor, and the removal is justified with the instability sentence: the single reactor meant the wiring was indirection with no decoupling. The removal is the YAGNI guard applied in the other direction, and the sentence that justifies it is the same three beats, said to prove a pattern is false instead of true.

## Common Mistakes

The first mistake is the flashcard catalog. The candidate who memorized the GoF list can recite the intent of all twenty-three and cannot produce one in a design session, because the recall is keyed to the pattern name and not to the problem. The interview almost never asks for the catalog. It asks for a design, and the catalog is only useful when it falls out of the problem.

The second mistake is name-dropping early. "I would use Strategy and Factory here" in the first minute reads as decoration, and it puts the candidate on the defensive for the rest of the round. Every follow-up question becomes a test of the premature choice. The fix is to hold the name until the problem justifies it, and to be visibly holding it, "so far this looks like it wants an abstraction on the discount axis, but I want to confirm the rule count first."

The third mistake is implementing instead of designing. The candidate who types the full Strategy framework with a hierarchy of rule classes before the interviewer finished the prompt has solved a problem that was not asked. The design round wants the shape and the reasoning, not the code. The small `@FunctionalInterface` shape is the right size because it is the size of the reasoning.

The fourth mistake is freezing into the first shape. The interviewer says "now the discount is per-customer, not per-order" and the candidate who insists the Strategy still works has stopped listening. The strong answer re-runs the beats: the instability moved, the constraint moved, so the shape moves, and the name may change. The interviewer is testing whether the pattern is a conclusion or a commitment.

## Interview Perspective

The pattern question is the one place where the full catalog is a liability, and the sharpest candidates treat it that way. The interviewer asks "what is the most useful design pattern" and the weak answer is a list of favorites with reasons. The strong answer runs the frame from this chapter. "The most useful one is the one that falls out of the selection method, because the pattern is the conclusion and the problem is the argument. In the code I actually ship, the ones that recur are Strategy as a functional interface, Factory Method behind a container, and Facade as a door to a subsystem. The rest of the catalog is there for the days the instability shows up."

The behavioral question "tell me about a time you used a design pattern" is the same frame in a story. The weak story is "I used the Observer pattern in a project." The strong story is "we had a module that needed a new handler without touching the pipeline, and the constraint was that the handlers must not know each other, so the pipeline took a functional interface. That was the Observer, or Chain of Responsibility, depending on how you read it, and the reason it worked is the constraint, not the name." The interviewer grades the same three beats, in a story instead of a design.

The follow-up "would you always abstract here" is the YAGNI guard asked directly. The weak answer is "yes, abstraction is good design." The strong answer is "no, the method is fine until the second variation is real, and the refactor from method to abstraction is small. The guard is that I can name what changes, and right now nothing does."

The strategy compresses to one line for the room: the pattern is the conclusion of the argument, and the argument is the answer. Lead with the instability, hold the name until the problem earns it, and treat the "no pattern" answer as the sign of someone who has shipped.

## Knowledge Check

1. A prompt asks for a notification system where the message type varies and the pipeline must not know the types. Walk the three beats out loud and name the pattern that falls out, in the order the beats run.

2. A design session prompt has one discount rule and no stated variation. State the answer that uses the YAGNI guard, and the exact refactor you would name as the escape hatch.

3. An interviewer asks why you named the pattern last instead of first. Give the one-line justification in terms of the instability and the argument.

## Key Takeaways

- Talk about the problem until the pattern is the only thing left to say, the name is the footnote, the reasoning is the answer.
- The three beats are the spoken selection method: name the instability, name the constraint, let the pattern fall out.
- The "no pattern" answer is a strength, and the refactor from method to abstraction is the escape hatch that makes it safe.
- The interviewer is grading the argument, and a pattern named in the first sentence has skipped the argument entirely.

## What's Next

The chapter's map is complete: the frame, the history, the categories, the selection, the failure modes, and the interview. The next chapter turns from the map to the first family and finally shows the catalog in the code. Creational Design Patterns covers the five patterns that control construction, the Singleton and its honest alternative, the Factory Method, the Abstract Factory, the Builder, and the Prototype, and the chapter begins the transition from choosing shapes to writing them.

---

This article explains the interview strategy for the design pattern catalog, leading with the instability and the constraint and letting the pattern fall out last. Its strongest claim is that the pattern is the conclusion of the argument, and that saying "no pattern" with the YAGNI guard reads as experience, not ignorance.
