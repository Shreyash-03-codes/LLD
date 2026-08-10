# Iterator Pattern

## Learning Objectives

- Define the Iterator as a cursor that hides a collection's internal layout behind a uniform walk.
- Implement an iterator over a linked list and expose it through `Iterable`, so the caller never touches the collection's storage.
- Explain the cursor contract and the fail-fast modification contract, and state the pattern's honest limits.

## Introduction

Iterator is the oldest and least glamorous pattern in this chapter, and the one you use constantly. Every Java `for`-each loop over a collection runs a hidden `Iterator` under the hood. When you write `for (String s : names)`, the compiler produces an iterator and calls `hasNext()` and `next()` in a loop behind your back.

The entire pattern is two interfaces:

```java
public interface Iterator<E> {
    boolean hasNext();
    E next();
}

public interface Iterable<E> {
    Iterator<E> iterator();
}
```

The `Iterable` is what a `for`-each loop can walk. The `Iterator` is the cursor that does the walking. The point is that the same loop works identically whether the collection is an `ArrayList`, a `LinkedList`, a `HashSet`, or a `TreeMap`. The loop never asks how the data is stored, and the iterator never reveals the layout.

## Problem Statement

The direct approach is to let a caller reach into the collection and walk it. Expose the head and let the caller follow pointers:

```java
public class LinkedList {

    public Node head() {
        return first;
    }

    public Node nextOf(Node current) {
        return current.next();
    }
}
```

That works, and it is a trap. Every consumer that walks the list has to know about `Node`. Every consumer re-implements the same while-loop against the node structure. A consumer that also needs to walk a `HashSet` writes a completely different traversal, because the set has no `next()` method at all. Now the traversal logic is duplicated across every consumer, and every consumer is coupled to the collection's internal storage. If the storage changes, every walk breaks.

The second cost is position. The cursor, where the walk currently is, lives inside the caller. If the same caller also needs a second concurrent walk, it cannot have one without threading a second cursor through the same code, and two such callers expose the same cursor in two methods that are easy to confuse. Move two cursors around the same loop body and you have to remember which one you are holding at each step.

## Core Concept

Iterator moves the walk into its own object and owns the position, so the position belongs to the iterator and never hides in the caller. The collection implements `Iterable<T>`, returns a fresh `Iterator<T>` for each call, and the cursor lives inside that iterator so two loops can never interfere:

```java
public class LinkedList implements Iterable<Object> {

    private static final class Node {
        private final Object value;
        private Node next;
    }

    private Node first;

    @Override
    public Iterator<Object> iterator() {
        return new Iterator<>() {
            private Node cursor = first;

            @Override
            public boolean hasNext() {
                return cursor != null;
            }

            @Override
            public Object next() {
                Object value = cursor.value;
                cursor = cursor.next;
                return value;
            }
        };
    }
}
```

The caller writes one loop and never touches `Node`:

```java
for (Object item : list) {
    process(item);
}
```

The cursor field is now private to the iterator instance. One caller can start a walk, another can start a second, and the two cursors never collide, because each `iterator()` call returns a fresh iterator with its own position.

The collection side and the consumer side agree on one contract, and nothing else. The collection answers `iterator()` and nothing more. The consumer walks `hasNext()` and `next()` and nothing else. A `for-each` loop over `names`, a `stream()` over the same collection, a manual `while (it.hasNext())`, all read the same two methods and none of them cares whether the backing store is a contiguous array or a linked chain of nodes. That is the pattern: the walk is moved into an object only that object understands, and the layout becomes a private detail nobody needs.

The same idea scales past a single linear structure. A tree, a skip list, a paginated result set, each provides an `Iterable`, and each iterator presents itself as the same two methods, `hasNext()` and `next()`. The complexity of the traversal, how to descend, when to back out, where to stop, lives inside the iterator and stays out of the loop. A `for`-each over a lazy stream looks no different from a `for`-each over a million-row list.

### The cursor contract

The two methods agree on a strict contract.

`hasNext()` answers whether another element is coming and must not consume it. Calling `hasNext()` twice without a `next()` between must return the same answer and walk nothing. `next()` returns the current element and advances the cursor. When the collection is exhausted, `hasNext()` returns `false`, and calling `next()` anyway must throw `NoSuchElementException` and leave the cursor unchanged.

Getting the order wrong produces the classic off-by-one. If `next()` advances before returning, a walk skips the first element. If `hasNext()` consumes, a walk skips every other one. The contract is why the enhanced-for loop is safe: the compiler calls `hasNext()` first, and only enters the body when something is present.

### Fail-fast

Java collections protect the walk with a contract called fail-fast. The collection keeps a modification counter, and each iterator captures that counter when it is created. On every `next()`, the iterator re-reads the counter. If the collection changed, added, deleted, replaced, mid-walk, the iterator throws:

