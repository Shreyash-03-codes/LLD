# Real-World Usage of Behavioral Patterns

## Learning Objectives

- Recognize behavioral patterns by their role in a running system, not by their class names.
- Read concrete systems, a request pipeline, a workflow engine, a message bus, and name the patterns behind them.
- See how the eleven behavioral patterns compose into one design instead of eleven separate tricks.

## Introduction

The last article is a map. Over this chapter we covered eleven behavioral patterns one at a time, and each reading, taken alone, can look like a box of disconnected tools. The point of this article is to connect them by reading real systems and finding several patterns working together in each.

The patterns we mapped are Strategy, Observer, Command, Chain of Responsibility, Template Method, State, Mediator, Iterator, Memento, Visitor, and the Strategy-vs-State distinction. None of them ships alone in production. They compose, and by the end of this article you should be able to look at a streaming request handler or a workflow engine and name three patterns behind it without opening a design document.

## Problem Statement

The problem is that a class seldom declares its pattern in its name. A filter that stops the request chain is rarely called `ChainFilter`, and a coordinator that routes events is rarely called `MediatorImpl`. Reading a diagram gives you a pattern in isolation; reading a running system gives you several patterns overlapping, and nothing labels them. A developer who learned patterns from their diagrams will search for a class named after the pattern, find none, and conclude the system uses no patterns at all. The real skill is reading a system by the role its objects play, and that skill is the whole point of this article.

## Core Concept

Behavioral patterns answer one question in different ways: who decides what happens next, and how is that decision kept out of the objects that carry it. When you read a system, ask where decisions are made, the order, the routing, the variation, and you will find a behavioral pattern at each decision point.

Four questions frame most of the patterns.

Who varies the algorithm? Strategy and Template Method. Who needs to know when state changed? Observer. What conditions the object is in? State. How does a request move through many handlers? Chain of Responsibility.

One system, a login gate, is worth a full walk because it exercises several at once.

## Real Production Usage

### A request pipeline: Chain of Responsibility, then Strategy, then Mediator

The most widespread production use of behavioral patterns is the request pipeline. An HTTP request arrives at a filter chain before it reaches the handler. Each filter can let it pass, reject it, or modify it, and the chain stops when a filter rejects it or the handler answers it.

```java
public interface Filter {
    void doFilter(Request request, FilterChain chain);
}

public class FilterChain {
    private final List<Filter> filters;
    private int index;

    public void doFilter(Request request) {
        if (index == filters.size()) {
            return;
        }
        Filter filter = filters.get(index++);
        filter.doFilter(request, this);
    }
}
```

That is Chain of Responsibility. The request advances through the filters in order, and any filter can stop the chain. The whole point is that each filter decides whether to pass the request on, and the decision is spread across the filters. This is the pattern, not a class named `Chain`.

The authentication filter routes by an algorithm that varies, so the `auth` step itself swaps a strategy:

```java
public class AuthFilter implements Filter {
    private final Authenticator authenticator;

    @Override
    public void doFilter(Request request, FilterChain chain) {
        Principal p = authenticator.authenticate(request);
        if (p == null) {
            return; // reject
        }
        chain.doFilter(request);
    }
}
```

Whether the gateway uses OAuth, a bearer token, or an API key is a Strategy behind the same `Authenticator` interface. The filter does not care which one is wired in. That is Strategy inside Chain of Responsibility.

### A workflow engine: State behind the object

A shipment order in a logistics system is a textbook State machine. The order lives in a small set of conditions, `Created`, `Paid`, `Assigned`, `Shipped`, and what operations are legal depends on which condition the order is in. You cannot ship an unassigned order, and you cannot cancel a shipped one.

The state field is behind an interface, and each state decides the next transition:

```java
public interface OrderState {
    void assignCourier();
    void ship();
    void cancel();
}
```

That is the State pattern doing real work, and it is not a coincidence. Long-lived domain objects with a life cycle are the honest home of State. The object's behavior genuinely follows its condition, and the pattern keeps the legality table out of the domain class.

The difference to re-read is Strategy. The shipment's `paymentMethod` strategy is a choice: charging the order the same way no matter the state. The order's `ship()` is a condition. Reading the two-classes-look-alike and knowing which question decides the difference is the Strategy-vs-State article, applied at runtime.

### The publisher: Observer and the Mediator

Two patterns show up in the notifications and event sides of the same system. A `PaymentEvents` subject publishes to subscribers: an email service, an audit, a fraud check. Each subscriber gets the event and handles it independently. That is Observer: a broadcast, everyone on the list.

Where the same event must reach exactly one handler in a controlled order, the mediator's hub appears. A checkout coordinator routes `form completed` to the total display and then to the submit handler, and it is a Mediator. The difference, a broadcast versus a directed handoff, is the print we read in the observer and mediator articles.

The two compose in one request. The payment subject broadcasts an event to its subscribers, and one subscriber is a coordinator that proceeds to route. Observer fans out, Mediator routes.

### Template Method and Iterator in the framework

