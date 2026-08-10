# Circuit Breaker Pattern

## Learning Objectives

- Explain the three states, closed, open, half-open, and which one fails fast, which one trips, and which one probes recovery.
- Design the transition thresholds: failure rate, wait duration in open, and the trial size of half-open, so recovery is neither a stampede nor a chill.
- Argue for the fail-fast default when a dependency is sick, and why rejecting calls early is sometimes the whole value.

## Introduction

The circuit breaker is the idiom from house wiring carried into a service call: when a dependency is failing, stop trying it before the retries make it worse. The breaker sits between the caller and the dependency and quietly counts. Closed when the dependency is healthy. Open when it has failed enough to give up. Half-open while it tests recovery with a trickle. The breaker gives the caller a policy for "this dependency is not worth 10 more seconds; fail fast and fail now," and it does the coordinating retry and backoff cannot: it stops the storm entirely.

## Problem Statement

A service depends on an internal pricing API that has begun returning 500s two minutes ago. Every checkout calls it, waits the full 5-second timeout, and then waits again under the retry budget. Every checkout is now 15 seconds of waiting on a dead endpoint, hundreds of threads are parked on timeouts, the connection pool is exhausted, and checkout is down, not because the dependency is down but because the callers are dutifully waiting out their retries. The correct behavior is the opposite of patient: once the dependency is demonstrably sick, the calls should fail in milliseconds so the checkout can respond with "temporarily unavailable" and stop consuming resources on a lost cause. The retry article gave the backoff budget; the circuit breaker provides the fail-fast gate at the end of that budget.

## Core Concept

### The three states

A breaker is a gate with a threshold. The breaker is a state machine, and the transitions between its states are the entire pattern.

- **Closed**: calls flow through as normal, failures counted. When the failure rate crosses the threshold (say 50% over the last 100 calls), the breaker trips to Open.
- **Open**: calls fail fast without touching the dependency. After a fixed **cooldown** (say 10 seconds) it does not recover by itself; it becomes Half-Open.
- **Half-Open**: a trickle of calls (say 5) is allowed through as a live test. If they succeed, Closed again and traffic flows. If they fail, back to Open and the cooldown restarts.

The name of the half-open question is the whole point of the pattern: the breaker must not trust the timer alone, it must probe. A dependency that recovers for a second and then fails again would flip-flop if the transition waited only on time.

```java
@Configuration
public class PricingConfig {
    @Bean
    public CircuitBreaker pricingBreaker() {
        return CircuitBreaker.of("pricing", CircuitBreakerConfig.custom()
                .failureRateThreshold(50)
                .waitDurationInOpenState(Duration.ofSeconds(10))
                .permittedNumberOfCallsInHalfOpenState(3)
                .slidingWindowSize(100)
                .build());
    }
}
```

Diagram: the breaker's three states and the transitions a failure or a recovery causes.

<svg width="860" height="300" viewBox="0 0 860 300" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="cb" markerWidth="10" markerHeight="10" refX="8" refY="4" orient="auto">
      <path d="M0,0 L8,4 L0,8 Z" fill="#333"/>
    </marker>
  </defs>

  <ellipse cx="150" cy="150" rx="78" ry="38" fill="#eef7ee" stroke="#2f6f3e" stroke-width="2"/>
  <text x="150" y="146" text-anchor="middle" font-family="Arial" font-size="14" fill="#222">CLOSED</text>
  <text x="150" y="164" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">calls flow</text>

  <ellipse cx="430" cy="80" rx="72" ry="38" fill="#fdeeee" stroke="#962828" stroke-width="2"/>
  <text x="430" y="76" text-anchor="middle" font-family="Arial" font-size="14" fill="#222">OPEN</text>
  <text x="430" y="94" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">fail fast</text>

  <ellipse cx="660" cy="150" rx="86" ry="38" fill="#fdf6e3" stroke="#8a6d00" stroke-width="2"/>
  <text x="660" y="146" text-anchor="middle" font-family="Arial" font-size="14" fill="#222">HALF-OPEN</text>
  <text x="660" y="164" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">trial traffic</text>

  <path d="M 228 150 C 300 150 300 100 356 92" stroke="#962828" stroke-width="2" fill="none" marker-end="url(#cb)"/>
  <text x="308" y="105" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">failure rate &gt; 50%</text>

  <path d="M 502 92 C 560 92 590 115 610 135" stroke="#8a6d00" stroke-width="2" fill="none" marker-end="url(#cb)"/>
  <text x="565" y="98" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">cooldown elapsed</text>

  <path d="M 575 160 C 520 160 500 150 240 150" stroke="#2f6f3e" stroke-width="2" fill="none" marker-end="url(#cb)"/>
  <text x="440" y="186" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">trial passes → CLOSED</text>

  <path d="M 660 115 C 640 60 520 60 500 80" stroke="#962828" stroke-width="2" fill="none" marker-end="url(#cb)"/>
  <text x="600" y="55" text-anchor="middle" font-family="Arial" font-size="11" fill="#555">trial fails → OPEN</text>
