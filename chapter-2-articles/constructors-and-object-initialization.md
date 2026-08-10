# Constructors and Object Initialization

## Learning Objectives

1. Explain the two-phase nature of object creation: `new` allocates, the constructor initializes.
2. Predict the exact order in which fields, initializer blocks, and constructor bodies run.
3. Explain why some classes need a no-arg constructor and why some constructors should be private.

## Introduction

When you write `new Person("Alice", 30)`, two things happen in sequence. The JVM allocates a block of heap for the object, and then a constructor runs to fill that block with meaningful state. The allocation is the boring part. The constructor is where the object's contract starts, and it is the only code that is guaranteed to run before the object is first used.

Constructors exist because an empty object is a hazard. A `Person` with a null name is not a person, it is a bug waiting for the first method call. The constructor is the place where the class enforces that no instance is ever born invalid. Every design decision about constructors, which ones exist, whether they chain, whether they are private, is a decision about how hard it is to create a broken object.

## Problem Statement

An entity class gains a new field. The team adds the field, updates the builder, updates one of the two constructors, and forgets the other. The next deploy starts, and every object built through the forgotten path has a null where the new field should be. The bug is not caught in tests because the test suite builds objects through the updated path. Production finds it first.

That is the everyday constructor failure: multiple ways to create an object, and not all of them establish the same invariants. The class allowed itself to be born in two shapes, and only one was valid. The fix is not discipline, it is design. Fewer construction paths, one of them canonical, and the others delegating to it, mean there is only one place where the invariants are set, and only one way to get them wrong.

## Core Concept

Start with the default constructor, because most beginners assume it does not exist. If a class declares no constructors at all, the compiler writes a no-arg constructor for it. It takes no arguments, calls the parent's no-arg constructor, and does nothing else. Fields that were initialized at declaration keep their values, and fields that were not stay at their zero values, null, 0, false.

The moment you declare any constructor, the compiler stops synthesizing the default. That is the most common surprise in the whole topic. A class with one parameterized constructor and no declared no-arg constructor cannot be created with `new Person()`. The compiler will not help you, because by declaring one constructor you told it you are taking over. If callers need a no-arg form, you write it yourself.

Constructors can call each other, and this is how the "one canonical path" idea becomes code. The `this(...)` call invokes another constructor of the same class.

```java
public class Person {
    private final String name;
    private final int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public Person(String name) {
        this(name, 0);   // delegate to the full constructor
    }
}
```

The two-argument constructor is canonical. The one-argument version is a convenience that delegates. Every real invariant is set in exactly one place, and any future field added to the full constructor is automatically handled by every delegation path. That is the structural reason to prefer delegation: the number of places that can get initialization wrong stays at one.

The other chaining direction is `super(...)`. Every constructor must call a superclass constructor, either explicitly or implicitly. If you do not write the `super(...)` call, the compiler inserts `super()` for you, calling the parent's no-arg constructor. If the parent has no no-arg constructor, the compiler cannot help, and you must call `super(args)` explicitly. Inheritance is later in this chapter, but the rule belongs here: a child object is built bottom up, parent state first, then child state. You cannot use a field of the child before the parent constructor has run, and you cannot even meaningfully use `this` methods that depend on the child's state before the chain completes.

Field initialization happens around the constructors, and the order is exact.

Order | What runs
--- | ---
1 | Superclass chain completes first, parent fields then parent constructor body
2 | Field initializers of this class, in declaration order
3 | Instance initializer blocks of this class, in order they appear
4 | This class's constructor body

The field initializers and initializer blocks run before the constructor body, which is why a field with a declared default is already set when the constructor body begins. A constructor assignment then overrides the field initializer. There is a subtlety hidden here: the initializers run after the parent constructor but before the child constructor body, so a method called from the parent's constructor dispatches to the child, and the child's fields are not yet initialized. This is why calling overridable methods from a constructor is a known hazard, discussed further in the inheritance article.

Two more constructor features that shape real code. A `final` field must be assigned exactly once, in the constructor or at declaration, and the compiler enforces it. This is the backbone of immutability: a class whose fields are all final, with no setters, cannot change after construction. And a constructor can be `private`, which is the standard trick for classes that should not be created directly. A private constructor plus a static factory method, or the builder pattern with a private constructor, gives the class control over how it is created, including caching instances or validating arguments before allocation.

