# Open-Closed Principle

## Learning Objectives

1. State OCP as "open for extension, closed for modification" and say what each half actually means.
2. Build the abstraction that turns a new type of thing into a new class instead of an edit.
3. Judge when closing a module is worth its price, and when it is speculation.

## Introduction

The Open-Closed Principle says a module should be open for extension and closed for modification. Translate that out of the slogan: you should be able to add new behavior to a system without editing the code that already exists. New behavior lands as new code, a new class, a new handler, a new implementation. The existing classes, the ones that were tested and shipped, stay exactly as they are.

That is the prize. Closed for modification means the code that passed its tests does not get touched when the requirement grows. Every edit to a working class is a risk, because edits can break behavior that was already correct. OCP is the principle of structuring so that growth happens by addition, not by revision.

## Problem Statement

A shipping cost calculator handles three countries. The first version works:

```
public class ShippingCost {
    public double calculate(String country, double weight) {
        if (country.equals("US")) {
            return weight * 2.0 + 5.0;
        } else if (country.equals("UK")) {
            return weight * 1.8 + 4.0;
        } else {
            return weight * 2.5 + 6.0;
        }
    }
}
```

A fourth country arrives, and the change is a new `else if`. Then a fifth. Every country is an edit to the one method, and the method grows into the switch that everyone is afraid to touch. The tests for the existing countries stay, but each new country adds a branch to the same block, and a mistake in one branch is a risk to every other branch. The calculator is not closed against anything. It is a wall that every new requirement walks up to and hammers.

The deeper problem is not the switch. It is that the calculator owns the knowledge of how every country is priced. The countries are the thing that varies, and their pricing knowledge is bolted into a method that also owns the flow. When the two mix, growth is an edit.

## Core Concept

The principle is achieved by inverting that ownership. The thing that varies, the pricing rule, becomes an abstraction, and the flow depends on the abstraction instead of on every concrete country.

```
public interface PricingRule {
    double calculate(double weight);
    boolean appliesTo(String country);
}

public class UsRule implements PricingRule {
    public double calculate(double weight) { return weight * 2.0 + 5.0; }
    public boolean appliesTo(String country) { return country.equals("US"); }
}

public class UkRule implements PricingRule {
    public double calculate(double weight) { return weight * 1.8 + 4.0; }
    public boolean appliesTo(String country) { return country.equals("UK"); }
}
```

Now the calculator depends on `PricingRule`, and a fourth country is a new class that implements it. The calculator does not change. The new behavior is an addition. That is the whole mechanism, and it is the same shape every time: find the thing that varies, put it behind an interface, let the stable flow depend on the interface.

Two words in the principle deserve the precision they rarely get. "Open for extension" does not mean the class is editable. It means the behavior of the module can be extended without the module changing, by supplying a new implementation of the seam it already exposes. "Closed for modification" is not a promise that you will never fix bugs or rewrite. It is a statement about how growth happens: growth adds classes; it does not revise the ones that work.

Now the uncomfortable truth that separates engineers who use OCP from engineers who quote it. Closing a module costs you something. The `PricingRule` interface, the registry that maps countries to rules, the wiring, all of that is indirection that the if-chain did not have. The if-chain was honest, it had three branches and it was readable. OCP bought you the ability to add a fourth without editing, and it charged you an interface, a registry, and a loss of the switch's directness.

The bet is only worth taking if the fourth country is coming. If a shipping calculator serves a company that will never ship outside three countries, the if-chain is correct and the interface is ceremony. If new countries arrive every quarter, the if-chain is a tax you pay forever, and the interface pays for itself in the second or third addition.

This is why the honest version of OCP is not "make everything open." It is "close the module against the axis of change that actually exists." The axis here is the country, the kind of thing that varies. You close the calculator against that axis, and you leave everything else alone. Speculative closing, an interface for the axis of change that never shows up, is not OCP, it is YAGNI's warning in the other direction.

There are established patterns for the closing, and they are the same idea in different costumes. The strategy pattern hands the flow a pluggable behavior. The observer pattern closes the subject against new kinds of listeners. The template method closes the flow against new steps. What they share is the seam: one interface, stable callers, and implementations that arrive later.

