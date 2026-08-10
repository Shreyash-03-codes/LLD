# Identifying Entities and Relationships Under Pressure

## Learning Objectives

- Learn a repeatable technique for pulling the noun list out of a problem in the first two minutes, before the pressure can turn it into guessing.
- Distinguish the entities that carry a responsibility from the nouns that are just words, and get comfortable deleting.
- Practice naming relationships precisely, because the relationship is where the behavior lives, not in the classes themselves.

## Introduction

The requirements phase is a conversation. The entities phase is a test of nerve. You have five minutes to look at a problem you may be seeing for the first time and name the objects the whole design is made of, while the interviewer watches. The pressure does not come from the difficulty of the nouns, it comes from the fact that there is no formula that guarantees the right list. There is a technique, though, and it is the difference between the candidate who stares at the whiteboard and the candidate who starts writing. That technique is not cleverness, it is extraction discipline: gather every noun, test each one for whether it carries a responsibility, and keep only the ones that do. Most of the interview's structure is decided in these five minutes, which is why the candidate who does this phase well is nearly impossible to beat in the ones that follow.

## Problem Statement

Watch a candidate who skips the entity technique. They hear "design a library management system" and start writing classes by feel. `Book`, obviously. `Member`, of course. `Librarian`, maybe. `Library`, sure. `Bookshelf`, why not. `ISBN`, no wait, that is a field. Ten minutes in, the list has thirteen entries and no reasoning, and when the interviewer asks "what is the difference between a book and a copy of a book?" the candidate realizes there is no `BookCopy` anywhere and the design has to be redrawn. The missed entity was not missed because the candidate lacked the vocabulary. It was missed because there was no process that forced the question "what nouns exist that I have not named?"

The failure is systematic. Without a technique, entity identification becomes a feel-based draw that stops at the obvious nouns, and the interesting entities, the ticket in the parking lot, the loan in the library, the calendar in the car rental, the offset in the broker, are exactly the ones the feel-based draw misses, because they are not nouns you can see, they are nouns you have to think about. The technique exists to make sure you think about them.

## Core Concept

The technique has four moves, and they should take about four minutes.

**Move one: dump every noun onto the board.** Do not filter, just write. The problem statement, your knowledge of the domain, everything. For a parking lot: car, spot, floor, ticket, gate, attendant, payment, receipt, garage, level, row, sign, vehicle, time. For an ATM: card, pin, account, balance, withdrawal, deposit, transfer, receipt, cash, cassette, bank, session. The dump is noisy on purpose. Its job is to get every candidate noun out of your head and into the open where it can be tested, and to guarantee you have not skipped a category. Candidates who skip the dump and go straight to the final list are gambling that their first draft is their best draft, which it is not.

**Move two: test each noun for a single responsibility.** This is the filter that separates the entities from the noise, and it is the whole trick. An entity earns its place if you can write one sentence about what it does that no other noun does, a responsibility that starts with a verb. "The ticket records the entry time and the spot." Keep it. "The receipt confirms payment." Delete it, the ticket can do that. "The garage houses the cars." Delete it, that is the lot. The test is brutal by design: it is easier to add a class back than it is to defend one you drew on a hunch, and the interviewer will ask you to defend every class you keep. Nouns that fail the test become fields or methods of nouns that pass.

**Move three: name the relationships, with the multiplicity.** This is the move candidates skip and the one interviewers probe. Every kept entity needs its connections named: a lot has many floors, a floor has many spots, a spot holds one vehicle at a time, a vehicle has one ticket while parked. The relationship sentence is what the class design will implement, and getting the multiplicity right, one-to-many, many-to-one, one-to-one, is what stops the design from building the wrong collections. "A member has many loans, a loan has one copy" tells you `Member` holds a list and `Loan` holds a single reference, and that is the class design before you have written a line of code.

**Move four: delete once, out loud.** Now that the list is filtered and the relationships are named, do a deliberate pass where you try to delete each remaining entity. "Can I remove `ParkingFloor` and store spots directly on the lot?" "Not without losing the per-floor free-spot query." "Can I remove `BookCopy` and let `Book` hold availability?" "Not if the library owns three copies of one title." That out-loud deletion pass is what makes the final list defensible. It is also the exact exercise the interviewer runs when they ask "do you really need that class," and if you have already run it, the answer is ready.

## Real Interview Context

