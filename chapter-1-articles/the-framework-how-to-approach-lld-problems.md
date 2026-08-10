# The Framework: How to Approach LLD Problems

## Learning Objectives

1. Run a repeatable sequence of steps on any LLD problem, from a parking lot to a cache.
2. Explain where each step of the framework maps to a quality you already know: coupling, cohesion, non-functional fit.
3. Keep the framework moving in an interview instead of freezing on the first class.

## Introduction

Everything in this chapter has been leading to a question you will actually face: a problem is handed to you, and you have to produce a design for it, on the spot, out loud. The parking lot, the elevator, the ticketing system, the URL shortener. People call these LLD problems, and they reward one skill above all others: not knowing the answer, but having a way to arrive at it.

That way is a framework. It is a sequence of steps you run every time, so the pressure of the moment does not get to decide the order of your thinking. This article gives you the framework, shows each step next to the principle it rests on, and then points out where the framework breaks, because every framework breaks, and the person who knows where is the one who trusts it correctly.

## Problem Statement

Watch a candidate hit a fresh design problem. They read the prompt, nod, and reach for a class called ParkingLot. They name methods. They draw arrows. And ten minutes in, they realize they never asked who uses the system, or what happens when two cars take the same spot, or whether the requirement was an app for drivers or a management tool for owners. They built a reasonable answer to a problem they never understood.

That is not a knowledge failure. It is a sequence failure. The brain, under time pressure, does what is easiest: it starts with structure, because structure is concrete and visible, and it defers the questions, because questions feel like stalling. The framework exists to force the reverse order. It makes you ask the questions first, on purpose, so that by the time you draw a class you have earned the right to.

## Core Concept

The framework is eight steps. Run them in order. Let each one finish before you start the next, and resist the pull to skip ahead, because the skipped step is where the mistake hides.

1. Restate the problem and the scope.
2. Enumerate the use cases and the actors.
3. Identify the core domain objects.
4. Assign responsibilities to classes.
5. Design the relationships and interfaces.
6. Specify the state and the methods.
7. Stress the edge cases: concurrency, failure, nulls.
8. Re-read the non-functional requirements against the shape.

Diagram: the eight-step LLD framework as a top-down sequence.

