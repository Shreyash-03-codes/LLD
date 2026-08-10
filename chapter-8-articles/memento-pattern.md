# Memento Pattern

## Learning Objectives

- Define the Memento as a sealed snapshot: state captured into an opaque token that only the originator can unpack.
- Separate the three roles, the originator that owns the state, the memento that holds the snapshot, and the caretaker that stores tokens without reading them.
- Argue the honest limits: the memory cost of full copies, the decision of when to snapshot, and the boundary between a memento and a full clone.

## Introduction

Memento is the pattern behind undo and snapshots. You have a document, an account balance, a game state, and you want to travel back to an earlier version. The pattern's answer is to capture the state into an opaque token, store tokens as plain data, and let the object that owns the state unpack a token when it is time to restore.

Three roles are in play:

```java
public interface Memento {
    // opaque to everyone except the originator
}
```

The originator owns the state and creates and applies mementos. The memento is the snapshot itself, sealed. The caretaker stores mementos and hands them back on request. The caretaker can never read the contents, because the contents are an opaque blob to anyone but the originator.

## Problem Statement

The naive undo is to keep the whole object around. Keep a list of full document objects and restore from them:

```java
public class Document {
    private String text;
    private String formatting;

    public void restore(Document earlier) {
        this.text = earlier.text;
        this.formatting = earlier.formatting;
    }
}
```

That works until it does not. Every saved version is a full live object with methods and behavior, so the history list can call anything on any past version, and the past versions can be mutated by accident when someone edits them as if they were current. A history that holds live objects is a history that can be reached into, and reaching into the past is how a bug corrupts an old snapshot and then "undo" restores a corrupted state.

The second failure is the boundary. The document's fields are what must be snapshotted, but fields are private. Either the history reaches through getters and setters, which leaks the whole internal shape to anyone who can hold a version, or the document exposes deep copies, which means the document's internals are public to the world. Undo should not be the reason an object's guts become public API.

## Core Concept

Memento resolves the boundary with a sealed token. The originator alone understands the token's contents:

```java
public class Document {
    private String text;
    private String formatting;

    public Memento createMemento() {
        return new DocumentMemento(text, formatting);
    }

    public void restore(Memento memento) {
        if (!(memento instanceof DocumentMemento m)) {
            throw new IllegalArgumentException("foreign memento");
        }
        this.text = m.text;
        this.formatting = m.formatting;
    }

    private record DocumentMemento(String text, String formatting) implements Memento {}
}
```

The caretaker knows nothing but the interface:

```java
public class History {
    private final Deque<Memento> undoStack = new ArrayDeque<>();

    public void push(Memento memento) {
        undoStack.push(memento);
    }

    public Memento pop() {
        return undoStack.pop();
    }
}
```

The memento is a plain record, just two strings and no behavior. The `History` stores it and hands it back without ever calling a method on it. The `Document` is the only class that can cast `DocumentMemento` and read the fields, because the record is private to it. The snapshot travels through the caretaker as an opaque token, and the internal shape of the document leaks nowhere.

The undo flow reads cleanly:

```java
history.push(document.createMemento());
document.typeMore();
// later
document.restore(history.pop());
```

### Who may touch what

The pattern is three rules:

The originator creates a memento and applies a memento. The memento holds the state and has no behavior beyond storage. The caretaker stores and returns mementos and never inspects them.

If the caretaker needs to label snapshots, with a timestamp or an action name, the label lives beside the token in a wrapper, not inside the token. The token stays opaque; the metadata is external. This is what makes the caretaker safe to write: it stores `Memento` and a name, and it never needs to know whether the token hides a document, a game board, or a stream of bytes.

### The copy question

A memento is a snapshot, and a snapshot is a copy. The pattern does not avoid the copying cost; it avoids the coupling. `createMemento()` copies exactly the fields that matter into an immutable record. A big object means a big record, and the history can grow without limit if nothing bounds it. The honest engineering question is how much to copy and when.

Two answers are common. Copy the whole state when the state is small and the snapshots are few, and bound the history, a cap of fifty undo steps is a normal rule. Or copy a reference to an immutable state object when the object already uses one, so the snapshot is a single pointer. The first is the simplest and the memory is predictable. The second is cheap and is why many editors store versions as immutable snapshots of a document.