The interviewers on the other side of this phase are not checking that you named the same entities they would have named. They are checking two things. The first is whether your entities have responsibilities, because a list of nouns without verbs is a list that cannot be walked through. The second is whether you can justify the cuts, because the candidate who says "I have eight entities" and the candidate who says "I have eight entities, and here is why not nine" are displaying different levels of the same skill. The relationship answers are what the walkthrough runs on, and the deletion reasoning is what survives the "why" follow-ups. An interviewer who sees a candidate name `Loan`, justify why it exists as a record, and state "member has many loans, copy has one active loan" is already forming the hire signal, because that candidate has demonstrated the entire technique in three sentences.

There is a calibration worth stating plainly: the length of the final list should worry you in both directions. A parking lot designed with twelve entities is almost certainly a parking lot with decoration. A chat system designed with four is almost certainly missing the broker or the registry. The right list for a classic LLD problem is usually between five and eight entities, and a candidate whose list lands in that range, whatever the specific nouns, is in the territory where the interviewer can have a productive conversation instead of a repair session.

## Common Mistakes

The most common mistake is the noun dump presented as the final design. The candidate writes thirteen nouns and starts drawing arrows, and the interviewer's "what does `Garage` do that `ParkingLot` does not" has no answer. The dump is a starting inventory, never the deliverable, and presenting it as the deliverable is how the entity phase becomes the whole interview in miniature.

The second mistake is treating fields as entities. `ISBN`, `price`, `plateNumber` promoted to classes. Every one of them is a field of a real entity, and every one of them that stays a class is a class without a responsibility that the interviewer will delete with one question. The single-responsibility test exists precisely to catch these, and the candidate who skips the test is the candidate who draws a `Money` class for a system with one currency.

The third mistake is stopping at the visible nouns. The candidate names the things they can see, the car, the floor, the spot, and never names the things they have to think about, the ticket, the loan, the calendar, the ledger. The invisible nouns are where the interesting design lives, and the trick to finding them is the responsibility test applied in the opposite direction: start from the verbs in the requirements, and ask "which object carries out this action?" The verbs will name the invisible nouns for you.

## Interview Perspective

A weak entity phase is a stream-of-consciousness list that grows while the interview runs, with entities added whenever a follow-up exposes a hole. The interviewer says "how do I know a spot is free?" and a `ParkingSpot.isAvailable` field appears, then "who computes the fee?" and a `FeeCalculator` appears. The design is being patched, not built, and the patches have no relationships to the classes around them.

A strong entity phase is a filtered list, a set of named relationships, and a deletion defense, delivered in four minutes. When the interviewer pushes, the candidate answers with the responsibility test, "I considered a `Reservation` class and cut it, because reservations are out of scope and it would have no responsibility this system needs." The follow-ups in a strong phase are all variations on "what about X?", and every one is answerable with the same two moves: run the responsibility test, name the relationship. The strongest candidates run the deletion pass unprompted and volunteer why they cut a noun, because they know the interviewer is about to ask.

## Knowledge Check

1. Run the noun dump and the responsibility test on a vending machine. Name the noun that looks like an entity but is really a field, and the noun that is invisible but carries the whole design.
2. "A member has many loans, and a loan has exactly one copy" is a relationship statement. Explain what this sentence tells you about the fields of `Member` and `Loan`, and why getting the multiplicity wrong breaks the class design.
3. The interviewer asks "why is `ParkingFloor` a class and not a list field on `ParkingLot`?" Give the responsibility-based answer, and the counter-scenario in which the floor genuinely would become a field.

## Key Takeaways

- Dump every noun first, filter second. The noisy inventory beats the confident guess.
- The single-responsibility test is the filter: one sentence, starting with a verb, or the noun goes.
- Name relationships with multiplicity. The relationship sentence is the class design in prose.
- Run the deletion pass out loud, and you have pre-answered half the interviewer's follow-ups.
- Invisible nouns, the ticket, the loan, the ledger, are found by asking which object carries out each verb in the requirements.

## What's Next

Entities are the nouns and the relationships. Patterns are the grammar that connects them, and the next article is where the mistake most candidates make is not knowing too few patterns, it is reaching for the wrong one because they learned them as a menu.

---

This article explains how to identify entities under pressure with four moves: dump every noun, filter by responsibility, name the relationships, and delete out loud. Its point of view is that the invisible nouns, the ticket, the loan, the ledger, carry the design.
