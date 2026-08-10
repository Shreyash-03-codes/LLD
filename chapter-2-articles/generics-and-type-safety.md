# Generics and Type Safety

## Learning Objectives

1. Use generic classes and methods so the compiler catches type errors instead of the JVM, and say exactly which errors those are.
2. Explain erasure, what the JVM actually sees after compilation, and which generics behaviors erasure forbids.
3. Read and write bounded type parameters and wildcards, and choose between `extends` and `super` by what the code produces and consumes.

## Introduction

Generics let a class or method take a type as a parameter. `List<String>` is a list that holds strings, `Optional<Order>` is an optional that either holds an order or holds nothing, and `Map<String, Long>` maps string keys to long values. The syntax is type parameters in angle brackets, and the value is type safety at compile time: the compiler checks what goes in and what comes out before the program ever runs.

The deep fact about Java generics, the one that explains almost every surprise, is that the type parameters exist only at compile time. At runtime the JVM has no idea a `List` was meant to hold strings. The compiler checks the types, inserts casts where they are needed, and erases the type arguments from the bytecode. Understanding erasure is the difference between using generics and being confused by them.

## Problem Statement

The pre-generics world is the problem, and it is worth one concrete look because it defines the value of the feature. A class held a collection of everything:

```
List users = new ArrayList();
users.add("alice");
users.add(new Long(7));
```

The list accepted both, because without a type argument it accepted `Object`. The bug arrives when the code reads:

```
String name = (String) users.get(1);
```

and the JVM throws `ClassCastException`, because `get(1)` returned a `Long`, not a string. The failure happens at runtime, in a line far from the line that inserted the wrong object, after the program is already deployed and processing a user's data. The error message says nothing about where the wrong value came from.

The failure mode is mislabeled as "the developer forgot a cast." The real failure is that the type system gave up. The collection could not say what it held, so nothing could be checked when the wrong thing went in, and the wrong thing was found only when it came out. Type safety is the guarantee that this class of failure cannot happen, and generics are how Java delivers it.

## Core Concept

A generic class declares a type parameter and uses it in fields, parameters, and return types:

```
public class Box<T> {
    private T value;

    public void put(T value) { this.value = value; }
    public T get() { return value; }
}
```

Now `Box<String>` can only ever hold a string, and `String s = box.get()` needs no cast. The compiler knows `get()` returns `T`, and for this instance `T` is `String`. The wrong type cannot go in, because `put(new Long(7))` on a `Box<String>` does not compile. The error moved from runtime to compile time, which is the entire point.

Generic methods work the same way, with the type parameter declared before the return type:

```
public static <T> T first(List<T> items) {
    return items.get(0);
}
```

The caller does not name the type; the compiler infers it from the arguments, and the return type is precise. Call it with `List<String>` and it returns `String`. This is type inference, and it is why `Collections.emptyList()` returns something assignable to `List<Order>` without a cast.

Bounds narrow the type parameter. `public static <T extends Comparable<T>> T max(List<T> items)` says T must be comparable to itself, which lets the method call `compareTo` on T without a cast. The bound is the contract the generic code needs, and without bounds the generic code can only treat T as `Object`.

Wildcards express "some type, I do not care exactly which." `List<? extends Number>` is a list whose element type is some subtype of `Number`, and `List<? super Integer>` is a list whose element type is some supertype of `Integer`. The classic rule, producer extends, consumer super, comes from the reading: if the list gives you elements, declare `? extends`, if the list takes elements, declare `? super`. A list declared `? extends Number` cannot take a number, because the list might be a `List<Integer>` and a `Double` would not fit, which is why only `get` is safe on it.

Now the heart: erasure. After the compiler finishes, all the `T` and the type arguments are gone. `Box<String>` and `Box<Order>` compile to the same bytecode, a single `Box` class whose `T` was erased to `Object` and whose casts were inserted at the use sites. The JVM runs one class, and the type safety is a compile-time artifact, already verified and then discarded.

Erasure explains the rules that otherwise feel like arbitrary restrictions. You cannot `new T()`, because there is no `T` at runtime to construct. You cannot write `T[]` and expect a typed array, because arrays check their component type at runtime and the runtime has no component type, so generic arrays are either forbidden or an unchecked cast. You cannot do `instanceof T`, because there is nothing to test against. You cannot have two overloads that differ only in type arguments, because `handle(List<String>)` and `handle(List<Long>)` erase to the same signature and the compiler refuses. And a static field cannot use the class's type parameter, because the parameter belongs to instances, not to the class.

There is a subtlety worth stating plainly because it trips everyone once: generic types are invariant. `List<String>` is not a `List<Object>`, and `List<Object>` is not a `List<String>`, even though `String` is an `Object`. The reason is that a `List<Object>` could receive a `Long`, and then code holding it as `List<String>` would read a `Long` as a string. Invariance is type safety on the collection level, the same guarantee that protects the element level. If you need covariance, you use the wildcard, `List<? extends Object>`.

