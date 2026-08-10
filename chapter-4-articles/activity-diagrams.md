# Activity Diagrams

## Learning Objectives

1. Draw a flow with the four core elements, initial node, action, decision diamond, and final node, with guards on every branch.
2. Show parallelism with fork and join bars, and say when a fork is honest and when it is a lie about the code.
3. Choose between an activity diagram and a sequence diagram by asking whether the flow belongs to the system or to one interaction.

## Introduction

The activity diagram is the flow itself, drawn without caring which object runs each step. It answers a different question than the sequence diagram. The sequence diagram asks "who calls whom, in what order." The activity diagram asks "what happens, in what order, and where does it branch." It is the diagram you reach for when the interesting thing is the flow, the decisions, the parallel work, and the objects that happen to execute the steps are a detail.
That makes it the most business-readable diagram in the set. A product owner can follow an activity diagram. A developer can verify it against the code. The same picture serves both, which is exactly what a good design artifact should do.

## Problem Statement

An order fulfillment flow is specified in a requirements doc, as prose. It reads: "When an order is placed, validate it, and if it is valid, charge the customer, then confirm the order, and also send the confirmation email and update inventory. If it is not valid, reject it." Everyone nods. The team builds it, and two things go wrong.
The first is the parallelism. "And also send the email and update inventory" was read as "then send the email and update inventory," so the engineer wrote the email send and the inventory update as sequential steps after the charge. Every order now waits on a slow email provider before the response returns, and the team cannot explain why the checkout is slow, because the sentence did not say which steps run in parallel and which must complete before the order is confirmed.
The second is the branch. The doc says "if it is valid, charge, else reject." The engineer implemented the `if` and the happy path, and the `else` branch, the reject, was never drawn or tested, because prose buried it in a comma. The first declined payment, three weeks later, takes the wrong path, and the bug is found in production by a customer.
Both failures are flow failures. The sequence diagram would not have caught them, because the sequence diagram is about one interaction between specific objects and does not naturally show that the email and the inventory update are independent of each other. The activity diagram is the diagram that makes both facts visible: the fork that says these two run in parallel, and the decision with two labeled branches that says every path is designed and tested.

## Core Concept

The activity diagram has a small vocabulary, and each symbol carries one fact.
The initial node is a filled circle where the flow starts. There is exactly one. The final node is a filled circle with a ring around it, and a flow can have more than one, one per exit path. The action is a rounded rectangle, one atomic step. The decision is a diamond with one arrow in and multiple arrows out, each out-arrow labeled with a guard, the condition that selects that path. The merge is a diamond with multiple arrows in and one out, where the branches rejoin without a decision. The fork is a thick bar with one arrow in and several out, and the join is the thick bar where parallel flows wait for each other before continuing.
The rule that keeps a decision honest: every out-arrow gets a guard, and the guards cover the possibilities. A decision "order valid?" with a `yes` branch and a `no` branch is complete. A decision with a `yes` branch and an unlabeled exit is a bug drawn in advance, the same comma-buried `else` from the problem statement.
The fork and join are where the activity diagram earns its uniqueness. A fork says the outgoing flows are independent; they can run in parallel, and no arrow between them implies ordering. A join says the incoming flows must all complete before the flow continues. The Java for a fork is an executor handing two tasks to threads, and the join is the code waiting on both futures before proceeding.

```
ExecutorService pool = Executors.newFixedThreadPool(4);
void fulfill(Order order) {
    try {
        Future<Void> email = pool.submit(() -> emailService.sendConfirmation(order));
        Future<Void> inventory = pool.submit(() -> inventoryService.reserve(order));
        email.get();
        inventory.get();
    } catch (InterruptedException | ExecutionException e) {
        throw new FulfillmentException(e);
    }
}
```

