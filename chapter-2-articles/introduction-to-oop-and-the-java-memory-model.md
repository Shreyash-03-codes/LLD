# Introduction to OOP and the Java Memory Model

## Learning Objectives

1. State what object-oriented programming gives you that structured code does not, and what it costs.
2. Trace where every variable actually lives, on the stack or on the heap, and explain why primitives and references behave differently.
3. Explain why most Java "memory bugs" are reference bugs, not allocation bugs.

## Introduction

Object-oriented programming is a way of organizing a program around objects, bundles of state and behavior that can be created, referenced, and discarded independently. Java is a language that forces the model on you. Everything except the primitives is an object, every object lives in a heap, and every variable you touch is either a primitive value or a reference to something in that heap.

The Java memory model is the physics under that abstraction. You can write correct code for years without naming it, and you can write confident buggy code for years without naming it too. This article is the foundation for everything else in the chapter: classes, inheritance, polymorphism, all of them assume you know what an object is, where it lives, and what a reference actually is.

## Problem Statement

A junior engineer is asked why a method returns an old value after another method already updated it. The answer "it was a copy" is accepted and nothing changes. The real answer lives in the memory model. The value was a primitive, so it was copied by value into the method. Or the value was an object, so the reference was copied and both names point at the same heap object. The two cases behave completely differently, and the engineer who cannot see the difference is guessing at every synchronization bug, every stale cache, every accidental shared mutation.

The failure is concrete. A team debugs a race for a week. The field is mutable and shared, and everyone who reads the code reads the variable, not the reference. The fix, "make it immutable," is applied as a rule, not as a consequence. The engineer who understands that a reference is the only thing standing between two threads and one heap object fixes it in an afternoon and can explain why.

## Core Concept

Start with the two regions where your program's data lives.

The stack is per-thread, fast, and tiny. Every time a method is called, the JVM pushes a frame onto the thread's stack. The frame holds the method's local variables and bookkeeping. When the method returns, the frame is popped and its contents are gone. Stack storage is allocated by moving a pointer and freed by moving it back. Nothing is ever allocated on the stack in Java except primitives and references that are local to a method.

The heap is shared by all threads, slower, and large. Every object created with `new` lives there. Objects in the heap are not freed by the programmer. The garbage collector reclaims them when nothing references them anymore. A reference is a handle, a pointer in disguise, that tells the JVM where in the heap the object lives.

Here is the shape of it.

Diagram: where stack variables and heap objects sit, and how references connect them.

