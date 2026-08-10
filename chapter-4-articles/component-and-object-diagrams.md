# Component and Object Diagrams

## Learning Objectives

1. Tell a component from a class and an object diagram from a class diagram, and use each where it belongs.
2. Read provided and required interfaces, the ball and the socket, and verify that a dependency between components runs through an interface and not around it.
3. Draw an object diagram as a snapshot, an underlined instance name and the links between instances, to settle a runtime question.

## Introduction

These two diagrams are the scale extremes of the chapter, and they are often confused because they both look like boxes with lines. The component diagram is the system at module scale, the services and packages and their interfaces, bigger than any class. The object diagram is the system at runtime scale, the concrete instances at one moment, smaller than any class diagram.
The component diagram is what you draw when the question is "where are the module boundaries, and do the dependencies cross them honestly." The object diagram is what you draw when the question is "at runtime, which instance is connected to which." One is architecture, the other is a snapshot, and neither is a class diagram with a different name.

## Problem Statement

A team is building a payments system and has a clean class diagram. The classes are sensible, the seams are injected, the code is testable. Then the module boundaries are added, and the trouble starts. The web module's `PaymentController` needs an order, so it calls a method on a class in the persistence module directly, because nobody ever said the controller may only depend on the service module's interface. The dependency graph now has a controller reaching straight into the repository, and the clean class diagram did not catch it, because the class diagram has no notion of module.
That is the failure the component diagram exists to prevent. A system of classes can be internally clean and architecturally wrong, with dependencies that cross module boundaries in ways nobody intended. The component diagram is the picture that draws the boundaries and the interfaces, and it is the review artifact that catches a controller reaching around the service before the code does.
The object diagram prevents a different, smaller failure. Two engineers are arguing about a runtime behavior. "When an order is loaded, does it share the same customer instance as another order?" The class diagram cannot answer it, the multiplicity `1` and `*` describes the possible shapes, not this moment. The argument runs in prose, with neither engineer holding the same picture. An object diagram, one snapshot with the actual instances and the links between them, settles it in one glance.

## Core Concept

The component diagram is built from components and interfaces. A component is a larger-grained unit of the system, a module, a package, a service, a jar, drawn as a box with the `<<component>>` stereotype. What makes a component a component is that it exposes and consumes interfaces rather than being reached into directly.
The provided interface is the ball, a small circle on a short line attached to the component, meaning "this component offers this contract to anyone." The required interface is the socket, a semicircle on a short line, meaning "this component needs this contract from someone." A dependency between components is a line connecting a required interface to a provided interface. The line from socket to ball is the whole meaning of the component diagram: the components do not touch, they meet through contracts.
The Java mapping is direct. A component is a module or a service, a Maven module, a Spring service, a package boundary. The provided interface is the Java interface the component's public classes implement. The required interface is the Java interface the component's classes depend on, injected, not constructed. An `OrderService` module that depends on an `OrderRepository` interface provides an `OrderFacade` interface and requires an `OrderStore` interface. The sockets and balls are the dependency inversion from an earlier chapter, drawn at module scale.
Diagram: the component diagram for a checkout, with the web module requiring the facade that the service provides, and the service requiring the store that the repository provides.

```
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1000 240" font-family="Arial, Helvetica, sans-serif">
  <rect x="80" y="80" width="220" height="130" rx="4" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="190" y="108" font-size="11" fill="#57606a" text-anchor="middle">&lt;&lt;component&gt;&gt;</text>
  <text x="190" y="140" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">WebController</text>
  <rect x="420" y="80" width="220" height="130" rx="4" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="530" y="108" font-size="11" fill="#57606a" text-anchor="middle">&lt;&lt;component&gt;&gt;</text>
  <text x="530" y="140" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">OrderService</text>
  <rect x="770" y="80" width="200" height="130" rx="4" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="870" y="108" font-size="11" fill="#57606a" text-anchor="middle">&lt;&lt;component&gt;&gt;</text>
  <text x="870" y="140" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">OrderRepository</text>
  <line x1="300" y1="135" x2="336" y2="135" stroke="#57606a" stroke-width="1.5"/>
  <path d="M 336,125 A 10,10 0 0 1 336,145" fill="none" stroke="#57606a" stroke-width="1.5"/>
  <line x1="346" y1="135" x2="378" y2="135" stroke="#57606a" stroke-width="1.5"/>
  <circle cx="388" cy="135" r="10" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <line x1="388" y1="135" x2="420" y2="135" stroke="#57606a" stroke-width="1.5"/>
  <text x="360" y="118" font-size="12" fill="#24292f" text-anchor="middle">OrderFacade</text>
  <line x1="640" y1="185" x2="672" y2="185" stroke="#57606a" stroke-width="1.5"/>
  <path d="M 672,175 A 10,10 0 0 1 672,195" fill="none" stroke="#57606a" stroke-width="1.5"/>
  <line x1="682" y1="185" x2="728" y2="185" stroke="#57606a" stroke-width="1.5"/>
  <circle cx="738" cy="185" r="10" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <line x1="738" y1="185" x2="770" y2="185" stroke="#57606a" stroke-width="1.5"/>
  <text x="705" y="166" font-size="12" fill="#24292f" text-anchor="middle">OrderStore</text>
</svg>
```

