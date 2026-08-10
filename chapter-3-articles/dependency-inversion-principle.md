# Dependency Inversion Principle

## Learning Objectives

1. State DIP as two rules: high-level modules do not depend on low-level modules, and abstractions do not depend on details.
2. Read the direction of the dependency arrows in a design and identify which way they point.
3. Draw the abstraction boundary so the stable policy defines the contract and the volatile details implement it.

## Introduction

The Dependency Inversion Principle is the most structural of the SOLID principles and the one whose name misleads everyone. Inversion is a claim about arrows. In a naive design, the high-level module, the policy, the thing that knows the business rules, points down at the low-level module, the detail, the thing that talks to the database or the card network. The arrow of dependence runs from the stable to the volatile. DIP says to flip it: both should depend on an abstraction, and the abstraction should belong to the high-level side.

The reason is the transitive reach of a dependency, which the relationship article covered. When the policy depends on the detail, a change to the detail is a change to the policy. The policy is the code that must not change, it encodes the business rules, and DIP is the principle that arranges the arrows so the volatile code changes without the policy noticing.

## Problem Statement

A `NotificationService` sends messages, and it was written in the most obvious way.

```
public class NotificationService {
    private EmailSender emailSender = new EmailSender();

    public void notifyUser(String address, String message) {
        emailSender.send(address, message);
    }
}
```

It works, and then the business adds SMS. The change hits the service: it needs an `SmsSender`, a decision about which to use, a constructor that wires both. The service, which is supposed to be about the business rule "notify the user," is now also about which sender exists and how to choose one. Two weeks later the team adds a push channel, and the service grows again. Every new channel is an edit to the policy class.

The test situation is worse. The test wants to verify the notification logic without sending real email, and the service constructed its own `EmailSender` inside the field, so the test cannot substitute anything. The test must mock the concrete class or skip the path. The service is welded to the detail, and the weld is the dependency arrow pointing the wrong way.

## Core Concept

The principle has two sentences, and both matter.

High-level modules should not depend on low-level modules. Both should depend on abstractions.

Abstractions should not depend on details. Details should depend on abstractions.

The first sentence is the structural claim. The policy should not name the detail. It should name an abstraction, and the detail should implement that abstraction, pointing up at it. The second sentence is the ownership claim. The abstraction is not a diagram drawn in the middle by a committee. It is defined by the high-level side, the side that will not change, and the details conform to it.

The fix for the notification service is the same shape every time.

```
public interface MessageSender {
    void send(String address, String message);
}

public class NotificationService {
    private final MessageSender sender;

    public NotificationService(MessageSender sender) {
        this.sender = sender;
    }

    public void notifyUser(String address, String message) {
        sender.send(address, message);
    }
}
```

The service depends on `MessageSender`, which is a contract it defined. The `EmailSender` and the future `SmsSender` implement that contract. The arrow from the service now points at the abstraction, and the arrows from the details point up at the same abstraction. A new channel is a new class and a wiring change, not an edit to the policy. The test hands the service an `EmailSender` fake, and the weld is gone.

Inversion is the right word, and it is worth saying exactly what flips. Before, the service pointed down at the detail. After, the detail points up at the abstraction the service defined. The dependency has not disappeared, nothing disappears, the detail still exists and the service still needs it. What changed is the direction of the arrow and the owner of the contract. The policy owns the seam. That ownership is the heart of the principle.

Diagram: dependency inversion, the arrow that flips direction when the policy owns the contract.

