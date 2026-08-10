# Common UML Mistakes to Avoid

## Learning Objectives

1. Recognize the diagram failures that cost the most, the arrow that lies, the diagram that drifted, the box that over-draws.
2. Fix a bad diagram by making it checkable against the code, and state the check for each element.
3. Keep diagrams small, current, and owned, so the picture never teaches the team the wrong structure.

## Introduction

Most UML problems are not notation problems. The arrowhead is easy to learn and easy to fix. The problems that cost teams are honesty problems: an arrow that points the wrong way and teaches the wrong dependency, a diagram that drifted and now shows the structure the code used to have, a diagram so detailed that nobody can read the decision it is making. The notation is the easy part. The discipline is the hard part.
This article is the chapter's failure catalog. It collects the mistakes in one place so you can use it as a review checklist, on your own drawings and on other people's.

## Problem Statement

A system has an architecture diagram in its wiki, drawn two years ago by an engineer who has since left. The diagram shows the report service calling a dedicated report database. The report database was decommissioned last year, and the report service now reads from the warehouse database through a shared layer. Nobody updated the picture, because nobody owned it.
A new engineer joins, reads the wiki as the onboarding, and builds a new feature against the old structure. The feature writes to the report database, which does not exist. The build fails, the engineer is confused, and the investigation finds the wiki diagram, which has been wrong for a year and which every new engineer has been trusting.
That is the drift failure, and it is the most expensive mistake in this list, because it is not a drawing error, it is a trust error. A diagram that is wrong is worse than no diagram, because no diagram forces the engineer to read the code, and the wrong diagram invites them not to. The mistake that teams keep making is treating the diagram as a deliverable that has a lifecycle separate from the code, when it is only honest for as long as it matches the code.

## Core Concept

The mistakes group into a small set of failure modes, and it is worth naming each one so you can recognize it on a whiteboard.
The arrow that lies. The direction of a relationship is the diagram's truth, and the most common lie is the arrowhead on the wrong end. The dependency arrow points at the dependency, the thing being used. A `PaymentService` with a `PaymentGateway` field has an arrow from the service to the gateway, because the field's type is the gateway. The triangle of inheritance points at the parent. Get either backwards and the diagram teaches the inverse of the code. The check is one line of Java: the arrow starts at the class that names the other type, and it points at the type it names.

```
public class PaymentService {
    private final PaymentGateway gateway;   // arrow points at PaymentGateway
}
```

The drifted diagram. This is the failure from the problem statement, and it has one reliable cause: the diagram is not regenerated or reviewed when the code changes. The fix that production teams have landed on is to make the diagram text or generated, PlantUML and Mermaid in the repository, reviewed in the same pull request as the code, or a diagram generated from the source. A diagram that lives with the code and diffs like the code cannot drift silently, and a diagram that cannot drift is the only diagram worth having.
The over-drawn diagram. A class diagram with every attribute, every method signature, and every auxiliary class is a diagram that has become a second version of the code, and a second version is guaranteed to drift. The diagram should carry the relationships and the decisions, the multiplicity, the direction, the interface, and nothing else. If a reviewer has to search the box for the relationship, the box is too full.
The under-specified diagram. The opposite failure is the freehand rectangle with an unlabeled line, the diagram that says nothing because it decided nothing. A line between two boxes without a direction, without a label, without a diamond, does not communicate a relationship, it communicates that a relationship exists somewhere. The fix is the arrowhead and the label, which force the decision the freehand line was avoiding.
The wrong diagram. Each diagram type answers one question, and using the wrong one hides the answer. A state diagram for a flow through a system, when the question was the ordering of steps. An activity diagram when the question was which object calls which. An object diagram when the question was the type structure. The check is the question: are you showing conditions an object sits in, or steps it passes through, or instances at a moment, or types and their relationships. Pick by the question, and the diagram follows.
The unowned diagram. A diagram with no owner is a diagram with no update path. When the code changes, the engineer who makes the change does not know the diagram belongs to them, so they do not update it, and it drifts. The fix is the same as the code review: the diagram is an artifact of the change, owned by the person who changed the structure, reviewed with the change.
The deciding-nothing diagram. A diagram produced after the code exists, with no decision riding on it, is archaeology. It records what happened, and it will be wrong within the week. The diagram earns its place only when a design question is on the table and the picture is what settles it. The test: if erasing the diagram would not change a single decision, the diagram should not have been drawn.
The one check that catches most of these is to ask of every element, can this be verified against the code. The arrow can be checked against the field type. The multiplicity can be checked against the schema or the collection type. The interface can be checked against the Java interface. The state can be checked against the enum. If an element cannot be checked, it is either a decision that needs stating, or decoration that needs cutting. A diagram whose every element is checkable is a diagram that is honest by construction.

## Real Production Usage