Read the boundaries as a dependency contract. `WebController` has a socket, it requires the `OrderFacade` that `OrderService` provides through its ball. `OrderService` has a socket, it requires the `OrderStore` that `OrderRepository` provides. There is no line from `WebController` to `OrderRepository`, which is the diagram stating the rule the class diagram could not: the web module has no business with the repository. A component diagram is reviewed by looking for exactly those missing lines, the dependency that bypasses an interface, and by checking that every dependency runs from a socket to a ball.
The object diagram is the other extreme, and its notation is the thing to learn. The box holds an instance, written `:ClassName` when the instance is unnamed, or `name:ClassName` when it has a name, and the name is underlined. That underline is the entire distinction from a class diagram: an underlined `:Order` is one order at one moment, and `Order` without the underline and without the colon is the class. The links between instances are plain lines, and they are instances of associations, not the associations themselves. There are no multiplicities on the links, because each link is one concrete connection between two concrete things.
The object diagram is a snapshot, and it exists to settle snapshot questions. Consider a customer with two orders, loaded by one query.

```
Customer alice = customerRepository.findById(1L);
Order first = alice.getOrders().get(0);
Order second = alice.getOrders().get(1);
```

The class diagram says a customer has many orders. The object diagram says which: `:Customer` with links to `:Order` and another `:Order`, one snapshot, one shared customer, two orders. The question "do the two orders share the same customer instance" is answered by whether both links point at the same `:Customer` box. That is the object diagram's entire job, and it does it in a picture that a sentence takes a paragraph to state.
Diagram: the object diagram as a snapshot, one customer linked to two orders and each order to its items.

```
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 300" font-family="Arial, Helvetica, sans-serif">
  <rect x="60" y="60" width="160" height="70" rx="4" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="140" y="105" font-size="13" fill="#24292f" text-anchor="middle" text-decoration="underline">:Customer</text>
  <rect x="330" y="60" width="160" height="70" rx="4" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="410" y="105" font-size="13" fill="#24292f" text-anchor="middle" text-decoration="underline">:Order</text>
  <rect x="680" y="60" width="160" height="70" rx="4" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="760" y="105" font-size="13" fill="#24292f" text-anchor="middle" text-decoration="underline">:OrderItem</text>
  <rect x="680" y="200" width="160" height="70" rx="4" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="760" y="245" font-size="13" fill="#24292f" text-anchor="middle" text-decoration="underline">:OrderItem</text>
  <line x1="220" y1="95" x2="330" y2="95" stroke="#57606a" stroke-width="1.5"/>
  <line x1="490" y1="95" x2="680" y2="95" stroke="#57606a" stroke-width="1.5"/>
  <line x1="490" y1="120" x2="760" y2="200" stroke="#57606a" stroke-width="1.5"/>
</svg>
```

This snapshot says: one customer, one order, two order items, and both items belong to the same order. The underlined names say these are instances, not classes. The links are the runtime references, the `alice`, the `first`, the `second`. If the question were whether the two orders share a customer, the answer would be a second `:Order` box with both links pointing at the same `:Customer`, and the picture would show it.
The relationship between the two diagrams is worth stating, because it is the source of most confusion. The component diagram is about modules and contracts, drawn from the architecture. The class diagram is about types and relationships. The object diagram is about instances and links. The class diagram says an order has many items. The object diagram says this order has these two items right now. The component diagram says the service depends on the repository through an interface. None of the three replaces another, and asking which to draw is asking what level of the system is being decided.

## Real Production Usage

