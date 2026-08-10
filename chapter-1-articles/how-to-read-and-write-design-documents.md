# How to Read and Write Design Documents

## Learning Objectives

1. Write a design document that records decisions and their reasoning, not a recap of the code.
2. Read a design document and extract what the author actually committed to, including what they quietly avoided.
3. Know the sections that decide a doc's quality, and the sections that are decoration.

## Introduction

A design document is the design made legible to other people. You have already done the design work: you have decided the pieces, the boundaries, the trade-offs. The document exists so a reviewer can check those decisions and so a future engineer can reconstruct them. Both audiences are reading the same text, and both are looking for the same thing: not what the system does, but why it is shaped the way it is.

Most design documents fail at this. They describe the structure, and they omit the reasoning. A reader who can see the classes but not the decisions behind them has a diagram of a system they still do not understand. The job of the writer is to leave the decisions behind. The job of the reader is to find them.

## Problem Statement

A design document lands in review. It is forty pages long and thorough. The reviewer opens it, scans for the decision that matters, and cannot find it. The doc describes every class and every method, and nowhere does it say why the system uses an event log for order changes instead of a database write, or why the state machine lives in one service and not another. The reviewer is expected to approve a system they cannot evaluate.

The doc is not useless, it is worse than useless. It looks complete, so nobody opens the right question. The team approves the shape, ships it, and the architectural decision the doc never stated quietly binds everyone for the next two years. The document's failure was not missing pages. It was missing the decisions, which is the only thing it existed to carry.

## Core Concept

A design document has two audiences and two jobs. The review audience wants to say yes or no to a shape with all the context that makes the shape defensible. The maintenance audience, two years later, wants to know why a decision was made so they can change it without repeating the original mistake. Both jobs point to the same content: the decisions, the reasons, and the roads not taken.

A good document does not describe what was built. It records what was decided and why. "The order service owns the state machine" is a description. "The order service owns the state machine because payment can arrive in any order relative to fulfillment, and the state machine is the only place a consistent transition can be enforced" is a decision. One sentence carries the reasoning, and that reasoning is what the reviewer checks and what the future engineer rebuilds from.

The sections of a real design doc tend to repeat across companies, and each one has a job:

Section | Its job
--- | ---
Context and problem | The one problem being solved, with enough background that a stranger can evaluate it
Goals and non-goals | What success looks like, and the things you are explicitly not trying to do
Proposed design | The chosen shape, at the level the decision actually lives at
Alternatives considered | The shapes you rejected, and the specific reason each was rejected
Trade-offs and risks | What the chosen shape gives up, and what could break
Open questions | The decisions still unsettled, and who owns them

The non-goals section is the one most teams skip, and it is the one that separates a real doc from a press release. Listing what you are not doing forces the author to draw a line around the scope, and it protects the review from a reviewer who says "what about also handling X" for a scope nobody agreed to. A doc with no non-goals is a doc with no boundary, and a review of it is a free-for-all.

Alternatives are what make a doc trustworthy. Anyone can present one design. Presenting the design you rejected, and the exact reason you rejected it, lets the reviewer re-litigate the decision with you instead of only ever seeing the survivor. The rule: every alternative you list needs one concrete reason it lost. A doc that lists alternatives with generic reasons is not trying to be honest, it is trying to look thorough.

Now the reading side, which gets less teaching and matters just as much. You read a design doc the way you read a contract: you look for what is committed and what is quietly missing. Start with the non-goals, because that is the true scope. Then read the alternatives, because that is the true reasoning. Then find the trade-offs section and check whether the doc admits a real cost. A doc that lists only benefits is a doc you cannot trust. The absence of a trade-off is itself a finding, and it is often the one worth raising at review.

A reader has to separate description from commitment. "We will" is a commitment. "We could" is a possibility. "We considered" is history. When a doc reads "we considered a message broker but chose a database," the reader should ask: who chose, and what changed their mind. If the answer is a named reason, the decision is real. If the answer is silence, the choice may not have been a choice at all, it may have been a default that nobody reviewed.

Two more rules for the writer. Write for the reviewer who has not lived in your head for three weeks. That means stating the problem, the constraint, and the decision, in that order, in plain language before any diagram. And write the reasoning next to the decision it explains, not in a separate "Rationale" section at the bottom, because a reader will not reassemble the doc for you. When the decision and the reason are separated, the reason goes unread, and the doc collapses back into a diagram.

