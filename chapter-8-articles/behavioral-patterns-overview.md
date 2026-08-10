# Behavioral Patterns Overview

## Learning Objectives

- Map each of the eleven behavioral patterns to the specific runtime problem it owns, so you can recognize the pattern from the failure instead of from a diagram.
- Name the one question all behavioral patterns answer: how do objects cooperate at runtime without hardcoding who talks to whom.
- Keep the creational and structural patterns straight from this chapter by remembering what axis changed: this chapter moves from shape to behavior.

## Introduction

Behavioral patterns are the eleven GoF patterns that deal with how objects interact at runtime: Strategy, Observer, Command, Chain of Responsibility, Template Method, State, Mediator, Iterator, Memento, Visitor, and Interpreter. Creational patterns put something else in charge of `new`. Structural patterns put something else in charge of the object graph's shape. Behavioral patterns put something else in charge of the conversation itself.

That shift is the whole chapter in one sentence. The two previous chapters asked what objects are built from and how they sit next to each other. This one asks what happens when the system is running, which is the phase where most production bugs actually live. A class diagram that looks perfect can still produce a system that talks to itself in circles, because the diagram says nothing about who triggers whom, who is allowed to change what, and who finds out when something happened.

## Problem Statement

The failure is hardcoded behavior, and it is sneakier than hardcoded structure. Consider a payment flow that has grown a few branches over time. A `PaymentService` computes fees differently per currency, sends a receipt, notifies inventory, and updates a ledger, all in one method, with `if (currency == ...)` sprinkled through it. Every new currency, every new side effect, means editing the same class. The structure of the system is fine. Nobody is wrapping anything in the wrong place. The behavior is the problem: the objects' interactions are written directly into each other, so the system can only ever do what the original author hardcoded.

Real systems decay along exactly these lines. A notification feature that started as one email has grown into email, push, and SMS, all driven by a boolean flag. A workflow with two steps is now ten, controlled by nested conditionals. An algorithm that was fine at launch now has four different variants chosen by an environment variable. In every case the failure is the same: the interaction, which variant to use, what to do when something changes, who gets told, is glued to a concrete implementation, and every change to the behavior is a change to existing code.

Behavioral patterns exist to move those decisions out of the code that makes them. The strategies, the observers, the commands, the states, they all give the runtime decision a home of its own.

## Core Concept

The eleven patterns are not interchangeable, and the way to keep them straight is to group them by the question they answer. Four clusters cover almost all of them, and the exceptions have nowhere else to live.

| Pattern | The question it answers | The shape of the answer |
|---------|-------------------------|------------------------|
| Strategy | Which algorithm should run? | Swap the algorithm behind an interface |
| State | What is the object's current condition? | The condition selects the behavior |
| Command | How do I package an action? | The action is an object |
| Observer | Who finds out when this changes? | A one-to-many push |
| Mediator | How do many objects talk without a tangle? | One central coordinator |
| Chain of Responsibility | Who handles this? | Pass it down until someone takes it |
| Iterator | How do I walk a collection without knowing its layout? | A cursor |
| Template Method | How do I fix the skeleton but let steps vary? | Subclasses fill in the steps |
| Memento | How do I undo without breaking encapsulation? | A snapshot outside the object |
| Visitor | How do I add an operation to a fixed set of classes? | Double dispatch |
| Interpreter | How do I evaluate a language? | A grammar of objects |

The first three, Strategy, State, and Command, are the workhorses of refactoring legacy conditionals, and they are the ones you will actually reach for. The next three, Observer, Mediator, and Chain of Responsibility, are about wiring: who talks to whom. Iterator and Template Method are old, boring, and everywhere. Memento and Visitor are the rare pair, and Visitor is the one with a reputation that exceeds its usefulness. Interpreter is the pattern nobody admits to using, and in this handbook it gets one mention here and no article of its own, because in practice you reach for a parser library, not a pattern named Interpreter.

Two of these patterns deserve a warning before you meet them in detail. Observer is the pattern people overuse until their codebase becomes a web of subscriptions nobody can trace, and it is the reason event-driven code is so often called "magic." Mediator is the opposite trap: it starts as a tidy coordinator and quietly becomes the god class you were trying to avoid. Both warnings are specific, and both articles in this chapter will say exactly why.

The question that separates this chapter from the last one is worth stating once, because it is the mental move the whole chapter depends on. Structural patterns ask how objects are arranged. Behavioral patterns ask what the objects do to each other while the system runs. A composite tree can be perfectly shaped and still have no idea when to notify its parents. A proxy can control access correctly and still compute fees with the wrong algorithm. Shape does not buy behavior. This chapter is where the behavior gets built.

