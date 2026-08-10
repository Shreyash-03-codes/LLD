# Decorator Pattern

## Learning Objectives

- Attach behavior to an object at runtime by wrapping it, and explain why the wrapper must implement the same interface as the wrapped object.
- Recognize the subclass explosion that Decorator prevents and show the count for feature combinations with and without it.
- Argue the identity problem: a wrapped object is not the same object, and code that assumes it is will break.

## Introduction

Decorator lets you add responsibilities to an object dynamically by wrapping it in another object that implements the same interface. The wrapper does its own work, then delegates the rest to the object it holds. Stack several wrappers and you get layered behavior: the innermost object does the core job, and each layer adds one feature on top.

This is Composite's mirror image. Composite builds a tree where a node holds many children. Decorator builds a stack where each node holds exactly one other node. The wrapping is single-threaded and linear, and that linearity is what makes the layers predictable.

## Problem Statement

The failure is the subclass explosion. A coffee shop sells a base espresso and every combination of add-ons: milk, sugar, whipped cream, caramel. The naive model makes a class for every combination:

```java
class EspressoWithMilk {}
class EspressoWithSugar {}
class EspressoWithMilkAndSugar {}
class EspressoWithMilkAndWhippedCream {}
class EspressoWithSugarAndWhippedCream {}
class EspressoWithMilkAndSugarAndWhippedCream {}
```

With three add-ons there are already eight classes. With eight add-ons you are writing two hundred and fifty-six classes, and the point where you stop writing them and start generating them is the point where the design has already lost. Two details make it worse. Each combination needs its own price logic, so the pricing code is scattered across hundreds of classes. And the set is closed at compile time, so a new add-on, oat milk, means writing another power of two of classes.

The alternative that looks cheaper is a giant mutable config object:

```java
class Espresso {
    boolean milk;
    boolean sugar;
    boolean whippedCream;
    double cost() { return 2.50 + (milk ? 0.50 : 0) + (sugar ? 0.25 : 0) + (whippedCream ? 0.75 : 0); }
}
```

That collapses the classes but poisons everything else. The object's shape changes as flags flip, code everywhere reads and writes the flags, and there is no longer a way to say "this specific drink" as a type. The explosion and the flag object are two ends of the same mistake: they both fail to model the simple fact that a drink is a core wrapped in layers.

## Core Concept

Decorator models exactly that. One interface describes a beverage:

```java
public interface Beverage {
    String description();
    double cost();
}
```

The core implements it plainly:

```java
public class Espresso implements Beverage {
    @Override
    public String description() {
        return "Espresso";
    }

    @Override
    public double cost() {
        return 2.50;
    }
}
```

Each add-on is a wrapper: same interface, holds the object beneath it, adds its own contribution, and delegates the rest:

```java
public abstract class Addon implements Beverage {
    protected final Beverage base;

    protected Addon(Beverage base) {
        this.base = base;
    }
}

public class Milk extends Addon {
    public Milk(Beverage base) {
        super(base);
    }

    @Override
    public String description() {
        return base.description() + ", milk";
    }

    @Override
    public double cost() {
        return base.cost() + 0.50;
    }
}

public class WhippedCream extends Addon {
    public WhippedCream(Beverage base) {
        super(base);
    }

    @Override
    public String description() {
        return base.description() + ", whipped cream";
    }

    @Override
    public double cost() {
        return base.cost() + 0.75;
    }
}
```

Ordering is just nesting:

```java
Beverage drink = new WhippedCream(new Milk(new Espresso()));
System.out.println(drink.description());  // Espresso, milk, whipped cream
System.out.println(drink.cost());         // 3.75
```

Eight add-ons now cost eight wrapper classes, not two hundred and fifty-six combination classes, and the set is open: oat milk is one new class, immediately combinable with everything else. No class in the system has to change when a new add-on arrives. That open-endedness, each layer knowing only the layer beneath it, is the entire payoff.

Diagram: layered wrapping

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 500" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#444"/>
    </marker>
  </defs>

  <rect x="120" y="24" width="520" height="440" fill="none" stroke="#33475b" stroke-width="1.5"/>
  <text x="140" y="60" font-size="13" font-weight="bold" fill="#1a2733">WhippedCream</text>

  <rect x="220" y="110" width="360" height="280" fill="none" stroke="#33475b" stroke-width="1.5"/>
  <text x="240" y="140" font-size="13" font-weight="bold" fill="#1a2733">Milk</text>

  <rect x="320" y="200" width="160" height="80" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="400" y="222" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">Espresso</text>
  <text x="400" y="248" text-anchor="middle" font-size="12" fill="#1a2733">cost() = 2.50</text>

  <rect x="680" y="190" width="180" height="100" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="770" y="212" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">Client</text>
  <text x="770" y="238" text-anchor="middle" font-size="12" fill="#1a2733">+cost()</text>

  <line x1="680" y1="240" x2="642" y2="240" stroke="#6b7a89" stroke-width="1.5" marker-end="url(#arrow)"/>