```
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 820 560" font-family="Arial, Helvetica, sans-serif">
  <defs>
    <marker id="ref" markerWidth="10" markerHeight="8" refX="9" refY="4" orient="auto">
      <polygon points="0 0, 10 4, 0 8" fill="#d1242f"/>
    </marker>
  </defs>

  <text x="190" y="52" font-size="16" font-weight="bold" fill="#24292f" text-anchor="middle">STACK</text>
  <text x="590" y="52" font-size="16" font-weight="bold" fill="#24292f" text-anchor="middle">HEAP</text>

  <rect x="40" y="70" width="300" height="430" rx="10" fill="none" stroke="#8b949e" stroke-width="2" stroke-dasharray="8,6"/>
  <rect x="400" y="70" width="380" height="430" rx="10" fill="none" stroke="#8b949e" stroke-width="2" stroke-dasharray="8,6"/>

  <rect x="60" y="110" width="260" height="130" rx="6" fill="#f6f8fa" stroke="#57606a" stroke-width="2"/>
  <text x="80" y="138" font-size="13" font-weight="bold" fill="#24292f">doSomething()</text>
  <line x1="80" y1="150" x2="300" y2="150" stroke="#d0d7de" stroke-width="1"/>
  <text x="80" y="176" font-size="12" fill="#24292f">String s -&gt;</text>
  <text x="80" y="206" font-size="12" fill="#24292f">int x = 7</text>

  <rect x="60" y="310" width="260" height="160" rx="6" fill="#f6f8fa" stroke="#57606a" stroke-width="2"/>
  <text x="80" y="338" font-size="13" font-weight="bold" fill="#24292f">main()</text>
  <line x1="80" y1="350" x2="300" y2="350" stroke="#d0d7de" stroke-width="1"/>
  <text x="80" y="380" font-size="12" fill="#24292f">int total = 42</text>
  <text x="80" y="414" font-size="12" fill="#24292f">Person p -&gt;</text>
  <text x="80" y="444" font-size="12" fill="#24292f">int flag = 1</text>

  <rect x="440" y="120" width="320" height="140" rx="6" fill="#e8f0fb" stroke="#1f6feb" stroke-width="2"/>
  <text x="460" y="148" font-size="13" font-weight="bold" fill="#0a3069">Person</text>
  <line x1="460" y1="160" x2="740" y2="160" stroke="#a5c1e8" stroke-width="1"/>
  <text x="460" y="188" font-size="12" fill="#1f4484">name: String</text>
  <text x="460" y="214" font-size="12" fill="#1f4484">age: int = 29</text>

  <rect x="440" y="300" width="300" height="70" rx="6" fill="#e6f6ec" stroke="#1a7f37" stroke-width="2"/>
  <text x="460" y="328" font-size="13" font-weight="bold" fill="#033d16">String</text>
  <line x1="460" y1="338" x2="720" y2="338" stroke="#b4d9c4" stroke-width="1"/>
  <text x="460" y="358" font-size="12" fill="#166534">value: "Alice"</text>

  <line x1="265" y1="414" x2="440" y2="170" stroke="#d1242f" stroke-width="2" marker-end="url(#ref)"/>
  <line x1="255" y1="176" x2="440" y2="330" stroke="#d1242f" stroke-width="2" marker-end="url(#ref)"/>
  <line x1="745" y1="190" x2="545" y2="300" stroke="#d1242f" stroke-width="2" marker-end="url(#ref)"/>
</svg>
```

Read the diagram from the bottom left. `main()` holds a primitive `total` and a reference `p`. The primitive is a value sitting in the frame. The reference is a pointer to a `Person` object in the heap. A second method `doSomething()` has its own frame with its own primitive `x` and its own reference `s`. Nothing is shared between the frames except what the references point at. The `Person` object holds its own references too, one to a `String` for the name. Every arrow is a reference, and a reference is the only way two parts of the program see the same object.

The rule that follows, and it is the rule this whole article stands on: primitives are copied by value, references are copied by value, and objects are never copied by assignment. When you assign `Person q = p`, you copy the reference. Now two names point at one object. When you pass `p` to a method, you copy the reference again. Mutating `q.age` changes what `p` sees, because there is only one object. Mutating a local `int` changes nothing outside the method, because the int was copied as a value.

That single distinction explains most Java behavior that looks like magic or bugs. The stale value returned from a method is usually a primitive copy or an object reference whose state was read at the wrong time. The "shared state" bug is usually a reference that escaped a scope and is now reachable from two threads. The immutable class is the design that makes the reference safe to share, because nobody can reach through it to mutate.

A few consequences of the model that engineers trip on repeatedly. Local primitives and references are thread-safe by default because each thread has its own stack; the danger is always an object reached through a shared reference. Method-local objects are not thread-safe just because they are local, if the reference escapes and someone else gets it. And `==` on references compares the pointer, not the contents, which is why two `String`s with the same text can fail `==` while `.equals` succeeds.

## Real Production Usage

`java.util.concurrent` is the standard library's answer to the shared-reference problem. The classes in that package exist because objects on the heap are reachable from multiple threads and the reference gives no protection on its own. `ConcurrentHashMap` exists because a plain `HashMap` shared through references will corrupt itself under concurrent writes. The point is not the classes, it is the underlying model they assume: shared heap, reference-based access, no implicit synchronization.

