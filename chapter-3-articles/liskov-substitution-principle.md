# Liskov Substitution Principle

## Learning Objectives

1. State LSP as a behavioral contract: a subtype must be usable anywhere its base type is used, without the caller knowing the difference.
2. Spot the three contract violations, strengthened preconditions, weakened postconditions, broken invariants, in a subclass.
3. Explain why the compiler cannot check LSP, and why that makes it the real reason to prefer composition.

## Introduction

The Liskov Substitution Principle is the dryest sentence in SOLID and the one with the sharpest teeth. Barbara Liskov's original statement, roughly: if a type S is a subtype of type T, then any property provable about objects of type T must also hold for objects of type S. The everyday version: a subclass must be substitutable for its base class. If code works with the base type, it must keep working when a subclass instance is handed to it, with no surprises.

The catch is in the word "works." The compiler checks types. Liskov is about behavior, and behavior is not a type. Two classes can share a type hierarchy and violate every expectation a caller holds, and the code still compiles. The principle is the contract that fills the gap between what the compiler can prove and what the caller actually relies on.

## Problem Statement

The canonical failure is a geometry model. `Rectangle` has `setWidth`, `setHeight`, and `area`. A `Square` extends it and overrides the setters so that setting the width also sets the height, keeping the square honest. The class hierarchy is perfectly reasonable. Then a caller, written against `Rectangle`, does the natural thing:

```
Rectangle r = new Square();
r.setWidth(5);
r.setHeight(10);
assert r.area() == 50;
```

The assertion fails. The square, built to stay square, set its height to 5 when the width was set to 5, then set both to 10. The caller of `Rectangle` had a perfectly valid expectation, the width and height are independent, and the square broke it. Nothing in the type system noticed, because everything is a `Rectangle`.

The deeper failure is not the assertion. It is that `Square` is not really a behavioral subtype of `Rectangle`. The two have different invariants. A rectangle's width and height vary independently. A square's do not. The subclass inherited a type and violated the type's implicit contract, and every caller that assumed the contract breaks. The is-a test from the inheritance article said "a square is a kind of rectangle," and it lied, because is-a in the type system is not is-a in behavior.

## Core Concept

LSP is about the contract between a type and its callers. Callers of a base type hold expectations, and a subtype is only valid if it does not break them. The contract has three parts, and every violation is one of these three.

Preconditions. What the caller must guarantee before calling. A method on the base takes any non-null string. A subclass override that throws on some strings, or requires the string to be non-empty, has strengthened the precondition. Callers that passed valid input to the base now hit an exception on the subclass. Strengthening preconditions is a violation.

Postconditions. What the method guarantees after returning. The base method returns a non-null result. A subclass returns null in some cases, or returns a value with different meaning, and it has weakened the postcondition. Callers that relied on a non-null result now get null, and the failure shows up far from the override. Weakening postconditions is a violation.

Invariants. Properties that always hold for the type. A rectangle's width and height are independent. A bank account never goes negative. A subclass that breaks the invariant, the square that couples the dimensions, the account subclass that allows the balance to dip, violates the contract even when no single call fails. Invariants are the quietest violations, because they do not throw, they just make the object behave unlike its base.

Beyond the three, there are the behavioral sins that are really special cases. Throwing an exception the base method never throws, for input the base accepted, is a strengthened precondition wearing a different hat. Returning a different subtype than the base's contract implies is fine, that is covariance, but returning null where the base never did is not. Changing the semantics of a method, `add` that now replaces instead of appends, is the most dangerous violation of all, because it passes every mechanical check and fails every caller.

This is why LSP is the reason the earlier chapter's rule exists. The inheritance article said to prefer composition when a child cannot honestly say it is a kind of the parent. LSP is the mechanism that makes that warning concrete. A `Square` cannot be a `Rectangle` behaviorally, so it should not extend it. A stack cannot be an `ArrayList` behaviorally, so it should not extend it. The class hierarchy is only as valid as the behavioral contract, and the contract cannot be compiled, so the only safe move is to refuse the hierarchy when the behavior does not fit.

The fix for the square is composition, and it is the pattern that recurs every time LSP is violated. A `Square` that holds a side length and is its own type, not a subclass of rectangle. The code that needs a rectangle works with rectangles, and the code that needs a square works with squares, and nothing pretends one is the other. The "shape" commonality, if any is genuinely shared, lives in an interface both implement.

The detection technique is brutal and simple: ask what a caller of the base type is allowed to assume, and check each assumption against the subclass. The rectangle caller assumes independent dimensions. The square breaks it. Write the caller's assumptions down and test the subclass against them. The technique works on real hierarchies, and it is the same shape as the is-a test, applied to behavior instead of nouns.