## Real Production Usage

The ORM world is where constructor policy stops being a style question and becomes a runtime requirement. Hibernate needs a no-arg constructor on entities, because it instantiates the entity class reflectively, without arguments, when it loads a row, and then sets fields directly through field access or setters. An entity without a no-arg constructor throws an exception at the least convenient moment. This is the default-constructor rule meeting a real framework.

Spring's constructor injection is the pattern the framework pushes for and it is a constructor-design lesson in disguise. When a bean's dependencies are passed through its constructor, the bean cannot be created without them, which means a partially wired bean cannot exist. Field injection allows a bean to be constructed empty and then patched, which hides wiring mistakes until the first method call. The constructor is the guarantee that the object is born complete, which is the exact principle this article has been arguing for.

The builder pattern, used across libraries and your own code, is a comment on constructors. A class with many optional fields would need either a constructor with twelve arguments or twelve overloads. The builder moves the parameter-setting into method calls and then calls one private, canonical constructor at the end. The private constructor is the gate: there is exactly one way into the object, and the builder is the only road to it.

## Common Mistakes

The most common mistake is losing the no-arg constructor by declaring any other constructor and not realizing the compiler stopped synthesizing it. The class compiles fine, and the first failure is in the framework, the serialization library, or the test that does `new Person()` and gets a compile error. The rule to internalize: declaring any constructor disables the default.

The second mistake is letting an object escape before it is fully constructed. Publishing `this` inside the constructor, to a registry, to another thread, or into a collection, means someone else can observe the object before the constructor body finishes. If the class is mutable, that observable half-built state is a real concurrency bug. The fix is to not publish `this` from the constructor, and to make the object immutable if you need to share it during construction.

The third mistake is the constructor that does real work it should not do. A constructor that opens a connection, sends an email, or starts a thread makes object creation a side effect, and every test that wants a plain object has to dodge that work. The constructor should establish invariants and stop. Work that is not about the object's state belongs in a factory or a lifecycle method, not in the birth ceremony.

## Interview Perspective

Interviewers use constructors as a lever to test whether a candidate thinks in terms of object validity, not just syntax. The question "why does Hibernate need a no-arg constructor on an entity" separates the candidate who knows the default-constructor rule from the candidate who can reason that a framework instantiates classes reflectively and therefore cannot pass arguments. The second one is the answer that counts.

Constructor versus setter injection is the same lever wearing a different coat. The strong answer says constructor injection makes the object's dependencies part of its birth, so an incompletely wired object cannot exist, while setter injection allows a half-created object. The weak answer lists pros and cons without tying them to validity. The candidate who says "constructor injection is the guarantee that the object is born complete" is describing the same principle this article is built on.

Expected follow-ups: what order do fields and constructor bodies run in when a class has a superclass, and why is calling an overridable method from a constructor dangerous? Both reward the candidate who can walk the initialization sequence and see where the child state is not yet ready.

## Knowledge Check

1. A class declares one constructor, `Person(String name)`. Does `new Person()` still compile? Explain what happens to the default constructor and what you would write to restore the no-arg form.

2. A class has a field initialized at declaration, `int x = 5`, and a constructor that assigns `this.x = 10`. What value does the object end up with, and what is the order of operations that produces it?

3. A child class has a parent with only a parameterized constructor. The child's constructor compiles only with an explicit `super(...)`. Explain why the implicit `super()` cannot work, and describe the order in which parent and child fields become usable.

## Key Takeaways

- `new` allocates and the constructor initializes; the two phases are separate, and the constructor is where validity is enforced.
- Declaring any constructor disables the compiler's default, which is why no-arg constructors vanish without warning.
- Delegate from convenience constructors to one canonical constructor so invariants are set in one place.
- The initialization order is fixed: superclass first, then fields and initializers, then the constructor body.

## What's Next

A constructor sets the state, but it does not decide who gets to read and change that state afterward. That decision is the subject of the next article. Encapsulation and Access Modifiers covers how Java lets a class keep its fields private and expose only what the outside world is allowed to touch, and why that wall is what makes the object model safe to use at all.

---

This article explains that constructors are the birth ceremony of an object, the place where an allocated heap block is turned into a valid instance, and walks the exact order of initialization. Its strongest claim is that every way to create an object should delegate to one canonical constructor, because the number of places that can initialize a field wrong should be exactly one.
