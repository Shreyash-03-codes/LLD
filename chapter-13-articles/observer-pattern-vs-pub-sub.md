# Observer Pattern vs Pub-Sub: Where They Differ

## Learning Objectives

- Describe the observer pattern and the reference it depends on.
- State the single fact that separates the observer pattern from pub-sub: who holds the reference.
- Choose the in-process java shape versus a broker by where the event is worth gathering.

## Introduction

A colleague writes "it is basically pub-sub" and means an `Observable`. The sentence is not wrong so much as it is dangerously coarse. The observer pattern and pub-sub share the same story: a subject notifies its dependents that something changed. The difference is not the plot, it is who holds the listener. In the observer pattern, the subject keeps a list of observers and calls each one directly. In pub-sub, no one holds the subscribers, the broker holds the subscription, and the two ends never look at each other. Same fan-out, a different vehicle.

## Problem Statement

A price view uses an observer list and updates too cheaply while there are two observers. Someone adds a logger observer that writes to a slow disk, and now every price change waits on the disk write, because the observer is called synchronously on the subject's own thread. A later observer that throws stops the loop, so the observers after it never hear about the change. The bug is not the logger; it is the pattern. The subject is coupled to the number and the speed of its observers, and the in-process call makes the subject's health depend on theirs.

## Core Concept

The observer pattern is a small set of object relationships. The subject, the observable, holds a collection of observers and offers attach and detach. When its state changes it calls `notifyObservers`, which meanwhile loops over the list.

```java
interface Observer {
    void update(String state);
}

class PriceSubject {
    private final List<Observer> observers = new ArrayList<>();
    private String price;

    void attach(Observer o) { observers.add(o); }

    void setPrice(String p) {
        this.price = p;
        observers.forEach(o -> o.update(p));
    }
}
```

The decisive layout says: the subject holds a direct reference to every observer and calls it on the subject's thread. That is an in-process, synchronous, same-thread exchange. In Java, this is the `Observable` and `Observer` classes and the `PropertyChangeListener` used by JavaBeans and Swing.

Pub-sub changes the reference. No subscriber reference sits in the publisher. A broker is between them, holding the topics and the subscriptions. The publisher publishes to a topic, a consumer subscribes to a topic, and the two are not linked. The in-process version of this is an `EventBus`, and the cross-process version is Kafka. Pub-sub can live asynchronously with a broker and can be durable, which the observer never is.

| Aspect | Observer | Pub-sub |
|--------|----------|---------|
| Reference | subject holds observers | broker holds subscriptions |
| Same process | yes, in-thread | broker, can be across processes |
| Failure of one subscriber | blocks/spreads the rest | isolated at the broker boundary |
| Durability | none | often yes |
| The Java take | `Observable` | `EventBus`, Kafka |

## Real Production Usage

The observer is where the recipient is a view that must never miss the change and wants it cheap. `java.util.Observable` is old and mostly retired, and `PropertyChangeListener` has a real life in Swing. Pub-sub is where the ends are separate processes: a payment event emitted in service A and handled in service B across a node. The in-process Spring `@EventListener` is an honest middle, it has the pub-sub shape, one-to-many, no topic reference, but it lives in one JVM. The honest rule: if the two ends must survive a machine restart, pick the broker. If they are two beans in one process and the reply must arrive in the same thread, an observer is the design.

## Common Mistakes

1. **Thinking the observer is async.** An observer `update` runs on the subject's thread. The slowest observer slows every subscriber.
2. **Quoting a topic where the ends are one process and the answer is synchronous.** An in-process, hand-rolled pub-sub for two beans is an observer with a detour through a thread, and a worse debugging surface.
3. **Putting a topic and a reply in the same shape.** An observer can return, a pub-sub event carries no request-reply. Asking a pub-sub consumer to reply is a translated request, not an event.

## Interview Perspective

Interviewers ask "observer vs pub-sub" to break a person who memorized "it's just what it is" from one who names the reference. Strong: "the subject holds a list of observers and calls them on its own thread; pub-sub holds the subscription in a broker, so the publisher never knows the subscribers. Same volume, different coupling." The follow-up is "when do I reach for a broker" and the answer names the machine: two processes that must survive and replay means the broker; two beans that must synchronously update mean the observer.

## Knowledge Check

- One observer throws and the others stop getting the update. What does that prove the subject owns, and could it produce the same bug? Explain.
- A service uses `@EventListener` in one process. Is it a true pub-sub or an observer? Answer with who holds the reference.
- A view needs fast in-process changes; a log consumer across a node needs a halt-free stream. Which goes with the observer and which with pub-sub, and why?

## Key Takeaways

- The observer pattern, the subject holds the list and calls each directly on its thread.
- In pub-sub the broker holds the subscription; the publisher does not know its subscribers.
- Decide on who holds the reference. `Observable` is gone and Kafka is real, but the rule between is the only one you keep.

## What's Next

The shapes decouple the ends, now the content: a correct event is worth more than an elegant pub-sub bus. The next article covers designing good events, events you name and version, so the subscriber does not have to reach into the entity to guess.

---

This article explains the difference between the observer pattern and pub-sub as a single fact, where the subscriber's reference is held, in the subject or in the broker. It argues that the shared fan-out hides a real coupling difference, and that dragging an observer into a cross-process problem without a broker is a failure and not a replay.