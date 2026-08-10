# Class Diagrams

## Learning Objectives

1. Translate a Java class into its diagram form, the box, the compartments, the visibility markers, so the mapping is second nature.
2. Read the five relationship arrows, inheritance, implementation, association, composition, and dependency, and draw each one in the correct direction.
3. Use a class diagram as a review artifact, the place where a wrong relationship is caught before it is shipped as a wrong structure.

## Introduction

The class diagram is the workhorse of low level design, and the reason is simple: it shows the shape of the code. A class diagram answers the question every reviewer is asking, what depends on what, who owns what, who extends what, in a single glance. Everything else in this chapter is about behavior over time or flow; the class diagram is about structure, the bones, and it is the diagram you will draw first in almost every design conversation.
It is also the easiest diagram to get subtly wrong, because its symbols are small and its mistakes are quiet. An arrow drawn the wrong way around is a diagram that teaches the team the opposite of the truth. Getting the notation right is not pedantry, it is precision that pays.

## Problem Statement

A design review is on. The team is adding a loyalty program, and someone sketches the structure on the whiteboard: a box called `LoyaltyService`, a box called `PointRepository`, a box called `Customer`, and a line between `LoyaltyService` and `PointRepository` with an arrowhead somewhere on it. Nobody can say which end the arrow is on, because nobody drew it clearly, and the conversation moves on without the question being resolved.
Three weeks later the code lands. `LoyaltyService` calls a static `PointRepository.update(...)` directly, because the engineer who drew the box meant the arrow that way, and the engineer who built it read it the other way. The service is now welded to a concrete repository, there is no seam for a test, and the diagram that was supposed to prevent exactly this did nothing, because the notation was too loose to carry the meaning.
The failure is not that the team drew. The failure is that they drew with symbols that could not express the distinction that mattered. A class diagram with real notation, the dependency arrow pointing at the dependency, the interface marked, the multiplicity stated, would have settled the question on the whiteboard, in the same meeting, for free.

## Core Concept

The unit of a class diagram is the class box. It is a rectangle with the class name in a top compartment, the attributes in a middle compartment, and the operations in a bottom compartment. That is the classic three-compartment box, and it is already more than most diagrams need. In a design discussion, the middle and bottom compartments are usually noise; the interesting information is in the name and in the relationships. A box with `Order`, a few attributes, and a couple of operations is usually enough, and you can drop the compartments entirely when the point is the relationships.
The visibility markers come from Java and go straight into the diagram. `+` is public, `-` is private, `#` is protected, and `~` is package-private, which you will almost never draw because package-private is rarely worth communicating. A method written as `+ checkout(gateway: PaymentGateway): Receipt` means the return type comes after the colon. If that reads familiar, it is because it is the Java signature with the parameter names dropped.
The relationships are where the class diagram earns its keep, and each one has a specific arrow.
| Relationship | Notation | Java meaning | Direction of the arrow |
| --- | --- | --- | --- |
| Inheritance | Solid line, hollow triangle | `class B extends A` | Triangle points at the parent |
| Implementation | Dashed line, hollow triangle | `class B implements I` | Triangle points at the interface |
| Association | Solid line, optional arrowhead | A field that references another type | Arrow, if drawn, points at the referenced type |
| Aggregation | Solid line, open diamond at owner | Shared lifetime, the parts can outlive the whole | Diamond at the owner |
| Composition | Solid line, filled diamond at owner | Exclusive lifetime, the parts die with the whole | Diamond at the owner |
| Dependency | Dashed line, open arrowhead | Uses as a parameter, return, or local | Arrow points at the dependency |
The hollow triangle has two variants, and mixing them up is the most common silent error. Solid line with hollow triangle means inheritance, `class SavingsAccount extends Account`. Dashed line with hollow triangle means implementation, `class SavingsAccount implements InterestBearing`. The triangle always points at the thing being extended or implemented. If you keep the rule "the triangle points at the parent, the arrow points at the dependency," the direction stops being a guess.

```
public abstract class Account {
    protected BigDecimal balance;
}
public interface InterestBearing {
    double applyInterest();
}
public class SavingsAccount extends Account implements InterestBearing {
    @Override
    public double applyInterest() { ... }
}
```

That snippet maps to two arrows. A solid line with a hollow triangle from `SavingsAccount` up to `Account`. A dashed line with a hollow triangle from `SavingsAccount` up to `InterestBearing`. Both triangles point at the supertype. Anyone who reads the diagram later knows, without looking at the code, that `SavingsAccount` is an `Account` and honors an `InterestBearing` contract.
The association family is where "has-a" gets its shades. A plain association is just a field reference, the customer an order points at. An aggregation, an open diamond, means the parts are shared or can outlive the container, a `Team` that references `Employee`s who still exist after the team is dissolved. A composition, a filled diamond, means the parts live and die with the container, the `OrderItems` that are deleted when their `Order` is deleted. The rule of thumb most teams use: if deleting the container deletes the parts, it is composition; if the parts can survive alone, it is aggregation; if you are not sure it matters, draw a plain association and move on.
The dependency arrow is the most useful and the most ignored. A dashed line with an open arrowhead means "this class uses that class," as a parameter, a return type, or a local. It is how you show that `Order.checkout` depends on `PaymentGateway` without claiming an ownership relationship. Dependency arrows are how a class diagram shows coupling, and coupling is what this entire handbook has been about. A diagram that shows every dependency arrow is a diagram that shows you where the seams are missing.
A diagram is only as good as its labels, and the multiplicities are the labels that carry the most design information. `1` on the customer end of the order association says an order has exactly one customer. `*` on the order-item end says an order has many items. Multiplicity is the part of a class diagram that corresponds to the constraints in your database schema, and it is the part most likely to catch a real bug, because "one" and "many" are exactly the assumptions that break when a requirement changes.
Diagram: the class structure of an order checkout, showing association, composition, and a dependency arrow.

