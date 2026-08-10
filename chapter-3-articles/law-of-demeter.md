# Law of Demeter

## Learning Objectives

1. State the Law of Demeter as "talk only to your immediate friends," and name who the friends are.
2. Recognize the train wreck, the chain of calls through other objects' return values, as the violation.
3. Distinguish the real violation from the harmless dot chains, and explain why the law is really about encapsulation.

## Introduction

The Law of Demeter is the most misquoted rule in design, which is saying something. The formal version: a method should only call methods on itself, on its parameters, on its own fields, and on objects it creates. Anything else, an object it got by calling another object's method, is a stranger, and calling into a stranger is a violation.

The famous informal version is "don't talk to strangers," and the equally famous misinterpretation is "don't chain more than one dot." Both are right about the symptom and wrong about the disease. The disease is that a method knows the shape of the object graph it is standing in, and the dots are the visible mark of that knowledge.

## Problem Statement

A shipping quote needs the customer's city, and the code gets it the direct way.

```
String city = order.getCustomer().getAddress().getCity();
```

It works, and it is a train wreck. The quote now depends on the fact that an order has a customer, that a customer has an address, and that an address has a city. If any of those three facts changes, if the address moves to a separate table, if the customer can have multiple addresses, the quote breaks, and the break is in a class that has nothing to do with addresses.

Worse, the knowledge is invisible. The quote's real dependency is "what city does this order ship to," and the code expresses it as a walk through three objects. The class that owns the address, the `Customer`, never gets asked, its internals are reached around it, and it cannot change its internals without breaking every walker. The city is a concern of the customer, and the quote is reaching through the customer to get at it.

## Core Concept

The law is precise about who the friends are. A method may call methods on:

- itself, `this` and its own fields,
- its parameters,
- objects it creates locally,
- objects it holds in fields.

It may not call methods on objects that arrive through those friends, the return value of another object's method, the element pulled from a list, the result of a getter. The stranger is not a stranger because it is far away. It is a stranger because the method did not invite it, it found it by reaching through something else.

Diagram: what the Law of Demeter allows, a method may call its friends but not reach through them to strangers.

```
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 820 500" font-family="Arial, Helvetica, sans-serif">
  <defs>
    <marker id="friend" markerWidth="10" markerHeight="8" refX="9" refY="4" orient="auto">
      <polygon points="0 0, 10 4, 0 8" fill="#1a7f37"/>
    </marker>
    <marker id="stranger" markerWidth="10" markerHeight="8" refX="9" refY="4" orient="auto">
      <polygon points="0 0, 10 4, 0 8" fill="#d1242f"/>
    </marker>
  </defs>

  <rect x="350" y="215" width="120" height="64" rx="6" fill="#f6f8fa" stroke="#57606a" stroke-width="2"/>
  <text x="410" y="254" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">a</text>

  <rect x="70" y="60" width="130" height="64" rx="6" fill="#e6f6ec" stroke="#1a7f37" stroke-width="2"/>
  <text x="135" y="92" font-size="13" font-weight="bold" fill="#033d16" text-anchor="middle">b</text>
  <text x="135" y="110" font-size="11" fill="#166534" text-anchor="middle">field</text>
  <rect x="70" y="215" width="130" height="64" rx="6" fill="#e6f6ec" stroke="#1a7f37" stroke-width="2"/>
  <text x="135" y="247" font-size="13" font-weight="bold" fill="#033d16" text-anchor="middle">c</text>
  <text x="135" y="265" font-size="11" fill="#166534" text-anchor="middle">parameter</text>
  <rect x="70" y="370" width="130" height="64" rx="6" fill="#e6f6ec" stroke="#1a7f37" stroke-width="2"/>
  <text x="135" y="402" font-size="13" font-weight="bold" fill="#033d16" text-anchor="middle">d</text>
  <text x="135" y="420" font-size="11" fill="#166534" text-anchor="middle">new</text>

  <rect x="620" y="60" width="130" height="64" rx="6" fill="#ffebe9" stroke="#d1242f" stroke-width="2"/>
  <text x="685" y="92" font-size="13" font-weight="bold" fill="#82071e" text-anchor="middle">e</text>
  <text x="685" y="110" font-size="11" fill="#cf222e" text-anchor="middle">b.getE()</text>
  <rect x="620" y="370" width="130" height="64" rx="6" fill="#ffebe9" stroke="#d1242f" stroke-width="2"/>
  <text x="685" y="402" font-size="13" font-weight="bold" fill="#82071e" text-anchor="middle">f</text>
  <text x="685" y="420" font-size="11" fill="#cf222e" text-anchor="middle">c.getF()</text>

  <line x1="350" y1="232" x2="205" y2="100" stroke="#1a7f37" stroke-width="2" marker-end="url(#friend)"/>
  <line x1="350" y1="247" x2="205" y2="247" stroke="#1a7f37" stroke-width="2" marker-end="url(#friend)"/>
  <line x1="350" y1="262" x2="205" y2="394" stroke="#1a7f37" stroke-width="2" marker-end="url(#friend)"/>
  <line x1="470" y1="215" x2="685" y2="124" stroke="#d1242f" stroke-width="2" stroke-dasharray="6,5" marker-end="url(#stranger)"/>
  <line x1="470" y1="279" x2="685" y2="370" stroke="#d1242f" stroke-width="2" stroke-dasharray="6,5" marker-end="url(#stranger)"/>

  <text x="205" y="470" font-size="12" fill="#1a7f37">Solid green: friend, allowed</text>
  <text x="540" y="470" font-size="12" fill="#d1242f">Dashed red: stranger, forbidden</text>
</svg>
```