One nuance keeps LSP from becoming a ban on inheritance. A subclass that adds behavior, extra methods, extra state, without changing any inherited contract, is perfectly substitutable. `Dog extends Animal`, adds `bark`, does not break `speak`, valid. LSP forbids the override that breaks the contract, not the inheritance that extends it. The difference is entirely in the overrides.

## Real Production Usage

The JDK ships its own LSP violations, and they are the best teaching material because they are canonical and famous. `Stack extends Vector` and `Properties extends Hashtable`. A `Stack` is a `Vector`, so `add(index, element)` exists on a stack, which is nonsense, and a `Properties` is a `Hashtable`, so `put(null, null)` exists on properties, which corrupts a config file. The JDK's designers violated LSP and the whole industry has paid for it since. When you read "don't extend a concrete class," this is what the warning is made of.

The more common production shape is the one nobody names. A base class with a method that "should not be called," and the subclass overrides it to throw. The `UnsupportedOperationException` inside an override is the tell. `Collections.unmodifiableList` wraps and throws, which is fine because it is composition, the wrapper is not pretending to be a mutable list's subclass. But a domain class whose subclass throws on a method of its own parent is a refused bequest, and the refusal is LSP breaking. The exception hierarchy is the third shape: a method declared to throw `Exception`, and a caller catching `Exception`, catches runtime problems too, which is a contract widened past what the caller can handle.

## Common Mistakes

The most common mistake is strengthening a precondition to "validate" input. A base method accepts a `String`, and the override adds a null check that throws. It feels like defensive coding, and it is a contract violation, because the base promised to handle null and the subclass broke the promise. Callers stop passing null after a while, and the bug lives quietly until one legitimate caller relies on the base contract.

The second mistake is weakening a postcondition to "be efficient." The base returns a full list, the override returns an unmodifiable copy, and callers that expected to mutate the result now throw. The optimization saved a defensive copy and broke every mutating caller. Postconditions are promises, and breaking a promise for speed is LSP failing.

The third mistake is the is-a test by noun. `Square` is a rectangle, a `Pizza` is a circle, a `Stack` is a list, so extend. The noun test passes and the behavioral test fails. The rule to replace it: the subclass must be a genuine kind of the base in behavior, and the override that fights the parent is the confession.

## Interview Perspective

The question "is a Square a Rectangle" is the classic LSP trap, and the weak answer is yes, because geometry. The strong answer separates the two readings. "In the type system yes, and that is exactly the problem. A square and a rectangle have different invariants, the dimensions are independent in one and coupled in the other, so a caller of `Rectangle` that sets width and height breaks on a `Square`. The answer is no, not behaviorally, and the design fix is composition, not inheritance."

The follow-up "how do you check LSP" wants the detection method, not the definition. "I list what a caller of the base type is allowed to assume, and I test the subclass against each assumption. Strengthened preconditions, weakened postconditions, broken invariants, any of the three means the hierarchy is wrong."

The sharper interviewer asks "is a subclass that throws on a parent method always a violation." The strong answer qualifies. "It depends. An `UnsupportedOperationException` in a wrapper that is not pretending to be the base is fine, that is composition. The same throw in a genuine subclass that inherited the method is a refused bequest, and it means the class should not have been a subclass."

## Knowledge Check

1. A base `Account` has `withdraw(int)` that returns true when the withdrawal succeeds. A subclass `OverdraftAccount` overrides `withdraw` to allow a negative balance and returns true. Which contract part did it change, and which callers break?

2. A `Logger` base has `log(String)` that accepts any string, including null. A subclass `NullSafeLogger` throws on null. Classify the violation, and describe the caller that would hit it.

3. A subclass `ReadOnlyList` extends `ArrayList` and overrides every mutating method to throw. Is this a valid extension or a violation, and what does the composition alternative look like?

## Key Takeaways

- LSP is a behavioral contract: a subclass must work wherever its base works, with no caller surprises.
- The three violations are strengthened preconditions, weakened postconditions, and broken invariants.
- The compiler cannot check behavior, which is why a hierarchy is only as valid as its behavioral contract.
- A subclass that has to fight its parent is a refused bequest, and the fix is composition, not a better override.

## What's Next

LSP governs how subclasses behave toward their base. The Interface Segregation Principle moves the same idea one level out, to the interfaces a client depends on: a client should not be forced to depend on methods it does not use. The next article covers the fat interface, the split by consumer, and why the JDK keeps its reading and writing contracts separate.

---

This article explains the Liskov Substitution Principle as a behavioral contract the compiler cannot check, and the three ways a subclass breaks it. Its strongest claim is that a subclass that has to fight its parent is a refused bequest, and that the fix is composition, not a cleverer override.
