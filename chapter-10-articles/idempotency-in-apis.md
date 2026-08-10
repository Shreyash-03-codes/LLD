# Idempotency in APIs

## Learning Objectives

- Distinguish an idempotent operation from a safe one, and say which HTTP methods are idempotent.
- Use an idempotency key in a header so a retried create returns the same result once.
- Design the store that makes idempotency real: write the key, keep the result, replay it on a duplicate.

## Introduction

A client that does not deliver quickly calls again. The network times out, the mobile app is busy, the batch job retries on the line. If the call was a create, the retry risks turning one click into two orders. Idempotency is the property that lets a server say "you already did that" and hand back the same result instead of doing it a second time. It is the difference between a PDF that tolerates retries and a checkout that charges twice.

## Problem Statement

The basic create is not idempotent. `POST /orders`, then a timeout, then a retry, now the client has two order rows with two ids, two charges, two reservations. The server has no way to tell "this is a second attempt at the same thing" from "the user made a second order", so it cannot reuse the work it already did.

The process of mitigating it by hand, "did this request already happen?", cannot be answered from the request alone. Two identical create requests are indistinguishable without a shared value the client sends and the server remembers. Without that value, the only defense is weak: disable the button, de-duplicate by some content hash, hope the user does not click twice. None of those survive a broken network.

## Core Concept

### Safe, idempotent, neither

The first step is vocabulary. A safe method has no side effect at all, it never changes server state, so repeating it is harmless. GET is safe, and a safe method is always idempotent. An idempotent method can be repeated with the same effect as a single call: the first call creates, and every subsequent identical call is the equivalent of the first. `PUT` and `DELETE` are naturally idempotent, setting a value twice leaves the same value, and a delete of an already-deleted resource is a no-op. `POST` is not, that is the sequence this by default.

So, in plain terms:

| Method | Side effect | Retry-safe |
|--------|-------------|-----------|
| GET    | none        | yes        |
| PUT    | replace     | yes        |
| DELETE | remove      | yes        |
| POST   | create      | no, not without a key |

The gap in the `POST` row is the whole of this article.

### The idempotency key

The fix is a value the client sends, with the request, an idempotency key in the header, a UUID the client generates once and reuses on retries. When the client creates an order, it sends `Idempotency-Key: xyz`. If the request times out, it sends the same key on the retry. The server stores the key with the result, sees the same key a second time, and replays the stored result instead of creating again.

```java
String key  = request.getHeader("Idempotency-Key");   // client's UUID
Order  saved = idempotency.runOnce(key, () -> orderService.create(payload));
```

What the client is really doing is handing trust to the server. A key per logical operation, one layout per intent, and the server only ever sees the retry's identity consistently. The server now has a discriminator it can trust.

### The store that makes it real

Idempotency needs state. On the first request with a key the server runs the command, stores the key mapped to the result (status, body), and returns it. On a duplicate key it looks up the stored result and replays it, unchanged, without running the command again. The store typically has a TTL, because the idempotency window only needs to cover the client's retry horizon, hours, not forever.

```java
@Transactional
public OrderResponse run(String key, Supplier<OrderResponse> command) {
    Optional<OrderResponse> existing = repo.findByKey(key);
    if (existing.isPresent()) {
        return existing.get();          // replay, no second create
    }
    OrderResponse result = command.get();
    repo.save(new IdempotencyEntry(key, result));
    return result;
}
```

The discipline is that the key write happens in the same transaction as the business write. If the server creates the order but crashes before it stores the key, the client retries, the server cannot tell it already created, and you are back to two. The two writes, the business effect and the idempotency record, must commit together.

Diagram: first call executes, retry replays

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 380" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#33475b"/>
    </marker>
  </defs>

  <text x="105" y="45" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">first call</text>
  <rect x="40" y="62" width="130" height="52" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="105" y="92" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">Client</text>

  <line x1="170" y1="88" x2="248" y2="88" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <text x="209" y="78" text-anchor="middle" font-size="11" fill="#5a6b7a">key K</text>

  <rect x="250" y="62" width="170" height="52" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="335" y="92" text-anchor="middle" font-size="11" fill="#1a2733">first request</text>

  <line x1="420" y1="86" x2="498" y2="86" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>

  <rect x="500" y="62" width="170" height="52" rx="10" fill="#fdf0ef" stroke="#a94442" stroke-width="1.5"/>
  <text x="585" y="92" text-anchor="middle" font-size="11" fill="#a94442">execute + store</text>

  <line x1="670" y1="86" x2="748" y2="86" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>

  <rect x="750" y="62" width="130" height="52" rx="10" fill="#eef6ee" stroke="#4a8a4a" stroke-width="1.5"/>
  <text x="815" y="92" text-anchor="middle" font-size="11" fill="#4a8a4a">response</text>

  <text x="40" y="225" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">retry</text>
  <rect x="40" y="242" width="130" height="52" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="105" y="272" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">Client</text>

  <line x1="170" y1="266" x2="248" y2="266" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <text x="209" y="256" text-anchor="middle" font-size="11" fill="#5a6b7a">key K</text>

  <rect x="250" y="242" width="170" height="52" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="335" y="272" text-anchor="middle" font-size="11" fill="#1a2733">duplicate seen</text>

  <line x1="420" y1="266" x2="498" y2="266" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>

  <rect x="500" y="242" width="170" height="52" rx="10" fill="#d6e6f7" stroke="#2a6fbf" stroke-width="1.5"/>
  <text x="585" y="272" text-anchor="middle" font-size="11" fill="#2a6fbf">lookup: found</text>

  <line x1="670" y1="266" x2="748" y2="266" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>

  <rect x="750" y="242" width="170" height="52" rx="10" fill="#eef6ee" stroke="#4a8a4a" stroke-width="1.5"/>
  <text x="835" y="272" text-anchor="middle" font-size="11" fill="#4a8a4a">replay stored</text>