The behavioral rule that holds all of it: if you are editing an existing class to handle a new kind of thing, and the new kind is not a bug fix, you have likely missed the seam. The first edit is the signal. The second edit is the confirmation. By the third new kind, the seam should already exist, because the edit has proven it will keep coming.

## Real Production Usage

JDBC is the largest example in daily use. The client code calls `DriverManager` and `Connection` and never names a database. A Postgres driver, an Oracle driver, a MySQL driver, all of them implement the same JDBC interfaces, and adding a new database vendor means adding a JAR and a driver class. Nothing in the client is edited. The entire JDK is closed against the vendor axis, and every vendor implements the seam. That is OCP at the scale of an ecosystem.

Spring works the same way at the application level. A `@Controller` is a new handler, and the framework dispatches to it without being modified. The `DispatcherServlet` is closed against the controller axis, and the framework ships without knowing your controllers exist. You add one, and the existing framework code never changes. The same shape is behind every `HandlerInterceptor` and every event listener in the ecosystem: the stable flow, the seam, the arriving implementations.

The message-handler pattern in Kafka or Spring applications is the same shape one level down. A new message type arrives, you write a new handler for it, register it, and the dispatcher that routes messages is untouched. The dispatcher depends on a `MessageHandler` contract, the new type supplies an implementation, and the closed loop stays closed.

## Common Mistakes

The most common mistake is closing against an axis that does not exist. An interface appears because "we might need a second implementation," and a year later there is still one. The interface charges you indirection forever for a bet that lost. The fix is the evidence rule: wait for the second implementation, then close, because the second implementation is what proves the axis is real.

The second mistake is closing at the wrong seam. The calculator was closed against countries, but the change that keeps arriving is the pricing formula itself, a new discount, a new surcharge. Closing against the wrong axis leaves the real variation still bolted into the flow, and the "open" module is open in the direction nobody changes. The axis is found by watching where the edits land, not by predicting.

The third mistake is treating bug fixes as modifications that violate OCP. A pricing formula with a typo gets corrected, and someone argues the fix should be a new class to preserve closure. That is nonsense. Closure is about growth. Bugs are fixed in place. The principle governs how new behavior arrives, not how broken behavior is repaired.

## Interview Perspective

The question "explain the Open-Closed Principle" is usually answered with the slogan, and the follow-up is where the interview happens. "Give me an example." The weak answer is a switch statement being replaced by polymorphism, described vaguely. The strong answer is concrete: "A shipping calculator with an if-chain per country becomes a `PricingRule` interface, and a new country is a new implementation. The calculator never changes."

The follow-up that separates the candidates who understand the price is "isn't the abstraction a waste of time if there's only one implementation." The strong answer admits the tension. "Yes. Closing a module costs an interface and a registry. I close it when the second implementation is coming, not on a guess, because the bet only pays when the axis of change is real."

The sharper interviewer asks "what is the axis of change." The candidate who answers "the thing that keeps varying, found by watching where the edits land" has the principle. The candidate who answers "everything" has turned OCP into the excuse for speculative abstraction that this article warned about.

## Knowledge Check

1. A `NotificationService` has a method that sends email with an if-chain on message type. Add a second channel, SMS, in a way that leaves the service unchanged. Name the seam and the two implementations.

2. An interface `PaymentGateway` exists with one implementation and no second one planned. Is keeping it a violation of OCP or of restraint, and under what evidence would the answer change?

3. A pricing method is closed against the product-type axis, and the change history shows every edit was actually a change to the discount formula. What does the history say about the chosen seam, and where should the seam have been?

## Key Takeaways

- Open for extension, closed for modification means growth by addition, new classes, not new edits.
- The mechanism is one seam: a stable flow depends on an interface, and new behavior implements it.
- Closing a module costs indirection, so close it against the axis of change that actually exists, found by watching the edits.
- A bug fix in place is not a modification that violates OCP; closure governs growth, not repair.

## What's Next

The seam OCP depends on is only safe if the new implementations genuinely behave like the old ones. The Liskov Substitution Principle is the contract underneath that promise: what a subclass must guarantee so that a caller of the parent type never has to know which implementation it got. The next article covers the behavioral contract that the compiler cannot check.

---

This article explains the Open-Closed Principle as growth by addition, new classes rather than new edits, behind the one seam the stable flow depends on. Its strongest claim is that closing a module costs indirection, so you close it only against the axis of change that actually exists, found by watching where the edits land.
