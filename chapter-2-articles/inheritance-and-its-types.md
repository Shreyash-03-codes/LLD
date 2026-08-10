# Inheritance and Its Types

## Learning Objectives

1. State what inheritance provides, and what a subclass actually receives from a superclass.
2. Draw the four shapes inheritance can take and know which ones Java allows.
3. Apply the is-a test before ever using `extends`.

## Introduction

Inheritance is the mechanism that lets a class be built on top of another class. A subclass extends a superclass, inherits its fields and methods, and adds its own. The promise is the is-a relationship: if a `Dog` extends `Animal`, then a `Dog` is an `Animal`, and every place that accepts an `Animal` can accept a `Dog`.

That promise is what makes inheritance powerful and what makes it dangerous. Used at the right seam, it is the backbone of polymorphic code. Used as a shortcut to grab code, it is how hierarchies rot. This article is about what inheritance actually hands down, the shapes it takes, and the test that tells you whether you are modeling an is-a or just stealing code.

## Problem Statement

A team has a class `Vehicle` with a method `move()`. A `Car` extends it and works fine. A `Bicycle` extends it, and `move()` mostly works, but a bicycle cannot have an engine, so the `startEngine()` method inherited from `Vehicle` throws an exception in every bicycle. The codebase now has a `Bicycle` that is-a `Vehicle` but fails the method contract of a vehicle. Every caller that handles vehicles by type now has to special-case bicycles.

That is the inheritance failure that starts quietly. The subclass inherited behavior it cannot honor, and the parent's contract broke. The fix is not more special-casing, it is recognizing that `Bicycle` was never really a `Vehicle` in the way the code assumed. The design paid for modeling with inheritance, and the model was wrong.

## Core Concept

Inheritance is declared with `extends`. The subclass inherits the superclass's non-private fields and methods, and it can override methods to replace their behavior.

```java
public class Animal {
    protected String name;

    public void speak() {
        System.out.println("...");
    }
}

public class Dog extends Animal {
    public Dog(String name) {
        this.name = name;
    }

    @Override
    public void speak() {
        System.out.println("Woof");
    }
}
```

`Dog` now has `name`, has a `speak()` that replaces the parent's, and is assignable to `Animal`. That last point is the deep one: `Animal a = new Dog()` is legal, and a variable typed as `Animal` can hold a `Dog`. Inheritance is what makes the type system hierarchical, and the hierarchy is what later articles on polymorphism will lean on.

What a subclass receives, and what it does not, is worth being precise about. A subclass inherits the accessible fields and methods, `protected` and `public`, and package-private if it lives in the same package. It does not inherit `private` members, it does not inherit constructors, and it does not inherit static members in the way it inherits instance members. Constructors are the detail that trips everyone: a subclass constructor must call a superclass constructor, explicitly or implicitly, but the subclass does not get a copy of the parent's constructor. The parent's fields are allocated because the parent constructor runs, not because the subclass inherited a constructor.

Overriding is where subclass behavior is defined. A method with the same signature as a parent method replaces it, and `@Override` makes the compiler check that the override is real. The `super` keyword reaches the parent's version, which is how an override can extend rather than replace, calling `super.speak()` and then adding its own output. Overriding is what inheritance actually buys you: a parent method can be invoked on a child, and the child's version runs.

There is a rule hidden in overriding that causes production bugs. The compiler resolves which method to call based on the object's actual type at runtime, not the variable's type. `Animal a = new Dog(); a.speak()` runs `Dog.speak()`, because the object is a Dog. This is called dynamic dispatch, and it is the subject of the polymorphism article. Inheritance sets it up, and it is also what makes calling overridable methods from a constructor dangerous, because during the parent constructor the dispatch can reach a child method before the child's fields are set.

Now the types of inheritance, because the word covers several shapes and Java only allows some of them.

Type | Shape | Java support
--- | --- | ---
Single | One class, one parent | Yes
Multilevel | A chain, C extends B extends A | Yes
Hierarchical | One parent, many children | Yes
Multiple | One class, several parents | No, except via interfaces

Diagram: the four inheritance shapes and which ones Java allows.

