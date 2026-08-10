# Requirement Gathering Techniques for Interviews

## Learning Objectives

- Turn the requirements phase from a nervous list of generic questions into a structured extraction of the few answers that actually shape the design.
- Learn the four requirement categories and which ones matter for which kind of system.
- Practice the scope restatement, the twenty-second habit that prevents designing the wrong system for half the interview.

## Introduction

Candidates treat requirements gathering as the insurance policy they hope they never need. They ask one or two questions, sense the interviewer getting impatient, and dive into classes to prove competence. That instinct is backwards and it is expensive. The requirements phase is not a tollbooth on the way to the real work; it is where the case study is won or lost, because every minute you spend designing against assumed requirements is a minute you will not get back when the interviewer corrects you. The uncomfortable truth is that interviewers deliberately leave the problem underspecified to see what you do with the ambiguity. The candidate who charges ahead designs the system the way they interpret the question. The candidate who extracts the requirements designs the system the interviewer had in mind. Only one of those interviews is going well.

## Problem Statement

Consider two candidates on the same parking lot question. The first hears "design a parking lot" and produces a multi-level lot with reservations, hourly billing, and a loyalty program, because that is the parking lot they read about. The interviewer asked for a simple single-floor lot with fixed pricing, and now the candidate is thirty minutes deep in a design for a system nobody wanted. The second candidate asks four questions: single floor or multiple, reservations in scope, pricing model, and who operates the gates. They restate the scope in one sentence, get a correction on pricing, and design exactly the system that was being asked for.

The failure in the first case is not engineering, it is extraction. The candidate answered the question they were asked on the surface and never checked whether their assumptions matched the interviewer's. The requirements phase exists precisely to surface those assumptions before they become forty minutes of wasted drawing. It is the cheapest insurance available, and it costs nothing but a little courage to ask questions.

## Core Concept

Good requirement gathering in an interview is structured, not scattershot. You are not interrogating the interviewer; you are running a short, deliberate extraction. The useful frame has four categories, and almost every question worth asking falls into one of them.

**Actors and actions.** Who uses this system, and what is their primary action? A parking lot has drivers entering and exiting, and a lot operator. A chat system has senders and receivers. A notification system has the business services that publish events and the users who receive messages. The question "who are the actors and what does each one do" forces you to name the flows before you name the classes, and the flows are the design. Most candidates skip this and design from the nouns. Asking it first means your verbs come first, which is the order that works.

**Rules and invariants.** What must always be true? Two bookings cannot share a seat. A payment must move exactly once. Stock cannot go negative. The invariant question is the concurrency question in disguise, and it is the highest-value question in the interview, because the interviewer will test your invariants later. Asking "what are the hard rules here" gives you the exact list the follow-ups will attack.

**Scale and concurrency.** How many users, how many concurrent requests, is this single-user or multi-user? Tic-tac-toe is single-user-pair; a rate limiter is defined by its concurrency. The scale answer decides whether the design is a plain class diagram or something with a queue and a lock in it. Candidates who skip it build single-user systems for multi-user problems and get found out at the first follow-up.

**Scope cuts.** What is explicitly not needed? This is the category candidates never touch, and it is the one interviewers most respect. "I'm assuming no reservations, no multi-floor, no monthly passes, unless you tell me otherwise." Declaring scope cuts is not laziness, it is judgment, and it tells the interviewer you know the difference between the core system and the appendix. Every case study article in this book has a scope-cut line in its requirements section for exactly this reason.

The execution matters as much as the categories. Ask five questions, not forty. Write the answers down. Then do the scope restatement: "so, to confirm, we have a single elevator, up and down calls, no capacity limits, and I should focus on the scheduling. Is that right?" That restatement is the moment the interviewer can correct you cheaply. In the first five minutes, a correction costs you nothing. In the thirty-fifth minute, it costs you the interview.

A worked example makes this concrete. On the elevator, the extraction sounds like this. "Who are the actors? Passengers in the cabin and people on the floors, and the building maintenance staff who would care about diagnostics. For this interview I'll focus on the passengers." That is actors and actions, answered and scoped in one breath. "What are the hard rules? A request for a floor I'm already on is dropped, floor calls carry a direction, and a request must never starve." That is the invariant category, and those are the exact three rules the follow-up questions will probe. "Is this one elevator or several? Because dispatch changes the controller completely." That is scale, asked at the moment it matters instead of as trivia. "And I'll assume no emergency handling and no door timers." That is the scope cut, declared instead of hidden. The whole exchange is under a minute, and the design that follows is constrained by the interviewer's real answers rather than by the candidate's guesses.