```java
Exception in thread "main" java.util.ConcurrentModificationException
	at java.base/java.util.ArrayList$Itr.checkType(...)
```

The point is to fail loudly instead of silently walking a collection whose backing array is shifting under the cursor. A silent walk can re-read or skip values, and the bug is subtle. A loud `ConcurrentModificationException` surfaces the mistake the moment it happens. The habits that avoid it, when you want to remove during a walk, are a two-pass approach, a `removeIf` call, or collecting the targets first and deleting them after the walk completes.

## Real Production Usage

The obvious production tool is the enhanced for loop, which is bytecode for iterator + `hasNext()` + `next()`. Backends use iterators constantly: the `try`-with-resources that reads a file line by line streams rather than loading the whole file into memory, the cursor over a `ResultSet` from a database, the `forEach` iteration across a `TreeMap` when a plain `HashMap` has no ordering at all. The traversal contract is what lets all of these share one mental model: call `hasNext()`, get the next element, move on, stop at the end.

Iterators are also the engine of the functional-style code. A `stream()` pipeline calls `spliterator()` and walks a separated iterator, and the `filter`, `map`, and `collect` steps operate on the elements as they flow. The ordering contract comes from the underlying structure, but the walk itself is always an iterator. When you see a pipeline that reads cleanly, an iterator is doing most of the work quietly underneath.

The honest takeaway is that production code rarely hand-writes a custom `Iterator`. The collections and streams libraries already provide them. You hand-write one when the library has no walk for your structure: a custom container, a sparse internal layout, a lazy or paginated source. Handwriting an iterator pays when you define a new container or a traversal the JDK does not give you.

One honest trade-off is worth naming. An `ArrayList` iterator walks by index, so a hot loop via `for`-each is cheap and predictable, but an iterator over a `LinkedList` follows a pointer per step, and a manual index loop over that same list would be worse. The iterator hides that difference, and you usually want the hiding. When the walker is the hot path and you know the structure, an explicit index over an array-backed list is faster than the iterator, but the win is small and conceded to a clearer, structure-agnostic loop. Choose the iterator and read the profile before breaking it.

## Common Mistakes

Iterator is a small pattern, and its mistakes are small and sharp. The three that show up most across a codebase are breaking the cursor, mutating during the walk, and dropping the generic type.

**Breaking the cursor contract.** New manual iterators advance in `next()` before returning and skip an element, or give `hasNext()` a consuming step so the loop alternates. The rule is to keep `hasNext()` pure, read-only, and to let `next()` be the only place the cursor moves.

**Mutating during the walk.** Structural changes, an `add` or `remove`, mid-walk against a mod-count-based collection. Tried-and-true is to not build the walk in the middle of the loop; collect the targets first and mutate the collection after the walk.

**Raw iterator type.** A raw `Iterator` returns `Object`, so a wrong type surfaces at runtime where it casts and crashes. Prefer the generic `Iterator<E>` so the compiler tells you the type mismatch at compile time.

## Interview Perspective

Interviewers use Iterator to check whether you reason about the cursor, not just the API. A weak answer quotes `hasNext()` and `next()`. A strong answer explains where the position lives, inside the iterator, one per consumer, why two concurrent walks do not collide, and why the raw iterator errors at runtime while the typed one errors at compile time.

Follow-up lines that sort candidates:

- "Where does the cursor live, and why does a second loop never collide with the first?"
- "How do you delete while walking, and what happens if you do not?"

## Knowledge Check

1. In the linked-list example above, which type answers `hasNext()` and which returns a fresh cursor per call?
2. You call `hasNext()` twice and `next()` once and then the collection is finished. What does the third `hasNext()` answer, and what does the next call do?
3. You delete from an `ArrayList` inside a `for`-each. What does Java raise, and what is the remedy?

## Key Takeaways

- Iterator moves the walk and the cursor out of the collection, so the caller never touches the storage.
- The cursor contract says `hasNext()` must not consume, and `next()` must throw when exhausted.
- Fail-fast is a contract that surfaces silent breakage as an immediate `ConcurrentModificationException`.
- The generic typed iterator pushes a wrong-type error to compile time, where a raw iterator lets it surface at runtime.
- The enhanced for loop is the compiler writing this pattern for you.

## What's Next

The next article is Memento. The shift is in what is being hidden: we hid how a collection is walked; Memento hides how an object's own state is captured and restored. It is the shape behind undo and snapshots, a sealed token only the originator can unpack. We will see the snapshot boundary and the memory cost.

---

This article explains the Iterator pattern as the cursor that moves the walk out of the collection, letting one loop serve any storage layout. It shows the cursor and fail-fast contracts, and how the typed generic form catches wrong-type errors at compile time.