The two quiet patterns run under almost everything. A framework's skeleton methods, the connection steps of a batch runner, the `init, process, cleanup` of a job, are a Template Method:

```java
public abstract class BatchRunner {
    public final void run() {
        connect();
        process();
        commit();
    }
    protected abstract void process();
}
```

The caller variants override `process()` and keep the rest. And every loop over the results is an Iterator, the cursor handing `hasNext()` and `next()` under the enhanced-for.

### A composition: Command for the work, Memento for the undo

The undo of a checkout, the order and the payment, composes Command and Memento. Each action is a command with `execute()` and `undo()`. Before executing, the command asks the order for a memento, pushes it on an undo stack, and the command's `undo()` restores the order from the most recent memento.

The stack is the Memento caretaker, storing opaque snapshots. The commands are the Command pattern carrying behavior. Neither knows the other's internals: the command seals a snapshot, the stack returns it, and the order alone unpacks it. Two patterns, one clean undo.

### The Visitor over a stable tree

The last hybrid is a Visitor over a stable tree. A pricing engine walks each product type with a `PricingVisitor`, a report engine walks the same tree with a `ReportVisitor`, and adding a new engine, a PDF export, adds a class without editing the products. The product hierarchy is stable, the export operations grow, which is exactly the contract the Visitor needs.

The tree itself is walked with an Iterator, and the work on each node is a Visitor. Iterator carries the cursor; Visitor carries the operation.

### The honest boundary

Every pattern has a boundary, and this chapter has stressed each one. The point of reading systems is to also see when a pattern stops paying. The strategy composes until the object has more than one behavior, and you reach for State. The Visitor pays while the type set is stable. The mediator pays while the coordination order is worth centralizing. The design that reads is the one whose patterns are matched to the set of change they expect.

No pattern can tell you which change to expect, and that is the honest limit. Production systems put these together when they have a clear, stable core of types and a set of orthogonal, growing variations. That is why the decision of this chapter, which pattern, is a question about the system's likely changes, asked before the code.

The composition matters more than the individual name. A design with one obvious pattern is almost always a sign of a missing second question. The filter chain appears to be a Chain of Responsibility, and the moment you ask how the auth step varies, you have added a Strategy. The object that changes its behavior by condition is a State, and the moment you ask who needs to know it changed, you have added an Observer. Reading a system well is reading the second question, the one the obvious pattern does not answer, because that is where the next pattern is hiding.

## Common Mistakes

The recurring mistake in production is reading the class name instead of the role. A class called `RequestPipeline` still implements a Chain of Responsibility, and a coordinator called `OrderFlow` is still a Mediator. If you search for the pattern's name, you guarantee you miss the pattern. The second mistake is forcing one pattern onto a system and calling the design done: naming a single pattern is the moment to ask the second question, who varies, who listens, who routes, because that is where the real design lives.

The third mistake is applying a pattern because you just read the chapter, not because the change asked for it. The strategy is only worth a class when the algorithm varies; the state machine is only worth a class when the conditions restrict. Reading systems well is the reverse of reading the pattern book: the book gives you tools, and the system tells you which tool fits the change it faces.

## Interview Perspective

The interview approach to this chapter is to name patterns by role, not by class name. When an interviewer walks a system and asks "which pattern is this filter chain?", the strong answer does not search for a class named `Chain`; it recognizes the ordered delegation and names the pattern.

A strong answer to "how do these work together?" picks a system, a login flow or a shipment, and walks two or three patterns at once: the chain passes the request, the strategy does the checking, the state gate keeps the account, the observer notified on an event. Being able to name several in one flow is the difference between learning the pattern box and applying it.

## Knowledge Check

1. An HTTP filter chain stops on the first rejection. Which pattern is it, and what does the `AuthFilter` layer add on top of it?
2. A logistics order rejects `ship()` until it is `Confirmed`, and the transition is decided inside the service. Which pattern is it, and how is that different from a delivery-fee strategy that sits on the order without restricting it?
3. Name three distinct patterns working together in one checkout flow, and state the role each one plays.

## Key Takeaways

- Name the patterns by role, not by class name: a chain that stops is `Chain of Responsibility` even if nothing is named chain.
- The pipeline composes a Chain of Responsibility with a Strategy at each step and a Mediator at the funnel.
- A lifecycle object is the State machine, and the strategy a condition is not a strategy for the same trouble.
- Observer broadcasts, Mediator routes, Iterator carries the cursor, Visitor runs over a stable tree.
- The deciding question, which pattern, is a question about the system's changes, not about the code.

## What's Next

The next chapter is Domain Modeling and Object Design. This chapter has been about how objects talk, and how decisions and behavior vary; the domain chapter is about what the object is at all. We move from the request cycle, the state machine, and the coordinator, and start designing the domain model, the entities, the boundaries, the relationships, and the invariants that give the objects meaning of their own. What changes from here is the target of the design: not the collaboration between objects, but the structure of the objects themselves and the rules they must uphold.

---

This article explains how behavioral patterns appear together in real systems, combining a Chain, Strategies, and a Mediator in one flow. It argues that production composition, not isolated diagrams, is the real test.