The trade is visibility and not just cost. A whole-state copy is easy to reason about and easy to debug, and its cost is simply the size of the snapshot, which is why debounced snapshots are the default. An immutable-reference copy is nearly free and supports a deep history, but it demands that the state already be immutable, and an immutable state of the whole record is a design decision on its own. Choose the whole copy when the state is small or snapshots are rare; choose the pointer when the state is large, changes rarely, and the user expects many undo steps.

### The siblings

Memento is often compared to two other shapes.

Snapshotting the whole state is what a deep clone does. The difference is who holds the copy. A clone is a live duplicate with the same public surface, so the caller can mutate the clone as if it were the original. A memento is sealed, so the caller can only hold it and give it back. The seal is the point.

The Command pattern and Memento combine well. An undo-capable command can capture a memento of the receiver before acting, push it on the undo stack, and let the command's undo apply it. One stack of mementos, each sealed by its own originator, and no command reaches into the receiver's fields.

## Real Production Usage

Editors are the honest home of the pattern. A document that keeps an undo history stores snapshots of its state, the caretaker is the history, and restoring rewinds to an earlier snapshot. Text editors, image editors, spreadsheet undo stacks, all practice this shape.

Backends use it with a name change. The "memento" becomes an event or a versioned record, and the "caretaker" becomes the event store. Instead of overwriting the current state, the system appends snapshots, and to restore an earlier state it replays the stored events or applies a stored snapshot. Event sourcing is Memento at database scale: the token is the event, the originator is the aggregate, and the store never inspects the meaning of what it keeps.

Two production cautions are worth repeating. Bound the history, because an unbounded undo stack is a memory leak with a nice name. And snapshot at meaningful boundaries, an autosave every keystroke on a large document is expensive, a debounced snapshot every few seconds is sane.

## Common Mistakes

**Making the caretaker understand the token.** A history that calls methods on the memento, reads a field to label it, or casts it to a concrete type has broken the seal. The caretaker stores and returns; the originator interprets. If a label is needed, wrap the token with metadata outside it.

**Snapshotting live references.** A memento that holds references to mutable objects is not a snapshot, it is a photo of a mirror. The record must hold copies, immutable values, or pointers to immutable state. Otherwise undoing later reads whatever the current object mutated meanwhile.

**Restoring into the wrong originator.** `Document.restore` casts the token and should verify the concrete type before unpacking. A memento created by one originator and applied to a different one is a type error at the worst moment, and the `instanceof` guard turns it into a loud `IllegalArgumentException` at the earliest one.

## Interview Perspective

Interviewers use Memento to test whether you understand encapsulation, not snapshots. A weak answer says "it saves state and restores it." A strong answer says the memento is sealed, the caretaker cannot read it, the originator is the only class that can unpack it, and that seal is what makes the pattern different from a clone.

The follow-up that sorts candidates is "why not just deep-copy the object?" The strong answer: a deep clone is a live object with a public surface, so callers can mutate it; a memento is an opaque token, so the caretaker can store it forever without ever coupling to the object's internals. The second follow-up is about cost: full copies are the price, so you bound the history and decide when to snapshot.

Common follow-ups:

- "Memento versus a deep clone: which one can the caller mutate?"
- "The history is growing without limit. What did the pattern not solve for you?"

## Knowledge Check

1. In the document example, which class can cast `DocumentMemento`, and which class can only store and return it?
2. The history needs to show an action name next to each undo step. Where does the name live, and why not inside the token?
3. A memento holds a reference to a mutable buffer. What will happen when the document restores from it after the buffer changed, and how is it fixed?

## Key Takeaways

- Memento is a sealed snapshot: an opaque token only the originator can unpack.
- The caretaker stores and returns tokens and never inspects them, so it is trivially safe to write.
- The difference from a clone is the seal: a clone is mutable, a memento is not.
- The cost is real copies, so the history must be bounded and snapshots placed at meaningful boundaries.
- The pattern scales in production as event sourcing, where the event store is the caretaker.

## What's Next

The next article is Visitor. Memento sealed an object's state behind an opaque token; Visitor changes who holds the behavior for a family of objects. It puts an operation into its own visitor object and dispatches by element type, so you can add new operations over a stable class hierarchy without editing the classes. We will cover double dispatch, the cost of the pattern, and why it fits compilers and ASTs better than everyday domain objects.

---

This article explains the Memento pattern as a sealed snapshot, with an undoable document as the example. It shows how the opaque token keeps the caretaker from ever coupling to the originator's internals.