```
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 660 720" font-family="Arial, Helvetica, sans-serif">
  <defs>
    <marker id="flowarrow" markerWidth="9" markerHeight="7" refX="8" refY="3.5" orient="auto">
      <polygon points="0 0, 9 3.5, 0 7" fill="#57606a"/>
    </marker>
  </defs>

  <rect x="150" y="40" width="360" height="52" rx="8" fill="#f6f8fa" stroke="#57606a" stroke-width="2"/>
  <text x="330" y="62" font-size="14" font-weight="bold" fill="#24292f" text-anchor="middle">1. RESTATE THE PROBLEM</text>
  <text x="330" y="80" font-size="12" fill="#444c56" text-anchor="middle">scope, actors, what is in and out</text>

  <line x1="330" y1="94" x2="330" y2="120" stroke="#57606a" stroke-width="2" marker-end="url(#flowarrow)"/>

  <rect x="150" y="124" width="360" height="52" rx="8" fill="#f6f8fa" stroke="#57606a" stroke-width="2"/>
  <text x="330" y="146" font-size="14" font-weight="bold" fill="#24292f" text-anchor="middle">2. ENUMERATE USE CASES</text>
  <text x="330" y="164" font-size="12" fill="#444c56" text-anchor="middle">the flows the system must support</text>

  <line x1="330" y1="178" x2="330" y2="204" stroke="#57606a" stroke-width="2" marker-end="url(#flowarrow)"/>

  <rect x="150" y="208" width="360" height="52" rx="8" fill="#f6f8fa" stroke="#57606a" stroke-width="2"/>
  <text x="330" y="230" font-size="14" font-weight="bold" fill="#24292f" text-anchor="middle">3. IDENTIFY CORE OBJECTS</text>
  <text x="330" y="248" font-size="12" fill="#444c56" text-anchor="middle">the nouns from the use cases</text>

  <line x1="330" y1="262" x2="330" y2="288" stroke="#57606a" stroke-width="2" marker-end="url(#flowarrow)"/>

  <rect x="150" y="292" width="360" height="52" rx="8" fill="#f6f8fa" stroke="#57606a" stroke-width="2"/>
  <text x="330" y="314" font-size="14" font-weight="bold" fill="#24292f" text-anchor="middle">4. ASSIGN RESPONSIBILITIES</text>
  <text x="330" y="332" font-size="12" fill="#444c56" text-anchor="middle">one clear job per class</text>

  <line x1="330" y1="346" x2="330" y2="372" stroke="#57606a" stroke-width="2" marker-end="url(#flowarrow)"/>

  <rect x="150" y="376" width="360" height="52" rx="8" fill="#f6f8fa" stroke="#57606a" stroke-width="2"/>
  <text x="330" y="398" font-size="14" font-weight="bold" fill="#24292f" text-anchor="middle">5. DESIGN RELATIONSHIPS</text>
  <text x="330" y="416" font-size="12" fill="#444c56" text-anchor="middle">who holds whom, who calls whom</text>

  <line x1="330" y1="430" x2="330" y2="456" stroke="#57606a" stroke-width="2" marker-end="url(#flowarrow)"/>

  <rect x="150" y="460" width="360" height="52" rx="8" fill="#f6f8fa" stroke="#57606a" stroke-width="2"/>
  <text x="330" y="482" font-size="14" font-weight="bold" fill="#24292f" text-anchor="middle">6. SPECIFY STATE AND METHODS</text>
  <text x="330" y="500" font-size="12" fill="#444c56" text-anchor="middle">fields, signatures, invariants</text>

  <line x1="330" y1="514" x2="330" y2="540" stroke="#57606a" stroke-width="2" marker-end="url(#flowarrow)"/>

  <rect x="150" y="544" width="360" height="52" rx="8" fill="#f6f8fa" stroke="#57606a" stroke-width="2"/>
  <text x="330" y="566" font-size="14" font-weight="bold" fill="#24292f" text-anchor="middle">7. STRESS THE EDGE CASES</text>
  <text x="330" y="584" font-size="12" fill="#444c56" text-anchor="middle">concurrency, failure, nulls, races</text>

  <line x1="330" y1="598" x2="330" y2="624" stroke="#57606a" stroke-width="2" marker-end="url(#flowarrow)"/>

  <rect x="150" y="628" width="360" height="52" rx="8" fill="#f6f8fa" stroke="#57606a" stroke-width="2"/>
  <text x="330" y="650" font-size="14" font-weight="bold" fill="#24292f" text-anchor="middle">8. RE-READ NON-FUNCTIONAL</text>
  <text x="330" y="668" font-size="12" fill="#444c56" text-anchor="middle">does the shape survive the load</text>
</svg>
```

Step by step, and the principle each one leans on.

Restate the problem. Say the problem back in your own words, out loud, and name the scope: who uses it, what is in, what is out. This is the define phase from the design process, and it is the cheapest insurance in the whole framework. A restatement the interviewer corrects is a problem statement that was about to be wrong.

Enumerate the use cases. List the concrete flows: a driver parks, a driver pays, a spot frees up, an admin views occupancy. Actors and flows are the seeds of both classes and non-functional requirements. If you can list the flows, you can later check that your design covers each one, and the check is the point.

Identify the core objects. Pull the nouns out of the flows. ParkingLot, Spot, Vehicle, Ticket, Payment. Do not design yet. Just name the nouns, because they will become the classes, and the nouns you miss here are the classes you will be missing later.

Assign responsibilities. Now give each noun one job and no more. The lot knows its spots. The ticket knows its vehicle and its time. The payment knows its state. This is cohesion applied, and it is the step where the god object is born if you skip it, so do not skip it.

Design the relationships. Decide who holds whom and who calls whom. The lot holds spots. The ticket references a vehicle. The payment service talks to the ticket through an interface. This is coupling applied, and the direction of every arrow here is a decision about what can change without rippling.

Specify the state and methods. Now you can get concrete. What fields does a ticket hold, what can a caller invoke, what invariant must hold after every call. This is where the LLD actually lives, and it only makes sense now that the shape above it exists.