The component diagram is the standard shape of architecture documentation, and the Java ecosystem gives it a natural mapping: a Maven module is a component, and the dependency between modules, declared in the pom, is the required interface made concrete. A team that keeps a component diagram of its module structure is keeping a map of the boundaries that Maven enforces, and a dependency that bypasses a boundary, one module importing another module's internals, shows up in the diagram before it shows up in the build.
Spring's own architecture is a component diagram made real: `spring-web`, `spring-context`, `spring-beans`, `spring-core`, each a module with documented dependencies between them. The Spring docs draw this exact picture, because a framework with circular module dependencies would be unusable, and the component diagram is how that constraint is communicated.
The object diagram has a thinner production presence, because a snapshot is usually a debugging session, not a document. It shows up in its genuine habitat when a team needs to explain a specific runtime situation: a JPA session with two entities sharing a managed instance, an N+1 query with duplicated references, a cache that holds the same object under two keys. In those discussions, the underlined boxes settle in a picture what a paragraph of prose leaves arguable.

## Common Mistakes

The first mistake is drawing a component diagram with class relationships. A line labeled `uses` between two component boxes, with no socket and no ball, is a diagram that says the modules reached into each other, which is the exact violation the component diagram exists to expose. The component diagram is honest only when every dependency runs through a required interface to a provided one. If you catch yourself drawing a plain arrow between components, you are drawing a dependency that bypasses a contract, and you should either fix the diagram or fix the architecture.
The second mistake is forgetting the underline in an object diagram, which silently turns instances into classes. An `:Order` box with a link to an `:OrderItem` box is a snapshot. The same boxes without the colons and the underlines are a class diagram, and they answer a different question. The colon and the underline are not decoration, they are the difference between "this order" and "the type Order."
The third mistake is using the object diagram to state rules that belong in the class diagram. A snapshot with a multiplicity label on a link, or a note claiming "orders always have two items," is smuggling a type-level rule into an instance-level picture. The object diagram shows a moment; the class diagram states the rule. A snapshot proves nothing about what is always true, and the diagram should not pretend otherwise.

## Interview Perspective

Component diagrams appear in interviews when the question is architectural, and the interviewer wants to see module boundaries and their contracts. The strong answer to "design a notification system" draws the components, the `NotificationService`, the `NotificationRepository`, the `ProviderClient`, and then draws the interfaces between them, so the interviewer can see that the service depends on a provider contract, not on a specific provider. That drawing is the dependency inversion, made visible, and it is the part of the answer the interviewer remembers.
The weak answer describes the modules in prose, "so there'd be a notification service, and it would talk to different providers," with no statement of the contracts. The strong answer draws the balls and sockets, and the interviewer can ask "how do you add a new provider" and the candidate points at the provided interface and says "implement the contract, nothing else changes."
The follow-up that probes the object diagram is smaller and sharper: "is this an instance or a class." The weak answer hesitates. The strong answer uses the notation. "The underlined `:Customer` is an instance, one customer at one moment. `Customer` without the underline is the class. If I want to show that two orders share one customer at runtime, I draw the instance boxes and the links, and the sharing is visible."

## Knowledge Check

1. A web module calls a repository method directly, and the team cannot see the problem. State which diagram makes the violation visible, and what the corrected picture should show between the web component and the repository.
2. You are shown two diagrams of the same system: one with `Order` boxes and an association with a `1` and `*`, and one with underlined `:Order` boxes linked to an underlined `:Customer`. Name each diagram type and state what question each one answers.
3. A teammate proposes adding a multiplicity label to an object diagram link to enforce a business rule. Assess the proposal: which diagram type owns that rule, and why the object diagram cannot state it?

## Key Takeaways

- The component diagram is module scale, and every dependency must run from a required socket to a provided ball.
- A class diagram can be clean while the module boundaries are broken, and the component diagram is what catches the controller reaching into the repository.
- The object diagram is a snapshot, underlined instances with links, and the underline is the whole difference from a class diagram.
- A snapshot proves what happened once, not what is always true, and type rules belong in the class diagram.

## What's Next

The chapter now has all the diagrams it needs: structure, interaction, flow, life, and scale. The remaining question is how to use them in the room that decides your grade. The next article covers using UML effectively in interviews, which diagrams to draw first, how much detail is enough, and how to turn a whiteboard into a design conversation instead of a monologue.

---

This article explains the component and object diagrams as the two scale extremes, the module boundaries with ball-and-socket interfaces, and the snapshot with underlined instances. Its strongest claim is that a component dependency is honest only through an interface, and the underline is all that separates an instance from a class.
