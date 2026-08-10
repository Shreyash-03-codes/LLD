# Mediator Pattern

## Learning Objectives

- Recognize when a set of objects has become a tangle of direct references, and name the Mediator as the coordinator that removes the tangle.
- Build a mediator and route all colleagues through it, so no colleague holds a reference to another.
- Apply the pattern's honest limit, the mediator coordinates the order but never decides the business outcome, or it becomes the god object it is meant to avoid.

## Introduction

Mediator puts a single coordinator between a group of objects and lets them talk through it instead of through each other. Instead of every colleague holding a reference to every other, each colleague holds one reference to the mediator and sends it events. The mediator knows all of them and routes each event to whoever needs it.

The mediator is the center of a star. Every peer links to the hub and to nothing else. The hub knows where each message goes, so the peers never have to know who cares.

## Problem Statement

The failure is a web of direct references, and it grows exactly like this. Consider a checkout dialog with a form, a submit button, a total display, and an error label. The button needs the form to validate before it can submit. Clicking submit must update the total display. The working relationship looks like this:

```java
public class SubmitButton {
    private final OrderForm form;
    private final TotalDisplay total;

    public void onClick() {
        if (form.isValid()) {
            total.refresh(form.total());
            placeOrder(form.purchase());
        }
    }
}
```

The button already knows the form and the total. The total display knows how to recompute. The form knows the products. Now add an error label that the form should tell when validation fails, and a confirmation banner that appears after an order is placed. Every new peering means editing the button, the form, or the total to add another reference, and each object accumulates a list of peers it has to notify. The choreography, validate, then recompute the total, then enable the button, then place the order, is scattered across the objects, and no single place can read the sequence.

That is the failure. The ordering knowledge is duplicated across every participating object, and each object knows far too many of its neighbors. The direct references are the problem, not the UI.

## Core Concept

Mediator removes the peer references. One interface is what the colleagues call:

```java
public interface Mediator {
    void notify(Colleague sender, String event);
}
```

Each colleague holds the mediator and sends events, holding a reference to nobody else:

```java
public class OrderForm {
    private final Mediator mediator;

    public OrderForm(Mediator mediator) {
        this.mediator = mediator;
    }

    public void onSubmit() {
        mediator.notify(this, "form completed");
    }
}

public class SubmitButton {
    private final Mediator mediator;

    public SubmitButton(Mediator mediator) {
        this.mediator = mediator;
    }

    public void clicked() {
        mediator.notify(this, "submit clicked");
    }
}
```

The concrete mediator owns every colleague and the choreography:

```java
public class CheckoutMediator implements Mediator {
    private final OrderForm form;
    private final TotalDisplay total;
    private final SubmitButton button;

    public CheckoutMediator(OrderForm form, TotalDisplay total, SubmitButton button) {
        this.form = form;
        this.total = total;
        this.button = button;
        this.form.attach(this);
        this.total.attach(this);
        this.button.attach(this);
    }

    @Override
    public void notify(Colleague sender, String event) {
        if ("submit clicked".equals(event)) {
            if (!form.isValid()) {
                form.showError();
                return;
            }
            total.refresh(form.total());
            form.placeOrder();
        }
    }
}
```

The colleagues no longer hold each other. They know only the mediator, and the mediator sees the whole choreography in one place. Adding a confirmation banner is one new colleague, one field, and one line in the mediator, and no existing object changes. The star replaced the mesh.

### The honest rule: what the mediator must not become

The trap is in the word coordinator. The mediator owns the sequence and the wiring. It must not own the business decisions. The moment `CheckoutMediator` starts applying discounts or deciding fraud policy, it has stopped routing messages and started being the largest class in the system, the exact god object the pattern exists to avoid.

The test is simple. Does a message route to the right receiver, or does the mediator also decide the content and the policy? Routing belongs in the mediator. Judgment belongs with the colleague that owns the business rule. A mediator that knows the discount formula is not coordinating, it is accumulating.

### The siblings

Three patterns look like Mediator at a glance, and the differences are the skill.

Observer broadcasts. Every subscriber gets the event. There is no routing and no order. Mediator routes to exactly the one that cares. Observer is a firehose, a reply-all; Mediator is a handoff. Use Observer when a change affects many, Mediator when one colleague must act in a controlled order.

Facade is a front door. It simplifies a subsystem for a client from the outside, and the objects inside still talk to each other directly. Mediator sits inside the group and stops them talking directly. Facade hides the shape from the outside, Mediator rewires the shape from the inside.

Command packages one action. Mediator routes many messages between peers. They combine freely, a mediator can hand a command to a receiver, but they answer different questions.