There is one interview reality that derails candidates who prepare a fixed script: the interviewer who deflects. You ask "is this single-user or multi-user?" and the answer is "it's up to you, what makes sense?" That is not a dead end, it is a decision handed to you, and the correct response is to take it with a reason. "I'll design for multiple users, because the interesting question in a parking lot is concurrent entry, and a single-user version has no interesting story at all. I'll note that assumption." The candidate who accepts the decision, states the assumption, and moves on has turned a deflection into a demonstration of judgment. The candidate who keeps fishing for an answer the interviewer will not give is burning the clock on a question that was already answered.

## Real Interview Context

The interviewers I know do not score the requirements phase by counting questions. They score it by two signals. The first is whether the questions changed the design. A candidate who asks "single floor or multi-level" and then designs a single floor is asking questions that matter. A candidate who asks "what color should the UI be" is performing. The second signal is whether the candidate treats the interviewer's answers as constraints or as trivia. The interviewer says "reservations are out," and the candidate either designs without them, which is correct, or keeps sneaking them back in, which is how you lose trust in the first ten minutes.

The strongest signal is the restatement. When a candidate says "here is what I'm going to build" and the interviewer agrees, the rest of the interview is cooperative. The candidate has turned the interviewer from an evaluator into a stakeholder with an approved spec. That is the entire game of the requirements phase, and it is available to anyone who will ask the questions out loud.

## Common Mistakes

The most common mistake is asking zero questions and designing from assumptions. The candidate who wants to look fast ends up slow, because the correction comes after thirty minutes of drawing instead of before the first line. Speed is not a requirement-phase virtue; correctness is.

The second mistake is the question dump. The candidate fires off a memorized list of twenty questions, from concurrency to caching to persistence, without listening to any of the answers. The interviewer cannot tell what is load-bearing, and neither can the candidate. Five questions with follow-ups beat twenty questions without.

The third mistake is apologizing for asking. "Sorry, just to clarify, is this multi-user?" The apology signals that the candidate thinks questions are a weakness. They are not. The candidate who apologizes for extracting requirements is the candidate who believes the interview is a performance rather than a collaboration, and that belief leaks into everything else they do.

## Interview Perspective

A weak requirements phase is characterized by the candidate who asks one question, does not write down the answer, and designs something that contradicts it. The interviewer says "no reservations," and the candidate draws a reservation system. The interviewer stops correcting at some point, and the candidate never knows why the interview cooled.

A strong requirements phase is characterized by a candidate who extracts, restates, and then designs exactly what was agreed. When the interviewer adds a twist at minute thirty, the strong candidate says "that changes the spot-allocation rule, here is the seam it lands on," which is only possible because the original scope was nailed down precisely. The follow-ups in a strong interview all read as confirmations of the candidate's own structure.

## Knowledge Check

1. The interviewer says "design a chat system." List the four questions you would ask, and for each one, say what specific part of the design its answer changes.
2. A candidate asks ten questions but designs a system that contradicts the answer to the first one. What is the single habit that would have prevented it, and why does it work?
3. The interviewer says "this is single-user, runs on one machine." Which of the four categories did they just answer, and how does that answer change your design versus a multi-user version?

## Key Takeaways

- Four categories: actors and actions, rules and invariants, scale and concurrency, scope cuts. Ask from them, deliberately.
- Five questions with intent beat twenty questions on autopilot.
- Write the answers down, or you will contradict them at minute thirty.
- Restate the scope in one sentence and invite correction. It is the cheapest fix in the interview.
- Ask until the questions change the design. Unloading trivia is performing, not gathering.

## What's Next

Requirements tell you what the system must do; entities tell you what the system is made of. The next article takes the moment after requirements, the moment where the interviewer is watching you name the nouns, and shows you how to do it fast without guessing.

---

This article explains how to gather requirements around four categories and the twenty-second scope restatement that follows. Its point of view is that the restatement is the cheapest insurance, since a fix in minute five costs nothing and one in minute thirty costs the round.
