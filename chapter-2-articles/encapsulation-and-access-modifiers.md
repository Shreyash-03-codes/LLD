# Encapsulation and Access Modifiers

## Learning Objectives

1. Explain encapsulation as the bundling of state and behavior, plus the hiding of state.
2. Know exactly who sees each of the four access levels, and when to use each one.
3. Defend why exposing a field through a getter is not the same as encapsulating it.

## Introduction

Encapsulation is the idea that an object owns its state and is the only thing allowed to touch it directly. The outside world talks to the object through methods, not through its fields. Access modifiers are the mechanism that makes that wall real in Java: `private`, package-private, `protected`, and `public` decide who is allowed to see what.

The two halves matter equally. Bundling state and behavior together is what makes an object a unit, and hiding the state is what makes the unit safe to trust. Without the hiding half, the object is just a struct with methods attached, and every caller can reach in and violate whatever rules the object is trying to enforce.

## Problem Statement

A `BankAccount` class keeps its balance in a public field. The withdrawal method checks that the balance cannot go negative. It works, until a caller in a distant module writes `account.balance = -500;` directly. The method's careful check is bypassed, because the field was public and the wall did not exist. The invariant, balance never negative, is now a suggestion.

This is the concrete cost of no encapsulation: the object's rules are only enforced for callers who use the object politely. Every invariant, the balance, the state machine, the one-way transition, is enforceable only if the state cannot be reached around the methods. The moment a field is public, the object's contract has a door in the back, and it is only a matter of time before someone walks through it.

## Core Concept

Encapsulation is two claims about a class. The first: the fields and the methods that operate on them live together, one unit. The second: the fields are not directly reachable from outside the unit. Code outside the class sees the methods, the public API, and nothing of the internals that make the methods work.

The access modifiers are the tool that enforces the second claim. Java has four levels of visibility, and each one is a specific answer to the question "who may touch this member?"

Modifier | Same class | Same package | Subclass | Anywhere
--- | --- | --- | --- | ---
`private` | Yes | No | No | No
package-private (no modifier) | Yes | Yes | No | No
`protected` | Yes | Yes | Yes | No
`public` | Yes | Yes | Yes | Yes

A field or method declared `private` is visible only inside its own class. That is the workhorse of encapsulation: the class's internals are locked away, and the class decides what to expose. Package-private is the default when you write no modifier, and it is underrated. It means "visible to the rest of this package," which is often exactly the right scope for helpers that a class's collaborators need but that no external caller should touch. `protected` adds subclass visibility on top of package visibility, useful when a base class wants to hand hooks to its children without exposing them to the world. `public` is the contract boundary, the API the class offers to everyone.

The rule of thumb worth holding: default to `private`, widen deliberately, and be suspicious of `public`. Every member you make public is a commitment you are stuck with, because callers will depend on it, and removing it later breaks them. The private members are free to change, because nobody outside the class can see them. Encapsulation is, among other things, a way to keep your freedom to change the inside of a class without asking anyone's permission.

The hiding does more than enforce invariants. It makes internals replaceable. A class can change its internal representation, an `ArrayList` to a `HashSet`, a computed value to a stored one, and as long as the public methods keep their behavior, no caller notices. That is the practical payoff: a change inside the wall is a change to one class, not to every class that touches it. Encapsulation is the memory-model article's coupling lesson made concrete at the class level.

Now the uncomfortable part. A getter and a setter on every field is not encapsulation, it is a public field with better grammar. If `getName()` and `setName(String)` are the only members and they do nothing but read and write the field, the wall is decorative. The caller can still read and write everything, just through method calls. Real encapsulation exposes behavior, not data: the methods are things the object can do, and the fields are the reasons the behavior works, kept out of sight.

```java
public class Temperature {
    private double celsius;

    public double inCelsius() { return celsius; }
    public double inFahrenheit() { return celsius * 9.0 / 5.0 + 32; }
    public void setInFahrenheit(double f) { celsius = (f - 32) * 5.0 / 9.0; }
}
```

That class is encapsulated. The unit of state is the celsius field, which nobody outside sees. The behavior is expressed in the public methods, and the internal choice of which unit to store is private and changeable. Compare a class with `getCelsius` and `setCelsius`: the field is still exposed, just through a door that looks like a method.

