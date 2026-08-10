# Prototype Pattern

## Learning Objectives

- Explain why Prototype exists when `new` is cheap: it buys you fast copies of objects whose construction is expensive, and it buys you reuse of a known-good shape.
- Implement `clone()` correctly in Java, and know precisely where `Object.clone()` is broken and why so many implementations ship the bug anyway.
- Draw the shallow versus deep copy distinction on a concrete object graph and say which one a given implementation actually produces.

## Introduction

Prototype creates new objects by copying an existing one, the prototype, instead of constructing from scratch. Every other creational pattern asks "how do I build this thing." Prototype asks "how do I avoid rebuilding something I already have."

In Java the pattern looks almost trivial, because `clone()` is right there on `Object`. The triviality is a trap. `Object.clone()` is the sharpest-edged API in the JDK, and the pattern's real content is in the details that everyone skips: what a copy shares with its source, what `Cloneable` actually does, and whether your "copy" is a copy at all.

## Problem Statement

Construction is not always cheap. Imagine a report template loaded at startup: parse a config file, fetch default styling, render a preview, validate against schema. Each of those steps happens once, then the template object is reused hundreds of times. Now a user wants a copy of that template to edit. What is the naive implementation?

```java
ReportTemplate copy = new ReportTemplate();
copy.setHeader(original.getHeader());
copy.setFooter(original.getFooter());
copy.setSections(new ArrayList<>(original.getSections()));
```

Two problems. First, the copy depends on the source's getters exposing every field, and it re-runs nothing but also knows nothing: the constructor `new ReportTemplate()` re-runs the expensive setup, or worse, it does not, and your copy is missing state the real template loaded. Second, the moment `ReportTemplate` gains a field, this copy code silently stops copying it. Nobody recompiles a field, the copy just quietly diverges.

The deeper failure is the situation Prototype was invented for. The expensive part is not the object itself, it is the *assembly*. A document template that took a minute of network calls to build should not be rebuilt for a tweak. You want "same shape as that one, but a fresh identity," and copying is the only construction strategy that honors that.

## Core Concept

Prototype makes the object responsible for copying itself. The class implements `clone()`, and callers copy instead of construct:

```java
public class ReportTemplate implements Cloneable {
    private String header;
    private String footer;
    private List<String> sections;

    public ReportTemplate() {
        // expensive setup: load config, fetch styles, validate
    }

    @Override
    public ReportTemplate clone() {
        try {
            return (ReportTemplate) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new AssertionError("unreachable: ReportTemplate is Cloneable", e);
        }
    }
}
```

A caller then writes:

```java
ReportTemplate draft = published.clone();
```

That is the whole pattern at its core. `clone()` centralizes the copy logic in the class that knows its own state, so adding a field updates the copy in one place, and the copy never re-runs the expensive constructor. This is the "prototype" piece: the source object is the template, and `clone()` stamps out variations.

The hard part is `super.clone()`. What does it actually give you? `Object.clone()` performs a *shallow* copy: every field is copied by value, which for reference fields means the clone's references point at the same objects the original's references point at. For `sections`, a `List`, the clone holds a reference to the *same list*. Modify the clone's list and you modify the original. That is the classic bug, and it is why `Object.clone()` is broken-by-default for any object that owns mutable state.

### Shallow versus deep

The fix is to deepen the copy at exactly the points where sharing is dangerous:

```java
@Override
public ReportTemplate clone() {
    try {
        ReportTemplate copy = (ReportTemplate) super.clone();
        copy.sections = new ArrayList<>(this.sections);
        return copy;
    } catch (CloneNotSupportedException e) {
        throw new AssertionError("unreachable", e);
    }
}
```

Now the clone has its own list. The decision rule is simple: share immutable fields, copy mutable fields. Whether "mutable field" means the field's object or the whole graph beneath it is a judgement call per field, and that judgement is the entire design of the pattern in practice.

Diagram: shallow versus deep copy on a two-object graph

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 860 490" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#444"/>
    </marker>
  </defs>

  <rect x="40" y="30" width="230" height="90" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="155" y="52" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">Original</text>
  <text x="155" y="74" text-anchor="middle" font-size="12" fill="#1a2733">id = 1</text>
  <text x="155" y="96" text-anchor="middle" font-size="12" fill="#1a2733">data = "a"</text>

  <rect x="40" y="200" width="230" height="90" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5" stroke-dasharray="6 4"/>
  <text x="155" y="222" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">Shallow clone</text>
  <text x="155" y="244" text-anchor="middle" font-size="12" fill="#1a2733">id = 2</text>
  <text x="155" y="266" text-anchor="middle" font-size="12" fill="#1a2733">data = "a"</text>

  <rect x="40" y="370" width="230" height="90" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5" stroke-dasharray="6 4"/>
  <text x="155" y="392" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">Deep clone</text>
  <text x="155" y="414" text-anchor="middle" font-size="12" fill="#1a2733">id = 3</text>
  <text x="155" y="436" text-anchor="middle" font-size="12" fill="#1a2733">data = "a"</text>

  <rect x="590" y="90" width="240" height="80" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="710" y="112" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">Inner (shared)</text>
  <text x="710" y="134" text-anchor="middle" font-size="12" fill="#1a2733">value = "a"</text>

  <rect x="590" y="350" width="240" height="80" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="710" y="372" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">Inner (copy)</text>
  <text x="710" y="394" text-anchor="middle" font-size="12" fill="#1a2733">value = "a"</text>

  <line x1="270" y1="75" x2="588" y2="145" stroke="#6b7a89" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="270" y1="245" x2="588" y2="115" stroke="#6b7a89" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="270" y1="415" x2="588" y2="385" stroke="#6b7a89" stroke-width="1.5" marker-end="url(#arrow)"/>
