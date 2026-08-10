# Common LLD Interview Mistakes

## Learning Objectives

- Learn the catalogue of failures that cost candidates the round, grouped so the pattern behind each one is visible.
- Distinguish the mistakes of the drawing from the mistakes of the posture, and see why the latter are the ones that actually sink candidates.
- Build the self-review habit that catches the top mistakes before the interviewer does.

## Introduction

Every LLD interview that goes badly goes badly in a few repeatable ways. The specific wrong class, the specific missed race, the specific rushed walkthrough, they all rhyme across candidates, and they all rhyme across case studies. That is good news, because it means the mistakes are not infinite, and most of them are preventable with a little rehearsal and a self-review habit. This article is the catalogue. It pulls together the failures scattered through every case study in this book, groups them so you can see the pattern underneath, and gives you a short self-review you can run at the end of any practice session. None of this is new material; it is the diagnosis of everything you have already read, written down in one place so you can check yourself against it.

## Core Concept

The mistakes fall into three groups, and it is worth seeing the grouping because the groups have different cures.

**Group one: the drawing mistakes.** These are the failures of the class design itself, and every one of them appeared in a case study in this book. The merged entity, a book that is also its copies, a product that owns its stock, a vehicle that owns its availability. The missing record, a system with no loan, no ticket, no ledger, no reservation, so no single object can answer "what is happening right now." The ownership error, an ATM that holds a balance, a cart that holds stock, a spot that decides where to go. The forced pattern, a visitor on a parking lot, a state hierarchy on tic-tac-toe. The unguarded transition, a status you can set from anywhere, which is how a payment gets double-charged or a seat gets double-sold. These mistakes are all the same root cause wearing different names: the candidate modeled the nouns without modeling the verbs, and the verbs, the ticket, the loan, the ledger, the reservation, the atomic check, are the design.

**Group two: the sequencing mistakes.** These are failures of the order of operations, and they are the second-most common way the walkthrough dies. The ordering error, checking the win before resolving the snake, debiting before checking the cassette, recording the movement before applying it, checking availability and assigning in two unguarded steps. The skipped phase, requirements when there was time for two questions, scope cuts when the interviewer is waiting to hear them, a walkthrough when the interviewer has to ask for one. The phase bleed, designing entities before asking what the system does, drawing classes before the scope is agreed, answering a follow-up about reservations in the middle of the entry flow. Every one of these is a time-sequencing failure, and every one of them is fixed by the same habit: follow the framework's order, and do not start a phase until the previous one is closed.

**Group three: the posture mistakes.** These are the failures that have nothing to do with the drawing, and they are the ones that lose the round even when the design is good. Folding, abandoning a defensible position at the first push. Rigidity, refusing to move when the suggestion is better. The apology, "sorry, is this multi-user?", which signals that questions are weakness. The performance, designing for the level instead of the requirements, so the room reads the pattern-hoarding for what it is. The silence, finishing the drawing and saying "and then it works" instead of walking the flow. These mistakes read as character to the interviewer, which is worse than reading as inexperience, because inexperience is fixable with practice and character is judged as fixed.

## Real Interview Context

The interviewers who run these rounds do not grade mistake-by-mistake; they grade the recovery and the pattern. A candidate who makes a drawing mistake and catches it, "wait, the win check has to run after the jump resolves, let me fix that," has turned a mistake into a demonstration. A candidate who makes the same mistake and hides it has turned a mistake into a failure. And a candidate who makes the same mistake in two different systems across two rounds is showing the interviewer a pattern that will repeat at work. The mistakes in this catalogue matter less than what they reveal, and what they reveal is whether the candidate runs a self-review, which is the difference between someone who improves and someone who repeats.

It is worth watching one bad round compound, because the compounding is the real story. The candidate hears "design a movie ticket booking system" and skips requirements to look fast, which is the sequencing mistake, so the design assumes a single hall and no hold window. Because there is no hold window, the seat is a boolean, which is the drawing mistake, so the abandoned checkout cannot exist and there is no race to find. The interviewer asks "what happens when someone pays late?" and the candidate invents a status field mid-design, the unguarded setter, and now the walkthrough has to be redrawn on the spot. Three mistakes, one in each group, each one making the next one worse, all of them traceable to the first skipped phase. That is how a round collapses: not in one dramatic failure, but in a cascade where each mistake gives the next one a place to live. The self-review catches the first one, and catching the first one stops the cascade.

