# Abstract Factory Pattern

## Learning Objectives

- Distinguish Abstract Factory from Factory Method on the real axis: a coordinated family of related objects versus a single object, and why the coordination is the whole point.
- Implement the pattern in Java with one abstract factory interface and two concrete family factories, and explain what breaks if the caller mixes products from different families.
- Argue the cost side honestly: when the pattern pays for itself and when it is overhead that a plain factory would have covered.

## Introduction

Abstract Factory provides an interface for creating families of related objects without specifying their concrete classes. The word that carries the meaning is "families." Factory Method creates one object. Abstract Factory creates a set of objects that are designed to work together, and it guarantees that everything a caller receives comes from the same set.

Why does that matter? Because "related" is doing real work. A button and a checkbox from the same UI toolkit match each other visually and behaviorally. A button from one toolkit and a checkbox from another do not. The pattern exists to make it structurally impossible to build a mismatched set, not merely unlikely.

## Problem Statement

Think about a UI framework that supports two looks: a native Windows look and a macOS look. Each look is really a bundle. Windows has `WindowsButton`, `WindowsCheckbox`, `WindowsDialog`. macOS has the Mac equivalents. The constraint is not "I need a button." The constraint is "I need a button that matches the checkbox and the dialog, because they are going to appear in the same window."

If you create each widget with its own little factory, you get this failure:

```java
Button button = ButtonFactory.create();
Checkbox checkbox = MacCheckboxFactory.create();
```

Nothing stops that code. It compiles. And then your window renders a Windows button next to a Mac checkbox, which is broken for reasons that have nothing to do with layout. The failure mode of Factory Method used naively for families is that each product is chosen independently, so the family invariant is not enforceable. The more factories you have, the more combinations you can accidentally mix.

The second failure mode is what happens when a new look arrives. Without a central place that knows "these products belong together," the codebase accumulates scattered branches: an `if (isMac)` here, a `switch (theme)` there. Adding a Linux look means hunting every branch. What you want is a single decision, made once, that everything downstream inherits.

## Core Concept

Abstract Factory inverts that. Instead of many little factories, you define one factory interface whose methods each create one product of the family. Then you implement the interface once per family.

```java
public interface Button {
    void paint();
}

public interface Checkbox {
    void paint();
}
```

```java
public interface UIFactory {
    Button createButton();
    Checkbox createCheckbox();
}
```

```java
public class WindowsButton implements Button {
    @Override
    public void paint() {
        // render a Windows-style button
    }
}

public class WindowsCheckbox implements Checkbox {
    @Override
    public void paint() {
        // render a Windows-style checkbox
    }
}

public class WindowsUIFactory implements UIFactory {
    @Override
    public Button createButton() {
        return new WindowsButton();
    }

    @Override
    public Checkbox createCheckbox() {
        return new WindowsCheckbox();
    }
}
```

And the same shape for the Mac family. The payoff is in the client. A window takes a `UIFactory` and builds everything from it:

```java
public class Window {
    private final Button button;
    private final Checkbox checkbox;

    public Window(UIFactory factory) {
        this.button = factory.createButton();
        this.checkbox = factory.createCheckbox();
    }
}
```

Because `Window` holds one `UIFactory` and asks it for everything, it cannot accidentally mix families. It is not a convention the team must remember; it is the only thing the code allows. That is the difference between a rule enforced by the type system and a rule enforced by code review.

