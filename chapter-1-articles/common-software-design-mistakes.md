# Common Software Design Mistakes

## Learning Objectives

1. Recognize the five design mistakes that show up in real codebases and spot them in your own design.
2. Tell over-engineering from careful planning, and premature abstraction from a justified interface.
3. Fix a mistake at the coupling that created it, instead of layering more code on top.

## Introduction

A short list of design mistakes accounts for most of the pain in production software. The same five keep reappearing, not because engineers are careless, but because each one is a mildly reasonable choice at the moment it is made. The problem is the slope, not the first step. A class gains one extra field, then another, and before anyone notices it has become a god object.

Name these mistakes and you can catch them at the review, when they are a small decision, instead of at the rewrite, when they are a project. This article catalogs the five, and it ends with a claim: four of them are the same disease at four different sites. They are all a coupling you chose to create.

## Core Concept

Mistake | What it is | What it looks like
--- | --- | ---
God object | One class absorbs every responsibility | Every change touches the whole class
Premature abstraction | An interface for a consumer that does not exist | Readers pay for indirection no one uses
Overengineering | A solution for a scale you cannot confirm | A cache or a broker with no load behind it
Ignoring non-functional | Shipping on "it works" | The system dies under the first real load
Copy-paste | Duplicating behavior that should be shared | The bug is fixed in two places, live in the third

### The god object

The god object is the class that has absorbed too many jobs. It holds data, validates input, talks to the database, decides policy, and formats output, all in one structure. It did not start that way. It started reasonable, and one responsibility at a time was added to it because adding a method to an existing class is the lowest-friction change a developer can make. No new file, no wiring, no new reviewer. Just one more method.

That cheapness is why the god object is the most common mistake. Every feature is a method on the same structure, and the class accumulates the cost of the whole feature in one place.

The god object is a coupling problem, not an aesthetic one. Everything shares the same fields, so every behavior depends on every other. Change the storage and you must check the whole class, because the class is a miniature of the whole system. High cohesion is gone, and it only gets worse.

The fix is not to break it apart right now. The fix is to stop feeding it. Each new responsibility is a point where you decide: does this belong in the class because it is the same job, or in a new class because it is not? The god object was built by skipping that decision. The way to kill it is to start taking that decision.

### Premature abstraction

The subtlest habit is premature abstraction. You define an interface and a handful of variants for a behavior that has exactly one implementation today. The motive is flexibility. The result is that every reader has to follow the extra indirection to reach the one real class, and the reader pays for a flexibility nobody has asked for yet.

The line between a useful interface and premature abstraction is whether the second use is real or imagined. A second consumer that you have already committed to is real. An abstraction you write "just in case it might change" is imagined. Flexibility for a problem you know you are about to solve is planning. Flexibility for a problem you hope someday exists is speculation with someone else's time.

Test your abstraction with one question: can you name the second consumer? If you cannot name the consumer who will exercise the second behavior, do not write the interface yet. An interface is cheap to add, so you can add it when the real second consumer arrives. That is the rule: a named future justifies the interface, an unnamed one kills it.

### Overengineering

Overengineering is solving a scale problem you do not have. A distributed cache in front of a database that handles the traffic with the database alone. A message broker in front of a feature that produces two events a day. A service split when the whole app fits in one engineer's head. It looks like careful work and it is the opposite. It is paying full price ahead of a problem you have not confirmed.

The cost is not just the cache investment math. It is the tax every next engineer pays. A cache today is an invalidation bug next month and a consistency surprise the quarter after. A broker today means debugging message order forever for a handful of events. You debug the infrastructure instead of the feature.

Scale is a requirement you earn, and you should design it as one. If the load is not real, and a simpler structure can handle it and grow when the real load arrives, pick the simpler one, and arrange the seams so the heavier part can be added later. That is how you prepare for growth without paying for it. The design that uses the load you have, and is arranged to grow into the load you find, is the one that lasts.

### Ignored non-functional requirements

