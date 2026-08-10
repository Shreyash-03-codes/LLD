# Sequence Diagrams

## Learning Objectives

1. Draw an interaction as lifelines and messages, with calls, returns, and activation bars in correct time order.
2. Distinguish a synchronous call from an asynchronous one on the page, and say why the distinction changes the design.
3. Use a sequence diagram to find the ordering bugs, the waits, and the races that prose descriptions of a flow cannot expose.

## Introduction

The class diagram shows the shape of the code. The sequence diagram shows the code doing something. It is a picture of one interaction between objects, laid out in time, read from top to bottom like a script: this message goes from this object to that object, this object waits, this object answers.
That time dimension is the whole value. A design can have a perfectly sensible class structure and a completely broken interaction order, and the class diagram will not tell you. The sequence diagram will, because it is the only diagram that forces you to say which call happens before which, who waits for whom, and where the flow blocks.

## Problem Statement

A checkout flow is being built. The team agrees on the structure: a controller, an order service, a payment gateway, an order repository. Everyone is satisfied with the class diagram, the arrows point the right way, the seams are clean. The first version of the service method looks like this.

```
public Receipt checkout(Order order) {
    Order saved = orderRepository.save(order);
    Authorization auth = paymentGateway.authorize(order.getTotal());
    return new Receipt(saved.getId(), auth);
}
```

It reads fine in prose. "We save the order, then charge the customer." But read as a sequence, it is a bug: the order is persisted before the payment is authorized. If the card is declined, there is now a saved order for a purchase that never happened, and someone has to reconcile it. If the payment is slow, the database write has already happened and the whole transaction is held open waiting on the network.
The class diagram could not catch this, because the class diagram has no time. Prose could not catch it either, because the sentence "we save then charge" was accepted without anyone asking which order. A sequence diagram would have forced the question in the drawing stage. The developer would have drawn the save message, then the authorize message, and the reviewer would have said "hold on, you persist before you charge," and the bug would have cost thirty seconds instead of a data cleanup.
That is the failure mode this article exists for: interaction order is design information, it lives in no other diagram, and it will be decided by whoever writes the code first unless someone draws it.

## Core Concept

A sequence diagram is built from a few elements, and it is worth naming them precisely because each one maps to a real property of the code.
The lifeline is the vertical line for one participant, an object or an actor, drawn dashed, running from the top of the diagram to the bottom. The participant is named in a box at the top. A lifeline represents the object over time, and its horizontal position never changes.
The message is the horizontal arrow between lifelines. A solid arrow with a filled arrowhead is a synchronous call: the caller sends the message and waits for the response. A solid arrow with an open arrowhead is an asynchronous message: the caller sends it and continues. A dashed arrow with an open arrowhead is the return, the response flowing back. The difference between the filled and open arrowhead is not decoration, it is whether the caller blocks.
The activation bar is the thin rectangle on a lifeline showing when that object is active, when it is executing or waiting on a synchronous call. An activation that extends down while the object waits on a child call is a visible wait. Read together, the activation bars show you where the system blocks.
The combined fragment is the box with a label in the top corner, `alt` for a conditional branch, `loop` for repetition, `opt` for an optional message. It wraps the messages that are inside the branch. The alt fragment is how a sequence diagram says "if the balance is sufficient, this happens, otherwise that happens."
The reading direction is top to bottom, and the vertical distance between messages carries meaning: it is proportional to time, which is why you draw a slow operation with a tall gap, a database round trip, a network call, and a fast one with a short gap.
Diagram: the checkout interaction as a sequence diagram, calls and returns in time order with activation bars.