```
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1000 560" font-family="Arial, Helvetica, sans-serif">
  <defs>
    <marker id="depArrow" markerWidth="12" markerHeight="10" refX="10" refY="5" orient="auto">
      <polygon points="0 0, 11 5, 0 10" fill="#ffffff" stroke="#57606a" stroke-width="1.5"/>
    </marker>
  </defs>
  <rect x="40" y="80" width="210" height="122" rx="4" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <line x1="40" y1="124" x2="250" y2="124" stroke="#d0d7de" stroke-width="1"/>
  <text x="145" y="108" font-size="14" font-weight="bold" fill="#24292f" text-anchor="middle">Customer</text>
  <text x="52" y="144" font-size="12" fill="#24292f">- id: Long</text>
  <text x="52" y="170" font-size="12" fill="#24292f">- name: String</text>
  <text x="52" y="196" font-size="12" fill="#24292f">+ getAddress(): Address</text>
  <rect x="390" y="80" width="270" height="148" rx="4" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <line x1="390" y1="124" x2="660" y2="124" stroke="#d0d7de" stroke-width="1"/>
  <line x1="390" y1="202" x2="660" y2="202" stroke="#d0d7de" stroke-width="1"/>
  <text x="525" y="108" font-size="14" font-weight="bold" fill="#24292f" text-anchor="middle">Order</text>
  <text x="402" y="144" font-size="12" fill="#24292f">- id: Long</text>
  <text x="402" y="170" font-size="12" fill="#24292f">- items: List&lt;OrderItem&gt;</text>
  <text x="402" y="196" font-size="12" fill="#24292f">- customer: Customer</text>
  <text x="402" y="222" font-size="12" fill="#24292f">+ checkout(PaymentGateway): Receipt</text>
  <rect x="730" y="80" width="210" height="122" rx="4" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <line x1="730" y1="124" x2="940" y2="124" stroke="#d0d7de" stroke-width="1"/>
  <text x="835" y="108" font-size="14" font-weight="bold" fill="#24292f" text-anchor="middle">OrderItem</text>
  <text x="742" y="144" font-size="12" fill="#24292f">- sku: String</text>
  <text x="742" y="170" font-size="12" fill="#24292f">- quantity: int</text>
  <text x="742" y="196" font-size="12" fill="#24292f">- price: BigDecimal</text>
  <rect x="390" y="400" width="270" height="96" rx="4" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5" stroke-dasharray="6,5"/>
  <line x1="390" y1="444" x2="660" y2="444" stroke="#d0d7de" stroke-width="1"/>
  <text x="525" y="428" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">&lt;&lt;interface&gt;&gt;</text>
  <text x="402" y="464" font-size="12" fill="#24292f">+ authorize(amount): void</text>
  <text x="402" y="490" font-size="12" fill="#24292f">+ capture(): void</text>
  <line x1="250" y1="141" x2="390" y2="154" stroke="#57606a" stroke-width="1.5"/>
  <text x="295" y="164" font-size="12" fill="#57606a" text-anchor="middle">1</text>
  <polygon points="660,180 674,172 688,180 674,188" fill="#24292f"/>
  <line x1="688" y1="180" x2="730" y2="180" stroke="#57606a" stroke-width="1.5"/>
  <text x="668" y="196" font-size="12" fill="#57606a" text-anchor="middle">1</text>
  <text x="712" y="196" font-size="12" fill="#57606a" text-anchor="middle">*</text>
  <line x1="525" y1="228" x2="525" y2="392" stroke="#57606a" stroke-width="1.5" stroke-dasharray="6,5" marker-end="url(#depArrow)"/>
</svg>
```

Reading this diagram is the whole lesson. The solid line between `Order` and `Customer` with a `1` on the customer end says an order references exactly one customer, a plain association. The filled diamond at the `Order` end of the `OrderItem` line, with a `*` on the item end, says an order owns its items and they die with it, composition. The dashed line with the arrowhead pointing at `PaymentGateway` says `Order.checkout` depends on the gateway but does not own it. The dashed box with `<<interface>>` says the dependency is on a contract, which means the gateway can be faked in a test. Read left to right, the diagram is a summary of the coupling decisions in the code, and every arrow is a decision someone should be able to defend.
The direction rule is the one to internalize. Inheritance and implementation point at the parent. Dependency points at the dependency. Association and composition point at the thing being referenced or owned, and when the ownership is the point, the diamond makes it unambiguous. If you draw the triangle or the arrowhead on the wrong end, you have inverted the truth, and the team that reads your diagram will build the inverse of what you meant.

