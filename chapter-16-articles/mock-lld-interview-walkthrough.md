# Mock LLD Interview Walkthrough: End to End

## Learning Objectives

- See a full forty-five-minute LLD case study the way it actually runs, with the candidate's thoughts and words separated, minute by minute.
- Learn where the time actually goes, and where the candidate who thinks they are fast is actually spending it.
- Practice reading an interviewer's signals, the push, the silence, the twist, and responding to the signal rather than the sentence.

## Introduction

Everything in this book has been about pieces: the framework, the entities, the patterns, the trade-offs, the mistakes. This article assembles the pieces into the whole, in real time. It is a walkthrough of one mock interview, an LLD case study on a system you have already designed, so the attention can go to the process instead of the domain. The point is not the parking lot, or the elevator, or whatever system the mock lands on; the point is the minute-by-minute arc: where the candidate asks, where they draw, where they walk, where they get pushed, and what they say when the push arrives. If you read this with the framework article in one hand, you should be able to see the framework being executed, and the execution is the only thing that matters on the day.

## Problem Statement

The failure this walkthrough prevents is the gap between knowing the framework and running it. Candidates who have read every article in this book still freeze on the day, and the reason is usually not knowledge, it is simulation. They have never run the whole arc under the clock with an interviewer-shaped pressure, so the first time the arc runs, it runs in the interview. This walkthrough is the rehearsal they skipped. Every sentence the candidate says, every question they ask, every trade-off they state, is there to be stolen, adapted, and practiced until the arc is not something you think about but something you do.

## Core Concept

The mock: "Design an elevator system." Interviewer: a senior backend engineer. Format: whiteboard, forty-five minutes. The walkthrough follows the candidate's external words and internal reasoning, because the gap between those two is where the interview is actually won.

**Minutes 0 to 6: the extraction.** The candidate does not draw. They ask. "Who are the actors here?" "Passengers, and people on floors." "What is the hardest rule to satisfy?" "Requests must not starve." "One elevator or several?" "Assume one, you can tell me how dispatch changes." "And I'll cut emergency handling and door timers." Then the restatement: "So I'm building a single elevator where the controller decides the next stop, with floor calls carrying direction, and the interesting part is the scheduling. Correct?" The interviewer nods. Six minutes spent, forty saved. Internally the candidate has already decided the whole structure, because the requirements named the controller, and the controller is the case study.

**Minutes 6 to 12: the entities.** The candidate writes four nouns: `Elevator`, `ElevatorController`, `Request`, `Direction`. One sentence each. "The elevator is the machine, it moves and holds the door. The controller is the brain, it decides where to go. A request is a floor plus a direction or a cabin intent. Direction is the enum that drives everything." The interviewer asks "where is the floor display?" and the candidate runs the responsibility test: "the display reads the controller's state, it does not need to be an entity." The list stays at four. The interviewer is already noting that the split between machine and brain happened before the drawing.

**Minutes 12 to 27: the class design.** This is the bulk, and the candidate spends it unevenly, on purpose. The elevator itself is ten lines and twenty seconds. The controller gets the time, because the controller is the design. The candidate writes the two sorted sets, `upStops` and `downStops`, and explains the SCAN algorithm as they write: "while moving up I only serve floors above, in order, then I reverse and serve the down set. Cabin requests always get served on the direction of travel; floor calls only match their called direction." The interviewer watches the data structures do the scheduling. The candidate names the SCAN quirk before it is asked: "a cabin request for a floor behind us waits for the return sweep, that is the cost of predictability." The interviewer has nothing left to ask about the algorithm, because the algorithm was already explained.

**Minutes 27 to 34: the walkthrough.** The candidate stops writing and runs one full scenario. "The elevator is on floor 3, idle. A passenger on floor 5 presses up, so floor 5 goes in the up set. A cabin request arrives for floor 7, also up. The controller serves 5, then 7. At 7, the passenger presses 2, which is below, so it goes in the down set. The up sweep is empty, the controller reverses, and serves 2." The interviewer follows along, and the walkthrough is a demonstration, not a recital, because every step names the exact method and the exact set. This is the part that most candidates skip, and this mock demonstrates why skipping it is a mistake: it is the part where the design proves itself.

