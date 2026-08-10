# Adapter Pattern

## Learning Objectives

- Implement an object adapter in Java and explain why composition-based adaptation beats inheritance in this language.
- Recognize the difference between adapting an interface and fixing broken code, which is where Adapter stops and a refactor begins.
- Point at the JDK's own adapters as the fastest way to understand the pattern in production form.

## Introduction

Adapter converts the interface of a class into an interface clients expect, so two systems that were never designed to talk to each other can work together. Nothing changes about the adapted class. It does not learn the client's vocabulary. A thin translator object sits between them and speaks both languages.

The pattern is boring on purpose. It is the structural equivalent of a power plug converter, and it is one of the few patterns where the mundane version is the correct version. If your adapter is clever, you are probably doing something else.

## Problem Statement

The setup is painfully common. Your team owns a `ChatService` that the whole codebase depends on:

```java
public class ChatService {
    public void sendMessage(String to, String text) {
        // ...
    }
}
```

Then a directive arrives: route messages through a vendor SDK, `LegacyMessenger`, that cannot be changed. Its API is from a different decade:

```java
public class LegacyMessenger {
    public void dispatch(int recipientId, String payload) {
        // ...
    }
}
```

The mismatches are every one you dread. Your callers pass a recipient name, the vendor wants an integer id. Your method is named `sendMessage`, theirs is `dispatch`. If you change `ChatService` to call the vendor directly, every call site in the codebase has to know about `recipientId` and `payload`, which means every call site has to know how to translate a name to an id. The translation logic, which is exactly the logic that should live in one place, gets copy-pasted everywhere, and when the vendor's API changes, you hunt every copy.

That is the concrete failure: an interface mismatch that, left unmanaged, spreads the vendor's vocabulary across the entire system. The translation does not belong in the caller, and it cannot belong in the vendor. It needs its own home.

## Core Concept

The adapter's home is a new class that implements the interface the clients expect and delegates to the vendor. Clients keep calling `ChatService`-shaped methods. The adapter does the translation.

```java
public interface MessageSender {
    void sendMessage(String to, String text);
}
```

The clients keep this interface, so their code never changes. The adapter implements it and translates into the vendor's language:

```java
public class LegacyMessengerAdapter implements MessageSender {
    private final LegacyMessenger messenger;
    private final ContactDirectory directory;

    public LegacyMessengerAdapter(LegacyMessenger messenger, ContactDirectory directory) {
        this.messenger = messenger;
        this.directory = directory;
    }

    @Override
    public void sendMessage(String to, String text) {
        int recipientId = directory.lookupId(to);
        messenger.dispatch(recipientId, text);
    }
}
```

That is the entire pattern. `LegacyMessenger` never changes. The callers never change. The adapter is the only class in the system that knows about `dispatch`, `recipientId`, and how a name becomes an id. When the vendor changes their API, one class changes. When a new vendor arrives, you write a second adapter and choose it at wiring time. This is Adapter at its most useful: the mismatch is contained in exactly one class, and the translation logic has a single owner.

Diagram: object adapter structure

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 960 390" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#444"/>
    </marker>
  </defs>

  <rect x="40" y="40" width="200" height="70" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="140" y="64" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">Client</text>
  <text x="140" y="88" text-anchor="middle" font-size="12" fill="#1a2733">+call()</text>

  <rect x="360" y="40" width="240" height="80" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="480" y="58" text-anchor="middle" font-size="12" font-style="italic" fill="#5a6b7a">&lt;&lt;interface&gt;&gt;</text>
  <text x="480" y="78" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">MessageSender</text>
  <text x="480" y="100" text-anchor="middle" font-size="12" fill="#1a2733">+sendMessage(to, text)</text>

  <rect x="360" y="260" width="240" height="80" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="480" y="282" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">LegacyMessengerAdapter</text>
  <text x="480" y="306" text-anchor="middle" font-size="12" fill="#1a2733">+sendMessage(to, text)</text>

  <rect x="660" y="260" width="240" height="80" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="780" y="282" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">LegacyMessenger</text>
  <text x="780" y="306" text-anchor="middle" font-size="12" fill="#1a2733">+dispatch(id, payload)</text>

  <line x1="240" y1="75" x2="358" y2="75" stroke="#6b7a89" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
  <line x1="480" y1="260" x2="480" y2="122" stroke="#33475b" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
  <line x1="600" y1="300" x2="658" y2="300" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
</svg>
```

### Why composition, not inheritance

GoF describes two flavors: the class adapter, which inherits from the adaptee, and the object adapter, which holds it. Java makes this a non-choice, because a class can extend only one class, and the class adapter needs to extend both the adaptee and, to be useful, the target. Since `MessageSender` is an interface here, the class adapter would extend `LegacyMessenger` and implement `MessageSender`, which is actually legal, and some Java code does exactly that:

```java
public class LegacyMessengerAdapter extends LegacyMessenger implements MessageSender {
    private final ContactDirectory directory;

