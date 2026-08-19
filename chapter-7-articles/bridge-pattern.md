# Bridge Pattern

## Learning Objectives

- Explain why Bridge exists by first watching the inheritance explosion happen, so the two-hierarchy split feels earned rather than clever.
- Implement the bridge in Java by separating an abstraction hierarchy from an implementation hierarchy joined by a reference.
- Distinguish Bridge from Adapter with a one-line rule about timing and intent.

## Introduction

Bridge decouples an abstraction from its implementation so the two can vary independently. Where Adapter patches a mismatch that already exists, Bridge is performed in advance: you notice that your domain has two independent dimensions of variation, and you split them into two hierarchies instead of letting their combinations multiply.

The name is literal. The two hierarchies are connected by a thin reference, the bridge, and neither side cares about the details of the other.

## Problem Statement

Watch a class hierarchy rot before it exists. You build a remote control system for a home theater. You have devices, TVs and sound bars, and you have input types, buttons and voice. The naive design creates a class for every combination:

```java
class TvButtonRemote {}
class TvVoiceRemote {}
class SoundBarButtonRemote {}
class SoundBarVoiceRemote {}
```

Two dimensions, two values each, four classes. Add a projector and a gesture input and the count goes to nine. Add another device and another input and you are at sixteen. This is the inheritance explosion, and it has a distinctive smell: the class names are concatenations of the two dimensions, `SoundBarVoiceRemote`. Every time you see a name that is really two concepts glued together, you are looking at a missing Bridge.

The deeper problem is not the count. It is that the dimensions are locked together. Every class in the grid hardcodes one choice from each axis, so changing how voice input works means touching every class that names voice in its title. The two dimensions cannot evolve independently, because they were never separated. What you want is a design where adding a new device touches one file and adding a new input type touches one file, and never the same file.

## Core Concept

Bridge separates the axes. One hierarchy holds the abstraction, the remote control, and delegates the device-specific work to a second hierarchy, the device, through a reference.

```java
public interface Device {
    void powerOn();
    void powerOff();
    void setVolume(int volume);
}

public class Tv implements Device {
    @Override
    public void powerOn() {
        // IR signal for the TV
    }

    @Override
    public void powerOff() {
        // IR signal for the TV
    }

    @Override
    public void setVolume(int volume) {
        // TV volume code
    }
}

public class SoundBar implements Device {
    @Override
    public void powerOn() {
        // IR signal for the sound bar
    }

    @Override
    public void powerOff() {
        // IR signal for the sound bar
    }

    @Override
    public void setVolume(int volume) {
        // sound bar volume code
    }
}
```

The abstraction hierarchy holds a reference to a `Device` and delegates:

```java
public abstract class RemoteControl {
    protected final Device device;

    protected RemoteControl(Device device) {
        this.device = device;
    }

    public abstract void pressButton(int code);
}

public class BasicRemote extends RemoteControl {
    public BasicRemote(Device device) {
        super(device);
    }

    @Override
    public void pressButton(int code) {
        if (code == 1) {
            device.powerOn();
        } else if (code == 2) {
            device.powerOff();
        }
    }
}

public class VoiceRemote extends RemoteControl {
    public VoiceRemote(Device device) {
        super(device);
    }

    @Override
    public void pressButton(int code) {
        // translate voice command into a device call
    }
}
```

Now the composition does the work the grid of classes was doing. `new VoiceRemote(new SoundBar())` is the old `SoundBarVoiceRemote`. Adding a projector means adding `Projector implements Device`, one file, and it immediately works with every remote. Adding a gesture remote means adding one class to the abstraction hierarchy, and it immediately works with every device. Two dimensions of variation, and each new value costs one class instead of one class per combination.

Diagram: bridge between the two hierarchies

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1060 370" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#444"/>
    </marker>
  </defs>

  <rect x="40" y="40" width="560" height="90" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="320" y="64" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">RemoteControl</text>
  <text x="320" y="88" text-anchor="middle" font-size="12" fill="#1a2733">-device: Device</text>
  <text x="320" y="110" text-anchor="middle" font-size="12" fill="#1a2733">+pressButton(code)</text>

  <rect x="40" y="230" width="260" height="90" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="170" y="254" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">BasicRemote</text>
  <text x="170" y="278" text-anchor="middle" font-size="12" fill="#1a2733">+pressButton(code)</text>

  <rect x="340" y="230" width="260" height="90" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="470" y="254" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">VoiceRemote</text>
  <text x="470" y="278" text-anchor="middle" font-size="12" fill="#1a2733">+pressButton(code)</text>

  <line x1="170" y1="230" x2="170" y2="132" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="470" y1="230" x2="470" y2="132" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>

  <rect x="660" y="40" width="260" height="80" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="790" y="58" text-anchor="middle" font-size="12" font-style="italic" fill="#5a6b7a">&lt;&lt;interface&gt;&gt;</text>
  <text x="790" y="78" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">Device</text>
  <text x="790" y="100" text-anchor="middle" font-size="12" fill="#1a2733">+powerOn() +setVolume(v)</text>

  <rect x="560" y="230" width="220" height="90" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="670" y="254" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">Tv</text>
  <text x="670" y="278" text-anchor="middle" font-size="12" fill="#1a2733">+powerOn() +setVolume(v)</text>

  <rect x="800" y="230" width="220" height="90" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="910" y="254" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">SoundBar</text>
  <text x="910" y="278" text-anchor="middle" font-size="12" fill="#1a2733">+powerOn() +setVolume(v)</text>

  <line x1="670" y1="230" x2="670" y2="122" stroke="#33475b" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
  <line x1="910" y1="230" x2="910" y2="122" stroke="#33475b" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>

  <line x1="600" y1="90" x2="658" y2="90" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