The JVM's escape analysis is a quiet confirmation of the model. When the JVM proves an object never escapes the method that created it, it can allocate that object on the stack or in registers instead of the heap, and it can even avoid allocating it at all. You cannot see this happening, and you should not try to control it. It is evidence that the model is exactly what the earlier diagram drew: the stack holds whatever cannot be seen elsewhere, and the heap holds whatever can.

Strings deserve a mention because they are the most common object in every Java program and the source of the most "is it a copy" confusion. A `String` is an object on the heap, but its contents are immutable, so sharing a reference to a `String` is safe. The language even interns string literals, reusing one object for the same literal text, which is why `==` can sometimes work on strings and usually should not be relied on.

## Common Mistakes

The most common mistake is treating primitives and references as if they behave the same. Engineers who learned C think of everything as a value and are surprised when an object "changes" through a copied reference. Engineers who learned Python think of everything as a reference and are surprised that `int` arithmetic never affects the caller. Java sits in the middle: primitives are values, objects are reached by reference, and there is no keyword that changes that.

The second mistake is thinking `new` allocates on the stack or that you can somehow control where things go. You cannot. The JVM decides, the escape analyzer decides, the garbage collector decides. The only thing you control is whether you hand the reference to someone else. Most "memory leaks" in Java are not allocations, they are references kept alive by a collection or a cache when the object is no longer needed.

The third mistake is using `==` on objects to test equality. On primitives `==` compares values. On references it compares addresses. The two strings that look identical and compare unequal are not a JVM quirk, they are two objects. The engineer who memorized "use equals" without the memory model behind it will still be surprised by the next reference trap.

## Interview Perspective

Every OOP interview opens with the model whether the interviewer says so or not. The question "is Java pass by value or pass by reference" is the memory model wearing an interview costume. The weak answer is the one-word version, "by value," because it is technically right and explains nothing. The strong answer says: primitives are passed by value, references are passed by value, and because objects are only ever reached through references, what you can do through the reference survives the call.

The stronger candidate demonstrates it. "When I pass an object to a method and the method mutates it, the caller sees the mutation, because both hold copies of the same reference to one heap object. When I reassign the reference inside the method, the caller does not see it, because reassignment only changed the local copy." That sentence contains the whole model.

Expected follow-ups: what happens when you assign one object variable to another, and why can two strings with the same characters be different objects? Both reward the candidate who answers from the model, stack, heap, reference, not from a memorized rule.

## Knowledge Check

1. A method receives an `int` and a `List`, increments the `int`, and adds an element to the list. Back in the caller, the `int` is unchanged and the list has the new element. Explain both outcomes using stack, heap, and references.

2. Two threads each run the same method. The method creates a local `HashMap`, writes to it, and returns it to no one. Why is this safe? Then change one line so the map is assigned to a field, and explain why it stops being safe.

3. `String a = "hi"; String b = new String("hi");` Then `a == b` is false. Using the memory model, explain what `==` actually compared, and whether `a` and `b` could ever be the same reference.

## Key Takeaways

- Primitives live in the stack frame as values; objects live in the heap and are reached only through references.
- Assignment and method calls copy values, including references, so copying a reference shares one object.
- `==` compares references on objects, which is why equality needs `.equals`.
- Most concurrency and stale-data bugs are reference bugs: a shared object reached from two places.

## What's Next

The model says objects live in the heap and variables reach them through references, which is all well drawn, but none of it says where an object's shape comes from. That shape is the class. The next article, Classes, Objects, and the this Keyword, takes the first step from the model to code: what a class declares, what an object actually instantiates, and why the `this` reference is the handle an instance uses to find its own fields.

---

This article explains how object-oriented programming maps onto the Java memory model, where the stack holds local values and the heap holds every object, connected by references. Its strongest claim is that Java passes both primitives and references by value, so the only real bug in most Java systems is a shared object reached through a reference that escaped.