Diagram: Abstract Factory family structure

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1280 580" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#444"/>
    </marker>
  </defs>

  <rect x="480" y="30" width="320" height="100" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="640" y="48" text-anchor="middle" font-size="12" font-style="italic" fill="#5a6b7a">&lt;&lt;interface&gt;&gt;</text>
  <text x="640" y="68" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">UIFactory</text>
  <text x="640" y="90" text-anchor="middle" font-size="13" fill="#1a2733">createButton()</text>
  <text x="640" y="112" text-anchor="middle" font-size="13" fill="#1a2733">createCheckbox()</text>

  <rect x="400" y="220" width="190" height="80" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="495" y="244" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">WindowsUIFactory</text>
  <text x="495" y="270" text-anchor="middle" font-size="13" fill="#1a2733">createButton()</text>
  <text x="495" y="292" text-anchor="middle" font-size="13" fill="#1a2733">createCheckbox()</text>

  <rect x="690" y="220" width="190" height="80" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="785" y="244" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">MacUIFactory</text>
  <text x="785" y="270" text-anchor="middle" font-size="13" fill="#1a2733">createButton()</text>
  <text x="785" y="292" text-anchor="middle" font-size="13" fill="#1a2733">createCheckbox()</text>

  <line x1="495" y1="220" x2="495" y2="132" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="785" y1="220" x2="785" y2="132" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>

  <rect x="30" y="30" width="220" height="60" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="140" y="48" text-anchor="middle" font-size="12" font-style="italic" fill="#5a6b7a">&lt;&lt;interface&gt;&gt;</text>
  <text x="140" y="66" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">Button</text>
  <text x="140" y="84" text-anchor="middle" font-size="13" fill="#1a2733">paint()</text>

  <rect x="30" y="220" width="220" height="80" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="140" y="246" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">WindowsButton</text>
  <text x="140" y="274" text-anchor="middle" font-size="13" fill="#1a2733">paint()</text>

  <rect x="30" y="390" width="220" height="60" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="140" y="408" text-anchor="middle" font-size="12" font-style="italic" fill="#5a6b7a">&lt;&lt;interface&gt;&gt;</text>
  <text x="140" y="426" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">Checkbox</text>
  <text x="140" y="444" text-anchor="middle" font-size="13" fill="#1a2733">paint()</text>

  <rect x="30" y="470" width="220" height="80" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="140" y="496" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">WindowsCheckbox</text>
  <text x="140" y="524" text-anchor="middle" font-size="13" fill="#1a2733">paint()</text>

  <line x1="140" y1="220" x2="140" y2="92" stroke="#33475b" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
  <line x1="140" y1="470" x2="140" y2="452" stroke="#33475b" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>

  <rect x="1030" y="30" width="220" height="60" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="1140" y="48" text-anchor="middle" font-size="12" font-style="italic" fill="#5a6b7a">&lt;&lt;interface&gt;&gt;</text>
  <text x="1140" y="66" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">Button</text>
  <text x="1140" y="84" text-anchor="middle" font-size="13" fill="#1a2733">paint()</text>

  <rect x="1030" y="220" width="220" height="80" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="1140" y="246" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">MacButton</text>
  <text x="1140" y="274" text-anchor="middle" font-size="13" fill="#1a2733">paint()</text>

  <rect x="1030" y="390" width="220" height="60" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="1140" y="408" text-anchor="middle" font-size="12" font-style="italic" fill="#5a6b7a">&lt;&lt;interface&gt;&gt;</text>
  <text x="1140" y="426" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">Checkbox</text>
  <text x="1140" y="444" text-anchor="middle" font-size="13" fill="#1a2733">paint()</text>

  <rect x="1030" y="470" width="220" height="80" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="1140" y="496" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">MacCheckbox</text>
  <text x="1140" y="524" text-anchor="middle" font-size="13" fill="#1a2733">paint()</text>

  <line x1="1140" y1="220" x2="1140" y2="92" stroke="#33475b" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
  <line x1="1140" y1="470" x2="1140" y2="452" stroke="#33475b" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>

  <line x1="400" y1="260" x2="252" y2="260" stroke="#6b7a89" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
  <line x1="400" y1="285" x2="252" y2="510" stroke="#6b7a89" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
  <line x1="880" y1="260" x2="1028" y2="260" stroke="#6b7a89" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
  <line x1="880" y1="285" x2="1028" y2="510" stroke="#6b7a89" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