</svg>
```

Each ring is a decorator. `WhippedCream` wraps `Milk` wraps `Espresso`, and a call to `cost()` passes through every layer, each adding its share before delegating inward. The arrow shows the client calling the outermost ring, which is all the client ever sees.

### Why the interface, and why it matters

The wrapper implements `Beverage`, and that choice is not incidental. Because `Milk` is a `Beverage`, it can wrap another decorator, and the chain can be any length. If wrappers implemented a different type, you could only wrap the core once. The uniform interface is what makes the layering unbounded. This is also the line that separates Decorator from the Adapter and Proxy patterns that look identical on paper: a decorator preserves the interface and adds behavior, an adapter changes the interface to make things fit, a proxy preserves the interface but controls access rather than adding behavior. Same shape, three intents, and the intent is what you have to name.

## Real Production Usage

The Java I/O stack is the canonical decorator, and reading it is like reading the pattern in a foreign accent. `new BufferedReader(new InputStreamReader(new FileInputStream(path)))` wraps a file stream in a character translator, then in a buffer. Each wrapper is a `Reader` or `InputStream` that holds the one beneath it. `Collections.synchronizedList(...)` wraps a list to add thread safety, and `Collections.unmodifiableList(...)` wraps one to forbid mutation; both are decorators that preserve the `List` interface while changing behavior.

Servlet wrappers are the other famous case. `javax.servlet.http.HttpServletRequestWrapper` and `HttpServletResponseWrapper` exist so frameworks can wrap a request, add a header, log the body, or rewrite a URL, while still presenting a `HttpServletRequest` to the application. Spring Security's `SecurityContextHolderAwareRequestWrapper` is a real wrapper of exactly this kind, decorating a request with security-related convenience methods. When you see an object that stands in for another object of the same interface and does a little extra, you are seeing a decorator, whether anyone named it or not.

The wrapping also has a cost you pay in the debugger. A `BufferedReader` around an `InputStreamReader` around a `FileInputStream` fails as a stack of frames, and the real exception sits at the bottom of the chain. That is the price of the pattern, and it is worth naming: every wrapper is a layer of indirection a debugging session has to unwind, usually by walking `getCause()` one frame at a time. The skill is keeping each wrapper small enough that the unwind is mechanical, which is another reason wrappers with four unrelated responsibilities are the worst wrappers of all.

## Common Mistakes

**Forgetting that the wrapper is not the wrapped object.** A `Beverage` wrapped in `Milk` is a different object, with different identity, `equals`, and `hashCode`. Code that stores `Beverage` instances in a `Map` and looks them up later will silently miss, because the wrapped reference never matches the stored one. If identity matters, do not wrap; if you must wrap, never rely on identity.

**Composing wrappers in the wrong order.** Each layer's behavior depends on what it calls beneath it. `new WhippedCream(new Milk(...))` and `new Milk(new WhippedCream(...))` are different drinks. With real wrappers like streams, order is often load-bearing: buffering an encrypted stream is not the same as encrypting a buffered stream. Think about which layer must see what.

**Wrapping when a simple implementation would do.** A decorator adds a class and a level of indirection. If the extra behavior is fixed and known, a conditional inside the core class, or a subclass, is cheaper. Decorator earns its stack when the combinations are open-ended, which is the exact situation where subclasses explode.

## Interview Perspective

Decorator is one of the few patterns interviewers can assume you have touched, because nobody writes Java I/O without meeting `BufferedInputStream`. So the bar is depth. A weak answer defines wrapping and draws the stack. A strong answer starts from the subclass explosion, shows the count for combinations with and without the pattern, and can name the identity problem, which is the part that comes from real usage rather than the textbook.

The follow-up that separates candidates is the trio again. "How is a Decorator different from a Proxy?" The strong answer names the intent: a decorator adds behavior and the client usually does not care that wrapping happened, a proxy controls access and the client usually must not know the real object exists.

Common follow-ups:

- "Your wrapped object is used as a `Map` key and lookups start failing. What happened?"
- "When does a decorator stop being worth it versus just writing the behavior into the class?"

## Knowledge Check

1. Count the classes for 5 add-ons with the naive combination model, the flag object model, and the decorator model, and say which model stays open to a sixth add-on.
2. `new Milk(new WhippedCream(espresso))` versus `new WhippedCream(new Milk(espresso))`: what changes and why does order matter in general, not just here?
3. A `BufferedReader` wraps a `FileReader`. Which is the component, which is the decorator, and what interface does each implement that makes the layering legal?

## Key Takeaways

- Decorator wraps one object in another of the same interface, adding behavior while delegating the rest, and the chain can be unbounded.
- The subclass explosion is the diagnostic, and the flag object is the same failure wearing different clothes.
- The uniform interface is what makes stacking legal, which is also what makes Decorator, Adapter, and Proxy look identical in a diagram.
- The Java I/O stack and the servlet request wrappers are Decorator in daily production use.
- Identity, ordering, and over-wrapping are the three ways the pattern breaks in real code.

## What's Next

The next article is Facade, which takes the opposite approach to the same problem of complexity. Instead of adding layers of wrappers that each do a little more, Facade removes layers of awareness: the client stops knowing about the ten subsystem classes and talks to one front door. We will cover what a facade can and cannot hide, and why the best facades are deliberately thin.

---

This article explains the Decorator pattern as wrapping an object in layers of same-interface objects, with the coffee add-on stack as the running example. It argues that the subclass explosion is the diagnostic, and that the identity and ordering problems are what separate people who have used the pattern from people who have only drawn it.
