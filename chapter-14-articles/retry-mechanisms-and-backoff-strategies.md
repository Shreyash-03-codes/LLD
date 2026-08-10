# Retry Mechanisms and Backoff Strategies

## Learning Objectives

- Decide which failures are retryable and which must never be retried, and how to tell by error type instead of by hope.
- Explain why naive fixed retries turn a recovering dependency into a thundering herd, and what exponential backoff plus jitter does about it.
- Wire retry together with idempotency so a duplicate delivery of a non-idempotent request is handled, not feared.

## Introduction

Retries are a simple story that gets complicated by two words: "again" and "when." Most upstream failures are transitory, and retrying the same call moments later succeeds. But a retry that ignores timing and idempotency converts one hiccup into a stampede and one request into two invoices. The design of retries is really the design of two decisions: which failures deserve a retry at all, and how to spread those retries so they do not amplify the very failure they are retrying. 

## Problem Statement

A checkout flow calls a payments gateway. The gateway is having a bad minute: half the calls time out. The service retries immediately, three times, with no delay, from every checkout request at once. The gateway is already struggling, and suddenly it is receiving four times its usual traffic in two-second bursts, because every client retried in the same instant. The gateway gets worse, retries get worse. Meanwhile the "safe" retry that succeeded on the first attempt actually reconciled twice, because the caller delivered the request, timed out waiting for the response, retried, and the gateway had already charged the card; a duplicate charge with no safety net. Retries made a blip into a self-inflicted outage and an outage into a double-billing.

## Core Concept

### Not all failures are retryable

The first decision is at the error, not the retry loop. Retrying a validation error is a waste; the client must change the payload. Retrying an auth failure is foolish, the token is still wrong. Retrying is for failures that plausibly pass if the future is slightly different: connection drops, timeouts, 5xx, 429 with a Retry-After. The rule you can hold: retry only when a fault is transient or its outcome is unknown, never when the outcome is deterministically "no." A failed auth cannot become a success by repetition; a read timeout might, because the server may have completed the work and simply the response was lost.

That last case is the subtle one. A timeout is ambiguous: the operation may have failed, or it may have succeeded and the answer is lost. Ambiguity is exactly why retries need idempotency.

### Idempotency is the retry's safety net

A retry doubles the request, so the downstream must be able to recognize the duplicate. The mechanism is an idempotency key: the client attaches a stable key per logical operation, and the server replays the stored result for the same key instead of executing again.

```java
// client side, one key per logical action, reused across retries
HttpRequest req = HttpRequest.newBuilder(uri)
        .header("Idempotency-Key", keyFor(orderId, "capture"))
        .POST(body)
        .build();
```

The server stores the response for each seen key and returns it on duplication. With the key in place, "retry until you get a definitive answer" is safe, because a retry that hits the completed operation returns the stored response and produces no new effect. Without the key, every timeout is a gamble. The rule that summarises: idempotent operations, safe to retry freely; non-idempotent operations, retry only with a key or not at all.

### Backoff: fixed, exponential, and exponential plus jitter

Given a retryable failure, when do you retry? The naive answers are wrong in opposite directions:

- **Immediate retry**: retries at the exact moment of failure, which is when the dependency is most loaded. It amplifies the load that caused the failure.
- **Fixed delay**: spreads nothing against spikes if a fixed 1 second is the same for every client; the herd still arrives together, just shifted.
- **Exponential backoff**: delay grows with each attempt, 1s, 2s, 4s, 8s. This is the core, it gives the dependency time to recover and spreads clients by their attempt count, but the same growth is deterministic, so all failing clients that started together still retry together.
- **Exponential backoff + jitter**: the delay is random around the exponential, 2^attempt scaled by a random factor. Jitter breaks the synchronization. This is the production answer, and it is what stops the thundering herd of simultaneous retries from a fleet that all failed at the same instant.

```java
public long nextDelay(int attempt, long baseMs, long maxMs, ThreadLocalRandom random) {
    long cap = Math.min(
        (long) (Math.pow(2, attempt) * baseMs),    // exponential growth
        maxMs
    );
    return random.nextLong(0, cap);                 // full jitter
}
```

The full-jitter variant picks uniformly between zero and the exponential cap. Some teams use equal-jitter, half the delay plus a random slice, which is fine and keeps a minimum delay. The essential property is the randomness on the shared cap, which spatializes the herd.

Drawing what happens:

Diagram: fixed delays keep retries synchronized, exponential plus jitter spreads them across the window.

