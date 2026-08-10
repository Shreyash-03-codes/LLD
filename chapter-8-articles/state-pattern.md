# State Pattern

## Learning Objectives

- Recognize a state machine hiding in a class full of boolean and enum flags before touching any code.
- Build the State interface, where the context delegates each operation to the current state object and swaps states at runtime.
- Enforce legal transitions in one place, the state classes, and make illegal ones fail loudly instead of silently falling through a switch.

## Introduction

State lets an object change its behavior when its state changes, while the object stays the same to its callers. Today the order accepts a confirmation, tomorrow it does not, and the calling code does not change. The object looks like it changed class, which is the phrasing the GoF uses, and it is accurate in the way that matters: the caller invokes the same method and gets behavior that depends on where the object is in its life.

The mechanism is delegation plus swapping. The context object holds a reference to a state object and forwards every call to it. A call that is a legal transition returns the next state, and the context replaces its current state. The state interface decides both behavior and transitions.

## Problem Statement

The decay is a class where one boolean or enum changes what the methods do over time. Take an order with a status field and methods that all switch on it:

```java
public class Order {
    private OrderStatus status = OrderStatus.NEW;

    public void confirm() {
        if (status == OrderStatus.NEW) {
            status = OrderStatus.CONFIRMED;
        } else {
            throw new IllegalStateException("cannot confirm from " + status);
        }
    }

    public void cancel() {
        if (status == OrderStatus.NEW || status == OrderStatus.CONFIRMED) {
            status = OrderStatus.CANCELED;
        } else {
            throw new IllegalStateException("cannot cancel from " + status);
        }
    }

    public void ship() {
        if (status == OrderStatus.CONFIRMED) {
            status = OrderStatus.SHIPPED;
        } else {
            throw new IllegalStateException("cannot ship from " + status);
        }
    }
}
```

Every method re-states the same transition table. There are four methods, and the rule "only NEW and CONFIRMED can cancel" appears in `cancel()`, and the mirror image of it, "SHIPPED and DELIVERED cannot cancel," is implied everywhere else. The rule has no home. When a new state arrives, say PAID, every one of these methods grows a case, and the machine's truth, which transitions are legal, is still scattered so thinly that a missing case is one missed branch in one method.

The deeper failure is that the illegal transitions are enforced by the absence of a branch. A shipped order being delivered is correct in one method and silently nothing in another. The rules of the machine are not a thing you can read; they are a thing you have to reason about from the whole class.

## Core Concept

The State pattern gives each status a class, and the class implements only the transitions that are legal from that status. The context delegates and replaces its current state.

The interface names the operations and returns the next state:

```java
public interface OrderState {
    OrderState confirm(Order order);
    OrderState ship(Order order);
    OrderState deliver(Order order);
    OrderState cancel(Order order);
}
```

Each concrete state states its own rules. From NEW you can confirm or cancel, and everything else is loud:

```java
public class NewState implements OrderState {
    @Override
    public OrderState confirm(Order order) {
        return new ConfirmedState();
    }

    @Override
    public OrderState ship(Order order) {
        throw new IllegalStateException("must confirm before shipping");
    }

    @Override
    public OrderState deliver(Order order) {
        throw new IllegalStateException("not shipped yet");
    }

    @Override
    public OrderState cancel(Order order) {
        return new CancelledState();
    }
}
```

From SHIPPED, delivery is legal and cancel is not:

```java
public class ShippedState implements OrderState {
    @Override
    public OrderState confirm(Order order) {
        throw new IllegalStateException("already confirmed");
    }

    @Override
    public OrderState ship(Order order) {
        throw new IllegalStateException("already shipped");
    }

    @Override
    public OrderState deliver(Order order) {
        return new DeliveredState();
    }

    @Override
    public OrderState cancel(Order order) {
        throw new IllegalStateException("cannot cancel once shipped");
    }
}
```

The order context holds one state and delegates each call, replacing the state with the returned one:

```java
public class Order {
    private OrderState state = new NewState();

    public void confirm() {
        state = state.confirm(this);
    }

    public void ship() {
        state = state.ship(this);
    }

    public void cancel() {
        state = state.cancel(this);
    }
}
```

The transition table, the whole machine, now lives in the state classes. A call that is legal returns the successor; a call that is not throws. Adding a state, say a PAID state between confirm and ship, is one new class plus its edges, and not a single existing array of switch branches to touch. The rules are no longer scattered; they are the code itself.

Diagram: order state machine

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 700 500" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#444"/>
    </marker>
  </defs>

  <rect x="30" y="60" width="150" height="60" rx="14" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="105" y="86" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">NEW</text>
  <text x="105" y="106" text-anchor="middle" font-size="11" fill="#1a2733">start</text>

  <rect x="275" y="60" width="150" height="60" rx="14" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="350" y="86" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">CONFIRMED</text>
  <text x="350" y="106" text-anchor="middle" font-size="11" fill="#1a2733">await payment</text>

  <rect x="520" y="60" width="150" height="60" rx="14" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="595" y="86" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">SHIPPED</text>
  <text x="595" y="106" text-anchor="middle" font-size="11" fill="#1a2733">in transit</text>

  <rect x="275" y="370" width="150" height="60" rx="14" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="350" y="396" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">DELIVERED</text>
  <text x="350" y="416" text-anchor="middle" font-size="11" fill="#1a2733">done</text>

  <rect x="30" y="370" width="150" height="60" rx="14" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="105" y="396" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">CANCELED</text>
  <text x="105" y="416" text-anchor="middle" font-size="11" fill="#1a2733">terminal</text>

  <line x1="180" y1="110" x2="273" y2="110" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <text x="226" y="100" text-anchor="middle" font-size="11" fill="#5a6b7a">confirm</text>

  <line x1="425" y1="130" x2="518" y2="130" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <text x="471" y="120" text-anchor="middle" font-size="11" fill="#5a6b7a">ship</text>

  <line x1="105" y1="160" x2="105" y2="366" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <text x="75" y="270" text-anchor="middle" font-size="11" fill="#5a6b7a">cancel</text>

  <path d="M 350 122 L 350 250 L 50 250 L 50 366" fill="none" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <text x="150" y="268" text-anchor="middle" font-size="11" fill="#5a6b7a">cancel</text>

  <path d="M 595 122 L 595 300 L 350 300 L 350 366" fill="none" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <text x="500" y="288" text-anchor="middle" font-size="11" fill="#5a6b7a">deliver</text>
