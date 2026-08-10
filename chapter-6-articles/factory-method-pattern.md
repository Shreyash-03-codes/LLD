# Factory Method Pattern

## Learning Objectives

- Implement Factory Method with subclasses overriding a factory method, and recognize when the "switch on a type parameter" version is a weaker cousin rather than the same thing.
- Explain why Factory Method removes the caller's dependence on concrete classes, and what you lose when you skip it.
- Use the pattern's real strength, which is that it sits on a decision point that subclasses can override, not that it "creates objects."

## Introduction

Factory Method defines an interface for creating one object and lets subclasses decide which concrete class to instantiate. The name is misleading. The pattern is not primarily about factories, it is about a seam. The caller does not write `new ConcreteThing()`. It calls a method whose implementation is open for subclasses to replace, and that replacement decides what comes back. The creation is where the decision is expressed, but the decision is the point.

This is the most broadly useful of the five creational patterns, and it is the one with the least ceremony. You do not need a separate factory object. You need a method that can be overridden.

## Problem Statement

Without it, adding a new variant is a game of whack-a-mole across the codebase. Say you have three notification channels and a service that routes to them:

```java
public class NotificationService {
    public void notifyUser(String userId, NotificationType type, String message) {
        Notification notification;
        switch (type) {
            case SMS -> notification = new SmsNotification();
            case EMAIL -> notification = new EmailNotification();
            default -> throw new IllegalArgumentException("Unknown type: " + type);
        }
        notification.send(message);
    }
}
```

This works until it does not. The `switch` is the tell. Every call site that needs a notification must either replicate this routing or go through a type. The class grows a `PUSH` branch, then a `SLACK` branch, and every branch is more code doing the same thing. The deeper problem is that `NotificationService` now knows about every concrete notification class in the system. Adding a channel means editing the router, which means editing a class that should have been stable. The failure is not that the switch is ugly. It is that the class that decides what to build is the same class that has to change when a new thing appears, and it is the same class that a test has to instantiate to exercise a single channel.

## Core Concept

Factory Method moves the "decide what to build" step into a method that subclasses override. The base class still orchestrates the work, but the act of construction is delegated.

```java
public interface Notification {
    void send(String message);
}

public class SmsNotification implements Notification {
    @Override
    public void send(String message) {
        // call the SMS provider
    }
}

public class EmailNotification implements Notification {
    @Override
    public void send(String message) {
        // call the email provider
    }
}

public abstract class NotificationService {
    public void notifyUser(String userId, String message) {
        Notification notification = createNotification();
        notification.send(message);
    }

    protected abstract Notification createNotification();
}
```

The concrete services only decide what to build:

```java
public class SmsNotificationService extends NotificationService {
    @Override
    protected Notification createNotification() {
        return new SmsNotification();
    }
}

public class EmailNotificationService extends NotificationService {
    @Override
    protected Notification createNotification() {
        return new EmailNotification();
    }
}
```

Now `notifyUser` no longer knows about `SmsNotification` or `EmailNotification` at all. It depends on `Notification`, and the subclass is the only place in the system that names a concrete class. Add a push channel tomorrow and you add two classes, `PushNotification` and `PushNotificationService`. You do not touch `NotificationService`. That is the payoff: the code that uses a product stays closed while the set of products keeps growing.

Diagram: Factory Method class structure

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1000 320" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#444"/>
    </marker>
  </defs>

  <rect x="30" y="30" width="440" height="80" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="250" y="52" text-anchor="middle" font-size="12" font-style="italic" fill="#5a6b7a">&lt;&lt;interface&gt;&gt;</text>
  <text x="250" y="72" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">Notification</text>
  <text x="250" y="92" text-anchor="middle" font-size="13" fill="#1a2733">send(message)</text>

  <rect x="30" y="200" width="210" height="80" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="135" y="222" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">SmsNotification</text>
  <text x="135" y="248" text-anchor="middle" font-size="13" fill="#1a2733">send(message)</text>

  <rect x="260" y="200" width="210" height="80" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="365" y="222" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">EmailNotification</text>
  <text x="365" y="248" text-anchor="middle" font-size="13" fill="#1a2733">send(message)</text>

  <line x1="135" y1="200" x2="135" y2="112" stroke="#33475b" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
  <line x1="365" y1="200" x2="365" y2="112" stroke="#33475b" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>

  <rect x="560" y="30" width="330" height="90" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="725" y="52" text-anchor="middle" font-size="12" font-style="italic" fill="#5a6b7a">&lt;&lt;abstract&gt;&gt;</text>
  <text x="725" y="72" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">NotificationService</text>
  <text x="725" y="92" text-anchor="middle" font-size="13" fill="#1a2733">notifyUser(userId, message)</text>
  <text x="725" y="110" text-anchor="middle" font-size="13" fill="#1a2733">createNotification()</text>

  <rect x="560" y="200" width="190" height="80" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="655" y="222" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">SmsNotificationService</text>
  <text x="655" y="248" text-anchor="middle" font-size="13" fill="#1a2733">createNotification()</text>

  <rect x="770" y="200" width="190" height="80" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="865" y="222" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">EmailNotificationService</text>
  <text x="865" y="248" text-anchor="middle" font-size="13" fill="#1a2733">createNotification()</text>

  <line x1="655" y1="200" x2="655" y2="122" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <line x1="865" y1="200" x2="865" y2="122" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>

  <line x1="560" y1="240" x2="242" y2="240" stroke="#6b7a89" stroke-width="1.5" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