```
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 820 680" font-family="Arial, Helvetica, sans-serif">
  <defs>
    <marker id="inh" markerWidth="10" markerHeight="8" refX="9" refY="4" orient="auto">
      <polygon points="0 0, 10 4, 0 8" fill="#ffffff" stroke="#57606a" stroke-width="1.5"/>
    </marker>
  </defs>

  <text x="215" y="82" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">Single</text>
  <rect x="160" y="110" width="120" height="60" rx="6" fill="#f6f8fa" stroke="#57606a" stroke-width="2"/>
  <text x="220" y="148" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">A</text>
  <rect x="160" y="230" width="120" height="60" rx="6" fill="#e8f0fb" stroke="#1f6feb" stroke-width="2"/>
  <text x="220" y="268" font-size="13" font-weight="bold" fill="#0a3069" text-anchor="middle">B</text>
  <line x1="220" y1="232" x2="220" y2="172" stroke="#57606a" stroke-width="2" marker-end="url(#inh)"/>

  <text x="605" y="82" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">Multilevel</text>
  <rect x="540" y="80" width="120" height="60" rx="6" fill="#f6f8fa" stroke="#57606a" stroke-width="2"/>
  <text x="600" y="118" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">A</text>
  <rect x="540" y="170" width="120" height="60" rx="6" fill="#e8f0fb" stroke="#1f6feb" stroke-width="2"/>
  <text x="600" y="208" font-size="13" font-weight="bold" fill="#0a3069" text-anchor="middle">B</text>
  <rect x="540" y="260" width="120" height="60" rx="6" fill="#e6f6ec" stroke="#1a7f37" stroke-width="2"/>
  <text x="600" y="298" font-size="13" font-weight="bold" fill="#033d16" text-anchor="middle">C</text>
  <line x1="600" y1="172" x2="600" y2="142" stroke="#57606a" stroke-width="2" marker-end="url(#inh)"/>
  <line x1="600" y1="262" x2="600" y2="232" stroke="#57606a" stroke-width="2" marker-end="url(#inh)"/>

  <text x="215" y="372" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">Hierarchical</text>
  <rect x="160" y="400" width="120" height="60" rx="6" fill="#f6f8fa" stroke="#57606a" stroke-width="2"/>
  <text x="220" y="438" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">A</text>
  <rect x="80" y="520" width="120" height="60" rx="6" fill="#e8f0fb" stroke="#1f6feb" stroke-width="2"/>
  <text x="140" y="558" font-size="13" font-weight="bold" fill="#0a3069" text-anchor="middle">B</text>
  <rect x="240" y="520" width="120" height="60" rx="6" fill="#e6f6ec" stroke="#1a7f37" stroke-width="2"/>
  <text x="300" y="558" font-size="13" font-weight="bold" fill="#033d16" text-anchor="middle">C</text>
  <line x1="140" y1="520" x2="205" y2="460" stroke="#57606a" stroke-width="2" marker-end="url(#inh)"/>
  <line x1="300" y1="520" x2="235" y2="460" stroke="#57606a" stroke-width="2" marker-end="url(#inh)"/>

  <text x="605" y="372" font-size="13" font-weight="bold" fill="#24292f" text-anchor="middle">Multiple (interfaces)</text>
  <rect x="460" y="400" width="120" height="60" rx="6" fill="#fff8c5" stroke="#9a6700" stroke-width="2"/>
  <text x="520" y="438" font-size="13" font-weight="bold" fill="#633c01" text-anchor="middle">I1</text>
  <rect x="620" y="400" width="120" height="60" rx="6" fill="#fff8c5" stroke="#9a6700" stroke-width="2"/>
  <text x="680" y="438" font-size="13" font-weight="bold" fill="#633c01" text-anchor="middle">I2</text>
  <rect x="540" y="540" width="120" height="60" rx="6" fill="#e8f0fb" stroke="#1f6feb" stroke-width="2"/>
  <text x="600" y="578" font-size="13" font-weight="bold" fill="#0a3069" text-anchor="middle">C</text>
  <line x1="570" y1="540" x2="518" y2="460" stroke="#57606a" stroke-width="2" stroke-dasharray="6,5" marker-end="url(#inh)"/>
  <line x1="630" y1="540" x2="682" y2="460" stroke="#57606a" stroke-width="2" stroke-dasharray="6,5" marker-end="url(#inh)"/>
</svg>
```

Single inheritance is one parent, one child. It is the default shape of most class hierarchies and the least controversial. Multilevel inheritance is a chain, and Java allows it, but a long chain is where the fragility starts, because each level inherits every level above it and the behavior becomes impossible to trace. Hierarchical inheritance is one parent with many children, which is how you model kinds of a thing, `Animal` with `Dog` and `Cat` below. Multiple inheritance is one class with several parents, and Java does not allow it for classes, because of the diamond problem.

The diamond problem is worth naming because it is why the rule exists. If `A` defines a method and `B` and `C` both inherit and override it, and `D` extends both, which version does `D` get? The ambiguity is unresolvable. Java sidesteps the whole question by refusing multiple class inheritance, and it recovers some of the power through interfaces, which carry no state and whose conflicts are resolved by explicit rules. The interface version of "multiple inheritance" is exactly what the diagram's fourth panel shows, and it is legal.