The fork is the two `submit` calls. The join is the two `get` calls, which block until both tasks finish. The moment the flow is drawn as a fork and a join, the reviewer sees that the email and the inventory run concurrently and that the flow waits on both. The same picture shows where the parallelism is missing, which is the bug the prose introduced: a flow drawn without a fork is a flow where every step is sequential, and the diagram makes that claim visible.
The distinction from the sequence diagram is the first thing to internalize, because it decides which diagram you draw. The sequence diagram shows one interaction between specific objects, a request flowing through a controller, a service, a gateway. The activity diagram shows a flow wherever it lives, and it does not name the objects unless you put them in swim lanes. Ask which is the interesting fact: if it is the call order between objects, draw a sequence diagram. If it is the flow with its branches and parallelism, draw an activity diagram. A payment flow with a fraud check and parallel notifications is an activity diagram. The exact request path through Spring's filters is a sequence diagram.
Swim lanes are the optional layer that brings ownership back. An activity diagram drawn with vertical lanes, one per component, and each action placed in the lane that runs it, is an activity diagram that also says who does what. The swim lanes turn "charge the customer" into "the payment service charges the customer," and they are the usual way teams combine flow with ownership when a whiteboard has room for both.
Diagram: the payment flow as an activity diagram, with decision diamonds and a parallel fork and join.

```
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 810" font-family="Arial, Helvetica, sans-serif">
  <defs>
    <marker id="flow" markerWidth="10" markerHeight="8" refX="9" refY="4" orient="auto">
      <polygon points="0 0, 10 4, 0 8" fill="#57606a"/>
    </marker>
  </defs>
  <circle cx="450" cy="40" r="16" fill="#24292f"/>
  <line x1="450" y1="56" x2="450" y2="88" stroke="#57606a" stroke-width="1.5" marker-end="url(#flow)"/>
  <rect x="350" y="90" width="200" height="50" rx="4" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="450" y="120" font-size="13" fill="#24292f" text-anchor="middle">Validate order</text>
  <line x1="450" y1="140" x2="450" y2="163" stroke="#57606a" stroke-width="1.5" marker-end="url(#flow)"/>
  <polygon points="450,165 520,210 450,255 380,210" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="450" y="214" font-size="13" fill="#24292f" text-anchor="middle">Order valid?</text>
  <line x1="450" y1="255" x2="450" y2="298" stroke="#57606a" stroke-width="1.5" marker-end="url(#flow)"/>
  <text x="470" y="280" font-size="12" fill="#57606a" text-anchor="middle">yes</text>
  <line x1="520" y1="210" x2="650" y2="210" stroke="#57606a" stroke-width="1.5" marker-end="url(#flow)"/>
  <text x="584" y="198" font-size="12" fill="#57606a" text-anchor="middle">no</text>
  <rect x="350" y="300" width="200" height="50" rx="4" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="450" y="330" font-size="13" fill="#24292f" text-anchor="middle">Authorize payment</text>
  <line x1="450" y1="350" x2="450" y2="373" stroke="#57606a" stroke-width="1.5" marker-end="url(#flow)"/>
  <polygon points="450,375 520,420 450,465 380,420" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="450" y="424" font-size="13" fill="#24292f" text-anchor="middle">Approved?</text>
  <line x1="450" y1="465" x2="450" y2="518" stroke="#57606a" stroke-width="1.5" marker-end="url(#flow)"/>
  <text x="470" y="490" font-size="12" fill="#57606a" text-anchor="middle">yes</text>
  <line x1="520" y1="420" x2="650" y2="420" stroke="#57606a" stroke-width="1.5" marker-end="url(#flow)"/>
  <text x="584" y="408" font-size="12" fill="#57606a" text-anchor="middle">no</text>
  <rect x="370" y="520" width="160" height="8" fill="#24292f"/>
  <line x1="370" y1="524" x2="240" y2="558" stroke="#57606a" stroke-width="1.5" marker-end="url(#flow)"/>
  <line x1="530" y1="524" x2="660" y2="558" stroke="#57606a" stroke-width="1.5" marker-end="url(#flow)"/>
  <rect x="140" y="560" width="200" height="50" rx="4" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="240" y="590" font-size="13" fill="#24292f" text-anchor="middle">Send confirmation email</text>
  <rect x="560" y="560" width="200" height="50" rx="4" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="660" y="590" font-size="13" fill="#24292f" text-anchor="middle">Update inventory</text>
  <line x1="240" y1="610" x2="370" y2="652" stroke="#57606a" stroke-width="1.5" marker-end="url(#flow)"/>
  <line x1="660" y1="610" x2="530" y2="652" stroke="#57606a" stroke-width="1.5" marker-end="url(#flow)"/>
  <rect x="370" y="650" width="160" height="8" fill="#24292f"/>
  <line x1="450" y1="658" x2="450" y2="689" stroke="#57606a" stroke-width="1.5" marker-end="url(#flow)"/>
  <rect x="350" y="690" width="200" height="50" rx="4" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="450" y="720" font-size="13" fill="#24292f" text-anchor="middle">Confirm order</text>
  <line x1="450" y1="740" x2="450" y2="754" stroke="#57606a" stroke-width="1.5" marker-end="url(#flow)"/>
  <circle cx="450" cy="770" r="14" fill="#ffffff" stroke="#24292f" stroke-width="2.5"/>
  <circle cx="450" cy="770" r="5" fill="#24292f"/>
  <rect x="650" y="185" width="180" height="50" rx="4" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="740" y="215" font-size="13" fill="#24292f" text-anchor="middle">Reject order</text>
  <line x1="740" y1="235" x2="740" y2="264" stroke="#57606a" stroke-width="1.5" marker-end="url(#flow)"/>
  <circle cx="740" cy="280" r="14" fill="#ffffff" stroke="#24292f" stroke-width="2.5"/>
  <circle cx="740" cy="280" r="5" fill="#24292f"/>
  <rect x="650" y="395" width="180" height="50" rx="4" fill="#f6f8fa" stroke="#57606a" stroke-width="1.5"/>
  <text x="740" y="425" font-size="13" fill="#24292f" text-anchor="middle">Notify decline</text>
  <line x1="740" y1="445" x2="740" y2="484" stroke="#57606a" stroke-width="1.5" marker-end="url(#flow)"/>
  <circle cx="740" cy="500" r="14" fill="#ffffff" stroke="#24292f" stroke-width="2.5"/>
  <circle cx="740" cy="500" r="5" fill="#24292f"/>
</svg>
```

