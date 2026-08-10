# Introduction to Event-Driven Architecture

## Learning Objectives

- State what an event is and what it is not, and the exact difference between an event and a command.
- Explain the two shifts event-driven architecture makes compared to a synchronous request-response call.
- Argue why event-driven is an availability and decoupling tool, not a speed shortcut.

## Introduction

For the last few chapters you designed systems where one service calls another for a thing and waits for the answer. Event-driven architecture turns that around. Instead of "give me the result," you say "something just happened" and move on. The producer expects no reply, and any number of consumers can react to the announcement whenever they choose. The whole model rests on one idea: decouple when a thing happens in the system from who needs to know about it.

## Problem Statement

A checkout method calls inventory, then shipping, then, added a year later, an email service. Each new dependency is another synchronous call bolted into a growing method and another place it can fail. One slow vendor call makes the whole checkout spin, and if email is down, checkout is down, because the platform refuses to send a receipt until the email service replied. The coupling is not in the data, it is in the call. The fix is to stop building the flow as a chain of blocking dependencies and instead publish what happened and let each downstream service decide its own timing.

## Core Concept

An event is a fact about the past: `OrderPlaced`, `PaymentCaptured`, `UserRegistered`. It has already happened, nothing can change it, and it is named with a noun and a past-tense verb. A command is the reverse: `PlaceOrder`, an instruction that asks for a change to state. The two are not interchangeable, and teams that blur them pay for it. When you publish `OrderPlaced` you are announcing that a fact now stands and anyone may react. When you send `PlaceOrder` you are demanding an effect and usually a reply.

| Kind | Words | What it means |
|------|-------|---------------|
| Command | Place, Create, Update, Delete | an instruction that expects an effect and usually a result |
| Event | Placed, Created, Updated, Deleted | a past fact that expects no reply |

The core shape is a publisher that emits and subscribers that react. The publisher never calls a subscriber. It puts the event on a channel, a broker, a topic, and whichever subscribers signed up for that event receive it. The publisher does not wait, does not receive a result, and does not crash if a subscriber is slow. That decoupling is the whole point, and the price is that the publisher no longer knows who, or whether anyone, handled the event.

```java
public class OrderService {
    private final EventPublisher publisher;

    public void placeOrder(Order order) {
        verifyStock(order);
        publisher.publish(new OrderPlaced(order.getId(), order.getItems()));
    }
}
```

The subscriber reacts in its own transaction, with its own failure handling:

```java
@Service
public class ReservationListener {
    @EventListener
    public void onOrderPlaced(OrderPlaced event) {
        reserveStock(event.getItems());
    }
}
```

Two ideas follow from that shape. The first is that the producer and the consumer live in different worlds of time: after the event is published, the subscriber's copy of the state is not updated yet. For a while it is stale. That lag is not a bug, it is the price of decoupling, and a later article in this chapter owns it under the name eventual consistency. The second is that an event stream is durable, so a consumer that joins later can replay what already happened instead of losing everything that occurred while it was asleep. Durable topics are what turn a mere notification bus into a system that can rebuild state, which is where event sourcing later lives.

A practical distinction drives a lot of the design: the event channel can be in-process and synchronous, like Spring's `ApplicationEventPublisher`, useful to wire one bean's outcome to another in the same JVM, or external and asynchronous, like a Kafka topic, useful when the subscribers are different services or must survive restarts. Choosing the wrong one for the job, using a shared in-process bus where the subscribers need durability, is one of the fastest ways to misjudge this model.

## Real Production Usage

Kafka and RabbitMQ are the concrete names, and here EDA stops being an abstraction. A payment platform publishes an event when money moves; a tax engine, a ledger, and a push-notification service each subscribe and produce their own copy. An e-commerce catalog stores product prices as events and each region renders its own view. The enduring gain is not that a single order is processed faster; it is that a brand new consumer joins by subscribing to a topic and no producer has to change a line. Once the producer is durable, the event stream is a source of history, which the event-sourcing article builds on intentionally.

The build needed before it: Kafka orders events per partition, and a service consuming them must keep that locale order when the events update one aggregate. Most teams that reach EDA want to scale readers and writers apart and isolate failure, and both come true. The honest debt is duplicated state, eventual views, and ordering rules that only show up under real load.

## Common Mistakes

1. **Publishing a command and calling it an event.** A consumer that must respond for the caller to continue is not processing an event, it is processing a request wearing a costume, and somewhere the code will wait for the answer.
2. **Assuming a subscriber ran.** An event guarantees no subscriber at all. Building a flow that must be handled, for example a refund that pauses if no one hears it, is the silent-loss bug in the making.
3. **Reading EDA as a speed up.** For one slow call with one interested party, a synchronous request is simpler and probably faster. The win is handling many reactions without coupling producers to consumers, not beating a single call.

## Interview Perspective

Interviewers ask "when would you go event-driven" to hear whether you weigh availability over latency. Weak: "it decouples things" and nothing else. Strong: "When a single action has several consumers that each want to react on their own schedule and the caller does not need a synchronous answer, I publish a fact, accept eventual consistency, and make the failure handling live on the subscriber side." They follow up with "what about ordering" and "can you get an actual request-response over a broker." Good answers say order holds per key or partition, not globally, and that a request-response over a broker is usually a bad translation of a synchronous call.

## Knowledge Check

- A CheckoutService used to call StockService and wait. You move it to publishing `OrderPlaced`. What specific failure becomes survivable now, and what new failure do you accept in exchange?
- A consumer that handles `OrderPlaced` never runs, and nothing seems wrong. Argue why that is and why it is dangerous.
- An event named `CreateOrder` arrives at a subscriber that returns a new order id. Classify it and explain the risk of modeling it as an event.

## Key Takeaways

- An event is a past-tense fact; a command is an instruction. The two need different handling and different trust.
- The publisher and the subscriber are decoupled in time and in knowledge; nobody waits on one for an event.
- Event-driven gives you availability and replay in exchange for eventual consistency, and that is a fair trade, not a free one.

## What's Next

That publish-and-react shape is the foundation, and the next article pulls the subscriber model out of the weeds. The publisher-subscriber model names the roles, the channel, and the two delivery modes, and turns the loose idea of "publish an event" into a concrete structure you can reason about.

---

This article explains event-driven architecture as trading synchronous calls for the publishing of past-tense facts that consumers react to on their own schedule. It argues that the model buys availability and a replayable record at the price of eventual consistency, and that a command dressed as an event is the most common first failure.