</svg>
```

The arrows are the legal transitions, each labeled by the method that takes it. DELIVERED and CANCELED have no outgoing edges, so they are terminal. The illegal transitions simply are not drawn, which is the same truth the `ShippedState.cancel()` throw expresses in code.

## Real Production Usage

The everyday Java object that acts on the state is the thread. Everyone who has written concurrency watches a `Thread` move through the states named by the `Thread.State` enum, NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, TERMINATED, and the methods that touch the thread, `start`, `join`, `interrupt`, behave differently according to the current state. That is a state machine, and the enum that ships with the JDK is exactly the class.

The entities a JPA runs through a lifecycle. A Hibernate entity is in a `transient` state when created, and moves to `managed` when persisted, and `detached` when the session closes, and `removed` when deleted; what operations the framework allows depends on the state. A connection pool is a set of states, inuse, idle, closed, and the get-and-release methods behave per state.

Several libraries build the shape explicitly. Spring Statemachine is an engine for defining the states, events, and transitions of a machine and guarding the transitions. A circuit breaker is a state machine, OPEN, CLOSED, HALF_OPEN, where the meaning of a request depends on the state its breaker is in. When you see an object whose methods throw because the object is in the wrong phase of its life, you are looking at a state machine, whether it is a pattern-named class or an enum full of branches.

## Common Mistakes

**Keeping the switch when the state is real.** A boolean that swaps behavior deserves the State treatment, but an enum that is purely data, just a label, does not. The question is whether the enum changes what the methods do. If it does, the switch is an implicit state machine and the State classes are honest about it.

**Leaking the context into the states.** The state classes hold a reference to the order and may need some context fields. That is normal; the state has to check a condition sometimes. The mistake is the reverse, when the order's business logic stays in the order and the state only routes. State is not a thin router. If the interesting decision is still in the context, the switch has not left, it has moved out of view.

**Failing to make illegal transitions loud.** If you write a default state that silently returns the same state instead of throwing, an illegal transition becomes a no-op, likely a missing payment just takes the call and returns and the caller has no idea. The pattern's whole value is that an illegal move is impossible because it throws in one unmistakable place. A silent illegal transition betrays it.

## Interview Perspective

State is where interviewers test whether you can see a machine under a surface that does not say "machine." A weak answer draws the state classes. A strong answer spots the enum, the boolean, the flag, counts how many methods switch on it, and says loudly "that is a state machine," before there draw. The skill being tested is recognition, not recall.

The follow-up that sorts people is the transition question: "Where does the rule 'you cannot cancel from SHIPPED' live?" The strong answer says it is scattered across every switch method in the old design, and in the State pattern it lives in exactly one class, `ShippedState`, and the failure happens where the state throws, one unmistakable, centralized place. The interviewer wants that word, loud and central, not a comment that says "you should not be here."

State and Strategy are the pair people blur, which is why this chapter gives them a dedicated comparison. In one line, Strategy picks an algorithm from a fixed menu, State is an object's current condition and it swaps behavior over the object's life. Same shape, opposite reason.

## Knowledge Check

1. Rebuild the switch order as states. Which state classes can `cancel()`? What does each legal call return, and what do the DELIVERED and CANCELED states do?
2. A caller calls `ship()` on an order in state NEW. Trace why it throws, where in the new design the "must confirm first" rule lives, and how that difference helps the tester.
3. Thread.State is a real machine. Pick one transition, say NEW to RUNNABLE, name the method that causes it, and say where that transition rule would live if `Thread` were modeled with State classes.

## Key Takeaways

- The State pattern uses a state interface where each concrete state owns what happens to each operation right now.
- Legal transitions return the next state, and illegal ones throw, in one place.
- A boolean or enum that swaps which behavior runs is the State pattern already, wearing a switch coat.
- State is not data labels; the enum has to change behavior, or the pattern is not earned.
- Thread, JPA entities, connection pools, circuit breakers, are state machines in production.

## What's Next

The next article is Mediator, which handles a different tangle. Where State solves how behavior changes over time inside one object, Mediator solves who can talk to whom across many objects. It is the coordinator that replaces a tangle of references, and like every focused pattern it fixes one mess. We will cover the hub-and-spoke shape, why Mediator breaks a tangle, and where it risks becoming the god object it was meant to prevent.

---

This article explains the State pattern as turning a growing switch into state classes that each own the legal behavior in the current state. It argues that illegal transitions are the whole value, and any object whose behavior changes over its life is this pattern.