Read it the way a reviewer reads a flow. The start node feeds the validation. The first decision has a labeled `yes` and a labeled `no`, so the reject path is visible and the reviewer can ask whether it is tested. The `yes` path charges, then hits the second decision, `Approved?`, again with both guards. The `no` of the second decision leads to the decline notification. The `yes` path reaches the fork bar, which splits into the email and the inventory update. The join bar waits for both, and only then is the order confirmed and the flow ended. Two facts are now on the page that the requirements doc hid: the email and the inventory are parallel, and every branch has a designed exit.
The fork is the detail to check most carefully, because it is the most often lied about. A fork drawn where the code runs sequentially is a diagram that claims parallelism the system does not have. A flow drawn without a fork where the code uses an executor is a diagram hiding parallelism the system does have. Both are the same failure, the diagram and the code disagreeing, and the activity diagram's unique job is to make concurrency visible. When you draw the fork, you should be able to point at the `submit` call that creates it.

## Real Production Usage

The activity diagram is the closest thing the industry has to a standard business-process notation. BPMN, used by workflow engines like Camunda, is an activity diagram with a formal grammar around it, and a BPMN model deployed to a workflow engine is an activity diagram that runs. The decision diamonds become exclusive gateways and the forks become parallel gateways, and the engine executes the picture. If you have ever read a BPMN process definition, you have read an activity diagram without the name.
AWS Step Functions and other workflow orchestrators are the same idea in infrastructure form. A state machine definition with `Choice` states and `Parallel` states is an activity diagram expressed as JSON, and teams write those definitions every day. The mental model carries over directly: a `Choice` is a decision diamond with guards, a `Parallel` is a fork and join.
The other production home is test design. Decision and branch coverage, the test-engineering practice of covering every guard and every branch, is applied directly against activity diagrams. A flow drawn with two decisions and four guarded branches tells the tester exactly how many paths exist and which ones no test touches. The diagram is not only a design artifact, it is a coverage checklist.

