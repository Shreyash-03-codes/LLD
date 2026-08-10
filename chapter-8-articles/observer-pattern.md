# Observer Pattern

## Learning Objectives

- Build a subject that notifies observers through a common interface, so new observers attach without the subject changing.
- Choose push versus pull notification deliberately, and explain what each one couples to what.
- Argue for Reactive Streams and event-driven frameworks as the successors to the plain pattern, and bluntly state when Observer stops paying for itself.

## Introduction

Observer defines a one-to-many dependency: when one object, the subject, changes state, it notifies every observer that subscribed, without the subject knowing anything about their concrete types. The observers know the subject through the notification it sends. The subject knows the observers only as a list of objects that implement the observer interface.

The pattern is the answer to a specific pain: the thing that changes does not want to hardcode everyone who cares about the change. A `UserService` should not hold references to an emailer, a cache warmer, and an analytics logger, because every new interested party means editing `UserService`. Observer inverts that, the interested parties subscribe, and the subject emails a list.

## Problem Statement

Here is the coupling that begs for the pattern. A user registers, and the registration flow has acquired side effects:

```java
public void register(User user) {
    save(user);
    emailer.sendWelcome(user);
    cacheWarmer.warm(user);
    analytics.trackRegistration(user);
    ledger.record(user);
}
```

Delete the word "yet" and hear what this code promises: every new consequence of a registration means editing `register`. Marketing wants a promo email. Support wants an audit trail. A recommendation system wants the user pushed to a queue. Each one is a new line in this method, and the method grows until it is doing five unrelated things and the person who edits it has to understand all five to add a sixth.

The coupling is the real cost, not the lines. `User` now depends on `EmailSender`, `CacheWarmer`, `Analytics`, and `Metrics`, through one method, and all of them depend on being called in the right order. The registration flow, which is the core business event, is welded to a growing pile of concerns that have nothing to do with saving a user. The subject, registration, should not know the audience. It has no business knowing that a promo team exists.

## Core Concept

Observer decouples the event from its consequences. The subject exposes subscribe and notify, and the consequences become observers:

```java
public interface UserListener {
    void onUserRegistered(User user);
}
```

The subject keeps a list and notifies every listener in a guard loop:

```java
public class UserService {
    private final List<UserListener> listeners = new ArrayList<>();

    public void addListener(UserListener listener) {
        listeners.add(listener);
    }

    public void register(User user) {
        save(user);
        for (UserListener listener : listeners) {
            listener.onUserRegistered(user);
        }
    }
}
```

Each consequence becomes an observer:

```java
public class WelcomeEmailService implements UserListener {
    @Override
    public void onUserRegistered(User user) {
        emailer.sendWelcome(user);
    }
}
```

The wiring moves to the composition root, where the events and their consequences are bound:

```java
userService.addListener(new WelcomeEmailService());
userService.addListener(new CacheWarmListener());
userService.addListener(new AnalyticsListener());
```

Now registering a user triggers welcome email, cache warming, and analytics without `UserService` knowing any of them exist. A new consequence is a new listener and one line of wiring. The subject stopped growing, and each consequence is testable on its own, an `AnalyticsListener` test does not need a `UserService`.

Diagram: subject broadcasting to its observers

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 960 400" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#444"/>
    </marker>
  </defs>

  <rect x="40" y="150" width="260" height="110" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="170" y="174" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">UserService</text>
  <text x="170" y="200" text-anchor="middle" font-size="12" fill="#1a2733">-listeners: List</text>
  <text x="170" y="224" text-anchor="middle" font-size="12" fill="#1a2733">+addListener()</text>
  <text x="170" y="248" text-anchor="middle" font-size="12" fill="#1a2733">notify() on register</text>

  <rect x="620" y="40" width="280" height="70" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="760" y="66" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">WelcomeEmailService</text>
  <text x="760" y="92" text-anchor="middle" font-size="12" fill="#1a2733">onUserRegistered(user)</text>

  <rect x="620" y="165" width="280" height="70" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="760" y="191" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">CacheWarmListener</text>
  <text x="760" y="217" text-anchor="middle" font-size="12" fill="#1a2733">onUserRegistered(user)</text>

  <rect x="620" y="290" width="280" height="70" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="760" y="316" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">AnalyticsListener</text>
  <text x="760" y="342" text-anchor="middle" font-size="12" fill="#1a2733">onUserRegistered(user)</text>

  <line x1="300" y1="195" x2="618" y2="85" stroke="#6b7a89" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="300" y1="205" x2="618" y2="200" stroke="#6b7a89" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="300" y1="215" x2="618" y2="315" stroke="#6b7a89" stroke-width="1.5" marker-end="url(#arrow)"/>
