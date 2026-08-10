# What Are Design Patterns?

## Learning Objectives

1. Define a design pattern by its four parts, name, problem, solution, and consequences, and separate it from a library or an algorithm.
2. Recognize the patterns you already write by name, so the vocabulary becomes useful instead of ornamental.
3. Apply a pattern as a starting point, not a goal, and state what it costs before you adopt it.

## Introduction

A design pattern is a named solution to a problem that shows up in a thousand codebases. Nothing more mystical than that. The name is the point, because the name is what lets two engineers say "that's a Strategy" and both know what the other means in a single sentence.

Most engineers already know several patterns. They have written a class that another class extends, overriding one hook method. They have injected a collaborator so behavior can be swapped at runtime. They just did not have the names. This article gives you the frame, the four parts, the boundaries, and the honest caveats, so the names in the rest of this chapter have somewhere to land.

## Problem Statement

Two teams solve the same problem in the same year. Team A needs to switch between payment providers at runtime. An engineer writes an interface called `PaymentGateway`, two implementations, and a service that receives a provider in its constructor. Team B, a month later, needs to switch between notification providers, and an engineer independently builds the same shape: an interface, two implementations, constructor injection. Nobody recognizes they solved the same problem twice, because neither had a name for the shape they were building. Team B re-litigates the edge cases Team A already settled.

Worse, when the teams finally compare notes, they describe the solution differently. "We abstract the provider" and "we swap the backend" are the same thing said two ways. The code review that should say "this is a Strategy, you solved this in the checkout module" cannot be said, because no shared word exists. The same is true of half the "clever" code you have ever written or reviewed. It was a pattern, unnamed, and being unnamed it was re-invented badly a few times before someone got it right.

## Core Concept

A pattern has four parts, and all four matter. Drop one and you have a diagram, not a pattern.

The name. The name is the vocabulary. It turns a three-paragraph explanation into two words. When an engineer says "let me wrap that in an Adapter," the team knows the shape being proposed, the class that translates one interface into another, without a drawing. The name is what makes the pattern a pattern instead of just a good idea.

The problem. Every pattern names the specific situation it solves. This is the part most people skip, and it is the part that prevents misuse. The Strategy pattern solves "an algorithm or policy varies at runtime and the caller should not know which one is active." If your code does not have that variation, Strategy is the wrong tool, no matter how clean the interface diagram looks.

The solution. The solution is a description of the participants and their responsibilities, not a piece of code. It says "define an interface for the family of algorithms, and have the context hold a reference to one of them." It does not say "write a class named `PaymentGateway`." You implement the solution in your codebase, with your names and your constraints, and the result will differ from every other implementation of the same pattern.

The consequences. A pattern trades something to get something. Strategy buys runtime flexibility and pays with a layer of indirection and more classes. Singleton buys one shared instance and pays with global state and a hidden dependency. The consequences are the honest part, and an engineer who can state the trade before adopting the pattern is an engineer who is using patterns, not collecting them.

A pattern is not a library. You cannot `import` a pattern. A library is finished code you call; a pattern is a shape you rebuild in your own code because your constraints differ. The `java.util.Collections` class ships sorting code, and the Strategy pattern ships nothing, because the sorting code is the algorithm and the `Comparator` you pass it is the strategy.

A pattern is not an algorithm either. An algorithm solves a computation, quicksort, binary search, and it has a definite procedure. A pattern solves a structural problem, how objects collaborate, how responsibility is divided, and it has a family of valid implementations. You implement quicksort the same way everywhere or it is not quicksort. You implement Strategy differently in every codebase and it is still Strategy.

The closest thing to seeing this distinction in code is the hook method, and it is worth meeting in Java because it is everywhere.

```
public abstract class NotificationSender {

    public final void send(Message message) {
        validate(message);
        deliver(message);
    }

    private void validate(Message message) {
        if (message.recipients().isEmpty()) {
            throw new IllegalArgumentException("no recipients");
        }
    }

    protected abstract void deliver(Message message);
}

public class EmailSender extends NotificationSender {
    @Override
    protected void deliver(Message message) {
        emailClient.send(message);
    }
}
```

That is the Template Method pattern. The base class fixes the skeleton, `validate` then `deliver`, and lets a subclass supply one step. You have written this shape, or a near cousin of it, even if you never had the name. The frame of the pattern tells you what you are looking at: the name, `Template Method`; the problem, subclasses share an algorithm but differ in one step; the solution, an abstract class with a final skeleton and an abstract hook; the consequence, reuse of the shared steps and a cost, the subclass is bound to the base class and the base class decides the order.

Once you have the four parts, the rest of the chapter is mostly naming. When the chapter lists the twenty-three patterns, what you are really learning is twenty-three names for shapes you have probably already drawn, plus the problems each one solves and the cost of using it.