</svg>
```

Both the original and the shallow clone point at the same `Inner`. The deep clone points at its own copy. The whole pattern, and most of its bugs, live in that one distinction.

### Prototype registries

The pattern is often paired with a registry, a map of named prototypes that hands out copies instead of building:

```java
public class TemplateRegistry {
    private final Map<String, ReportTemplate> templates = new HashMap<>();

    public void register(String name, ReportTemplate template) {
        templates.put(name, template);
    }

    public ReportTemplate create(String name) {
        return templates.get(name).clone();
    }
}
```

`TemplateRegistry.create("invoice")` returns a fresh copy of the invoice template every call. This is where Prototype becomes more than a clever `clone()`. The registry gives you a catalog of known-good starting points, each copied cheaply, none of which have to be re-constructed. Game engines run on this exact shape: a catalog of entity templates, and spawning is a copy.

There is one more honest thing to say. In modern Java, `clone()` is so maligned that a plain copy constructor is often the better tool, and Guava's `copyOf` factories and records (`record` types with their built-in copy semantics) cover most copy needs with far less danger. Prototype is the right call when the copy must happen through an interface without naming the concrete class, or when the object graph is deep enough that a hand-written copy constructor is untenable. It is the rarest of the five patterns for a reason.

## Real Production Usage

The JDK uses cloning where it is safe and convenient, which is mostly in collections. `ArrayList.clone()` and `HashMap.clone()` exist and are used by the framework internally, but they document their limits: the array is copied, the elements are not. That is the pattern's DNA in production: copy the container, share the contents.

The pattern as a whole shows up where construction cost or shape-reuse dominates. Configuration templates in CI and build systems, document and report templates, and entity templates in games all follow it. Libraries rarely expose `clone()` through their public API anymore because the contract is too easy to break; they prefer copy constructors and `copyOf` factories. That is a strong signal about how fragile the pattern is in the real world, and worth internalizing.

## Common Mistakes

**Using `super.clone()` and shipping a shallow copy of a mutable graph.** The clone shares internal collections with the original, so every mutation of the copy silently corrupts the source. This is the single most common Prototype bug, and it is invisible until two objects start fighting over the same list.

**Throwing `CloneNotSupportedException` up instead of handling it.** `super.clone()` declares the checked exception, but if your class implements `Cloneable` it can never throw it. Propagating it forces every caller to handle an exception that is structurally impossible, which is why the `AssertionError` pattern exists. Do not make callers catch impossible exceptions.

**Prototyping immutable objects.** If a class has no mutable state, copying it is pointless and `clone()` is a lie that only costs you a diagram. Copy constructors or shared references serve better. Prototype earns its keep on mutable state, expensive construction, or both.

## Interview Perspective

Prototype is where interviewers test whether you have read the JDK rather than the book. A weak answer defines the pattern and draws the diagram. A strong answer can name what `Object.clone()` actually does, why the shallow default is dangerous, and why the checked exception is a design mistake in the JDK that every implementation has to work around.

The follow-up that sorts people is usually the shallow versus deep question, or "why is this pattern rare in modern Java?" The answer that lands names the competition: copy constructors, `copyOf` factories, and records all do copies with better guarantees than `clone()` ever did.

Common follow-ups:

- "Your class holds a `List<Line>`. Write the `clone()` and say what is shared and what is copied."
- "When would you still choose Prototype over a copy constructor?"

## Knowledge Check

1. `super.clone()` returns a shallow copy. Given a `ReportTemplate` with a `List<String> sections`, trace what two objects share after the shallow clone, and what breaks when one appends to its list.
2. Write a `deepClone()` for a class that holds a `Map<String, User>` where `User` itself is mutable, and say how deep you actually have to go.
3. Why is the `CloneNotSupportedException` catch an `AssertionError` rather than a normal return path, and what does that say about the `Cloneable` marker's design?

## Key Takeaways

- Prototype copies an existing object instead of constructing one, which pays off when construction is expensive or when you want a catalog of known-good shapes.
- `Object.clone()` is shallow by default, and for any mutable reference field that is a bug, not a detail.
- Deepening the copy is a per-field decision: share immutable, copy mutable, and copy the graph beneath when ownership matters.
- Prototype registries turn the pattern into a catalog of cheap starting points, which is how games and template systems use it.
- Modern Java mostly prefers copy constructors and `copyOf` factories over `clone()`, which is the honest reason the pattern is rare.

## What's Next

The next article is the comparison chapter: Factory versus Builder. We have now met all five patterns, and the two that get confused most often deserve a direct head-to-head. The article will pin down the real axis, which class changes versus how one class is assembled, and give you a decision procedure you can actually use under interview pressure instead of a memorized difference.

---

This article explains the Prototype pattern as copying a known object instead of constructing a new one, centered on the distinction between shallow and deep copies. Its central claim is that `Object.clone()` is shallow by default and therefore dangerous for any mutable field, and that the decision of what to share and what to copy is the entire real content of the pattern.
