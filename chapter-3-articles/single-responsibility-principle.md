# Single Responsibility Principle

## Learning Objectives

1. Define SRP as one reason to change, not "does one thing," and explain why the two readings produce opposite results.
2. Find the reason a class changes by asking who asks for the change, not by counting its methods.
3. Recognize when splitting a class by responsibility is really revealing a missing collaborator.

## Introduction

The Single Responsibility Principle is the most cited and most mangled of the SOLID principles. Uncle Bob's phrasing: a class should have one, and only one, reason to change. The mangled version, taught in a hundred blog posts, is that a class should do one thing. Those are not the same sentence, and the difference between them is the difference between a working principle and a license to shred your codebase into a thousand pointless files.

"Does one thing" is unmeasurable. Does `sendEmail` do one thing? It validates, builds, connects, writes, disconnects. Every method does many things if you squint. "Has one reason to change" is answerable. You can look at a class and say what, concretely, would need to be edited in this file, and by whose request.

## Problem Statement

An `OrderService` has grown for two years. It validates the order, applies discounts, calculates tax, writes to the database, sends an email, and generates a PDF. Every requirement that touches any of those flows ends up in this file. A new tax rule. A new email template. A new validation. Each one is a pull request that touches `OrderService`, and each one carries the risk of breaking the other flows, because the class is one big method sequence with fields shared across all of it.

The tests tell the story better than the code. The test file for `OrderService` is 900 lines. Testing the email behavior requires building a full order and running the whole chain, because there is no way to get at the email step alone. Every new test makes the suite slower and every refactor makes it more fragile. The class is not too long, that is a symptom. It has four reasons to change and they are welded together, so changing any one of them means touching all four.

## Core Concept

The precise statement: a class should have one reason to change. The word that does the work is "reason," and the way you find a reason is to ask who asks for the change.

That is the part the "does one thing" reading drops. A reason to change is attached to a person or a role. The accounting department asks for the tax rule to change. The marketing team asks for the email template to change. Those are two actors, two directions of change, and the class that serves both will be pulled in both directions. The moment both ask for changes in the same release, the class is a negotiation between them.

The test for whether two behaviors are one responsibility or two: would these two things change for different reasons, at different times, requested by different people? If yes, they belong in different classes. If no, if they always change together, then splitting them is slicing one responsibility into pieces and paying for the seam for nothing.

Here is where the principle gets counterintuitive, and it is the part most engineers miss. When you split a class by responsibility, you often find you are not making two smaller versions of the original. You are discovering a collaborator that should have existed all along.

Take the `OrderService`. The tax calculation is not a second responsibility of the order service. It is a job that belongs to a `TaxCalculator`, a thing the order service should call. The email is not a second responsibility either. It belongs to an `OrderNotifier`. The order service does not shrink into two order services. It keeps its job, placing orders, and delegates the other jobs to collaborators. That is the "missing collaborator" insight: SRP violations are usually not "this class does two things," they are "this class is doing someone else's job because nobody built the someone else."

The shape of the fix is visible in the constructor, because the collaborators are the class's dependencies.

```
public class OrderService {
    private final TaxCalculator taxCalculator;
    private final OrderRepository repository;
    private final OrderNotifier notifier;

    public OrderService(TaxCalculator taxCalculator,
                        OrderRepository repository,
                        OrderNotifier notifier) {
        this.taxCalculator = taxCalculator;
        this.repository = repository;
        this.notifier = notifier;
    }

    public void place(Order order) {
        order.applyTax(taxCalculator.calculate(order));
        repository.save(order);
        notifier.sendConfirmation(order);
    }
}
```

The `OrderService` still owns the flow, placing the order, and the three jobs that used to be glued into its body are now objects that each own one. A change to the tax rules edits `TaxCalculator` and nothing else. The service's job did not multiply, it stayed one job with three collaborators doing their own.

This is also why SRP is a cohesion principle. Cohesion, the subject of a later article in this chapter, is the measure of whether the things in a class belong together. SRP is the operational form of it: the things that belong together are the things that change together, for one reason, at one time. The class that has high cohesion automatically has one reason to change, because everything in it moves as a unit.

The detection methods are concrete. List the things the class does and, for each, write down who would ask for it to change. If the list has two entries with different authors, you have two responsibilities. Or look at the change history: which files did the last ten pull requests touch? The file that shows up in every change, no matter the feature, is the file with too many reasons to change. Git history is the most honest SRP detector there is.