```
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 860 460" font-family="Arial, Helvetica, sans-serif">
  <defs>
    <marker id="dep" markerWidth="10" markerHeight="8" refX="9" refY="4" orient="auto">
      <polygon points="0 0, 10 4, 0 8" fill="#57606a"/>
    </marker>
    <marker id="impl" markerWidth="10" markerHeight="8" refX="9" refY="4" orient="auto">
      <polygon points="0 0, 10 4, 0 8" fill="#ffffff" stroke="#1f6feb" stroke-width="1.5"/>
    </marker>
  </defs>

  <text x="205" y="48" font-size="14" font-weight="bold" fill="#24292f" text-anchor="middle">Before</text>
  <rect x="60" y="90" width="290" height="64" rx="6" fill="#e8f0fb" stroke="#1f6feb" stroke-width="2"/>
  <text x="205" y="124" font-size="13" font-weight="bold" fill="#0a3069" text-anchor="middle">NotificationService</text>
  <rect x="60" y="320" width="290" height="64" rx="6" fill="#fff8c5" stroke="#9a6700" stroke-width="2"/>
  <text x="205" y="354" font-size="13" font-weight="bold" fill="#633c01" text-anchor="middle">EmailSender</text>
  <line x1="205" y1="154" x2="205" y2="320" stroke="#57606a" stroke-width="2" marker-end="url(#dep)"/>
  <text x="225" y="240" font-size="12" fill="#57606a">depends on</text>

  <text x="650" y="48" font-size="14" font-weight="bold" fill="#24292f" text-anchor="middle">After</text>
  <rect x="510" y="90" width="290" height="64" rx="6" fill="#e8f0fb" stroke="#1f6feb" stroke-width="2"/>
  <text x="655" y="124" font-size="13" font-weight="bold" fill="#0a3069" text-anchor="middle">NotificationService</text>
  <rect x="510" y="210" width="290" height="64" rx="6" fill="#ffffff" stroke="#1f6feb" stroke-width="2" stroke-dasharray="6,5"/>
  <text x="655" y="244" font-size="13" font-weight="bold" fill="#0a3069" text-anchor="middle">MessageSender</text>
  <rect x="460" y="340" width="170" height="64" rx="6" fill="#fff8c5" stroke="#9a6700" stroke-width="2"/>
  <text x="545" y="374" font-size="13" font-weight="bold" fill="#633c01" text-anchor="middle">EmailSender</text>
  <rect x="660" y="340" width="170" height="64" rx="6" fill="#fff8c5" stroke="#9a6700" stroke-width="2"/>
  <text x="745" y="374" font-size="13" font-weight="bold" fill="#633c01" text-anchor="middle">SmsSender</text>
  <line x1="655" y1="154" x2="655" y2="210" stroke="#57606a" stroke-width="2" marker-end="url(#dep)"/>
  <text x="672" y="190" font-size="12" fill="#57606a">depends on</text>
  <line x1="545" y1="340" x2="570" y2="274" stroke="#1f6feb" stroke-width="2" stroke-dasharray="6,5" marker-end="url(#impl)"/>
  <line x1="745" y1="340" x2="740" y2="274" stroke="#1f6feb" stroke-width="2" stroke-dasharray="6,5" marker-end="url(#impl)"/>

  <text x="135" y="435" font-size="12" fill="#57606a">Solid arrow: depends on</text>
  <text x="540" y="435" font-size="12" fill="#1f6feb">Dashed arrow: implements</text>
</svg>
```

The wiring has to live somewhere, and the name for that somewhere is the composition root. The service no longer constructs its sender, so something must. The composition root is the one place in the application that knows the concrete classes and assembles the graph, usually at startup, in the application configuration or a dependency injection container. The composition root is the exception that proves the rule: it depends on the details, on purpose, and it is the only place that does. Everything else depends on abstractions.

This is the exact point where DIP and dependency injection get confused, and the confusion costs candidates in interviews and engineers in designs. Dependency injection is a mechanism, passing the dependency in through the constructor. Dependency inversion is a principle, the direction of the arrows. Injection is one way to implement inversion, and you can invert without a container, by assembling the graph by hand in the composition root. You can also inject without inverting, if the injected thing is a concrete class, and then the arrows still point wrong. Mechanism and principle, two different things, and the engineer who holds them apart can explain both.

There is a placement rule that keeps the arrows honest. The abstraction should live where the high-level module can see it without seeing the details, and it should not live in the low-level package. The interface `MessageSender` belongs beside the service or in a shared contract package, not inside the `email` package next to `SmtpEmailSender`. If the interface lives in the detail's package, the service must depend on the detail's package to reach the abstraction, and the inversion is a circle, a diagram that claims to invert while importing the detail. The physical location of the interface is part of the design, not an implementation detail.

When to apply DIP is where the restraint family, DRY and YAGNI, reasserts itself. Not every low-level class needs an interface. The rule from the dependency article holds: the abstraction pays when there is a real second implementation or a real test fake. The notification service earned its interface the day the second channel was contemplated. A `DatabaseConnection` with one vendor, one environment, and no fake will earn it the day those things appear. DIP is a direction for the arrows that exist, not a license to manufacture arrows.