</svg>
```

Two details in that diagram matter more than the shapes.

First, the factory method is `protected abstract`, and notice what that buys you. The pattern is really Template Method wearing a creational hat. `notifyUser` runs the algorithm (create, send), and `createNotification` is the hook where the algorithm is parameterized. So Factory Method does not just decouple construction, it lets a subclass change construction without reimplementing the whole operation. That is a strictly more powerful move than calling a static factory, and it is the reason the pattern survives in real frameworks.

Second, the diagram shows the inheritance version, but most working code uses a parameterized cousin that is worth being honest about:

```java
public class NotificationService {
    public Notification create(NotificationType type) {
        return switch (type) {
            case SMS -> new SmsNotification();
            case EMAIL -> new EmailNotification();
            case PUSH -> new PushNotification();
        };
    }
}
```

This is the version you will find scattered through production codebases. It centralizes construction in one method, which already fixes the worst of the scattered-`new` problem. It is not the pattern as specified, because nothing is overridden and there is no seam for a subclass to redirect, but it is the pragmatic 80 percent. If the switch version is what your codebase actually looks like, the pattern you want to study is still worth it: it is what the switch version becomes when someone later needs a subclass that builds differently.

## Real Production Usage

The JDK is full of factory methods, and the line between a static factory method and the Factory Method pattern is worth drawing. `java.util.concurrent.Executors.newFixedThreadPool(4)`, `List.of(...)`, `Collections.singletonList(...)`, and `NumberFormat.getNumberInstance()` are static factory methods. They centralize construction but they are not overridable, so they capture the spirit, not the subclass seam. `Stream.of` and the entire streams API follow the same shape.

For the pattern proper, the place to look is Spring. `FactoryBean<T>` exists exactly to let a bean definition decide, in code, how a product is built, and framework callbacks like `ApplicationContext.getBean(String)` hand back objects whose concrete type the caller never names. Hibernate does the same with `SessionFactory.getCurrentSession()`. The pattern is less visible in the JDK than in frameworks because a library usually has no reason to give you a seam to override; frameworks have every reason to.

## Common Mistakes

**Treating Factory Method as a synonym for "any method that calls new."** The pattern is defined by the override seam. If your factory method cannot be overridden to change what it returns, you have a helper method, not the pattern. The distinction matters in interviews and it matters when you are deciding whether a seam is worth adding.

**Overusing the pattern until there is a factory for everything.** A factory for a class with one implementation and one constructor is pure overhead. The pattern earns its keep when the number of product classes is expected to grow, or when a subclass must be able to change the product. Apply it at the seam, not at every constructor.

**Returning the concrete type from the factory method.** `public SmsNotification createNotification()` in a base class that returns only SMS defeats the purpose. The return type should be the abstraction, or callers still compile against the concrete class and the decoupling is fictional.

## Interview Perspective

Interviewers like Factory Method because it is the creational pattern with the most conceptual depth per line of code. A weak answer describes the diagram. A strong answer does two things: connects it to Template Method, and can say exactly what breaks when you replace the pattern with a switch statement.

The follow-up that separates people is usually "when would the subclass version be worth more than the switch version?" The right answer names the seam: when different subclasses of the service need to produce different concrete products, or when the factory method's output must itself be customizable. If nothing ever needs to override, the switch is fine and the full pattern is ceremony.

Common follow-ups:

- "What is the difference between Factory Method and Abstract Factory?"
- "If the factory method is called by the base class's own algorithm, what other pattern does that make this?"

## Knowledge Check

1. Refactor the `switch` version of `NotificationService` so a new channel can be added without modifying the service, and name every file you had to touch.
2. A teammate argues that the switch version and the subclass version are "basically the same." Give one concrete scenario where they are not, where only the subclass version solves the problem.
3. Why does the base class `notifyUser` avoid depending on any concrete `Notification`? Trace the dependency direction and say which classes change when `PushNotification` is added.

## Key Takeaways

- Factory Method delegates the choice of concrete class to an overridable method, keeping callers coupled only to the abstraction.
- The pattern is Template Method in disguise: `notifyUser` is the algorithm, `createNotification` is the hook.
- The parameterized switch version is the everyday reality and fixes most of the real pain, but it gives up the override seam.
- Add the pattern where the set of products is expected to grow, not where a single concrete class is certain.
- Frameworks like Spring and Hibernate use the pattern at their seams, which is why a library rarely needs it but a framework cannot live without it.

## What's Next

The next article moves from one object to a coordinated set. Abstract Factory is what Factory Method becomes when you need to create a family of related objects and guarantee that everything comes from the same family. We will build the shape with a concrete UI example, cover why the "consistency across a family" constraint is the entire reason the pattern exists, and look at the tradeoff that makes it the most expensive creational pattern to add to a codebase.

---

This article explains Factory Method as a creation seam that subclasses override, not as a factory object, and shows how it keeps callers coupled to an abstraction instead of a concrete class. Its central claim is that the pattern is really Template Method in disguise, and that the override seam, not the act of creating, is what gives it lasting value.
