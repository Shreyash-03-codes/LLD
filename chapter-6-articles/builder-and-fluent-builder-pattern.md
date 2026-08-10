# Builder and Fluent Builder Pattern

## Learning Objectives

- Diagnose the telescoping constructor problem and explain why optional fields, not object size, are what make direct construction fail.
- Implement a Builder with a private constructor and an immutable product, and enforce invariants at `build()` time rather than mid-construction.
- Judge when the fluent style is a genuine improvement and when it is a method-chain decoration that made the code worse.

## Introduction

Builder separates the construction of a complex object from its representation, so the same construction process can create different representations. The phrase is more ceremony than the problem needs. The real problem Builder solves is narrow and concrete: constructors with too many parameters, especially optional ones, are unreadable, error-prone, and impossible to validate cleanly.

The pattern is a companion to immutability. If your objects are built once and never change, every field has to be supplied before the object exists, and the set of valid combinations can be large. Builder gives you a mutable staging area (the builder itself) that produces an immutable result.

## Problem Statement

Watch the constructors grow. A `Computer` starts reasonable and then everything optional arrives:

```java
public class Computer {
    private final String cpu;
    private final int ram;
    private final int storage;
    private final String graphicsCard;

    public Computer(String cpu) {
        this(cpu, 8, 256, null);
    }

    public Computer(String cpu, int ram) {
        this(cpu, ram, 256, null);
    }

    public Computer(String cpu, int ram, int storage) {
        this(cpu, ram, storage, null);
    }

    public Computer(String cpu, int ram, int storage, String graphicsCard) {
        this.cpu = cpu;
        this.ram = ram;
        this.storage = storage;
        this.graphicsCard = graphicsCard;
    }
}
```

This is the telescoping constructor anti-pattern, and it fails the moment the field count passes about four. Every call site reads as an unlabeled pile of arguments:

```java
Computer c = new Computer("Intel i7", 32, 1024, "RTX 4080");
```

Which argument is the storage? The programmer who wrote the call knows, for about three weeks. Then a new field arrives, say a `wifiCard`, and now there are five constructors, and the middle ones need default wiring, and someone adds a `boolean hasBluetooth` and the 2^n explosion of overloads starts feeling real. The deeper problem is that validity checks have nowhere to live. Is `storage == 0` invalid? Is a null graphics card allowed for a laptop but not a workstation? The constructor cannot answer, because a constructor signature cannot express policy.

## Core Concept

The Builder pattern replaces the pile of overloads with a mutable staging object that holds defaults, collects values through named methods, and produces the final object in one `build()` call. The product keeps a private constructor, so the only way in is through the builder.

```java
public class Computer {
    private final String cpu;
    private final int ram;
    private final int storage;
    private final String graphicsCard;

    private Computer(Builder builder) {
        this.cpu = builder.cpu;
        this.ram = builder.ram;
        this.storage = builder.storage;
        this.graphicsCard = builder.graphicsCard;
    }

    public static class Builder {
        private final String cpu;
        private int ram = 8;
        private int storage = 256;
        private String graphicsCard;

        public Builder(String cpu) {
            this.cpu = cpu;
        }

        public Builder ram(int ram) {
            this.ram = ram;
            return this;
        }

        public Builder storage(int storage) {
            this.storage = storage;
            return this;
        }

        public Builder graphicsCard(String graphicsCard) {
            this.graphicsCard = graphicsCard;
            return this;
        }

        public Computer build() {
            if (storage <= 0) {
                throw new IllegalArgumentException("storage must be positive");
            }
            return new Computer(this);
        }
    }
}
```

The calling code becomes self-documenting in a way a constructor never can:

```java
Computer workstation = new Computer.Builder("Intel i7")
        .ram(32)
        .storage(1024)
        .graphicsCard("RTX 4080")
        .build();

Computer office = new Computer.Builder("AMD Ryzen 5")
        .build();
```