There is one more subtlety that trips people up, the mutable internal. A class stores a `List` and exposes `getItems()` that returns the internal list reference. Every caller now holds a reference to the object's own list, and anyone can mutate it, which silently breaks the encapsulation that the private field appeared to provide. The fix is either to return a copy, or to return an unmodifiable view, so that reading the state does not hand out the key to changing it.

## Real Production Usage

The standard library is the best running argument for encapsulation. `java.util.Collections.unmodifiableList` returns a view that rejects mutation, so a class can expose its internal list without surrendering it. An `UnsupportedOperationException` on `add` is the framework enforcing the wall the class declared. And the classes in `java.util.concurrent`, `AtomicInteger`, `ConcurrentHashMap`, hide their internals completely and expose only thread-safe behavior, which is encapsulation taken to the point where the hidden internals are what guarantee safety.

Reflection is the honest counterpoint. `setAccessible(true)` lets any framework reach into private fields, which is how Spring wires dependencies and how Hibernate sets fields on entities. That does not make encapsulation a lie. It makes it a compile-time contract rather than a runtime fortress. The frameworks are trusted parties that opt out deliberately. Your code, and everyone else's code, is expected to respect the wall, and the compiler enforces that respect.

The frameworks also show the price of the wall. When Hibernate cannot reach into a class, or when Spring cannot find a setter, they refuse to do their job. That friction is the framework honoring the contract the class declared. A class that hides everything from everyone also hides it from the frameworks that need to see it, which is why JPA entities expose getters and setters even though pure encapsulation would prefer not to.

## Common Mistakes

The most common mistake is declaring fields `public` "for convenience" or to shorten the code. The convenience lasts one afternoon, and the invariant damage lasts forever. Every public field is a permanent commitment and a permanent hole. Default to `private`.

The second mistake is the getter/setter reflex that mistakes ceremony for encapsulation. A class of `private` fields with trivial getters and setters for every one of them has all the code of encapsulation and none of the protection. The tell is behavioral: if you cannot name a rule the object enforces, the methods are just indirection. Encapsulate behavior, not fields.

The third mistake is leaking the mutable internals. Returning the internal `List` or the internal map from a getter hands the caller the keys. The class looks encapsulated, the field is `private`, and the list still gets mutated from outside. Return a copy or an unmodifiable view, and only when the caller genuinely needs it.

## Interview Perspective

Interviewers probe encapsulation because it is the fastest way to separate the candidate who memorized the four modifiers from the candidate who knows why the wall exists. The weak answer recites the access levels. The strong answer explains that `private` state plus public behavior is what lets a class enforce its own invariants and change its internals without breaking callers.

The deeper probe is usually immutability. "How do you make a class immutable" wants: final fields, no setters, don't leak mutable references, and make the class itself final or the fields final and deep. The candidate who answers "final fields and no setters" and stops has missed the part where the class hands out its internal list, the mutable-reference leak that breaks immutability through the getter.

Expected follow-ups: what is the difference between `protected` and package-private, and why does exposing a getter that returns the internal list break encapsulation? The first tests modifier mechanics, the second tests whether the candidate sees the wall as a real boundary or as a label.

## Knowledge Check

1. A class has a private field `balance` and a method `withdraw(int)`. A caller in another package tries `account.balance = 0;`. What happens at compile time, and what would the caller have to do to make that line legal?

2. A class exposes `getItems()` that returns its internal `List`. Name two ways to keep the field private while still letting callers read the items, and explain what each one gives up.

3. Two classes both have private fields and public getters and setters for everything. One also enforces an invariant in its setter, the other just assigns. Which one is encapsulated, and how would you tell from the behavior alone?

## Key Takeaways

- Encapsulation is bundling state with behavior and then hiding the state; access modifiers are the enforcement.
- Default to `private`; `public` is a commitment you will live with, so make it deliberately.
- Getters and setters on every field are not encapsulation, they are a public field with better grammar.
- Do not leak mutable internals; if you must expose a collection, expose a copy or an unmodifiable view.

## What's Next

Encapsulation decides who can see the internals of a class. The next question is what the outside world is even allowed to depend on, which is the subject of the next article. Abstraction: Interfaces vs Abstract Classes covers the two ways Java lets you define a contract without tying callers to a concrete class, and where each one is the right tool.

---

This article explains encapsulation as the bundling of state and behavior with the state hidden behind access modifiers, and walks exactly who can see each of the four levels of visibility. Its strongest claim is that a getter and setter on every field is a public field with better grammar, and that real encapsulation exposes behavior, not data.
