# Template Method Pattern

## Learning Objectives

- Define a `final` template method that owns the skeleton of an algorithm, with `abstract` steps and optional hooks left for subclasses.
- Explain the three seams (template, step, hook) and why overriding the template method destroys the pattern's invariant.
- Recognize the pattern in `InputStream`, `Servlet`, and Spring's `JdbcTemplate`, and argue when composition beats it.

## Introduction

Template Method fixes the shape of an algorithm in a single base class and lets subclasses supply the parts that vary. The base class owns a method that runs the whole sequence, marked `final` so no subclass can dilute it, and leaves the changeable steps `abstract` or overridable as hooks. A subclass implements only the steps; the base class drives them in the order it chose.

The defining move is who controls the sequence. The subclass does not control when its step runs, only what the step does. Order lives at the base, variation lives at the leaves.

## Problem Statement

The failure is an algorithm duplicated across concrete classes with the ordering drifting apart. A notification system runs the same sequence everywhere: render a message in the user's locale, deliver it through a channel, then log the result:

```java
public void sendEmail(User user, EmailTemplate template) {
    String body = renderEmail(template, user.getLocale());
    boolean ok = smtp.send(user.getEmail(), body);
    log(ok, user.getId());
}

public void sendSms(User user, SmsTemplate template) {
    String body = renderSms(template, user.getLocale());
    boolean ok = sms.send(user.getPhone(), body);
    log(ok, user.getId());
}

public void sendPush(User user, PushTemplate template) {
    String body = renderPush(template, user.getLocale());
    boolean ok = apns.send(user.getDeviceToken(), body);
    log(ok, user.getId());
}
```

Three methods, the same send loop copy-pasted three times. Now a requirement lands: add a retry on failed delivery. That change goes to three methods, and last time someone added a retry they only edited two. The ordering, "render, deliver, log," is expressed once per concrete class, so the sequence means three slightly different things and there is no single place that states it once.

That is the failure: the algorithm's order is duplicated, and every change must be replayed everywhere. The ordering is the part that never varies, yet it is the part copy- written the most.

## Core Concept

Template Method pulls the ordering into the base class and owns it with `final`:

```java
public abstract class NotificationSender {

    public final void send(Message message, User user) {
        String payload = render(message, user.getLocale());
        boolean ok = deliver(payload, recipientOf(message, user));
        afterDelivery(ok, user);
    }

    protected abstract String render(Message message, Locale locale);
    protected abstract String recipientOf(Message message, User user);
    protected abstract boolean deliver(String payload, String recipient);

    protected void afterDelivery(boolean ok, User user) {
        // hook: do nothing unless overridden
    }
}
```

The order, "render, deliver, then log," is now written once, and it is `final`, so no subclass can reorder or drop a step. The steps are `abstract`. Each channel implements exactly the parts that vary:

```java
public class EmailNotificationSender extends NotificationSender {
    protected String render(Message message, Locale locale) {
        return emailTemplate.render(message, locale);
    }

    protected String recipientOf(Message message, User user) {
        return user.getEmail();
    }

    protected boolean deliver(String payload, String recipient) {
        try {
            return smtp.send(recipient, payload);
        } catch (IOException e) {
            return false;
        }
    }
}
```

A new channel is a new subclass filling three methods. The retry lives in the base template, once. The ordering is fixed whether or not a subclass agrees. That is the contract: the base owns the "before and after," the subclass owns the "how."

The hook is where a subclass reacts without being forced to. A Kafka sender wants to treat a failed delivery differently, so it overrides the no-op hook:

```java
public class KafkaNotificationSender extends NotificationSender {
    @Override
    protected void afterDelivery(boolean ok, User user) {
        if (!ok) {
            deadLetter.push(user.getId());
        }
    }
}
```

The template still runs its sequence; this subclass just reacts at the point the base opened for it. No other channel is forced to care that Kafka dead-letters failures, because the hook default is nothing.

### The three seams

Template Method offers three kinds of variable part, and choosing which you need is the craft.

- The `final` template. The whole-algorithm method. It is the skeleton, and it must be `final` because the skeleton is the invariant. If a subclass overrides it, the ordering is gone and the pattern is wrecked.
- The `abstract` step. A step with no default that the subclass must supply. These are the genuine variations. They are the point of the pattern.
- The **hook**. An overridable method with a default that does nothing. Hooks let subclasses react at a point in the sequence, "after delivery," "before start," without being forced to implement anything. A hook nobody overrides is dead weight, so add a hook only when a second subscriber is genuinely likely.

The `final` keyword is where beginners get confused. The method says "override my subclasses, but not this one." so the pattern looks contradictory, a base class built to be extended yet sealed on its main entry. The resolution is the seams: extend the steps, never the template.