## Real Production Usage

The most famous design document in the industry is the RFC, the Request for Comments, the format the Linux kernel community and the Apache projects and countless teams use for exactly this purpose. An RFC is a design document with a review discipline attached: proposals go out, the community comments on the reasoning, the author revises, and the document either gets adopted with its decisions recorded or it dies. The point of the format is not the template, it is that the decision and its context are reviewed in public and kept as history.

Inside a company, the pattern usually appears as the ADR, the Architecture Decision Record. An ADR is a short document that records one decision: the context, the decision, and the consequence. Its whole value is that it is cheap, so it actually gets written, and one decision per document, so the record stays searchable. If you keep nothing else, keep the ADR shape: a decision with its reason, dated and owned.

The lesson both formats teach is that the document is only as good as the discipline around it. An RFC that nobody comments on is a memo. An ADR that nobody reads is a file. The document does not produce the review; the review produces the decision, and the document preserves it. That is why the writing rule matters: if the reasoning is not in the doc, the review cannot do its job, and the preservation job fails.

## Common Mistakes

The first mistake is writing the document after the code is done. By then the interesting decisions have been made, forgotten, or rationalized, and the doc becomes a description of what exists with invented reasons. Write the document while the decisions are still live, while the review can still change them. A doc that cannot change anything is a diary.

The second mistake is describing the design at the wrong level. Some documents are all boxes and arrows, with every component drawn and no decision stated. Others are all rationale, with the shape so vague that a reviewer cannot tell what the system even is. The right level is the level where the decision lives. If the decision is which service owns a state machine, the doc draws that. If the decision is how a method handles a null, the doc is a few lines on the class, not a diagram of the cluster.

The third mistake is a doc with no opinions. A design document that presents one option without alternatives, with no non-goals and no trade-offs, reads like a description of an inevitable outcome. Reviews need friction. The doc that presents its choice as inevitable is not design, it is marketing, and it will be approved without ever being understood.

## Interview Perspective

Interviewers rarely hand you a real design doc, but they watch two proxies. The first is whether you structure your answer the way a doc would be structured: problem, goals, non-goals, design, alternatives, trade-offs. The candidate who says "here is what I am not building, and here is the alternative I rejected" is running a doc review inside the interview. The candidate who only ever presents the survivor is writing the marketing version.

The second proxy is how you handle trade-offs when the interviewer pushes. The candidate who can name what their design gives up, and defend why the sacrifice is worth it, is the one who can survive a real review. The candidate who treats every push as a question to answer rather than a decision to weigh is the one who will rewrite their design for whoever asked last.

Expected follow-ups: "what is the alternative to this design, and why did you reject it?" and "what is the one thing this design gives up, and why is that acceptable?" Both want the trade-off section made out loud. The candidate who has a concrete reason ready is the one who actually made a decision, and the candidate who has to invent a reason on the spot just discovered they had not.

## Knowledge Check

1. A design doc lists the chosen architecture, then an alternatives section that says "we also considered event-driven and chose synchronous because it was simpler." Why is that alternatives section worthless, and what specific information would make it a real alternative?

2. A doc has no non-goals section. What is the specific review failure you should expect, and what does the absence of non-goals tell you about the author's understanding of scope?

3. You are asked to review a doc written three weeks after the code shipped. What difference in quality should you expect, and what specific section will likely be the weakest? Why?

## Key Takeaways

- A design document carries decisions and their reasons, not a recap of the code.
- Non-goals and alternatives are the sections that tell you the author actually decided; everything else is presentation.
- Write the reasoning next to the decision, not in a separate rationale section nobody reads.
- Reading a doc is reading for commitment: what was chosen, what was rejected, and what cost was accepted.

## What's Next

You can now write a design down in a way that survives contact with a reviewer, which is most of the skill. The remaining gap is the one every LLD interview actually tests: producing the design in the first place, on the spot, under pressure. The final article of this chapter, The Framework, is how to approach LLD problems, and it turns everything in this chapter, the boundary, the requirements, the process, the qualities, into a sequence of steps you can run on any design question.

---

This article explains how to write and read design documents so that decisions, not structure, are what survive the review. Its strongest claim is that the sections teams skip, the non-goals and the alternatives, are precisely the ones that prove a real decision was made.