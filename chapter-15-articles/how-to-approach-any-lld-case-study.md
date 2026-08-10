# How to Approach Any LLD Case Study: The 45-Minute Framework

## Learning Objectives

- Walk into any "Design a \_\_\_" question with a repeatable, time-boxed plan instead of improvising on the spot.
- Learn what proportion of the 45 minutes belongs to each phase: requirements, entities, class design, and walkthrough.
- Understand why the walkthrough is the part interviewers actually grade, and how to make yours worth grading.

## Introduction

Every LLD question you will ever be asked is the same question wearing a different costume. Parking lot, elevator, vending machine, Splitwise, movie tickets: the costume changes, the underlying motion is identical. You gather requirements, you name the entities, you write classes, you trace a request through them, and you defend your choices. Once you see that, the interview stops being about whether you happen to know parking lots and becomes about whether you can do that motion reliably under pressure.

The reason this matters is time. Forty-five minutes is not long enough to do everything well. It is exactly long enough to do the right five things competently if you refuse to spend your budget on the wrong ten. Most candidates fail the case study not because the problem is hard but because they spend twenty minutes designing the parking fee calculator and never leave time to show the interviewers how a car actually enters the lot. That ordering mistake is fatal, and it is entirely preventable.

## Problem Statement

Here is what a bad session looks like, and I have seen this dozens of times as an interviewer.

The candidate hears "design a parking lot." They nod, pick up the marker, and start drawing classes immediately. `ParkingLot`, `ParkingSpot`, `Car`, `Ticket`. Then a `Vehicle` base class, then `Truck extends Vehicle`, `Bus extends Vehicle`, `ElectricCar extends Vehicle`. Then a `PaymentStrategy` because they read a book once. Thirty minutes in, they have a wall of classes that look impressive and do nothing. The interviewer asks "how do I park a car?" and the candidate goes silent, because nowhere in those classes is a method that answers that question.

The concrete failure is not a knowledge failure. It is a sequencing failure. The candidate never asked what the system was for. They never asked who uses it, how many spots there are, whether pricing exists, whether the lot has multiple floors, whether the customer pays before leaving. They designed a generic domain model and hoped it would be right. It never is.

## Core Concept

The framework has five phases. The names do not matter. What matters is that you allocate roughly this much time to each and that you refuse to let any phase bleed into the next.

**Phase 1: Requirements (8 minutes).** Your only job here is to make the interviewer talk. Ask what the system does, who uses it, and what the most important flow is. Do not ask forty questions. Ask five good ones and let the answers shape the design. The interviewer has probably heard the exact same five questions from a hundred candidates, so ask them with intent: "who is the user and what is their primary action?" "what does success look like in one sentence?" "are there multiple actors?" "is this single-user or multi-user?" "any persistence or storage requirements?" Write the answers down as you go. If you write them down, you will not re-ask them later, and you will not design against requirements you invented.

At the end of this phase, restate the scope back in one or two sentences. "So we have a multi-level parking lot, single entry and exit, hourly billing, and we're not handling reservation. Is that right?" That restatement is the cheapest insurance policy in the whole interview. It takes twenty seconds and it means everything you draw afterward is agreed upon.

**Phase 2: Entities (7 minutes).** Name the nouns. Do not yet design them. A parking lot: `ParkingLot`, `ParkingFloor`, `ParkingSpot`, `Vehicle`, `Ticket`. An elevator: `Elevator`, `ElevatorController`, `Request`, `Floor`. A vending machine: `VendingMachine`, `Product`, `Shelf`, `Coin`, `Payment`. If you can name the entities for a given problem, you understand it. If you are stuck naming entities, you skipped Phase 1, go back.

Keep this phase honest by writing each entity as a one-liner: "Ticket is issued at entry and contains the entry time and spot." If an entity cannot earn a one-line responsibility, it does not belong in the design. `ElectricCar extends Vehicle` cannot earn a one-line responsibility in a basic parking lot, so it does not belong. Same for `ParkingSpotObserver`. Draw the line early.

**Phase 3: Class design (15 minutes).** This is the bulk of your time, and this is where most candidates go wrong in the other direction: they start writing full Java with getters and setters for everything. Do not. Write the classes that carry the story of the system. A class earns space in the design if a request from the outside world passes through it. `ParkingLot.enter(Vehicle)` and `ParkingLot.exit(Vehicle, Payment)` earn space. `Ticket.getIssuedAt()` does not.

Design each class with its fields, its public methods, and the relationships to its neighbors. Show the important methods with signatures and a few lines of body when the body carries meaning. Do not fill in trivial bodies. The interviewer wants to see `findAvailableSpot(vehicleType)` return a spot based on whether it fits, not twenty lines of `return null`.