Diagram: the sealed template and the steps a subclass fills

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 860 380" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#444"/>
    </marker>
  </defs>

  <rect x="40" y="40" width="360" height="260" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="220" y="66" text-anchor="middle" font-size="14" font-weight="bold" fill="#1a2733">NotificationSender</text>
  <text x="220" y="120" text-anchor="middle" font-size="12" fill="#1a2733">send() : final</text>
  <line x1="90" y1="136" x2="340" y2="136" stroke="#33475b" stroke-width="1"/>
  <text x="220" y="166" text-anchor="middle" font-size="12" fill="#1a2733">render() : abstract</text>
  <text x="220" y="198" text-anchor="middle" font-size="12" fill="#1a2733">deliver() : abstract</text>
  <text x="220" y="230" text-anchor="middle" font-size="12" fill="#1a2733">recipientOf() : abstract</text>
  <text x="220" y="280" text-anchor="middle" font-size="12" fill="#1a2733">afterDelivery() : hook</text>

  <rect x="620" y="150" width="200" height="150" fill="#eef2f7" stroke="#33475b" stroke-width="1.5"/>
  <text x="720" y="176" text-anchor="middle" font-size="13" font-weight="bold" fill="#1a2733">EmailSender</text>
  <text x="720" y="202" text-anchor="middle" font-size="12" fill="#1a2733">render()</text>
  <text x="720" y="226" text-anchor="middle" font-size="12" fill="#1a2733">deliver()</text>
  <text x="720" y="250" text-anchor="middle" font-size="12" fill="#1a2733">recipientOf()</text>

  <line x1="400" y1="200" x2="618" y2="210" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <text x="510" y="168" text-anchor="middle" font-size="12" fill="#5a6b7a">extends</text>
</svg>
```

The base box seals `send()` above the divider and leaves `render`, `deliver`, and `recipientOf` open below it. The `EmailSender` extends the base and fills only the abstract steps. The arrow shows subclass, not composition. The order lives above the line, the variation below it, and the two never mix.

## Real Production Usage

The JDK templates are everywhere if you know the seam to look for. `AbstractList` implements the whole `List` contract from the abstract size and get, so subclasses implement little. `InputStream` is one of the cleanest: it gives you `read(byte[], int, int)`, builds `int read()` on top of it, and then `skip()` and `available()` on top of that. `Servlet` is a template: `service()` does the HTTP dispatch and hands off to the `doGet` and `doPost` hooks a subclass implements.

Spring's `JdbcTemplate` is the favorite. Its `execute`/`query`/`update` methods are templates: they open the resource, run your callback, handle warnings, and close, always so; you implement the callback. `RestTemplate.doExecute` does the same for HTTP, and `TransactionTemplate` around a transaction. In each, the fixed part owns the lifecycle and the varying part is yours. When you see a framework method that does the setup and teardown around a callback you supply, you are looking at a template method.

The pattern sits at the center of an honest debate. Template Method's inheritance is rigid: changing the base changes every subclass, and a class that does not fit the fixed order has no escape. Spring's newer `JdbcClient` and `RestClient` reach for composition, a fluent builder, instead of the template, which is a quiet admission that composition often fits modern code better. The template is strongest when the skeleton is genuinely static and shared; when the sequence itself is in flux, the composition wins.

## Common Mistakes

**Letting a subclass copywrite its own skeleton.** If a handler subclasses and then rewrites the whole method instead of the steps, the ordering escapes the base and the `final` is defeated. Fix it by extending the steps only.

**Marking the template before the sequence is stable.** The `final` locks the order. If the sequence is changing monthly, the base becomes a choke point that has to be touched every time, which is worse than no template. The stable skeleton is the prerequisite.

**Adding hooks nobody overrides.** Each hook is machinelike you have to read. If there is no second likely subscriber for "after delivery," skip the hook and call the common outcome inline.

## Interview Perspective

Behavioral interviewers use Template to check whether you know not just the shape but the trade. A weak answer draws a base class and two subclasses. A strong answer gives the three seams, says the template is `final` and why, and can point at `InputStream`, `Servlet.service()`, and `JdbcTemplate`. That last, JdbcTemplate, is the line between someone who has read the pattern and someone who has used it.

The follow-up that separates the concepts: "Template vs Strategy, both delegate the variable part. Why pick one?" The template answer: the base owns the sequence, the subclass fills the changing steps, inheritance. Strategy hands the whole variable behavior to the caller, composition, no `final`, no base. Template is right when the order is fixed and shared; Strategy is right when the variant can be chosen and changed at runtime or wiring.

Common follow-ups:

- "Your subclass overrides the `final` template method. What follows?"
- "Two callers want a different order for the same steps. Does Template Method work here? What would the alternative be?"

## Knowledge Check

1. In `NotificationSender.send()` add a "retry once on failure" step. Where does that step live in the design, and why must it not be inside any subclass?
2. What exactly is lost if the template `send()` is not `final`, and a subclass overrides it to a reordered sequence?
3. Take a method you maintain that has grown three near-identical copies. State the skeleton and its steps, and whether the skeleton is actually stable enough to lock down with Template or whether composition fits.

## Key Takeaways

- The base class owns the whole algorithm as a `final` template method; subclasses fill the varying steps.
- `abstract` steps are mandatory, hooks are optional, and overriding the template destroys the pattern.
- The base's `final` protects the order, so the skeleton has to be genuinely stable or the pattern costs you.
- `AbstractList`, `InputStream`, `Servlet.service()`, and Spring's `JdbcTemplate` are the real templates.
- Inheritance is the tool; where the base and the subclasses both change, composition beats the template.

## What's Next

The next article is State. Template Method froze one algorithm's order behind a `final` method; State freezes an object's legal behavior behind its current condition. The first is a fixed sequence of steps, the second is a set of behaviors and transitions where the object's condition decides which operations are allowed. We will cover the transition rules and when a state machine is worth its classes over a simple flag.

---

This article explains the Template Method pattern as a base class owning the algorithm's order in a final method. It argues that the final skeleton is the invariant and that composition often beats the template when the sequence is not stable.