Every named method removes a class of mistakes. You cannot swap the ram and storage arguments anymore because they have names. The defaults live in one place. The invariants live in `build()`, which is the only point where the object transitions from "being assembled" to "existing," and therefore the only sensible place for validation to run.

Diagram: Builder class structure

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 930 360" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#444"/>
    </marker>
  </defs>

  <rect x="40" y="30" width="220" height="60" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="150" y="52" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">Director</text>
  <text x="150" y="74" text-anchor="middle" font-size="12" fill="#1a2733">+construct(config)</text>

  <rect x="300" y="30" width="260" height="130" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="430" y="46" text-anchor="middle" font-size="12" font-style="italic" fill="#5a6b7a">&lt;&lt;interface&gt;&gt;</text>
  <text x="430" y="66" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">Builder</text>
  <text x="430" y="88" text-anchor="middle" font-size="12" fill="#1a2733">setCpu(String)</text>
  <text x="430" y="108" text-anchor="middle" font-size="12" fill="#1a2733">setRam(int)</text>
  <text x="430" y="126" text-anchor="middle" font-size="12" fill="#1a2733">setStorage(int)</text>
  <text x="430" y="144" text-anchor="middle" font-size="12" fill="#1a2733">build(): Computer</text>

  <rect x="300" y="200" width="260" height="125" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="430" y="222" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">Computer.Builder</text>
  <text x="430" y="244" text-anchor="middle" font-size="12" fill="#1a2733">-product: Computer</text>
  <text x="430" y="266" text-anchor="middle" font-size="12" fill="#1a2733">+setCpu(String)</text>
  <text x="430" y="288" text-anchor="middle" font-size="12" fill="#1a2733">+setRam(int)</text>
  <text x="430" y="310" text-anchor="middle" font-size="12" fill="#1a2733">+build(): Computer</text>

  <rect x="620" y="30" width="280" height="220" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="760" y="50" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">Computer</text>
  <text x="760" y="74" text-anchor="middle" font-size="12" fill="#1a2733">-cpu: String</text>
  <text x="760" y="96" text-anchor="middle" font-size="12" fill="#1a2733">-ram: int</text>
  <text x="760" y="118" text-anchor="middle" font-size="12" fill="#1a2733">-storage: int</text>
  <text x="760" y="140" text-anchor="middle" font-size="12" fill="#1a2733">-graphicsCard: String</text>
  <text x="760" y="162" text-anchor="middle" font-size="12" fill="#1a2733">+Computer(Builder)</text>

  <line x1="260" y1="60" x2="298" y2="60" stroke="#6b7a89" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
  <line x1="430" y1="200" x2="430" y2="162" stroke="#33475b" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
  <line x1="560" y1="60" x2="618" y2="60" stroke="#6b7a89" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
  <line x1="560" y1="270" x2="618" y2="210" stroke="#6b7a89" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
</svg>
```

One structural note before the fluent part. The GoF Builder has a `Director` that drives the builder through a fixed construction sequence, which is useful when many products share a procedure. In Java practice the Director is almost always dropped, and the client plays that role by chaining methods directly. The fluent chain you use daily is the pattern minus the director. If your building sequence is identical across many call sites, consider reintroducing it, but for most code the chain is clearer.

### The fluent variant

Fluent builder is the Builder where every setter returns the builder itself, enabling the chain. The trick that makes it ergonomic is that the *compile-time* type can shrink as you call methods. You can encode "cpu must be set first" by having the first step return a different type that only exposes the remaining methods:

```java
public class Computer {
    public static CpuStep builder() {
        return new Builder();
    }

    public interface CpuStep {
        RamStep cpu(String cpu);
    }