## Real Production Usage

The JDK is built on generics, and the collections are the daily evidence. `Map<String, String>` keys, `List<Transaction>`, `Optional<User>`, `Stream<Order>`, all compile-time checked, all erased at runtime. The standard library's `Comparator`, `Function`, and `Consumer` are all generic interfaces, and the polymorphism article's callback example was a generic `Function` all along.

The generic repository is a production pattern worth naming. A `JpaRepository<T, ID>` interface in Spring Data lets one declaration produce a repository for any entity:

```
public interface OrderRepository extends JpaRepository<Order, Long> { }
```

Spring generates the implementation at runtime, and the calls are type safe: `findById(Long)` returns `Optional<Order>`, not an object to cast. This is generics plus runtime generation, and it shows the division of labor: the compiler checks the types, and the framework does the erasure-time work.

The places erasure hurts are real production facts. Reflection on generic types needs the `TypeToken` pattern or a framework's type capture, because the runtime does not keep type arguments. Any code that must know "the actual T at runtime" has to smuggle it in, usually by passing a `Class<T>` parameter. And a raw type, `List` without the angle brackets, is legal but carries an unchecked warning, because the compiler cannot check the cast it has to insert, and it silently reintroduces the exact bug from the problem statement.

## Common Mistakes

The first mistake is the raw type. `List users = new ArrayList()` compiles, and the IDE flags an unchecked operation, and the flag is the problem statement knocking. A raw type is a promise to the compiler that you will handle every cast yourself, and nobody does. Always give the type argument, even when it is `List<?>`.

The second mistake is assuming invariance is a bug. A method `print(List<Object>)` will not accept a `List<String>`, and the developer "fixes" it with a cast or a raw type, losing the safety the invariance was protecting. The correct move is `print(List<?>)` or a bound, not a cast.

The third mistake is the wildcard misuse, usually `List<? extends Number>` where the code wants to add numbers. The declaration allows reading, not writing, and the `add` call fails to compile, and the failure is correct, because the list might be a `List<Integer>`. The reader who understands producer-extends-consumer-super fixes the declaration instead of casting.

The fourth mistake is ignoring erasure and writing the forbidden things, `new T()`, `T[]`, `instanceof T`, and then being surprised the compiler refuses. The refusal is not a limitation to work around with casts, it is the feature working: the runtime genuinely cannot do these, and an unchecked cast is the code promising safety the compiler cannot verify.

## Interview Perspective

The question "explain type erasure" wants the mechanism in two moves. "The compiler verifies the type arguments, inserts casts, and removes the type parameters from the bytecode, so all generic types share one class at runtime." The candidate who then connects it to the consequences, no `new T()`, no `instanceof T`, no overload by type argument, has the full model.

The question "why can't `List<String>` be a `List<Object>`" wants invariance. "Because a `List<Object>` could contain a `Long`, and code holding it as `List<String>` would then cast a `Long` to `String` and crash. The compiler blocks it at the collection level." The wildcard follow-up wants the PECS reading on a concrete case.

The practical question "when would you use `? extends` versus `? super`" wants the rule and an example. "When a method only reads elements, declare `? extends T` so any subtype's collection works. When it only adds, declare `? super T` so it accepts a collection of the type or any supertype." The candidate who can produce both directions on a `List` is done.

## Knowledge Check

1. Write a generic `Result<T>` that holds a value and a flag for whether it is an error, and show why `Result<String>` cannot accidentally hold an `Order`.

2. Explain why `new T()`, `T[]`, and `instanceof T` all fail to compile, using the erasure model, not the rule list.

3. A method `average(List<? extends Number>)` computes a mean. State which wildcard it uses and why, and what breaks if you switch it to `List<Number>`.

## Key Takeaways

- Generics move type errors from runtime to compile time, and the compiler inserts the casts it checks.
- Erasure is the model: type arguments vanish at runtime, one class per generic type, no `T` to construct, test, or make arrays from.
- Generic types are invariant; reach for wildcards, `? extends` and `? super`, when you need variance.
- A raw type is a promise to handle every cast yourself, and it reintroduces the bug the feature exists to kill.

## What's Next

Types have been parameters, and relationships have been drawn. The last article in the chapter puts the tools together and makes a judgment that has outlived every other OOP recommendation. The next article, The Golden Rule: Composition Over Inheritance, is why the inheritance article's warning gets sharper, why `extends` is a decision and not a habit, and how the composition discipline keeps a hierarchy from becoming a liability.

---

This article explains generics, the feature that lets the compiler check what a collection holds and erases every type argument before the JVM runs. Its strongest claims are that erasure is the model behind every generics rule, that generic types are invariant, and that a raw type silently reintroduces the exact runtime failure the feature was built to remove.
