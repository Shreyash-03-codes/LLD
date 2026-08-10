# State Diagrams

## Learning Objectives

1. Model one object's life as states, events, and transitions, with an initial state and the transitions that are and are not allowed.
2. Read a state diagram against Java code and catch the missing guard that lets an illegal transition through.
3. Decide when an object deserves a state machine and when a status field is enough, without over-modeling.

## Introduction

The state diagram is the one that looks at an object from the inside. The class diagram shows what an object is connected to. The sequence diagram shows one interaction it participates in. The state diagram shows the life of a single object: the states it can be in, the events that move it between them, and the transitions that are not allowed.
The reason it exists is that some objects change their behavior depending on where they are in their life. An order that is `CREATED` can be cancelled, and an order that is `SHIPPED` cannot. The state diagram is the only diagram that makes that kind of rule visible and checkable, and it is the diagram that catches the illegal transition before the bug ships.

## Problem Statement

An order entity has a status field. The schema starts simple: `CREATED`, `PAID`, `SHIPPED`, `DELIVERED`. Every place that touches an order checks the status with its own conditional, so the rules live scattered across the codebase.

```
if (order.getStatus() == OrderStatus.PAID) {
    refundService.refund(order);
}
```

One day a customer service agent needs to cancel an order that has already shipped. There is no rule saying that is impossible, because nobody ever wrote a transition table, so the code happily lets a shipped order be cancelled. The shipment goes out, the money comes back, and the investigation finds four different places that each decided independently what cancelling an order means. One of them allowed the illegal transition, and none of them could see the others.
That is the failure the state diagram exists to prevent. A status field with scattered conditionals is a state machine that nobody drew, which means nobody owns the rules, and the illegal transitions are decided one bug at a time. The moment the states and the allowed transitions are drawn as one picture, the same rules have an owner, and the reviewer can see the shipped-to-cancelled transition before it is shipped.

## Core Concept

The state diagram is a finite state machine drawn on a page, and its vocabulary is small. The state is a rounded rectangle, a condition an object can be in for a period of time. The initial state is the filled circle where the object's life starts. The final state is the bullseye, the filled circle in a ring, where the object's life ends. The transition is an arrow from one state to another, labeled with the event that fires it. The guard is a condition on a transition, written in brackets, and the transition fires only if the guard is true.
The rule that carries the design is the rule that some transitions simply do not exist. A state diagram is not a list of all possible moves, it is a statement of which moves are allowed. The absence of an arrow from `SHIPPED` to `CANCELLED` is a design decision as strong as the presence of an arrow from `CREATED` to `CANCELLED`. This is what a status field cannot express: the field stores the current state, but nothing in the schema says what the next state may be. The diagram is the schema for the transitions, and the transitions are where the rules live.
The Java for this is an enum and a transition method that enforces the diagram. The enum is the set of states. The method is one transition, and its guard is the check that makes the illegal transition throw.

```
public enum OrderState {
    CREATED, PAID, SHIPPED, DELIVERED, CANCELLED, REFUNDED
}
public class Order {
    private OrderState state;
    public void cancel() {
        if (state == OrderState.CREATED) {
            state = OrderState.CANCELLED;
        } else {
            throw new IllegalStateException("cannot cancel an order in state " + state);
        }
    }
}
```

That `if` is the guard from the diagram, made concrete. The diagram says there is one arrow out of `CREATED` named `cancel`, and the code says the same thing with a check. A team that draws the diagram first writes the method to match the picture, and a team that reads the picture later can verify the method against it. The two artifacts, diagram and code, are two views of the same rules, and when they disagree, one of them is the bug.
Diagram: the order lifecycle as a state diagram, with the events that move an order between states and the terminal states.