</svg>

The failure-rate transition is the gate that starts it, but the cooldown and the trial are what keep it from thrashing. The half-open rate of three calls will test a recovering service without the flood; if it still fails, the breaker rejects once more and lets the dependency rest.

### Why fail fast is the win

Most of a circuit breaker's value is in the Open state. With the dependency sick, the breaker drops every call at the gate, so the response is a fast "no" instead of a 5-second timeout; the caller can fall back or fail fast; and the dependency that was overwhelmed now only sees the half-open trickle. This is why a breaker implemented as little more than "cooldown then retry" misses the point: the half-open probe and the failure-rate window are the part that prevents thrashing. A breaker without the probe is a timer with a guess.

### Retries and the breaker compose

Retry and a breaker both react to a failing dependency, and they order: retry first within a budget, and then let the breaker trip. An idempotent call is retried twice, the breaker trips, and traffic fails fast after the second failure. If you flip the order, the breaker opens on the first error and the retry never exercises a moment of the dependency recovering. The two form one policy: excuse a bounded number of failures, then stop the storm. The budget and the gate sit at different layers, and they have to rank.

### What to send when it breaks

The fail-fast response is a product decision as much as an engineering one. When the breaker is open, the caller should return something with a distinct signal, not a generic error. The choices:

- a clear "temporarily unavailable, retry later" with a retry header the client can use,
- a fallback: serve a known-stale value (an old price) when freshness is less important than availability,
- no fallback, a plain error, where no alternative exists.

Which dependencies get a fallback and which fail hard is a deliberate choice. Blanket-fallback everything and you serve confidently wrong data; blanket-fail everything and a cacheable dependency takes down traffic it could have served.

## Real Production Usage

Resilience4j's `CircuitBreaker` is the Java mainstream, configured in code or properties and named per policy. It composes with its own Retry, and the order is the important part, the retry decorates the call on the inside and the breaker sits outside it. Spring Cloud Circuit Breaker exposes a common API over Resilience4j. The library handles the sliding-window bookkeeping and the half-open probe; teams that reach for it generally get the semantics right and then misallocate the thresholds, so the first config opens the breaker on a normal blip or, worse, never closes.

## Common Mistakes

1. **Confusing the breaker with the retry budget.** The breaker is not "another retry"; it is the gate that stops the retries from a sick dependency. Ordering retry-inside-breaker is right, breaker-inside-retry is a storm on the first failure.
2. **A half-open with a threshold too close to closed.** The trial set must be small and measured against a distinct failure count; if it reuses the same wide window, the probe barely tests anything and the breaker flips on a single straggler.
3. **Ignoring the fallback.** A breaker without a fallback or a clear "unavailable" reads as a random outage to the caller; a generic thrown error is not fail-fast, it is just fast and unexplained.

## Interview Perspective

The question "walk me through a circuit breaker" wants the three states, and the transition is the value. The strong answer: "closed until the failure rate crosses the threshold, open fails fast, half-open with a trickle after the cooldown, and the trip is a probe of recovery; the retry sits inside the breaker budget and the fallback is a product choice, not an error." The weak answer explains open and closed only; an interviewer then asks "why half-open," and the answer is the probe that prevents flip-flopping, not "you test."

## Knowledge Check

1. A dependency starts failing 60% but recovers in short pauses every few seconds. How does the timing of the half-open test run and how does it avoid the flip-flop?
2. The breaker trips and the caching fallback returns data from 10 minutes ago. The caller doesn't know it's a fallback. Is that a mistake? What's the reason one way?
3. Stacking the order retry→breaker vs breaker→retry materially within a calling sequence. Spell out the period when each pairing makes a dependency worse at a break.

## Key Takeaways

- The three states with the failure threshold, the cooldown, and the half-open probe form the whole policy.
- Fail-fast is the primary win: a closed dependency is cheap; a sick dependency is cheapest to make fast.
- The breaker pairs with retries (retry inside, breaker on top), and the fallback is a product decision, not code noise.

## What's Next

The circuit breaker gates one service's behavior toward a failing dependency. The next article flips to the outer boundary and the thinnest client: rate limiting algorithms, how a single service defends its own capacity against too much demand, and why the enforcement time window is the decision that separates a rate limit from a breaker.

---

This article explains the circuit breaker as a three-state fail-fast gate for a failing dependency. It argues the value is in stopping the storm fast, that a retry sits inside the breaker, and that a fallback is a product choice, not an error.