The is-a test is the gate before every `extends`. A subclass should be a genuine kind of the parent, a specialization, not a consumer of the parent's code. `Dog extends Animal` passes, a dog is an animal. `Stack extends ArrayList` is the famous failure, a stack is not a kind of array list, it is a different contract that happens to be buildable with a list. The test is brutal and it is right: if you cannot say "this child is a kind of parent" out loud and mean it, you should not be using `extends` at all.

## Real Production Usage

The Java standard library is the largest real inheritance hierarchy in daily use, and you read it constantly. `RuntimeException` and `Exception` both extend `Throwable`, and the hierarchy is why `catch (Exception e)` works at all. `ArrayList`, `LinkedList`, `Vector` all extend `AbstractList`, and the whole collection framework is a multilevel hierarchy built on interfaces and shared abstract classes. The library models kinds, a list is a kind of collection, an ArrayList is a kind of abstract list, and every level passes the is-a test.

Spring uses inheritance for its extension points, the abstract base classes and template method patterns that let a subclass fill in a step while inheriting the flow. `JdbcTemplate` and the `*Support` classes let a concrete DAO inherit the connection handling and override only the query. That is inheritance used as shared machinery, and it is the legitimate use, the kind-of relationship with real inherited implementation.

The exception hierarchy also shows the contract break, the warning from the problem statement. A method declared to throw `Exception` is a giant net, and a caller catching `Exception` catches both runtime and checked problems, blurring the contract. Inheritance gave the hierarchy, and using the hierarchy badly, catching too high, is how the flexibility leaks into buggy handling.

## Common Mistakes

The most common mistake is using inheritance to grab code. `Bicycle extends Vehicle` to reuse `move()` is not modeling a kind, it is importing a method, and it drags in `startEngine()` and the whole vehicle contract along with it. The is-a test catches it immediately. If the relationship is not a true kind, the code belongs in composition, which the end of this chapter covers, not in `extends`.

The second mistake is deep multilevel hierarchies. Three or four levels deep, the behavior of a single object is spread across every level, and no one can say which class owns what. The fragile base class problem is real: a change at the top of the chain silently changes behavior in every descendant. Prefer shallow hierarchies, and treat any chain beyond two or three levels as a design smell.

The third mistake is calling overridable methods from a constructor. During construction, dispatch goes to the most derived override, and the child's fields are not initialized yet, so the override reads null or zero. The fix is to not call overridable methods in constructors, or to design the base so its constructor does not depend on child state.

## Interview Perspective

The question "what are the types of inheritance" is a trap for fluent recitation, because the interesting part is not the list, it is the why. The weak answer names single, multilevel, hierarchical, and multiple. The strong answer adds the reason Java bans multiple: the diamond problem, and how interfaces recover the capability without the ambiguity.

The stronger answer connects to the real world. "Java has single class inheritance, so the collection framework uses interfaces for contracts and abstract classes for shared machinery, and that is why `ArrayList` can be many things at once, a list, a collection, an iterable, but has exactly one superclass." That answer shows inheritance as a design constraint, not a keyword.

Expected follow-ups: what is the diamond problem, and how do you test whether inheritance is the right relationship? The first wants the ambiguity explained, the second wants the is-a test stated and applied. The candidate who runs the is-a test on an example instead of reciting a definition is the one who will survive a real codebase.

## Knowledge Check

1. A `Square` extends `Rectangle`, and the rectangle's `setWidth` method is overridden so a square stays square. A caller holding a `Rectangle` calls `setWidth(10)` and the height changes too. What contract of `Rectangle` did the inheritance violate, and what test would have caught it before the code shipped?

2. Java allows multilevel inheritance but not multiple class inheritance. Explain the diamond problem in a few sentences, and describe how interfaces give Java a form of multiple inheritance without the ambiguity.

3. A class `Car` needs `move()` from `Vehicle`, `sellable()` from `Merchant`, and `insurance()` from `Insured`. Which of these would you express with `extends` and which with interfaces, and why?

## Key Takeaways

- Inheritance is the is-a relationship; a subclass inherits accessible fields and methods but never constructors.
- Java allows single, multilevel, and hierarchical inheritance, and multiple only through interfaces.
- The diamond problem is why multiple class inheritance does not exist in Java.
- Run the is-a test before every `extends`; if it fails, you are stealing code, not modeling a kind.

## What's Next

Inheritance builds the hierarchy, but a hierarchy is only useful if callers can use the parent type and get child behavior. The next article, Polymorphism: Compile-Time vs Runtime, explains exactly how that works: the two flavors of "many forms," which one the compiler decides, which one the JVM decides at runtime, and why the runtime flavor is the one that powers real object-oriented design.

---

This article explains inheritance as the is-a mechanism, what a subclass actually inherits, and the four shapes it can take, including why Java refuses multiple class inheritance. Its strongest claim is that inheritance is a model of kinds, not a way to reuse code, and the is-a test should gate every `extends` you ever write.