## Common Mistakes

The top mistakes, ranked by how often they actually cost the round, in no particular order but with the ones that recur most often first.

Skipping requirements to look fast. The candidate charges into classes to prove competence and designs the wrong system. It is the number one mistake in this book and it is in the first article of the chapter for a reason.

The missing concurrency story. The candidate designs a single-threaded system for a problem that is obviously concurrent, and the interviewer has to ask about the race. Finding the race before being asked is the difference between a pass and a strong pass, and skipping it entirely is a fail.

The god class. One class with the board, the players, the rules, the display, and the retry logic. The interviewer's "how do I add a feature" is unanswerable because every feature touches everything.

The untested walkthrough. The candidate presents a design they have never traced, and the first time a request moves through their own classes is in front of the interviewer, out loud, where every hole is public.

The dead pattern. A pattern with no variation behind it, defended with its name instead of its reason.

The unguarded setter. `setStatus`, `setAvailable`, `setStock`, called from anywhere, so the invariants the design depends on are open to any caller.

And the umbrella mistake that contains most of them: no self-review. The candidate who finishes the design, walks it once, checks the race, checks the ordering, and asks themselves "what did I assume?" does not make most of the mistakes in this list, because the review catches them before the interviewer does.

The self-review is also the cheapest practice known to LLD preparation. It takes ninety seconds at the end of any mock, it requires no interviewer, and it compounds: every session that ends with a written list of the three mistakes made is a session that makes the next one better. Candidates who mock without reviewing are repeating the same session forever. Candidates who review are subtracting a mistake each time. The gap between those two candidates after ten mocks is the whole difference between a nervous round and a confident one.

## Interview Perspective

A weak round is one mistake stacked on another, each one discovered by the interviewer, none of them caught by the candidate. The interviewer ends the round having done the candidate's review for them, and the feedback session starts with the candidate discovering there were problems, which is the worst possible position to discover them.

A strong round is one where the interviewer's role is confirmatory. The candidate makes a small mistake, catches it, names the fix, and moves on, and the interviewer spends the round watching a self-review happen in real time. The strongest candidates volunteer the mistakes they could have made, "the ordering here is the trap, roll before resolve before win, and I almost got it wrong," which is not bravado, it is showing that the review is built into the process. The candidate who reviews themselves out loud is the candidate who does not need the interviewer to find the flaws, and that is the candidate worth hiring.

## Knowledge Check

1. A candidate's design for a vending machine has no state concept and a boolean for money collected. Run the three groups against it and name which group contains the failure and what the fix is.
2. A candidate catches their own ordering mistake mid-walkthrough and fixes it in front of the interviewer. Explain why this is a pass, not a failure, and what the interviewer is actually observing.
3. You finish a parking lot design and have two minutes left. Give the self-review checklist you would run, and name which mistakes from the catalogue each check targets.

## Key Takeaways

- The drawing mistakes are the missing verbs: no ticket, no loan, no ledger, no atomic check. Model the verbs.
- The sequencing mistakes are the wrong order: phases skipped, checks misordered, scope unagreed. Follow the framework.
- The posture mistakes are the silent ones: folding, rigidity, apology, performance. They read as character.
- Run the walkthrough before the interview does. An untraced design is an unbuilt design.
- The self-review is the umbrella cure: walk it, check the race, check the ordering, ask what you assumed.

## What's Next

The mistakes are best caught in rehearsal, and rehearsal is the subject of the next article. A mock interview is where everything in this book gets practiced, and the walkthrough article shows you, minute by minute, what a strong session actually looks like from the first question to the last.

---

This article explains the failures that lose LLD interviews, from missing records and misordered checks to folding under a push. It argues that nearly all of them trace back to one thing: no self-review.