```
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 860 450" font-family="Arial, Helvetica, sans-serif">
  <defs>
    <marker id="trans" markerWidth="10" markerHeight="8" refX="9" refY="4" orient="auto">
      <polygon points="0 0, 10 4, 0 8" fill="#57606a"/>
    </marker>
  </defs>
  <circle cx="58" cy="56" r="12" fill="#24292f"/>
  <line x1="70" y1="56" x2="108" y2="56" stroke="#57606a" stroke-width="1.5" marker-end="url(#trans)"/>
  <rect x="110" y="28" width="140" height="56" rx="12" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="180" y="60" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">CREATED</text>
  <rect x="380" y="28" width="140" height="56" rx="12" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="450" y="60" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">PAID</text>
  <rect x="650" y="28" width="140" height="56" rx="12" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="720" y="60" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">SHIPPED</text>
  <line x1="250" y1="56" x2="378" y2="56" stroke="#57606a" stroke-width="1.5" marker-end="url(#trans)"/>
  <text x="314" y="48" font-size="12" fill="#57606a" text-anchor="middle">pay</text>
  <line x1="520" y1="56" x2="648" y2="56" stroke="#57606a" stroke-width="1.5" marker-end="url(#trans)"/>
  <text x="584" y="48" font-size="12" fill="#57606a" text-anchor="middle">ship</text>
  <line x1="180" y1="84" x2="180" y2="256" stroke="#57606a" stroke-width="1.5" marker-end="url(#trans)"/>
  <text x="192" y="170" font-size="12" fill="#57606a">cancel</text>
  <line x1="450" y1="84" x2="450" y2="256" stroke="#57606a" stroke-width="1.5" marker-end="url(#trans)"/>
  <text x="462" y="170" font-size="12" fill="#57606a">refund</text>
  <line x1="720" y1="84" x2="720" y2="256" stroke="#57606a" stroke-width="1.5" marker-end="url(#trans)"/>
  <text x="732" y="170" font-size="12" fill="#57606a">deliver</text>
  <rect x="110" y="258" width="140" height="56" rx="12" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="180" y="290" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">CANCELLED</text>
  <rect x="380" y="258" width="140" height="56" rx="12" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="450" y="290" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">REFUNDED</text>
  <rect x="650" y="258" width="140" height="56" rx="12" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="720" y="290" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">DELIVERED</text>
  <line x1="180" y1="314" x2="180" y2="402" stroke="#57606a" stroke-width="1.5" marker-end="url(#trans)"/>
  <circle cx="180" cy="416" r="12" fill="#ffffff" stroke="#24292f" stroke-width="2.5"/>
  <circle cx="180" cy="416" r="4" fill="#24292f"/>
  <line x1="450" y1="314" x2="450" y2="402" stroke="#57606a" stroke-width="1.5" marker-end="url(#trans)"/>
  <circle cx="450" cy="416" r="12" fill="#ffffff" stroke="#24292f" stroke-width="2.5"/>
  <circle cx="450" cy="416" r="4" fill="#24292f"/>
  <line x1="720" y1="314" x2="720" y2="402" stroke="#57606a" stroke-width="1.5" marker-end="url(#trans)"/>
  <circle cx="720" cy="416" r="12" fill="#ffffff" stroke="#24292f" stroke-width="2.5"/>
  <circle cx="720" cy="416" r="4" fill="#24292f"/>
</svg>
```

Reading the diagram is reading the rules. The order starts at the initial node and enters `CREATED`. From `CREATED`, exactly two arrows leave: `pay` to `PAID` and `cancel` to `CANCELLED`. From `PAID`, two arrows: `ship` to `SHIPPED` and `refund` to `REFUNDED`. From `SHIPPED`, exactly one: `deliver` to `DELIVERED`. The three states with a bullseye beneath them, `CANCELLED`, `REFUNDED`, and `DELIVERED`, are terminal. The absence of arrows is the substance: there is no `cancel` out of `PAID`, no `refund` out of `SHIPPED`, no arrow from `DELIVERED` anywhere. Those are the rules the status field could not state.
The state diagram is not the same tool as the activity diagram, and the confusion is worth killing early. The activity diagram is a flow that moves through actions and decisions, and it does not care which object it is inside. The state diagram is one object's possible conditions, and it does not show a flow through the system. An order going through checkout is a flow, an activity diagram. An order being in `PAID` and being allowed to move to `SHIPPED` but not to `CANCELLED` is a set of states, a state diagram. The question that picks between them: is the interesting thing the sequence of steps, or the conditions an object is allowed to be in?
Guards are the detail that makes the state machine honest. A transition that says "refund from `PAID`" without a guard means any paid order can be refunded forever. The guard, the condition in brackets, is what makes the transition conditional, "`refund` only when the payment was settled." When an interviewer pushes on a state diagram, it is usually at the guards, because the guards are where the business rules actually live. The states are the skeleton; the guards are the policy.

## Real Production Usage