## Real Production Usage

The class diagram shows up in production as a documentation artifact, and the most common form in the Java world is generated. IntelliJ IDEA renders a live class diagram from your source, and every team that works in it has seen the view that turns a package into boxes and arrows. That generated diagram is the honest version: it cannot lie, because it is derived from the code, and it rots the instant the code changes, which is why nobody treats it as a deliverable. It is a tool for reading, not a document to maintain.
ArchUnit is the closest thing to a class diagram that enforces itself. An ArchUnit test declares the dependencies the architecture allows, "controllers may not depend on repositories directly," and fails the build when the code violates it. That is the dependency arrow from this article turned into a build gate. Teams that use it are not drawing the diagram, they are compiling it, and the coupling decisions are enforced instead of sketched.
The framework docs you read are full of class diagrams for the same reason they work in a review. Spring's reference documentation draws the request lifecycle and the bean definitions as boxes and arrows because a framework's structure is read fastest as relationships. When you look at one of those diagrams, you are using the exact skill from this article: identify the boxes, read the arrows, and know immediately which classes the framework will wire together.

## Common Mistakes

The most common mistake is drawing the inheritance triangle with a solid line where an interface is involved. The distinction between `extends` and `implements` is a solid versus dashed line, and mixing them up silently teaches the wrong relationship. The fix is the triangle rule: solid for class, dashed for interface, triangle at the parent, and it becomes mechanical with practice.
The second mistake is arrow direction. The dependency arrow points at the dependency, and a large fraction of hand-drawn diagrams get this backwards, showing `LoyaltyService <- PointRepository` when the meaning is `LoyaltyService -> PointRepository`. When the direction is inverted, the diagram claims the repository depends on the service, and a reviewer will draw the wrong conclusions about which side owns the seam. If you are unsure, check the Java: the class that names the other type in its signature is the one the arrow starts at.
The third mistake is over-drawing. A class diagram that lists every field and every method of every class in a system is unreadable, and it is usually produced by someone who confused the diagram with the code. The diagram earns its keep when it shows the relationships, the multiplicities, the interfaces, and the direction of the dependencies. Attributes are context, not content. When a box is taller than the relationships are interesting, you have over-drawn.

## Interview Perspective

The class diagram is the first thing most interviewers expect from an LLD question, even when they do not name it. The question "design a parking lot" is, in practice, "draw me the class diagram of a parking lot." The candidate who draws boxes named `ParkingLot`, `Level`, `Spot`, `Vehicle`, and `ParkingTicket` and then labels the relationships, filled diamond from `Level` to `Spot`, association from `ParkingLot` to `Level`, has answered the structural half of the question before the interviewer has finished asking.
The weak answer describes classes in prose, "so we'd have a parking lot class, and it would have levels, and levels would have spots," and the interviewer has to assemble the structure from the stream of words. The strong answer draws and lets the drawing carry the structure, then talks. The difference is visible in the first two minutes: one candidate is building a picture the interviewer can point at, the other is asking the interviewer to hold a paragraph in their head.
The follow-up that tests depth is "why is the diamond filled here and not there." The weak answer says "because that's how composition is drawn." The strong answer explains the ownership decision. "The level owns its spots, so a level with no spots is a level with nothing, filled diamond. The ticket references a vehicle but the vehicle exists without the ticket, plain association, and the vehicle can be checked out and returned, so it outlives the ticket." The interviewer is not testing notation recall. They are testing whether the notation is carrying a real design decision.

## Knowledge Check

1. A `Warehouse` references `Location`s that are assigned by an external system and may be re-assigned between warehouses. State which relationship arrow you draw between the two, and justify why aggregation rather than composition.
2. You are shown a class diagram with a solid line and hollow triangle from `Car` to `Vehicle`, and a dashed line with an open arrow from `CarService` to `Car`. Translate both relationships into the Java that would produce them, including the modifiers.
3. A reviewer looks at your diagram and asks why the dependency arrow points at `PaymentGateway` when the gateway is "used by" the order. Write the one-sentence rule that settles the direction dispute, and give the Java line that confirms it.

## Key Takeaways

- A class diagram is the bones of the design, and it answers "what depends on what" faster than any prose.
- The triangle points at the parent, the arrow points at the dependency, the diamond marks the owner.
- Composition is filled and exclusive, aggregation is open and shared, and when it does not matter, draw a plain association.
- Multiplicity carries the schema constraints, and the diagram is only as honest as its arrow directions.

## What's Next

The class diagram is static, a snapshot of structure. The sequence diagram is the same system moving, and it shows one interaction between objects over time, who calls whom in what order. The next article covers the lifelines, the activation bars, and the arrows that turn a class diagram's relationships into a story about a single request.

---

This article explains the class diagram as the structural workhorse of low level design, with the box and the relationship arrows mapped to Java. Its strongest claim is that arrow direction is the diagram's honesty, and a wrong-end arrowhead teaches the inverse of the truth.