```
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1000 600" font-family="Arial, Helvetica, sans-serif">
  <defs>
    <marker id="callArrow" markerWidth="10" markerHeight="8" refX="9" refY="4" orient="auto">
      <polygon points="0 0, 10 4, 0 8" fill="#57606a"/>
    </marker>
    <marker id="retArrow" markerWidth="10" markerHeight="8" refX="9" refY="4" orient="auto">
      <polygon points="0 0, 10 4, 0 8" fill="#ffffff" stroke="#57606a" stroke-width="1.5"/>
    </marker>
  </defs>
  <rect x="40" y="60" width="160" height="40" rx="4" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="120" y="84" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">Controller</text>
  <rect x="260" y="60" width="160" height="40" rx="4" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="340" y="84" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">OrderService</text>
  <rect x="480" y="60" width="160" height="40" rx="4" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="560" y="84" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">PaymentGateway</text>
  <rect x="720" y="60" width="160" height="40" rx="4" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="800" y="84" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">OrderRepository</text>
  <line x1="120" y1="100" x2="120" y2="520" stroke="#d0d7de" stroke-width="1" stroke-dasharray="4,4"/>
  <line x1="340" y1="100" x2="340" y2="520" stroke="#d0d7de" stroke-width="1" stroke-dasharray="4,4"/>
  <line x1="560" y1="100" x2="560" y2="520" stroke="#d0d7de" stroke-width="1" stroke-dasharray="4,4"/>
  <line x1="800" y1="100" x2="800" y2="520" stroke="#d0d7de" stroke-width="1" stroke-dasharray="4,4"/>
  <rect x="114" y="120" width="12" height="340" fill="#eaeef2" stroke="#57606a" stroke-width="1"/>
  <rect x="334" y="180" width="12" height="260" fill="#eaeef2" stroke="#57606a" stroke-width="1"/>
  <rect x="554" y="190" width="12" height="80" fill="#eaeef2" stroke="#57606a" stroke-width="1"/>
  <rect x="794" y="310" width="12" height="80" fill="#eaeef2" stroke="#57606a" stroke-width="1"/>
  <line x1="120" y1="140" x2="338" y2="140" stroke="#57606a" stroke-width="1.5" marker-end="url(#callArrow)"/>
  <text x="225" y="130" font-size="12" fill="#24292f" text-anchor="middle">checkout(order)</text>
  <line x1="340" y1="200" x2="558" y2="200" stroke="#57606a" stroke-width="1.5" marker-end="url(#callArrow)"/>
  <text x="445" y="190" font-size="12" fill="#24292f" text-anchor="middle">authorize(amount)</text>
  <line x1="558" y1="260" x2="342" y2="260" stroke="#57606a" stroke-width="1.5" stroke-dasharray="5,4" marker-end="url(#retArrow)"/>
  <text x="450" y="252" font-size="12" fill="#24292f" text-anchor="middle">authOk</text>
  <line x1="340" y1="320" x2="798" y2="320" stroke="#57606a" stroke-width="1.5" marker-end="url(#callArrow)"/>
  <text x="640" y="310" font-size="12" fill="#24292f" text-anchor="middle">save(order)</text>
  <line x1="798" y1="380" x2="342" y2="380" stroke="#57606a" stroke-width="1.5" stroke-dasharray="5,4" marker-end="url(#retArrow)"/>
  <text x="630" y="372" font-size="12" fill="#24292f" text-anchor="middle">id</text>
  <line x1="338" y1="440" x2="122" y2="440" stroke="#57606a" stroke-width="1.5" stroke-dasharray="5,4" marker-end="url(#retArrow)"/>
  <text x="235" y="432" font-size="12" fill="#24292f" text-anchor="middle">receipt</text>
</svg>
```

Read this diagram the way you would read the code that produced it. The controller calls checkout and waits, its activation bar stays open for the whole interaction. The order service receives the call, sends authorize to the gateway, and waits, its bar still open. The gateway answers and its activation closes. Only after the authorization returns does the service call save. The diagram makes the ordering of authorize before save visible in a way the sentence "we charge after we save" hid. Notice also what is not in the diagram: no call from the controller to the repository, which is the diagram saying the controller has no business touching persistence.
The thing a sequence diagram shows that no other tool shows is the async boundary. Draw a call with an open arrowhead and it says the caller fires and forgets, no activation wait, no return. A Kafka producer sending an event is an open arrowhead. A `CompletableFuture.runAsync(...)` hand-off is an open arrowhead. Getting that arrowhead wrong is how diagrams lie about whether the system blocks, and the lie propagates straight into the review.

```
Executor executor = Executors.newFixedThreadPool(4);
public void notifyOnOrderPlaced(Order order) {
    executor.submit(() -> auditService.record(order));  // async, fire and forget
    paymentGateway.capture();                            // sync, caller waits
}
```

That snippet is two different arrowheads. The `executor.submit` line is an open arrowhead, the caller continues without waiting for the audit. The `paymentGateway.capture()` line is a filled arrowhead, the caller blocks until the capture returns. A sequence diagram of the two lines would show a thin activation for the audit hand-off and a thick wait on the gateway. The picture and the code agree, which is the test of a good diagram.
The alt fragment is the other element that carries real design information. When a flow has a branch, draw the branch as an `alt` box instead of writing "then if this, we do that" in prose. The fragment shows the two paths side by side and makes it visible which messages are conditional. A reviewer can see, without reading a paragraph, that the refund call happens only on the declined path.

## Real Production Usage