</svg>
```

The first call executes and stores its result under the key. The retry carries the same key, finds the stored entry, and replays it. The business command runs exactly once, and the duplicate gets the identical, repeated response.

### The response is a replay, not a re-run

A clean idempotency returns the same body and status both times. The point is that the replay and the execute both answer with the same shape and code, so a client cannot tell them apart, which is the point, both are "the result of the operation with that key". If a client that simply retries sees a different thing the first and second time, the de-dup is not done.

There is a subtlety worth stating: the key does not make two calls safe by magic, it makes them identical. The first call creates, and the replay returns that same result, so the second identical body never reaches the business logic. That is exactly why the key has to be generated per intent and reused across retries, the server's whole claim of safety rests on the client being consistent about that one header.

## Real Production Usage

Stripe made the idempotency key famous. A client that retries a payment with the same key is guaranteed not to charge twice, and the docs recommend it for any create-like call. The store-and-replay is the same pattern, a cached entry keyed by the request, in front of the payment. Any write that creates a billable thing, a charge, a reservation, a checkout, has the same need, and the platform standards put the key on the same pattern.

In Java, the idiomatic home is a small idempotency component sitting in front of the use-case, writing to the same transaction, and a TTL. Spring Data JPA, or a single Redis entry keyed by, can hold the record, both of which are the already-seen replay. The rule "key + result writes with the business write" is the whole design.

## Common Mistakes

**Using the business payload as the key.** Two different intents may have an identical body, and two copies of the same order then collide. The key is a random value the client generated, so a retry and a genuinely new order are told apart.

**Storing the key in a second phase / outside the transaction.** The order commits, the key write fails on a crash, and the retry creates again. The idempotency and the business change must be one atomic transaction.

**The key TTL shorter than the client retries.** A TTL of a few seconds, the client retries after an hour and gets a double. The TTL must cover the longest actual retry window.

## Interview Perspective

An interviewer is checking that you know idempotency is a de-dup-by-key, not a method jargon. A weak answer conflates it with safe operations. A strong answer separates safe (GET) from idempotent (PUT/DELETE, retry-safe) from POST, adds a client-generated key in the header, stores the result, and replays on the duplicate, with the key write in the transaction.

The follow-up that lands is "how do you retry a `POST` without charging twice?" The strong answer sends an `Idempotency-Key`, the server executes once, stores the result, and replays it for the same key. The candidate who answers "make POST return the idempotency key" or "have the client double-check" is describing a hack. The server-held state is the theme they need to lead with.

Common follow-ups:

- "What is the difference between a safe and an idempotent request?"
- "Where is the idempotency key allowed to be (body, path, header), and why the header?"
- "What goes wrong if the key is a hash of the payload?"

## Knowledge Check

1. GET is safe, DELETE is idempotent but not safe. Layer out the distinction with a side effect in the middle.
2. A create times out and the client retries with the same key. What has to be true of the server's transaction for that to be safe?
3. Why is the stored response replayed verbatim, and what would a client see if the replay produced a different body the second time?

## Key Takeaways

- Idempotent is not the same as safe, POST is the case that needs the key.
- The key is a client-generated UUID reused on every retry.
- Store the result, and on a duplicate key replay it without re-running the command.
- Write the key and the business result in the same transaction or the replay lies.

## What's Next

The API can now run in a world of retries; how the callers identify themselves is a separate axis. API security basics covers the next layer of entry: how a request proves the caller, the tokens and the principal, the difference between authentication and authorization, and the shape of the 401 and 403 the error article promised.

---

This article explains idempotency as keyed de-duplication, the result stored under a client key, the duplicate replayed verbatim. It argues a POST is not safe, so the retry needs a key, and the key must commit atomically with the write.