Stress the edge cases. What if two cars take the last spot at once. What if a payment fails halfway. What if a vehicle leaves without paying. This is the step that separates a design from a demo, and it is the step the interviewer is quietly waiting for.

Re-read the non-functional. Bring the requirements back and hold your shape against them. If the design needs to serve two thousand entries an hour, does a plain in-memory count survive the concurrency of step seven? If it does not, loop back to step four or five and change the shape. The loop is the point. The framework is not one pass; it is a spiral, and step eight is where the spiral turns.

The order has a reason and it is not ceremony. It is a dependency order. You cannot assign responsibilities before you have the objects. You cannot design relationships before the responsibilities exist. You cannot specify methods before the relationships are drawn. Each step is a prerequisite for the next, and jumping ahead does not save time, it makes you redo the skipped step at the worst moment, when the shape is already drawn and you are attached to it.

## Common Mistakes

Every framework has a failure, and this one has three you should know.

It breaks when the problem is tiny. A single class with two methods does not need eight steps. The framework is for problems with enough surface to misjudge. For the small stuff, run it at one sentence per step and move on. Rigidity on a toy problem is not discipline, it is noise.

It breaks when the non-functional requirement changes the problem. If the interviewer says "and the system must handle a million users," the framework needs a second lap. Step eight does not just check the shape, it can flip the shape. Do not treat step eight as a verification; treat it as permission to start a new loop.

It breaks when you are asked to implement, not design. The framework produces a design, not code. If the interviewer moves to "write the code," the framework has done its job and you switch modes. Dragging the framework into a coding question is how you talk about ParkingLot for ten minutes without writing the method.

## Interview Perspective

The interviewer on an LLD round is watching for one thing above all: does the candidate have a sequence, or are they improvising. The candidate with a framework restates the problem, lists the flows, and then builds, and every move can be traced to a step. The candidate without one reaches for the first class and spends the rest of the hour defending a shape built on nothing.

The strong answer narrates the steps as it runs them. "Before I name classes, let me list the flows the system must support, because the classes should fall out of the flows, not the other way around." That sentence is worth more than a fluent class diagram, because it shows the order is intentional. The weak answer says nothing about order and hopes the output carries it.

The expected follow-up is the edge case, always. "What if two cars reserve the same spot?" The candidate who has a step for this, stress the edge cases, answers with the mechanism, an atomic reserve, a lock, a state transition. The candidate who has no step improvises and usually discovers a hole in their own shape in front of the interviewer.

## Knowledge Check

1. You are asked to design a movie ticketing system. List the steps of the framework in order, and for each step, one concrete output for this system. The class diagram is not one of the outputs until step five or six. Where does the class diagram actually appear, and why not before?

2. A candidate designs a parking lot and reaches step eight, where the requirement "thousands of check-ins at peak hour" appears. The candidate decides to keep the shape and move on. What specifically breaks, and which earlier step must be revisited, step four or step five, and why?

3. The framework is described as a spiral, not a pass. Give one concrete example of a design where step eight sent you back to step two, changing the use cases rather than the shape. What did the second pass find that the first pass could not?

## Key Takeaways

- The framework is a dependency-ordered sequence: scope, flows, objects, responsibilities, relationships, methods, edges, non-functional.
- Each step is a prerequisite for the next, and the skipped step is where the mistake hides.
- Step eight is a loop back, not a sign-off; non-functional requirements can flip the shape.
- Interviewers reward the visible sequence, the narrated order, over the fluent output.

## What's Next

This chapter has been about design as ideas: decisions, boundaries, requirements, process, qualities, documents. The next chapter changes the register completely. Object-Oriented Programming Fundamentals is where this handbook stops talking about design in the abstract and starts talking about it in the language of the machine: classes, objects, inheritance, polymorphism, encapsulation, and how Java and the JDK actually honor them. The framework you just learned stays, but from here on, "assign responsibilities" and "design relationships" are no longer suggestions, they become specific language mechanics you will implement, and the vocabulary of the next chapter is the vocabulary every LLD answer you ever give will be spoken in.

---

This article condenses the whole chapter into an eight-step framework for attacking any LLD problem, from restating the scope to re-reading the non-functional requirements. Its strongest claim is that the steps are a dependency order, not a checklist, so the skipped step, not the hard one, is where every mistake hides.