<svg width="880" height="300" viewBox="0 0 880 300" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="rt" markerWidth="10" markerHeight="10" refX="8" refY="4" orient="auto">
      <path d="M0,0 L8,4 L0,8 Z" fill="#333"/>
    </marker>
  </defs>

  <text x="30" y="42" font-family="Arial" font-size="13" fill="#222">Fixed delay: retries land together</text>
  <line x1="30" y1="60" x2="850" y2="60" stroke="#888" stroke-width="1"/>
  <rect x="40" y="50" width="14" height="14" fill="#333"/>
  <rect x="290" y="50" width="14" height="14" fill="#333"/>
  <rect x="540" y="50" width="14" height="14" fill="#333"/>
  <rect x="790" y="50" width="14" height="14" fill="#333"/>
  <text x="40" y="92" font-family="Arial" font-size="11" fill="#555">t=0</text>
  <text x="290" y="92" font-family="Arial" font-size="11" fill="#555">+2s</text>
  <text x="540" y="92" font-family="Arial" font-size="11" fill="#555">+4s</text>
  <text x="790" y="92" font-family="Arial" font-size="11" fill="#555">+6s</text>

  <text x="30" y="152" font-family="Arial" font-size="13" fill="#222">Exp + jitter: each client scatters within its window</text>
  <line x1="30" y1="170" x2="850" y2="170" stroke="#888" stroke-width="1"/>
  <rect x="40" y="160" width="14" height="14" fill="#333"/>
  <rect x="80" y="160" width="14" height="14" fill="#2f4f8a"/>
  <rect x="40" y="160" width="14" height="14" stroke="#333" fill="none"/>
  <rect x="180" y="160" width="14" height="14" fill="#333"/>
  <rect x="250" y="160" width="14" height="14" fill="#2f4f8a"/>
  <rect x="310" y="160" width="14" height="14" fill="#8a2f2f"/>
  <rect x="560" y="160" width="14" height="14" fill="#333"/>
  <rect x="640" y="160" width="14" height="14" fill="#2f4f8a"/>
  <rect x="700" y="160" width="14" height="14" fill="#8a2f2f"/>
  <text x="40" y="210" font-family="Arial" font-size="11" fill="#555">t=0</text>
  <text x="290" y="210" font-family="Arial" font-size="11" fill="#555">window 0-2s</text>
  <text x="540" y="210" font-family="Arial" font-size="11" fill="#555">window 0-4s</text>
  <text x="790" y="210" font-family="Arial" font-size="11" fill="#555">window 0-8s</text>

  <rect x="30" y="240" width="16" height="16" fill="#333"/>
  <text x="54" y="253" font-family="Arial" font-size="11" fill="#555">client A</text>
  <rect x="120" y="240" width="16" height="16" fill="#2f4f8a"/>
  <text x="140" y="253" font-family="Arial" font-size="11" fill="#555">client B</text>
  <rect x="210" y="240" width="16" height="16" fill="#8a2f2f"/>
  <text x="230" y="253" font-family="Arial" font-size="11" fill="#555">client C</text>
</svg>

The top row is the fixed-delay herd: every client's third retry lands at roughly the same second, on a dependency that is already struggling. The bottom row is the jittered version: the same clients hit inside widening windows, so the arrival is a smear instead of a spike.

### Max attempts and total time

Backoff assumes a bound. Retry forever is not a strategy, it is a busy loop with a time budget. Three to five attempts is a common range, and the total elapsed is bounded by the caller's timeout. A retry beyond the client's patience just wastes upstream work. The check to design: the deadline, "I will try for 5 seconds total, backing off," beats "I will try forever," because a hoarding client that retries for minutes holds a thread and a connection the whole time and can exhaust the pool.

### Retry at the right layer

Retries can live in the HTTP client, the service layer, or the message transport, and they compose badly if stacked. The classic mistake is retries at every level: the LB retries, the HTTP client retries, the method-level wrapper retries, so the downstream sees many more issues than designed. Decide one layer for retries (usually the client at the boundary), with a single budget, and let every other layer propagate. A useful number to keep: if the frontend retries 3 times and the backend retries 3 times, the upstream can see 3x per logical request at peak.

The exception: idempotency must live as close as possible to the effect. A refund stage will not be retried in the same way as a plain query; retrying a query is free, retrying an effect needs the idempotent key.

## Real Production Usage

Spring Retry and Resilience4j both implement backoff with jitter; Resilience4j's Retry decorates a callable with max attempts and a backoff, and it integrates with the same library that offers the circuit breaker (next article, the Retry and the breaker compose: retry first within budget, then break). A Kafka consumer that fails enough retries hands the message to the DLQ (dead letter queue), the terminal bucket after the retries exhaust. The piece about real systems: the retry budget is shared with the circuit, and teams set the max attempts in config, not in code, so the pager can turn retries down live.

## Common Mistakes

1. **Immediate or fixed retries on a transient outage.** Retry at the moment of the failure and you amplify the failure. The categorical remedy is exponential and jitter.
2. **Retrying a non-idempotent effect with the same key.** The duplicate charge has no safety net; idempotency is a separate requirement, and the missing key is reported from the duplicate bill, not the log.
3. **Retries layered too many times.** Client plus LB plus service-wrapper retries multiply what the upstream sees. One layer with the budget, the rest pass through.

## Interview Perspective

The question "how do you retry a failing call" wants to separate theory from the herd. The weak answer is "just retry a few times." The strong answer names the missing pair: "retry only transient or ambiguous failures, key idempotency for operations, exponential backoff with jitter, a max-attempt budget, and one layer owning the retries, leaving the circuit breaker to stop the storm." The interviewer's follow-up "why jitter" is answered with the thundering herd, and "what is idempotency" with the replay result.

## Knowledge Check

1. A client times out on a POST that charges a card. The server may or may not have charged. What does your retried path add first, and why?
2. Ten clients hit the same service with a retryable 503 at the same second. Show with fixed delay vs exponential+jitter how the drop arrives on the server in each, and the difference it saves.
3. Your library has retry in the HTTP client and a proxy retry outside. What does the upstream see on a 5-minute stall, and why is the one-layer rule the fix?

## Key Takeaways

- Retry only transient or ambiguous failures, never a deterministic no.
- Idempotency keys convert ambiguous timeouts into safe replays; non-idempotent retries need the key.
- Exponential backoff plus jitter is what defeats the thundering herd, with a max budget at one layer.

## What's Next

Retries give a bounded repeated attempt, but they can also make a sick dependency worse by piling on. The next article is the guard that sits behind the retry's budget: the circuit breaker pattern lets a failing dependency fail fast and stop the storm, which is what you reach for once "retry with backoff" is no longer enough.

---

This article explains retries as two decisions, which failures are safe to retry and how to spread them, via exponential backoff with jitter. It argues a deterministic delay keeps a herd synchronized, and an effect retried without an idempotency key is a double bill waiting.