## Common Mistakes

The first mistake is drawing an activity diagram when the question is about object interactions. The reflex is to put a sequence of steps in a flow and call it done, but if the interesting fact is which object calls which, the activity diagram is the wrong shape, and the ownership is being hidden instead of shown. The test: can you name the one interaction the flow belongs to? If yes, it may be a sequence diagram. If the flow spans many participants, keep the activity diagram and consider swim lanes.
The second mistake is the unlabeled branch. A decision diamond with an exit arrow that has no guard is a hole in the design, and it is the comma-buried `else` again. Every out-arrow of a decision gets a guard, and the guards must cover every case. The reviewer's first question at a decision diamond is "what is the other path," and the answer must be visible on the page.
The third mistake is forks without joins or joins without forks. A fork that splits and never rejoins is a flow that loses threads. A join drawn where the code does not wait is a diagram that claims blocking that does not exist. The fork and the join are a matched pair, and the pair must match the executor and the `get()` in the code.

## Interview Perspective

Interviewers use activity diagrams for flow-heavy design questions, the ones where the shape of the flow is the point. An OAuth flow, a checkout with a fraud check, a refund pipeline, these are questions where the candidate who draws the flow communicates in seconds what the candidate who talks takes minutes to convey. The strong candidate draws the start, the first decision, the parallel steps with a fork, the join, and the final node, and then walks the interviewer along the picture.
The weak answer is the prose monologue of steps, "first we validate, and then if it passes we do the payment, and we also send an email, and then we confirm," and the interviewer is left to rebuild the branching and the parallelism from the stream. The strong answer draws the decision with labeled guards and the fork for the parallel work, and the interviewer can point at a branch and ask about it.
The follow-up that shows depth is "when would you draw an activity diagram instead of a sequence diagram." The weak answer guesses. The strong answer states the test. "When the interesting thing is the flow, the branches, and the parallel work, I draw an activity diagram. When the interesting thing is which object calls which and who blocks, I draw a sequence diagram. The activity diagram does not care who runs the step, the sequence diagram is all about who runs the step."

## Knowledge Check

1. A flow description says "process the order, and at the same time notify the customer and update the warehouse, then confirm." Draw the minimal activity diagram for the sentence, and state which element the words "at the same time" map to.
2. An activity diagram has a decision diamond with a `yes` exit labeled and an unlabeled second exit. Name the defect and the two questions a reviewer should ask at that diamond.
3. You inherit code that runs the email send and the inventory update sequentially, but the team's activity diagram shows them under a fork and join. State which artifact is lying, what the code should look like to match the diagram, and the Java construct that creates the parallelism.

## Key Takeaways

- The activity diagram is the flow without an owner, and the decision, the fork, and the join are its reason to exist.
- Every decision branch gets a labeled guard, and the guards cover every path, or the diagram is hiding a bug.
- The fork and join must match the code, an executor and its waits, or the diagram lies about concurrency.
- Ask whether the flow or the interaction is the point, and that answer picks between the activity and the sequence diagram.

## What's Next

The activity diagram showed a flow moving through branches and parallel paths. The state diagram looks at one object from the inside, at the states it can be in and the events that move it between them. The next article covers the states, the transitions, the initial and final states, and why the state diagram is the only diagram that models an object's life.

---

This article explains the activity diagram as the flow without an owner, with guarded decisions and fork-and-join bars that make parallelism visible. Its strongest claim is that an unlabeled branch is a hidden else, and a fork without the matching executor is a diagram lying about concurrency.