## Real Production Usage

Spring's constructor injection is the mechanism doing DIP's work at scale. A service declares `MessageSender` in its constructor, the container resolves the concrete bean at startup, and the service's source never names a concrete sender. The container is the composition root, the bean definitions are the wiring, and the entire application runs on the inverted arrows.

The repository pattern is the same principle pointed at persistence. The domain service depends on a `UserRepository` interface, the JPA implementation lives in the infrastructure layer, and the domain never imports the database. Swap a `JpaUserRepository` for an in-memory fake in a test, or a Postgres-backed implementation in production, and the domain code does not change. The direction is inverted, the stable domain owns the contract, the volatile persistence implements it.

The ports and adapters, hexagonal, architecture is DIP drawn as a whole system. The application core defines ports, the interfaces it needs, and the adapters, the HTTP controllers, the database gateways, the message brokers, implement the ports. The arrows point inward toward the core. The core owns every contract, and the outside world conforms. It is the same two-sentence principle, scaled up to the architecture.

## Common Mistakes

The most common mistake is putting the interface in the wrong package. The `MessageSender` interface sits next to `SmtpEmailSender` in the email package, and the service imports the email package to reach it. The inversion is decorative, the policy still depends on the detail's package, and a change to the email package still reaches the service. The interface belongs with its consumer.

The second mistake is confusing injection with inversion and stopping there. The constructor takes a concrete `SmtpEmailSender`, which is injection, and the team calls it DIP. The arrows still point from the policy down at the detail, nothing inverted. Injection without an abstraction is just a parameter.

The third mistake is inverting everything. An interface for the internal helper class that will never have a second implementation, an abstraction for a leaf utility, the speculative seam from the OCP article wearing a DIP costume. The principle governs the arrows that matter, the policy-to-detail boundaries, and applying it everywhere is the same ceremony the restraint principles exist to stop.

## Interview Perspective

The question "explain the Dependency Inversion Principle" is often answered with a slogan, and the interviewer usually follows with "is that the same as dependency injection." The weak answer says yes. The strong answer separates them. "Injection is a mechanism, passing the dependency into the constructor. Inversion is a principle, the arrows point at an abstraction the high-level side owns. You can inject a concrete class and still not invert, and you can invert and wire by hand in the composition root."

The follow-up "draw it for me" wants the direction, and the candidate who can say "high-level points at the interface, the details point up at the same interface, and the interface lives with the high-level side" has the principle. The candidate who draws arrows everywhere has missed the inversion.

The sharper question: "where does the interface belong." The strong answer is a placement rule. "Beside the consumer, not beside the implementation. If the interface lives in the detail's package, the policy still depends on the detail to reach it, and the inversion is a circle."

## Knowledge Check

1. A `PaymentService` constructs a `StripeGateway` in its field. Redraw the dependency so DIP holds, naming the interface, its owner, and the composition root.

2. A team adds constructor injection of `StripeGateway` into `PaymentService` and calls it dependency inversion. Identify what is missing, and what the arrows actually look like.

3. The `MessageSender` interface is placed in the `email` package next to `SmtpEmailSender`. Explain why the placement defeats the inversion, and where the interface should have gone.

## Key Takeaways

- DIP is about arrow direction: the policy depends on an abstraction it owns, and the details implement that abstraction.
- The composition root is the one place that wires concrete classes, and everything else depends on abstractions.
- Injection is a mechanism and inversion is a principle; the one does not imply the other.
- The interface belongs with the high-level consumer, not in the low-level package, or the inversion is a circle.

## What's Next

The three SOLID principles that govern seams are covered. The next article changes register: DRY, KISS, and YAGNI are the restraint family, the brakes on the abstraction the change-oriented principles are so eager to build. It covers what duplication actually costs, when simple beats clever, and why not building the thing is sometimes the whole design.

---

This article explains the Dependency Inversion Principle as a claim about arrow direction, with the policy owning the abstraction and the details implementing it. Its strongest claim is that injection is a mechanism and inversion is a principle, and that an interface placed beside the implementation defeats the inversion.