    public interface RamStep {
        OptionalStep ram(int ram);
        OptionalStep storage(int storage);
        BuildStep graphicsCard(String graphicsCard);
    }
    ...
}
```

That "staged builder" is overkill for most real objects. It pays off only when the construction order genuinely encodes correctness, for instance when a query builder requires a `from` before a `where`. The ordinary fluent builder, where every method returns the same type and you trust the caller to set what matters, covers 95 percent of needs. Do not build the staged version on day one. Build it when the bug reports tell you the order matters.

## Real Production Usage

The JDK ships the most famous one: `StringBuilder` is the pattern stripped to the bone, a mutable buffer you fill and then `toString()` once. `java.net.URI` and `javax.ws.rs.core.UriBuilder` are fluent builders. Guava's `ImmutableMap.builder()` and `ImmutableList.builder()` are the pattern applied to collections that refuse mutation. Spring uses it everywhere: `UriComponentsBuilder`, `WebClient` through its `RequestHeadersSpec` family, and the `CriteriaBuilder` in JPA. OkHttp's `Request.Builder` and Retrofit's `Retrofit.Builder` show the pattern dominating the HTTP client world.

The other common sight is Lombok's `@Builder`, which generates the builder for you. Using it is not cheating. The pattern is about the shape of construction, not who typed the boilerplate.

## Common Mistakes

**Turning every object into a builder because builders look modern.** A `Point` with three coordinates does not need a builder. A builder hides the real constructor behind indirection, and for small objects the direct constructor is clearer. The pattern earns its keep with several optional fields or validation policy, not with a large class.

**Mutating the product through the builder after build.** If the builder keeps references to collections it handed to the product, callers can mutate the "immutable" object. Copy the data at `build()` time. The immutable product is the entire point; a builder that leaks internal collections is a trap.

**Letting the builder grow validation and defaults until it becomes a second implementation of the class.** When the builder starts re-deriving values and coordinating fields, you have two places that know how the product works. Keep the builder dumb: collect fields, apply defaults, validate in `build()`.

## Interview Perspective

Builders are the creational pattern interviewers can safely assume you have touched, because nobody survives a real codebase without meeting one. So the bar moves from "can you describe it" to "do you know why it exists." A weak answer defines it as "a class that builds objects step by step." A strong answer starts from the telescoping constructor, explains that named setter methods fix argument-order errors that no compiler catches, and notes that `build()` is the only sane home for invariants on an immutable object.

The strongest candidates can say when they would *not* use a Builder, which is the tell that the pattern was learned through pain rather than a diagram.

Common follow-ups:

- "Why does Builder work so well with immutable objects, specifically?"
- "How would you validate required fields, and why does that not belong in the setters?"

## Knowledge Check

1. The `Computer` gains a `boolean hasWifi` field with a default of `true`. Describe the telescoping-constructor change versus the Builder change, and which class of bug each introduces or removes.
2. A teammate wants the builder to throw on missing `cpu` at `build()` time. Where does the check have to live, and what can it not catch that a staged builder could?
3. Explain why `StringBuilder` is a Builder even though it never validates anything and never returns a `Builder` from a build method.

## Key Takeaways

- Builder exists for optional fields and validation policy, not for object size, and it pairs naturally with immutability.
- Named setter methods eliminate argument-order errors that constructors cannot catch.
- `build()` is the only point where the object is complete, so it is the only place invariants belong.
- The fluent chain is Builder without the Director; reintroduce the Director only when many callers share a construction sequence.
- The JDK, Spring, JPA, and the HTTP client world all run on the pattern, which is why it is worth knowing cold.

## What's Next

The next article is Prototype, the creational pattern that flips the premise of everything so far. Instead of constructing an object from nothing, Prototype constructs new objects by copying an existing one. We will cover when construction is expensive enough to justify copying, why `Object.clone()` is the sharpest-edged API in the whole JDK, and the difference between shallow and deep copies that trips up almost everyone.

---

This article explains the Builder pattern as the answer to telescoping constructors and unlabeled argument lists, with an immutable `Computer` as the running example and a fluent chain in Java. It argues that `build()` is the only legitimate home for validation and defaults, and that a builder is justified by optional fields and policy, not by object size.