</svg>
```

The horizontal line at the top is the bridge. `RemoteControl` holds a `Device`, and that single reference is what lets two hierarchies each grow without multiplying. Nothing about `Device` knows remotes exist, and nothing about `RemoteControl` knows TVs or sound bars exist.

### The timing test

The cleanest way to separate Bridge from Adapter is to ask when the split happens. Adapter responds to an existing mismatch: the vendor's interface is already there and it is wrong for you, so you translate. Bridge is proactive: you look at a domain with two growing axes and you split them before the class grid appears. Adapter is called when it is too late. Bridge is the move you make when it is just in time. If someone hands you a pile of `X_Y_Z` classes and says "make this better," you refactor into a bridge. If someone hands you a vendor class with a wrong method name, you write an adapter. One fixes the past, the other shapes the future.

There is a caution worth stating. Bridge adds a level of indirection to every call, and it is tempting to over-apply it. A domain with a single dimension of variation does not need a bridge, it needs inheritance or composition on its own. Bridge is justified when two dimensions are *independently* likely to grow. If the remote controls and the devices both change rarely, the grid of four classes is fine and the bridge is extra machinery. The pattern pays off at the intersection of two real growth curves.

## Real Production Usage

The JVM's most famous bridge is `java.sql.DriverManager` and the JDBC driver contract. The abstraction is the `Connection`, `Statement`, and `ResultSet` interfaces, fixed by the JDK, and the implementation is whatever the database vendor's driver provides. Your code writes to the abstraction, and the driver changes underneath without your code noticing. That is Bridge with the abstraction and implementation sold separately, which is exactly what happens when a spec body fixes one side and vendors compete on the other.

The logging world shows why the boundary blurs. SLF4J is usually called a facade, and its backends adapters, but the shape is the same one this article draws: an API your code depends on, several implementations you never name, and no inheritance between the two sides. Read it either way and the lesson is identical. Spring's `PlatformTransactionManager` interface with `DataSourceTransactionManager`, `JpaTransactionManager`, and `JmsTransactionManager` implementations is a bridge between the abstraction of "a transaction" and the reality of where the transaction lives. In each case the pattern is recognizable by the same signature: one interface your code depends on, several implementations you never name, and no inheritance between the two sides.

## Common Mistakes

**Trying to fix a broken hierarchy with an adapter.** If the class grid already exists, an adapter patches one corner and the grid keeps growing. The correct move is the refactor into two hierarchies. Adapter is a bandage; Bridge is the reconstruction. Using one where the other belongs is how these get tangled.

**Over-applying the bridge to one-dimensional domains.** A single axis of variation does not need the split. The bridge earns its indirection at the intersection of two growth curves, and adding it elsewhere is ceremony with a diagram.

**Letting the two hierarchies leak into each other.** If the abstraction starts referencing concrete `Tv` methods, or the implementation starts knowing about remotes, the bridge is a lie and you are back to a fused grid with extra classes.

## Interview Perspective

Bridge is a differentiator in interviews because it is the pattern people can name but rarely justify. A weak answer defines the two hierarchies and draws the line between them. A strong answer starts with the inheritance explosion, shows the class grid forming, and explains the split as a preemptive move, which is what separates Bridge from Adapter.

The interviewers probe the distinction directly, since Adapter and Bridge look similar in a diagram and are opposite in timing. Being able to say "Adapter reacts to a fixed mismatch, Bridge splits a growing pair of axes in advance" in one sentence demonstrates that you have seen both failures in real code.

Common follow-ups:

- "When is a bridge overkill, and what would you use instead?"
- "Draw the class count for two dimensions of size N and M with and without a bridge."

## Knowledge Check

1. A domain has 3 device types and 4 remote types. Compute the class count with the fused grid and with a bridge, and show how each new value of either dimension changes the count.
2. You are handed `ProjectorVoiceRemote` and `ProjectorButtonRemote`. Reconstruct the two axes and sketch the refactor into a bridge.
3. `java.sql.Connection` is an interface with one implementation per database vendor. Which hierarchy is the abstraction, which is the implementation, and who is the bridge reference?

## Key Takeaways

- Bridge splits two independently growing dimensions into two hierarchies joined by a reference.
- The inheritance explosion is the diagnostic: class names that glue two concepts together are a missing bridge.
- Bridge is proactive, Adapter is reactive, and that timing test is the whole distinction.
- Each new value of either dimension costs one class with a bridge, not one class per combination.
- JDBC drivers are Bridge at industrial scale, one fixed abstraction and many hidden implementations.

## What's Next

The next article is Composite, which solves a different shape problem: part-whole hierarchies. Where Bridge splits things into parallel hierarchies, Composite builds a single tree and makes every node, leaf or branch, look the same to the client. We will cover the uniform `Component` contract, the recursive structure, and why treating a whole panel exactly like a single button is the entire point.

---

This article explains the Bridge pattern as a proactive split of two independently growing dimensions into parallel hierarchies joined by a single reference. It argues that the class grid whose names glue two concepts together is the diagnostic, and that the timing test, proactive split versus reactive adapter, is what separates Bridge from Adapter.