Diagram: a web before, a hub-and-spoke after

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 840 400" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#444"/>
    </marker>
  </defs>

  <text x="210" y="24" text-anchor="middle" font-size="12" font-weight="bold" fill="#5a6b7a">before: a web</text>
  <text x="630" y="24" text-anchor="middle" font-size="12" font-weight="bold" fill="#5a6b7a">after: hub-and-spoke</text>

  <rect x="40" y="120" width="90" height="50" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="85" y="150" text-anchor="middle" font-size="13" fill="#1a2733">A</text>
  <rect x="165" y="60" width="90" height="50" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="210" y="90" text-anchor="middle" font-size="13" fill="#1a2733">B</text>
  <rect x="165" y="220" width="90" height="50" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="210" y="250" text-anchor="middle" font-size="13" fill="#1a2733">C</text>
  <rect x="40" y="320" width="90" height="50" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="85" y="350" text-anchor="middle" font-size="13" fill="#1a2733">D</text>

  <line x1="85" y1="170" x2="210" y2="110" stroke="#6b7a89" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="154" y1="120" x2="174" y2="217" stroke="#6b7a89" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="130" y1="145" x2="58" y2="324" stroke="#6b7a89" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="255" y1="220" x2="85" y2="318" stroke="#6b7a89" stroke-width="1.5" marker-end="url(#arrow)"/>

  <rect x="500" y="150" width="60" height="60" rx="12" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="530" y="177" text-anchor="middle" font-size="11" font-weight="bold" fill="#1a2733">Mediator</text>
  <text x="530" y="196" text-anchor="middle" font-size="11" fill="#1a2733">hub</text>

  <rect x="395" y="50" width="70" height="44" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="430" y="76" text-anchor="middle" font-size="12" fill="#1a2733">A</text>
  <rect x="655" y="50" width="70" height="44" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="690" y="76" text-anchor="middle" font-size="12" fill="#1a2733">B</text>
  <rect x="655" y="280" width="70" height="44" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="690" y="306" text-anchor="middle" font-size="12" fill="#1a2733">C</text>
  <rect x="395" y="280" width="70" height="44" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="430" y="306" text-anchor="middle" font-size="12" fill="#1a2733">D</text>

  <line x1="465" y1="90" x2="528" y2="147" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="704" y1="98" x2="570" y2="150" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="658" y1="276" x2="560" y2="205" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="398" y1="288" x2="500" y2="205" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
</svg>
```

The left side is four objects wired to each other, a web of direct references. The right side is the same four objects, each linking only to the mediator, with no peer-to-peer line left. The star is the point of the pattern, and the hub is where the coordination lives.

## Real Production Usage

The honest answer is that the Mediator rarely appears as a class named `Mediator`, but the role is everywhere. It is the use-case class, the application service, the controller, the coordinator that owns the choreography of several collaborators so the collaborators do not hold each other. When you read "keep the services decoupled and let one orchestrator own the flow," you are describing the mediator role under another name.

In a GUI toolkit the pattern is real. The dialog coordinator lets a button and a slider work together without storing references to each other, events funnel through the coordinator. In a chat server, the room is the mediator, every participant talks to the room and nobody holds everyone. When you find yourself writing a class whose constructor takes every collaborator and whose one method decides who acts next, you have written a mediator whether you named the pattern or not.

The pattern has a cost and it is worth naming: every call goes through the hub, one extra indirection per message. That is fine when the group is genuinely tangled, and it is dead weight when two objects could be wired directly. The gauge is group size and coordination. The star pays for itself once the peer-to-peer mesh is harder to change than the hub is to read.

## Common Mistakes

**The god object at the center.** The mediator that starts applying discounts and deciding policy stops being a coordinator and becomes the largest class in the system. The discipline is that the mediator knows the choreography, never the judgment. The moment a rule arrives, route the message to the colleague that owns the rule instead of implementing it in the hub.

**Choosing the mediator for two.** Mediator costs a hub and an indirection. For a pair of collaborators, direct wiring is cheaper and clearer. The star pays for itself when the web is actually a web, a real tangle, not a pair. Applying Mediator to a two-object relationship is ceremony.

**Colleagues that peek past the hub.** A colleague that reaches into the mediator to touch another colleague directly has recreated the web with the hub as an extra hop. The colleague knows the mediator and the event, nothing else. If the colleague needs something, it sends an event and lets the mediator decide.

## Interview Perspective

Mediator is the pattern interviewers use to check whether you know the hub from a bus. A weak answer defines the coordinator and draws the star. A strong answer says the mediator knows the order and the routing, the colleague knows only the event, and the mediator never decides the business outcome, which is the sentence most candidates never reach.

The follow-up that sorts people is "Mediator versus Observer." The strong answer says Observer broadcasts to everyone who subscribed and Mediator routes to exactly the one that cares, so Observer is for fan-out and Mediator is for handoff. The god object follow-up follows the same logic: a mediator becomes a god object when it stops routing and starts deciding policy, which is a sign the routing belongs in a colleague.

Common follow-ups:

- "Mediator versus Observer: same message, which pattern sends it to everyone?"
- "Your mediator just gained a method that applies a discount. What has happened?"

## Knowledge Check

1. The checkout dialog: which objects are colleagues, which is the mediator, and what does the mediator now own that the button used to own alone?
2. A discount formula arrives and someone puts it in `CheckoutMediator.notify`. What has the mediator become, and where does the rule belong?
3. A chat room routes a message from one participant to all the others. Name the mediator, the colleagues, and what each colleague knows about the others.

## Key Takeaways

- Mediator turns a web of peer references into a star, and every peer links only to the hub.
- The colleagues know the mediator and the event, never the peers.
- The mediator owns the sequence and the routing, never the business decisions, and that line is the whole trap.
- Facade is a front door, Observer is a broadcast, Mediator is a handoff, and the difference is the skill.
- The star pays for itself when the web is a real tangle; for a pair, direct wiring wins.

## What's Next

The next article is Iterator, which is the oldest and least glamorous pattern in the chapter. Where Mediator reorganizes how many objects talk, Iterator reorganizes how one collection is walked: it hides the collection's internal layout behind a cursor, so the traversal code looks the same whether the collection is an array, a list, or a tree. We will cover the cursor contract, the fail-fast behavior, and why every `for-each` loop is quietly an Iterator.

---

This article explains the Mediator pattern as a hub that replaces a web of direct references, with a checkout dialog as the example. It argues the mediator routes and sequences without deciding business policy, earning its keep only for a genuine tangle.