</svg>
```

The diagram earns its space because the structural claim of the pattern is hard to state precisely in prose: the factory interface points at every product interface, and each concrete factory points at exactly the concrete products of its own family. The dashed arrows that stay inside one horizontal band are the whole guarantee.

There is a real tradeoff, and the overview said it would be honest. Abstract Factory is the most expensive creational pattern to introduce. Every new product type means touching the factory interface, which means touching every concrete factory. Add a `Slider` to the UI example and you modify `UIFactory`, `WindowsUIFactory`, and `MacUIFactory`, plus the new slider classes. The pattern buys family consistency at the price of a rigid interface. That rigidity is acceptable when the product set is stable, and painful when it is not. The moment your product list is still being invented, a plain factory or even direct construction is the cheaper bet.

Also worth knowing: in Java, a whole class of "abstract factories" are actually implemented as static factory methods that return factory objects. `DocumentBuilderFactory.newInstance()` returns a concrete `DocumentBuilderFactory` chosen by system properties. That is the factory-of-factories shape the pattern describes, industrialized.

## Real Production Usage

`javax.xml.parsers.DocumentBuilderFactory` is the canonical JVM example. You call `newInstance()`, and the JVM picks a concrete factory from `META-INF/services` or system properties. From that factory you get a `DocumentBuilder`, and from the builder a `Document`. The family is "a coherent XML parsing stack," and the pattern guarantees the builder you hold matches the factory that produced it.

The same shape repeats across the JDK: `SAXParserFactory`, `TransformerFactory`, `SchemaFactory` all follow it. Spring's `BeanFactory` and `ApplicationContext` are abstract-factory-like at a much larger scale: one object answers "give me a bean," and the concrete implementation (XML, annotation, Groovy, Kotlin) determines the entire family of behavior. The pattern is not exotic in the JVM, it is the backbone of how parsers and containers get built.

## Common Mistakes

**Using Abstract Factory when there is no family to keep consistent.** If you have two factory methods that never need to stay coordinated, the pattern adds an interface and a contract with no payoff. The tell is whether a caller could meaningfully combine products from different factories. If it could and nothing breaks, you do not have a family, you have two unrelated factories, and you should have used Factory Method twice.

**Mixing families anyway because "it is just a demo."** The pattern's guarantee is only as strong as the discipline of the callers. If someone constructs a `WindowsButton` directly next to a `MacCheckbox`, the type system cannot stop it. The pattern defends the happy path; it does not make the unhappy path unthinkable.

**Adding a product type to the family without planning for it.** Every interface change ripples through every concrete factory. If you anticipate new product types frequently, the rigidity of Abstract Factory becomes a tax you pay on every change. Budget for it or pick a less rigid construction strategy.

## Interview Perspective

Interviewers ask Abstract Factory to test whether you understand the difference between "a factory that creates objects" and "a factory that creates a consistent family." The follow-ups are where the depth shows.

A weak answer: "It is a factory that creates other factories." A strong answer names the invariant: "Abstract Factory groups the construction of related products so that a client holding one factory cannot assemble a mismatched set, at the cost of touching every concrete factory whenever the product list changes." If you can also say when you would decline to use it, you have actually used it.

Common follow-ups:

- "Walk me through the class structure of Abstract Factory with a new product type added to an existing family."
- "Your family has three factories and five product types. A sixth product type arrives. Which files change?"

## Knowledge Check

1. Add a `Slider` product to the UI example above. List every file that changes and explain why the factory interface, not just the concrete classes, has to change.
2. A teammate proposes replacing the `UIFactory` interface with two standalone factory methods, `createButton()` and `createCheckbox()`, because "the interface has only two methods anyway." What invariant do you lose, and how does the failure show up at runtime?
3. `DocumentBuilderFactory.newInstance()` returns a factory chosen by runtime properties. Where does the family guarantee live in that design, and who is responsible for keeping it?

## Key Takeaways

- Abstract Factory guarantees family consistency: everything a client gets comes from one coordinated set.
- The pattern is Factory Method applied to a set of products that must stay compatible, and the interface spans all of them.
- The guarantee is structural, not procedural: a client holding one factory cannot easily mix families.
- It is the most expensive creational pattern to evolve, because every product type change touches every concrete factory.
- The JDK's parser factories and Spring's bean containers are the pattern at industrial scale.

## What's Next

The next article switches from "which class" to "how a single class gets assembled." Builder and its fluent variant solve the constructor problem: objects with many optional fields, immutable objects, and construction sequences that are easy to get wrong. We will cover the standard telescoping-constructor failure, how the Builder pattern fixes it, and where the fluent style becomes a tool in its own right.

---

This article explains Abstract Factory as the pattern that guarantees a family of related objects stays consistent, using a UI toolkit with Windows and macOS variants as the running example. It argues that the family guarantee is the entire reason the pattern exists, and that its rigid interface is the price you pay for that guarantee.