    @Override
    public void sendMessage(String to, String text) {
        dispatch(directory.lookupId(to), text);
    }
}
```

It works, and it is worse. Inheritance bakes in the adaptee's entire surface, including methods you never wanted to expose. The object adapter exposes only what you choose. Inheritance also means the adapter cannot adapt an interface or a class you do not control cleanly, and it cannot adapt a `final` class at all. Composition gives you one dependency you can mock in tests, which inheritance cannot do. Use the object adapter. The class adapter is a curiosity in this language, not a tool.

### The adapter's place in the codebase

The adapter pattern has a reputation for being trivial, and that reputation is the danger. The real skill is knowing where the adapter belongs and where it does not. An adapter belongs at the boundary between code you own and code you do not: vendor SDKs, legacy services, third-party libraries. It does not belong between two classes you own, where fixing the interface directly is almost always cheaper than maintaining a translator. If you control both sides, change one side. Adapter is for the boundary where you cannot change the other side.

## Real Production Usage

The JDK is full of adapters, and they are the clearest way to internalize the pattern. `InputStreamReader` adapts a byte-oriented `InputStream` to the character-oriented `Reader` interface. `Arrays.asList(...)` adapts an array to the `List` interface, which is why that list cannot be structurally modified, the array underneath is fixed. `Collections.list(Enumeration)` adapts the old enumeration API to the modern `List`. Every JDBC driver is an adapter: the `java.sql.Driver` contract is fixed by Sun, and each database vendor implements a driver that adapts their native protocol to that contract.

The logging world runs on it too. SLF4J exists because Java has several logging frameworks, and SLF4J's backends, `logback`, `log4j` bindings, are adapters that translate the unified API to each implementation. When you see a library whose whole job is to make API X look like API Y, you are looking at the pattern, even if nobody named it.

The adapter also makes the boundary testable, which is worth more than the interface neatness. Because the object adapter holds the vendor dependency behind a reference, the rest of your tests can hand it a fake and never touch the vendor at all. The client's tests run against `MessageSender`, the adapter's tests assert the translation, and the vendor stays out of the test suite entirely. That division is a concrete payoff of composition, and it is the same reason the pattern survives when a vendor ships a new version with different method names or changed semantics: you swap one adapter for another and re-run the same harness, instead of rewriting every call site.

## Common Mistakes

**Adapting instead of fixing.** If both classes are yours, an adapter is a permanently installed excuse for a mismatch you should have removed. The pattern is for boundaries you cannot change, not for internal laziness. Adapting your own broken design just gives the breakage a permanent home.

**Putting business logic in the adapter.** The adapter should translate interface calls, not implement behavior. If your adapter is reordering arguments, transforming domain objects, and enforcing policy, it has become a service in disguise. Translation belongs in the adapter; decisions belong elsewhere.

**Exposing the adaptee through the adapter.** If callers can reach past the adapter and touch `LegacyMessenger` methods, the adapter is decorative. The whole point is that nothing outside the adapter knows the adaptee exists.

## Interview Perspective

Adapter is where interviewers check whether you understand the *why* more than the *how*. The how is five lines. A weak answer defines the pattern and draws the wrapper. A strong answer names the boundary: "I use Adapter when the mismatch is between my code and code I cannot change, and I use the object adapter because composition lets me mock the dependency and expose only the interface I chose."

The follow-up that sorts people is the comparison with the other wrappers. "Adapter and Facade both wrap a class. What is different?" The answer is intent: Adapter changes the interface to make incompatible things work, Facade simplifies a subsystem to make a complex thing easy. Same shape, opposite jobs.

Common follow-ups:

- "Why is the object adapter preferred over the class adapter in Java?"
- "Your vendor ships a new version with the same method names but different semantics. Is an adapter the answer?"

## Knowledge Check

1. Draw the call chain for `Arrays.asList(...)`: who is the client, who is the target interface, and what does the adapter sacrifice about the array that callers must know about?
2. A class extends `LegacyMessenger` and implements `MessageSender`. List three concrete problems this class adapter has that the composed version does not.
3. You own both `ChatService` and `LegacyMessenger`. Argue whether an adapter is the right move or a shortcut, and say what you would do instead.

## Key Takeaways

- Adapter translates one interface into another so incompatible code can cooperate, without touching either side.
- The object adapter, composition-based, is the Java answer; the class adapter exposes too much and cannot adapt final classes.
- The adapter's home is the boundary with code you do not control, and adapting your own code is how the pattern rots.
- The JDK is full of adapters, `InputStreamReader`, `Arrays.asList`, JDBC drivers, and SLF4J backends being the clearest.
- An adapter translates calls. If it is making decisions, it is not an adapter anymore.

## What's Next

The next article covers Bridge, which is what happens when you recognize a two-dimensional mismatch *before* anything is broken and split it in advance instead of patching it. We will look at the inheritance explosion that Bridge prevents, the two hierarchies it creates, and why the pattern is routinely confused with Adapter even though it attacks a different problem.

---

This article explains the Adapter pattern as a translator that sits between code you own and code you cannot change, using a legacy messenger SDK as the running example. It argues that the object adapter is the only serious choice in Java, and that the pattern belongs strictly at external boundaries, never between classes you control.
