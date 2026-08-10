# The Golden Rule: Composition Over Inheritance

## Learning Objectives

1. State the rule precisely, and defend it without turning it into "never use inheritance."
2. Recognize the inheritance failure modes, the deep hierarchy, the brittle base, and the subclass that exists to disable.
3. Build with composition and delegation, and name the two or three cases where `extends` is still right.

## Introduction

The most repeated sentence in object-oriented design is "favor composition over inheritance." It has outlived every pattern book and every framework because it survives contact with real code. The advice is not a ban. It is a default. When a new class needs behavior, reach for a collaborator before you reach for a parent, and let the inheritance article's is-a test earn the `extends`.

The reason the rule survives is that inheritance is the strongest coupling Java has, stronger than any field. A subclass is welded to its parent's implementation, its parent's parent, and every sibling that shares the parent. Change the parent and every subclass changes with it. Composition is the opposite, a field and a method call, a relationship you can swap at runtime. The rule is a bet that the weaker coupling wins, and the bet has held for thirty years.

## Problem Statement

Model a zoo with inheritance. `Animal` has `eat()`. `Dog extends Animal` adds `bark()`. `Bird extends Animal` adds `fly()`. Then the requirements grow, and the hierarchy starts to fight back.

A `Penguin` is a bird that cannot fly. The clean answer, no `fly()` for penguins, is blocked, because `Bird.fly()` is inherited and `Penguin` must either throw or no-op. A `Dog` needs to fly for a scene, and now there is `FlyingDog extends Dog`, and then a dog that swims, `SwimmingDog extends Dog`, and then a dog that does both, which cannot extend both. The designer faces the two classic walls of inheritance: behavior that does not fit a child, and behavior that a child needs twice.

Then the base class changes. Someone adds `walk()` to `Animal`, and suddenly every subclass has a `walk()`, including the `Fish` that should not, and the `Bat` that should. The change was made in one place, and it landed everywhere, because inheritance broadcasts every parent change to the entire subtree. The zoo model has become a liability, and the liabilities all trace to one decision: classes were shaped by what they extend instead of what they contain.

## Core Concept

The rule in one line: build an object from the parts it has, not from the parents it is. Where inheritance says "I am a kind of X," composition says "I have an X." Where inheritance takes the parent's code wholesale, composition asks for a collaborator and delegates.

The delegation move is the mechanical heart of the rule. A class holds a collaborator and forwards calls to it:

```
public class Duck {
    private final FlyBehavior flyBehavior;

    public Duck(FlyBehavior flyBehavior) {
        this.flyBehavior = flyBehavior;
    }

    public void performFly() {
        flyBehavior.fly();
    }
}
```

`Duck` does not extend a flying class. It holds one, an interface, and hands the call over. The penguin problem disappears, a penguin holds a `NoFly` behavior. The flying swimming dog problem disappears, a dog holds whatever behaviors it needs. Behavior becomes a field you can swap at runtime, and that is the difference: inheritance bakes the behavior in at compile time, composition installs it at construction.

Why inheritance fails as a reuse tool deserves precision, because the failure is not cosmetic. First, inheritance couples a class to its parent's implementation, so the child cannot change behavior the parent hard-codes, and the parent cannot change behavior without auditing every child. Second, inheritance forces a class to take the parent's whole contract, which is how `Penguin` inherits `fly()` and `Stack` inherits `add(index, element)` it should not have. Third, inheritance creates the depth problem, a change at level three of a five-level chain ripples through everything below, and nobody can say what `Base` actually does without reading five files.

Composition does not have those problems because it has no contract to inherit and no depth to ripple through. A composed class takes exactly the collaborators it needs, and a change to one collaborator touches only the classes that use it.

When is `extends` still right? The exceptions matter, because a rule you cannot violate is a rule nobody trusts. First, genuine is-a with behavior that fits, a `PaymentProcessor` implementation hierarchy in your own domain where every child is a real kind and overrides all the abstract methods. Second, when the parent exists to define a contract, abstract classes and interfaces, and the child only implements. Third, when a framework requires it, rare in modern Java, but `Thread` subclasses predate the `Runnable` composition pattern that replaced them. The test is always the is-a test from the inheritance article: say the sentence out loud, "this child is a kind of parent," and if the sentence is false, or the child must disable a parent method to survive, the child wants composition.

The final piece is how the two combine. Composition does not forbid interfaces, and the composed class usually implements one. A `Duck` implements `Bird`, which is the polymorphism the design needs, and holds a `FlyBehavior`, which is the reuse the design needs. You get the contract from the type system and the behavior from the fields, and nothing is inherited that should not be. That is the full move: is-a through interfaces, has-a through fields.