The train wreck is the pure violation. `order.getCustomer().getAddress().getCity()` calls methods on the return values of the previous calls. The quote invited the order and went on to talk to people the order introduced. The fix is to ask the friend to do the walking:

```
String city = order.getShippingCity();
```

Now the quote depends on one thing, the order's ability to know its shipping city. The walk moved into the `Order`, where the knowledge belongs, and the `Order` can change its internals, the address, the customer, the table layout, without the quote noticing. This is the move that makes the law worth obeying: the violation is always a knowledge leak, and the fix is always moving the knowledge back behind a friend's method.

The "tell, don't ask" rule is the positive form of the same idea. Instead of asking an object for its data and then doing something with it, tell the object to do the thing. `if (order.getCustomer().getAddress() != null) { ... }` becomes `if (order.hasShippingAddress()) { ... }`. The asking forces the caller to know the object's internals. The telling moves the decision into the object, which is where it belongs.

The law is not a style rule about dots, and the distinction is the part that keeps it usable. Fluent builders, `builder.withName("x").withAge(30).build()`, chain dots and violate nothing, because each call returns the same object, `this`, and the chained calls are calls on the same friend. Streams, `list.stream().filter(...).map(...).toList()`, chain dots through new objects the caller created, not through strangers it discovered. The violation is not the chain, it is reaching through someone else's return value into their internals. `order.getCustomer()` returns a stranger, and calling `.getAddress()` on it is the law breaking, regardless of the number of dots.

Why the law exists is more important than the rule, because the rule is a proxy for it. The law protects encapsulation at the object-graph level. A class that walks the graph knows the graph's shape, and knowledge of shape is coupling. The walker is coupled to every class in the chain, and it breaks when any link changes. The class that asks its friend has one dependency, the friend, and the friend owns its own graph. The law is the encapsulation principle from an earlier chapter, applied across several objects instead of one.

The cost of the law, and it has one, is pass-through methods. When the quote calls `order.getShippingCity()`, the order has to implement a method that calls `customer.getAddress().getCity()` internally, or delegate further. The concern is that the law produces a chain of delegating methods that add no behavior. The counter is that the delegation is the point: each object hides its internals from the next, and the knowledge stays where it belongs. A pass-through method that preserves the hiding is not ceremony, it is the law working. The alternative, walking through, is the coupling the law exists to prevent.

There is a test for when the law is actually being broken, and it is better than counting dots: can this method survive if the object graph changes shape? If the customer's address moves, does the quote change? If the answer is yes, the quote knew too much, and the knowledge should have been behind a friend's method.

## Real Production Usage

