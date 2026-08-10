# Flyweight Pattern

## Learning Objectives

- Split object state into intrinsic, shareable state and extrinsic, per-context state, and say which half belongs in the shared object.
- Build a flyweight factory that hands out shared instances keyed by their intrinsic state.
- Argue when the pattern is worth it and when it is a premature optimization, because Flyweight is the most misused of the seven.

## Introduction

Flyweight shares as much state as possible among many similar objects. When a text document has fifty thousand character instances that differ only in position, most of each object is identical: the letter, the font, the size. Flyweight keeps one shared object per unique letter-font pair and stores only the position outside the object.

The name comes from boxing: a lightweight flyweight is the boxer who packs less weight. The pattern packs less state per object by moving the repetitive part into a shared pool.

## Problem Statement

The failure is memory, and it is a specific kind of memory. A text editor renders a page. The naive model creates a `Glyph` object for every character on the page:

```java
public class Glyph {
    private final char symbol;
    private final Font font;
    private final int x;
    private final int y;

    public Glyph(char symbol, Font font, int x, int y) {
        this.symbol = symbol;
        this.font = font;
        this.x = x;
        this.y = y;
    }
}
```

A page with fifty thousand characters creates fifty thousand objects, each carrying its own `symbol` and `Font` reference. But a page of text has maybe forty unique symbol-font combinations. The other forty-nine thousand nine hundred sixty objects are duplicates of those forty, differing only in `x` and `y`. The editor is paying fifty thousand allocations to store about forty pieces of real information, plus fifty thousand copies of the position that is the only genuinely per-character data.

Scale the picture and it gets worse. An animation engine with ten thousand visible trees, a chat app rendering the same emoji tens of thousands of times, a map with fifty thousand pins. In every case the failure is identical: most of each object is repeated state, and the repetition is what burns the heap.

## Core Concept

Flyweight splits state in two. The intrinsic state is the part that is shared and immutable, the character and its font. The extrinsic state is the part that varies per use, the position. The shared object holds the intrinsic state and receives the extrinsic state as method arguments at use time.

```java
public class Glyph {
    private final char symbol;
    private final Font font;

    Glyph(char symbol, Font font) {
        this.symbol = symbol;
        this.font = font;
    }

    void draw(int x, int y) {
        // render symbol in font at position x, y
    }
}
```

The factory owns the pool and guarantees sharing:

```java
public class GlyphFactory {
    private final Map<String, Glyph> pool = new HashMap<>();

    public Glyph get(char symbol, Font font) {
        String key = symbol + ":" + font;
        return pool.computeIfAbsent(key, k -> new Glyph(symbol, font));
    }
}
```

Callers ask the factory, never the constructor, and the sharing is the point:

```java
Font arial = new Font("Arial", 12);

Glyph g1 = factory.get('a', arial);
Glyph g2 = factory.get('a', arial);

// g1 and g2 are the same object
g1.draw(10, 20);
g2.draw(30, 40);
```

Fifty thousand characters now cost a few dozen shared `Glyph` objects and fifty thousand small position records, which can live as primitives in an array or in small per-line context objects. The intrinsic state is created once and reused; the extrinsic state is passed in at render time and never stored on the shared object.

Diagram: shared flyweights referenced by many clients

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1010 400" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#444"/>
    </marker>
  </defs>

  <rect x="40" y="40" width="200" height="70" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="140" y="62" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">Glyph context 1</text>
  <text x="140" y="88" text-anchor="middle" font-size="12" fill="#1a2733">pos = (10, 20)</text>

  <rect x="40" y="150" width="200" height="70" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="140" y="172" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">Glyph context 2</text>
  <text x="140" y="198" text-anchor="middle" font-size="12" fill="#1a2733">pos = (30, 40)</text>

  <rect x="40" y="260" width="200" height="70" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="140" y="282" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">Glyph context 3</text>
  <text x="140" y="308" text-anchor="middle" font-size="12" fill="#1a2733">pos = (5, 90)</text>

  <rect x="650" y="70" width="330" height="290" fill="none" stroke="#6b7a89" stroke-width="1.5" stroke-dasharray="8 4"/>
  <text x="800" y="92" text-anchor="middle" font-size="12" font-style="italic" fill="#5a6b7a">Flyweight pool</text>

  <rect x="680" y="100" width="240" height="80" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="800" y="122" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">Glyph('a', Arial)</text>
  <text x="800" y="148" text-anchor="middle" font-size="12" fill="#1a2733">intrinsic state</text>

  <rect x="680" y="260" width="240" height="80" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="800" y="282" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">Glyph('b', Arial)</text>
  <text x="800" y="308" text-anchor="middle" font-size="12" fill="#1a2733">intrinsic state</text>

  <line x1="240" y1="75" x2="678" y2="155" stroke="#6b7a89" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="240" y1="185" x2="678" y2="125" stroke="#6b7a89" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="240" y1="295" x2="678" y2="300" stroke="#6b7a89" stroke-width="1.5" marker-end="url(#arrow)"/>