**Phase 4: Walkthrough (10 minutes).** Stop writing. Trace a real scenario through the classes you drew. "A car enters. The gate attendant calls `parkingLot.enter(car)`. That asks the ground floor for a spot matching the car's size, marks the spot occupied, issues a ticket, and raises the gate. The driver parks, comes back two hours later, hands over the ticket, we look up the entry time, compute the fee, take the payment, free the spot." This walkthrough is where the interview is won or lost, and most candidates treat it as optional. It is not optional. It is the demonstration that your design works. If you cannot walk a request through your own classes, you have not designed a system, you have drawn a diagram.

**Phase 5: Trade-offs and close (5 minutes).** The interviewer will push on one or two spots. Maybe the fee computation should be a strategy so new pricing rules plug in without touching `ParkingLot`. Maybe the spot search should be a service so you can change allocation logic. Address the push, change the design in the direction they suggest if the suggestion is sound, and summarize where you would take it with more time. Then stop. Do not keep adding features. You are done.

## Real Interview Context

I have sat on the other side of this table enough times to tell you exactly what the score sheet looks like. Interviewers are not counting classes. They are answering three questions about you: Did you understand the problem before you started solving it? Did you produce a design that a competent engineer could implement from, without a hundred follow-up clarifications? And when I pushed on your choices, did you reason or did you fold?

The framework exists to make those three answers "yes." Phase 1 buys the first. Phases 2 and 3 buy the second. Phase 5, plus your demeanor during the walkthrough, buys the third. Notice that nothing in the scoring requires you to know the "correct" answer to a parking lot. There is no correct answer. There is a coherent one, and there is an incoherent one. The framework pushes you toward coherent.

## Common Mistakes

The most common mistake is skipping requirements to look smart. Candidates assume that asking questions signals weakness, so they charge straight into classes to prove they know the domain. Interviewers do not read it that way. They read it as "this person will design the wrong thing and not notice." Asking two sharp questions is worth more than drawing ten correct classes.

The second mistake is over-engineering. You have twenty minutes of class design and you spend them on the visitor pattern for the payment system. Nobody asked for that. A parking lot with a `PaymentStrategy` abstraction and four implementations is a parking lot with a problem, because the interviewer cannot verify any of it in the time left. Put the abstraction behind the seam, name it, and move on. If they want to dig into it, they will.

The third mistake is the dead walkthrough. The candidate finishes drawing, looks up, and says "and then it works." That is not a walkthrough. A walkthrough names concrete objects and concrete calls: "ticket.twoHourFee() returns the amount, cashier.collect(ticket) marks it paid." Vague is indistinguishable from wrong, so be specific or be gone.

## Interview Perspective

A weak answer is characterized by a candidate who draws more classes than they can explain. Ask them to trace a flow and they lose their place. They invented a `ParkingLotObserver` and they do not remember what it observes. The amount of drawing is inversely proportional to the amount of thinking.

A strong answer is characterized by a candidate who does the opposite: a handful of classes, each one defensible, and a five-minute walkthrough that any implementer could follow. When pushed, they either give a concrete reason ("the spot finder is a method because we have one allocation strategy and no evidence we need more") or they revise cheerfully ("fair, let me extract a SpotAllocationStrategy interface, here's where it goes").

The follow-ups are predictable. "What if the lot has multiple floors?" "What if spots are reserved?" "How do you price different vehicle types?" None of these are gotchas. They are tests of whether your design has seams. A good seam absorbs a follow-up in ten seconds. A monolith absorbs it by being redrawn. Build seams.

## Knowledge Check

1. A candidate spends the first twenty minutes of a parking lot interview designing `Vehicle` subclasses and payment abstractions, and has ten minutes left when the interviewer says "show me how a car leaves." What went wrong, and what should they have done with those first twenty minutes?
2. The interviewer says "we only need hourly pricing, single floor, no reservations." You had planned a multi-floor, strategy-driven design. Do you downgrade your design, keep it, or somewhere in between, and why?
3. A follow-up asks you to add a second pricing rule. Name the seam you would have built in Phase 3 that makes this a two-line change instead of a rewrite.

## Key Takeaways

- Five phases, roughly this split: requirements 8, entities 7, class design 15, walkthrough 10, close 5. The walkthrough is not a bonus; it is the deliverable.
- Restate the scope out loud after requirements. Twenty seconds that saves twenty minutes.
- A class only earns space if an external request passes through it. Delete the rest.
- Trace one full user journey through your classes before you declare the design done.
- Build seams where the interviewer is obviously going to push. They always push at the same spots.

## What's Next

The framework stays the same for every case study in this chapter; only the domain changes. Next we apply it cold to the most over-asked question in LLD interviews, starting with how most people get its entity model wrong before they write a single class.