The train wreck is the everyday production shape, and the repair is the story of a real refactor. A reporting service that reached `payment.getUser().getPlan().getTier()` for every report broke when plans were restructured. The fix moved a `getTier()` method onto the payment or the user, and the reporting service stopped knowing the plan structure. The blast radius of the plan change shrank from every report to one class.

The "god service" is the same violation at scale. A service that orchestrates a dozen objects by reaching into each one's internals, `entity.getRepo().getStatus()`, `config.getCache().getTtl()`, has a dependency on every shape it touches. It is untestable, because testing it requires building the whole graph, and it is fragile, because any link in the chain moves. The law is the first rule that drags the service back to talking to its friends.

The pattern that institutionalizes the law is the domain method that hides a walk. `order.hasFreeShipping()`, `account.isOverdrawn()`, `trip.isRefundable()`, each one buries a graph walk behind a friend's method, and each one is the Law of Demeter expressed as design. The method names are the "tell" form: the caller states the question, the object does the walking.

## Common Mistakes

The most common mistake is counting dots and flagging every chain. The builder and the stream chain dots through the same object or through objects the caller created, and neither violates the law. The violation is reaching through a stranger's return value, and the mechanical count cannot see the difference. The dot rule is a heuristic that works only if you know what it is a proxy for.

The second mistake is the opposite: the pass-through method with no hiding. A `getAddress()` on the order that returns the customer's address, added so the quote has "fewer dots," does not fix anything, because the order has now exposed its customer's internals to anyone. The delegation must move the knowledge, not reroute it. The fix is a method that answers the question, `getShippingCity()`, not a method that hands the internals onward.

The third mistake is applying the law to data holders. A DTO with getters that return plain data, `order.getItems().get(i).getPrice()`, is not a train wreck, because the DTO has no behavior to hide, and the chain is reading data, not reaching through behavior. The law protects the objects that own behavior and internals. Applied to a data bag, it produces a museum of pointless methods.

## Interview Perspective

The question "explain the Law of Demeter" is usually answered with the dot rule, and the interviewer pushes on it. The strong answer separates the proxy from the principle. "A method should talk to its immediate friends, itself, its parameters, its fields, its own creations. `order.getCustomer().getAddress().getCity()` reaches through strangers, and the fix is `order.getShippingCity()`. The dots are not the point, the knowledge of the graph shape is."

The follow-up "is the builder pattern a violation" wants the distinction applied. "No. The builder returns `this` each time, so the chained calls are on the same friend. Streams create their own intermediate objects. The violation is reaching into a return value you did not invite, not the number of dots."

The sharper question: "doesn't the law produce a chain of delegating methods." The strong answer owns the trade. "It produces delegation, and the delegation is the point. Each object hides its internals from the next, and the knowledge stays with the owner. A pass-through method that reroutes the internals is fake, a method that answers the question is the law working."

## Knowledge Check

1. `order.getCustomer().getAddress().getCity()` is flagged in review. State which call violates the law and which friend the quote was allowed to talk to, then write the fixed version.

2. A builder chain `new Builder().withA().withB().build()` has many dots. Explain why it is not a violation, using the friends list from the law.

3. An engineer "fixes" a train wreck by adding `order.getCustomerAddress()` that returns the address. Explain why the fix did not work, and what the correct method would have looked like.

## Key Takeaways

- The Law of Demeter: talk to your immediate friends, and never reach through one object's return value into another.
- The train wreck is the violation, and the fix always moves the walk behind a friend's method.
- The dot count is a proxy for the real crime: knowing the shape of the object graph.
- The law is encapsulation applied across objects, and the delegation it causes is the point, not the cost.

## What's Next

The Law of Demeter kept a method from knowing too much about the objects around it. Separation of Concerns and Information Hiding is the same instinct scaled up to the whole module: split the code by what it is for, and hide the details that are likely to change. The next article covers the layering, the module boundary, and why hiding a decision is the most reliable design move there is.

---

This article explains the Law of Demeter as talk only to your immediate friends, and separates the real violation from the harmless dot chain. Its strongest claim is that the dot count is a proxy for the real crime, knowing the shape of the object graph, and that the delegation the law causes is the point, not the cost.
