# The Publisher-Subscriber Model

## Learning Objectives

- Name the three roles and the channel in a pub-sub system.
- Explain the broadcast semantics of a topic versus the one-worker semantics of a queue.
- Choose the right delivery mode for a workload instead of defaulting to one.

## Introduction

The last article introduced the idea of publishing events. This one names the parts. Pub-sub runs on three players: the publisher that produces, the broker that holds the events, and the subscriber that consumes. Two things separate it from a plain producer-consumer queue, and the distinction decides what you can build. First, subscribers pick the events they want. Second, a published event reaches every subscriber, not just a worker. The broadcast is what makes the model strong and what makes the delivery guarantees subtle.

## Problem Statement

Two services both need to react to a user signing up, one sends an email and the other records analytics. A point-to-point queue hands each message to exactly one consumer, so one service receives the event and the other never hears about it. The naive fix, two duplicate queues, forces the producer to know both consumers. What you want is a single publication that every interested subscriber receives a copy of. That behavior is a topic, and the design falls apart the moment you use queue semantics where a topic belongs, or the reverse.

## Core Concept

The model has two delivery modes, and every design here is picking one.

- **Topic:** a broadcast channel. Every subscriber that is interested receives every message published. This is the fan-out, one write, many readers.
- **Queue:** a work queue. A message is handed to exactly one consumer from a competing set, and after one takes it, the others do not.

Kafka and RabbitMQ each wrap these in a confusing way. Kafka exposes a durable append-only topic, and inside that topic a consumer group behaves like a queue: each partition of the topic is consumed by one member of the group. Multiple consumer groups over the same topic give fan-out, one fan per group, while each group is a queue. RabbitMQ has a plain queue primitive that hands a message to one worker, and a topic exchange that routes copies to every binding. So the practical rule is: a topic with several consumer groups is a broadcast to each group, and one consumer group is a queue over the topic's partitions.

| Delivery | Queue | Topic |
|----------|-------|-------|
| Who receives the message | one worker | every subscriber |
| Same message for two consumers | no | yes |
| Position tracking | consumed, deleted | offset per consumer |
| Replay | usually no | often yes |
| Java tool | a work queue / a consumer group | a Kafka topic or a topic exchange |

When you model a stream, you also decide the coupling: Kafka gives you durability and per-partition order but not global order. The Java tool you reach for, `@KafkaListener` for external topics or Spring's `@EventListener` for in-JVM wiring, sets how many copies the event fans out to and how honest the durability is, but the semantics of the event model are the same on paper.

## Real Production Usage

Real systems mix the two deliberately. A `UserRegistered` event is a topic: the email service and the analytics service each subscribe with their own group and each gets its own copy. A "send this one email" retry job is a queue: several workers compete, and exactly one must send it or the users get duplicate mail. The same Kafka topic can serve both with different groups. The pattern that holds: an event topic drives producer-side views, one consumer per concern, while a job topic or a queue drives one-of-N workers that each take a distinct message. Decide, before writing the listener, whether the consumer is a group member (queue) or one of several independent groups (fan-out), because changing it later means reworking the offset and the replay.

## Common Mistakes

1. **Using a topic where one worker must act.** A resend job with three subscribers and each one sends the email. That needs queue semantics, where each message moves to exactly one worker.
2. **Assuming replay is always free.** Replay works from a saved offset, not from thin air. If the consumer advances the offset before committing the work, then a crash and the replay silently skips the event.
3. **Ignoring partition order.** Events for one order share a key, so order holds per key, but two consumers in a group over different partitions can process different events concurrently and reorder one entity's history.

## Interview Perspective

Interviewers ask "topic versus queue" to hear whether you map delivery to the workload and not a memo. Weak: a memorized, two-word distinction. Strong: "I use a topic when several subscribers each want the event and a queue or a consumer group when exactly one worker should act; Kafka's topic plus one consumer group is both, so it covers the two." Follow-ups: "how does a consumer group behave over a Kafka topic" and "what changes when a member leaves." A good answer names rebalancing: the partitions reassign, the offset track continues, and the consumer that took over starts from the group offset, not from where it was.

## Knowledge Check

- A `StockLow` event is published. An email service and a single checkout worker both read it. Which one needs the topic semantics and which may be fine on a queue, and why?
- A Kafka topic has three partitions and two members. How many partitions can be actively consumed at a time, and what never happens?
- A consumer reads the event, writes the reply to its own database, then commits. Why must the commit happen after the write, and what failure does each order cause?

## Key Takeaways

- A topic fans one event out to every subscriber; a queue hands the message to one worker.
- Kafka's topic-with-consumer-groups combines both in one broker, which is why it dominates.
- Define each topic's consumers as groups or fans before you write the listener.

## What's Next

The model has the broker now, and a Java programmer inevitably asks: is this not just the observer case? The next article draws the line between the observer pattern and pub-sub, where they share the same shape and where the differences, in who holds the reference and in how many actors move, are real.

---

This article explains the publisher-subscriber model as the three roles and the two delivery modes, a topic that broadcasts to every subscriber and a queue that hands each message to one worker. It argues that Kafka's topic plus consumer group is the deciding power because it covers both, and that mixing the two semantics is the cheapest way to lose or duplicate an event.