</svg>
```

One subject on the left, three observers on the right, each arrow a single notification. The subject does not know which observers exist; it knows only the interface and the list. That is the decoupling the pattern sells.

### Push versus pull

The design decision that matters most is whether the subject sends the data or the observers fetch it. Push delivers the state as part of the notification:

```java
public interface UserListener {
    void onUserRegistered(User user);
}
```

Pull delivers just the signal and lets the observer query the subject:

```java
public interface UserListener {
    void onUserChanged();
}
```

Push is convenient and common, but it couples every observer to the exact payload. Add a field to the event and all observers' signatures change. Pull keeps observers thin, but every observer reaches back into the subject, which can re-couple them. The honest answer is a hybrid: push the parts everyone needs, the identifier and the facts, and let observers pull the details they happen to care about. That, a small immutable event object carrying the essentials, is what most production systems converge on, and it is what event classes in Spring and Guava represent.

### Threading is not optional

The naive loop notifies observers on the caller's thread, synchronously, in order. That is correct and it is also a footgun. A slow emailer blocks registration for everyone, and an observer that throws aborts the remaining observers. The subject needs a policy for both: it should fail fast on a broken observer, and any observer that can be slow or unreliable should move off the hot path, onto a queue, an executor, or its own thread, so a mail outage cannot stall registration. The pattern does not grant threading. The subscription model makes it easy to wire a new observer onto a different thread, but you have to own that decision.

"Ordering and idempotency. The notification order is the subscription order, and the moment two observers both react to the same event, order can matter. An observer that decrements inventory must run after the one that validates stock, not before. The subject gives no ordering guarantee beyond insertion order, and it gives no guarantee of exactly-once delivery either, an observer can be notified once, twice, or never, depending on where it was in the list when an exception stopped the loop. Depend on a single event type, not on a position; and any observer that must run in a defined order relative to another is better made a dependency instead of a sibling subscription.

## Real Production Usage

The JDK shipped the pattern for decades as `java.util.Observer` and `java.util.Observable`, and deprecated both, which is the most public admission that the classic shape has problems, the `setChanged` flag is awkward and letting a class extend `Observable` costs an inheritance slot. The successor is `java.util.concurrent.Flow`, with its `Publisher`, `Subscriber`, and `Subscription`, and it keeps the same push model while adding backpressure. Reactive Streams, the `Flux` and `Mono` types in Project Reactor and RxJava, is Observer at industrial scale: a stream of events, subscribers, and backpressure.

Spring events are Observer in daily use. `@EventListener` methods on beans subscribe to events published by `ApplicationEventPublisher`, and Spring handles the wiring, the filtering, and, optionally, async delivery. Swing is built on it: `ActionListener`, `MouseListener`, and every other `*Listener` is an observer attached to a component. When a GUI button does something, an observer received the event. When a `JmsTemplate` or Kafka consumer gets a message and a `@KafkaListener` method runs, that is a subscriber far removed from the producer. When you see an annotation that is invoked when something else happens, you are looking at an observer.

## Common Mistakes

**Overusing it until the subscription graph is untraceable.** Observer's strength, that observers don't know each other, is its weakness. You subscribe and subscribe and then you cannot say who causes what. The question to ask before adding an observer is whether the subscriber genuinely cares about the event or whether you are using it to avoid naming a dependency. Too many observers should be dependencies.

**Sharing state through the event.** If the observable event object carries the subject's internal field, observers start depending on an ordering that is not their business. The event should be a fact, not a view of everything.

**Ignoring the synchronous bomb.** A listener that throws aborts the loop, and a slow network listener holds the subject's thread. The subject needs a policy, catch per observer, or async, or both, before the first slow subscriber appears.

## Interview Perspective

Observer is where interviewers test whether you understand the trade behind the pattern. A weak answer draws the subject and the observers and says "one change, many reactions." A strong answer names push versus pull, and can say why the old `Observable` was deprecated and what `Flow` fixed, backpressure. Interviewers probe the coupling too. "Who is the coupling between?" The subject knows the observer interface, the observer knows the subject's event, and both know the event type, and the strong answer names all three instead of claiming anyone has zero coupling.

The follow-up that sorts people is the overuse question: "When would you avoid Observer?" The strong answer says when the wiring becomes implicit and untraceable, or when the coupling through a shared event model is worse than a direct call, and it can point to Mediator as the pattern when you need wiring that is coordinated and explicitly named.

Common follow-ups:

- "Push or pull, and why?"
- "An observer throws. What happens to the other observers?"

## Knowledge Check

1. Refactor the `register` method's four side effects into observers, then add a promo-email subscriber and show what changes in `UserService`.
2. Compare push and pull on this dimension: how many code sites change when a new field is added to the event? Walk both.
3. A slow `AnalyticsListener` stalls registration. Describe two ways to make it non-blocking, and one consequence each.

## Key Takeaways

- Observer inverts side effects: the thing that changes stops knowing who cares, and the interested parties subscribe.
- The one-to-many push is the shape, and the list of listeners is the only coupling the subject keeps.
- Push, or pull, or the hybrid event object is a real design decision with real consequences.
- Synchronous, in-thread notification is the default and a footgun; threading and failure policy are yours to own.
- `Flow`, Reactive Streams, and Spring's `@EventListener` are Observer in production, and the old `Observable` was deprecated for good reason.

## What's Next

The next article is Command, which has a narrower job than Observer and does it without any listener plumbing. Where Observer answers who gets told about a change, Command packages an action itself into an object, so a whole operation, a method call, its receiver, and its parameters, can be stored, queued, and undone. We will cover the invoker-receiver split, and why Command is the pattern behind tasks, menus, and transactions.

---

This article explains the Observer pattern as a one-to-many broadcast with the observer detached from the subject. It argues that the broadcast, not the listener, is the whole point of the pattern.