This mistake belongs on the list because a requirements failure and a design failure look identical from the outside. A feature works in tests and dies under the first real load. Was the spec wrong or the design wrong? Both, honestly, but the design is the half you control, so treat it as yours. A design that never asked how many events per second it handles, and never put that number anywhere the code honors, has a hole.

The design is where the volume becomes structure. Asking how much traffic and how fast, then choosing the cache, concurrency, and state ownership to meet it, is the definition of LLD. Skipping that and building toward "it works" is not an accident, it is a decision to defer the question until the day the traffic forces it.

### Copy-paste that looks like DRY

A separate species of bug. Someone needs the same behavior in a second place and copies the code instead of extracting it. Now there are two real copies of the same behavior. When the fix arrives, it is applied to one copy, and the other becomes the known bug waiting to explode.

DRY exists to prevent this, but DRY is a judgment and people mistake it for a rule. Two copies are often a signal that the difference between the two versions is about to arrive, the moment the two copies will start to disagree. That is the moment to extract: when the two versions differ. So do not unify at copy two. Watch for the first real difference between the copies, and extract at the moment it appears. The signal is the distinction, not the repetition.

### The review angle

None of the five shows up as a failing test. Each shows up in a reading. That is why reviews catch them and suites do not. When you read a design, the flags are specific: does the class name still match what it does? Can you name the second consumer of that interface? Is there a cache for a load that has not arrived? Is there a number anywhere for the volume? Is the same behavior written down in three spots? Once you can name the five, you do not need to say "this feels off"; you can say which one it is.

There is a common reaction to naming a mistake that is itself a sixth mistake, worth a warning. When a review says "this is a god object," the fix is not to wrap the class in a facade or to bolt on an interface that leaves all the responsibilities where they were. That is treating the label as the problem. The fix is to move a responsibility out, actually relocate the code, and if the reviewer is not willing to do that, the comment was decoration. Fixing a design mistake by adding a layer is how you get seven layers of architecture over the same god object. The only honest fix is the one that moves or removes the coupling, and every fix that leaves the coupling in place is a postponement.

## Interview Perspective

You will not be asked to list these five, but you will be watched for the chances you take. Design problems are a live sample of your review judgment. The candidate who says "I almost added an interface here, but I cannot name the second consumer, so I will not" tells the whole story in one sentence: that you can see the line and stop before it.

A weak answer describes a design by what it wants. A strong answer describes it by what it chooses not to do. "I am not caching this right now, the reads are fine and the invalidation was not the risk" is a real design statement. "I will add a cache for speed" is a guess. The design conversation is often about the abstractions you chose not to build, and the candidate who can defend the things they did not add is the one with the honesty to cut.

## Knowledge Check

1. A repository behind an interface has one implementation. Give two conditions: one that makes it premature abstraction and one that makes it justified. Name the decisive test, the question that separates them.

2. The same formula appears in four places, and two of them hide a subtle difference. What is the right short term move, and what is the right long-term move? Why is the seductive move, unifying all four, the wrong one?

3. A design feels over-scoped for the load it faces. What concrete evidence would turn it from over-scale into exactly the right scale, and why does the evidence have to arrive before the part is acceptable?

## Key Takeaways

- Four of the five mistakes are the same disease: a coupling you chose to introduce.

- The god object dies by refusing the next responsibility, not by the big break.

- An interface needs a named second consumer or it is decoration.

- The review is where the mistakes are caught, and a review that names one of the five is a finding, not a vibe.

## What's Next

You can now recognize and name a design mistake, but nothing in a meeting gets started until it is written down. The next article, How to Read and Write Design Documents, covers how to record a design so the reader can see the god object and the second consumer, and how to read someone else's document without guessing what they actually committed. A design document is where these mistakes are caught or enshrined, and you will see what makes the difference.

---

This article lists the five design mistakes that keep appearing in production code, from god objects to premature abstraction, and argues they are mostly one deep problem at different sites. Its strongest claim is that the fix that lasts is made at the review, when a ten-minute decision is cheaper than the refactor it prevents.