Distributed tracing is a sequence diagram generated from reality. Jaeger and Zipkin render a trace as a waterfall of spans, one per service call, in time order, with the parent-child relationships drawn as connections. That waterfall is exactly the shape of a sequence diagram, produced automatically from the actual calls. When an engineer says "the payment is slow, look at the trace," they are asking the same question the sequence diagram answers: where is the time being spent, and who is waiting on whom.
The saga pattern for distributed transactions is documented almost exclusively with sequence diagrams, because a saga is an ordering constraint, a sequence of local transactions with compensating actions on failure. The order of the steps and the compensating step on each failure is pure sequence information, and teams draw it as a sequence diagram because prose cannot hold a multi-step failure path in anyone's head.
The Spring request lifecycle documentation uses sequence-style diagrams to show a request moving through the filter chain, the dispatcher, the handler mapping, and the controller. It is the same lesson as the checkout example: a framework's behavior is a sequence of calls, and the diagram is how the framework communicates the order without a wall of prose.

## Common Mistakes

The first mistake is drawing the calls and forgetting the returns. A sequence diagram with five arrows in and no dashed returns is a diagram that shows the requests but hides the waits, and the waits are where the design lives. The rule: every synchronous call gets a return, and the return is drawn only when the caller blocks on it. If the call is asynchronous, there is no return and no wait, and the open arrowhead says so.
The second mistake is drawing an async call as if it were sync. The Kafka producer that returns immediately gets a filled arrowhead and a tall activation, and the diagram now claims the producer waits for the broker when it does not. The lie changes the design conversation, because the reviewer will reason about latency and retries that do not exist. Draw what the code does, not what the design wishes it did.
The third mistake is a diagram with a message for every method call. A sequence diagram that shows `getX()`, `isValid()`, `toString()` and every other detail is unreadable, the same way a class diagram with every field is unreadable. The sequence diagram is for the interaction's critical path, the calls that carry the flow, not the plumbing. If a message does not change the ordering or the blocking, it does not belong in the diagram.

## Interview Perspective

The sequence diagram is how interviewers see if you understand time in a design. Given a design question, the strong candidate draws the class structure and then immediately draws the sequence for the critical path, the place the system is doing its real work. For a rate limiter, the sequence for the request that gets checked against the token bucket. For a booking system, the sequence for the seat being reserved and the transaction being committed. The interviewer can now ask "what happens on contention" and the candidate points at the part of the diagram where two requests arrive at the same resource.
The weak answer describes the flow in prose, "so the request comes in, and we check the cache, and if it's a miss we go to the database, and then we update the cache," a paragraph the interviewer must convert into time order by themselves. The strong answer draws the same flow as lifelines and messages, and the ordering, the cache check before the database, the cache update after, is visible without a sentence of explanation.
The follow-up that separates the thoughtful from the rehearsed is "is this call synchronous or asynchronous, and why." The weak answer says "synchronous" without a reason. The strong answer ties the arrowhead to the requirement. "The payment call has to be synchronous, the receipt depends on the result. The audit notification is asynchronous, the request does not wait for it, and a slow auditor must not slow down checkout." The interviewer is not checking arrowhead recall, they are checking whether the drawing encodes the blocking decisions.

## Knowledge Check

1. A service calls an external API and then writes to a local database, and the team cannot decide which first. State which sequence diagram element makes the difference visible, and draw the two message arrows that argue for each order.
2. You are shown a sequence diagram with a filled arrowhead from `OrderService` to `KafkaProducer` and a return arrow beneath it. Name the design error, and state what the corrected arrowheads and activations should look like.
3. A flow has a retry: if the gateway call fails, the service retries twice before giving up. State which combined fragment expresses the retry, and which part of the interaction goes inside the fragment.

## Key Takeaways

- The sequence diagram is the only diagram with time, and it is where ordering bugs get caught for free.
- Filled arrowhead means the caller waits, open means fire and forget, and getting them backwards lies about blocking.
- Every synchronous call gets a return, and the activation bar is the wait made visible.
- Draw the critical path of the interaction, not every method call, and the ordering decisions carry the diagram.

## What's Next

The sequence diagram shows one interaction between specific objects. The activity diagram zooms out to the flow itself, a decision tree of steps and branches that does not care which object executes each step. The next article covers the swim lanes, the decision diamonds, and the start and end nodes that turn a business flow into a picture anyone can follow.

---

This article explains the sequence diagram as the only design tool that carries time, with lifelines, messages, returns, and activation bars read top to bottom. Its strongest claim is that interaction order is design information no other diagram captures, and the arrowhead is the difference between blocking and not.
