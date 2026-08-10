# Strategy vs State: Key Differences

## Learning Objectives

- Tell Strategy from State by one question: does the object know which behavior it is in, or does it only know which behavior to call?
- Recognize that both patterns swap a field behind an interface, and that the difference is the object's intent, not the shape of the code.
- Decide, for a real scenario, whether to model changing behavior with a Strategy, a State machine, or neither.

## Introduction

Strategy and State are the two behavioral patterns that get confused most often, and the confusion is fair. Both put a behavior behind an interface, both swap that behavior at runtime by changing a field, and both look like the same diagram, a context holding an interface with a few concrete implementations.

The difference is not in the code. The difference is in what the object thinks it is doing. A Strategy is a choice: the object decides which algorithm to use and can use it whenever it wants. A State is a situation: the object is in a condition, and the condition decides which behaviors are legal.

## Problem Statement

The problem this article exists to solve is that the two patterns are often indistinguishable by their shape. A team that learns Strategy and State from their diagrams alone will, in a design review, call an order life cycle a Strategy and a checkout fee a State, and get both backwards. The shape of the code teaches nothing, because the shape of a Strategy and a State is identical. What tells them apart is intent, and intent is the thing the code does not show. So the problem is precisely: look at a real object with a swappable field and decide whether the object is choosing a behavior or living a condition.

## Core Concept

### The two patterns in their own words

Strategy takes an algorithm and makes it replaceable. The context holds one strategy and delegates to it:

```java
public class CheckoutService {
    private final DeliveryFeeStrategy deliveryFee;

    public double totalWith(Order order) {
        return order.total() + deliveryFee.calculate(order);
    }
}
```

The caller picks the strategy when it builds the service, and the service never needs to know which one it holds. The strategy does not change the rest of the behavior; it changes one decision, how much to charge for delivery.

State takes a situation and makes its behavior follow that situation. The context holds the current state, and what is legal depends on which state it is in:

```java
public class Order {
    private OrderState state = new NewOrder(this);

    public void confirm() { state.confirm(); }
    public void ship()    { state.ship(); }
    public void cancel()  { state.cancel(); }
}
```

`OrderState` has implementations for `NewOrder`, `Confirmed`, `Shipped`, and each one decides what `ship()` and `cancel()` mean, or whether they mean anything at all. A `NewOrder` can cancel; a `Shipped` order cannot.

The code shapes are nearly identical. Both have a context, a field of interface type, and several implementations. If you screenshot the two diagrams, you cannot tell them apart.

### The deciding question

The difference is one question: does the object know which state it is in, and does the current state restrict what the object can do?

A Strategy does not know. The checkout service holds a delivery fee strategy and calls it, and nothing about the strategy changes the legal operations of the service. The service can charge, can refund, can discount, all regardless of which delivery strategy is plugged in. The strategy is a knob on one decision.

A State does know. The order is `NewOrder`, `Confirmed`, `Shipped`, or `Canceled`, and that knowledge restricts everything. A `Shipped` order does not offer `confirm()` as a legal action. The state is not a knob on one decision; it is the current condition of the whole object, and the behavior follows it.

That is the deciding question: is the field a choice the object uses, or is it a condition the object is in?

### The same shape, different rules

Because the shape is the same, the honest way to talk about these two is to say what each pattern changes when the field changes.

Strategy changes the outcome. The delivery fee goes from `0.00` to `12.00` when the caller plugs in the international strategy. The rest of the checkout, the total, the tax, the confirmation, behaves identically. The swap is a preference.

State changes the behavior. A `Shipped` order receiving a `cancel()` either throws or does nothing, where a `NewOrder` receiving the same `cancel()` cancels and emits a refund event. Same method name, same object, different result, because the object is in a different condition. The swap is a transition.

The other practical rule is about who triggers the change. In Strategy, the change comes from outside. The caller decides which strategy to construct and pass. In State, the change often comes from inside. The state object itself decides that an action moves it to the next state, `ship()` on a `Confirmed` order returns a `Shipped` state, and the state machine advances itself.

### Two concrete reads

Take the delivery fee from the first example and push it one step. The checkout holds a strategy, and the strategy has no idea what the checkout is doing. The international strategy cannot refuse to be used on a domestic order, because it has no authority and no knowledge of context. It charges a number. That is Strategy: the behavior is a pure function the object delegates to.

Take the order state and push it one step. The `Shipped` state does know context. It knows an order that shipped has a carrier and a tracking number, and it knows that `confirm()` is meaningless for it. When the order receives `ship()`, the `Confirmed` state checks the address, marks the order, and returns a `Shipped` state, so the next call to `cancel()` is answered by the shipped behavior. That is State: the behavior is a consequence of where the object is.