**Minutes 34 to 40: the push.** The interviewer makes the move every elevator interview ends with. "What if there are two elevators?" The candidate does not freeze and does not redraw. "The controller owns a list instead of a single elevator, and dispatch picks which one takes a floor call. The cheap policy is nearest idle elevator. If I wanted smarter, that is a Strategy seam on the dispatcher, but for this interview I would keep it nearest." The interviewer pushes once more. "What if both elevators are moving?" The candidate: "then dispatch picks the one heading the same direction, and the SCAN structure per elevator does not change." The push is absorbed because the seams were placed when the design was drawn, not when the question landed.

**Minutes 40 to 45: the close.** The candidate summarizes the two decisions that carried the design, the machine-brain split and the direction-keyed sets, and names where they would go with more time, "priority to cabin requests over floor calls, and the capacity limit for the follow-up." The interviewer asks if the candidate has questions, the candidate asks what the team uses for scheduling in production, and the round ends on a collaboration. The candidate leaves with the interviewer having seen the extraction, the split, the algorithm, the walkthrough, and the trade-off, which is the entire rubric.

## Real Interview Context

The mock reflects the real rubric in three ways worth making explicit. First, the time allocation: six minutes on requirements, six on entities, fifteen on design, seven on walkthrough, five on close, matches the framework article almost exactly, and the mock shows why, because every minute the candidate spent asking was a minute the candidate did not spend redrawing. Second, the interviewer's silence is a signal. When the candidate explained the SCAN quirk before being asked, the interviewer went quiet, not because the answer was wrong but because there was nothing to add, and the candidate read that silence correctly as approval rather than confusion. Third, the push is a gift. The two-elevator question was always coming, and the candidate who had the controller owning a single elevator was one field change from the answer, which is exactly why the machine-brain split was placed where it was. Interviewers push at the seams they expect you to have built.

## Common Mistakes

The first mistake this mock avoids is spending the entity phase drawing. The candidate named four nouns in under a minute and moved on, while the typical candidate is thirty minutes in before the controller exists. The second mistake it avoids is the walkthrough-as-afterthought. The mock spends seven dedicated minutes tracing one scenario, and that is the seven minutes that makes the design real to the interviewer. The third mistake it avoids is answering the two-elevator question by redrawing the whole system. The answer was a field change and a seam, because the design was built to absorb exactly that question. If you read the mock and your own practice does not resemble it, the gap is not the domain, it is the process.

## Interview Perspective

A weak mock is the same problem solved in the wrong order: forty minutes of drawing, two minutes of explaining, no walkthrough, and a two-elevator push that produces a redraw and a shrug. The interviewer leaves with no proof the design works, because the candidate never demonstrated it. A strong mock is what you just read: extraction up front, a machine-brain split, the algorithm named as it is drawn, a full walkthrough, and a push absorbed at a seam. The difference between the two is not intelligence and it is not domain knowledge. It is rehearsal. The strong candidate has run this arc twenty times on different domains, and on the day it runs itself.

## Knowledge Check

1. In the mock, the candidate spent six minutes on requirements. Defend that allocation against the objection that it is wasted time, using the minute-by-minute consequences shown in the walkthrough.
2. The interviewer went quiet after the SCAN quirk explanation. What is the candidate reading in that silence, and what is the risk of misreading it as disapproval?
3. The two-elevator push was answered with "the controller owns a list" rather than a redraw. Identify which earlier design decision made that answer possible, and what the answer would have looked like without it.

## Key Takeaways

- The arc is the skill: extract, entity, design, walkthrough, push, close, and the time split is roughly six, six, fifteen, seven, five.
- The machine-brain split is what makes the two-elevator push a one-line answer instead of a redraw.
- Name the algorithm and the quirk before being asked. The interviewer's silence is approval, read it correctly.
- The walkthrough is a demonstration, not a recital. Seven minutes of tracing one scenario is the design proving itself.
- The push is a gift at the seam you built. Absorb it, never redraw for it.

## What's Next

The walkthrough ends with the candidate asking the interviewer a question, which is the first move of the next article's subject. After the interview is over, the learning begins, and how you close that loop determines whether the round was a data point or a turning point.

---

This article explains a full mock LLD interview on an elevator, minute by minute, from extraction to the two-elevator push. Its point of view is that the machine-brain split is what turns the interviewer's hardest follow-up into a one-line answer instead of a redraw.
