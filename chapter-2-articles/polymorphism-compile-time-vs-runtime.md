# Polymorphism: Compile-Time vs Runtime

## Learning Objectives

1. Separate the two flavors of polymorphism, one decided by the compiler, one by the JVM at runtime.
2. Explain why the runtime flavor is the one that makes object-oriented design work.
3. Code against the parent type and get the child's behavior, and explain why that is a superpower.

## Introduction

Polymorphism means many forms, and in Java it means one name that does different things depending on context. The word covers two unrelated mechanisms that share the name, and confusing them is a guaranteed interview and design error.

Compile-time polymorphism is method overloading. Several methods with the same name and different parameter lists, and the compiler picks which one to call based on the argument types, all before the program runs. Runtime polymorphism is method overriding. A parent class declares a method, a child overrides it, and the JVM decides which version runs based on the actual type of the object, at the moment of the call. Same name, same signature, different behavior, chosen while the program is already running.

## Problem Statement

A system needs to render several kinds of content, text, image, video. The first version works with an `if` chain on type. `if (content instanceof Text) { renderText(...) } else if (content instanceof Image) { renderImage(...) }`. Every new content type adds a branch to the chain, and every place that switches on type adds the same branch. A fourth type arrives, and the team finds the switch in three files, missing one, and a video renders as text in that path.

The if-chain is the absence of runtime polymorphism. The code is asking "what type is this, so I can pick the behavior," when the object already knows its own behavior. The polymorphic version hands the object to a single `render()` method, and each content class implements `render()` for itself. The new type adds a class, not a branch, and there is no switch to forget to update. The failure was not the if-chain's logic, it was the design asking the caller to know what the object already knows.

## Core Concept

Start with the two mechanisms side by side, because the similarity of the names is where the confusion lives.

Overloading: same method name, different parameter list, same class or different classes, resolved by the compiler. `print(int)`, `print(String)`, `print(double)` can all exist in one class. When you write `print(5)`, the compiler picks `print(int)` based on the static type of the argument, and the choice is baked into the bytecode. Overloading is a convenience for the caller, a set of entry points with the same name, and it has nothing to do with objects deciding their behavior.

Overriding: same method name, same signature, in a subclass, resolved by the JVM at runtime. The compiler sees `Animal.speak()` on a variable typed `Animal`. The JVM looks at the actual object, discovers it is a `Dog`, and runs `Dog.speak()`. The variable's type is irrelevant once the program runs. This is dynamic dispatch, and it is the heart of the second flavor.

Compile-time (overloading) | Runtime (overriding)
--- | ---
Same name, different parameters | Same name, same signature
Resolved by the compiler | Resolved by the JVM, at the call
Static types of arguments decide | Actual type of the object decides
A convenience for callers | The mechanism of polymorphic design

The runtime flavor is the one that matters for design, and here is the argument for it. A method that accepts a parent type can be written once and behave correctly for every child type that exists, including children that do not exist yet. `void notify(Notification n)` calls `n.send()`, and whatever subclass arrives next year implements its own `send()` and the caller works unchanged. The design depends on the contract, `Notification`, not on the concrete class, and every concrete class that honors the contract plugs in. That is open-closed thinking, and it is what runtime polymorphism gives you.

The mechanism underneath is worth being precise about, because it explains why the variable's type does not matter. An object in the heap carries its actual class, the JVM knows it is a `Dog` even when the variable says `Animal`. When the JVM invokes a virtual method, it looks up the method by signature in the object's actual class and runs the override. There is no ambiguity, the object always knows what it is. Overriding is the memory model article's "the object knows its type" made into a dispatch mechanism.

Three details sharpen the picture.

The `@Override` annotation. It is a compiler check, not a behavior. It verifies that the method really does override a parent method, and it catches the embarrassing case where you meant to override but the signature was wrong and you silently overloaded instead. Always write it, and let the compiler tell you when the parent contract changed.

Covariant return types. An override may return a subtype of the parent's return type. `Animal getFactory()` can be overridden with `Dog getFactory()` when the child genuinely produces dogs. The return type is part of the override contract, and covariance keeps the child honest about what it makes.

The signature rule. To override, the signature must match, same name, same parameters, same return type or a covariant one. Change any parameter and you are no longer overriding, you are overloading, and the JVM will not call your method polymorphically. This is the single most common silent break: a subclass method with a slightly different parameter list looks right and does nothing when called through the parent.