## Real Production Usage

The JDK runs on behavioral patterns so heavily that you stop noticing. `java.util.Collections.sort` and `Arrays.sort` accept a `Comparator`, which is Strategy in one line, and the streams library is Strategy everywhere, because you supply the behavior and the pipeline supplies the structure. `Observer` and `Observable` were in the JDK for twenty years before being deprecated in favor of `java.util.concurrent.Flow`, which keeps the same subscriber-push shape. The `Executor` framework is Command: every `Runnable` and `Callable` you submit is a command object that the executor schedules and runs. `Iterator` is so fundamental that every `for-each` loop in Java is secretly an iterator call, and the `ListIterator` and `Spliterator` variants extend the same idea.

Spring is the other half of the story. The Spring Security filter chain is Chain of Responsibility, and it was named as such in the previous chapter. `RestTemplate` and `JdbcTemplate` are Template Method, because they fix the skeleton of an operation and let you supply the callback. The `PlatformTransactionManager` hierarchy, with `DataSourceTransactionManager` and `JpaTransactionManager`, is Strategy for the "how do I manage a transaction" algorithm. Hibernate's entities use State for lifecycle transitions, and its first-level cache uses a memento-ish snapshot to detect dirty state. When you understand the behavioral patterns, a framework like Spring stops being a pile of annotations and becomes a catalog of runtime decisions, each with a shape you already know.

## Common Mistakes

**Treating behavioral patterns as a menu to decorate the code with.** Strategy, Command, and State exist to replace conditionals that actually vary. Applying Strategy to an algorithm that never changes just adds an interface and a factory for zero payoff. The pattern is justified by the variation, not by the diagram.

**Underestimating Observer's hidden coupling.** The whole selling point of Observer is that observers do not know each other. The whole danger is that nobody knows anything, and a subscription web that spans ten classes is unreadable. Observer trades explicit coupling for implicit coupling, and the implicit kind is the one that surprises you at three in the morning.

**Confusing the wiring patterns with each other.** Observer, Mediator, and Chain of Responsibility all answer "who gets this message," and engineers blur them constantly. Observer pushes to everyone who subscribed. Mediator routes through one coordinator. Chain passes to the first handler that takes it. If you cannot say which one a requirement needs, you will build the wrong one.

## Interview Perspective

Behavioral patterns are where interviewers test whether you can apply patterns to a running system instead of to a drawing. A weak answer recites definitions and draws class diagrams. A strong answer names the failure first, "this method has three currency branches and every change touches the class," and then names the pattern that removes the branch, "Strategy, one interface, three implementations, chosen at wiring time."

The follow-up that sorts people is usually about a specific refactor. "This class has a boolean flag that changes its behavior between two modes. What do you do?" The strong answer names State, because a flag that changes behavior over time is a state machine in disguise, and then explains why the boolean leaks through the whole class while the State pattern keeps each mode isolated. The skill being tested is not pattern recall. It is the ability to see the runtime failure and pick the shape that removes it.

## Knowledge Check

1. A notification service has grown `if (channel == EMAIL)` through every method. Which pattern removes the conditionals, and what does each branch become?
2. Observer, Mediator, and Chain of Responsibility all move a message. Describe the same message in all three shapes and say when each is the right choice.
3. `for (String line : file)` compiles to a hidden iterator. What does the iterator actually encapsulate, and what would the loop look like if the collection were a tree?

## Key Takeaways

- Behavioral patterns own the runtime conversation: who triggers, who is told, who handles, which algorithm runs.
- The failure they attack is hardcoded behavior, conditionals and wiring glued into concrete classes.
- Strategy, State, and Command are the refactoring workhorses for legacy conditionals; Observer, Mediator, and Chain handle wiring.
- Observer overuse creates unreadable subscription webs, and Mediator overuse creates a god class, both traps are real.
- Shape does not buy behavior. The previous two chapters arranged objects, this chapter makes them act.

## What's Next

The next article is Strategy, the pattern you reach for when a method keeps growing conditionals. Strategy takes the algorithm out of the method and puts it behind an interface, so the code that uses it picks the variant at wiring time instead of at write time. We will cover the interface, the concrete strategies, and the honest limit: when the variation is small enough that Strategy is overhead.

---

This article explains the eleven behavioral patterns as answers to how objects cooperate at runtime, grouped into four clusters that map to the failures they remove. The failure it fixes is hardcoded behavior, and recognizing that failure matters more than recalling a diagram.