The production history of UML is mostly a history of fixing the drift problem, and the tooling that survived is the tooling that solved it. PlantUML and Mermaid render diagrams from text, so the diagram lives in the repository, is diffable, and is reviewed in the same merge request as the code that changes the structure. A diagram that is a file in the repo cannot sit in a wiki for two years being trusted, because the code review is the point where the drift gets caught.
ArchUnit takes the same idea further, replacing the diagram with a compiled rule. The dependency constraints, "the web module may not import the repository module," become a test that fails the build, so the architecture diagram is enforced instead of trusted. A team that uses ArchUnit has solved the arrow-that-lies problem permanently for the rules it encodes, because there is no arrow to be wrong.
The generated class diagram, IntelliJ's view of a package as boxes and arrows, is the honest extreme: it cannot lie because it is the code, and it cannot drift because it is regenerated. Its weakness is the same honesty, it records, it does not decide. The production pattern is to use the generated diagram to read the code and the enforced rule or the reviewed text diagram to state the decision, and to keep the two roles separate.

## Common Mistakes

The first mistake is treating the diagram as a deliverable with a life of its own. Teams that produce a diagram and move on have created the drift failure, and teams that worship the diagram have created the over-drawing failure. The diagram is a tool that decides and communicates, and the moment it stops doing either, it should be erased or regenerated.
The second mistake is letting the diagram and the code disagree and deciding that the diagram wins. The diagram is a hypothesis and the code is the test, and when they conflict, the code is the truth. The engineer who updates the diagram instead of the code has made the code a lagging artifact, and the team pays for it in drift. The rule that avoids it: the code changes, then the diagram follows, or the diagram is generated.
The third mistake is drawing a diagram to justify a decision that was already made. A diagram drawn after the fact, with the conclusion baked in, is advocacy, not design. The diagram earns its place when it is drawn before the decision, when the arrow direction is still a question that the drawing answers. The candidate and the team both fail when the picture is used to defend instead of to decide.

## Interview Perspective

Interviewers notice diagram mistakes the way they notice code mistakes, as evidence of how the candidate thinks. The arrow that lies is the most common, a candidate who draws the dependency arrow pointing at the dependent has either misremembered the notation or, more likely, never had to defend the direction, and the interviewer pushes on it.
The weak answer to "why does this arrow point here" is "that's just how I draw it." The strong answer is the check. "The arrow points at the dependency. The service names the gateway in its field, so the arrow starts at the service and points at the gateway, at the thing it uses. If I drew it the other way, the diagram would claim the gateway depends on the service."
The follow-up that tests the drift discipline is "your class diagram and the code you just wrote do not match, which is wrong." The weak candidate says "I'll update the diagram." The strong candidate says "the code is the source of truth, and the diagram is the lagging artifact. I draw the diagram to decide and to communicate, and when it disagrees with the code, the diagram is what I fix, or I regenerate it from the source."
The hardest follow-up is about the wrong-diagram mistake, "why is this a state diagram and not an activity diagram." The strong candidate states the question the diagram answers. "The vending machine's behavior depends on which state it is in, the coin, the selection, the dispensing, so the state diagram is the question. If I were showing the sequence of steps in a checkout flow, that would be an activity diagram, but the machine is about conditions, not steps."

## Knowledge Check

1. A teammate's class diagram shows the payment service with an arrow pointing at the controller, and the comment says "the service uses the controller." State the defect and the check that exposes it.
2. A wiki diagram has not changed in eighteen months, and a new feature was built against it and failed. Name the mistake, the root cause in how the diagram was maintained, and the tooling change that would have prevented it.
3. An interviewer asks why the design uses an activity diagram instead of a sequence diagram. Write the one-sentence answer that states the question each diagram type answers, and the element in your design that made the choice.

## Key Takeaways

- The arrow direction is the diagram's truth, and the check is the Java: the arrow points at the type the class names.
- A diagram that drifted is worse than no diagram, and the fix is text or generated diagrams that live with the code.
- The diagram decides and communicates, and the moment it stops doing either it should be erased.
- Every element should be checkable against the code, and the code is the source of truth when they disagree.

## What's Next

The chapter closes where the handbook turns. You have spent this chapter learning to draw and to read a design before it exists, and the next chapter changes the material: Design Pattern Foundations is no longer about visualizing your own structure, it is about reusing structures that are already proven. The diagrams you have learned are the medium patterns are drawn in, a Strategy pattern is a class diagram with an interface arrow, an Observer is a sequence diagram with a subscription call, and the vocabulary shifts from drawing relationships to naming them. The change is from design as a blank page to design as a library of known answers.

---

This article explains the failure catalog of UML, the lying arrow, the drifted diagram, and the box that over-draws. Its strongest claim is that every diagram element must be checkable against the code, and a drifted diagram is worse than none, because it teaches the wrong structure.