There is a caution that belongs here and it was flagged in the constructors article. Because dispatch goes to the runtime type, a method called from a parent constructor runs the child override before the child's fields exist. `Animal`'s constructor calls `speak()`, and `Dog.speak()` reads a field that is still null. The rule: do not call overridable methods from constructors, and if the parent must touch something, make the method `final` or `private` so dispatch cannot escape the parent.

## Real Production Usage

The standard library is a museum of runtime polymorphism, and streams make it visible. `Stream.map(Function)` accepts a `Function`, and the function's `apply` method is dispatched to whatever lambda or class you passed. The stream code does not know what the function does, it only knows the contract, and the behavior arrives at runtime. Every callback, every listener, every handler interface in the JDK runs on this same dispatch.

Spring is runtime polymorphism turned into a framework. A `@Service` is a class that implements some contract, and callers are wired to the contract, not the class. The container decides at runtime which bean satisfies which injection point, and swapping an implementation for a mock or a second version requires no change to the caller. The whole dependency injection model is the runtime-flavor principle: depend on the type, let the runtime supply the instance.

The strategy pattern is the same idea in miniature, and you will write it in interviews. An interface `PricingStrategy` with several implementations, and a `Ticket` that holds a `PricingStrategy` and calls `compute(price)` on whatever strategy it was given. The ticket does not know which strategy, it knows the contract, and the strategy changes the behavior at runtime. That is the entire point of the pattern, and it only works because of dynamic dispatch.

## Common Mistakes

The most common mistake is confusing overloading and overriding, usually by describing both as "the same method doing different things." The distinction that matters is who decides and when. Overloading is the compiler, before runtime, based on static types. Overriding is the JVM, during runtime, based on the actual object. The engineer who cannot state that will misdiagnose a lot of behavior.

The second mistake is breaking an override by accident, usually by adding or changing a parameter and losing the `@Override` check. The method compiles, it looks like an override, and it is actually a new overload that the dispatch never reaches. The parent method runs instead, and the bug is invisible until someone checks which version actually executed. The `@Override` annotation is the tripwire, and skipping it is how the break sneaks in.

The third mistake is using `instanceof` chains where polymorphism should do the work. Every switch on type is a sign the object's own behavior was not put in the object. Not every `instanceof` is wrong, frameworks use it legitimately, but in your domain code, a chain of type checks is usually a design asking to be replaced by one overridden method.

## Interview Perspective

The classic question is "overloading vs overriding," and the classic weak answer is a description of each. The strong answer states the deciding factors: overloading is resolved by the compiler using static argument types, overriding is resolved by the JVM using the object's runtime type, and only overriding gives you dynamic dispatch.

The stronger answer demonstrates it. "A method parameter typed `Animal` calls `speak()`, and the JVM runs `Dog.speak()` because the object is a Dog. Overloading would be two methods named `print`, one for `int` and one for `String`, where the compiler chooses before runtime." The candidate who can show both mechanisms with concrete examples has the actual distinction.

Expected follow-ups: why does a parent-typed variable call the child method, and what breaks when you change a parameter in an override? The first wants the object-carries-its-type explanation, the second wants the overload-not-override failure, the accidental overload, and the `@Override` tripwire.

## Knowledge Check

1. A class has `void handle(Animal a)` and `void handle(Dog d)`. A caller with `Animal x = new Dog(); handle(x);` invokes which version? Explain why, and what this says about the limits of compile-time polymorphism.

2. A subclass changes an overridden method's parameter type, adds `@Override`, and the compiler errors. Explain what the annotation caught, and what would have happened silently without it.

3. A `render()` switch on type is replaced with a polymorphic `render()` on each content class. Name the concrete ways the system improves, and identify what the caller's code looks like before and after.

## Key Takeaways

- Overloading is compile-time, decided by static argument types; overriding is runtime, decided by the object's actual type.
- Runtime polymorphism lets one caller, written against the parent type, work for every future child type.
- `@Override` is the tripwire that catches accidental overloads and broken contracts.
- A chain of `instanceof` in domain code is usually a polymorphic method waiting to be written.

## What's Next

Polymorphism is about objects deciding their own behavior, and the relationships between those objects are the next subject. The next article, Association, Aggregation, and Composition, covers the has-a family of relationships, how they differ from inheritance, and how the arrows between your classes say which objects outlive the others.

---

This article explains the two flavors of polymorphism, compile-time overloading decided by the compiler and runtime overriding decided by the JVM, and why only the second powers real design. Its strongest claim is that a caller written against the parent type works for every future child type, and that an instanceof chain is usually the polymorphic method the code forgot to write.