## Real Production Usage

The JDK is the best museum of the rule, and its unmodifiable collections are the cleanest exhibit. `Collections.unmodifiableList` does not subclass `ArrayList`. It wraps a list in a `UnmodifiableList` that delegates every read and throws on every write. Same contract, no inheritance, because subclassing `ArrayList` to block writes would produce a class that violates its parent's contract, and the JDK declined to build that. The wrapper is composition with delegation, and it is in the standard library.

`BufferedReader` is the same shape. It wraps a `Reader` and delegates, layering buffering over an existing reader without being a kind of reader's implementation. The stream and reader libraries are a chain of wrappers, decorators, and the wrapper is always composition. When you see a class whose constructor takes an object of the type it wraps, you are watching the rule in production.

Inheritance appears where it is genuinely is-a. `IllegalArgumentException extends RuntimeException` is a subtype relationship that adds no behavior and fights no parent. `Dog extends Animal` in a well-designed domain hierarchy, each subclass overriding the abstract methods, is the other legitimate case. The JDK has both, wrapper composition for reuse and subtype inheritance for contract, and the distinction is visible in the code.

## Common Mistakes

The first mistake is "I need the method, so I extend." The class needs one behavior, and instead of holding a collaborator, it becomes a subclass and inherits the whole parent, including the methods it does not want. The fix is the delegation move, one field and one forwarded call, and the coupling drops from the whole class to one method.

The second mistake is the deep hierarchy. Four and five levels of `extends`, each level adding a method, until the base class is a grab bag and every child inherits everything. The depth is the problem, not the individual relationships, and the fix is to flatten: push shared behavior into collaborators and keep the inheritance two levels where it exists at all.

The third mistake is the subclass that exists to disable. `Stack extends Vector`, and then the code overrides or throws on the methods a stack should not have, `add(index, element)` and `set`. A class that fights its own parent is a confession that it should not be a subclass, and the classic `Stack` is the case study. The fix is composition, `Stack` holding a `Deque`.

The fourth mistake is the brittle base class. A parent is changed for one child, and every sibling inherits the change, and the whole subtree pays for one subclass's feature. The parent should not be edited for a child's sake; the child should hold a collaborator. Every time you find yourself adding to a base class for one child, you are repeating the problem statement's `walk()`.

## Interview Perspective

The question "what is your favorite design principle" is often this rule, and the answer must show both sides. "Favor composition over inheritance. Inheritance couples a class to its parent and its siblings, and forces it to accept a whole contract, so a Penguin inherits fly() and a Stack inherits add(index). Composition holds a collaborator and delegates, which is why the JDK wraps lists instead of subclassing them. Inheritance is still right for genuine is-a, an abstract contract with real subclasses."

The follow-up "when do you decide between them" wants the decision procedure, not a platitude. "I ask the is-a sentence. If the child cannot honestly say it is a kind of the parent, or has to disable a parent method, I compose. If it is a real subtype and the hierarchy is shallow and I own it, I extend." The candidate who can run the procedure on a concrete class is done.

The sharper probe: "favor composition, but you used inheritance in that answer." The candidate who can name the contradiction and hold it is showing the actual understanding. The rule is a default, not a law, and the answer is the exceptions paragraph: contract inheritance for is-a, composition for reuse.

## Knowledge Check

1. A `Vehicle` needs `startEngine()` and `honk()`. Model it with composition and name the collaborators, then explain what changes if a `Bicycle` appears.

2. The zoo model has a `FlyingSwimmingDog` requirement. Explain why `extends` cannot express it, and write the composed version.

3. State the exact condition under which `extends` is the right choice, and give one real class pair from the JDK where the condition holds.

## Key Takeaways

- Favor composition over inheritance: build from collaborators, not parents, and delegate.
- Inheritance couples a class to its parent and siblings and forces a whole contract; composition couples nothing beyond the field.
- Is-a through interfaces, has-a through fields, and `extends` only for genuine subtype contracts.
- A subclass that exists to disable a parent method is the clearest signal the design wants composition.

## What's Next

This is the last article on the mechanics of the language. The relationships, the memory model, the dispatch, the generics, they are now vocabulary. The next chapter, Core Design Principles, changes the subject from how Java expresses a design to how a design should be judged, and its first article, the Single Responsibility Principle, will make you look at a class and ask one question this whole chapter has been circling: what, exactly, is this class for?

---

This article explains the golden rule, favor composition over inheritance, and grounds it in the failure of deep hierarchies and subclasses that disable their parents. Its strongest claim is that the is-a test earns every extends, that delegation replaces reuse-by-inheritance, and that is-a belongs to interfaces while has-a belongs to fields.