One more boundary, and it is the one that keeps patterns honest: a pattern is a starting point, not a goal. The pattern gives you a known-good shape, and then real constraints bend it. The moment you catch yourself applying a pattern so that you can say you applied it, the pattern has cost you more than it returned. The test is the problem part: if the problem the pattern solves is not present, the pattern has no business in your code.

## Real Production Usage

Patterns in production are mostly invisible, because they were absorbed into the framework you are standing on. Spring is a museum of patterns. The `ApplicationContext` is a Factory, it creates and wires objects, and a Singleton registry, it hands back the same bean for the same name. `JdbcTemplate` is Template Method with a lighter skin. The whole `ProxyFactoryBean` mechanism is the Proxy pattern, intercepting calls on a bean. When you use Spring, you are using patterns that have been compiled into the framework, and you mostly do not need to write them yourself.

The JDK is the same story. Java I/O is the Decorator pattern done in the standard library: `new BufferedInputStream(new FileInputStream("x"))` wraps an input stream in another input stream, each wrapper adding behavior, buffering, gzipping, and the caller treats the stack as one stream. `java.util.Collections.unmodifiableList(...)` is a decorator that forbids mutation. Every Java engineer has used Decorator without naming it.

The production value of the names shows up in review. "This is a Template Method, move the shared validation up" is a one-sentence review that would otherwise be three paragraphs. The names compress knowledge, and compression is what lets a large team reason about structure at all. That is why patterns survived the backlash that killed the heavier modeling fads: the vocabulary survived because it is cheap and it works.

## Common Mistakes

The first mistake is treating the catalog as the subject. Engineers who memorize twenty-three names and can recite them on demand but cannot say what problem each solves have learned a party trick, not a skill. The name without the problem is a word you cannot apply. In an interview, the recitation is a giveaway, and in a review, it is noise.

The second mistake is using patterns as goals. "We should introduce the Observer pattern here" said before anyone has named the problem is pattern-collecting. The honest sequence is the reverse: name the problem, "when an order is placed, several modules need to react, and adding a new reaction should not touch the order code," and then Observer is the answer that falls out. Patterns follow problems, never the other way around.

The third mistake is conflating the pattern with its textbook example. The GoF book draws Strategy with classes named `Strategy` and `ConcreteStrategy`, and a generation of engineers has copied the names instead of the idea. Your Strategy will be named `PaymentGateway` and `StripeGateway`, and it will still be Strategy. The shape matters, the names are yours.

## Interview Perspective

The question "what design patterns have you used" is a filter, and it filters out the reciters fast. The weak answer is a list: "Singleton, Factory, Observer, Builder, Decorator, all of them." The strong answer picks two or three and goes through the four parts for each. "I used Strategy for payment providers. The problem was the checkout had to support multiple gateways, and adding one should not touch the checkout. The solution was an interface and constructor injection, and the cost was a layer of indirection and more classes. I used it because the variation was real and runtime."

The follow-up that separates the practiced from the mechanical is "isn't a pattern just good code?" The weak answer says "no" and cannot explain why. The strong answer makes the distinction concrete. "A pattern is a named solution to a named problem with stated consequences. Good code solves a specific problem once. The pattern is the reusable shape, and the name is what lets a team reuse it without re-deriving it. Template Method is not good code that happens to be named, it is a documented trade, the subclass gives up control of the skeleton and gets the shared steps for free."

The sharper follow-up is "when would you not use a pattern." The strong answer states the guard. "When the problem it solves is not present. Strategy without variation is ceremony. The question is always whether the change actually happens at runtime, and if it does not, a method or a lambda is enough, and the pattern is overhead."

## Knowledge Check

1. A teammate says "let's apply the Strategy pattern to the validation logic." List the four parts you would need to pin down before agreeing, and state which one is most likely to reveal that the pattern is unnecessary.

2. Read the following sketch: a class named `SafeFileReader` wraps a `FileReader` and catches an `IOException` on every read, returning a default instead. Name the pattern, and state the problem it solves and the cost of the wrapper.

3. Your codebase has an interface with two implementations, injected through constructors, and nobody has ever named it. Explain, in two sentences, why naming it Strategy changes how the team can reason about it.

## Key Takeaways

- A pattern is four parts, name, problem, solution, and consequences, and the problem and the consequences are the parts that stop misuse.
- A pattern is not a library and not an algorithm; it is a shape you rebuild because your constraints differ.
- You already write patterns without the names, and the names are a vocabulary, not a curriculum.
- A pattern follows a problem, never precedes one, and the moment it becomes a goal it has cost you more than it returned.

## What's Next

The frame is set: patterns are named solutions with problems and consequences. The history is the part most guides skip, and it explains why the catalog looks the way it does. The next article covers where the twenty-three patterns actually came from, the Gang of Four book, what it drew on, and why a book from 1994 still names the shapes you draw today.

---

This article explains what a design pattern is, the four parts of name, problem, solution, and consequences, and how a pattern differs from a library or an algorithm. Its strongest claim is that patterns follow problems, never precede them, and that the name is what turns a re-invented shape into shared vocabulary.