</svg>
```

The first two contexts point at the same `Glyph('a', Arial)`, sharing its intrinsic state, while each keeps its own position. The third context points at a different glyph. Sharing is visible in the diagram as two arrows converging on one object.

### The rules that keep it honest

The split has two hard rules, and breaking either one turns the pattern into a shared-mutable-state bug.

The intrinsic state must be immutable. The moment any client mutates a shared `Glyph`, it mutates it for every client holding it. That is why the flyweight's constructor sets the state and the object never changes. If you catch yourself adding a setter to a flyweight, you have destroyed the sharing guarantee and should stop.

The extrinsic state must never be stored on the shared object. It travels with the caller and is passed in at use time. The moment you store position inside the `Glyph`, the shared object can no longer be shared, because two characters cannot occupy the same position, and you have quietly undone the pattern.

## Real Production Usage

The honest truth is that Flyweight is rare in modern Java application code, and the places it survives are mostly inside the JDK itself. `Integer.valueOf(int)` shares cached instances for the range -128 to 127, so two calls returning the same value return the same object, which is why `Integer.valueOf(5) == Integer.valueOf(5)` is true and people get burned comparing larger values. The string constant pool interns strings so repeated literals share one backing instance. `java.awt`'s font and color objects get pooled too. In each case the shape is the same: an immutable shared object handed out by a factory, with the varying part kept outside.

Application-level flyweights appear where object count is the measured problem and sharing is safe: document editors, game engines pooling textures and sprites, UI toolkits pooling icons. If you are not in that regime, with millions of instances and a bounded set of intrinsic combinations, Flyweight is almost certainly a premature optimization. Measure the heap first. The pattern saves memory that you might not be spending.

The pool has a lifecycle decision nobody flags on the diagram: does it grow without bound, or does it evict? A glyph factory keyed by string accumulates a new glyph for every unique key, which is the correct behavior for a bounded glyph set and a slow leak for anything unbounded. The escape hatches are the ones the JDK already uses. `Integer.valueOf` keeps its cached range fixed and lets values beyond it allocate normally, and the string pool hands the lifecycle to the runtime's GC. If your keys are unbounded, either bound the key space or use a weak-keyed map so the pool shrinks as the objects that reference it become unreachable. A flyweight pool that never forgets is a memory leak wearing a memory-saver costume.

## Common Mistakes

**Storing extrinsic state inside the flyweight.** Position, owner, or any per-use data gets added to the shared object, and sharing collapses because the shared object is now different for every use. The intrinsic-extrinsic line is the whole pattern; crossing it silently undoes the memory win.

**Making the shared state mutable.** A setter on a flyweight means every client sees the change. Shared mutable state is the one bug in this chapter that corrupts other users' data, and it is not caught by a test until two clients collide.

**Applying the pattern before measuring.** The pattern is for the regime of millions of instances and a bounded set of shared combinations. Applying it to a few thousand objects saves little and adds a factory, a pool, and an immutability contract. Measure, then decide.

## Interview Perspective

Flyweight is a knowledge filter. A weak answer describes sharing and stops. A strong answer names the intrinsic-extrinsic split, states the immutability rule, and can say when the pattern is a waste, which is the honest note most candidates never reach.

The follow-ups push the two rules. "What happens if two clients share a glyph and one changes its font?" The strong answer explains that the question is unaskable under the pattern, because the intrinsic state is immutable by contract, and explains why that contract is what makes sharing safe.

Common follow-ups:

- "Your flyweight has a setter. What has broken, and who is affected?"
- "Where does the position live in a text renderer, and why can it not live on the shared glyph?"

## Knowledge Check

1. A text renderer stores `x` and `y` on the `Glyph` object shared by the factory. Trace what happens to the second character on the same line and explain which rule was broken.
2. `Integer.valueOf(5) == Integer.valueOf(5)` returns true but the same comparison fails for 200. Explain both results in terms of the flyweight pool.
3. A game has 100,000 bullets that differ in position and velocity. Which state is intrinsic, which is extrinsic, and is the pattern even the right tool here?

## Key Takeaways

- Flyweight shares intrinsic state and keeps extrinsic state per use, and the split is the entire pattern.
- The intrinsic state must be immutable and the extrinsic state must never live on the shared object.
- The factory owns the pool, keyed by intrinsic state, and is the only door to the flyweights.
- The JDK's integer cache and string pool are the pattern in production, and application use is genuinely rare.
- Measure before applying it, because for most codebases Flyweight is a premature optimization.

## What's Next

The next article steps back from the seven to settle the comparisons the overview promised. Adapter, Facade, and Proxy all wrap, and this chapter has named their intents in passing. The article puts them side by side with a real decision procedure, so you can look at any wrapper in any codebase and say, with confidence, which of the three it is and whether it should exist.

---

This article explains the Flyweight pattern as sharing immutable intrinsic state across many instances while keeping per-use state outside, using a text renderer's glyphs as the running example. It argues that the intrinsic-extrinsic split and the immutability rule are the entire pattern, and that for most codebases the pattern is a premature optimization until measured.