The names in the code are the giveaway too. Strategy implementations are usually named by the algorithm, `UsDeliveryFee`, `EuDeliveryFee`, `OvernightShipping`. State implementations are usually named by the condition, `NewOrder`, `Confirmed`, `Shipped`. When a class's name describes a stage of a life cycle, it is almost certainly a State.

### The patterns compose

The two patterns are not rivals; they compose. A state machine can carry a strategy inside it. The `Shipped` state can hold a shipping-carrier strategy and use it to pick the courier. The state decides which behavior is legal, and the strategy decides how that behavior is performed. The deciding question separates those, and nothing stops you from using both in one object.

The confusion also resolves when you look at the object's verbs. An object whose verbs mean different things based on its condition is State, the order `cancel()` that behaves differently at `Shipped` is State. An object whose verbs mean the same thing but are performed by a swappable collaborator is Strategy, the checkout that always totals the order but picks a delivery rule is Strategy.

## Real Production Usage

The two patterns land in different parts of a backend. A service holds a strategy as a behavior it performs, a payment interface, a delivery rule, a tax calculator, and it delegates to whichever implementation is wired in. That is Strategy, and it is the more common of the two. Its honest limit is that the class does not restrict its own behavior, so a strategy can be switched without the object noticing anything changed.

A long-lived domain object, an order, an invoice, a job, a subscriber, carries its current condition and the condition rules what is allowed. That is State. Its requirement is that the conditions genuinely restrict, a shipped order cannot cancel, and when the object has only two states, a flag beats a machine. The production giveaway is the object's verbs: if `cancel()`'s meaning depends on a condition, it is State; if `pay()` always means the same act and the fee grows by a choice, it is Strategy.

## Common Mistakes

The mistake here is not in the code, because the code of the two patterns looks the same. The mistakes are in the naming and the intent, so they show up in design conversation as much as in code.

**Calling every field swap a State.** The most common mistake is to name a Strategy a State because both have a field and an interface. The test is the legal operations. If swapping the field does not change what the object may do, it is Strategy.

**Forcing State where a boolean is enough.** A two-condition object, `active` and `inactive`, does not need a state machine. An `isActive` flag with an `if` is simpler and clearer. State earns its classes when the conditions are several and the transitions are rules.

**Forgetting that State restricts.** A state machine whose every state implements every method the same way is a Strategy wearing a costume. The point of State is that some operations are illegal or different in some states. If nothing is restricted, the pattern is doing nothing.

## Interview Perspective

The interview section is the whole pattern, because the interview question is the whole problem. When an interviewer asks "Strategy vs State," they want the deciding question, not the code shapes.

A strong answer structure is short. First, name the shared shape, both swap a behavior behind an interface. Second, name the difference, Strategy is a choice and State is a condition, and the condition restricts the legal operations. Third, give the trigger, strategy swapped from outside, state transitioned from inside. Fourth, give one concrete example for each, a delivery fee strategy and a shipping state.

The traps are in the details. Do not say "State is like Strategy with more classes," which confuses the two. Do not say "State is about undo," which confuses it with Memento. The deciding question, condition versus choice, is the whole answer.

## Knowledge Check

1. A checkout service holds a `PaymentMethod` interface and calls `pay()` on it. Is that Strategy or State? Which part of the deciding question tells you?
2. An order's `cancel()` throws on a shipped order but succeeds on a new order. Is that Strategy or State? Why?
3. The same class now holds a shipping-carrier strategy inside its `Shipped` state. Are the two patterns in conflict? What does each one decide?

## Key Takeaways

- The shared shape, a context, a field, an interface, concrete classes, is why the two get confused.
- The deciding question: Strategy is a choice the object uses; State is a condition the object is in.
- The state restricts the legal operations; the strategy tunes one operation.
- The trigger: strategy is swapped from outside, state transitions from inside.
- The two compose: a state machine can hold a strategy for how it performs a legal behavior.

## What's Next

The next article closes the chapter with a map. We have covered the eleven behavioral patterns one at a time, and the final article looks at how they show up together in real production systems: which patterns share a skeleton, which ones combine in a pipeline, and which ones you are likely to find already running under the frameworks you use. We will read a few concrete systems and identify the patterns by their role.

---

This article explains the difference between Strategy and State by one deciding question: a choice versus a condition. It argues that the state restricts legal operations while the strategy tunes a single decision, and that the two share the same shape.