One clarification that prevents a whole category of mistakes. SRP is not "one public method per class." A `TaxCalculator` can legitimately have `calculate`, `estimate`, `apply`, all public, all sharing one reason to change: the tax rules. Splitting it into `TaxCalculateService`, `TaxEstimateService`, and `TaxApplyService` because it "does three things" is the mangled reading doing damage. The number of methods is irrelevant. The number of reasons to change is everything.

## Real Production Usage

The Spring layering of entity, repository, service, controller is SRP made visible, and you can see the principle in the file names. The `OrderRepository` changes when the persistence contract changes. The `OrderService` changes when the order workflow changes. The `OrderController` changes when the HTTP contract changes. A tax rule change is a change to one service, not to the controller that exposes it or the repository that stores it. When you read a change request that says "tax logic changed" and the diff is one file, that is SRP working.

The counterexample is the real production story that everyone has lived. A service that started with one job and absorbed the notification and the validation and the audit trail over a year. Nobody decided to violate SRP. Each new behavior was a small addition, and the file grew by accretion. That is the actual origin of most violations, and it is why the fix is structural, not disciplinary: the seams should be visible from the start, so each new behavior has a place to land that is not the god class.

## Common Mistakes

The most common mistake is applying the "does one thing" reading and fragmenting. A class with a handful of methods that all change together gets split into a class per method, and the result is three classes that always change together, plus the indirection of wiring them. The test for whether you fragmented wrongly: the new classes change in every pull request, together, forever. If the split did not separate change, it separated nothing.

The second mistake is declaring a class a violation by size. A 700-line class that is one cohesive algorithm with one reason to change is fine. A 100-line class with three reasons to change is a violation. Line count is not the unit. Reasons to change are the unit.

The third mistake is creating the collaborator but stopping halfway. The `TaxCalculator` exists, and the order service still builds the email, still writes the audit log, still validates. One collaborator extracted from four responsibilities is progress, but it leaves the class with three reasons to change instead of four. The principle is not satisfied by partial extraction.

## Interview Perspective

The question "explain the Single Responsibility Principle" filters candidates by which definition they give. The weak answer is "a class should do one thing." The strong answer is "a class should have one reason to change, and you find the reason by asking who asks for the change, because the accounting department and the marketing team are two different reasons even inside one class."

The question that follows is usually "how big should a class be." The strong answer refuses the premise. "Size is a symptom, not the definition. A 700-line class with one reason to change is fine, and a 50-line class with three is not." Candidates who answer size with reasons have the principle; candidates who answer with a line limit have the superstition.

The sharper follow-up: "what does it mean to split a class by responsibility." The strong answer includes the missing-collaborator move. "You do not usually split a class into two versions of itself. You find the job that belongs elsewhere, a tax calculator, a notifier, and you delegate to it. The class keeps its own job." That answer shows the principle as design, not as tidying.

## Knowledge Check

1. A `BankAccount` class has `deposit`, `withdraw`, and `printStatement`. Which of these are one responsibility and which are a second, and what does the answer depend on?

2. An engineer splits a `ReportGenerator` into `ReportCreateService`, `ReportFormatService`, and `ReportPublishService`, and every subsequent change still touches all three files. What did the engineer get wrong, and what test would have caught it?

3. A class has a 600-line method that does validation, pricing, and logging in sequence, and every step changes together for one reason. Is this an SRP violation? Defend your answer with the reason-to-change framing.

## Key Takeaways

- SRP is one reason to change, and the reason is found by asking who asks for the change, not by counting methods.
- "Does one thing" is the mangled reading that fragments cohesive classes into classes that always change together.
- Splitting by responsibility usually reveals a missing collaborator, not a second smaller version of the same class.
- Line count is a symptom; the change history is the honest detector.

## What's Next

SRP told you how many jobs a class should have. The Open-Closed Principle is about what happens when a new kind of job arrives. The next article covers how a module stays untouched while still gaining behavior, the abstraction that makes new behavior a new class instead of an edit, and the honest price of closing a module against change.

---

This article explains the Single Responsibility Principle as one reason to change, not one thing to do, and shows how the reason is found by asking who asks for the change. Its strongest claim is that splitting a class by responsibility usually reveals a missing collaborator, and that line count is a symptom, not the definition.