The state machine shows up in production in the places where an object's legality depends on its history, and Java gives you one famous example for free: the `Thread` lifecycle. `java.lang.Thread.State` is `NEW`, `RUNNABLE`, `BLOCKED`, `WAITING`, `TIMED_WAITING`, and `TERMINATED`, and the JLS defines which transitions are legal. A thread cannot go from `NEW` to `TERMINATED` without running, and it cannot leave `WAITING` except by being notified. Every Java developer has been bitten by an illegal thread state, which is the `IllegalStateException` that fires when code calls `start()` twice. That is a state machine, drawn by the platform, enforced by the runtime.
Spring StateMachine is the framework that turns this into a first-class modeling tool. You define the states, the transitions, and the guards as configuration, and the framework enforces them at runtime, refusing an event that has no valid transition from the current state. Payment flows and order workflows are the classic use, and the pattern is exactly this article's diagram made into a runnable artifact: the picture and the code are the same thing.
The idempotent consumer in Kafka messaging is a state machine by another name. A consumer tracks the state of its processing, `initializing`, `active`, `rebalancing`, `recovering`, and the transition rules prevent it from processing during a rebalance. The state diagram is how that protocol is specified, because the legality of an action, "consume now," depends entirely on which state the consumer is in.

## Common Mistakes

The first mistake is drawing actions as states. A state named `SendEmail` is not a state, it is a transition action, and the diagram is confusing what the object is with what the object does. The test: can the object pause in this box? If the answer is no, if it passes through, it is not a state, it is part of a transition. The states in this article's diagram are all places an order can sit and wait: `PAID` waits for the shipment. There is no `ChargingCard` state, because charging is an event, not a resting place.
The second mistake is a diagram with no initial state, or worse, transitions into and out of every state from everywhere. A state machine where every arrow is drawn in both directions is a diagram that says anything can happen, which means the diagram drew nothing. The value of the state machine is the restrictions, and a diagram with no restrictions has no value.
The third mistake is over-modeling. Not every class with a field deserves a state machine. A `Customer` with a status that flips once, from `ACTIVE` to `SUSPENDED`, is a status field, and modeling it as a full state machine is ceremony. The state machine earns its keep when there are multiple transitions, real guards, and illegal moves that the code must refuse. If the diagram has two states and one arrow, the field would have done.

## Interview Perspective

The vending machine question is the classic state diagram interview, and it is a state machine in disguise. The interviewer wants to see states, `IDLE`, `COIN_INSERTED`, `DISPENSING`, the events that move between them, `insertCoin`, `selectItem`, and the guards, the item must be in stock. The candidate who draws the states and the arrows has answered the question structurally, and the interviewer can now ask about the edge cases against the picture.
The weak answer lists the statuses, "so it can be idle, or it has a coin, or it's dispensing," a list with no transitions and no rules. The strong answer draws the initial state, the states with the allowed moves, and the guard on each transition, and the interviewer can ask "what happens if you insert a coin and then another coin" and point at the diagram for the answer, a self-transition that returns to `COIN_INSERTED`.
The follow-up that tests depth is "what happens if a customer cancels an order that is already paid." The weak answer says "we refund it." The strong answer draws the transition. "From `PAID`, there is a `refund` transition to `REFUNDED`, and from `CREATED` a `cancel` transition to `CANCELLED`. There is no transition out of `SHIPPED` for a cancel, so the agent cannot do it, and the code throws. The diagram is where that rule lives, and the `IllegalStateException` is where the code enforces it."

## Knowledge Check

1. An order can be refunded from `SHIPPED` but the current code has no rule against it. State whether the design needs a state diagram or an activity diagram, and name the specific element, the missing arrow or the missing guard, that represents the defect.
2. A teammate proposes a state named `SendingConfirmationEmail` on an order's state machine. Assess the proposal: is it a state or an action, and where does the work of sending the email actually belong?
3. Given the states `NEW`, `RUNNABLE`, `WAITING`, `TERMINATED`, draw the three legal transitions you are sure about, and state which transition is illegal, the one `start()` on an already started thread triggers, and which Java type represents it.

## Key Takeaways

- The state diagram is one object's possible conditions and the moves between them, and the missing arrows are the rules.
- A status field is a state machine nobody drew, and the scattered conditionals are the illegal transitions arriving one bug at a time.
- Guards are the policy, the condition that makes a transition legal only when it should be, and states are only the skeleton.
- An action is not a state, an initial state is mandatory, and a two-state diagram was a field all along.

## What's Next

The state diagram looked at one object's life. The component and object diagrams zoom back out, the component diagram to the module boundaries and their interfaces, the object diagram to a frozen moment of concrete instances. The next article covers the two, the boxes with ports and the snapshots, and why they are the diagrams you draw when the system is bigger than a class.

---

This article explains the state diagram as one object's possible conditions and the legal moves between them, with states, events, transitions, and guards. Its strongest claim is that a status field is a state machine nobody drew, and the absent arrows are where the design rules live.
